# localca

## Overview

A CA whose state can be stored in a local file or Kubernetes secret. A Pool
holds multiple CAs, one active for signing and the rest trusted for
verification, so rotation is seamless: publish a new root, age it in, switch,
age out, remove (`localca.go:Pool` doc).

## Interface

- `Pool` (`localca.go:Pool`): `CreateCertificate(template, subjectPublicKey)` signs with an active CA; `TrustAnchors()` returns all roots.
- `RefreshingPool` / `NewRefreshingPool(stateFile string) (*RefreshingPool, error)` (`localca.go:NewRefreshingPool`): Pool wrapper reloading CA state from disk, for controllers and servers.
- `ConcretePool` (`localca.go:ConcretePool`): in-memory pool with `CAs` and `ActiveForSigning`; implements `Pool`.
- `CA` (`localca.go:CA`): one signing root (`ID`, `SigningKey`, `RootCertificate`), with `TLSCertificateChainPEM` and `TLSPrivateKeyPEM` (`localca.go:TLSCertificateChainPEM`).
- `Marshal(pool *ConcretePool) ([]byte, error)` / `Unmarshal(wireBytes []byte) (*ConcretePool, error)` (`localca.go:Marshal`): JSON serialization with PKCS#8 keys.
- `GenerateCA(id string, keyType KeyType, validity time.Duration) (*CA, error)` (`localca.go:GenerateCA`): self-signed CA with a fresh key (ED25519 or ECDSA P-256, `localca.go:KeyTypeED25519`).

## Invariants

- A pool with no CAs fails to sign (`localca.go:ConcretePool.CreateCertificate`); an `ActiveForSigning` ID not in the list fails too.
- For backward compatibility, the first CA signs when none is designated (`localca.go:ConcretePool.CreateCertificate`).
- `RefreshingPool` reloads state at most once per minute and holds the lock across refresh and use, so signers survive administrator rotation without restarts (`localca.go:refreshIfNecessary`).
- Marshal fails with an explicit message for signers with no exportable key material, such as KMS or HSM signers (`localca.go:Marshal`).
- Generated roots are self-signed CA certificates with random CommonName content and DigitalSignature|CertSign key usage (`localca.go:GenerateCA`).

## Side Effects

- `RefreshingPool` reads the pool state file from disk on load and at most once per minute (`localca.go:refreshIfNecessary`).
- Certificate signing writes no external state; `GenerateCA` generates fresh key material (`localca.go:GenerateCA`).

## Verification

- Run: `go test ./internal/localca/...`
