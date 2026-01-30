# C# Security Review Rules

## Input Validation

- Use `[Required]`, `[StringLength]`, `[Range]` data annotations on all model properties bound from requests.
- Use FluentValidation for complex validation rules; validate at the controller/endpoint level.
- Reject unexpected fields with `[JsonExtensionData]` awareness or strict deserialization settings.
- Limit request body size with `[RequestSizeLimit]` attribute or Kestrel configuration.
- Validate file uploads: check file extension, MIME type, and size. Never trust `ContentType` header.
- Use `Uri.TryCreate` with `UriKind.Absolute` and validate scheme/host before outbound requests.

## Authentication/Authorization

- Use ASP.NET Core Identity or external providers (Azure AD, Duende IdentityServer) for auth.
- Apply `[Authorize]` attribute at the controller level; use `[AllowAnonymous]` selectively.
- Use policy-based authorization with `IAuthorizationHandler` for complex permission logic.
- Never roll custom password hashing. Use `PasswordHasher<T>` from Identity framework.
- Configure cookie auth with `HttpOnly = true`, `Secure = true`, `SameSite = SameSiteMode.Strict`.
- Flag: `[AllowAnonymous]` on sensitive endpoints without documented justification.

## SQL/NoSQL Injection

- Use Entity Framework Core parameterized queries exclusively. Flag raw SQL with interpolation.
- EF Core: use `FromSqlInterpolated` or `FromSqlRaw` with explicit parameters, never string concat.
- Dapper: use parameterized queries with `@param` syntax. Flag string interpolation in SQL.
- Flag: `ExecuteSqlRaw($"DELETE FROM t WHERE id = {id}")` — use `ExecuteSqlInterpolated` instead.
- For Cosmos DB, use parameterized LINQ queries; never build query strings from user input.

## XSS Prevention

- Razor views auto-encode by default. Flag `@Html.Raw()` with user-controlled data.
- Use `HtmlEncoder.Default.Encode()` when manually constructing HTML strings.
- Set `Content-Security-Policy` header via middleware; disallow `unsafe-inline`.
- For Blazor, be cautious with `MarkupString` — never create from user input without sanitization.
- API responses: set `Content-Type: application/json` to prevent browser HTML rendering.
- Use anti-forgery tokens with `[ValidateAntiForgeryToken]` on all form POST actions.

## Secrets Management

- Use `IConfiguration` with `User Secrets` for local development; Azure Key Vault or AWS Secrets Manager in production.
- Never hardcode connection strings, API keys, or credentials in `appsettings.json` committed to source.
- Flag: string literals that look like connection strings, passwords, or API keys in source code.
- Use `IOptions<T>` pattern for typed configuration; never pass raw configuration strings around.
- `appsettings.Development.json` must be in `.gitignore` if it contains secrets.

## Dependency Vulnerabilities

- Run `dotnet list package --vulnerable` in CI to detect known vulnerabilities.
- Use `Directory.Packages.props` for centralized version management across solutions.
- Review NuGet package additions in PRs; verify publisher and download counts.
- Enable NuGet package signature validation.
- Avoid packages from untrusted sources; prefer `nuget.org` only.

## Deserialization

- Never use `BinaryFormatter`, `SoapFormatter`, or `NetDataContractSerializer` — they enable RCE.
- Use `System.Text.Json` with strict `JsonSerializerOptions`: no `TypeNameHandling` equivalent.
- Newtonsoft.Json: never use `TypeNameHandling.All` or `TypeNameHandling.Auto` with untrusted data.
- Flag: `JsonConvert.DeserializeObject` without explicit type parameter on untrusted input.
- Use `[JsonConstructor]` and immutable models to prevent unexpected property injection.
- Limit JSON depth with `JsonSerializerOptions.MaxDepth`.

## File System Access

- Use `Path.GetFullPath` and verify the result is under the allowed root directory.
- Flag: `Path.Combine(basePath, userInput)` without path traversal validation (absolute paths bypass base).
- Use `IWebHostEnvironment.WebRootPath` and `ContentRootPath` as safe base paths.
- Never serve files from user-controlled paths without strict allowlist validation.
- Use `FileStream` with `FileShare.Read` to prevent concurrent modification attacks.

## Cryptography

- Use `RandomNumberGenerator.GetBytes()` for secure random values. Never use `System.Random` for security.
- Hash passwords with `Rfc2898DeriveBytes` (PBKDF2) with 600,000+ iterations, or use Identity's `PasswordHasher<T>`.
- Use `Aes.Create()` with CBC+HMAC or GCM mode. Never use ECB or DES/3DES.
- Use `RSA.Create()` with at least 2048-bit keys for asymmetric operations.
- Use `HMACSHA256` for message authentication. Never use plain SHA for MACs.
- Use `X509Certificate2` for TLS certificate operations; validate chains properly.

## Common CVE Patterns

- XML External Entity (XXE): use `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit }`.
- SSRF: validate outbound URLs; block internal IP ranges and metadata endpoints (169.254.169.254).
- Open redirect: use `Url.IsLocalUrl()` or validate against a domain allowlist for redirects.
- Mass assignment: use DTOs or `[Bind]` attribute to restrict which properties are model-bound.
- HTTP header injection: validate that user data set in headers contains no CR/LF characters.
- Regex DoS: use `Regex` with `RegexOptions.Compiled` and `Regex.MatchTimeout` for user patterns.
- Server-side request forgery via `HttpClient`: validate and allowlist target hosts.
