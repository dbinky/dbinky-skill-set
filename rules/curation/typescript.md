# TypeScript Curation Rules

## Idiomatic Patterns

What senior TypeScript engineers write:

- **Discriminated unions over class hierarchies**: Model variants as a tagged union. `type Shape = { kind: 'circle'; radius: number } | { kind: 'rect'; w: number; h: number }`. Exhaustive `switch` on `kind` gives narrowing for free.
- **Type narrowing over casting**: Write type guard functions (`function isCircle(s: Shape): s is Circle`) instead of silencing the compiler with `as`. Narrowing is proven; casting is a promise the compiler cannot verify.
- **`const` assertions**: Use `as const` to freeze literal types. `const DIRECTIONS = ['north', 'south', 'east', 'west'] as const` produces a readonly tuple of string literals, not `string[]`.
- **`satisfies` operator**: Validate an expression against a type without widening the inferred type. `const palette = { red: [255, 0, 0] } satisfies Record<string, number[]>` keeps the literal type while enforcing structure.
- **Utility types**: Prefer built-ins over hand-rolled equivalents. `Partial<T>`, `Required<T>`, `Pick<T, K>`, `Omit<T, K>`, `Record<K, V>`, `Readonly<T>`, `ReturnType<F>`, `Parameters<F>`.
- **Nullish coalescing and optional chaining**: `value ?? defaultValue` for null/undefined fallback (not `||`, which also catches `0` and `""`). `obj?.prop?.method()` for safe traversal of nullable chains.
- **Async/await over `.then()` chains**: `await` produces linear, debuggable code. Reserve `.then()` for composing independent promises with `Promise.all`.
- **`unknown` over `any`**: `unknown` forces narrowing before use; `any` silently disables the type checker. Use `unknown` for external data and narrow with type guards or Zod schemas.
- **Barrel exports judiciously**: Use `index.ts` barrel files only at public API boundaries (library root, major feature boundary). Barrels inside implementation modules break tree-shaking and create circular-import risks.
- **Strict mode**: Enable `"strict": true` in `tsconfig.json`. This activates `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, and others. Opting out trades safety for short-term convenience.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in TypeScript:

- **`any` everywhere**: Sprinkling `any` to silence type errors defeats the purpose of TypeScript. Each `any` is a hole in the type system that can propagate silently into callers.
- **`as` casting instead of type guards**: `data as User` asserts without checking. A runtime mismatch produces silent undefined behavior, not a type error. Write a guard or use a validation library.
- **Redundant type annotations on inferred literals**: `const x: string = "hello"` annotates what TypeScript already knows. Annotate at boundaries (function signatures, exported types), not on every local variable.
- **Unnecessary try/catch around non-throwing code**: Wrapping `JSON.stringify`, array methods, or pure arithmetic in try/catch adds visual weight and misleads readers about what can fail.
- **Enum overuse for simple string unions**: TypeScript `enum` compiles to an object, has surprising reverse-mapping behavior, and does not interop cleanly with external data. Prefer `type Status = 'active' | 'inactive'` for plain variants.
- **Class overuse**: Classes are appropriate for stateful objects with identity. For bags of related functions or plain data, plain objects and exported functions are simpler, tree-shakeable, and easier to test.
- **Nested ternaries**: `a ? b ? c : d : e` requires careful parsing by every future reader. Extract to an `if/else` block or a lookup object.
- **Barrel files importing everything**: An `index.ts` that re-exports every symbol in a directory forces bundlers to include the entire module graph even when only one export is used.
- **Unnecessary `async`**: An `async` function with no `await` inside wraps its return value in an extra `Promise` and adds a microtask tick with no benefit. Remove `async` or add the missing `await`.
- **Duplicate interfaces**: Defining the same shape (`User`, `ApiResponse`, `Pagination`) in multiple files creates silent drift. Centralize shared types and import them.
- **Overly defensive optional chaining**: `config?.host?.trim()` when `config` and `config.host` are guaranteed to exist by the caller's contract. Defensive chaining on guaranteed-present values hides logic errors that should throw.

## Simplification Strategies

### 1. Replace `any` with proper types

Before — `any` bypasses type safety:

```typescript
function processUser(data: any): any {
  return {
    id: data.id,
    name: data.name.trim(),
    age: data.age,
  };
}
```

After — typed interface and return type:

```typescript
interface RawUser {
  id: string;
  name: string;
  age: number;
}

interface User {
  id: string;
  name: string;
  age: number;
}

function processUser(data: RawUser): User {
  return {
    id: data.id,
    name: data.name.trim(),
    age: data.age,
  };
}
```

### 2. Replace enum with string union type

Before — enum with unnecessary runtime artifact:

```typescript
enum OrderStatus {
  Pending = 'pending',
  Processing = 'processing',
  Shipped = 'shipped',
  Delivered = 'delivered',
  Cancelled = 'cancelled',
}

function canCancel(status: OrderStatus): boolean {
  return status === OrderStatus.Pending || status === OrderStatus.Processing;
}
```

After — string union, zero runtime overhead, JSON-safe:

```typescript
type OrderStatus = 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled';

function canCancel(status: OrderStatus): boolean {
  return status === 'pending' || status === 'processing';
}
```

### 3. Flatten nested ternaries to a lookup with fallback

Before — nested ternary requiring careful mental parsing:

```typescript
function getLabel(status: OrderStatus): string {
  return status === 'pending'
    ? 'Awaiting confirmation'
    : status === 'processing'
    ? 'Being prepared'
    : status === 'shipped'
    ? 'On the way'
    : status === 'delivered'
    ? 'Delivered'
    : 'Cancelled';
}
```

After — Record lookup with nullish fallback, exhaustive and readable:

```typescript
const STATUS_LABELS: Record<OrderStatus, string> = {
  pending: 'Awaiting confirmation',
  processing: 'Being prepared',
  shipped: 'On the way',
  delivered: 'Delivered',
  cancelled: 'Cancelled',
};

function getLabel(status: OrderStatus): string {
  return STATUS_LABELS[status] ?? 'Unknown';
}
```

### 4. Remove redundant type annotations

Before — annotations restate what the compiler already infers:

```typescript
const maxRetries: number = 3;
const baseUrl: string = 'https://api.example.com';
const isEnabled: boolean = true;

function double(n: number): number {
  const result: number = n * 2;
  return result;
}
```

After — annotate at boundaries; trust inference for locals:

```typescript
const maxRetries = 3;
const baseUrl = 'https://api.example.com';
const isEnabled = true;

function double(n: number): number {
  return n * 2;
}
```

## Dead Code Signals

TypeScript-specific indicators that code is no longer exercised or referenced:

- **Unused exports**: The TypeScript compiler does not warn about exported symbols with no consumers. Look for exported functions, classes, and types that appear in no import statement inside the project and in no public API documentation.
- **Unused `package.json` dependencies**: A package in `dependencies` or `devDependencies` that nothing in `src/` imports. Common after refactors that swap libraries but leave the old entry in the manifest.
- **Dead `else` branches that log and return a default**: A branch that logs `"should never happen"` and returns a sentinel is a sign the condition was valid once but is no longer reachable. If no test exercises it, it is likely dead.
- **Unused types and interfaces**: Types declared in a shared file that no other file imports. IDEs surface these, but they often survive code review because they look like documentation.
- **Stale route handlers**: Express/Fastify/Next.js route files that define handlers never registered in the router, or registered routes whose corresponding page or API file was deleted.
- **Barrel file exports with no consumers**: An `index.ts` that re-exports `Foo` when no file outside the barrel's directory imports `Foo`. The export exists to support consumers that were removed.
