# Rust Development Standards

Standards passed to sub-agents working on Rust projects. Source: `rust-code-standards`
memory-mcp memory (scope: global) + user workflow preferences.

## Anti-patterns to avoid

1. `.clone()` instead of borrowing — unnecessary allocations
2. `.unwrap()`/`.expect()` overuse — use `unwrap_or`, `unwrap_or_default`, or propagate with `?`
3. `.collect()` too early — prefer lazy iteration; only collect when multiple passes needed
4. `unsafe` without clear need
5. Premature abstraction — don't introduce traits/generics until the pattern is clear
   from at least two concrete uses. But when a trait boundary or generic genuinely
   clarifies intent (e.g., `impl Read` expressing "any reader"), use it — good generics
   improve readability by making contracts explicit
6. Global mutable state — breaks testability and thread safety
7. Macros that hide logic — keep logic visible and debuggable
8. Ignoring lifetime annotations — but don't add them where not needed
9. Premature optimization — correctness first

## Positive patterns

- `const` slices for rule sets (zero-cost, no allocation)
- `.contains()` over `.iter().any()` for slice membership
- `#[cfg(test)] mod tests` embedded in the same source file
- Stripped + LTO release builds (`[profile.release]` in Cargo.toml)
- Two-pass iteration over short strings preferred to collecting into Vec

## Build verification (run in this order)

1. `cargo fmt` — format code
2. `cargo clippy -- -D warnings` — zero warnings, treat as errors
3. `cargo nextest run --workspace` — all tests pass (use nextest, not `cargo test`)
4. `cargo build --release` — release binary builds
5. Verify binary size hasn't grown unexpectedly

## Error handling

- Propagate with `?` — don't swallow errors
- Use `thiserror` for library error types, `anyhow` for application code
- Include context in error messages: what failed and why
- Match on error variants only when different recovery paths exist

## Testing

- Tests live in `#[cfg(test)] mod tests` in the same file
- Name tests descriptively: `test_<function>_<scenario>_<expected>`
- Test edge cases: empty input, boundary values, error paths
- Use `assert_eq!` over `assert!` for better failure messages
