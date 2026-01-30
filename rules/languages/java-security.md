# Java Security Review Rules

## Input Validation

- Use Bean Validation (`jakarta.validation`) annotations on all request DTOs: `@NotNull`, `@Size`, `@Pattern`.
- In Spring, use `@Valid` or `@Validated` on controller parameters to trigger validation.
- Reject unexpected fields with `@JsonIgnoreProperties(ignoreUnknown = false)` on request DTOs.
- Limit request body size at the server level (Tomcat `maxPostSize`, Spring `max-http-request-header-size`).
- Validate file uploads: check extension, MIME type, and size. Never trust `Content-Type` header alone.
- Use `java.net.URI` for URL parsing and validate scheme/host before outbound requests.

## Authentication/Authorization

- Use Spring Security with `SecurityFilterChain` configuration. Never roll custom auth filters from scratch.
- Apply method-level security with `@PreAuthorize` or `@Secured` annotations.
- Hash passwords with `BCryptPasswordEncoder` or `Argon2PasswordEncoder`. Never use MD5/SHA for passwords.
- JWT: validate issuer, audience, expiration, and algorithm. Use `nimbus-jose-jwt` or `jjwt` with explicit algorithm.
- Flag: `Algorithm.NONE` acceptance or algorithm confusion (RSA public key used as HMAC secret).
- Configure CSRF protection; disable only for stateless API-only services with documented justification.

## SQL/NoSQL Injection

- Use JPA/Hibernate named parameters: `query.setParameter("id", id)`. Never concatenate strings.
- Flag: `entityManager.createNativeQuery("SELECT ... WHERE id = " + id)`.
- Spring Data JPA: use method-name queries or `@Query` with `:param` placeholders.
- JDBC: use `PreparedStatement` with `?` placeholders. Never use `Statement` with concatenated SQL.
- For MongoDB (Spring Data Mongo): use `Criteria` API, never string-based queries with user input.

## XSS Prevention

- Thymeleaf escapes by default with `th:text`. Flag `th:utext` with user-controlled content.
- For REST APIs, return `application/json` content type. Never render user data as HTML.
- Set `Content-Security-Policy`, `X-Content-Type-Options: nosniff` headers via Spring Security.
- Sanitize user HTML input with OWASP Java HTML Sanitizer before storage.
- Flag: JSP `<%= %>` expressions without JSTL `<c:out>` escaping (legacy codebases).

## Secrets Management

- Use Spring Cloud Config, HashiCorp Vault, or AWS Secrets Manager for production secrets.
- Never hardcode secrets in `application.properties` or `application.yml` committed to source.
- Use `@Value("${secret}")` with externalized configuration, not literals.
- Flag: string literals matching patterns of passwords, API keys, tokens, or JDBC URLs with credentials.
- Use `jasypt-spring-boot` for encrypted properties if vault integration is not available.

## Dependency Vulnerabilities

- Run OWASP Dependency-Check or Snyk in CI to detect known CVEs in dependencies.
- Use Maven `dependencyManagement` or Gradle `platform` for centralized version control.
- Review `pom.xml` or `build.gradle` diffs in PRs for new or changed dependencies.
- Avoid dependencies with known unpatched vulnerabilities. Pin transitive versions when necessary.
- Use `mvn dependency:tree` or `gradle dependencies` to audit the full dependency graph.

## Deserialization

- Never use Java native serialization (`ObjectInputStream`) with untrusted data. It enables RCE.
- Flag: any class implementing `Serializable` without a clear internal-only justification.
- Use Jackson with default typing disabled. Flag `@JsonTypeInfo` with `Id.CLASS` or `Id.MINIMAL_CLASS`.
- Configure Jackson `ObjectMapper` with `activateDefaultTyping` only for trusted internal communication.
- Use `SnakeYAML` `SafeConstructor` for YAML parsing; never allow arbitrary object construction.
- Flag: `XMLDecoder`, `XStream` without security framework, or `Kryo` with untrusted data.

## File System Access

- Use `Path.normalize()` and `Path.startsWith(allowedRoot)` for path validation.
- Flag: `new File(basePath + userInput)` without path traversal checks.
- Use Spring's `Resource` abstraction for file access in web applications.
- Validate uploaded filenames: strip path separators, limit characters, generate server-side names.
- Set restrictive file permissions with `Files.setPosixFilePermissions` or ACLs.

## Cryptography

- Use `SecureRandom` for all cryptographic random generation. Never use `java.util.Random`.
- Use `Cipher.getInstance("AES/GCM/NoPadding")` for symmetric encryption. Never use ECB.
- Use `MessageDigest` with SHA-256+ for hashing. Never use MD5 or SHA-1 for security purposes.
- Use BouncyCastle for algorithms not in the standard JCE provider.
- Generate keys with `KeyGenerator` or `KeyPairGenerator`, not from hardcoded bytes.
- Use TLS 1.2+ exclusively. Configure `SSLContext` with modern cipher suites.

## Common CVE Patterns

- Log4Shell-style JNDI injection: never pass user input to logging frameworks without sanitization.
- XXE: configure `DocumentBuilderFactory` with `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`.
- SSRF: validate outbound URLs; block internal IPs, link-local, and cloud metadata endpoints.
- Server-Side Template Injection: never pass user input as template source to Freemarker, Velocity, or Thymeleaf.
- Expression Language injection: never evaluate user input as SpEL, OGNL, or EL expressions.
- Zip slip: validate `ZipEntry.getName()` does not traverse outside the target directory.
- ClassLoader manipulation: never use `Class.forName()` with user-controlled class names.
