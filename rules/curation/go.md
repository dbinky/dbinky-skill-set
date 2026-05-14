# Go Curation Rules

## Idiomatic Patterns

What senior Go engineers write:

- **Error handling**: Wrap errors with context and return early. `if err != nil { return fmt.Errorf("context: %w", err) }`. Never log and return the same error — pick one.
- **Interfaces**: Accept interfaces, return structs. Define interfaces at the call site (consumer), not at the implementation site. Single-method interfaces use the `-er` suffix (e.g., `Reader`, `Stringer`, `Closer`).
- **Constructor functions**: `NewFoo(deps) *Foo` — inject dependencies via constructor, not global variables or field assignment after construction.
- **Table-driven tests**: Use `t.Run` for subtests. Use `t.Helper()` in helper functions so failures report the caller's line, not the helper's.
- **Context propagation**: Pass `context.Context` as the first parameter to any function that performs I/O or may be cancelled.
- **Package structure**: Keep packages flat. Use `internal/` for packages that must not be imported outside the module.
- **Named returns**: Use named return values only for documentation or to enable deferred mutation. Never use naked returns in functions longer than a few lines.
- **Slice pre-allocation**: `make([]T, 0, expectedCap)` when the capacity is known or estimable, avoiding unnecessary reallocations.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in Go:

- **Over-interfacing**: Interfaces with 5+ methods defined at the implementation site. Go interfaces should be small and discovered by consumers, not declared upfront.
- **Java-style OOP**: Unnecessary struct embedding for fake inheritance, factory patterns wrapping 2-field structs, builder types that add no value over a struct literal.
- **Defensive nil checks on guaranteed non-nil values**: Checking for nil on values that constructors or the type system already guarantee are non-nil adds noise without safety.
- **Catch-all error handling**: `log.Fatal` in library code (belongs only in `main`), swallowing errors with `_` without a justifying comment.
- **Channel overuse**: Using channels for simple counter or flag coordination where a `sync.Mutex` or sequential code is clearer and less error-prone.
- **Package grab-bags**: Packages named `util`, `common`, `helpers`, or `base` that collect unrelated types and functions. Each package should do one thing.
- **Unnecessary goroutines**: Launching goroutines for work that is inherently sequential or single-use, adding synchronization overhead with no throughput benefit.
- **`interface{}`/`any` overuse**: Using `any` where a concrete type, a small interface, or a generic type parameter would be self-documenting and type-safe.
- **Redundant `else` after `return`**: An `else` block after an `if` that returns is dead vertical space; the else branch is the fall-through.

## Simplification Strategies

### 1. Flatten nested error handling

Before — LLM-generated deep nesting:

```go
func process(r io.Reader) (*Result, error) {
    data, err := io.ReadAll(r)
    if err == nil {
        parsed, err := parse(data)
        if err == nil {
            validated, err := validate(parsed)
            if err == nil {
                return validated, nil
            } else {
                return nil, fmt.Errorf("validate: %w", err)
            }
        } else {
            return nil, fmt.Errorf("parse: %w", err)
        }
    } else {
        return nil, fmt.Errorf("read: %w", err)
    }
}
```

After — idiomatic early returns:

```go
func process(r io.Reader) (*Result, error) {
    data, err := io.ReadAll(r)
    if err != nil {
        return nil, fmt.Errorf("read: %w", err)
    }

    parsed, err := parse(data)
    if err != nil {
        return nil, fmt.Errorf("parse: %w", err)
    }

    validated, err := validate(parsed)
    if err != nil {
        return nil, fmt.Errorf("validate: %w", err)
    }

    return validated, nil
}
```

### 2. Inline unnecessary interfaces

Before — over-interfaced at implementation site:

```go
// In the service package
type UserRepository interface {
    Create(ctx context.Context, u User) error
    Update(ctx context.Context, u User) error
    Delete(ctx context.Context, id string) error
    FindByID(ctx context.Context, id string) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
}

type UserService struct {
    repo UserRepository
}

func (s *UserService) Deactivate(ctx context.Context, id string) error {
    u, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("find user: %w", err)
    }
    u.Active = false
    return s.repo.Update(ctx, *u)
}
```

After — consumer-side interface scoped to what is actually used:

```go
// In the service package — only what Deactivate needs
type userFinder interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Update(ctx context.Context, u User) error
}

type UserService struct {
    repo userFinder
}

func (s *UserService) Deactivate(ctx context.Context, id string) error {
    u, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("find user: %w", err)
    }
    u.Active = false
    return s.repo.Update(ctx, *u)
}
```

### 3. Replace channel-based synchronization with mutex

Before — channel used as a clunky mutex:

```go
type Counter struct {
    ch chan int
}

func NewCounter() *Counter {
    c := &Counter{ch: make(chan int, 1)}
    c.ch <- 0
    return c
}

func (c *Counter) Increment() {
    val := <-c.ch
    c.ch <- val + 1
}

func (c *Counter) Value() int {
    val := <-c.ch
    c.ch <- val
    return val
}
```

After — straightforward mutex:

```go
type Counter struct {
    mu  sync.Mutex
    val int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.val++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.val
}
```

## Dead Code Signals

Go-specific indicators that code is no longer exercised or referenced:

- **Unused exports**: The compiler does not warn about exported identifiers with no callers outside the package. Look for exported types, functions, and constants that appear in no `go.sum`-tracked import and no internal call site.
- **Unused aliased imports**: `import _ "pkg"` retained for side effects that no longer apply. Check whether the registered driver, codec, or plugin is still consumed anywhere.
- **Orphaned test helpers**: Unexported functions in `_test.go` files that no test in the package calls. Common after test refactors that inline setup.
- **Unused struct fields**: Fields that are set in constructors or `json.Unmarshal` but never read by any method or caller. Frequently left behind after a schema change.
- **Dead `init()` functions**: `init()` bodies whose sole purpose was registering something (a flag, a metric, a codec) that has since been removed or replaced.
- **Stale build tags**: `//go:build ignore` or feature-flag build constraints (`//go:build integration`) on files whose tests or commands have been superseded. The file compiles in no configuration and accumulates drift.
