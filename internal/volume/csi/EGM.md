# csi

## Overview

CSI adapter for the volume plugin interfaces: a gRPC `Client` for a CSI driver
(Identity, Controller, and Node services) and a `Plugin` that maps
`volume.VolumePluginControlPlane` and `volume.VolumePluginWorkerPlane` onto CSI
RPCs, with capability discovery and per-driver TLS.

## Interface

- `Client` (`client.go:Client`): gRPC connection wrapper with Identity, Controller, and Node service clients.
- `NewCSIClient(endpoint string, tlsCfg *tls.Config) (*Client, error)` (`client.go:NewCSIClient`): dials a CSI driver over unix, tcp, or dns endpoints.
- `(*Client).Close() error` (`client.go:Close`): closes the gRPC connection.
- Identity RPCs: `GetPluginInfo`, `GetPluginCapabilities`, `Probe` (`identity.go`).
- Controller RPCs: `CreateVolume`, `DeleteVolume`, `ControllerPublishVolume`, `ControllerUnpublishVolume`, `ControllerGetCapabilities` (`controller.go`).
- Node RPCs: `NodeStageVolume`, `NodeUnstageVolume`, `NodePublishVolume`, `NodeUnpublishVolume`, `NodeGetCapabilities` (`node.go`).
- `Plugin` / `NewPlugin(client *Client) *Plugin` (`plugin.go:NewPlugin`): implements `volume.VolumePluginControlPlane` and `volume.VolumePluginWorkerPlane` (`plugin.go` interface assertions), exposing `DriverName`, `CreateVolume`, `DeleteVolume`, `AttachVolume`, `DetachVolume`, `MountVolume`, `UnmountVolume`.
- `NewCSIPlugin(ctx, getter CSIDriverConfigGetter, driverName string, isController bool) (*Plugin, error)` (`plugin.go:NewCSIPlugin`): resolves the endpoint from a `CSIDriverConfig`, establishes the client, verifies the driver name, and runs capability discovery.
- `(*Plugin).SupportsControllerCapability` / `SupportsNodeCapability` (`plugin.go:SupportsControllerCapability`) and `InitControllerCapabilities` / `InitNodeCapabilities` (`plugin.go:InitControllerCapabilities`): capability reporting.
- `DefaultClientCertPath` / `DefaultCACertPath` (`plugin.go:DefaultClientCertPath`): pod identity and servicedns bundle paths.

## Invariants

- `Plugin` implements both volume plugin interfaces (`plugin.go` `var _ volume.VolumePluginControlPlane/VolumePluginWorkerPlane` assertions).
- `NewCSIPlugin` fails when the driver-reported name does not match the requested name (`plugin.go:newCSIPlugin`).
- Capability discovery is best-effort: a failed or empty query assumes every RPC is supported (`plugin.go:InitControllerCapabilities`); an `Unimplemented` answer disables that RPC for the process (`plugin.go:disableControllerCap`).
- Attach and detach are skipped when the driver does not report `PUBLISH_UNPUBLISH_VOLUME` support (`plugin.go:AttachVolume`).
- Staging runs only when the driver reports `STAGE_UNSTAGE_VOLUME` support, under `<nodepath.BasePath>/staging/<volumeID>` (`plugin.go:stageVolume`).
- Controller endpoint comes from the CSIDriverConfig; the node endpoint uses `NodeSocketOverride` or defaults to `/var/lib/kubelet/plugins/<driver>/csi.sock` (`plugin.go:newCSIPlugin`, `plugin.go:kubeletPluginSocketPath`).
- Only pod identity TLS is supported; when TLS is enabled a `ServerName` is required, TLS is at least 1.3 with ALPN h2, and the server chain is verified against the CA bundle in `VerifyConnection` on every handshake, cached until the file changes (`plugin.go:resolveTLSConfig`).
- Endpoints must use the `unix`, `tcp`, or `dns` scheme (`client.go:parseEndpoint`).
- All volume operations use a hardcoded `SINGLE_NODE_WRITER` mount capability (`plugin.go:getStandardCapabilities`).

## Side Effects

- Dials and holds a gRPC connection to the CSI driver; every RPC mutates or queries external storage state (`client.go:NewCSIClient`, `controller.go`, `node.go`).
- Creates and removes staging directories on the node filesystem (`plugin.go:stageVolume`, `plugin.go:removeStagingDir`).
- Reads the client certificate bundle (with rotation reloads) and the CA trust bundle from disk, caching the parsed pool between handshakes (`plugin.go:resolveTLSConfig`, `plugin.go:caPoolCache`).
- Emits slog warnings for unimplemented RPCs and capability fallbacks.

## Verification

- Run: `go test ./internal/volume/csi/...`
