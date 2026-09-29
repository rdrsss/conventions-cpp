# C++ conventions

The doctrine for how I organize C++ projects, write C++, and build and verify
the result. This repository is the reusable reference for those choices across
projects.

## Doctrine

- Use modern C++ with named modules and `import std;`. The baseline is C++26.
- Make architecture visible in directories, module names, and build targets.
  Dependencies point downward; shared vocabulary belongs in a lower layer.
- Keep executable entry points thin. Put product behavior in
  domain modules and reusable infrastructure in library modules.
- Own resources explicitly. Represent expected failures in return values,
  and preserve invariant checks in release builds.
- Use one consistent style, mechanically formatted with the same LLVM
  toolchain that builds the project.
- Treat public contracts, documentation, and realistic tests as part of
  delivering behavior. A green command must prove the intended work ran.
- Keep dependencies reproducible and build policy explicit. Avoid ambient
  machine configuration deciding what the project means.

## Source hierarchy

```text
src/
├── cmds/<command>/       # command entry points and composition
├── services/<service>/   # service entry points and lifecycle, when needed
├── libs/<library>/       # reusable, domain-independent facilities
├── <domain>/             # project-specific behavior, organized by domain name
└── tools/<tool>/         # optional development tools
```

Keep unit and focused component tests beside their modules as `*.t.cpp`. Place
system integration scenarios in root-level `integration_tests/<domain>/`, with
external test stacks where needed. See [integration
testing](docs/testing.md#integration-testing). The [full directory
map](docs/organization.md#directory-hierarchy) covers module files, build
configuration, documentation, and dependency caches.

## Reference

| Document | Covers |
|---|---|
| [Project organization](docs/organization.md) | Directories, module boundaries, dependency direction, build targets |
| [C++ style](docs/cpp-style.md) | Naming, formatting, modules, ownership, errors, documentation |
| [Doxygen and comments](docs/comments.md) | API contracts, comment forms, ownership documentation, implementation rationale, documentation lint |
| [Build and tooling](docs/tooling.md) | LLVM, CMake, dependency acquisition, lint configurations |
| [CMake conventions](docs/cmake.md) | Build-file hierarchy, presets, modules, CPM package management, dependency scope, tests, installation |
| [Testing](docs/testing.md) | Colocated tests, BDD integration scenarios, external test stacks, isolation, meaningful gates |
| [Sources and scope](docs/sources.md) | Source revision, evidence, exceptions, observed patterns |

## Reusable configurations

Copy [.clang-format](.clang-format), [.clang-tidy](.clang-tidy), and
[Doxyfile.lint](Doxyfile.lint) into a project's root. The Clang configs apply to
first-party C++, including integration tests. The Doxygen profile scans API
documentation under `src/`.

These files configure tools; they do not install tools or create build and CI
targets. Follow [the integration requirements](docs/tooling.md#integration) when
adopting them. This conventions repository has no application build.

Explicit rules are stated as rules. Code patterns that are not established
policy are marked **observed**. Historical source details and project-specific
exceptions live in [Sources and scope](docs/sources.md).

Each guide owns its subject: organization defines placement and dependency
boundaries; C++ style defines language and naming rules; comments defines
documentation contracts; CMake defines build and package policy; testing defines
scenario and gate requirements. Tooling explains how to run the checks.
Configurations govern the mechanical checks they actually implement; rules such
as member placement and comment accuracy also require review. Historical
examples in the source notes do not override these conventions.
