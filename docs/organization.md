# Project organization

## Directory hierarchy

Organize first-party code under `src/` by responsibility. Commands live in
`src/cmds/`, services in `src/services/`, reusable libraries in `src/libs/`, and
project domains directly under `src/<domain>/`. Use the domain's actual name,
such as `catalog`, `rendering`, or `scheduling`. Create only the directories the
project needs.

```text
.
├── CMakeLists.txt
├── CMakePresets.json
├── Makefile
├── .clang-format
├── .clang-tidy
├── Doxyfile.lint
├── cmake/                          # toolchain, dependencies, target helpers
├── src/
│   ├── CMakeLists.txt              # compose first-party targets
│   ├── cmds/                       # executable entry points and composition
│   │   └── <command>/
│   │       ├── CMakeLists.txt
│   │       ├── main.cpp
│   │       ├── handlers/           # command families, when needed
│   │       ├── internal/           # invocation context and private support
│   │       └── <scenario>.t.cpp
│   ├── services/                   # service executables, when needed
│   │   └── <service>/
│   │       ├── CMakeLists.txt
│   │       ├── main.cpp
│   │       ├── handlers/           # request or event adapters, when needed
│   │       ├── internal/           # lifecycle and private support
│   │       └── <scenario>.t.cpp
│   ├── libs/                       # reusable, domain-independent facilities
│   │   └── <library>/
│   │       ├── CMakeLists.txt
│   │       ├── <library>.cppm
│   │       ├── <library>.cpp
│   │       └── <library>.t.cpp
│   ├── <domain>/                   # behavior specific to this project
│   │   ├── CMakeLists.txt
│   │   └── <module>/
│   │       ├── CMakeLists.txt
│   │       ├── <module>.cppm
│   │       ├── <module>.cpp
│   │       └── <module>.t.cpp
│   └── tools/                      # optional project-development tools
│       └── <tool>/
├── integration_tests/              # system integration and BDD scenarios
│   ├── CMakeLists.txt              # compose integration suites
│   ├── <domain>/                   # workflows, fixtures, test registration
│   └── support/                    # optional shared harness and stack setup
├── docs/                           # maintained reference documentation
├── scripts/                        # build, validation, maintenance scripts
├── vendor/                         # committed third-party source cache
├── external/                       # optional ignored first-party cache
└── build/<preset>/                 # ignored, out-of-source build products
```

The angle-bracket names are placeholders. Each domain is its own directory
directly under `src/`. A small domain can keep its module files directly in
`src/<domain>/`. Add subdirectories as distinct responsibilities emerge.
Interface-only modules need no implementation file.

## Where code belongs

| Location | Responsibility |
|---|---|
| `src/cmds/<command>/` | Parse inputs, assemble dependencies, call domain/library APIs, render results, map exit codes |
| `src/services/<service>/` | Assemble dependencies, manage startup and shutdown, adapt requests or events to domain/library APIs |
| `src/libs/<library>/` | Reusable facilities such as HTTP, logging, process execution, or storage wrappers |
| `src/<domain>/` | Product concepts, rules, operations, and state transitions |
| `src/tools/<tool>/` | Development tooling used to build, validate, or maintain the project |
| `integration_tests/<domain>/` | System integration scenarios, including end-to-end workflows where appropriate |

Keep executable entry points thin. A feature belongs in a domain when it
expresses product behavior, even if several commands or services use it. Move it
into `libs/` when it is a reusable facility with no dependency on product rules.

Add `src/services/<service>/` when the project ships a service. Keep service
entry points thin, with product rules in domain modules and reusable transport
or runtime facilities in libraries.

Unit and focused component tests live beside the code as `*.t.cpp`. System
integration scenarios live at the repository root in
`integration_tests/<domain>/`. They may span commands, services, and external
resources such as a database or cache; see [integration
testing](testing.md#integration-testing). Keep generated files in the build
tree. Product data such as migrations and templates may have root directories
when needed.

## Dependency direction

```text
src/cmds/<command> ────┐
                      ├→ src/<domain> → src/libs/<library> → third-party dependencies
src/services/<service>┘
```

Commands and services may also use libraries directly. Libraries may depend on
other libraries, provided the graph remains acyclic; they must not depend on
commands, services, or product domains. Command and service executables share
code through domain or library modules, not dependencies on one another.

Define allowed dependencies between domains explicitly for each project. Keep
independent domains independent. When domains share a reusable primitive, place
that primitive in `libs/`; keep shared product concepts in an explicitly owned
domain module. Dependencies must remain acyclic and respect the declared
boundaries.

Enforce the declared graph at CMake configure time. Register every first-party
production module, command, service, and tool target with the architecture
checker through common helpers. Check implementation and interface link
dependencies, and detect cycles explicitly. Product-specific forbidden
dependencies may need transitive checks.

## Module and target shape

The [CMake guide](cmake.md) describes how to express this structure in the
build.

- Give a module a `.cppm` interface and `.cpp` implementation units as needed.
- Use a project prefix and dotted module path, such as `project.http` for a
  library or `project.catalog.item` for a domain module. Use nested C++
  namespaces for the same conceptual ownership. Structural directory names
  such as `libs` need not appear in the public module name.
- Keep each module's `CMakeLists.txt` small. Declare interfaces, implementation
  sources, production dependencies, and test-only dependencies explicitly.
- Centralize target policy in project-owned module and executable helpers.
  Every production target must participate in architecture enforcement.
- Register interfaces with CMake's `FILE_SET CXX_MODULES`. Set standard-module
  support and warning policy on first-party targets.
- Build a separate test executable for each module's colocated tests.
  Test-only dependencies must not create production edges.
- Command and service tests can compile implementation sources with Catch2's entry
  point, excluding the application's `main`. Identify that entry-point file
  explicitly in the build helper.

The dependency diagram describes production code. Development tools follow the
same dependency direction as other executables. Test targets may compose the
modules required by a scenario across production boundaries, but production
targets must never depend on test targets or test support. Keep test edges
separate when checking the production graph.

## Boundaries

Keep command parsing, presentation, and service lifecycle separate from domain
behavior. Wrap third-party APIs behind first-party modules. Carry process
resources through explicit context objects rather than global mutable
application state.

Reference documentation belongs in `docs/` and changes alongside behavior or
architecture. Source provenance and adaptation notes live in the [source
map](sources.md).
