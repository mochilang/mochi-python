# mochi-python

[![CI](https://github.com/mochilang/mochi-python/actions/workflows/ci.yml/badge.svg)](https://github.com/mochilang/mochi-python/actions/workflows/ci.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/mochilang/mochi-python.svg)](https://pkg.go.dev/github.com/mochilang/mochi-python)

The Mochi+Python package bridge. Standalone Go module that mirrors the `package3/python/` tree from [mochilang/mochi](https://github.com/mochilang/mochi); see [MEP-71](docs/mep/mep-0071.md) for the design and [docs/implementation](docs/implementation/index.md) for per-phase tracking.

## What this is

Two directions, both deterministic and offline-gated:

- **Consume**: `import python "<package>@<semver>" as <alias>` in Mochi source. The bridge resolves the package via uv, ingests PEP 561 stubs (4-tier precedence: inline `py.typed`, sibling `<name>-stubs`, typeshed, stubgen fallback), lowers via a closed type table, synthesises a CPython extension wrapper, and exposes Python items as Mochi `extern fn` declarations.
- **Publish**: `mochi pkg publish --to=pypi`. The bridge lowers the Mochi package via a `TargetPythonPackage` (sdist + wheel), runs the `mochi-build` PEP 517 backend, and uploads through PyPI Trusted Publishing (Sigstore OIDC + PEP 740 attestations).

All 19 umbrella phases (0-18) are LANDED at baseline. The deferred sub-phases (live IO, CLI verbs, live crypto) ship as N.1 / N.2 / N.3 follow-ups; see the [implementation index](docs/implementation/index.md) for status.

## Package layout

```
errors/         SkipReason + BridgeError (phase 0)
build/          Workspace + Driver + Venv + libpython link (phase 0)
semver/         PEP 440 version parser (phase 1)
simple/         PEP 503 / 691 / 700 simple-index client + content-addressed cache (phase 1)
toml/           Minimal TOML reader scoped to uv.lock + pylock.toml + pyproject.toml (phase 2)
uv/             uv subprocess bridge + uv.lock parser + pylock.toml round-trip (phase 2)
stubs/          PEP 561 stub discovery + typeshed pin + stubgen sandbox + .pyi parser (phase 3)
typemap/        Closed type table + Mochi/Python rendering (phase 4)
wrapper/        <pkg>_externs.py + .pyi + _mochi_wrap.py synthesiser (phase 5)
emit/           Mochi shim emitter; <pkg>_shim.mochi with extern python fun + type aliases (phase 6)
importspec/     import python "<spec>" body parser (phase 7)
lockfile/       mochi.lock [[python-package]] + --check diff + capability database (phase 9)
pypackage/      TargetPythonPackage emit: sdist + wheel + mochi_build PEP 517 backend (phase 10)
publish/        PyPI publish + Sigstore + PEP 740 attestations (phase 11)
asyncbridge/    async-fn to sync-fn shim renderer + cross-loop guard (phase 12)
abi3/           abi3 wheel slimming + auditwheel-equivalent platform validator (phase 13)
subproc/        JSON-RPC 2.0 stdio protocol + Python worker source renderer (phase 14)
attest/         Install-time PEP 740 verification + policy (phase 15)
pyodide/        pyodide / emscripten / wasi-p2 platform-tag matcher + WIT emitter (phase 16)
freethread/     cp3XYt ABI matcher + PEP 703 module-marker audit + PyMutex lock shim (phase 17)
abi2026/        2026-Q1 ABI-tag transition: TagClass + Policy + Selector + Promote/Downgrade (phase 18)
```

## Usage

```go
import "github.com/mochilang/mochi-python/abi2026"

result, err := abi2026.Selector{Policy: abi2026.PolicyBoth}.Select(candidates)
```

```go
import "github.com/mochilang/mochi-python/freethread"

score := freethread.WheelCompat{Target: target}.Score(wheel)
```

See per-package GoDoc on [pkg.go.dev](https://pkg.go.dev/github.com/mochilang/mochi-python).

## Development

```
go vet ./...
go build ./...
go test ./... -count=1
```

All test files (~400 cases across 21 packages) are deterministic and run offline. CI runs the same matrix on Ubuntu and macOS with Go 1.26.

## Relationship to mochilang/mochi

This repo is a one-way mirror of `package3/python/` in [mochilang/mochi](https://github.com/mochilang/mochi). The monorepo is authoritative; this repo lets downstream consumers vendor the bridge without pulling in the rest of the Mochi compiler. Phase work happens upstream and is mirrored here.

## License

Apache-2.0.
