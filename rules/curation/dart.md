# Dart Curation Rules

## Idiomatic Patterns

What senior Dart engineers write:

- **`final` by default, `const` where possible**: Local variables are `final` unless reassignment is needed. Widget constructors, collection literals, and compile-time values use `const`. This signals intent and enables Flutter's widget caching.
- **Null safety without escape hatches**: Declare non-nullable types by default. Use `?` only when absence is semantically meaningful. Avoid `!` (bang operator) — if you need it, the type model is wrong or the code needs restructuring.
- **Named parameters with `required`**: Functions with more than two parameters use named parameters. Mark non-optional ones `required` so call sites are self-documenting: `User({required this.name, required this.email, this.avatarUrl})`.
- **Cascade notation**: Chain mutations on a single object with `..` instead of repeating the variable name. `list..add(1)..add(2)..sort()` reads as a single conceptual operation.
- **Pattern matching and switch expressions (Dart 3+)**: Use `switch` expressions with pattern matching over `if`/`else` chains. Sealed class hierarchies + exhaustive switches replace the visitor pattern.
- **Records for lightweight grouping**: Return `(int, String)` or `({int id, String name})` instead of single-use classes or lists-of-dynamic for multi-value returns.
- **Extension methods**: Add behavior to existing types without inheritance. `extension StringValidation on String { bool get isEmail => contains('@'); }` keeps the type closed and the extension discoverable.
- **Sealed classes for closed hierarchies**: Model finite variants as `sealed class Result { }` with `final class Success extends Result` and `final class Failure extends Result`. The compiler enforces exhaustive handling.
- **`const` constructors on immutable widgets**: Every stateless widget whose fields are all final should have a `const` constructor. This is the single highest-leverage Flutter optimization.
- **Tear-offs over lambdas**: Pass `list.add` instead of `(x) => list.add(x)`. Tear-offs are clearer, avoid an extra closure allocation, and compose with higher-order functions.

## Vibe-Code Anti-Patterns

Common patterns LLM-assisted coding produces in Dart:

- **`dynamic` everywhere**: Using `dynamic` to skip type declarations defeats Dart's sound type system. Every `dynamic` is an unchecked cast that defers failure to runtime.
- **Bang operator (`!`) as a type-system silencer**: `user!.name` asserts non-null without proving it. A runtime null produces a confusing `Null check operator used on a null value` error far from the cause. Restructure the code so the type is non-nullable, or handle the null case explicitly.
- **`late` for laziness, not necessity**: `late final _controller = TextEditingController()` is fine. `late String name;` assigned conditionally across multiple methods is a deferred `LateInitializationError` waiting to happen. Prefer nullable-with-check or constructor initialization.
- **Business logic in `build()` methods**: Filtering lists, formatting dates, or computing derived state inside `build()` re-executes on every frame. Extract to the state layer or a computed getter.
- **`setState()` in complex widgets**: `setState` rebuilds the entire widget. In widgets with multiple independent pieces of state, this triggers unnecessary rebuilds. Use Riverpod, Bloc, or `ValueNotifier` for granular reactivity.
- **Mutable global variables**: Top-level `var currentUser = ...` creates implicit coupling and makes testing impossible. Use dependency injection or a scoped provider.
- **`print()` in production code**: `print()` is synchronous, unstructured, and has no log levels. Use `package:logging` or the framework's logger.
- **Catching `Exception` or `Object` generically**: `catch (e) { /* ignore */ }` swallows errors silently. Catch specific types: `on FormatException catch (e)`.
- **Deeply nested widget trees without extraction**: A single `build()` method with 10+ levels of nesting is unreadable and untestable. Extract subtrees into named widgets or helper methods.
- **Unused `async`**: An `async` function with no `await` wraps the return in an unnecessary `Future` and adds a microtask hop. Remove `async` or add the missing `await`.
- **Stringly-typed maps**: `Map<String, dynamic>` used as a data container instead of a typed class. This trades compile-time safety for runtime key-typo bugs.
- **Over-importing `dart:mirrors` or relying on reflection**: Dart AOT compilation (Flutter, server) does not support `dart:mirrors`. Code that depends on reflection breaks silently when compiled ahead-of-time.

## Simplification Strategies

### 1. Replace `dynamic` maps with typed classes

Before — stringly-typed map propagates through the app:

```dart
Map<String, dynamic> fetchUser() {
  return {
    'id': 42,
    'name': 'Alice',
    'email': 'alice@example.com',
    'isActive': true,
  };
}

void greet(Map<String, dynamic> user) {
  final name = user['name'] as String;
  final active = user['isActive'] as bool;
  if (active) print('Hello, $name');
}
```

After — typed class with factory constructor:

