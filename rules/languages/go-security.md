# Go Security Review Rules

## Input Validation

- Use `strconv` functions for numeric parsing; never trust user input as valid integers or floats.
- Validate and sanitize all HTTP path parameters; `net/http` does not auto-decode path traversal sequences.
- Limit `io.Reader` consumption with `io.LimitReader` to prevent memory exhaustion from large payloads.
- For Gin/Echo/Fiber: always bind request bodies to typed structs with validation tags, never read raw strings.
- Reject unexpected `Content-Type` headers; do not auto-detect format from body content.
- Use `net/url.Parse` and verify scheme and host before making outbound requests from user input (SSRF).

## Authentication/Authorization

- Use middleware for auth checks; never inline token validation in handlers.
- JWT: validate issuer, audience, and expiration. Use `golang-jwt/jwt/v5`, not the archived `dgrijalva` fork.
- Never roll custom session management; use `gorilla/sessions` or equivalent with secure cookie settings.
- Set `HttpOnly`, `Secure`, and `SameSite=Strict` on all session cookies.
- RBAC checks must occur on every protected endpoint, not just at the router level.

## SQL/NoSQL Injection

- Always use parameterized queries: `db.Query("SELECT * FROM users WHERE id = $1", id)`.
- Flag any string concatenation or `fmt.Sprintf` building SQL strings.
- For `sqlx`, use named parameters: `sqlx.NamedExec("INSERT INTO t (col) VALUES (:val)", map)`.
- For MongoDB with the official driver, never pass `bson.M` built from raw user input without validation.

## XSS Prevention

- `html/template` auto-escapes by default. Never use `text/template` for HTML output.
- Flag any use of `template.HTML()`, `template.JS()`, or `template.CSS()` type conversions with user data.
- Set `Content-Type: application/json` on API responses to prevent browser HTML rendering.
- Set `X-Content-Type-Options: nosniff` via middleware.

## Secrets Management

- Never hardcode secrets, API keys, or connection strings. Use environment variables or a vault client.
- Flag any string literal that looks like a key, token, password, or DSN in source code.
- Use `os.Getenv` or a config library like `viper`; fail loudly at startup if required secrets are missing.
- Never log secrets. Redact sensitive fields in structured logging.
- `.env` files must be in `.gitignore`.

## Dependency Vulnerabilities

- Run `govulncheck ./...` in CI to detect known vulnerabilities in dependencies.
- Pin dependency versions in `go.sum`; review `go.sum` changes in PRs.
- Avoid replacing modules with local paths in committed `go.mod` (`replace` directives).
- Audit transitive dependencies periodically with `go mod graph`.

## Deserialization

- Use `encoding/json` with typed structs. Never unmarshal into `map[string]interface{}` for external input without validation.
- Set `json.Decoder.DisallowUnknownFields()` to reject unexpected fields.
- Limit JSON body size with `http.MaxBytesReader` before decoding.
- Never use `encoding/gob` or `encoding/xml` with untrusted input without strict schema validation.
- Flag use of `unsafe.Pointer` for type punning with external data.

## File System Access

- Sanitize file paths with `filepath.Clean` and verify they remain under the allowed root with `filepath.Rel`.
- Never use `os.Open` with user-supplied paths without validation.
- Use `filepath.Join` instead of string concatenation for path construction.
- Reject paths containing `..`, null bytes, or symlink chains outside the sandbox.
- Set restrictive permissions: `os.OpenFile(path, flags, 0600)` not `0777`.

## Cryptography

- Use `crypto/rand` for random bytes. Never use `math/rand` for security-sensitive values.
- Hash passwords with `golang.org/x/crypto/bcrypt` or `argon2`. Never use MD5/SHA for passwords.
- Use `crypto/tls` with `tls.Config{MinVersion: tls.VersionTLS12}` minimum.
- For symmetric encryption, use `crypto/aes` with GCM mode. Never use ECB.
- Do not implement custom cryptographic primitives.

## Common CVE Patterns

- Unsafe use of `net/http` default client (no timeout): always set `http.Client.Timeout`.
- SSRF via unvalidated URL construction: validate scheme is `https` and host is allowed.
- Zip slip: when extracting archives, validate that entry paths do not escape the target directory.
- Integer overflow in 32-bit builds: use `int64` for sizes and offsets from untrusted sources.
- Race conditions on shared maps: maps are not goroutine-safe; use `sync.Map` or a mutex.
