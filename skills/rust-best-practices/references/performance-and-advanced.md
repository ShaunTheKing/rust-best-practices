# Performance, systems & advanced Rust

Use only for performance-sensitive, embedded, parser/protocol, or systems code.
Measure first: readability beats unmeasured speed.

## Allocation discipline
- Decide a memory profile per module up front: **strict heapless** (no heap, `no_std`),
  **allocation-free steady state** (allocate at init only), or **allocation-conscious**
  (allocate at explicit boundaries). It shapes the API, so choose before profiling.
- Make allocation visible in signatures. Hot-path shape:
  `fn parse(input: &[u8], out: &mut [Record]) -> Result<usize, ParseError>`
  For general libraries, offer both: an `_into` variant + an allocating convenience wrapper.
- Capacity exhaustion → typed error. Never silently spill to the heap, drop data, or retry forever.
- Core errors: small `Copy` enums; implement `Display` writing to the formatter; no stored `format!` strings.
- Format into `&mut impl fmt::Write` instead of building `String`s.
- `Vec::with_capacity(n)` when `n` is known; `Vec::new()` for empty (no alloc guarantee).
- Avoid intermediate `collect()`s; iterators are lazy — keep them chained or write into caller storage.
- `Arc`/`Rc` are heap-backed even if cloning doesn't allocate.
- Prove allocation claims with a counting `#[global_allocator]` integration test,
  run with `--test-threads=1`. Source inspection alone is not proof.

## Optimization order
asymptotics → bounded work → skip unnecessary work → contiguous layout → fewer bytes moved
→ no allocation/copies → cache locality → branch predictability → SIMD last.
Keep a simple safe reference implementation for every optimized path (test oracle).
Benchmark in release mode with representative inputs; look at latency distributions.

## Bounds and arithmetic
- Cap everything at trust boundaries: iterations, recursion depth, queue length, retries, output size.
- Checked arithmetic on lengths/offsets/capacities before indexing or allocating.
  Choose one overflow semantics per operation (checked / saturating / wrapping) and document it.
- Consider `overflow-checks = true` in `[profile.release]` and a `release-assertions` profile in CI
  (deliberate trade-off — measure the cost).
- `debug_assert!` only re-checks invariants already guaranteed; no side effects inside it.

## Dispatch
- Static dispatch (`impl Trait`, generics) in hot paths; enum dispatch for small closed sets;
  `Box<dyn Trait>` at outer boundaries only.

## Async and concurrency
- Prefer threads when async isn't needed.
- Never block inside async (`std::fs`, `sleep`, CPU-heavy loops) — use `spawn_blocking`.
  It hides on 16-core dev machines and hurts on 2-core servers.
- `async` doesn't mean allocation-free (futures, boxing, spawn, channels all may allocate).
- Bound concurrency: worker counts, bounded channels, backpressure, cancellation.
- Prefer worker-local scratch + merge over shared `Mutex<Vec<_>>` pushes; collect a batch then append once.
- Determinism: never depend on `HashMap` iteration order or thread completion order for output/IDs.

## Unsafe and FFI
- `#![forbid(unsafe_code)]` by default (note: `forbid` can't be overridden — put unsafe test
  helpers like counting allocators in a separate crate).
- Unsafe in leaf modules only; safe public API; `// SAFETY:` per block; Miri tests.
- Hand-written `Send`/`Sync` impls need a written proof.
- FFI docs: ownership, which allocator frees, pointer validity/alignment, lifetimes, thread affinity.
- Rust panics must not unwind across `extern "C"`.

## CI lint baseline (adapt, don't copy blindly)
```toml
[workspace.lints.rust]
unsafe_code = "forbid"
unused_must_use = "deny"

[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "warn", priority = -1 }  # noisy: curate, or expect suppressions
unwrap_used = "deny"   # then allow it in test modules
dbg_macro = "deny"
todo = "deny"
```
CI: `cargo fmt --all -- --check`, `cargo clippy --workspace --all-targets --all-features --locked -- -D warnings`,
`cargo test --workspace --locked`, `cargo test --doc`, `cargo audit` / `cargo deny check`, Miri if unsafe.
Run CI on multiple OSes if the crate touches platform APIs.

## Workspace structure (larger projects)
core (pure logic, minimal deps) / format / runtime / std adapters (I/O, threads) / thin cli.
Don't split crates so finely that trait impls need wrapper types. Proc macros need their own crate.
Test feature combinations (`--no-default-features`); if you promise an MSRV, test it in CI.
