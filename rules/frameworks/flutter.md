# Flutter Review Rules

## Architecture Patterns

Flutter uses a widget-based declarative UI model. Everything is a widget. The framework rebuilds the widget tree on state changes and efficiently diffs against the element tree.

**Expect to see:**
- Clear separation of UI (widgets), business logic (blocs/cubits/providers/riverpod), and data layer (repositories, data sources)
- Feature-first or layer-first folder organization, consistently applied
- Immutable state objects
- Composition over inheritance for widgets

**Flag in review:**
- Business logic inside `build()` methods — extract to state management layer
- Direct HTTP or database calls in widgets
- Deep widget nesting (10+ levels in a single file) without extraction
- Stateful widgets where stateless widgets suffice
- Inheritance-based widget reuse (`extends MyBaseWidget`) — prefer composition

## Common Anti-Patterns

- **God Widget**: Single widget file with 500+ lines. Decompose into smaller widgets.
- **setState Everywhere**: Using `setState` for complex state or state shared across widgets. Use proper state management.
- **Build Method Side Effects**: Triggering network calls, writes, or navigation in `build()`. The build method must be pure.
- **Mutable State Objects**: State classes with mutable fields. Prefer `@immutable` classes with `copyWith`.
- **String Typing**: Using raw strings for routes, keys, or asset paths. Use constants or code generation.
- **Ignoring Constraints**: Not understanding Flutter's layout protocol (constraints go down, sizes go up). Flag `UnconstrainedBox` and `OverflowBox` without justification.

## State Management

- **Local state**: `setState` for single-widget ephemeral state (animations, form field focus). Flag for anything shared.
- **Riverpod**: Preferred for dependency injection and reactive state. Flag `Provider` (legacy package) in new code.
- **BLoC/Cubit**: Event-driven state management. Flag BLoCs for trivial state (a boolean toggle does not need events).
- **Provider**: Acceptable in existing codebases. Flag `ChangeNotifier` with 10+ fields — split into multiple notifiers.
- Flag state management solutions mixed inconsistently in the same app without justification.
- Flag global mutable singletons for state. Use the chosen state management solution.

## Testing Patterns

- **Widget tests**: Use `testWidgets` and `WidgetTester`. Flag tests that only test logic and could be unit tests.
- **Unit tests**: Pure Dart tests for business logic, blocs, repositories. Flag untested business logic.
- **Integration tests**: `integration_test` package for end-to-end flows. Flag if no integration tests exist for critical paths.
- **Golden tests**: Screenshot comparison for complex UI. Flag if used for simple layouts (brittle).
- Flag `find.byType` for widgets that appear multiple times — use `find.byKey` with `ValueKey`.
- Flag tests missing `pumpAndSettle()` or `pump()` — async operations won't complete.
- Mock HTTP with `http_mock_adapter` or `mockito`. Flag real network calls in tests.

## Performance

- **Rebuilds**: Flag widgets that rebuild the entire tree on state change. Use `const` constructors, `Selector`, or scoped state.
- **const Constructors**: Flag widget constructors that could be `const` but aren't — `const` widgets skip rebuilds.
- **ListView.builder**: Flag `ListView(children: [...])` for long or dynamic lists. Use `ListView.builder` for lazy construction.
- **Image Caching**: Flag `Image.network` without `cacheWidth`/`cacheHeight` for large images. Use `cached_network_image`.
- **Opacity**: Flag `Opacity` widget for hiding elements. Use `Visibility` or conditional rendering — `Opacity` still lays out and paints.
- **RepaintBoundary**: Flag complex custom painters without `RepaintBoundary` isolation.
- **Shader Compilation Jank**: Flag first-frame animations on complex widgets. Use `SkSL` warmup or simplify.
- Flag `MediaQuery.of(context)` in deeply nested widgets — it triggers rebuilds on every metric change. Use `MediaQuery.sizeOf(context)` (Flutter 3.10+).

## Lifecycle

- `initState()` — one-time setup. Flag async work here; use `didChangeDependencies` or trigger from state management.
- `dispose()` — cancel controllers, subscriptions, animation controllers. Flag missing `dispose` on `TextEditingController`, `ScrollController`, `AnimationController`.
- `didChangeDependencies()` — called when inherited widgets change. Flag heavy logic here without guards.
- `didUpdateWidget()` — flag equality checks missing when updating controllers based on new widget config.
- Flag `GlobalKey` usage — it's expensive and prevents widget reuse. Use `ValueKey` or `ObjectKey`.

## Configuration

- Use `--dart-define` or `--dart-define-from-file` for build-time configuration. Flag hardcoded URLs.
- Use flavors/schemes for environment-specific builds (dev, staging, prod).
- Flag API keys committed in Dart source files — use `String.fromEnvironment` or secure build injection.
- Use `pubspec.yaml` assets section for bundled files. Flag hardcoded asset path strings — use generated constants (`flutter_gen`).
- Flag `minSdkVersion` below 21 (Android) or iOS deployment target below 12 without strong justification.

## Error Handling

- Use `FlutterError.onError` for framework errors and `PlatformDispatcher.instance.onError` for async errors. Flag apps without global error handlers.
- Flag empty `catch` blocks. At minimum log the error.
- Flag `catchError` on Futures — prefer `try/catch` in `async` functions for readability.
- Use `Result` types or sealed classes for expected error cases. Flag stringly-typed errors.
- Flag `print()` for error logging — use a logging package (`logger`, `logging`).
- Verify error boundaries exist: `ErrorWidget.builder` for custom error UI in production.

## Platform Integration

- Flag platform channel code (`MethodChannel`) without error handling — native code can throw.
- Flag platform-specific code without corresponding implementations for all supported platforms.
- Verify `PlatformException` is caught when invoking native methods.
- Flag `dart:io` usage in packages intended to be platform-agnostic.
- Use `defaultTargetPlatform` for platform checks, not `Platform.isAndroid` (fails on web).
