# Angular Review Rules

## Architecture Patterns

Angular follows a module-based architecture with components, services, and dependency injection as core building blocks. It is opinionated about structure and enforces patterns through its CLI and compiler.

**Expect to see:**
- Feature modules grouping related components, services, pipes, and directives
- Standalone components (Angular 14+) as the modern alternative to NgModules
- Services decorated with `@Injectable({ providedIn: 'root' })` for singletons
- Smart (container) components and dumb (presentational) components separation
- Angular CLI-generated structure with consistent naming conventions

**Flag in review:**
- Business logic in components — extract to services
- Components with 300+ lines — decompose
- Direct DOM manipulation (`document.querySelector`, `ElementRef.nativeElement`) — use Angular APIs (Renderer2, template refs)
- Missing `OnPush` change detection on presentational components
- `any` types in templates (disable `strictTemplates` to allow this is a red flag)

## Common Anti-Patterns

- **Subscription Leaks**: Subscribing to Observables without unsubscribing. Use `async` pipe, `takeUntilDestroyed()`, or `DestroyRef`.
- **Manual Change Detection**: Calling `ChangeDetectorRef.detectChanges()` frequently — usually indicates a design problem.
- **Oversized Modules**: A single module with 20+ declarations. Split into feature modules.
- **Service in Component Providers**: Providing a service in a component when it should be a singleton. Flag unless intentionally scoped.
- **Nested Subscriptions**: `subscribe()` inside `subscribe()`. Use RxJS operators (`switchMap`, `mergeMap`, `concatMap`).
- **String-Based Queries**: `querySelector` inside components. Use `@ViewChild` / `@ContentChild`.

## State Management

- **Simple state**: Services with `BehaviorSubject` or Angular signals for local feature state.
- **Signals**: Angular 16+ signals (`signal()`, `computed()`, `effect()`) are the preferred reactive primitive. Flag new code using BehaviorSubject where signals suffice.
- **Complex state**: NgRx (Redux pattern) or NGXS for enterprise-scale state. Flag if introduced for simple CRUD.
- **Server state**: Flag manual HTTP caching. Use NgRx ComponentStore, query libraries, or caching interceptors.
- Flag mutable state modifications — prefer immutable updates, especially with `OnPush`.
- Flag `@Input()` properties mutated inside child components. Inputs should be treated as read-only.

## Testing Patterns

- Use `TestBed` for integration tests. Flag `TestBed` for pure service unit tests — instantiate directly.
- Use `ComponentFixture` and `DebugElement` for component tests.
- Flag tests that access `nativeElement` directly when `DebugElement` queries work.
- Use `HttpClientTestingModule` and `HttpTestingController` for HTTP tests. Flag real HTTP calls in tests.
- Flag tests without `fixture.detectChanges()` — the template won't render.
- Use `fakeAsync` / `tick` for async tests. Flag `setTimeout` in tests for synchronization.
- Marble testing for complex Observable pipelines.

## Performance

- **Change Detection**: Use `OnPush` on all presentational components. Flag `Default` change detection on leaf components.
- **Lazy Loading**: Flag feature modules loaded eagerly that are not on the critical path. Use `loadChildren` or `loadComponent`.
- **TrackBy**: Flag `*ngFor` without `trackBy` on lists that update. Without it, Angular re-creates DOM elements.
- **Pure Pipes**: Use pure pipes for computed template values. Flag method calls in templates (`{{ getTotal() }}`) — they run on every change detection cycle.
- **Bundle Size**: Flag imports from the root of large libraries. Use deep imports or tree-shakable APIs.
- **Zone.js**: Flag unnecessary async operations inside the Angular zone. Use `NgZone.runOutsideAngular()` for animations, scroll handlers, and timers that don't update UI.
- **Signals in Templates**: Prefer signals over Observables in templates to avoid `async` pipe overhead.

## Lifecycle

- `ngOnInit` — initialization logic. Flag HTTP calls in constructors; use `ngOnInit` or resolvers.
- `ngOnDestroy` — cleanup subscriptions, timers, event listeners. Flag missing cleanup.
- `ngOnChanges` — flag complex logic here. Prefer signals or setter-based inputs.
- `ngAfterViewInit` — DOM access. Flag DOM access in `ngOnInit` (view not ready).
- `DestroyRef` (Angular 16+) — prefer `takeUntilDestroyed()` over manual `ngOnDestroy` unsubscription.
- Flag lifecycle hooks on services — only use them with `DestroyRef` injection.

## Configuration

- Use `environment.ts` / `environment.prod.ts` for build-time config. Flag runtime secrets here — they are bundled.
- Use `APP_INITIALIZER` for runtime configuration fetched at startup.
- Flag hardcoded API URLs in services. Inject base URLs via `InjectionToken`.
- Flag `angular.json` budgets missing or set too high — enforce bundle size limits.
- Flag disabled `strict` mode in `tsconfig.json`. Strict templates and null checks prevent bugs.

## Error Handling

- Use a global `ErrorHandler` implementation. Flag unhandled errors in components.
- Use HTTP interceptors for global error handling (401 redirects, retry logic, error normalization).
- Flag `.subscribe()` without an error callback or `catchError` operator.
- Flag `catchError` that returns `EMPTY` without logging — silently swallowed errors.
- Verify user-facing error messages are not raw server responses.
- Flag error handling that differs across features — centralize.

## RxJS Patterns

- Flag `subscribe()` in components when the `async` pipe works. The `async` pipe auto-unsubscribes.
- Flag `mergeMap` where `switchMap` is intended (HTTP requests should typically cancel previous).
- Flag `tap` with side effects that modify component state in complex ways — hard to debug.
- Flag deeply nested `pipe` chains (5+ operators). Extract into named functions.
- Flag `toPromise()` / `firstValueFrom()` used to avoid learning RxJS. Acceptable for one-shot operations.
- Flag `shareReplay` without `{ refCount: true }` — it retains subscriptions indefinitely.
