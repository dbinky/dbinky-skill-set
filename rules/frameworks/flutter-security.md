# Flutter Security Review Rules

## Platform Channel Security

- Flag `MethodChannel` calls that pass sensitive data (tokens, keys, PII) without encryption — platform channels are not encrypted in transit between Dart and native.
- Verify platform channel names use reverse-domain notation and are not guessable — other apps on the device could potentially register the same channel.
- Flag `BasicMessageChannel` used for sensitive operations — prefer `MethodChannel` with explicit method names for auditability.
- Flag native side channel handlers that don't validate method arguments (type, range, nullability).
- Verify platform channel error handling doesn't leak native stack traces to the Dart layer.
- Flag `EventChannel` streams carrying sensitive data without considering that listeners could be added by injected code in debug builds.

## Secure Storage for Tokens

- Flag tokens, API keys, or secrets stored with `SharedPreferences` — it is plaintext on both platforms.
- Use `flutter_secure_storage` (Keychain on iOS, EncryptedSharedPreferences/Keystore on Android). Verify it is the storage mechanism for all credentials.
- Flag tokens stored in Dart-level global variables that persist in memory — they survive hot reload in debug.
- Verify secure storage access is wrapped in try/catch — Keychain/Keystore can fail (locked device, biometric required).
- Flag `sqflite` databases storing tokens without encryption. Use `sqflite_sqlcipher` or encrypt at the application layer.
- Verify tokens are cleared from secure storage on logout.

## Certificate Pinning

- Flag production apps making HTTPS calls without certificate pinning for sensitive endpoints.
- Verify pinning is implemented via `SecurityContext` in Dart or native-level configuration (network security config on Android, ATS on iOS).
- Flag pinning only the leaf certificate — pin the intermediate or root CA for rotation resilience.
- Verify pinning failure surfaces a clear error, not a silent fallback to unpinned connection.
- Flag apps that disable certificate validation for debugging (`badCertificateCallback: (cert, host, port) => true`) and verify this is impossible in release builds.
- Verify `http_certificate_pinning` or `ssl_pinning_plugin` packages, if used, are configured for all API hosts.

## Obfuscation and Reverse Engineering

- Verify `--obfuscate` and `--split-debug-info` flags are used in release builds. Flag their absence.
- Flag hardcoded API keys, secrets, or encryption keys in Dart source — they are readable in release binaries even with obfuscation.
- Verify sensitive strings are fetched at runtime from a secure backend, not bundled in the app.
- Flag `debugPrint`, `print`, or `log` statements that output sensitive data — they appear in device logs in release.
- Verify ProGuard/R8 rules are configured for Android release builds to strip unused code.
- Flag debug-mode checks (`kDebugMode`) that gate security features — ensure security features are always on.

## Deep Link Validation

- Flag deep link handlers (`uni_links`, `go_router`, `app_links`) that navigate to sensitive screens without re-validating authentication.
- Verify deep link parameters are validated and sanitized — they are attacker-controlled input.
- Flag deep links that trigger actions (delete, purchase, transfer) without confirmation or auth re-check.
- Verify Android `intent-filter` and iOS `Associated Domains` are configured with specific hosts, not wildcards.
- Flag `getInitialLink()` and `linkStream` handlers that don't handle malformed URIs (parse errors, missing parameters).
- Verify universal links / app links use HTTPS with proper `assetlinks.json` (Android) and `apple-app-site-association` (iOS).

## Input Validation

- Flag `TextField` inputs used for sensitive data (passwords, PINs) without `obscureText: true`.
- Flag form submissions without client-side validation (and verify server-side validation exists).
- Flag `WebView` usage with `javascriptMode: JavascriptMode.unrestricted` loading untrusted URLs.
- Verify `WebView` `navigationDelegate` blocks navigation to unexpected domains.
- Flag `Uri.parse` on user input without catching `FormatException`.

## API Security

- Verify all HTTP clients enforce HTTPS. Flag `http://` URLs in production code.
- Flag API keys in Dart source files. Use build-time injection (`--dart-define`) or fetch from secure backend.
- Verify request/response logging (e.g., `dio` interceptors) does not log auth headers or tokens in release.
- Flag missing request timeouts on HTTP clients — default can be infinite.
- Verify retry logic has exponential backoff and a maximum retry count. Flag unbounded retries.

## Data Exposure

- Flag screenshots enabled on screens with sensitive data. Use `FlutterWindowManager` or native flags to prevent screen capture.
- Verify the app clears sensitive data from memory when backgrounded (iOS app snapshot, Android recent apps).
- Flag Clipboard usage for sensitive data (`Clipboard.setData` with tokens or passwords).
- Verify biometric auth (`local_auth`) uses the `biometricOnly` option where appropriate — fallback to device PIN may be insufficient.
- Flag `debugShowCheckedModeBanner` left as default in release — it should be `false` but is cosmetic, not security.

## Dependency Security

- Flag dependencies not from `pub.dev` (git dependencies, path dependencies in production).
- Verify `pubspec.lock` is committed. Flag if missing.
- Flag packages with known vulnerabilities — check `pub.dev` advisories.
- Flag native dependencies (CocoaPods, Gradle) pinned to outdated versions with known CVEs.
- Verify `dart pub outdated` is run in CI to catch stale dependencies.

## Configuration Security

- Flag Firebase configuration files (`google-services.json`, `GoogleService-Info.plist`) containing restricted API keys committed without `.gitignore` consideration. These files must be in the build but should not contain admin keys.
- Verify `AndroidManifest.xml` does not declare `android:debuggable="true"` for release.
- Verify `android:allowBackup="false"` is set to prevent ADB backup extraction of app data.
- Flag `NSAppTransportSecurity` exceptions that allow arbitrary HTTP loads on iOS.
- Verify `export-compliance` keys are set in `Info.plist` if using encryption.
