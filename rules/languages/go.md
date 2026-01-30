# Go Review Rules

## Idiomatic Patterns

- Functions should return `(result, error)`, never `panic` for expected failures.
- Use value receivers for immutable methods, pointer receivers for mutations or large structs.
- Accept interfaces, return structs. Do not define interfaces on the implementer side.
- Use `context.Context` as the first parameter for any function that does I/O or may be cancelled.
- Prefer table-driven tests with `t.Run` subtests over sequential assertions.
- Use `errors.Is` and `errors.As` instead of string comparison or type assertion on errors.
- Embed interfaces in structs only when delegating behavior, not for fake inheritance.
- Use `io.Reader` and `io.Writer` for streaming data, not `[]byte` buffers.

## Common Anti-Patterns

- Flag: goroutine launched without clear ownership or shutdown mechanism.
- Flag: `init()` functions that perform I/O or have side effects beyond registration.
- Flag: naked `return` in functions with named return values longer than 5 lines.
- Flag: `interface{}` / `any` used where a concrete type or generic would suffice.
- Flag: channel used where a mutex would be simpler and clearer.
- Flag: error ignored with `_` without an explicit comment justifying it.
- Flag: `sync.WaitGroup` passed by value instead of pointer.
- Flag: using `time.Sleep` in tests instead of synchronization primitives.

## Error Handling

- Wrap errors with `fmt.Errorf("context: %w", err)` to preserve the error chain.
- Define sentinel errors as package-level `var ErrFoo = errors.New("foo")`.
- Never log and return the same error; pick one.
- Custom error types should implement `Error() string` and optionally `Unwrap() error`.
- Use `errors.Join` for collecting multiple errors in Go 1.20+.
- Return early on error; avoid deep nesting with else blocks.

## Naming Conventions

- Exported names are PascalCase, unexported are camelCase. No underscores.
- Acronyms are all-caps: `HTTPClient`, `userID`, not `HttpClient`, `userId`.
- Package names are lowercase, single-word, no underscores. Avoid `util`, `common`, `base`.
- Interface names: single-method interfaces use method name + `er` (e.g., `Reader`, `Stringer`).
- Test files: `foo_test.go` in the same package for white-box, `foo_test` package for black-box.
- Getters omit `Get`: use `Name()` not `GetName()`. Setters use `SetName()`.

## Testing Patterns

- Use `testing` stdlib. Reach for `testify` only when assertions significantly improve clarity.
- Table-driven tests with named subtests: `for _, tc := range tests { t.Run(tc.name, ...) }`.
- Use `t.Helper()` in test helper functions so failures report the caller's line.
- Use `t.Cleanup()` over `defer` for test resource teardown.
- Use `t.Parallel()` for independent tests. Never use it with shared mutable state.
- Use `testdata/` directory for fixture files; it is ignored by the go tool.
- Integration tests gated with `if testing.Short() { t.Skip() }`.
- Prefer `httptest.NewServer` over mocking HTTP clients.

## Concurrency

- Every goroutine must have a clear shutdown path, typically via `context.Context` cancellation.
- Use `errgroup.Group` from `golang.org/x/sync` for coordinated goroutine work.
- Protect shared state with `sync.Mutex`; prefer `sync.RWMutex` for read-heavy workloads.
- Never pass `sync.Mutex` or `sync.WaitGroup` by value.
- Channel direction should be specified in function signatures: `chan<-` or `<-chan`.
- Beware goroutine leaks: if a goroutine reads from a channel, ensure the channel is closed.
- Use `sync.Once` for lazy initialization, not double-checked locking.
- `select` with `default` makes channel operations non-blocking; ensure that is intentional.

## Package/Module Structure

- Flat package structure preferred. Avoid deep nesting (`pkg/foo/bar/baz`).
- `internal/` for packages that must not be imported outside the module.
- `cmd/<binary>/main.go` for each executable entry point.
- Avoid the `pkg/` convention unless the project is a library exposing multiple packages.
- One package should do one thing. If a package has multiple unrelated types, split it.
- Dependency injection via constructor functions, not global variables.

## Performance Gotchas

- Pre-allocate slices with `make([]T, 0, expectedCap)` when the size is known or estimable.
- Use `strings.Builder` for string concatenation in loops, not `+` or `fmt.Sprintf`.
- Avoid `reflect` in hot paths; use generics or code generation instead.
- Beware of unintended allocations from `append` when capacity is exceeded.
- Use `sync.Pool` for frequently allocated and discarded objects of uniform size.
- Profile before optimizing. Use `pprof` and benchmarks, not guesswork.
- Pointer indirection can defeat CPU cache; slices of values outperform slices of pointers for small structs.
- `defer` in tight loops adds overhead; move the call outside the loop if possible.
