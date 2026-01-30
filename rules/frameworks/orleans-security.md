# Orleans Security Review Rules

## Authentication and Authorization

- **Grain Access Control**: Flag grains exposing sensitive operations without access checks. Orleans has no built-in per-grain auth — you must implement it.
- Use `IIncomingGrainCallFilter` to enforce authorization at the silo level before grain methods execute.
- Flag grain interfaces marked as public API that lack caller identity validation.
- For multi-tenant systems, verify grain key includes tenant context and that cross-tenant access is impossible.
- Flag `[StatelessWorker]` grains performing privileged operations — any silo can activate them.

## Grain State Serialization Security

- Flag grain state containing secrets (API keys, passwords, tokens) — state is persisted to storage in serialized form.
- Verify state encryption at rest is configured for storage providers holding sensitive state.
- Flag custom serializers that skip validation or deserialize untrusted input without bounds checking.
- Orleans serialization can instantiate arbitrary types — flag any scenario where untrusted data feeds into grain state deserialization.
- Verify `[Immutable]` is used on types passed between grains to prevent accidental mutation and reduce serialization.

## Cluster Membership Authentication

- Flag clusters configured without TLS for silo-to-silo communication.
- Verify mutual TLS (mTLS) is enabled for production cluster membership.
- Flag `UseDevelopmentClustering()` in production code paths.
- Flag hardcoded membership connection strings — use secret management (Azure Key Vault, AWS Secrets Manager).
- Verify clustering provider credentials are rotated and not embedded in source.

## Timer and Reminder Security

- Flag timers or reminders that trigger privileged operations without re-validating authorization at execution time.
- Reminders persist across activations — flag reminder-triggered actions that assume the original authorization context is still valid.
- Flag timers with very short intervals that could amplify an attack (e.g., polling an external service in a tight loop).
- Verify reminder registration is not exposed to untrusted callers who could create arbitrary reminders.

## Stream Security

- Flag stream namespaces that lack access control — any grain with the stream ID can subscribe.
- Verify implicit stream subscriptions (`[ImplicitStreamSubscription]`) don't expose data to unintended grain types.
- Flag streams carrying sensitive data without encryption or access validation in the subscriber.
- Verify stream providers are configured with authentication for external transport (EventHub, Kafka, SQS).
- Flag fire-and-forget stream publishing of sensitive events without audit logging.

## Input Validation

- Grain method parameters arrive deserialized — flag methods that trust input without validation.
- Flag grain methods accepting `string` or `byte[]` for structured data. Prefer typed DTOs with validation.
- Verify grain key parsing is safe — flag `string` grain keys parsed without validation.
- Flag grains that pass user input directly to external systems (SQL, HTTP, file paths) without sanitization.

## API Gateway Security

- Flag Orleans clients (web API controllers calling grains) that pass unsanitized HTTP input directly to grain methods.
- Verify the web API layer performs authentication before invoking grains — grains should not be the first auth boundary.
- Flag grain factory usage in middleware or filters where the caller identity is not yet established.

## Data Exposure

- Flag grain methods returning full state objects to external callers. Return projections or DTOs.
- Flag grain observer callbacks that send sensitive data to client-registered observers.
- Verify logging inside grains does not log state containing PII or secrets.
- Flag `ToString()` overrides on grain state that might expose sensitive fields in logs.

## Configuration Security

- Flag connection strings, storage credentials, or cluster secrets in `appsettings.json` committed to source control.
- Verify `appsettings.Development.json` is in `.gitignore`.
- Flag Orleans Dashboard enabled without authentication in production.
- Verify silo ports (11111, 30000) are not exposed to public networks.
- Flag `EnableDirectClient` in production — it bypasses network-level security boundaries.

## Deployment and Infrastructure

- Verify silo endpoints use TLS termination.
- Flag Docker images running the silo as root.
- Verify health check endpoints (`/health`) do not expose cluster topology details.
- Flag missing network segmentation — silo-to-silo traffic should be on a private network.
- Verify grain storage accounts use least-privilege access policies.

## Common Vulnerability Patterns

- **Insecure Deserialization**: Orleans deserializes grain calls. Flag custom `IExternalSerializer` implementations that handle untrusted types.
- **Denial of Service**: Flag grains that allocate memory proportional to untrusted input without limits.
- **Information Disclosure**: Flag exception messages returned to external callers containing stack traces or internal state.
- **Privilege Escalation**: Flag grain methods that change roles or permissions without re-authenticating the caller.
