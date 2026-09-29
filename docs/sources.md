# Sources and scope

The initial extraction on 2026-09-29 used the clean local Planar checkout at
commit
[`aa410e57c4d1548ac72789701362119c028b5ab6`](https://github.com/rdrsss/planar/tree/aa410e57c4d1548ac72789701362119c028b5ab6).
The links below identify that revision rather than a moving branch. The evidence
was read locally. Later guidance incorporates explicit conventions specified for
this repository and links to official tool documentation. This page records
historical provenance rather than prescribing current policy.

## Adaptation to the shared convention

The shared layout uses `src/cmds/`, optional `src/services/`, `src/libs/`, and
`src/<domain>/`, as specified for this conventions repository. The source
project's singular `cmd/` and `lib/` directories and intermediate `engine/`
layer are historical implementation choices. They are not the prescribed
directory names here. Domain dependency rules are declared per project; the
source's prohibition on every sibling engine dependency is not a universal
domain rule.

Project-neutral guidance lives in the other documents. This file preserves the
source-specific evidence and qualifications behind that guidance.

The root-level `integration_tests/<domain>/` layout and BDD-style system
integration guidance are explicit conventions specified for this repository.
They supersede the source project's command-local integration test placement.
Member variables at the top of classes is also an explicit convention added
here, rather than a rule inferred from the source snapshot.

## Source map

All paths in this table are relative to that Planar revision.

| Source | What was extracted |
|---|---|
| [CLAUDE.md](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/CLAUDE.md) (`AGENTS.md` links to it) | Explicit source layout, style, error, documentation, dependency, and test rules |
| [.clang-format](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/.clang-format) | Effective formatting configuration, preserved here |
| [.clang-tidy](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/.clang-tidy) | Naming settings and deliberate exceptions, preserved here |
| [CMakeLists.txt](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/CMakeLists.txt), [CMakePresets.json](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/CMakePresets.json) | Standard, generator, presets, import-std version gates, warning scope |
| [cmake/module.cmake](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/cmake/module.cmake) | Module and executable helpers, distinct test dependencies and targets |
| [cmake/architecture.cmake](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/cmake/architecture.cmake) | Downward layering, base-library exception, cycle and reachability checks |
| [cmake/dependencies.cmake](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/cmake/dependencies.cmake) | CPM archive/hash policy, caches, platform TLS exception |
| [cmake/llvm-toolchain.cmake](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/cmake/llvm-toolchain.cmake), [docs/toolchain-parity.md](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/docs/toolchain-parity.md) | Toolchain discovery, version floor, recorded validation and workarounds |
| [Makefile](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/Makefile), [Doxyfile.lint](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/Doxyfile.lint) | Actual gate composition, tool resolution, documentation lint settings |
| [docs/testing.md](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/docs/testing.md) | Test layers, isolated fixtures, registration checks, red-then-green fixes |
| [src/lib/db/db.cppm](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/src/lib/db/db.cppm) | RAII, move-only ownership, module-owned error records, naming discrepancies |
| [src/lib/http/http.cppm](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/src/lib/http/http.cppm), [http.cpp](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/src/lib/http/http.cpp) | API shape, global module fragment, timeout, local cleanup and initialization |
| [src/lib/core/check.cppm](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/src/lib/core/check.cppm) | Runtime invariants that remain active in release builds |
| [scripts/thread-lambda-catch2-lint.py](https://github.com/rdrsss/planar/blob/aa410e57c4d1548ac72789701362119c028b5ab6/scripts/thread-lambda-catch2-lint.py) | Worker-thread assertion rule and documented heuristic limits |

## What carries across projects

The reusable doctrine covers source organization, C++ style, module boundaries,
resource ownership, error handling, reproducible builds, documentation, and test
discipline. Planar's named libraries illustrate how to apply it:

| Planar choice | Transferable convention |
|---|---|
| SQLite behind `db`, one process connection carried in context | Explicit ownership and a narrow storage boundary; the database and connection count depend on the product |
| CLI11 behind `cliapp` | Tokenization separated from owned help, schema, and exit-code contracts; the manual execution parser is a documented exception |
| libcurl behind `http`, with timeouts | Own the transport boundary and bound network operations |
| Glaze for JSON/TOML | JSON for machine data, TOML for operator configuration in projects following this stack |
| spdlog behind `log`, named logger per module | Central logging integration with identifiable module ownership |
| Stable line-oriented text or JSON | Script-facing output is an undecorated contract |

These choices are the reference stack where applicable, not a requirement to add
SQLite, Lua, HTTP, or a CLI to every project. Planar's binaries, database write
surfaces, migration format, claim protocol, generated agent surfaces, and
external planning workflow remain product-specific.

## Evidence boundaries

Explicit rules take priority over incidental code. Observed implementation
idioms are labeled in the style guide and do not become undocumented mandates.
Not every source file was audited for compliance. Existing code can drift from
stated rules, and broad prose can omit practical exceptions.

In particular:

- “Pinned LLVM” currently means a selected coherent installation with a
  checked minimum major, not an enforced exact patch version.
- “Modules only” permits entry points, tests, and justified harness headers.
- “Downward only” includes acyclic dependencies among base libraries.
- “Every dependency vendored” has platform and first-party-cache qualifications.
- “Errors are enums” also includes module-owned error records in working code.
- Naming tidy is advisory; its ignored-name regex is broader than the intended
  keyword/collision exception. It does not enforce the whole style guide.
- The Makefile includes worker-thread lint in the gate even where the prose
  summarizes it only as formatting and Doxygen.

The CMake helpers and scripts have not been copied as generic drop-in tooling:
they contain product-specific names and policy. The configs here are reusable;
the prose records the requirements for integrating the remaining checks. Future
extractions should cite a new source revision and distinguish changes in
doctrine from version-specific workarounds or existing code drift.

## Toolchain and implementation history

At the inspected revision, Planar requires CMake 4.3 or newer, recognizes
import-std gates for 4.3 and 4.4, and refuses unknown newer minors. LLVM
selection enforces a 23+ floor; the source records validation with 23.1.0. These
are historical evidence for validating a coherent toolchain, rather than
permanent version requirements for all consuming projects.

The source prohibits `std::format_to(std::back_inserter(out), ...)` because of a
recorded libc++ stack-buffer problem when an argument's byte length is a nonzero
multiple of 256. Revalidate the workaround on toolchain upgrades.

Some source public fields, including `db_error::code_` and `message_`, retain
suffixes despite the bare-public-member convention. Harness headers exist under
both `src/cmd/` and `src/lib/http/`. These examples qualify source compliance
claims; they do not prescribe a new naming or directory rule.
