# Dart Review Rules

## Idiomatic Patterns

- Use `final` for variables that are assigned once. Use `const` for compile-time constants.
- Use null safety fully: declare non-nullable types by default, use `?` only when absence is meaningful.
- Use `named parameters` with `required` keyword for functions with more than 2-3 parameters.
- Use `cascade notation` (`..`) for chaining method calls on the same object.
- Use `extension` methods to add functionality to existing types without inheritance.
- Use `sealed` classes (Dart 3+) for exhaustive pattern matching on type hierarchies.
- Use `pattern matching` and `switch` expressions (Dart 3+) over `if`/`else` chains.
- Use `records` for lightweight anonymous data grouping (Dart 3+).

## Common Anti-Patterns

- Flag: `dynamic` type used where a specific type or generic would suffice.
- Flag: `!` (null assertion operator) without clear justification; prefer null-safe alternatives.
- Flag: `late` variables that could be initialized in the constructor or made nullable.
- Flag: deeply nested `FutureBuilder`/`StreamBuilder` widgets; extract into separate widgets or use state management.
- Flag: business logic in `build()` methods; extract to separate classes or providers.
- Flag: `setState()` in complex widgets; use Riverpod, Bloc, or other state management.
- Flag: mutable global variables; use dependency injection or scoped providers.
- Flag: `print()` statements in production code; use a logging framework.

## Error Handling

- Use specific exception types; define custom exceptions for domain errors.
- Catch specific exceptions: `on FormatException catch (e)`, not bare `catch (e)`.
- Use `Result` pattern (custom sealed class) for operations that fail predictably.
- Never catch `Error` subtypes (`StackOverflowError`, `OutOfMemoryError`) — those are programmer bugs.
- Use `Zone.current.handleUncaughtError` or `FlutterError.onError` for global error handling in Flutter.
- Return `Future.error` or throw in async functions; never return `null` to indicate failure.

## Naming Conventions

- `lowerCamelCase` for variables, functions, parameters, named parameters.
- `UpperCamelCase` for classes, enums, typedefs, extensions, mixins.
- `lowercase_with_underscores` for libraries, packages, directories, source files.
- `_leadingUnderscore` for library-private members (privacy is per-library, not per-class).
- Boolean variables: use `is`, `has`, `can`, `should` prefixes.
- Constants: `lowerCamelCase` (Dart convention), not `UPPER_SNAKE_CASE`.
- Avoid abbreviations: `buttonController` not `btnCtrl`.

## Testing Patterns

- Use `test` package for unit tests, `flutter_test` for widget tests.
- Use `group()` and `test()` with descriptive human-readable names.
- Use `expect()` with matchers: `expect(result, equals(42))`, `expect(fn, throwsA(isA<MyException>()))`.
- Use `mockito` (with `build_runner` code generation) or `mocktail` for mocking.
- Widget tests: use `tester.pumpWidget()`, `find.byType()`, `find.text()`, `tester.tap()`.
- Use `setUp()` and `tearDown()` for test lifecycle management.
- Integration tests: use `integration_test` package with `IntegrationTestWidgetsFlutterBinding`.
- Golden tests: use `matchesGoldenFile` for UI regression testing.

## Concurrency

- Use `async`/`await` for all asynchronous operations. Dart is single-threaded on the event loop.
- Use `Stream` and `StreamController` for event-driven data flows.
- Use `Isolate` or `compute()` in Flutter for CPU-intensive work off the main thread.
- Never perform blocking operations on the main isolate; it freezes the UI.
- Use `Completer<T>` to bridge callback-based APIs to `Future`-based APIs.
- Use `StreamSubscription.cancel()` to prevent memory leaks; always cancel in `dispose()`.
- Use `debounce` and `throttle` (via `rxdart` or custom) for high-frequency event streams.
- Use `FutureGroup` or `Future.wait` for parallel async operations.

## Package/Module Structure

- Follow the `lib/src/` convention: public API in `lib/`, implementation in `lib/src/`.
- Export public API from a single barrel file: `lib/my_package.dart`.
- Use `part`/`part of` sparingly; prefer separate library files with imports.
- Feature-first organization in Flutter: `lib/features/auth/`, `lib/features/home/`.
- Separate data, domain, and presentation layers within each feature.
- Use `pubspec.yaml` with version constraints (`^1.0.0`). Pin with `pubspec.lock` for apps; not for packages.

## Performance Gotchas

- Use `const` constructors for widgets that never change; enables Flutter's widget caching.
- Avoid rebuilding large widget subtrees; use `const` widgets, `RepaintBoundary`, and selective state management.
- Use `ListView.builder` over `ListView(children: [...])` for long scrollable lists.
- Minimize `setState` scope; rebuild only the widgets that actually changed.
- Use `compute()` for JSON parsing of large payloads to avoid jank on the main isolate.
- Avoid `Opacity` widget for hiding elements; use `Visibility` or conditional rendering.
- Use `cached_network_image` for network images; never load unbounded images directly.
- Profile with Flutter DevTools: check for unnecessary rebuilds, jank, and memory leaks.
