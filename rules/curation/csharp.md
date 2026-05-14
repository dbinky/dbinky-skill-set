# C# Curation Rules

## Idiomatic Patterns

What senior C# engineers write:

- **LINQ over manual loops**: Compose readable pipelines with `Where`, `Select`, `FirstOrDefault`, etc. `items.Where(x => x.IsActive).Select(x => x.Name).ToList()` is clearer than an equivalent `foreach` with a conditional `Add`.
- **Pattern matching**: Use `if (obj is string s)`, switch expressions with patterns, and `is not null` instead of explicit type checks and casts. Modern C# pattern matching eliminates most `GetType()` and explicit `as`/null-check pairs.
- **Nullable reference types**: Enable `<Nullable>enable</Nullable>` project-wide. Use `string?` for genuinely nullable references and `string` for guaranteed non-null. Respect the compiler warnings — a suppressed `!` is a deferred bug.
- **Async/await**: Return `Task<T>` or `ValueTask<T>` from async methods. Never block with `.Result` or `.Wait()` on the calling thread. Use `ConfigureAwait(false)` in library code to avoid capturing the synchronization context.
- **Records for DTOs**: `record UserDto(string Name, string Email)` gives value equality, deconstruction, and `ToString` for free. Prefer records over plain classes for data carriers that have no behavior.
- **Primary constructors (C# 12+)**: `class Service(ILogger<Service> logger, IRepo repo)` eliminates the constructor body boilerplate for dependency injection. Use for types whose only constructor job is assignment.
- **Collection expressions (C# 12+)**: Use `[1, 2, 3]` and `[..existing, newItem]` instead of `new List<int> { 1, 2, 3 }` or `Enumerable.Concat`.
- **Using declarations**: `using var stream = File.OpenRead(path);` disposes at end of scope without an extra nesting level. Reserve `using (...)  { }` blocks for when you need to control disposal mid-scope.
- **Proper IDisposable implementation**: Implement `IDisposable` with a `protected virtual void Dispose(bool disposing)` pattern when the class owns unmanaged resources or nested disposables. Seal or document the pattern for subclasses.
- **Extension methods for fluent APIs**: Group extension methods in a static class named `<Subject>Extensions`. Keep each method focused on a single operation so callers can compose rather than nest.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in C#:

- **Null checks on non-nullable types**: Checking `if (name == null)` on a `string` parameter in a nullable-enabled project adds noise that contradicts the type contract and confuses the compiler's flow analysis.
- **Manual loops instead of LINQ**: A `foreach` with an `if` and an `Add` is almost always replaceable with `Where`/`Select`/`ToList`. Manual loops should appear only when mutation, early-exit performance, or side-effect ordering matters.
- **Empty catch blocks**: `catch (Exception) { }` silently swallows failures. At minimum, log the exception. Catching `Exception` broadly is usually a sign the calling code should not be handling the error at all.
- **Stringly-typed patterns**: Using `string` to represent a fixed set of values (status codes, operation names, roles) where an `enum` or a discriminated union via records would give compile-time safety and remove string comparisons throughout the codebase.
- **God services**: Services with 20+ methods that handle unrelated concerns. A class that manages users, sends emails, and updates audit logs is three services collapsed into one. Split by responsibility.
- **Unnecessary `Task.Run` wrapping synchronous code**: `Task.Run(() => ComputeSync())` offloads CPU-bound work to the thread pool, which is appropriate for UI apps keeping the main thread free but harmful in ASP.NET Core where it wastes a thread and adds scheduling overhead with no benefit.
- **`async void` outside event handlers**: `async void` methods cannot be awaited and their exceptions are unobservable — they crash the process or disappear silently. Use `async Task` everywhere except genuine event handler signatures.
- **`dynamic` overuse**: `dynamic` disables IntelliSense, compiler checks, and AOT compatibility. Use generics, interfaces, or `object` with pattern matching. Reach for `dynamic` only when interoperating with COM or late-bound scripting hosts.
- **Repository wrapping EF with no additional value**: A `UserRepository` that directly delegates `Add`, `Find`, `SaveChanges` to `DbContext` without adding query encapsulation, caching, or cross-cutting behavior is an abstraction layer with no abstraction. Inject `DbContext` directly, or add real value before wrapping it.
- **Redundant `.ToString()` in string interpolation**: `$"Hello {name.ToString()}"` — string interpolation already calls `ToString()`. The explicit call is noise.

## Simplification Strategies

### 1. Replace manual loop with LINQ

Before — LLM-generated foreach accumulator:

```csharp
var activeNames = new List<string>();
foreach (var item in items)
{
    if (item.IsActive)
    {
        activeNames.Add(item.Name);
    }
}
return activeNames;
```

After — idiomatic LINQ pipeline:

```csharp
return items
    .Where(x => x.IsActive)
    .Select(x => x.Name)
    .ToList();
```

### 2. Use pattern matching instead of type-check and cast

Before — GetType comparison with explicit cast:

```csharp
string Describe(Shape shape)
{
    if (shape.GetType() == typeof(Circle))
    {
        var c = (Circle)shape;
        return $"Circle with radius {c.Radius}";
    }
    else if (shape.GetType() == typeof(Rectangle))
    {
        var r = (Rectangle)shape;
        return $"Rectangle {r.Width}x{r.Height}";
    }
    else
    {
        return "Unknown shape";
    }
}
```

After — switch expression with type patterns:

```csharp
string Describe(Shape shape) => shape switch
{
    Circle c      => $"Circle with radius {c.Radius}",
    Rectangle r   => $"Rectangle {r.Width}x{r.Height}",
    _             => "Unknown shape",
};
```

### 3. Replace class with record for data carriers

Before — class with manually implemented equality:

```csharp
public class Point
{
    public double X { get; }
    public double Y { get; }

    public Point(double x, double y) { X = x; Y = y; }

    public override bool Equals(object? obj) =>
        obj is Point p && p.X == X && p.Y == Y;

    public override int GetHashCode() => HashCode.Combine(X, Y);

    public override string ToString() => $"Point({X}, {Y})";
}
```

After — positional record (compiler generates equality, hash, and ToString):

```csharp
public record Point(double X, double Y);
```

## Dead Code Signals

C#-specific indicators that code is no longer exercised or referenced:

- **Unused private methods**: The compiler may warn but projects with `<NoWarn>` or suppression attributes can hide these. Look for `private` methods with no call sites inside the class — common after refactors that inlined logic.
- **Unreferenced projects in solution**: Projects listed in `.sln` or `Directory.Build.props` that are not referenced (directly or transitively) by any executable or test project. They accumulate build time and version drift with no consumer.
- **`#if` blocks for removed configurations**: Preprocessor symbols (`DEBUG`, `LEGACY`, `NETSTANDARD2_0`) whose corresponding build configurations or target frameworks have been removed from the project. The enclosed code compiles in no active configuration.
- **Unused DI registrations in Program.cs/Startup.cs**: `services.AddScoped<IFoo, Foo>()` calls for types that are never resolved — either because the feature was removed or the interface was renamed. These registrations instantiate at scope creation (for Singleton) or waste scanning time.
- **`[Obsolete]` members with zero references**: Members marked `[Obsolete]` that have no remaining callers inside the solution. The deprecation warning was the off-ramp; the code itself should now be deleted.
- **Dead event handlers**: Events declared on a class but never raised, or event-handler registrations (`+= OnFoo`) for events that are never fired. Common in UI code migrated away from event-driven patterns toward reactive or command-based models.
- **Unused NuGet packages**: Packages listed in `.csproj` with no `using` directives or direct API calls anywhere in the project. Check with `dotnet list package` and cross-reference usage — transitive dependencies can mask direct ones.
