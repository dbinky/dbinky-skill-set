# Java Review Rules

## Idiomatic Patterns

- Use `record` types for immutable data carriers (Java 16+). Avoid POJOs with manual getters/setters for DTOs.
- Use `sealed` interfaces/classes to model closed type hierarchies (Java 17+).
- Use `Optional<T>` for return types that may be absent. Never use `null` return for "not found."
- Use `var` for local variables when the type is obvious from the initializer (Java 10+).
- Use `switch` expressions with pattern matching (Java 21+) over `if`/`else` chains for type dispatch.
- Use `Stream` API for declarative collection transformations. Avoid streams for simple iterations with side effects.
- Use `List.of()`, `Map.of()`, `Set.of()` for immutable collection literals.
- Use `try-with-resources` for all `AutoCloseable` resources.

## Common Anti-Patterns

- Flag: raw types (`List` instead of `List<String>`).
- Flag: `Optional.get()` without `isPresent()` check or, better, use `orElseThrow()` / `map()` / `ifPresent()`.
- Flag: `null` returned where `Optional` or empty collection would be appropriate.
- Flag: checked exceptions used for control flow; use unchecked exceptions for programmer errors.
- Flag: mutable static fields without synchronization.
- Flag: `synchronized` on `this` in public classes; use private lock objects.
- Flag: `String` concatenation with `+` in loops; use `StringBuilder`.
- Flag: `java.util.Date` or `Calendar` in new code; use `java.time` API.

## Error Handling

- Use unchecked exceptions (`IllegalArgumentException`, `IllegalStateException`) for programmer errors.
- Use checked exceptions sparingly and only when the caller can meaningfully recover.
- Wrap low-level exceptions with domain exceptions: `throw new OrderException("msg", cause)`.
- Never catch `Throwable` or `Error` unless implementing a top-level error boundary.
- Never swallow exceptions: at minimum log at WARN level with full stack trace.
- Use `try-with-resources` to ensure cleanup; never rely on `finally` for `close()`.

## Naming Conventions

- `PascalCase` for classes, interfaces, enums, records, annotations.
- `camelCase` for methods, fields, parameters, local variables.
- `UPPER_SNAKE_CASE` for `static final` constants.
- Package names: all lowercase, reverse domain notation (`com.example.feature`).
- Boolean methods: `is`, `has`, `can`, `should` prefixes.
- Interfaces: no `I` prefix. Implementations may use `Impl` suffix only if no better name exists.
- Test classes: `FooTest` for unit tests, `FooIntegrationTest` for integration tests.

## Testing Patterns

- Use JUnit 5 (`jupiter`) for new tests. Avoid JUnit 4 annotations in new code.
- Use `@ParameterizedTest` with `@MethodSource` or `@CsvSource` for table-driven tests.
- Use AssertJ for fluent assertions: `assertThat(result).isEqualTo(expected)`.
- Use Mockito for mocking: `@ExtendWith(MockitoExtension.class)` with `@Mock` and `@InjectMocks`.
- Prefer constructor injection over field injection for testability.
- Use `@Testcontainers` for integration tests requiring databases, message brokers, etc.
- Test names: descriptive method names (`shouldReturnEmptyWhenUserNotFound`).
- Use `@Nested` classes in JUnit 5 to group related test cases.

## Concurrency

- Use `java.util.concurrent` utilities. Never use raw `Thread` or `synchronized` for complex coordination.
- Use `ExecutorService` with virtual threads (Java 21+) for I/O-bound concurrency.
- Use `CompletableFuture` for async composition. Avoid blocking on futures in async code.
- Use `ConcurrentHashMap` over `Collections.synchronizedMap`.
- Use `AtomicReference`, `AtomicInteger` for lock-free atomic operations.
- Prefer immutable objects shared across threads to reduce synchronization needs.
- Flag: `double-checked locking` without `volatile` on the checked field.
- Use `StructuredTaskScope` (Java 21+ preview) for structured concurrency.

## Package/Module Structure

- Organize by feature/domain: `com.example.order`, `com.example.user`, not `com.example.controller`.
- Use Java modules (`module-info.java`) for library projects to enforce encapsulation.
- Keep Spring `@Configuration` classes in a top-level `config` package.
- One public class per file (enforced by the language).
- Use Maven or Gradle with dependency management; declare versions in a BOM or parent POM.
- Separate API modules from implementation modules in multi-module projects.

## Performance Gotchas

- Use `StringBuilder` for string concatenation in loops, not `+` operator.
- Use primitive-specialized collections (Eclipse Collections, `IntStream`) to avoid autoboxing.
- Avoid creating unnecessary `Stream` pipelines for simple single-element operations.
- Use `HashMap` initial capacity when the size is known to avoid rehashing.
- Prefer `ArrayList` over `LinkedList` in almost all cases (cache locality).
- Use `jmh` for micro-benchmarks; never benchmark with `System.nanoTime()` in a loop.
- Enable GC logging in production; tune GC for your workload (G1, ZGC, Shenandoah).
- Use `String.intern()` cautiously; it can cause memory leaks in PermGen/Metaspace.
