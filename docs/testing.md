# Testing conventions

Use Catch2 cases in `*.t.cpp` files for C++ tests. Colocate unit and focused
component tests with the code; place system integration scenarios under
`integration_tests/<domain>/` at the repository root. Build tests as separate
executables and discover individual cases through CTest.

Unit, component, and integration describe test scope. Black-box describes how a
test interacts with the subject: through an observable interface rather than
internal implementation calls. An integration test may be black-box or exercise
components in-process. For CLI black-box tests, supply arguments and assert
stdout, stderr, exit status, and resulting state; service tests exercise the
service's request or event contract and observable state.

## Integration testing

Integration tests exercise systems as a whole or substantial combinations of
their components. They verify behavior across boundaries: domains, services,
processes, storage, and other dependencies working together. Some cover an
end-to-end user workflow; others focus on a subsystem or a specific interaction.
An integration test does not have to cover the entire deployed product.

Organize these tests by the domain whose behavior they verify:

```text
integration_tests/
├── CMakeLists.txt              # compose integration suites
├── <domain>/
│   ├── CMakeLists.txt          # C++ scenario targets and test registration
│   ├── <workflow>.t.cpp       # system behavior scenarios
│   └── fixtures/              # domain-specific setup and test data, if needed
└── support/                   # shared harness and stack setup, if needed
```

Usually write scenarios in BDD style: **Given** a meaningful system state,
**When** an actor performs an operation, **Then** observable behavior and
resulting state satisfy the contract. Use scenario names and steps that describe
domain behavior. Catch2's BDD-style cases are suitable for C++; separate feature
files or a particular BDD runner are not required.

For example: given a stored catalog item and a populated cache, when the item is
updated through the service, then a subsequent read returns the updated value.
This exercises service, storage, and cache behavior together without necessarily
involving a user interface or the entire deployment.

### External resources and execution

See [CMake test targets](cmake.md#unit-and-integration-test-targets) for suite
registration, labels, and CTest fixtures for stack setup and cleanup.

Some scenarios run entirely in-process or with local fixtures. Others require a
stack with a database, cache, broker, or other services. Declare each suite's
requirements and provide a repeatable way to start or connect to its test
resources, initialize state, and clean up afterward. Keep stack configuration
and fixture setup with the suite or in shared `integration_tests/support/`.

- Use isolated test resources and data. Keep concurrent runs from sharing
  mutable fixture state, and clean up resources owned by the run on failure.
- Check readiness with bounded waits before exercising the system. Report
  setup failures with enough diagnostics to distinguish them from failed
  behavior assertions.
- Document the command, configuration, and resources needed to run each suite.
  Label integration tests separately so they can be selected deliberately.
- When an integration suite is selected, missing required resources are a
  setup failure, not a silent skip or a successful empty run. If the ordinary
  local test command excludes stack-dependent suites, state that explicitly
  and identify the full gate or CI job that runs them.

## Meaningful scenarios

- Name scenarios for the workflow they exercise.
- A new command or flag needs a realistic workflow test. Existence and valid
  JSON alone do not establish useful behavior.
- For command workflow tests, seed through the public command surface and
  assert observable post-state after mutations. Successful exit alone does
  not prove a mutation occurred. Low-level storage unit tests can operate
  directly on their storage fixture.
- Snapshot values between steps when testing preservation across transitions.
  Avoid rebuilding the expected answer from the implementation under test.
- Test helpers fail loudly with useful diagnostics when their contract cannot
  be fulfilled. Do not return plausible fallback values that hide failures.
- Changes to public output, exit codes, or state transitions update the
  corresponding contract tests and reference docs together.

## Isolation and concurrency

Give each test its own temporary resources and clean them up. Exercise local
HTTP implementations against a loopback fixture server. From-source binaries
must receive a scratch data path and scratch home/config paths whenever they can
touch persistent state. Isolate every write surface before launching the binary.

Keep Catch2 assertions and reporting on the test thread. Worker threads return
or record results; join them before asserting. Audit helpers called from workers
too, since an indirect `REQUIRE` has the same problem as a direct one. A
heuristic lint can help catch violations, but review indirect and cross-file
calls as well.

## Gates that prove work ran

`make test` runs the documented local suite. It may exclude explicitly labeled
stack-dependent integration suites. `make test-all` runs the full required
pre-PR gate, including those integration suites, contract checks, and lint.
Provision required resources first, or have the gate provision them. If CI
splits this work across jobs, all required jobs together must cover the same
gate; a local-only pass must not be reported as a full-gate pass. Tidy remains
advisory; formatting and documentation lint gate.

Compare the cases in test binaries with those registered by CTest. Missing or
duplicate discovery entries can still yield a green run. Verify that a filtered
run matches at least one real case. Expected skips within the selected suite are
zero; documented suite selection is distinct from silently skipping a selected
test. Do not start Catch2 case names with `-`, because CTest passes names as
arguments.

For bug fixes, first create a failing regression test commit whose subject
starts `Red test:`, followed by the fix making that same test pass. Never narrow
a test to the subset of behavior that happens to work.

When using a mutation probe, distinguish a killed mutant from a compile failure,
a no-op mutation, a filter matching no tests, or a mutation that does not affect
executable behavior. Restore the source, rebuild, and prove the test passes
again. These checks make the probe evidence meaningful.

Historical examples and provenance: [source map](sources.md).
