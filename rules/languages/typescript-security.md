# TypeScript Security Review Rules

## Input Validation

- Use Zod, Yup, or io-ts to validate all external input at API boundaries. Never trust `req.body` raw.
- Define validation schemas co-located with route handlers; share schemas between frontend and backend.
- Validate query parameters and path parameters explicitly; Express/Fastify do not type-check them.
- Limit request body size via middleware (`express.json({ limit: '1mb' })`).
- For file uploads, validate MIME type, extension, and size server-side. Never trust client metadata.
- Use `URL` constructor to parse and validate URLs before making outbound requests.

## Authentication/Authorization

- Use established libraries: `passport`, `next-auth`, `lucia`, or framework-native auth.
- Store sessions server-side with secure cookie settings: `httpOnly`, `secure`, `sameSite: 'strict'`.
- JWT: always specify `algorithms` explicitly in verification. Never allow `"alg": "none"`.
- Use `jose` library for JWT operations, not deprecated `jsonwebtoken` versions.
- Implement RBAC or ABAC in middleware; never inline permission checks in business logic.
- Apply CSRF protection on state-changing endpoints; use `csurf` or SameSite cookies.

## SQL/NoSQL Injection

- Use parameterized queries with any SQL driver: Prisma, Drizzle, Knex, or raw `pg` with `$1` placeholders.
- Flag: string interpolation or template literals in SQL: `` `SELECT * FROM t WHERE id = ${id}` ``.
- Prisma: use `prisma.$queryRaw` with `Prisma.sql` tagged template, not raw string interpolation.
- MongoDB: never pass `JSON.parse(userInput)` as a query. Validate operator keys against allowlists.
- For Drizzle/Knex, use the query builder API; avoid `.raw()` with user data.

## XSS Prevention

- React escapes JSX by default. Flag any use of `dangerouslySetInnerHTML` with user-supplied content.
- Use `DOMPurify` to sanitize HTML before rendering if raw HTML display is genuinely required.
- Set `Content-Security-Policy` headers; disallow `unsafe-inline` and `unsafe-eval`.
- Never construct HTML strings with template literals for DOM insertion.
- For server-rendered apps (Next.js, Remix), sanitize data passed to client-side hydration.
- Set `X-Content-Type-Options: nosniff` on all responses.

## Secrets Management

- Never import or reference `.env` values directly in client-side code.
- Use `NEXT_PUBLIC_` or framework-specific prefixes only for truly public configuration.
- Flag: hardcoded API keys, tokens, passwords, or connection strings in source code.
- Use `dotenv` for local development only; load secrets from vault/secrets manager in production.
- Never log secrets. Strip sensitive fields before logging request/response objects.

## Dependency Vulnerabilities

- Run `npm audit` or `pnpm audit` in CI; fail builds on high/critical vulnerabilities.
- Use lockfile (`package-lock.json`, `pnpm-lock.yaml`) and review lockfile diffs in PRs.
- Enable Dependabot or Renovate for automated dependency updates.
- Avoid packages with low download counts, no recent updates, or suspicious name similarity.
- Use `socket.dev` or `snyk` for supply chain attack detection.
- Flag: `postinstall` scripts in new dependencies that execute arbitrary code.

## Deserialization

- Use Zod or similar to validate JSON payloads after parsing; do not trust `JSON.parse` output.
- Never use `eval()`, `new Function()`, or `vm.runInNewContext()` with user input.
- Flag: `JSON.parse` of user-controlled strings without subsequent schema validation.
- For MessagePack, Protocol Buffers, or other binary formats, use typed deserialization.
- Avoid `node:vm` for sandboxing untrusted code; it is not a security boundary.

## File System Access

- Sanitize file paths with `path.resolve` and verify the result starts with the allowed root.
- Flag: `path.join(base, userInput)` without checking for `..` traversal or absolute paths in input.
- Use `fs.createReadStream` with size limits for serving files to prevent memory exhaustion.
- Never serve files from a user-controllable directory without path validation.
- Set appropriate file permissions; avoid world-readable temp files.

## Cryptography

- Use `crypto.randomUUID()` or `crypto.getRandomValues()` for secure random generation.
- Never use `Math.random()` for tokens, nonces, or security-sensitive values.
- Hash passwords with `bcrypt` or `argon2`. Never use MD5/SHA for password storage.
- Use `crypto.timingSafeEqual` for comparing secrets to prevent timing attacks.
- For encryption, use `crypto.createCipheriv` with AES-256-GCM. Never use ECB mode.

## Common CVE Patterns

- Prototype pollution: flag `Object.assign(target, userInput)` or deep merge of untrusted objects.
- ReDoS: avoid complex regex on user input. Use `re2` or validate input length before matching.
- Path traversal: sanitize and validate all filesystem paths derived from user input.
- SSRF: validate URLs and block requests to internal IP ranges (`127.0.0.1`, `10.x`, `169.254.x`).
- Open redirects: validate redirect URLs against an allowlist of domains.
- Insecure `child_process.exec` with user input: use `execFile` with argument arrays instead.
- Header injection: validate that user-controlled values in HTTP headers contain no newlines.
