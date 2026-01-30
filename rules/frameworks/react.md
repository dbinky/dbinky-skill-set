# React Review Rules

## Architecture Patterns

React follows a component-based architecture with unidirectional data flow. Components are the unit of composition; state flows down via props, events flow up via callbacks.

**Expect to see:**
- Functional components with hooks (not class components unless legacy codebase)
- Clear separation: presentational components vs. container/smart components
- Custom hooks encapsulating reusable stateful logic
- Co-located files: component, styles, tests, types in the same directory

**Flag in review:**
- Class components in new code (unless wrapping error boundaries)
- Business logic inside components — extract to hooks or utility modules
- Deeply nested component trees without composition (prop drilling > 3 levels)
- Components exceeding 200 lines — likely doing too much

## Common Anti-Patterns

- **Prop Drilling**: Passing props through 4+ levels. Use context, composition, or state management.
- **God Component**: A single component managing too many concerns. Decompose.
- **useEffect for Everything**: Using `useEffect` for derived state or synchronous computation. Use `useMemo` or compute inline.
- **Premature Abstraction**: Abstracting a component used once. Wait for the third use.
- **Index as Key**: Using array index as `key` in lists that reorder, filter, or mutate. Use stable IDs.
- **State Mirroring**: Copying props into state with `useState(props.value)`. Derive instead.
- **Barrel Files**: Re-exporting everything from `index.ts` — slows bundling and breaks tree shaking. Flag in large codebases.

## State Management

- **Local state**: `useState` for component-scoped state. Flag lifting state higher than necessary.
- **Shared state**: Context API for low-frequency updates (theme, auth, locale). Flag context for high-frequency updates (form inputs, animations).
- **Server state**: React Query / TanStack Query or SWR for API data. Flag manual `useEffect` + `useState` fetch patterns.
- **Complex state**: `useReducer` for state with multiple sub-values or complex transitions.
- **Global state**: Zustand, Jotai, or Redux Toolkit if needed. Flag vanilla Redux with manual action creators and reducers.
- Flag state stored in refs (`useRef`) when it should trigger re-renders.
- Flag `useState` with objects that are partially updated — spread bugs are common.

## Testing Patterns

- Use React Testing Library (not Enzyme). Flag Enzyme in new code.
- Test behavior, not implementation. Flag tests that assert on component internals or state values.
- Flag `container.querySelector` — use RTL queries (`getByRole`, `getByText`, `getByLabelText`).
- Flag snapshot tests as the primary test strategy — they are brittle and low-signal.
- Use `userEvent` over `fireEvent` for realistic interaction simulation.
- Flag tests without assertions — `render(<Component />)` alone is not a test.
- Mock API calls with MSW (Mock Service Worker), not manual fetch mocks.

## Performance

- **Unnecessary Re-renders**: Flag components re-rendering on every parent render without `React.memo` when they receive stable props.
- **Missing Keys**: Flag list rendering without keys or with index keys on dynamic lists.
- **Inline Functions in JSX**: Flag `onClick={() => doThing(id)}` in hot paths — creates new references every render. Use `useCallback` or extract.
- **Large Bundles**: Flag direct imports from large libraries (`import _ from 'lodash'`). Use specific imports (`import debounce from 'lodash/debounce'`).
- **Missing Code Splitting**: Flag routes without `React.lazy` / `Suspense` in SPAs with many routes.
- **useMemo/useCallback Overuse**: Flag memoization on cheap computations or primitives. Memoization has a cost.
- **Context Splitting**: Flag context providers with large value objects that change frequently — split into multiple contexts.

## Lifecycle (Hooks)

- `useEffect` cleanup functions must cancel subscriptions, timers, and async operations. Flag missing cleanups.
- Flag `useEffect` with missing dependencies — the exhaustive-deps lint rule should be enforced, not disabled.
- Flag `// eslint-disable-next-line react-hooks/exhaustive-deps` without a comment explaining why.
- Flag `useEffect` that runs on every render (empty effect doing side effects without dependency array).
- `useLayoutEffect` — flag unless measuring DOM or preventing visual flicker. It blocks paint.
- Flag `useEffect` chains that could be a single effect or an event handler.

## Configuration

- Environment variables must use `REACT_APP_` prefix (CRA) or `VITE_` prefix (Vite). Flag others — they won't be embedded.
- Flag runtime secrets in environment variables — client bundles expose them.
- Flag missing `.env.example` when `.env` files are gitignored.
- Flag hardcoded API URLs. Use environment variables.
- Flag missing `basename` in `BrowserRouter` for apps deployed to subpaths.

## Error Handling

- Use Error Boundaries (`componentDidCatch` / `ErrorBoundary`) at route and feature boundaries. Flag apps without them.
- Flag `catch` blocks that silently swallow errors in async operations.
- Flag missing loading and error states in data-fetching components.
- Flag `console.error` as the sole error handling strategy.
- Verify error boundaries display user-friendly fallback UI, not raw error messages.
- Flag unhandled promise rejections in event handlers.

## TypeScript Integration

- Flag `any` types — use `unknown` and narrow, or define proper types.
- Flag component props without TypeScript interfaces or type aliases.
- Flag type assertions (`as`) used to bypass type checking rather than fixing the type.
- Verify event handler types (`React.ChangeEvent<HTMLInputElement>`, etc.) are correct.
- Flag `@ts-ignore` without an accompanying explanation comment.
