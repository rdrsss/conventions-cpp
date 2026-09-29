# Build and tooling

See [CMake conventions](cmake.md) for build-file organization, target and module
declarations, dependency scope, test registration, generation, and installation.

## Toolchain policy

Use C++26, Clang/LLVM, libc++, CMake, and Ninja. Select a coherent installation:
compiler, standard-library headers and libraries, module metadata, formatter,
and tidy must agree. The platform baseline is macOS and Linux.

Record the versions a project actually validates. Recheck module support and
formatter output on upgrades. CMake experimental gate identifiers belong in the
consuming build's version table; copying one UUID forever is insufficient.
Resolve libc++ module metadata and runtime paths from the selected installation
instead of hardcoding a particular Homebrew prefix.

## Build organization

- Reject in-source builds; use `build/<preset>/`.
- Use a hidden base preset with the Ninja generator, toolchain file, and
  compilation database. Derive `debug` and `release` presets from it.
- Use Debug with first-party warnings-as-errors for development. Use
  a release preset with `RelWithDebInfo`, retaining debug information.
- Require the C++ standard, disable language extensions, and enable module
  scanning. Opt into `CXX_MODULE_STD` per first-party target, never globally
  across vendored targets.
- Scope warnings-as-errors and narrowly justified diagnostic suppressions to
  first-party targets. For example, scope an embedding-related warning
  workaround only to targets that embed resources.
- Export `compile_commands.json`. Build module artifacts before clang-tidy:
  a compilation database alone does not materialize imported modules.
- Keep live Git SHA/dirty metadata opt-in for install builds. Routine commits
  should not invalidate the module graph merely to change version text.
- A no-change build should perform no work. Never build concurrently in the
  same build directory; investigate repeated full rebuilds as a defect.

Expose stable convenience commands through a Makefile wrapping CMake presets:
`build`, `test`, `test-all`, `fmt`, `fmt-check`, `cpp-lint`, and `help`. The
consuming project supplies these targets; this reference does not.

## Dependencies

Use CPM.cmake for C++ package management. The
[CPM guide](cmake.md#package-management-with-cpm) is the authoritative
package policy: central declarations, versioned archive URLs and SHA256 hashes,
a pinned CPM bootstrap, and builds from source. Third-party source caches are
committed under `vendor/`; separately fetched first-party caches live in ignored
`external/`. Document platform-library exceptions and any network requirements.

## Integration

The [Doxygen and comments guide](comments.md) defines what to document and how
to write and validate comments with the shared lint profile.

The root [format](../.clang-format) and [tidy](../.clang-tidy) files define the
shared style settings. Tidy enables only `readability-identifier-naming`; it is
**advisory** and has no `WarningsAsErrors` setting. The written style remains a
review requirement.

[Doxyfile.lint](../Doxyfile.lint) preserves the important source-lint settings:
recursive `src/` scanning, `.cppm` mapping to C++, documentation warnings as
errors, and `export=` preprocessing. HTML generation remains enabled so Doxygen
performs the documentation checks; its output goes to the disposable
`doxygen-lint-output/` directory. Other unlisted options use the installed
Doxygen defaults.

When integrating:

1. Resolve `clang-format` and `clang-tidy` from the compiler installation or an
   explicit override. Fail if missing; do not silently choose a different
   LLVM from `PATH`.
2. Run the formatter on first-party sources with `--dry-run --Werror` as a
   gate; use `-i` for the explicit formatting command. Include `src/`,
   `integration_tests/`, owned harness headers, and standalone C++ probes.
   Exclude vendored code and generated outputs.
3. Configure and build the chosen preset before running tidy with
   `-p build/<preset>`. Tidy only sources represented by that build database,
   including configured integration tests; standalone toolchain probes may
   need separate handling.
4. Run `doxygen Doxyfile.lint` from the project root as a gate. Change `INPUT`
   and exclusions if the project uses another source layout.
5. Compose the project's full gate from tests, discovery checks, formatting,
   documentation lint, and applicable contract checks, following the
   [full-gate requirements](testing.md#gates-that-prove-work-ran). Keep advisory
   tidy separate from the required gate.

Add checks for Catch2 assertions reached from worker threads where concurrent
tests are used. The three configs here do not implement that check or the
architecture checker. See [testing](testing.md) for the rules and
[sources](sources.md) for reference implementations and toolchain history.
