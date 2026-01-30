# Dart Security Review Rules

## Input Validation

- Validate all form inputs with `TextFormField` validators and server-side validation.
- Use typed model classes with `fromJson` factories that validate fields, not raw `Map<String, dynamic>` access.
- Validate URL inputs with `Uri.tryParse` and check scheme/host before making requests.
- Limit text input length with `maxLength` on `TextField` and server-side size limits.
- For Shelf/Dart Frog backends: validate request bodies against typed DTOs, not raw JSON maps.
- Sanitize HTML content with a sanitizer before rendering in `WebView` or `Html` widgets.

## Authentication/Authorization

- Use `firebase_auth`, `appauth`, or established OAuth libraries. Never implement auth protocols manually.
- Store tokens securely with `flutter_secure_storage` (uses Keychain/Keystore), never `SharedPreferences`.
- Flag: JWT tokens or API keys stored in `SharedPreferences` — they are not encrypted.
- Implement token refresh logic; never store long-lived tokens client-side.
- Use certificate pinning with `http` package or `dio` for sensitive API communication.
- Check authorization on every API call server-side; never trust client-side role checks alone.

## SQL/NoSQL Injection

- SQLite (`sqflite`): use parameterized queries with `?` placeholders: `db.query('t', where: 'id = ?', whereArgs: [id])`.
- Flag: string interpolation in SQL: `db.rawQuery('SELECT * FROM t WHERE id = $id')`.
- For Drift (Moor): use the type-safe query builder; avoid `customSelect` with raw strings.
- For Firestore: rules are enforced server-side; never trust client-side query construction for security.
- Validate all data written to local databases; malicious input can corrupt local state.

## XSS Prevention

- Never load user-controlled URLs in `WebView` without validation and allowlisting.
- Use `flutter_html` or `flutter_widget_from_html` with sanitization for rendering user HTML.
- Set `javascriptMode: JavascriptMode.disabled` on `WebView` unless JavaScript is required.
- For web Flutter targets: apply CSP headers server-side. Flutter web renders to canvas but HTML mode has XSS risks.
- Flag: `dart:html` `Element.innerHtml` set with user-controlled content in Flutter web.

## Secrets Management

- Never hardcode API keys, client secrets, or credentials in Dart source files.
- Use `--dart-define` or `--dart-define-from-file` for build-time configuration.
- Flag: string literals matching API key, token, or secret patterns in source code.
- Store runtime secrets in environment variables on the server; use `flutter_secure_storage` on device.
- Never commit `.env` files or `firebase_options.dart` with production keys to source control.
- Use `String.fromEnvironment` for compile-time constants; they are embedded in the binary (acceptable for public config only).

## Dependency Vulnerabilities

- Run `dart pub outdated` regularly; review changelogs before upgrading.
- Review `pubspec.lock` diffs in PRs for unexpected dependency additions.
- Check pub.dev package scores, verification badges, and publisher trust.
- Avoid packages with no recent updates, low pub points, or unverified publishers.
- Flag: packages that require excessive permissions (platform channels) without justification.
- Use `dart pub deps` to audit the full transitive dependency tree.

## Deserialization

- Use `json_serializable` with `@JsonSerializable(checked: true)` for strict deserialization.
- Flag: `jsonDecode(input) as Map<String, dynamic>` without subsequent typed deserialization.
- Never use `dart:mirrors` in production code; it is unavailable in AOT and a reflection-based attack surface.
- Validate all fields after deserialization; null-safe types help but do not validate ranges or formats.
- Handle `FormatException` and `TypeError` from deserialization gracefully; never let them crash the app.

## File System Access

- Use `path` package for path manipulation. Validate paths stay within the app's document directory.
- Flag: `File(userInput)` without path validation.
- Use `getApplicationDocumentsDirectory()` from `path_provider` as the base path; never write outside it.
- Validate file names: strip path separators and `..` sequences from user-provided filenames.
- On mobile, leverage OS-level sandboxing but do not rely on it exclusively.

## Cryptography

- Use `pointycastle` or `cryptography` packages for crypto operations.
- Use `Random.secure()` for cryptographic random values. Never use `Random()` for security.
- Hash passwords with bcrypt (`dbcrypt` package) or argon2 on the server. Never hash passwords client-side only.
- Use AES-GCM for symmetric encryption. Never use ECB mode.
- For TLS certificate pinning, use `SecurityContext` with trusted certificates.
- Flag: MD5 or SHA1 used for security-sensitive hashing.

## Common CVE Patterns

- Insecure data storage: flag any sensitive data in `SharedPreferences`, plain files, or SQLite without encryption.
- Deep link hijacking: validate deep link parameters; never navigate to arbitrary URLs from deep links.
- Man-in-the-middle: enforce HTTPS and certificate pinning for all API communication.
- Intent/URL scheme redirection: validate callback URLs against allowlists in OAuth flows.
- Clipboard data exposure: clear clipboard after sensitive paste operations; avoid copying secrets.
- Debug mode leaks: ensure `kReleaseMode` checks guard sensitive logging and debug features.
- Platform channel injection: validate all data received from native platform channels.
