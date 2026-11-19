---
id: rust-testing
version: 1
description: Testing Rust that catches real defects - unit, integration, property-based, regression and failure-path tests, deterministic and honest.
role: reference
kind: code
requires:
variables:
---

# Testing Rust

Background for writing or judging tests in a Rust crate. A test is worth adding when it would fail
for a plausible defect; one that only restates the current implementation is not coverage.

## Where tests go

- **Unit tests** in a `#[cfg(test)] mod tests` beside the code, for private helpers and edge cases.
- **Integration tests** in `tests/`, using only the public API, as a caller would.
- **Doc tests** for examples in public docs; they must compile and assert something.
- Follow the crate's existing layout, fixtures and helpers rather than starting a second style.

## What to cover

**Boundaries, not just the happy path.** Empty input, one element, the maximum, `0`, `u64::MAX`,
negative values, invalid UTF-8, a value one past every limit the code checks.

**Every error variant the code can return.** Assert the variant (`matches!(err, Error::TooLong
{ .. })`), not merely `is_err()`; a test that accepts any error passes when the wrong check fires.

**Round trips and invariants with properties.** If the crate already uses `proptest` or
`quickcheck`, write properties: decode(encode(x)) == x, an operation keeps a structure's invariant,
sorting is idempotent. Do not add the dependency without the objective's say-so. Keep failing seeds
(proptest's `proptest-regressions/` files) so the case replays.

**Regressions first.** For a reported defect: write the test, see it fail on the defective code,
then fix or confirm the fix, then see it pass. Say in your report that you saw it fail.

## Failure paths for systems code

The bugs that cost the most live where things go wrong. Where the code touches them, test:

- missing, truncated, corrupted or wrong-version files and records;
- a write interrupted half way, and what a reopen sees afterwards;
- a rejected transaction or a failed step leaving state unchanged;
- concurrent callers on shared state (with `loom` if the crate already uses it);
- tampered ciphertext, bad signatures and wrong keys being refused. Use the real cryptography; a
  mocked verifier proves nothing about verification.

## Keeping tests trustworthy

- **Deterministic.** No reliance on test order, wall-clock time, real network or `thread::sleep` to
  wait for something; seed any randomness.
- **Isolated.** Use `tempfile` (or the crate's helper) for disk state; tests run in parallel by
  default, so never share a fixed path or port.
- **Strong assertions.** Compare exact values. Never loosen an assertion, add `#[ignore]` or
  `#[should_panic]` without a message to make a test pass.
- **Named for the behaviour**, such as `rejects_truncated_header`, so a failure reads as a sentence.

## Running them

Read the project's CI config for the real commands and features. Typical ones are `cargo test`,
`cargo test --workspace --all-features`, and `cargo test -- --nocapture` to debug one. Run the
targeted test while working and the whole suite before finishing, when `cargo` is installed. Report
each command you ran with its result, and each you could not run with the reason.
