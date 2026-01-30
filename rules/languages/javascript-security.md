# JavaScript Security Review Rules

## Input Validation

- Validate all user input server-side, even if client-side validation exists.
- Use schema validation libraries (Joi, Yup, Zod) at API boundaries for every endpoint.
- Reject requests with unexpected fields; use strict schemas that disallow additional properties.
- Limit request body size: `express.json({ limit: '100kb' })`.
- Validate Content-Type headers; reject unexpected media types.
- Sanitize strings destined for regex with a literal-escaping function before `new RegExp()`.

## Authentication/Authorization

- Use `express-session` with a server-side store (Redis, Postgres), not the default memory store.
- Set cookie options: `httpOnly: true`, `secure: true`, `sameSite: 'strict'`, `maxAge` with reasonable TTL.
- Use `helmet` middleware to set security headers in Express apps.
- Never expose session IDs or tokens in URLs (query strings or path parameters).
- Implement rate limiting on login endpoints with `express-rate-limit`.
- Use CSRF tokens for state-changing requests; `sameSite` cookies alone are insufficient in older browsers.

## SQL/NoSQL Injection

- Use parameterized queries with placeholders: `db.query('SELECT * FROM t WHERE id = $1', [id])`.
- Flag: any string concatenation or template literal building SQL or MongoDB queries.
- Sequelize: use model methods and query builder; avoid `sequelize.query()` with interpolated strings.
- MongoDB/Mongoose: flag `$where` queries and `JSON.parse(userInput)` as query filters.
- For raw queries, use the driver's parameterization, never manual escaping.

## XSS Prevention

- React/JSX escapes by default. Flag `dangerouslySetInnerHTML` with user-controlled values.
- Use `DOMPurify` before rendering any user-supplied HTML if unavoidable.
- Never use `document.write()`, `innerHTML`, `outerHTML`, or `insertAdjacentHTML` with user data.
- Set `Content-Security-Policy` to disallow `unsafe-inline` scripts and `unsafe-eval`.
- Escape user data in URL parameters with `encodeURIComponent`.
- For server-rendered HTML (EJS, Pug, Handlebars), ensure auto-escaping is enabled. Flag raw output helpers.

## Secrets Management

- Never bundle secrets into client-side JavaScript. Use server-side env vars only.
- Flag: hardcoded API keys, tokens, or credentials in any `.js` file.
- Use `dotenv` for local development; source secrets from vault or secrets manager in production.
- Never log secrets, tokens, or full request headers containing auth data.
- `.env` and any credential files must be in `.gitignore`.

## Dependency Vulnerabilities

- Run `npm audit --production` in CI; break builds on critical/high severity.
- Review `package-lock.json` diffs in every PR for unexpected additions or version changes.
- Enable Dependabot, Renovate, or Socket for automated vulnerability and supply chain monitoring.
- Avoid packages with `postinstall` scripts that download or execute external code.
- Flag: packages with very low weekly downloads or names similar to popular packages (typosquatting).
- Prefer well-maintained packages with active security disclosure processes.

## Deserialization

- Never use `eval()`, `new Function()`, or `vm.runInContext()` with user-controlled input.
- Validate `JSON.parse()` output against a schema before use.
- Flag: `unserialize` from third-party libraries that execute code during deserialization.
- For binary formats (MessagePack, CBOR), use typed deserialization with schema validation.
- Avoid `node:vm` for sandboxing; it does not provide security isolation.

## File System Access

- Resolve and validate file paths before any `fs` operation: `path.resolve()` then check prefix.
- Flag: `path.join(base, userInput)` without verifying the result stays within the intended directory.
- Never serve static files from a directory based on user-controlled path without sanitization.
- Use `fs.createReadStream` with size checks for serving user-requested files.
- Create temporary files with `os.tmpdir()` and unique names; clean up on process exit.

## Cryptography

- Use `crypto.randomBytes()` or `crypto.randomUUID()` for secure random values.
- Never use `Math.random()` for tokens, secrets, nonces, or any security-related value.
- Use `crypto.timingSafeEqual()` for comparing secrets to prevent timing side channels.
- Hash passwords with `bcrypt` (cost factor >= 12) or `argon2`. Never use MD5/SHA for passwords.
- For symmetric encryption, use AES-256-GCM via `crypto.createCipheriv`.
- Never implement custom crypto. Use audited libraries.

## Common CVE Patterns

- Prototype pollution: flag `_.merge`, `_.defaultsDeep`, `Object.assign` with untrusted objects. Use `Object.create(null)` for dictionary objects.
- ReDoS: flag complex regex applied to user input. Use `re2` for untrusted patterns.
- SSRF: validate outbound request URLs; block internal IP ranges and link-local addresses.
- Open redirect: validate redirect targets against a domain allowlist, not just path checks.
- Command injection: use `child_process.execFile` with arrays, never `exec` with interpolated strings.
- Path traversal: validate all file paths against allowed directories with resolved absolute paths.
- Clickjacking: set `X-Frame-Options: DENY` or use CSP `frame-ancestors` directive.
