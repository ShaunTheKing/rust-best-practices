# Strict house style & contested rules

Most of this comes from Canonical's Rust style guide (an organization's policy, not universal idiom).
Apply only if the user or project explicitly adopts a strict style. Otherwise, mention as optional.

## Canonical-style rules
- Write the crate's `Error` type first. Library: `Error` + `type Result<T>` in `lib.rs`;
  binary: `error.rs`/`result.rs` at the root.
- Errors start with a verb ("cannot ..."); unrecoverable payload in a `String` field named `reason`;
  public `Internal(#[from] InternalError)` variant wrapping private error types.
- Internal-fault panics prefixed `internal error:`.
- Builders: `Foo::builder()`, builder type not publicly constructible, methods take `self` by value.
- Use `Self` in impl blocks, but name the concrete type when constructing associated types.
- Never shadow a type name. Shadowing budget: once per variable for type conversions.
- Struct literals: field order matches declaration; compute all fields in `let`s or none.
- No explicit `drop(x)` — use a `{}` scope; discard with `|_| ()`, errors with `.ok()`.
- Exhaustively destructure structs in `Ord`/`Eq` impls (`let Self { a, b, c: _ } = self;`).
- `|x| *x` instead of `|&x| x`. No pattern matching in `fn` parameters.
- Pattern bindings reuse the original name or its first letter.
- Lifetime names describe the referenced thing (`'cursor`, `'tree`), not `'a` — **contested**:
  most of the ecosystem and compiler suggestions use `'a`.
- `foo/mod.rs` over `foo.rs` + `foo/` — **contested**: modern Rust mostly uses `foo.rs`.
- `mod.rs`/`lib.rs` contain only declarations and re-exports.
- Never `use Enum::*`; alias long enums locally.
- One `#[derive]`, `Copy` first then alphabetical. Struct fields ordered `pub` → `pub(crate)` → private.
- `impl Type` directly below the type; trait impls in the file of the trait or the type.
- Lowercase hex literals. Expressions returning `()` end with `;`. `=> {}` not `=> ()`.
- Unused params in default trait impls: `let _ = param;` rather than `_param` — **contested**.
- Don't call methods on an expression ending in `}`; bind it first.
- First doc sentence ≤ 2 lines and says *when* to use the item.
- Don't leak `serde` into core types; use local (de)serialization mirror structs.
- Prefer `.collect()` over `FromIterator::from_iter`.

## Where the sources disagree
| Topic | Positions |
|---|---|
| `unwrap` vs `expect` | `expect("reason")` (reason reaches logs) vs `unwrap` + comment vs never in prod. Tests: `unwrap` preferred (better trace than `?`). |
| `anyhow` | Fine in binaries; Canonical bans type-erased errors in libraries outright. Some prefer `thiserror` even in apps. |
| Clone while learning | Reddit: clone first, refine later. Gist: never clone to silence the borrow checker. Middle: clone in drafts, flag it, refine. |
| Iterators vs `for` | Same machine code; pick clearer. Clippy pedantic sometimes pushes the less readable form. |
| Clippy pedantic | Some teams gate on it; it's noisy — curate per module rather than blanket-deny. |
| Imports | Canonical: no globs, nested `use`. Some teams use full paths, no `use`, so `tokio::sync::Mutex` vs `std::sync::Mutex` is visible. |
| Async | "Use threads if you can" vs async for I/O-heavy servers. Tokio work-stealing vs thread-per-core runtimes. |
| Output-buffer APIs | Great for hot paths; poor ergonomics for general libraries. |

## Sources
- Canonical Rust Best Practices — github.com/canonical/rust-best-practices
- Rust best practices gist (allocation discipline) — gist.github.com/auser/c3161f55a8393faa8af5ddda68c6befa
- r/rust "What are some Rust practices you learned in a team" (read via archive snapshot)
- users.rust-lang.org "Best practices to write Rust code" (module layout) → defers to
  Rust API Guidelines: rust-lang.github.io/api-guidelines
