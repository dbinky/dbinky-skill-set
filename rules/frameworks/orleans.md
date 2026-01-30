# Orleans Review Rules

## Architecture Patterns

Orleans follows the Virtual Actor Model. Every grain is a single-threaded, isolated unit of state and logic activated on demand by the runtime.

**Flag in review:**
- Direct grain-to-grain calls that form deep call chains (risk of deadlocks with non-reentrant grains)
- Grains holding references to non-grain objects or shared mutable state
- Using `Task.Run` or manual threading inside grains — this breaks the single-threaded guarantee
- Instantiating grains with `new` instead of obtaining references through `IGrainFactory`
- Placing business logic outside of grains when it accesses grain state

**Expect to see:**
- Grain interfaces defined in a separate contracts project
- `[Alias]` attributes on grain interfaces and state classes for schema evolution
- Grain keys chosen intentionally (string, Guid, long, compound)

## Common Anti-Patterns

- **God Grain**: A single grain type handling too many responsibilities. Split by bounded context.
- **Chatty Grains**: Many fine-grained calls between grains per operation. Prefer coarser grain methods.
- **Blocking Calls**: `Task.Result`, `Task.Wait()`, or `.GetAwaiter().GetResult()` inside grain code causes deadlocks.
- **Static Mutable State**: Any `static` mutable field in a grain class breaks isolation guarantees.
- **Over-Persistence**: Writing state on every call. Batch writes or use conditional persistence.
- **Reentrant by Default**: Marking grains `[Reentrant]` without understanding that it allows interleaved execution.

## State Management

- Use `IPersistentState<T>` with `[PersistentState("name", "store")]` for durable state.
- Call `state.WriteStateAsync()` explicitly — Orleans does NOT auto-persist.
- Flag any grain that reads state, mutates, then does an `await` before writing — another call could interleave if reentrant.
- Prefer `ReadStateAsync()` in `OnActivateAsync()` only when needed; the runtime calls it automatically.
- For transient/cache state, plain fields are fine — no persistence needed.
- Avoid storing large object graphs in grain state; consider external storage with grain-held keys.

## Testing Patterns

- Use `TestCluster` from `Microsoft.Orleans.TestingHost` for integration tests.
- Unit test grain logic by extracting pure functions; avoid testing grain activation directly in unit tests.
- Flag tests that use `Task.Delay` for synchronization — use `TestCluster` utilities or semaphores.
- Grain tests should verify state transitions, not internal implementation details.
- Mock `IGrainFactory` for isolated grain unit tests.

## Performance

- **Grain Activation Overhead**: Flag grains that activate/deactivate frequently for short operations. Consider stateless worker grains (`[StatelessWorker]`) for compute-heavy, stateless work.
- **Serialization Cost**: Flag complex object graphs in grain state. Prefer flat DTOs.
- **Timer vs Reminder**: Timers are in-memory (lost on deactivation). Reminders survive but have minimum 1-minute resolution. Flag misuse.
- **Streams**: Implicit subscriptions create activations. Flag unbounded stream subscriptions.
- **Reentrancy**: `[Reentrant]` or `[AlwaysInterleave]` improves throughput but risks consistency. Flag unless justified.

## Lifecycle

- `OnActivateAsync()` — initialization logic. Flag anything that can fail transiently without retry.
- `OnDeactivateAsync()` — cleanup. Not guaranteed to run (crash, silo shutdown). Do NOT rely on it for critical persistence.
- `DeactivateOnIdle()` — explicit deactivation. Flag if called in a loop or hot path.
- Grain activation is transparent to callers. Flag code that assumes a grain is "alive."
- Silo lifecycle: flag `ISiloLifecycle` participants that do blocking I/O in Stage 0.

## Configuration

- Use `SiloBuilder` and `ClientBuilder` fluent APIs. Flag direct configuration object mutation.
- Clustering: flag hardcoded cluster membership endpoints. Use service discovery or config.
- Storage providers: flag missing `AddMemoryGrainStorage` / `AddAzureTableGrainStorage` registrations.
- Serialization: flag missing serializer configuration. Orleans 7+ requires explicit serializer setup.
- Dashboard: flag `UseDashboard()` left enabled in production builds.

## Error Handling

- Grain method exceptions propagate to callers as `OrleansException`. Flag swallowed exceptions inside grains.
- Flag empty `catch` blocks in grain code — they hide failures from callers.
- Use `ExceptionDispatchInfo` or `throw;` — never `throw ex;`.
- For transient failures (storage, external services), implement retry with `Polly` or equivalent.
- Flag grains that enter an inconsistent state after a partial failure (wrote state, then external call failed).
- `OnActivateAsync` failures prevent activation — flag missing error handling there.

## Dependency Injection

- Orleans supports constructor injection in grains. Flag service locator patterns (`GetService` calls).
- Flag injecting scoped services into grains — grains are effectively singletons while active.
- `IGrainFactory`, `ILogger<T>`, and `IPersistentState<T>` are the standard injected dependencies.
- Flag injecting `HttpClient` directly; use `IHttpClientFactory`.

## Versioning and Schema Evolution

- Flag grain state classes without `[Alias]` — renaming or moving them breaks deserialization.
- Flag grain interfaces without `[Alias]` — same issue for grain references.
- New fields should have sensible defaults for backward compatibility.
- Flag removal of fields from state classes without a migration strategy.