```dart
class User {
  final int id;
  final String name;
  final String email;
  final bool isActive;

  const User({
    required this.id,
    required this.name,
    required this.email,
    required this.isActive,
  });

  factory User.fromJson(Map<String, dynamic> json) => User(
        id: json['id'] as int,
        name: json['name'] as String,
        email: json['email'] as String,
        isActive: json['isActive'] as bool,
      );
}

void greet(User user) {
  if (user.isActive) print('Hello, ${user.name}');
}
```

### 2. Extract nested widget trees into named widgets

Before — deeply nested build method:

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(title: const Text('Profile')),
    body: Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        children: [
          Row(
            children: [
              CircleAvatar(
                radius: 40,
                backgroundImage: NetworkImage(user.avatarUrl),
              ),
              const SizedBox(width: 16),
              Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(user.name, style: Theme.of(context).textTheme.headlineSmall),
                  Text(user.email, style: Theme.of(context).textTheme.bodyMedium),
                ],
              ),
            ],
          ),
          const SizedBox(height: 24),
          ElevatedButton(
            onPressed: () => _editProfile(),
            child: const Text('Edit Profile'),
          ),
        ],
      ),
    ),
  );
}
```

After — extracted into focused, reusable widgets:

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(title: const Text('Profile')),
    body: Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        children: [
          ProfileHeader(user: user),
          const SizedBox(height: 24),
          ElevatedButton(
            onPressed: _editProfile,
            child: const Text('Edit Profile'),
          ),
        ],
      ),
    ),
  );
}

class ProfileHeader extends StatelessWidget {
  final User user;
  const ProfileHeader({super.key, required this.user});

  @override
  Widget build(BuildContext context) {
    final textTheme = Theme.of(context).textTheme;
    return Row(
      children: [
        CircleAvatar(radius: 40, backgroundImage: NetworkImage(user.avatarUrl)),
        const SizedBox(width: 16),
        Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(user.name, style: textTheme.headlineSmall),
            Text(user.email, style: textTheme.bodyMedium),
          ],
        ),
      ],
    );
  }
}
```

### 3. Replace if/else chains with switch expressions

Before — sequential type checks with casting:

```dart
String describe(Shape shape) {
  if (shape is Circle) {
    return 'Circle with radius ${shape.radius}';
  } else if (shape is Rectangle) {
    return 'Rectangle ${shape.width}x${shape.height}';
  } else if (shape is Triangle) {
    return 'Triangle with base ${shape.base}';
  } else {
    return 'Unknown shape';
  }
}
```

After — exhaustive switch expression on a sealed hierarchy:

```dart
String describe(Shape shape) => switch (shape) {
      Circle(:final radius) => 'Circle with radius $radius',
      Rectangle(:final width, :final height) => 'Rectangle ${width}x$height',
      Triangle(:final base) => 'Triangle with base $base',
    };
```

### 4. Replace bang operators with null-safe restructuring

Before — bang operators papering over nullable types:

```dart
class OrderService {
  User? _currentUser;

  Future<Order> placeOrder(List<Item> items) async {
    final userId = _currentUser!.id;
    final address = _currentUser!.address!;
    final order = await _api.createOrder(userId, address, items);
    return order;
  }
}
```

After — require non-null at the boundary, use non-nullable types downstream:

```dart
class OrderService {
  final User _currentUser;

  OrderService({required User currentUser}) : _currentUser = currentUser;

  Future<Order> placeOrder(List<Item> items, {required Address shippingAddress}) async {
    return _api.createOrder(_currentUser.id, shippingAddress, items);
  }
}
```

## Dead Code Signals

Dart-specific indicators that code is no longer exercised or referenced:

- **Unused imports**: The Dart analyzer warns about these, but they survive when analyzer warnings are ignored or suppressed. Look for `import` statements flagged by `dart analyze` or with `// ignore: unused_import`.
- **Unreferenced public API in packages**: Exported symbols in `lib/src/` that no barrel file re-exports and no internal file imports. Common after API surface refactors.
- **Unused `part` files**: `part 'old_feature.dart'` directives left in a library file after the functionality moved to a separate library or was deleted.
- **Orphaned generated code**: `.g.dart`, `.freezed.dart`, or `.mocks.dart` files whose source annotation (`@JsonSerializable`, `@freezed`, `@GenerateMocks`) was removed. The generated file persists because `build_runner` does not auto-delete.
- **Dead route definitions**: Named routes in a `MaterialApp` route table (`'/old-screen': (context) => OldScreen()`) that no `Navigator.pushNamed` call references.
- **Unused `pubspec.yaml` dependencies**: Packages listed in `dependencies` or `dev_dependencies` that nothing in `lib/` or `test/` imports. Frequent after swapping packages (e.g., `http` to `dio`).
- **Stale platform channel handlers**: `MethodChannel` handlers registered in Dart for native method names that no longer exist in the iOS/Android native code, or vice versa.
- **Unused mixin declarations**: `mixin OldBehavior on StatelessWidget` that no class applies with `with OldBehavior`. Mixins are easy to forget because they have no constructor to trace.
