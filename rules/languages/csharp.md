# C# Review Rules

## Idiomatic Patterns

- Use `var` for local variables when the type is obvious from the right-hand side.
- Use expression-bodied members for single-expression properties and methods.
- Use `record` types for immutable data transfer objects; use `class` for mutable entities with behavior.
- Use collection expressions (`[1, 2, 3]`) in C# 12+ and `ImmutableArray` for immutable collections.
- Use `required` keyword on properties that must be set during initialization.
- Use `init`-only setters for immutable properties set at construction time.
- Use pattern matching (`is`, `switch` expressions) over `if`/`else` chains for type/value dispatch.
- Use `using` declarations (without braces) over `using` blocks for simpler scope management.

## Common Anti-Patterns

- Flag: `async void` methods outside of event handlers. Use `async Task` instead.
- Flag: `Task.Run` wrapping an already-async method (async-over-async).
- Flag: catching `Exception` without rethrowing or logging the full stack trace.
- Flag: `string.Format` or concatenation where string interpolation (`$""`) would be clearer.
- Flag: `public` fields on classes. Use properties with getters/setters.
- Flag: `DateTime.Now` instead of `DateTimeOffset.UtcNow` or injected clock for testability.
- Flag: `Thread.Sleep` in async code; use `Task.Delay`.
- Flag: mutable static fields without thread-safety guarantees.

## Error Handling

- Throw specific exception types: `ArgumentNullException`, `InvalidOperationException`, etc.
- Use `throw;` to rethrow, never `throw ex;` (preserves stack trace).
- Use `ArgumentNullException.ThrowIfNull(param)` in .NET 6+ for guard clauses.
- Implement `Result<T>` pattern for operations that fail predictably; reserve exceptions for unexpected failures.
- Use `ExceptionDispatchInfo.Capture` when rethrowing across async boundaries.
- Log structured exception data, not just `ex.Message`.

## Naming Conventions

- `PascalCase` for public members, types, namespaces, methods, properties, events.
- `camelCase` for local variables and parameters. `_camelCase` for private fields.
- Prefix interfaces with `I`: `IRepository`, `ILogger`.
- Async methods must end with `Async` suffix: `GetUserAsync()`.
- Boolean properties/methods: use `Is`, `Has`, `Can`, `Should` prefixes.
- File name matches the primary type name. One public type per file.

## Testing Patterns

- Use xUnit for new projects. NUnit is acceptable in existing codebases.
- `[Fact]` for single-case tests, `[Theory]` with `[InlineData]` for parameterized tests.
- Use `FluentAssertions` for readable assertions: `result.Should().Be(expected)`.
- Mock dependencies with `NSubstitute` or `Moq`. Prefer constructor injection.
- Use `WebApplicationFactory<T>` for ASP.NET Core integration tests.
- Name tests: `MethodName_Scenario_ExpectedBehavior`.
- Use `ITestOutputHelper` for test logging, not `Console.WriteLine`.
- Test async methods with `await` directly; never use `.Result` or `.Wait()` in tests.

## Concurrency

- Use `async`/`await` for I/O-bound operations. Use `Parallel.ForEachAsync` for CPU-bound parallelism.
- Avoid `Task.Result` and `Task.Wait()`; they can deadlock in synchronization contexts.
- Use `SemaphoreSlim` for async-compatible resource throttling.
- Use `CancellationToken` on all async method signatures; propagate cancellation throughout.
- Use `Channel<T>` for producer/consumer patterns over `BlockingCollection`.
- Prefer `ConcurrentDictionary` over `Dictionary` with manual locking.
- Use `lock` (or `Lock` in .NET 9+) for simple critical sections; avoid `Monitor` directly.
- `ConfigureAwait(false)` in library code to avoid deadlocks in non-ASP.NET contexts.

## Package/Module Structure

- Use folder-per-feature with matching namespaces, not folder-per-layer.
- Keep `Program.cs` minimal: configure services and middleware, do not put business logic.
- Use `internal` visibility by default; expose only what consumers need.
- Separate projects: `MyApp.Domain`, `MyApp.Application`, `MyApp.Infrastructure`, `MyApp.Api`.
- Use `Directory.Build.props` for shared project properties across a solution.
- Register services in `IServiceCollection` extension methods per feature area.

## Performance Gotchas

- Use `Span<T>` and `Memory<T>` to avoid heap allocations for slicing operations.
- Use `StringBuilder` for string concatenation in loops, not `+` or interpolation.
- Use `ValueTask<T>` over `Task<T>` for methods that often complete synchronously.
- Avoid LINQ in hot paths; hand-written loops with `Span<T>` are significantly faster.
- Use `ArrayPool<T>.Shared` for temporary buffer allocations.
- Use `IAsyncEnumerable<T>` for streaming large result sets instead of materializing lists.
- Avoid boxing value types: use generic collections, not `ArrayList` or `object` parameters.
- Profile with `BenchmarkDotNet` for micro-benchmarks; use dotnet-trace/dotnet-counters for runtime analysis.
