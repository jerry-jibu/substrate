# deviceplugin

## Overview
Serves the kubelet device plugin API for host device nodes that worker sandboxes need (currently `/dev/kvm`), advertising each resource only on nodes where the device exists.

## Interface
- `deviceplugin.go:SandboxDevices` - the device list atelet advertises; only devices the container runtime denies by default belong here.
- `deviceplugin.go:HostDevice` - a device node with `ResourceName` and `Path`; `Present(devRoot)` reports whether it exists on the node.
- `deviceplugin.go:Available` - filters a device list to those present under `devRoot`.
- `deviceplugin.go:Plugin`, `deviceplugin.go:New`, `deviceplugin.go:Run` - serves the plugin and keeps it registered until the context is cancelled; implements `ListAndWatch` and `Allocate`.

## Invariants
- Advertising a resource only where its device exists keeps pods requesting it off nodes that cannot run them (`deviceplugin.go:Available`).
- Kubelet turns each `Allocate` `DeviceSpec` into a device node plus a matching cgroup allow, so the container gets this device and no other (`deviceplugin.go:Allocate`).
- Kubelet forgets registered plugins when it restarts; the plugin re-registers when kubelet recreates its socket or the resource disappears from the node (`deviceplugin.go:Run`).
- The device set is static: `ListAndWatch` sends it once and holds the stream open (`deviceplugin.go:ListAndWatch`).

## Side Effects
- Creates and serves a Unix socket per registration, registering it with kubelet's own socket (`deviceplugin.go:serveAndRegister`, `deviceplugin.go:register`).
- Watches kubelet's plugin socket with fsnotify to detect kubelet restarts (`deviceplugin.go:waitForKubeletRestart`).

## Verification
- Run: `go test ./internal/deviceplugin/...`
