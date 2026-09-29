# Doxygen and comments

Document the contract a caller relies on and the reasoning a maintainer cannot
recover from the code alone. Keep comments accurate as behavior changes. Doxygen
describes APIs; ordinary comments explain implementation decisions; `docs/`
describes architecture, workflows, and operation.

## Comment forms

Use `///` immediately before a documented declaration. Begin with an explicit
`@brief` summary sentence, then add details where needed. Use a blank `///` line
between paragraphs. Use `///<` after a field or enumerator for a short member
description. These forms are supported by [Doxygen's comment
syntax](https://www.doxygen.nl/manual/docblocks.html).

Use ordinary `//` comments for implementation rationale and at the top of test
files. An implementation `.cpp` may have a Doxygen file header; keep the public
API contract on the declaration in its `.cppm`. Avoid decorative banners,
commented-out code, and comments that merely repeat an identifier or narrate the
next statement. Let clang-format handle wrapping and alignment using the
repository's configuration.

## Module and API documentation

Every `.cppm` starts with `@file` and `@brief`. Explain the module's purpose,
principal entry points, and error boundary. Keep detailed API contracts next to
their declarations rather than duplicating them in the file introduction.
Document each exported declaration, including types, functions, and constants.
Public members should explain their meaning, units, defaults, and constraints
where these matter.

This example illustrates a documented module interface:

```cpp
/// @file limits.cppm
/// @brief Request limits and validation through validate().
/// Invalid limits are reported as limit_error values.
export module sample.limits;

import std;

/// @brief Reasons request limits cannot be used.
export enum class limit_error {
  zero_capacity, ///< No requests would be admitted.
};

/// @brief Limits applied to one service instance.
export struct limits {
  std::size_t max_requests = 32; ///< Maximum simultaneous requests; must be nonzero.
};

/// @brief Check whether request limits can be applied.
/// @param value Limits to validate; the call neither modifies nor retains them.
/// @return Success, or limit_error::zero_capacity when max_requests is zero.
export auto validate(const limits& value) -> std::expected<void, limit_error>;
```

For functions, explain meaningful parameter constraints and result semantics.
Avoid filler such as “the value” or “returns the result.” Keep tag names in sync
with the signature. Include parameter and return documentation required by the
lint profile even when the declaration looks simple; a useful description can
state units, valid ranges, ownership, or failure behavior.

Use these tags when they add contract information:

| Tag | Use |
|---|---|
| `@param` | Parameter meaning, constraints, ownership, and direction when relevant |
| `@tparam` | Template parameter requirements beyond an obvious name |
| `@return` | Result meaning, including expected success and failure outcomes |
| `@pre` / `@post` | Actual caller obligations and guaranteed resulting state |
| `@throws` | Exceptions that are part of the documented contract |
| `@note` | A relevant qualification that does not fit the main description |
| `@see` | Related APIs or documentation that clarify the contract |

These commands are described in the [Doxygen command
reference](https://www.doxygen.nl/manual/commands.html). Use `@return` to
document `std::expected` errors, not `@throws`. Invalid user input handled by an
error return is not an unchecked precondition. Do not promise exception freedom,
thread safety, or atomicity without an implementation that provides it.

## Ownership and state

For resource-owning or stateful types, document the facts callers need:

- Whether instances own or borrow resources, and when those resources are released.
- Whether copying or moving is supported and what operations remain valid after a move.
- How long returned references, pointers, spans, and views remain valid.
- Which operations mutate state, and what remains true after failure.
- Whether instances can be shared across threads, and who owns synchronization.
- Whether an operation blocks and how timeout or cancellation affects its result.

Cover relevant facts rather than adding the entire list to every type. Put the
shared type-level contract on the type and operation-specific qualifications on
the corresponding members. Private invariants still deserve comments even when
private declarations are excluded from generated documentation.

## Implementation comments

Explain a non-obvious choice, constraint, ordering requirement, or workaround at
the code it affects. Prefer a concrete reason:

```cpp
// Publish only after the transaction commits, so readers cannot cache a value
// that a rollback would discard.
```

A comment saying “publish the value” would add nothing. If an explanation grows
into an architectural discussion, put it in `docs/` and link to it from a short
local comment. Keep API contracts on their declarations; implementations should
explain how a difficult constraint is upheld without copying the entire API
comment.

For a workaround, name the affected dependency or condition, link the issue or
recorded evidence where available, and state what would permit removal. For
unfinished work, use a specific `TODO` tied to a tracked issue or decision and
describe the remaining action. Do not use comments to imply that missing
behavior already works. Remove obsolete comments and dead code when changing an
implementation; version control preserves history.

## Tests and build files

Test names and BDD steps describe behavior. Add `//` comments for surprising
fixture setup, timing constraints, or the regression a scenario protects. Do not
add Doxygen file headers to `*.t.cpp` or document every assertion as an API.
Apply the same approach to `integration_tests/<domain>/` scenarios.

Use `#` comments in CMake for rationale, option semantics, and toolchain
workarounds. A target name and a straightforward dependency declaration usually
need no narration. Keep user-facing build instructions in the CMake guide or the
consuming project's documentation.

## Documentation lint

Run the checked-in profile from a consuming project's root:

```sh
doxygen Doxyfile.lint
```

The [profile](../Doxyfile.lint) scans `src/` recursively for `.cppm` and `.cpp`,
maps `.cppm` to C++, enables documentation warnings as errors, and keeps HTML
output enabled under ignored `doxygen-lint-output/`. It excludes private and
static extraction and does not enable `EXTRACT_ALL`. Configuration semantics are
documented in the [Doxygen configuration
reference](https://www.doxygen.nl/manual/config.html).

This scan is a lint gate, not a complete audit of exported C++ declarations or
comment truthfulness. Review contracts for accuracy even when lint passes. Keep
the lint gate enabled and fix the relevant comments instead of suppressing
warnings wholesale. If source layout changes, update `INPUT` and exclusions
explicitly; root-level integration tests are outside this profile's scan.

Run the project formatter and documentation lint when changing API comments. For
changes to the Doxygen profile itself, check both a valid documented API and a
deliberately malformed comment so a silently weakened gate is noticed. A project
publishing generated API docs should also inspect the rendered result; this
repository supplies the shared conventions and lint profile.
