# JavaScript Curation Rules

## Idiomatic Patterns

What senior JavaScript engineers write:

- **Optional chaining and nullish coalescing**: `obj?.prop ?? default` — not `obj && obj.prop || default`, which has falsy-value bugs and reads as an accident.
- **Destructuring**: Pull values out of objects and arrays at the call site. `const { name, email } = user` and `const [first, ...rest] = items` signal intentional access and remove repeated dotting.
- **Template literals**: `` `Hello, ${name}!` `` over `'Hello, ' + name + '!'`. Template literals compose; concatenation accretes.
- **Arrow functions with implicit return**: `items.map(x => x.id)` — not `items.map(function(x) { return x.id; })`. Reserve `function` for named declarations and methods that need their own `this`.
- **`const` by default, `let` only when needed, never `var`**: `const` communicates "this binding does not change." `var` brings function-scoped hoisting surprises that `let`/`const` eliminate.
- **`for...of` over indexed loops**: `for (const item of items)` — not `for (let i = 0; i < items.length; i++)`. Use indexed iteration only when the index is itself meaningful.
- **Object shorthand**: `{ name, email }` — not `{ name: name, email: email }`. The shorthand is not a convenience; it is the idiomatic form.
- **`Promise.all` for parallel I/O**: When two async operations are independent, run them in parallel. `const [user, org] = await Promise.all([fetchUser(id), fetchOrg(id)])`.
- **`Array.from` and spread operators**: `Array.from(nodeList)` and `[...set]` convert iterables without manual loops. `{ ...defaults, ...overrides }` merges objects without `Object.assign`.
- **ES modules (`import`/`export`) over CommonJS**: `import` and `export` are statically analyzable, tree-shakeable, and the platform standard. `require()` belongs only in scripts targeting environments without module support.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in JavaScript:

- **Callback pyramids and nested `.then()` chains**: Deeply nested `.then(result => { return foo(result).then(r2 => { return bar(r2).then(...) }) })` is async/await without the readability. Flatten with `async`/`await`.
- **Loose equality `==` confusion**: `==` coerces types in ways that surprise even experienced developers (`0 == false`, `'' == false`, `null == undefined`). Use `===` everywhere; switch lint enforcement on.
- **`var` declarations**: `var` is function-scoped and hoisted, making control-flow reasoning harder. There is no valid modern use case that `let` or `const` does not cover.
- **Unnecessary IIFE in module contexts**: `(function() { ... })()` exists to create a scope in script files. In ES modules every file already has its own scope; an IIFE is noise.
- **`arguments` object instead of rest parameters**: `function f() { const args = Array.from(arguments); }` — use `function f(...args)` instead. `arguments` is not available in arrow functions and breaks destructuring.
- **Manual `Promise` wrapping around existing Promises**: `return new Promise((resolve, reject) => { fetchData().then(resolve).catch(reject) })` — the inner Promise is already a Promise; just return it.
- **Redundant `return await` outside try/catch**: `return await somePromise()` resolves the Promise and then immediately re-wraps it for the caller. Bare `return somePromise()` is equivalent and avoids an extra microtask tick. (Inside `try/catch`, `return await` is intentional to catch rejections — that is the only valid use.)
- **Over-defensive checks**: `if (arr && arr.length && arr.length > 0)` → `if (arr?.length)`. Cascading guards that optional chaining eliminates in one expression.
- **`.forEach` for everything**: `.forEach` is a statement — it cannot be chained, returns nothing, and cannot short-circuit. Prefer `.map` when transforming, `.filter` when selecting, `.reduce` when accumulating, and `for...of` when you need early exit.
- **String concatenation in loops**: Building strings with `+=` inside a loop produces O(n²) allocations. Collect into an array and `.join('')`, or use a template literal outside the loop.

## Simplification Strategies

### 1. Replace callback chains with async/await and Promise.all

Before — nested `.then()` chains with sequential I/O that could be parallel:

```js
function loadDashboard(userId) {
  return fetchUser(userId)
    .then(user => {
      return fetchOrg(user.orgId)
        .then(org => {
          return fetchPermissions(user.id)
            .then(perms => {
              return { user, org, perms };
            });
        });
    });
}
```

After — flat async/await with parallel I/O where order does not matter:

```js
async function loadDashboard(userId) {
  const user = await fetchUser(userId);
  const [org, perms] = await Promise.all([
    fetchOrg(user.orgId),
    fetchPermissions(user.id),
  ]);
  return { user, org, perms };
}
```

### 2. Replace manual null coalescing with optional chaining and nullish coalescing

Before — cascading existence checks that miss falsy-but-valid values:

```js
function getDisplayName(user) {
  if (user && user.profile && user.profile.displayName) {
    return user.profile.displayName;
  } else if (user && user.name) {
    return user.name;
  } else {
    return 'Anonymous';
  }
}
```

After — optional chaining with nullish coalescing:

```js
function getDisplayName(user) {
  return user?.profile?.displayName ?? user?.name ?? 'Anonymous';
}
```

### 3. Replace verbose conditionals with guard clauses

Before — deep nesting that makes the happy path hard to follow:

```js
async function processOrder(order) {
  if (order) {
    if (order.items && order.items.length > 0) {
      if (order.status === 'pending') {
        const result = await chargeCard(order);
        if (result.success) {
          await fulfillOrder(order);
          return { ok: true };
        } else {
          return { ok: false, reason: 'payment_failed' };
        }
      } else {
        return { ok: false, reason: 'invalid_status' };
      }
    } else {
      return { ok: false, reason: 'empty_order' };
    }
  } else {
    return { ok: false, reason: 'missing_order' };
  }
}
```

After — guard clauses that exit early and keep the success path linear:

```js
async function processOrder(order) {
  if (!order) return { ok: false, reason: 'missing_order' };
  if (!order.items?.length) return { ok: false, reason: 'empty_order' };
  if (order.status !== 'pending') return { ok: false, reason: 'invalid_status' };

  const result = await chargeCard(order);
  if (!result.success) return { ok: false, reason: 'payment_failed' };

  await fulfillOrder(order);
  return { ok: true };
}
```

## Dead Code Signals

JavaScript-specific indicators that code is no longer exercised or referenced:

- **Unused exports**: There is no compiler warning for exported names with no importers. Look for named exports that appear in no `import` statement across the codebase, especially after module refactors.
- **Unused `package.json` dependencies**: Packages listed in `dependencies` or `devDependencies` that are not imported anywhere in source or config files. Common after swapping libraries without removing the old entry.
- **Dead event listeners**: `addEventListener` calls whose corresponding `removeEventListener` was dropped when the component or view was refactored, and whose handler function has no other callers.
- **Unreachable code after returns**: Statements placed after a `return`, `throw`, or `break` that can never execute. Linters catch most cases, but they slip through in complex control flow.
- **Commented-out `require`/`import`**: Lines left as `// import { foo } from './foo'` are either dead (remove them) or temporarily disabled (a code smell that should become a flag or branch).
- **Unused Express/Koa middleware**: `app.use(someMiddleware())` registrations whose functionality was replaced — old body parsers, removed auth layers, deprecated request loggers — that still run on every request and add latency for no benefit.
