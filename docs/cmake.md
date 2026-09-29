# CMake conventions

CMake owns the build graph: targets, dependencies, generated sources, tests, and
installation. Presets select supported configurations. The Makefile provides
convenient entry points that invoke those configurations.

## File organization

```text
CMakeLists.txt                      # project setup and composition
CMakePresets.json                   # shared configure/build/test presets
cmake/
├── CPM.cmake                       # pinned package-manager script
├── llvm-toolchain.cmake            # compiler and libc++ selection
├── dependencies.cmake              # pinned application dependencies
├── targets.cmake                   # first-party target helpers and policy
└── architecture.cmake              # dependency-boundary checks
src/
├── CMakeLists.txt                  # compose libraries, domains, executables
├── libs/<library>/CMakeLists.txt
├── <domain>/CMakeLists.txt
├── cmds/<command>/CMakeLists.txt
├── services/<service>/CMakeLists.txt
└── tools/<tool>/CMakeLists.txt      # when development tools are present
integration_tests/
├── CMakeLists.txt                  # compose integration suites
└── <domain>/CMakeLists.txt
```

Helper filenames are descriptive conventions; split them further when their
responsibilities warrant it. Each directory declares its own targets or composes
child directories with `add_subdirectory()`. Keep the root file small and avoid
collecting every source file into one global list.

The root establishes the supported CMake version and policies, rejects in-source
builds, enables languages, loads dependency and target helpers, enables testing,
adds source and integration-test directories, then validates the completed
architecture graph. Include optional directories only when the corresponding
feature is enabled.

## Configure and presets

Declare a tested minimum with `cmake_minimum_required()`. Enable only required
languages in `project()`; include C when vendored C dependencies require it.
Select the toolchain before language detection. Where required by the chosen
CMake release, enable its matching experimental `import std` gate before
`project()` enables C++. Keep the supported gate/version mapping explicit.

Commit a `CMakePresets.json` with a hidden base preset specifying Ninja,
`build/${presetName}`, the toolchain file, and compilation-database export.
Derive `debug` and `release` configurations, with corresponding build and test
presets. Use Debug with first-party warnings-as-errors and RelWithDebInfo for
release. Keep developer-specific paths in ignored `CMakeUserPresets.json` or
explicit cache overrides. CMake distinguishes shared and user preset files and
supports inheritance between presets. [CMake presets
reference](https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html).

A project's documented routine should work through those presets:

```sh
cmake --preset debug
cmake --build --preset debug
ctest --preset debug --output-on-failure
```

Reconfigure after changing preset options. Use a fresh build directory when
changing compiler or standard-library installations. Never run simultaneous
builds against the same build directory.

## Target policy

Use project-prefixed, `snake_case` target names that make ownership clear, such
as `sample_catalog`, `sample_cmd_catalog`, and `sample_service_catalog`. The
installed executable name can differ from its internal target name.

Centralize first-party target creation and policy in project-owned functions.
Those helpers must register production targets for architecture checks, keep
test dependencies separate, and apply:

- C++26, required standard support, extensions disabled, and module scanning.
- `CXX_MODULE_STD` on first-party targets using `import std;`.
- First-party warning options and the selected warnings-as-errors policy.
- Output locations, test registration, and installation rules as applicable.

Prefer functions to macros so temporary variables remain locally scoped. Use
`target_compile_options()`, `target_compile_definitions()`,
`target_include_directories()`, and `target_link_options()` for target policy.
Do not spread directory-global flags or link directories through the tree.
Toolchain-wide compiler and runtime selection belongs in the toolchain file;
first-party warning policy must not leak into third-party builds.

Define feature switches with project-prefixed `option()` names and document
their defaults and effects. Fail configure with an actionable diagnostic when an
explicitly requested feature lacks a required dependency.

## Modules and source ownership

