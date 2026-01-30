# TypeScript Review Rules

## Idiomatic Patterns

- Use strict mode: `"strict": true` in `tsconfig.json`. Never use `// @ts-ignore` without justification.
- Prefer `interface` for object shapes that may be extended; use `type` for unions, intersections, and mapped types.
- Use discriminated unions with a `kind` or `type` field over class hierarchies for data modeling.
- Use `readonly` for properties that should not be mutated after construction.
- Use `as const` for literal type inference on objects and arrays.
- Prefer `unknown` over `any` for values of uncertain type; narrow with type guards.
- Use `satisfies` operator to validate types without widening.
- Use template literal types for string pattern enforcement.

## Common Anti-Patterns

- Flag: `any` used as a type annotation without a comment justifying it.
- Flag: type assertions (`as Type`) that could be replaced by type narrowing or generics.
- Flag: non-null assertions (`value!`) outside of test code.
- Flag: `enum` with computed values; prefer `as const` objects for string unions.
- Flag: barrel files (`index.ts` re-exports) that cause circular dependencies or bundle bloat.
- Flag: `Object`, `Function`, `{}` as type annotations; use specific types.
- Flag: optional chaining (`?.`) more than 3 levels deep; restructure the data or validate earlier.
- Flag: `export default` in library code; prefer named exports for refactoring safety.

## Error Handling

- Use `Result<T, E>` pattern (custom union type) for functions that can fail predictably.
- Throw only for truly exceptional, unrecoverable errors.
- Catch specific error types; use `instanceof` narrowing in catch blocks.
- Define error classes extending `Error` with a `code` property for programmatic handling.
- Never catch and ignore errors silently. At minimum, log or rethrow.
- Use `Promise.allSettled` over `Promise.all` when partial failure is acceptable.

## Naming Conventions

- `PascalCase` for types, interfaces, classes, enums, React components.
- `camelCase` for variables, functions, methods, parameters.
- `UPPER_SNAKE_CASE` for true constants (compile-time values, config).
- Prefix interfaces with `I` only in legacy codebases; modern TS omits it.
- Boolean variables: use `is`, `has`, `can`, `should` prefixes.
- Generic type params: `T` for single, descriptive names for multiple (`TKey`, `TValue`).
- File names: `kebab-case.ts` for modules, `PascalCase.tsx` for React components.

## Testing Patterns

- Use Vitest or Jest with `ts-jest` or SWC transform. Avoid Mocha in new projects.
- Co-locate test files: `foo.test.ts` next to `foo.ts`, or in `__tests__/` directory.
- Use `describe`/`it` with clear human-readable descriptions.
- Type-check test files; do not exclude them from `tsconfig`.
- Use `@testing-library/react` for component tests; avoid Enzyme.
- Mock modules with `vi.mock()` or `jest.mock()` at the top level; prefer dependency injection.
- Test types with `expectTypeOf` (Vitest) or `tsd` for type-level assertions.
- Avoid snapshot tests for anything other than serialized output formats.

## Concurrency

- Use `async`/`await` over raw Promise chains for readability.
- Never use `void` to fire-and-forget a promise without error handling.
- Use `AbortController` for cancellable async operations.
- Use `Promise.allSettled` for independent parallel operations that should not fail together.
- In Node.js, use `worker_threads` for CPU-bound work, not the main event loop.
- Flag: `await` inside a loop when `Promise.all` could parallelize the operations.
- Use `AsyncLocalStorage` for request-scoped context in Node.js, not global state.
- Rate-limit concurrent operations with a semaphore pattern or `p-limit`.

## Package/Module Structure

- Use path aliases in `tsconfig.json` (`@/` prefix) to avoid deep relative imports.
- Organize by feature/domain, not by technical layer (`components/`, `services/`, `utils/`).
- Export public API from feature index files; keep internal modules unexported.
- Use `package.json` `exports` field for dual ESM/CJS packages.
- Keep `tsconfig.json` strict: `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`.
- Separate build config (`tsconfig.build.json`) from IDE config (`tsconfig.json`).

## Performance Gotchas

- Avoid re-creating objects/arrays in React render paths; use `useMemo`/`useCallback` judiciously.
- Use `Map` and `Set` over plain objects for frequent insertion/deletion of dynamic keys.
- Avoid deep cloning with `structuredClone` or `JSON.parse(JSON.stringify(...))` in hot paths.
- Tree-shake by avoiding side effects in module scope and using `"sideEffects": false`.
- Use `WeakMap`/`WeakRef` for caches that should not prevent garbage collection.
- Prefer `for...of` over `.forEach()` for early exit capability with `break`.
- Lazy-import heavy modules with dynamic `import()` for code splitting.
- Avoid excessive type complexity (deeply recursive conditional types) that slows the compiler.
