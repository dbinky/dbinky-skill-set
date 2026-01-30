# Rust Security Review Rules

## Input Validation

- Validate all external input at the deserialization boundary using `serde` with strict types, not raw strings.
- Use newtypes with validation in `TryFrom` implementations for domain values (emails, IDs, URLs).
- For Actix-web/Axum: use extractors with typed structs. Never read raw request bodies as strings for parsing.
- Limit request body size at the framework level (`actix_web::web::JsonConfig::limit()`).
- Use the `validator` crate with derive macros for declarative field validation.
- Parse, don't validate: convert strings to typed values at the boundary and pass typed values internally.

## Authentication/Authorization

- Use established crates: `jsonwebtoken`, `argon2`, `oauth2`. Never implement auth primitives from scratch.
- JWT: specify allowed algorithms explicitly. Flag `Algorithm::None` or dynamic algorithm from header.
- Use tower middleware (Axum) or actix middleware for auth checks; never inline in handlers.
- Store sessions server-side with signed, HttpOnly, Secure cookies.
- Implement authorization as a middleware layer that checks permissions before handler execution.

## SQL/NoSQL Injection

- Use `sqlx` with compile-time checked queries (`sqlx::query!`) or parameterized runtime queries.
- Diesel: use the query builder DSL exclusively. Flag `.sql_query()` with string interpolation.
- `sqlx`: use `query_as!` macros for type-safe queries. For dynamic queries, use `.bind()` exclusively.
- Flag: any `format!` or string concatenation constructing SQL strings.
- For MongoDB with `mongodb` crate: construct `bson::doc!` from validated typed fields, not raw user strings.

## XSS Prevention

- For server-rendered HTML, use template engines that auto-escape: `askama` (compile-time), `tera` (runtime).
- Flag: `tera` templates using `| safe` filter with user-controlled data.
- For API servers, set `Content-Type: application/json` on all JSON responses.
- Set `X-Content-Type-Options: nosniff` and `Content-Security-Policy` headers via middleware.
- If generating HTML strings programmatically, use `html_escape` crate, never manual replacement.

## Secrets Management

- Never hardcode secrets. Use environment variables with `std::env::var` or config crates (`config`, `figment`).
- Flag: string literals that match patterns of API keys, tokens, passwords, or connection strings.
- Use `secrecy::Secret<String>` to wrap sensitive values; prevents accidental logging via `Debug`/`Display`.
- Zeroize secrets in memory when no longer needed using the `zeroize` crate.
- `.env` files must be in `.gitignore`.

## Dependency Vulnerabilities

- Run `cargo audit` in CI to detect known vulnerabilities.
- Use `cargo deny` to enforce license policies and ban specific crates or versions.
- Review `Cargo.lock` diffs in PRs for unexpected dependency additions.
- Minimize dependency count; audit transitive dependencies with `cargo tree`.
- Flag: crates with `build.rs` that download or execute external code.
- Use `cargo vet` for supply chain verification in high-security projects.

## Deserialization

- Use `serde` with typed structs. Never deserialize into `serde_json::Value` for untrusted input without validation.
- Set `#[serde(deny_unknown_fields)]` on structs receiving external input.
- Limit nesting depth and string lengths in custom `Deserialize` implementations.
- Flag: `serde_yaml` or `serde_xml_rs` with untrusted input without size/depth limits.
- Never use `bincode` or `postcard` for untrusted external data without validation wrappers.

## File System Access

- Canonicalize paths with `std::fs::canonicalize` and verify they remain under the allowed root.
- Flag: `std::path::Path::join` with user input without verifying the result stays within bounds.
- Use `tempfile` crate for temporary files and directories; it handles cleanup and secure creation.
- Reject paths containing `..` components before any file system access.
- Set restrictive permissions using `std::fs::set_permissions` with mode `0o600`.

## Cryptography

- Use `rand::rngs::OsRng` or `getrandom` for cryptographic random values. Never use `rand::thread_rng` for secrets.
- Hash passwords with `argon2` or `bcrypt` crates. Never use SHA/BLAKE for password storage.
- Use `aes-gcm` crate for symmetric encryption. Never use ECB mode or unauthenticated encryption.
- Use `ed25519-dalek` or `ring` for digital signatures.
- Flag: any `unsafe` code in cryptographic implementations.
- Use `constant_time_eq` or `subtle` crate for secret comparison to prevent timing attacks.

## Common CVE Patterns

- `unsafe` code: every `unsafe` block must have a `// SAFETY:` comment documenting invariants maintained.
- Integer overflow: use `.checked_add()`, `.saturating_mul()` for arithmetic on untrusted input.
- Buffer overflows in `unsafe`: verify all pointer arithmetic and slice indexing stays within bounds.
- Denial of service: limit input sizes, collection capacities, and recursion depth.
- SSRF: validate URLs and block internal IP ranges before making outbound HTTP requests.
- Zip slip: validate archive entry paths do not escape the target directory during extraction.
- Stack overflow from deep recursion: use iterative algorithms or `stacker` crate for deep recursion.