List production interfaces and implementation sources explicitly. Register
module interfaces in a `CXX_MODULES` file set; keep implementation units
private. Public module interfaces use `PUBLIC` file-set visibility. CMake
manages their scanning and build ordering. [CMake source file
sets](https://cmake.org/cmake/help/latest/command/target_sources.html).

This fragment illustrates the body of a module helper, after it has created
`sample_catalog`. The helper must also apply policy and register the target; the
fragment alone is not a complete build:

```cmake
target_sources(sample_catalog
  PUBLIC
    FILE_SET CXX_MODULES
    BASE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}"
    FILES catalog.cppm
  PRIVATE
    catalog.cpp)

set_target_properties(sample_catalog PROPERTIES
  CXX_STANDARD 26
  CXX_STANDARD_REQUIRED ON
  CXX_EXTENSIONS OFF
  CXX_SCAN_FOR_MODULES ON
  CXX_MODULE_STD ON)
```

An interface-only C++ module can still produce build artifacts; it is not
necessarily a CMake `INTERFACE` library. Do not hand-maintain BMI paths or
compiler module build order. Standard-module support also requires the chosen
toolchain and CMake gate to support it; setting the target property alone is
insufficient. [CMake standard-module
configuration](https://cmake.org/cmake/help/latest/variable/CMAKE_CXX_MODULE_STD.html).

Keep `*.t.cpp` out of production targets. A helper may discover colocated tests
with a narrow `CONFIGURE_DEPENDS` glob, provided discovery is verified and never
silently drops tests. Avoid recursive globs that mix unrelated domains,
production files, fixtures, and generated code into a single target.

## Package management with CPM

Use **CPM.cmake** for C++ package management. Keep package acquisition and
configuration in `cmake/dependencies.cmake`; individual libraries, commands, and
services consume the resulting CMake targets. CPM configures CMake-based
dependencies and makes their targets available to the build. Its implementation
uses FetchContent internally. [CPM
documentation](https://github.com/cpm-cmake/CPM.cmake#usage).

### Bootstrap and declarations

Keep a pinned `cmake/CPM.cmake` in the repository. Record its release and
SHA256; if a bootstrap download is provided, it must use that exact release and
verify the hash. Do not download `latest` during routine configuration.

Include CPM before declaring packages. Use one explicit `CPMAddPackage()` block
per application dependency, with a versioned archive URL and `URL_HASH SHA256`.
The URL and hash identify the accepted source bytes; a version label alone is
not sufficient. Avoid Git shorthand, branches, and alternate package-manager
paths for these dependencies.

This example shows the declaration shape in `cmake/dependencies.cmake`. The
package, options, URL, and hash are placeholders to replace with verified
values; it is not a runnable dependency declaration:

```cmake
set(CPM_SOURCE_CACHE "${PROJECT_SOURCE_DIR}/vendor" CACHE PATH
  "Committed third-party source cache")
include("${CMAKE_CURRENT_LIST_DIR}/CPM.cmake")

CPMAddPackage(
  NAME Example
  VERSION 1.2.3
  URL "https://codeload.github.com/owner/example/tar.gz/refs/tags/v1.2.3"
  URL_HASH "SHA256=<verified-archive-sha256>"
  OPTIONS
    "EXAMPLE_BUILD_TESTS OFF"
    "EXAMPLE_BUILD_EXAMPLES OFF")
```

Set package-specific options deliberately, particularly upstream tests,
examples, tools, and installation behavior. Keep first-party warnings and module
settings out of dependency targets. For packages without a usable CMake build,
use `DOWNLOAD_ONLY YES` and define a small target wrapper over the acquired
sources. CPM exposes `<name>_SOURCE_DIR` for this purpose. [CPM package
arguments](https://github.com/cpm-cmake/CPM.cmake#usage).

### Source caches and reproducibility

Use the committed `vendor/` cache for third-party sources and build them from
source. Configure `CPM_SOURCE_CACHE` before package acquisition. It caches
source trees; it does not replace each preset's build artifacts. [CPM source
caching](https://github.com/cpm-cmake/CPM.cmake#cpm_source_cache).

Keep separately fetched first-party dependencies in ignored `external/`, with
the same archive-and-hash pinning policy. Make that cache routing explicit in
the dependency setup. Document any first-configure network or credential
requirements. Validate offline claims from a fresh build directory using only
the committed sources, rather than relying on a developer's populated cache.

Keep local package substitution out of supported reproducible builds. CPM's
`CPM_USE_LOCAL_PACKAGES` can try `find_package` before fetching sources; leave
it disabled under this policy. Declare necessary platform-library exceptions
explicitly. [CPM local-package
option](https://github.com/cpm-cmake/CPM.cmake#cpm_use_local_packages).

### Consumption and updates

Link the dependency's exported target to the first-party module that owns its
integration, using the appropriate scope below. Keep dependency details behind
that module's API. Do not repeat package declarations in consuming directories
or manually edit cached source trees to fix integration issues.

For an update, change the release URL, version metadata, and verified hash
together; update the committed source cache for third-party packages. Refresh
ignored first-party caches without committing them. Review upstream build-option
and API changes, preserve required license files, then configure a fresh build
and run the affected tests. Review transitive dependencies too: an upstream
download or system-package fallback must not silently evade the project's
acquisition policy. Update CPM itself deliberately and verify the build with its
new pin.

## Dependency scope

Link to targets, rather than hardcoded archive paths or global linker search
paths. Declare application dependencies centrally under the [dependency
acquisition policy](tooling.md#dependencies).

| Scope | Intent |
|---|---|
| `PRIVATE` | Needed by this target's implementation |
| `PUBLIC` | Needed by this target and its consumers |
| `INTERFACE` | Required by consumers, not by this target's own compilation/linking |

Choose scope according to the usage requirements of the API, rather than making
every dependency public to fix a build error. A private static-library
dependency can still be needed at the final link; `PRIVATE` is not a promise
that the dependency disappears from the final executable. [CMake link dependency
semantics](https://cmake.org/cmake/help/latest/command/target_link_libraries.html).

Keep test dependencies on test targets. Validate command, service, domain, and
library boundaries after all relevant targets exist. Check cycles and both
direct and propagated production edges. Test targets may compose modules across
those boundaries; production targets must never depend on test targets or test
support. An architecture checker must account for conditional edges or reject
forms it cannot evaluate reliably.

## Unit and integration test targets

Enable CTest at the root, commonly with `include(CTest)`, and honor
`BUILD_TESTING` when creating test targets. Link C++ tests against the module
under test, declared test-only dependencies, and the vendored Catch2 target. Use
Catch2 discovery to register individual cases. If testing is enabled but the
framework or registration helper is missing, fail configure rather than silently
omitting the suite.

Add `integration_tests/` separately from `src/`, with a child directory per
domain. Build scenario executables as test targets; label their registered cases
`integration` and with their domain. Labels belong to CTest cases, not just the
executable target. Document which presets include stack-dependent suites. The
[full gate](testing.md#gates-that-prove-work-ran) must cover all required
suites, whether run locally or across required CI jobs.

External stacks start during test execution, not configure or compilation. CTest
fixtures can model their lifecycle:

- A setup test with `FIXTURES_SETUP` provisions the stack and waits for readiness.
- Scenario tests declare `FIXTURES_REQUIRED` for that stack.
- A cleanup test with `FIXTURES_CLEANUP` releases resources owned by the run.

CTest orders fixture setup and cleanup, includes necessary fixture tests when
selecting a subset, and prevents dependent tests from running if setup fails.
Cleanup remains scheduled after scenario failures. Fixture ordering does not
isolate concurrent scenario data; the harness must do that. Abruptly terminated
runs also need recoverable cleanup. [CTest
fixtures](https://cmake.org/cmake/help/latest/prop_test/FIXTURES_REQUIRED.html).

Use bounded test timeouts and actionable setup diagnostics. Verify discovery
counts and reject empty selections. Follow the broader [integration testing
conventions](testing.md#integration-testing) for BDD scenarios, isolation, and
resource ownership.

## Generated files and installation

Write generated files under `CMAKE_CURRENT_BINARY_DIR`. Use `configure_file()`
for configure-time substitutions and declared custom commands for build-time
generation. Give custom commands explicit outputs, dependencies, and byproducts
where appropriate, and use `VERBATIM` for argument handling. Attach generated
sources to the owning target so input changes trigger the required work. Avoid
always-running generators that rewrite identical output on every build.

Use `install(TARGETS ...)` for shipped commands and services. Respect the
install prefix and make required runtime resources relocatable. Install neither
tests nor temporary integration stacks by default. Keep build-tree execution and
installed-artifact verification distinct.

## Reviewing a CMake change

Check that a clean preset configures and builds, relevant tests are registered
and run, and an immediate no-change rebuild performs no work. For changes to
installation, test installation into a scratch prefix. For new architecture
rules, verify that an intentionally forbidden dependency is rejected.

Examples in this guide describe a consuming project's build. This conventions
repository supplies documentation and lint configs, not a runnable CMake
scaffold or an implementation of the named target helpers.
