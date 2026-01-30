# Spring Boot Security Review Rules

## Spring Security Configuration

- Flag `SecurityFilterChain` beans using `.permitAll()` on sensitive endpoints.
- Flag `http.csrf().disable()` without justification — only acceptable for stateless JWT APIs.
- Flag `http.authorizeHttpRequests()` with overly broad matchers (e.g., `/**` permitted).
- Verify the security filter chain order when multiple chains exist (`@Order` matters).
- Flag `WebSecurityConfigurerAdapter` — deprecated since Spring Security 5.7. Use component-based config.
- Flag `.httpBasic()` enabled in production without TLS.
- Verify `formLogin()` endpoints are not exposed on pure API services.

## CORS Policies

- Flag `@CrossOrigin("*")` — never allow all origins in production.
- Verify CORS is configured centrally via `CorsConfigurationSource`, not per-controller.
- Flag missing `allowedMethods` restrictions — default may be too permissive.
- Flag `allowCredentials(true)` combined with `allowedOrigins("*")` — this is invalid and Spring will throw, but flag the intent.
- Verify preflight cache (`maxAge`) is set to avoid excessive OPTIONS requests.

## Actuator Endpoint Exposure

- Flag `management.endpoints.web.exposure.include=*` — exposes all actuator endpoints.
- Verify `/actuator/env`, `/actuator/configprops`, and `/actuator/heapdump` are NOT exposed to the public.
- Flag actuator endpoints without authentication. Use Spring Security to protect them.
- Prefer exposing only `/actuator/health` and `/actuator/info` publicly.
- Flag `/actuator/shutdown` enabled without authentication — allows remote shutdown.
- Verify actuator runs on a separate port (`management.server.port`) in production.

## Bean Injection Security

- Flag `@Value("${secret}")` pulling secrets from properties files committed to source control.
- Verify secrets come from environment variables, Vault, or AWS Secrets Manager.
- Flag `@ConfigurationProperties` classes with secrets that have `toString()` methods — risk of logging secrets.
- Flag beans that store decrypted secrets in mutable fields accessible via getters.

## @PreAuthorize / @Secured Patterns

- Flag `@PreAuthorize` using string expressions without tests verifying them — SpEL errors are runtime.
- Verify `@EnableMethodSecurity` (or `@EnableGlobalMethodSecurity`) is present when method security annotations are used.
- Flag service methods performing authorization checks that should use `@PreAuthorize` instead.
- Flag `@Secured("ROLE_ADMIN")` on controller methods only — also secure the service layer.
- Verify role hierarchy is configured if roles have inheritance relationships.
- Flag `hasPermission()` expressions without a registered `PermissionEvaluator`.

## Input Validation

- Flag `@RequestBody` parameters without `@Valid` or `@Validated`.
- Flag custom validation that reimplements what `javax.validation` annotations provide.
- Verify `@PathVariable` values used in database queries are parameterized, not concatenated.
- Flag `@RequestParam` values passed to `Runtime.exec()` or `ProcessBuilder`.
- Flag missing `@Size`, `@NotNull`, `@Pattern` on DTO fields exposed to user input.

## SQL Injection

- Flag native queries built with string concatenation: `"SELECT * FROM x WHERE id = " + id`.
- Verify `@Query` uses named parameters (`:param`) not positional concatenation.
- Flag `JdbcTemplate` queries using string formatting for parameters.
- `EntityManager.createNativeQuery()` with concatenated strings is a critical flag.

## Session Management

- For stateless APIs: verify `sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)`.
- Flag session fixation vulnerability — verify `sessionFixation().migrateSession()` or `.newSession()`.
- Flag sessions stored in cookies without `httpOnly` and `secure` flags.
- For distributed deployments: verify session storage (Redis, JDBC) is configured, not in-memory.

## JWT and Token Security

- Flag JWT secret keys shorter than 256 bits for HMAC algorithms.
- Flag JWT secrets hardcoded in source — use environment variables or secret managers.
- Verify JWT expiration (`exp`) is validated and tokens have reasonable TTLs.
- Flag missing JWT signature verification — never trust an unverified token.
- Flag `none` algorithm acceptance in JWT parsing libraries.
- Verify refresh token rotation is implemented.

## API Security

- Flag endpoints returning JPA entities with sensitive fields (password hashes, internal IDs, tokens).
- Verify rate limiting is configured for authentication endpoints.
- Flag missing HTTPS enforcement (`requiresChannel().anyRequest().requiresSecure()`).
- Verify API keys are not logged in request logging filters.
- Flag file upload endpoints without size limits, type validation, and virus scanning.

## Data Exposure

- Flag `@JsonIgnore` as the sole mechanism to hide sensitive fields — defense in depth requires DTOs.
- Flag `@ToString` (Lombok) on entities containing passwords or tokens.
- Verify error responses do not include stack traces in production (`server.error.include-stacktrace=never`).
- Flag H2 console enabled in production (`spring.h2.console.enabled=true`).

## Configuration Security

- Flag `spring.datasource.password` in committed property files.
- Verify `application-prod.yml` is not in source control if it contains secrets.
- Flag `debug=true` or `logging.level.root=DEBUG` in production profiles.
- Verify `server.error.include-message=never` in production.
- Flag Spring DevTools dependency in production builds — it enables remote restart.
