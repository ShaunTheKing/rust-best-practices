---
name: rust-best-practices
description: Rules for writing and reviewing idiomatic, production-quality Rust (error handling, ownership/borrowing, API and type design, modules, testing, clippy/fmt, unsafe). Use whenever writing, refactoring, or reviewing Rust code, or when the user runs /rust-best-practices or asks "is this idiomatic Rust?".
---

# Rust best practices

Apply these when writing or reviewing Rust. The user is **learning Rust** (knows C++ and Python):
- Follow CLAUDE.md: **ask before writing Rust** (write + explain, or guide them).
- When a rule changes their code, name the rule and give the one-line *why*.

Target (optional): $ARGUMENTS — a file/dir to review, or a task. For a review, report
findings as `file:line — rule — why — suggested fix`, most important first.

## 1. Errors and panics
- `Option` for absence, `Result` for recoverable failure. Never encode absence as `""`, `0`, `-1`.
- Propagate with `?`. Log only at the layer that can act — don't log and return `()`.
- **Libraries:** typed error enums (`thiserror`), wrapped inner error in a `source` field,
  never expose a dependency's error type publicly. **Binaries:** `anyhow` is fine.
- Convert foreign errors early, at the failing call (`.map_err(MyError::Io)?`).
- Error messages: lowercase, no trailing period, consistent form (`"cannot open {path}"`).
- Panic only for programmer bugs, **never on user input**. Document library panics under `# Panics`.
- No bare `.unwrap()` in production code. Prefer, in order: make it impossible via types →
  `?` / `ok_or` → `let ... else` / `if let` → `.expect("why this can't fail")`.
  `unwrap()` is fine in tests and on compile-time constants (`Regex::new("...")`).
- Never `if x.is_some() { x.unwrap() }` — pattern-match instead.
- Binaries: `fn main() -> anyhow::Result<()>` (or `Result<(), Box<dyn Error>>`) so `?` works in `main`.

## 2. Ownership and borrowing
- Borrow by default; take ownership only when you need it.
  Params: `&str` not `&String`, `&[T]` not `&Vec<T>`, `&Path` not `&PathBuf`.
- For owned-string params, `impl Into<String>` is ergonomic.
- `.clone()` to get unstuck is OK **while learning/prototyping**, but flag every one with
  why it was needed — then refine. Never clone without understanding the ownership problem.
