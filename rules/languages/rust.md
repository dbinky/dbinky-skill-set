# Rust Review Rules

## Idiomatic Patterns

- Use `Result<T, E>` for recoverable errors, `panic!` only for programmer bugs and unrecoverable states.
- Use `Option<T>` instead of sentinel values; never use `-1` or empty strings to represent absence.
- Use iterators and combinators (`.map()`, `.filter()`, `.collect()`) over manual loops when clearer.
- Use `impl Trait` in argument position for polymorphism; use `dyn Trait` only when dynamic dispatch is required.
- Use `#[derive(...)]` for standard traits: `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash` as appropriate.
- Use `enum` with variants for state machines and tagged unions; avoid stringly-typed dispatch.
- Use `From`/`Into` for type conversions; implement `From<A> for B`, not standalone conversion functions.
- Use `?` operator for error propagation; avoid explicit `match` on `Result` when chaining.

## Common Anti-Patterns

- Flag: `.unwrap()` or `.expect()` in production code paths where failure is possible. Acceptable in tests.
- Flag: `.clone()` used to satisfy the borrow checker without understanding why borrowing fails.
- Flag: `Box<dyn Trait>` where a generic parameter or `impl Trait` would eliminate allocation.
- Flag: `unsafe` blocks without a `// SAFETY:` comment explaining invariants.
- Flag: `String` parameters where `&str` would suffice; prefer borrowing over ownership transfer.
- Flag: manual `Drop` implementations that could be replaced by RAII wrapper types.
- Flag: `Arc<Mutex<T>>` passed everywhere; consider restructuring to reduce shared mutable state.
- Flag: `#[allow(unused)]` or `#[allow(dead_code)]` on non-WIP code.

## Error Handling

- Define a crate-level error enum using `thiserror` for library code.
- Use `anyhow::Result` in application code for ad-hoc error context with `.context()`.
- Implement `std::error::Error` and `Display` for all custom error types.
- Use `#[from]` attribute in `thiserror` enums for automatic `From` conversions.
- Return `Result` from `main()` for clean error reporting.
- Never use `String` as an error type; create structured errors.

## Naming Conventions

- `snake_case` for functions, methods, variables, modules, crates. `PascalCase` for types and traits.
- `UPPER_SNAKE_CASE` for statics and constants.
- Conversion methods: `as_` (cheap, borrowed), `to_` (expensive, owned), `into_` (consuming).
- Boolean methods: `is_`, `has_`, `can_` prefixes.
- Builder pattern methods: take and return `self` for chaining.
- Module files: `foo.rs` or `foo/mod.rs`. Prefer `foo.rs` with `foo/` subdirectory in edition 2021+.

## Testing Patterns

- Place unit tests in `#[cfg(test)] mod tests` at the bottom of each source file.
- Integration tests go in `tests/` directory at the crate root.
- Use `#[test]` attribute. Use `#[should_panic(expected = "message")]` for panic tests.
- Return `Result<(), E>` from test functions to use `?` operator in test bodies.
- Use `assert_eq!`, `assert_ne!`, `assert!` with descriptive messages.
- Use `proptest` or `quickcheck` for property-based testing of pure functions.
- Use `mockall` sparingly; prefer trait-based dependency injection with real test implementations.
- Use `#[ignore]` for slow tests; run with `cargo test -- --ignored`.

## Concurrency

- Use `tokio` or `async-std` for async I/O. Use `rayon` for CPU-parallel computation.
- Prefer message passing (`mpsc`, `crossbeam-channel`) over shared state (`Mutex`, `RwLock`).
- Use `Arc<T>` for shared ownership across threads; combine with `Mutex` only when mutation is needed.
- Send work to threads with `rayon::spawn` or scoped threads (`std::thread::scope`).
- Use `tokio::select!` for racing async operations with cancellation.
- Avoid holding `MutexGuard` across `.await` points; it blocks the executor thread.
- Use `tokio::sync::Mutex` (not `std::sync::Mutex`) when a lock must be held across await.
- Mark async functions as `Send` when they will be spawned on a multi-threaded runtime.

## Package/Module Structure

- Workspace layout for multi-crate projects: shared `Cargo.toml` at the root with `[workspace]`.
- Separate binary crates (`src/main.rs`) from library crates (`src/lib.rs`); keep `main.rs` thin.
- Use `pub(crate)` for internal-only visibility. Default to private; expose only what is needed.
- Feature flags for optional functionality; document features in `Cargo.toml` comments.
- Keep dependency counts minimal; audit with `cargo tree` and `cargo deny`.
- Use `cfg` attributes for platform-specific code, not separate modules with runtime checks.

## Performance Gotchas

- Avoid allocations in hot paths: use `&str` over `String`, `&[T]` over `Vec<T>`.
- Use `SmallVec` or `ArrayVec` for small, stack-allocated collections.
- Prefer `&[u8]` and `std::io::Write` over `String` for building output in hot paths.
- Use `#[inline]` only on small functions in library crates called across crate boundaries; do not over-inline.
- Avoid `format!` in hot paths; it allocates. Use `write!` to an existing buffer.
- Use `cargo bench` with `criterion` for benchmarks; do not guess at performance.
- Use `Box<[T]>` over `Vec<T>` for fixed-size heap allocations (avoids capacity overhead).
- Enable LTO and `codegen-units = 1` in release profile for maximum optimization.
