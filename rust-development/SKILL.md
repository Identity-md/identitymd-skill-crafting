---
id: rust-development
version: 1
description: Idiomatic, safe Rust for applications, libraries, CLIs, services and systems code - ownership, errors, APIs, async, unsafe and Cargo practice.
role: reference
kind: code
requires:
variables:
---

# Writing Rust

Background for work on a Rust crate or workspace. It does not change your objective, your budget or
your judge; it is what an experienced Rust engineer would tell you before you start.

## Before you write

**Read the manifest first.** `Cargo.toml` (and the workspace root's, if there is one) tells you the
edition, `rust-version` (the MSRV), features, and which crates exist. Do not use a language or
library feature newer than the MSRV, and do not bump it, the edition or a dependency's major version
unless the objective asks.

**Copy the house style.** Look at how the crate already returns errors (`thiserror`, `anyhow`, a
hand-written enum), which async runtime it uses, how modules are laid out, and how tests are named.
A second error type or a second runtime is a defect even when it compiles.

**Do not add dependencies to save a few lines.** Every new crate is code a stranger now trusts. Use
what is already in `Cargo.lock`; add one only when the objective needs it, and say why.

## Ownership and types

- **Borrow unless you must own.** Take `&str`, `&[T]` and `&Path` in arguments; return owned values.
  A `.clone()` added to make the borrow checker quiet usually means the ownership design is wrong.
- **Never add `'static`, `Box::leak`, `Rc<RefCell<_>>` or `Arc<Mutex<_>>` just to silence the
  compiler.** Use shared ownership only when the design actually shares.
- **Make invalid states unrepresentable.** Enums for exclusive states, newtypes for ids and units,
  constructors that validate. A `bool` pair that can be "both true" is a bug waiting.
- **Keep public APIs small.** `pub(crate)` by default. Do not change a public signature, trait or
  serialized format unless the objective asks; that breaks every caller you cannot see.

## Errors

- **`Result` for anything that can fail at runtime; `?` to propagate.** Add context where the cause
  would otherwise be lost (which file, which key).
- **No `unwrap()` or `expect()` on a production path** unless an invariant makes failure impossible,
  and then `expect("why this cannot fail")` states it. Tests may unwrap.
- **Never turn an error into a default.** `unwrap_or_default()` on a parse or I/O result hides
  corruption. `let _ =` on a `Result` needs a comment saying why ignoring it is correct.
- **No panics across an FFI boundary**, and none in library code for bad input.

## Integers and input

- **Untrusted lengths, offsets and counts are hostile.** Use `checked_*` / `saturating_*` where
  overflow is possible, `try_from` instead of `as` for narrowing casts, and bound any allocation
  sized by input.
- **Index with `get` when the index comes from input.** A slice panic on a malformed file is a
  denial of service.

## Concurrency and async

- **Do not hold a `std::sync` lock across `.await`**, and prefer not to hold any lock across one.
- **Do not block the runtime.** File I/O, heavy CPU and `std::thread::sleep` in async code go through
  `spawn_blocking` or the runtime's own APIs.
- **Know what cancellation does.** A dropped future stops at its last `.await`; state half-updated
  at that point must still be valid.
- **Use the weakest atomic ordering you can justify in a comment**, and `SeqCst` when you cannot.

## Unsafe

- **Avoid it when a safe alternative exists**, including a small cost in speed.
- **When needed, keep the block tiny and write a `// SAFETY:` comment** naming each invariant:
  pointer validity, alignment, initialization, aliasing, lifetime, and thread-safety for
  `Send`/`Sync` impls.
- **Never use `unsafe` to get past the borrow checker.** That is the borrow checker being right.

## Storage, crypto and secrets

- **Assume a crash between any two writes.** Write to a temporary file, `sync_all`, then rename;
  keep metadata consistent if the process stops mid-operation.
- **Use established crates (`ring`, RustCrypto, `rustls`); never write a primitive or protocol.**
  Compare secrets in constant time. Do not weaken a verification to make a test pass.
- **Keep secrets out of `Debug`, logs and error messages.** Derive `Debug` by hand, or use a
  redacting wrapper, on types that hold keys or tokens.

## Checking your work

If `cargo` is on the machine, run `cargo build`, `cargo test` and `cargo clippy --all-targets` with
the features the project uses (read its CI config for the real commands), and fix warnings you
introduced. If it is not installed, or a command cannot run, say so in your final message. Never
report a build or test result you did not see.