- Naming convention: `as_*` cheap borrow, `to_*` may allocate, `into_*` consumes `self`.
- Self-referential struct? Split into owner + borrowing view. Never transmute a lifetime to `'static`.
- Prefer indices/offsets into a `Vec` over pointer-heavy graphs or `Rc<RefCell<_>>` webs.
  (`Rc<RefCell<T>>` is acceptable for prototypes; encapsulate it behind a struct's methods.)
- `Arc<Mutex<T>>` everywhere signals unclear ownership — consider one owner + channels.
- Scope mutability: `let x = { let mut x = ..; ..; x };` then `x` is immutable.
- **Structs own their data; functions borrow.** Store `String`/`Vec<T>`/`PathBuf` in structs,
  take `&str`/`&[T]`/`&Path` in parameters. A struct holding a reference (`struct Foo<'a>`) is
  the classic beginner lifetime trap — only do it for short-lived views.
- Integer overflow: debug builds panic, release builds silently wrap. Where overflow is
  possible, choose explicitly: `checked_*`, `saturating_*`, or `wrapping_*`.

## 3. Types and API design
- **Newtypes** for IDs, units, offsets (`struct UserId(u32)`) — stops mixing look-alike numbers.
- **Enums over booleans/sentinels:** not `fn run(strict: bool, retry: bool)`.
- "Parse, don't validate": private fields + constructor that validates once; the type is the proof.
- Object-like types: constructor or builder (`Foo::builder()...build() -> Result<Foo>`),
  not public fields. Config/DTO with sane `Default`: public fields + `#[non_exhaustive]`.
- Start with a function; add a struct/trait only when there's a real need
  (don't create a trait just because two APIs look alike).
- Keep hard features (HRTBs, complex lifetimes, heavy generics) inside a library behind a
  simple API — don't leak them across the codebase.
- Hide generic noise: `impl Trait` args, `'_` elided lifetimes, `where` clauses when long.
- Write `MyRef<'_>`, never bare `MyRef` for a type that borrows.
- `dyn Trait` is fine outside hot paths; prefer enums when the set of variants is closed.
- Narrow visibility: private → `pub(crate)` → `pub` only for real consumers.
- Derive `Debug` on all types; add `Clone`, `PartialEq`, `Default` etc. where they make sense.
- Public items get `///` docs: what/when to use, plus `# Errors`, `# Panics`, and a runnable `# Examples`.

## 4. Idioms and readability
- Use `let ... else` for "get it or bail". Use `match` when you need all cases.
- Iterators when they read clearly; a plain `for` loop when it's clearer (multiple
  accumulators, early exits). Don't write `map(..).for_each(..)`.
- Return values from blocks instead of `let x;` + later assignment.
- Destructure tuples into named bindings rather than `.0`, `.1` (except newtypes).
- Inline format args: `format!("{path}/{file}")`.
- Comments explain **why**, not what. `// TODO(#issue)` — no ownerless TODOs.
- Prefer functions over macros; write a macro only when a function can't do it.

## 5. Modules and project layout
- Organize by **feature/responsibility**, not by syntax (no `types.rs` + `impl.rs` split).
- Keep a type and its `impl` blocks in the same file.
- Start with one module; split when real boundaries appear. Keep each module's public surface small.
- No glob imports (`use foo::*`) except `use super::*` in test modules.
- Import groups: `std` → external crates → `crate`/`super`.
- Keep `main.rs` thin; logic lives in `lib.rs` so it's testable.

## 6. Testing
- Unit tests in `#[cfg(test)] mod tests` beside the code; integration tests in `tests/`.
- Test names state condition + result: `parse_fails_when_input_is_empty`.
- Assert on error **variants**, not strings. Test the edge cases: empty, one, max, malformed.
- Doc examples on public items are tests — keep them compiling.
- Types don't replace tests. Consider `proptest` (properties) and `insta` (snapshots).

## 7. Tooling (definition of done for Rust)
Run and show output, per CLAUDE.md:
```
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```
- Fix warnings; don't silence them. If you must, use the narrowest `#[expect(lint, reason = "...")]`
  / `#[allow]` on the item — never crate-wide for one call site.
- Feature flags? Also `cargo clippy --all-features`. Clippy `--all-targets` ≠ all OSes.
- Apps: commit `Cargo.lock`. Add `cargo audit` in CI; be choosy about new dependencies.

## 8. Unsafe
- Avoid it. "It's faster" isn't a reason without profiling evidence.
- If needed: smallest possible block, safe public wrapper, and a `// SAFETY:` comment
  stating every precondition. Never use `debug_assert!` as the safety proof. Test with Miri.

## Gotchas (where Claude usually goes wrong)
- Reaching for `.clone()`, `.unwrap()`, or `Rc<RefCell>` to silence the compiler without
  explaining the underlying ownership issue to the user.
- Adding explicit lifetimes when restructuring ownership is the real fix.
- Inventing crate APIs or versions — check with `cargo doc`/docs.rs or compile before claiming.
- `debug_assert!` used as validation: it vanishes in release builds. Validate inputs in all builds.
- Blocking calls (`std::fs`, `thread::sleep`, heavy CPU) inside `async` — compiler won't warn.
- Over-engineering: traits, generics, and builders for code with one caller.
- Applying the opinionated or performance rules (references/) to beginner code.

## References — read only when relevant
- `references/performance-and-advanced.md` — allocation discipline, hot paths, async,
  concurrency, unsafe/FFI, CI lint tables. Use for performance-sensitive or systems code.
- `references/style-opinions.md` — strict house-style rules (Canonical guide) and where
  sources disagree. Use only if the user/project adopts a strict style.
