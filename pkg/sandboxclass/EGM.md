# sandboxclass

Names the sandbox runtime families a `WorkerPool` or `SandboxConfig` can
select. Imports nothing from Kubernetes, so any package can name a class
without pulling in the `ate.dev/v1` API types.

## Interface

- `sandboxclass.go:GVisor` / `sandboxclass.go:MicroVM` - the two class constants (`"gvisor"`, `"microvm"`).
- `sandboxclass.go:All` - every sandbox class, as a fresh slice.
- `sandboxclass.go:Valid` - reports whether `class` is one of `All`; the match is case-sensitive.

## Invariants

- `All` must stay in sync with the `+kubebuilder:validation:Enum` markers in `api/infra/k8s/ate/v1` (`sandboxclass.go:All` doc comment).
- The package has no imports beyond the standard library, so naming a class never drags in Kubernetes API types (`sandboxclass.go` import block).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./pkg/sandboxclass/...`
