# mochi-python

[![CI](https://github.com/mochilang/mochi-python/actions/workflows/ci.yml/badge.svg)](https://github.com/mochilang/mochi-python/actions/workflows/ci.yml)
[![Go Reference](https://pkg.go.dev/badge/github.com/mochilang/mochi-python.svg)](https://pkg.go.dev/github.com/mochilang/mochi-python)

The Mochi+Python toolchain. Standalone Go module mirroring two surfaces from [mochilang/mochi](https://github.com/mochilang/mochi):

- **Package bridge** (MEP-71): bidirectional bridge between Mochi and the Python package ecosystem. Resolve / ingest PEP 561 stubs / synthesise wrappers / build wheels (PEP 740 attestations); Pyodide / WASI / free-threaded / abi2026 transition. See [MEP-71](docs/mep/mep-0071.md) and [docs/implementation](docs/implementation/index.md).
- **Python transpiler v3** (MEP-51): lowers Mochi source to Python 3.12 modules + sdist / wheel artefacts via the `mochi-build` PEP 517 backend. Lives under [`transpiler3/python/`](transpiler3/python).

## Surfaces

### Package bridge (MEP-71)

Two directions, both deterministic and offline-gated:

- **Consume**: `import python "<package>@<semver>" as <alias>` in Mochi source. The bridge resolves the package via uv, ingests PEP 561 stubs (4-tier precedence: inline `py.typed`, sibling `<name>-stubs`, typeshed, stubgen fallback), lowers via a closed type table, synthesises a CPython extension wrapper, and exposes Python items as Mochi `extern fn` declarations.
- **Publish**: `mochi pkg publish --to=pypi`. The bridge lowers the Mochi package via a `TargetPythonPackage` (sdist + wheel), runs the `mochi-build` PEP 517 backend, and uploads through PyPI Trusted Publishing (Sigstore OIDC + PEP 740 attestations).

All 19 umbrella phases (0-18) are LANDED at baseline. The deferred sub-phases (live IO, CLI verbs, live crypto) ship as N.1 / N.2 / N.3 follow-ups; see the [implementation index](docs/implementation/index.md) for status.

### Transpiler v3 (MEP-51)

[`transpiler3/python/`](transpiler3/python) lowers Mochi to a Python 3.12 source tree + builds a PEP 517 / 518 sdist + wheel. The build pipeline is reproducible (`SOURCE_DATE_EPOCH`-aware) and validated against the [`tests/transpiler3/python/fixtures/`](tests/transpiler3/python/fixtures) corpus (546 files, phase-01-hello through phase-18 inclusive).

Vendored upstream Mochi packages (production sources only; their own test suites live upstream): `ast/`, `diagnostic/`, `parser/`, `types/`, `types/plan/`, `transpiler3/c/aotir/`, `transpiler3/c/lower/`, `runtime/python/` (the bundled `mochi_runtime` Python package).

## Package layout

```
# MEP-71 bridge (package3/python/ upstream)
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

# Transpiler v3 (MEP-51)
transpiler3/python/pysrc/    Python source AST nodes (Module / FunctionDef / Stmt / Expr)
transpiler3/python/lower/    Mochi AST -> Python pysrc lowering
transpiler3/python/emit/     pysrc -> source text rendering (deterministic, gofmt-style)
transpiler3/python/build/    PEP 517 sdist + wheel builder, reproducibility gate
runtime/python/mochi_runtime/ The runtime helpers the emitted source imports

# Vendored upstream Mochi compiler (production sources only)
ast/                         Mochi AST nodes
diagnostic/                  Mochi diagnostic / error formatter
parser/                      Mochi grammar + parser (participle/v2)
types/                       Mochi type checker
types/plan/                  Query plan IR
transpiler3/c/aotir/         Shared AOT IR used by both C and Python lowering
transpiler3/c/lower/         Shared lowering primitives

# Fixtures
tests/transpiler3/python/fixtures/    546 input .mochi files + golden outputs
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

Tests for the MEP-71 bridge packages run fully offline; the transpiler v3 build tests shell out to `python3` (3.12+) for the PEP 517 backend. CI sets up Python 3.12 on both ubuntu-latest and macos-latest.

## Relationship to mochilang/mochi

This repo is a one-way mirror of the Python-relevant subtree of [mochilang/mochi](https://github.com/mochilang/mochi). The monorepo is authoritative; this repo lets downstream consumers vendor the Python toolchain (bridge + transpiler v3) without pulling in the rest of the Mochi compiler (C / Go / JVM / .NET / Kotlin / PHP / Ruby / Swift / Beam / Rust / TypeScript transpilers, the VM, the LSP, etc.). All phase work happens upstream and is mirrored here.

## License

Apache-2.0.
