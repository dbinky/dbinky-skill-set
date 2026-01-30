# Spring Boot Review Rules

## Architecture Patterns

Spring Boot follows layered architecture with dependency injection as the core mechanism. Convention over configuration drives most defaults.

**Expect to see:**
- `@RestController` / `@Controller` for web endpoints
- `@Service` for business logic, `@Repository` for data access
- Constructor injection (not field injection)
- Profiles (`@Profile`) for environment-specific behavior

**Flag in review:**
- Field injection (`@Autowired` on fields) — use constructor injection for testability
- Business logic in controllers — controllers should only delegate
- `@Component` used where `@Service` or `@Repository` is more descriptive
- Circular dependencies — redesign, do not use `@Lazy` as a bandaid
- `@SpringBootApplication` on a class that also contains beans or business logic

## Common Anti-Patterns

- **Service Locator**: Using `ApplicationContext.getBean()` instead of injection. Flag every occurrence.
- **Anemic Domain Model**: Entities with only getters/setters and all logic in services. Note but don't always flag — it's a valid choice in CRUD apps.
- **God Service**: Single service class with 20+ methods. Split by domain.
- **Repository Bloat**: Custom query methods that could use Spring Data derived queries or `@Query`.
- **Exception Swallowing**: Empty catch blocks or `catch (Exception e) { log.error(e); }` without rethrowing or returning an error.
- **Transactional Misuse**: `@Transactional` on private methods (doesn't work — proxy-based AOP). Flag immediately.

## State Management

- Spring beans are singletons by default. Flag mutable instance fields on singleton beans.
- Use `@RequestScope` or `@SessionScope` only when necessary — flag session-scoped beans in stateless APIs.
- For caching: prefer `@Cacheable` / `@CacheEvict` with Spring Cache abstraction.
- Flag `static` mutable state on any Spring-managed bean.
- Flag `ThreadLocal` usage without cleanup — causes leaks in thread pools.

## Testing Patterns

- Use `@SpringBootTest` sparingly — it loads the full context. Prefer sliced tests.
- `@WebMvcTest` for controller tests, `@DataJpaTest` for repository tests.
- Flag tests using `@SpringBootTest` that only test a single service — use `@ExtendWith(MockitoExtension.class)`.
- Use `@MockBean` only in integration tests. Unit tests should use plain Mockito.
- Flag `Thread.sleep()` in tests. Use `Awaitility` for async assertions.
- Testcontainers for database integration tests — flag tests hitting real external services.

## Performance

- **N+1 Queries**: Flag `@OneToMany` / `@ManyToOne` without fetch strategy consideration. Default LAZY is correct but callers must use `JOIN FETCH` or `@EntityGraph`.
- **Missing Indexes**: Flag new `@Entity` fields used in `WHERE` clauses without `@Index`.
- **Open Session in View**: Flag `spring.jpa.open-in-view=true` (the default). Disable it.
- **Blocking in WebFlux**: Flag blocking calls (`Thread.sleep`, JDBC, `RestTemplate`) in reactive endpoints.
- **Missing Pagination**: Flag repository methods returning `List` for potentially large datasets. Use `Page` or `Slice`.
- **Connection Pool**: Flag missing HikariCP configuration for production.

## Lifecycle

- `@PostConstruct` — initialization after injection. Flag heavy I/O here; use `ApplicationRunner` instead.
- `@PreDestroy` — cleanup. Flag critical logic here; not guaranteed on hard shutdown.
- `ApplicationReadyEvent` — flag startup logic that should wait until the app is fully ready but uses `@PostConstruct` instead.
- Bean lifecycle: flag beans that depend on initialization order without explicit `@DependsOn`.
- Flag `@Scheduled` methods without `@EnableScheduling` on a configuration class.

## Configuration

- Use `@ConfigurationProperties` with `@Validated` for typed configuration. Flag raw `@Value` for complex config.
- Flag `application.properties` with environment-specific values. Use profiles or environment variables.
- Flag missing `@Validated` on `@ConfigurationProperties` classes.
- Flag hardcoded URLs, ports, or credentials anywhere in Java code.
- Prefer `application.yml` over `application.properties` for nested configuration.
- Flag `@ConditionalOnProperty` without a `havingValue` — behavior is ambiguous.

## Error Handling

- Use `@RestControllerAdvice` with `@ExceptionHandler` for global error handling. Flag per-controller exception handling.
- Flag controllers returning raw exception messages to clients.
- Define a consistent error response DTO. Flag inconsistent error shapes across endpoints.
- Flag `catch (Exception e)` — catch specific exceptions.
- Use `ResponseStatusException` or custom exceptions with `@ResponseStatus` for HTTP error mapping.
- Flag `@Transactional` methods that catch and swallow exceptions — the transaction won't roll back.

## API Design

- Flag REST endpoints without proper HTTP method semantics (POST for reads, GET with side effects).
- Flag missing `@Valid` on `@RequestBody` parameters.
- Flag endpoints returning JPA entities directly — use DTOs to control the API contract.
- Flag missing API versioning strategy on public APIs.
- Flag `@PathVariable` and `@RequestParam` without validation constraints.
