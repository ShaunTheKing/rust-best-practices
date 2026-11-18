# Rust Best Practices — Agent Skill

A Claude Code skill for writing and reviewing idiomatic Rust, aimed at someone learning Rust
coming from C++ and Python. Layout follows [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills).

## Skills

| Skill | Description |
|---|---|
| [`rust-best-practices`](skills/rust-best-practices/SKILL.md) | Core rules: errors & panics, ownership & borrowing, types & API design, idioms, modules, testing, tooling, unsafe — plus a gotchas list |

### References (loaded only when relevant)
- [`performance-and-advanced.md`](skills/rust-best-practices/references/performance-and-advanced.md) — allocation discipline, hot paths, async, concurrency, unsafe/FFI, CI lint tables
- [`style-opinions.md`](skills/rust-best-practices/references/style-opinions.md) — strict house style and where sources disagree

## Install

```bash
git clone https://github.com/ShaunTheKing/rust-best-practices.git
cp -R rust-best-practices/skills/rust-best-practices ~/.claude/skills/
```

Then it auto-loads when writing/reviewing Rust, or run `/rust-best-practices <path>` for a review.

## Sources
- [canonical/rust-best-practices](https://github.com/canonical/rust-best-practices)
- [Rust best practices gist (allocation discipline)](https://gist.github.com/auser/c3161f55a8393faa8af5ddda68c6befa)
- [r/rust: Rust practices learned in a team](https://www.reddit.com/r/rust/comments/1d28yqe/what_are_some_rust_practices_you_learned_in_a/)
- [users.rust-lang.org: Best practices to write Rust code](https://users.rust-lang.org/t/best-practices-to-write-rust-code/110040)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
