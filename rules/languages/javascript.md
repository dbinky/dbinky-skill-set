# JavaScript Review Rules

## Idiomatic Patterns

- Use `const` by default; use `let` only when reassignment is necessary. Never use `var`.
- Use arrow functions for callbacks and short expressions; use `function` declarations for top-level named functions.
- Use destructuring for extracting properties from objects and arrays.
- Use optional chaining (`?.`) and nullish coalescing (`??`) over manual null checks.
- Use template literals over string concatenation.
- Use `Array.from()`, spread syntax, and `Object.entries/keys/values` for data transformation.
- Use `class` syntax with `#private` fields for encapsulation in OOP code.
- Use ESM (`import`/`export`) over CommonJS (`require`/`module.exports`) in new code.

## Common Anti-Patterns

- Flag: `==` or `!=` instead of `===`/`!==` (loose equality causes type coercion bugs).
- Flag: `var` declarations anywhere in the codebase.
- Flag: nested ternaries deeper than one level; use `if`/`else` or early returns.
- Flag: `arguments` object usage; use rest parameters (`...args`).
- Flag: `for...in` on arrays (iterates keys, not values); use `for...of` or array methods.
- Flag: modifying function parameters directly; treat them as immutable.
- Flag: `typeof x === "undefined"` checks that could use optional chaining or default values.
- Flag: `new Array(n)` for array creation; use `Array.from({ length: n })` with a mapper.

## Error Handling

- Wrap async operations in try/catch with specific error handling.
- Always attach `.catch()` to floating promises or use try/catch with await.
- Create custom error classes with `name`, `message`, and `code` properties.
- Never throw non-Error objects (strings, numbers).
- Use `process.on('unhandledRejection')` and `process.on('uncaughtException')` as safety nets, not primary handlers.
- Return error states from functions instead of relying on exceptions for control flow.

## Naming Conventions

- `camelCase` for variables, functions, methods. `PascalCase` for classes and constructor functions.
- `UPPER_SNAKE_CASE` for true constants and configuration values.
- Boolean variables: prefix with `is`, `has`, `can`, `should`, `will`.
- Event handlers: prefix with `handle` (`handleClick`) or `on` (`onClick`).
- File names: `kebab-case.js` for modules, `PascalCase.jsx` for React components.
- Private methods/properties: use `#` prefix (ES2022+), not underscore convention.

## Testing Patterns

- Use Vitest or Jest. Avoid Mocha/Chai in new projects unless ecosystem-specific.
- Structure tests as `describe` > `it` blocks with clear behavioral descriptions.
- Test behavior, not implementation. Avoid testing private methods directly.
- Use `@testing-library` for DOM testing; query by role, label, or text, not CSS selectors.
- Use `msw` (Mock Service Worker) for API mocking over custom fetch mocks.
- Avoid excessive mocking; prefer integration tests with real dependencies where practical.
- Use `beforeEach` for test isolation; avoid shared mutable state across tests.
- Assert on one concept per test; multiple assertions are fine if they verify one behavior.

## Concurrency

- Use `async`/`await` over raw `.then()` chains.
- Use `Promise.all` for independent parallel operations; `Promise.allSettled` when partial failure is acceptable.
- Never use `await` in a loop when operations are independent; batch with `Promise.all`.
- Use `AbortController` to cancel fetch requests and other async operations.
- In Node.js, offload CPU-intensive work to `worker_threads`, not the event loop.
- Use `queueMicrotask` for deferring work to the microtask queue, not `setTimeout(fn, 0)`.
- Rate-limit concurrent operations with `p-limit` or a custom semaphore.
- Beware: `async` callbacks in `forEach` run concurrently but are not awaited.

## Package/Module Structure

- Use ESM (`import`/`export`) with `.js` extensions in Node.js or bundler path resolution.
- Organize by feature/domain, not by file type (`controllers/`, `models/`, `views/`).
- Keep `index.js` files thin; re-export public API only.
- Use `package.json` `"type": "module"` for ESM projects.
- Configure `"exports"` field in `package.json` for library packages.
- Separate runtime dependencies from dev dependencies in `package.json`.

## Performance Gotchas

- Avoid synchronous operations in Node.js event loop: `fs.readFileSync`, `crypto.pbkdf2Sync`, etc.
- Use `Map` and `Set` over plain objects for dynamic key collections.
- Avoid creating closures in tight loops; hoist function definitions outside.
- Use `requestAnimationFrame` for DOM updates; batch reads and writes to avoid layout thrashing.
- Lazy-load modules with dynamic `import()` for code splitting.
- Use `structuredClone` for deep copies; avoid `JSON.parse(JSON.stringify())` which drops functions and special types.
- Avoid memory leaks: clean up event listeners, intervals, and subscriptions on teardown.
- Use `WeakRef` and `FinalizationRegistry` for cache patterns that should not prevent GC.
