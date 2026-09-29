# C++ style

## Modules and imports

Use named modules for first-party library and domain code, with `import std;`
for the standard library. Do not introduce a parallel first-party header API.
Include C, platform, and third-party headers in a global module fragment when
needed; expose the project's own vocabulary across the module boundary.

```cpp
/// @file transport.cpp
/// @brief Implementation of the project's transport boundary.
module;

#include <curl/curl.h>

module project.transport;

import std;
```

“Modules only” describes the architecture. Executable entry points, Catch2
translation units, and justified test harness headers are practical exceptions.

## Naming

| Kind | Convention | Example |
|---|---|---|
| Types, namespaces, functions, methods | `snake_case` | `transport_error`, `send_request` |
| Variables and parameters | `snake_case` | `request_body` |
| Enum types and enumerators | `snake_case` | `step_result::done` |
| Private/protected data members | Leading underscore | `_timeout` |
| Public data members and struct fields | Bare `snake_case` | `timeout` |
| Template parameters | CamelCase | `ValueType` |

A trailing underscore is an escape for a keyword or a real name collision, not
the member convention: `import_` is an example. The Lua API's `L` is an explicit
recognizable-name exception. Do not use double underscores or other
implementation-reserved identifiers.

The tidy exclusions preserve these exceptions, but their regex also accepts more
spellings than the prose intends. An accepted name is not proof that an
exception is justified.

## Class layout

Declare member variables at the top of the class, before constructors and member
functions. Do not put the data-member section at the bottom. Private and
protected member variables use a leading underscore, such as `int _member;`,
never a trailing underscore.

This example shows layout only; an exported class also needs the API comments
described in [Doxygen and comments](comments.md).

```cpp
class counter {
private:
  int _member = 0;

public:
  counter() = default;

  auto increment() -> void {
    ++_member;
  }

  auto value() const -> int {
    return _member;
  }
};
```

Public data members, including fields in plain data structs, retain bare
`snake_case` as specified above. Member placement is a review convention; the
naming configuration checks prefixes but does not enforce class layout.

## Formatting

[`.clang-format`](../.clang-format) is authoritative for mechanical layout:

- LLVM base style, two-space indentation, 130-column limit.
- Attached opening braces; pointers and references align with the type.
- No single-line function bodies, `if` statements, loops, or blocks. Defaulted
  and deleted function declarations remain single-line declarations.
- Align consecutive assignments, declarations, and macros.
- Reflow comments, while preserving the configured documentation pragmas.
- Use the latest language dialect supported by the selected formatter.

Do not manually maintain a competing formatting style. Use the formatter from
the LLVM installation selected for the build.

**Observed idioms:** trailing return types (`auto read() -> result`), `auto
const` for immutable locals, `enum class`, designated initialization of plain
records, anonymous namespaces for implementation helpers, and RAII wrappers for
C handles. These are useful exemplars, not extra checks secretly enabled by
clang-tidy. Both `const T&` and `T const&` appear in the source; the extracted
config does not impose one const-placement style. Constants sometimes use `k_`;
this is not a required prefix.

## Ownership, state, and errors

Use RAII for owned resources. Use move-only handles, deleted copy operations,
destructor cleanup, and explicit lifetime contracts where they express the
resource semantics. Keep raw third-party handles out of exported function
signatures where a first-party wrapper can express ownership.

Return `std::expected<T, E>` for expected failures at API boundaries. Translate
dependency failures into a module-owned error type. This can be an enum or a
record carrying necessary details, such as a driver code and message. Do not
force every error into an enum and lose information.

Reserve exceptions for unrecoverable conditions. An invariant guard expresses a
condition believed impossible and terminates if it fails. Use a guard that
survives release builds (a project-owned `check` helper), never `assert()` for
such runtime protection. Expected bad input, missing data, and network failures
belong in error returns. `static_assert` remains appropriate.

Avoid global mutable application state. Pass resources and cancellation
explicitly; use signals or atomics where appropriate. Document synchronization
and thread safety. Encapsulated one-time dependency initialization, such as a
wrapper-local `std::once_flag`, belongs inside the dependency wrapper.

## Text and documentation

Append formatted text with `out += std::format(...)`. The reference toolchain
has a recorded issue with `std::format_to(std::back_inserter(out), ...)`;
revalidate that workaround when upgrading. The historical finding is recorded in
[Sources and scope](sources.md).

Follow [Doxygen and comments](comments.md) for module headers, exported API
contracts, ownership and thread-safety documentation, implementation comments,
and lint requirements. Use `///` for API documentation, `///<` for short member
descriptions, and ordinary `//` for implementation and test commentary.

Historical examples and provenance: [source map](sources.md).
