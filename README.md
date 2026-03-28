# SPHUTA Arka Platform

> **Status:** Architecture frozen — canonical reference document
> **Last updated:** March 2026
>
> All decisions in this document were deliberated and locked.
> Deviations require explicit agreement and an update to this document before implementation.
> Code and documentation must move together.

---

## Contents

1. [Platform overview](#1-platform-overview)
2. [Channel family](#2-channel-family)
3. [Architecture — shared engine and channel edges](#3-architecture--shared-engine-and-channel-edges)
4. [Tier model — OSS, PaaS, SaaS](#4-tier-model--oss-paas-saas)
5. [Module inventory](#5-module-inventory)
6. [Starter composition](#6-starter-composition)
7. [Activation philosophy](#7-activation-philosophy)
8. [Dependency direction](#8-dependency-direction)
9. [Send flow — single send](#9-send-flow--single-send)
10. [Send flow — batch send](#10-send-flow--batch-send)
11. [Provider resolution](#11-provider-resolution)
12. [Application codes](#12-application-codes)
13. [Exception model](#13-exception-model)
14. [Validation layers](#14-validation-layers)
15. [Transport and timeout](#15-transport-and-timeout)
16. [Retry and resilience](#16-retry-and-resilience)
17. [Async and threading](#17-async-and-threading)
18. [Security model](#18-security-model)
19. [Role-aware UI](#19-role-aware-ui)
20. [Multi-tenancy](#20-multi-tenancy)
21. [Observability](#21-observability)
22. [Extension points](#22-extension-points)
23. [Technology stack](#23-technology-stack)
24. [Phased development plan](#24-phased-development-plan)
25. [Naming conventions](#25-naming-conventions)
26. [Repository structure](#26-repository-structure)

---

## 1. Platform overview

SPHUTA Arka is a flow-based communication platform for the JVM. It provides a modular, policy-driven framework for sending transactional messages across multiple channels. The architecture follows a shared-engine-plus-channel-edge model: a set of channel-agnostic shared capabilities form the engine, and each channel contributes its own edge modules for transport, message format, and channel-specific concerns.

The platform ships as Spring Boot starters. A client application adds a single Maven dependency and gets a production-ready communication runtime with startup validation, pluggable transport, recipient handling, and configurable flow definitions.

Three editions serve different operational needs: OSS for self-managed open-source use, Enterprise (PaaS) for self-managed commercial deployments with operational hardening, and Cloud (SaaS) for fully managed delivery at `cloud.sphuta.net`.

---

## 2. Channel family

Four standalone products. Each has its own repository, its own starters, and its own editions page. No umbrella grouping.

| Product | Channel | Maven root | GitHub |
|---|---|---|---|
| SPHUTA Arka Email | Email | `sphuta-arka-email` | `Pommala-LLC/sphuta-arka-email` |
| SPHUTA Arka SMS | SMS | `sphuta-arka-sms` | `Pommala-LLC/sphuta-arka-sms` |
| SPHUTA Arka WhatsApp | WhatsApp | `sphuta-arka-whatsapp` | `Pommala-LLC/sphuta-arka-whatsapp` |
| SPHUTA Arka Slack | Slack | `sphuta-arka-slack` | `Pommala-LLC/sphuta-arka-slack` |

Each product follows the same architectural skeleton. A shared multi-channel core extraction happens only when genuine duplication becomes painful across three or more channels. Premature abstraction across channels is harder to reverse than boilerplate.

---

## 3. Architecture — shared engine and channel edges

The platform separates channel-agnostic capabilities (the shared engine) from channel-specific capabilities (the edges). Every shared capability works identically regardless of whether the channel is Email, WhatsApp, SMS, or Slack. Only the transport, message format, and channel-specific concerns differ.

```
Shared Engine (21 capabilities)
├── Foundation: core contracts, send orchestration, startup validation
├── Providers: YAML, DB, hybrid resolution
├── Reliability: async, retry, circuit breaker
├── Observability: metrics, health
├── Security: auth, RBAC, role-aware UI
├── Governance: audit, outbox, idempotency, tenant
├── Durability: DLQ, scheduled dispatch
└── Platform surface: REST API, typed client, web console

Channel Edges
├── Email (5): contracts, SMTP transport, recipient handling, MIME construction, Thymeleaf templates
├── WhatsApp (5): contracts, Meta Cloud API transport, Meta templates, webhook status, console screens
├── SMS (TBD): contracts, Twilio/SNS transport, segment handling
└── Slack (TBD): contracts, webhook/API transport, Block Kit messages
```

Core principle: core owns contracts only. Every module owns exactly one capability. Every capability is replaceable via `@ConditionalOnMissingBean`. No module imports another module except through core SPIs. Starters are composition only — they own nothing.

---

## 4. Tier model — OSS, PaaS, SaaS

### OSS — 4 shared capabilities + channel edges

The pure synchronous core. Define flows in YAML, send through the channel transport, fail fast on misconfiguration. No async fan-out, no retry.

| Group | Capability |
|---|---|
| Foundation | Core contracts and SPIs |
| Foundation | Send orchestration |
| Foundation | Startup validation |
| Providers | YAML flow configuration |

OSS channel edges:

| Channel | OSS Capabilities |
|---|---|
| Email | Email contracts, SMTP transport, recipient handling, MIME message construction, Thymeleaf templates |
| WhatsApp | WhatsApp contracts, Meta Cloud API transport, Meta template management |

### PaaS — adds 13 shared + channel-specific edges

The runtime, operational, security, and web console layer. Self-managed and commercial.

| Group | Capability |
|---|---|
| Reliability | Async execution and throttling |
| Reliability | Fixed retry policy |
| Providers | Database flow configuration |
| Providers | Hybrid resolution (DB-first, YAML fallback) |
| Reliability | Circuit breaker |
| Observability | Metrics, tracing, and MDC bridge |
| Observability | Health indicators and readiness probes |
| Security | Authentication, authorization, and RBAC |
| Governance | Immutable audit trail |
| Governance | Transactional outbox |
| Governance | Duplicate send prevention (idempotency) |
| Platform | REST API surface |
| Platform | Web console shell (role-aware, security-integrated) |

PaaS channel edges:

| Channel | PaaS-Only Capabilities |
|---|---|
| WhatsApp | Webhook delivery status, WhatsApp console screens |

### SaaS — adds 4 shared + IdP add-ons

The tenant, durability, and typed-client layer. Adds managed operational delivery.

| Group | Capability |
|---|---|
| Governance | Multi-tenant isolation |
| Durability | Dead letter queue |
| Durability | Scheduled and recurring dispatch |
| Platform | Generic typed client |

SaaS add-ons (by client decision):

| Add-on | Extraction trigger |
|---|---|
| Keycloak integration | Keycloak SDK burden on non-Keycloak consumers |
| Okta integration | Okta SDK burden on non-Okta consumers |
| Entra ID integration | MSAL burden on non-Entra consumers |
| Auth0 integration | Auth0 SDK burden on non-Auth0 consumers |

SaaS operational delivery (no new modules):

| Capability | Delivered by |
|---|---|
| Tenant-scoped access control | Security + tenant with SaaS configuration |
| Tenant administration UI | Web console with tenant-aware screens |
| Delegated tenant administration | Security + tenant + web console |
| Tenant-scoped role-based UI | Tenant + security + web console extension |
| Managed hosting | SPHUTA operational infrastructure |
| Automatic upgrades | SPHUTA operational infrastructure |
| Managed transport operations | SPHUTA operational infrastructure |
| SLA-backed uptime | SPHUTA operational infrastructure |

### Tier capability counts

| Tier | Shared | Email edge | WhatsApp edge | Total |
|---|---|---|---|---|
| OSS | 4 | 5 | 3 | 12 |
| PaaS adds | 13 | 0 | 2 | 15 |
| SaaS adds | 4 | 0 | 0 | 4 |
| **Cumulative** | **21** | **5** | **5** | **31** |

Total inventory: 31 capability modules + 3 composition artifacts = 34 base artifacts. With 4 IdP add-ons = 38 total.

---

## 5. Module inventory

### Shared platform — 21 capabilities + 3 composition

| Group | Capability | OSS | PaaS | SaaS | Hard prerequisites |
|---|---|---|---|---|---|
| **Foundation** | Core contracts and SPIs | ✓ | ✓ | ✓ | None |
| | Send orchestration | ✓ | ✓ | ✓ | Core + channel service adapters |
| | Startup validation | ✓ | ✓ | ✓ | Core + channel validator adapters |
| **Providers** | YAML flow configuration | ✓ | ✓ | ✓ | Core + channel flow config mapper |
| | Database flow configuration | — | ✓ | ✓ | Core + DataSource + JPA + channel DB mapper |
| | Hybrid resolution | — | ✓ | ✓ | Both YAML and DB providers present |
| **Reliability** | Async execution and throttling | — | ✓ | ✓ | Core |
| | Fixed retry policy | — | ✓ | ✓ | Core + channel retry classifier |
| | Circuit breaker | — | ✓ | ✓ | Core + Resilience4j |
| **Observability** | Metrics, tracing, and MDC bridge | — | ✓ | ✓ | Core + Micrometer |
| | Health indicators and readiness | — | ✓ | ✓ | Core + Spring Actuator |
| **Security** | Authentication, authorization, RBAC | — | ✓ | ✓ | Core + Spring Security |
| **Governance** | Immutable audit trail | — | ✓ | ✓ | Core + DataSource + JPA |
| | Transactional outbox | — | ✓ | ✓ | Core + DataSource + JPA + TransactionManager |
| | Duplicate send prevention | — | ✓ | ✓ | Core + DataSource + JPA |
| | Multi-tenant isolation | — | — | ✓ | Core + strategy-specific prerequisites |
| **Durability** | Dead letter queue | — | — | ✓ | Core + DataSource + JPA + channel DLQ handlers |
| | Scheduled and recurring dispatch | — | — | ✓ | Core + DataSource + JPA + channel scheduler handlers |
| **Platform** | REST API surface | — | ✓ | ✓ | Core + DispatcherServlet |
| | Generic typed client | — | — | ✓ | Core + unique channel descriptors |
| | Web console shell | — | ✓ | ✓ | Core + DispatcherServlet |

### Email edge — 5 capabilities

| Group | Capability | OSS | PaaS | SaaS | Hard prerequisites |
|---|---|---|---|---|---|
| **Core** | Email contracts and adapters | ✓ | ✓ | ✓ | Core |
| **Delivery** | SMTP transport | ✓ | ✓ | ✓ | Email core + JavaMail |
| | Recipient handling | ✓ | ✓ | ✓ | Email core |
| | MIME message construction | ✓ | ✓ | ✓ | Email core |
| **Templates** | Thymeleaf template rendering | ✓ | ✓ | ✓ | Email core + Thymeleaf |

### WhatsApp edge — 5 capabilities

| Group | Capability | OSS | PaaS | SaaS | Hard prerequisites |
|---|---|---|---|---|---|
| **Core** | WhatsApp contracts and adapters | ✓ | ✓ | ✓ | Core |
| **Meta** | Meta Cloud API send transport | ✓ | ✓ | ✓ | WhatsApp core + HTTP client + Meta credentials |
| | Meta template management | ✓ | ✓ | ✓ | WhatsApp core + HTTP client + Meta credentials |
| **Runtime** | Webhook delivery status | — | ✓ | ✓ | WhatsApp core + DataSource + JPA + DispatcherServlet |
| | WhatsApp console screens | — | ✓ | ✓ | WhatsApp core + shared console shell |

### Security add-ons — 4 (SaaS, by client decision)

| Add-on | Status | Extraction trigger |
|---|---|---|
| Keycloak integration | SaaS add-on | Keycloak SDK burden on non-Keycloak consumers |
| Okta integration | SaaS add-on | Okta SDK burden on non-Okta consumers |
| Entra ID integration | SaaS add-on | MSAL burden on non-Entra consumers |
| Auth0 integration | SaaS add-on | Auth0 SDK burden on non-Auth0 consumers |

### Cross-reference — external dependency to affected capabilities

| External dependency | Capabilities that fail without it |
|---|---|
| DataSource + JPA | Database flow config, hybrid resolution, audit trail, transactional outbox, duplicate prevention, dead letter queue, scheduled dispatch, webhook delivery status |
| DispatcherServlet | REST API surface, web console shell, webhook delivery status, WhatsApp console screens, multi-tenant isolation (HTTP header mode) |
| Spring Security | Authentication / authorization / RBAC, multi-tenant isolation (JWT claim mode) |
| Micrometer | Metrics, tracing, and MDC bridge |
| Spring Actuator | Health indicators and readiness probes |
| Resilience4j | Circuit breaker |
| JavaMail | SMTP transport |
| Thymeleaf | Thymeleaf template rendering |
| Meta API credentials | Meta Cloud API send transport, Meta template management |
| Shared console shell | WhatsApp console screens |
| Both YAML + DB providers | Hybrid resolution |

---

## 6. Starter composition

Three composition starters per channel. Composition artifacts own no code — they are dependency-only POMs that bundle capability modules.

| Starter | Tier | What it bundles |
|---|---|---|
| `starter` | OSS | 4 shared OSS capabilities + selected channel edges |
| `starter-enterprise` | PaaS | `starter` + 13 PaaS shared capabilities + channel PaaS edges where applicable |
| `starter-full` | SaaS | `starter-enterprise` + 4 SaaS shared capabilities |

**Example for Email:**

| Artifact | Tier |
|---|---|
| `sphuta-arka-email-starter` | OSS |
| `sphuta-arka-email-starter-enterprise` | PaaS |
| `sphuta-arka-email-starter-full` | SaaS |

**Example for WhatsApp:**

| Artifact | Tier |
|---|---|
| `sphuta-arka-whatsapp-starter` | OSS |
| `sphuta-arka-whatsapp-starter-enterprise` | PaaS |
| `sphuta-arka-whatsapp-starter-full` | SaaS |

`starter-full` bundles security directly. Separate IdP bridge artifacts are SaaS add-ons delivered based on client decision.

---

## 7. Activation philosophy

Module presence on the classpath is the activation signal. There are no `enabled` flags, no `provider` switches, no `api.enabled` or `async.enabled` properties. Properties tune behavior, never toggle existence.

### Strict fail-fast rules

| Rule | Behavior |
|---|---|
| Dependency present means intent | If a module JAR is on the classpath, the framework assumes the consumer wants that capability |
| Missing prerequisites fail startup | If a module is present but its hard prerequisites are absent, the application fails to start with a clear diagnostic message |
| Ambiguity fails fast | If competing beans or conflicting module combinations are detected, startup fails rather than silently picking one |
| Properties tune, never toggle | Configuration properties control thresholds, timeouts, policies, and formatting — never whether a capability is active |

### Pluggability mechanism

Every default implementation backs off via `@ConditionalOnMissingBean`. To replace a capability, the consumer provides a bean of the relevant SPI type. The default disappears automatically.

```java
// Example: replace SMTP transport with SendGrid
@Bean
public MailTransport sendGridTransport(SendGridClient client) {
    return message -> {
        // map FinalEmailMessage to SendGrid request and send
    };
}
// JavaMailTransport backs off — no further configuration needed.
```

---

## 8. Dependency direction

```
demo → starter-full → starter-enterprise → starter → core ← all modules
```

Core has no outgoing dependencies except Java itself. No class in core may import anything outside `java.*`. All modules depend inward through core SPIs. No module imports another module directly.

---

## 9. Send flow — single send

```
Consumer
  │
  ▼
FlowService.send(SendCommand)
  │
  ├── [1. Idempotency check]
  │       if idempotency module present → return cached outcome if key already processed
  │
  ├── [2. Send interceptors — beforeSend]
  │       all SendInterceptor beans called in registration order
  │
  ├── [3. Flow resolution]
  │       FlowProvider.resolve(flowKey)
  │       ├── YAML provider
  │       ├── DB provider
  │       └── Hybrid (DB first → YAML fallback on not-found only)
  │
  ├── [4. Recipient / destination normalization]
  │       PolicyHandler.normalize(destinations)
  │       ├── parse and validate each destination
  │       ├── apply policy action per destination type
  │       └── deduplicate
  │
  ├── [5. Empty-destination guard]
  │       throw if no valid primary destination after normalization
  │
  ├── [6. Message construction]
  │       MessageBuilder.build(flow, command)
  │       ├── render template (if template engine present)
  │       ├── merge static flow destinations with runtime destinations
  │       ├── apply sender identity
  │       └── bind attachments / media
  │
  ├── [7. Circuit breaker check]
  │       if circuit-breaker module present → fail fast if circuit is OPEN
  │
  ├── [8. Retry loop]
  │       if retry module present
  │       ├── Attempt 1 → Transport.send(message)
  │       │     ├── Success → done
  │       │     └── Retriable failure → wait delay → Attempt 2
  │       └── Non-retriable failure → throw immediately
  │
  └── [9. Send interceptors — afterSuccess / afterFailure]
            all SendInterceptor beans called
```

### Key invariants

| Invariant | Where enforced |
|---|---|
| `flowKey` not null or blank | `SendCommand` constructor |
| Flow exists | `FlowProvider.resolve(flowKey)` |
| At least one valid primary destination after normalization | Service layer step 5 |
| No duplicate destinations | PolicyHandler step 4 |

---

## 10. Send flow — batch send

```
Consumer
  │
  ▼
BatchDispatcher.dispatch(flowKey, destinations)
  │
  ├── [1. Validate input]  — destinations not null and not empty
  │
  ├── [2. Fan out — for each destination]
  │       ├── semaphore.acquire()                                    ← throttle
  │       ├── Build single-destination SendCommand
  │       ├── Wrap task through all ExecutionContextPropagators
  │       ├── future = CompletableFuture.supplyAsync(task, executor)
  │       │     └── .orTimeout(recipientTimeout)                     ← per-destination deadline
  │       └── future.whenComplete → semaphore.release()              ← throttle release
  │
  ├── [3. Wait for all]
  │       CompletableFuture.allOf(futures).orTimeout(batchTimeout)   ← batch deadline
  │
  └── [4. Return BatchSendResult]
            ├── successes(), failures()
            ├── successCount(), failureCount()
            └── per-destination results
```

**Privacy rule (Email):** Each recipient receives a separate email. The batch dispatcher never puts multiple recipients in a single `to` field.

---

## 11. Provider resolution

```
FlowProvider.resolve(flowKey)
  │
  ├── YAML only (provider-yaml on classpath, no provider-db)
  │     ├── Found → return resolved flow
  │     └── Not found → throw UnknownFlowException
  │
  ├── DB only (provider-db on classpath, no provider-hybrid)
  │     ├── Found → return resolved flow
  │     ├── Not found → throw UnknownFlowException
  │     └── DB error → throw ConfigurationException (propagate immediately)
  │
  └── Hybrid (provider-hybrid on classpath)
        ├── Try DB provider
        │     ├── Found → return flow
        │     ├── Not found → try YAML provider
        │     │     ├── Found → return flow (log DEBUG: hybrid fallback)
        │     │     └── Not found → throw UnknownFlowException
        │     └── DB exception → propagate immediately
        │               NEVER fall back to YAML on DB error
```

---

## 12. Application codes

Two-layer system. Internal domain codes drive logs, exceptions, metrics, and support tickets. HTTP-facing codes drive API responses only.

### Internal code families (Email example)

| Enum | Prefix | Layer |
|---|---|---|
| `EmailValidationCode` | `EMAIL-VAL-*` | Startup validation |
| `EmailRecipientCode` | `EMAIL-RCP-*` | Recipient normalization |
| `EmailProviderCode` | `EMAIL-PROV-*` | Provider resolution |
| `EmailBatchCode` | `EMAIL-BATCH-*` | Batch dispatch |
| `EmailTemplateCode` | `EMAIL-TPL-*` | Template rendering |
| `EmailMessageCode` | `EMAIL-MSG-*` | Message composition |
| `EmailTransportCode` | `EMAIL-TRN-*` | SMTP transport |
| `EmailAsyncCode` | `EMAIL-ASYNC-*` | Async execution |
| `EmailCoreCode` | `EMAIL-CORE-*` | Service lifecycle |
| `EmailPersistenceCode` | `EMAIL-PERSIST-*` | DB read operations |

### WhatsApp code family

| Prefix | Layer |
|---|---|
| `WA-*` | All WhatsApp-specific codes follow the same family pattern |

One enum per prefix family. Never mix prefixes within a single enum.

### `ApplicationCode` interface

Every application code implements the `ApplicationCode` interface, which provides `code()` and `message()` accessors. Each `ValidationCode` also carries a `configCategory()` method that maps the code to the configuration section responsible for the failure.

---

## 13. Exception model

Every exception carries an `ApplicationCode` via the `internalCode()` accessor. The exception hierarchy is channel-specific but follows a common structural pattern.

### Three-constructor discipline

Every typed exception provides exactly three constructors:

```java
// 1. Code + message
public EmailTransportException(EmailTransportCode code, String message) { ... }

// 2. Code + message + cause
public EmailTransportException(EmailTransportCode code, String message, Throwable cause) { ... }

// 3. Code only (uses code's default message)
public EmailTransportException(EmailTransportCode code) { ... }
```

### `InterruptedException` rule

Never swallow `InterruptedException`. Always restore the interrupt flag before throwing a typed exception.

```java
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new EmailTaskInterruptedException(
        EmailAsyncCode.TASK_INTERRUPTED, "Task was interrupted", e);
}
```

### Error response shape

API error responses use a typed details list. The structure includes the HTTP-facing code, a human-readable message, and a list of structured detail objects that carry the internal code, affected field, and diagnostic context. The `internalCode` field is included only when allowed by security and exposure policy.

---

## 14. Validation layers

Five layers. Each owns exactly one responsibility. No layer duplicates another.

| Layer | Trigger | Owns |
|---|---|---|
| HTTP shape | Per HTTP request | Presence and blank checks on DTOs only |
| Command invariants | At construction | Non-null/non-blank flowKey, defensive copies |
| Startup config | Application startup | Flow config correctness, syntax under policy toggles |
| Destination policy | Per send | Parse, validate, deduplicate, apply policy action |
| Send invariants | Pre-dispatch | At least one valid primary destination after normalization |

### Critical rules

- Startup validation collects all errors before throwing — the operator sees the complete picture in one failure
- In multi-tenant mode, validation runs for all loaded tenants and fails globally if any is misconfigured
- The empty-destination guard belongs in the service layer, not in the normalizer
- `@Email` (or equivalent) must not appear on config property classes — it bypasses policy toggles and fails at bind time

---

## 15. Transport and timeout

Each channel has its own transport implementation. The transport SPI is the only module that talks to the external delivery system.

### Email transport (JavaMail)

```yaml
sphuta.arka.email.transport:
  connection-timeout: 5s
  read-timeout: 10s
  write-timeout: 10s
```

Exception translation maps low-level transport exceptions to typed application exceptions with specific codes:

| Root cause | Code | Retriable |
|---|---|---|
| SocketTimeoutException | `EMAIL-TRN-5040` | Yes |
| MailAuthenticationException | `EMAIL-TRN-5043` | No |
| ConnectException / UnknownHostException | `EMAIL-TRN-5045` | Yes |
| SSLException | `EMAIL-TRN-5044` | No |
| Generic MailSendException | `EMAIL-TRN-5020` | Conditional |

### WhatsApp transport (Meta Cloud API)

| Meta API error | Retriable |
|---|---|
| Rate limit exceeded | Yes — with exponential backoff |
| Session window expired | No — caller must handle fallback |
| Template not approved | No — configuration problem |
| Invalid phone number | No — data problem |

---

## 16. Retry and resilience

### Retry policy (PaaS)

Fixed-interval retry with channel-specific retriability classification. The retry policy consults the application code on each failure to determine whether the error is retriable.

Default configuration: 3 attempts, 1-second fixed delay.

Replaceable via `@ConditionalOnMissingBean` — consumers can provide Resilience4j exponential backoff or any custom policy.

### Circuit breaker (PaaS)

Resilience4j circuit breaker protects the transport layer from cascading failures. When the circuit is OPEN, sends fail fast without attempting transport.

### Throttling (PaaS)

Semaphore-based throttling controls maximum concurrent sends. The semaphore is acquired before task submission and released in `whenComplete`, regardless of success or failure.

---

## 17. Async and threading

### Virtual threads (PaaS)

Batch fan-out uses Java 21 virtual threads via `VirtualThreadTaskExecutor`. Each destination gets its own virtual thread. Semaphore-based throttling prevents unbounded concurrency.

### Context propagation

`ExecutionContextPropagator` is an additive SPI. All registered propagator beans are applied at task submission time. Each propagator captures thread-local state from the calling thread and restores it in the virtual thread.

Standard propagators: MDC trace bridge (observability module), tenant context (tenant module).

Propagators are independent of each other. Adding or removing one does not affect the others.

### Tracking ID rule (platform-wide)

Tracking IDs (`correlationId`, `batchId`, `requestId`, `traceId`) are created once at the request entry point (web layer, API filter, or caller) and propagated via MDC. Downstream modules read from MDC only — they never generate their own IDs. Missing context results in `"untraced"` plus a `WARN` log. Fallback UUID generation is explicitly prohibited. Cross-thread MDC propagation is owned by `ExecutionContextPropagator` beans, not by service or dispatcher code.

---

## 18. Security model

### PaaS security foundation

The `security` module provides:

- API key authentication filter (default, always active in PaaS)
- JWT bearer token validation (optional, opt-in)
- OAuth 2.0 Resource Server (optional, opt-in)
- OIDC login / SSO (optional, opt-in)
- RBAC with flow-scoped and sender-scoped permissions (optional, opt-in)

`starter-enterprise` bundles `security` directly. There are no separate IdP artifacts at PaaS tier.

### SaaS security extensions

SaaS adds tenant-scoped RBAC enforcement in both UI and backend, delegated tenant administration, and tenant-specific role mapping.

### IdP bridges (SaaS add-ons by client decision)

| Add-on | Dependency |
|---|---|
| Keycloak | Keycloak SDK |
| Okta | Okta SDK |
| Entra ID | MSAL |
| Auth0 | Auth0 SDK |

UI role mapping from IdPs applies only when the client chooses that integration.

---

## 19. Role-aware UI

Role-aware UI is not a separate artifact. It is a capability of `security` + `web-console`, extended with tenant-scoping in SaaS.

### PaaS — platform-level role-aware UI

| Concern | Behavior |
|---|---|
| Menu visibility | By role |
| Screen access | By role |
| Action/button visibility | By role |
| Route guards | By role |
| Page-level authorization | Per platform role |

Platform roles:

| Role | Access |
|---|---|
| Platform Admin | All screens |
| Operator | Delivery dashboards, retries, health, webhooks |
| Auditor | Audit trail, logs, read-only views |
| Flow Manager | Flow/provider configuration only |
| Support | Status and diagnostic screens, no config changes |

### SaaS — tenant-scoped role-aware UI

All PaaS role-aware behavior, plus:

- Tenant-scoped menus and data isolation
- Delegated tenant administration
- Tenant admin screens
- Tenant-specific role mapping
- Tenant-scoped RBAC enforcement in UI and backend

Tenant roles:

| Role | Access |
|---|---|
| Tenant Admin | Full tenant management |
| Tenant Operator | Tenant delivery operations |
| Tenant Auditor | Tenant audit trail, read-only |
| Tenant Read-Only User | View-only access to tenant data |

---

## 20. Multi-tenancy

Multi-tenant isolation is a SaaS capability. The `tenant` module provides tenant-scoped flows, senders, and configuration with context propagation across virtual threads.

### Tenant resolution modes

| Mode | Needs | Fails if |
|---|---|---|
| Thread-local | Nothing beyond core | Resolver bean missing |
| HTTP header | Web / request context | Web layer absent |
| JWT claim | Security / JWT context | Security absent |

### Startup validation in multi-tenant mode

Validation runs for all loaded tenants and fails globally if any tenant is misconfigured. No tenant gets a pass.

---

## 21. Observability

### Metrics (PaaS)

Micrometer-based metrics with per-flow, per-provider, and per-failure-code tags. MDC trace bridge propagates correlation IDs into structured logs.

### Health indicators (PaaS)

Spring Actuator health indicators for transport readiness (SMTP connectivity for Email, Meta API availability for WhatsApp) and provider readiness (DB connectivity for DB/hybrid providers).

### Structured logging

All log lines carry `flowKey`, `correlationId`, and `batchId` (where applicable) via MDC. Log levels follow a strict rule: `INFO` for lifecycle events, `WARN` for degraded but recoverable situations, `ERROR` for failures that affect the send outcome.

---

## 22. Extension points

Every default is replaceable. Provide a bean of the relevant SPI type and the default backs off automatically.

| To replace | Provide bean of type | Exclude module |
|---|---|---|
| Transport | `MailTransport` / `WhatsAppTransport` | `transport-javamail` / `transport-meta` |
| Retry policy | `RetryPolicy` | `retry-fixed` |
| Circuit breaker | `CircuitBreaker` | `circuit-breaker` |
| Flow source | `FlowProvider` | `provider-yaml` (if fully replacing) |
| Template engine | `TemplateRenderer` + `TemplateProbe` | `template-thymeleaf` |
| Destination handling | `PolicyHandler` | `recipient` |
| Message construction | `MessageBuilder` | `message` |
| Startup validation | `FlowValidator` | `validation` |
| Send orchestration | `FlowService` + `BatchDispatcher` | `service` |
| Executor | `TaskExecutor` | `async` (if fully replacing) |
| Tenant resolution | `TenantResolver` | — |

Additive SPIs (multiple beans supported):

| SPI | Purpose |
|---|---|
| `SendInterceptor` | Pre/post send hooks for metrics, audit, tracing |
| `ExecutionContextPropagator` | Cross-thread context capture and restoration |

---

## 23. Technology stack

| Technology | Version | Purpose |
|---|---|---|
| Java | 21 | Platform baseline — virtual threads, records, switch expressions |
| Spring Boot | 4.x | Auto-configuration, starters, actuator |
| Spring Framework | 7.x | Core container, web, security |
| Jakarta Mail | — | SMTP transport (Email) |
| Thymeleaf | — | Email template rendering |
| Resilience4j | — | Circuit breaker |
| Micrometer | — | Metrics and observability |
| GreenMail | — | SMTP integration testing |
| Testcontainers | — | Database provider testing |
| JUnit Jupiter | 5.11+ | Test framework |

---

## 24. Phased development plan

### Phase 1 — Arka Email OSS

Complete the Email OSS tier. `sphuta-arka-email-starter` bundles core + service + validation + provider-yaml + Email edge modules. The architecture, coding standards, and all SPIs are battle-tested here first.

Exit criteria: A client application adds `sphuta-arka-email-starter`, defines flows in YAML, and sends transactional emails with startup validation and recipient normalization — all on Java 21 / Spring Boot 4.x.

### Phase 2 — Arka Email PaaS (Enterprise)

Add PaaS capabilities to Email. `sphuta-arka-email-starter-enterprise` bundles starter + async, retry, DB/hybrid providers, circuit breaker, observability, health, security, audit, outbox, idempotency, REST API, and web console.

Exit criteria: Enterprise clients get async fan-out, retry, runtime-managed flows, observability, crash-safe outbox, duplicate prevention, role-aware web console, and RBAC — by swapping the starter dependency.

### Phase 3 — Arka WhatsApp OSS

Apply the proven Arka pattern to WhatsApp. `sphuta-arka-whatsapp-starter` bundles core + service + validation + provider-yaml + WhatsApp OSS edge modules (contracts, Meta transport, Meta templates).

What carries over unchanged: dependency direction, `@ConditionalOnMissingBean` pluggability, classpath-presence activation, fail-fast prerequisites, tracking ID propagation, `ExecutionContextPropagator`, application code pattern, three-constructor exception discipline, error response shape.

What is WhatsApp-specific: Meta Cloud API integration, 24-hour session window logic, template approval status, E.164 phone normalization, media message types, interactive messages, per-phone-number rate limit handling.

Exit criteria: A client application adds `sphuta-arka-whatsapp-starter`, defines WhatsApp flows in YAML, and sends template or session messages through the Meta Cloud API.

### Phase 4 — Arka WhatsApp PaaS (Enterprise)

Same enterprise hardening pattern. `sphuta-arka-whatsapp-starter-enterprise` adds all PaaS capabilities plus webhook delivery status and WhatsApp console screens.

### Phase 5 — Arka Email + WhatsApp SaaS

Add SaaS capabilities (tenant, DLQ, scheduled dispatch, typed client) to both channels. `starter-full` for each channel bundles `starter-enterprise` + SaaS modules. IdP add-ons delivered based on client decision.

### Phase 6 — Cloud editions

Fully managed SaaS at `cloud.sphuta.net`. Hosting, managed transport, tenant provisioning API, monitoring dashboards, SSO federation, SLA-backed uptime.

### Build ordering rationale

Email first → WhatsApp second. SMS is structurally closest to Email (next after WhatsApp). Slack is the simplest transport but a different message model. A shared core extraction happens only when genuine duplication becomes painful across three or more channels.

---

## 25. Naming conventions

### Email

| Layer | Value |
|---|---|
| Parent POM | `sphuta-arka-email` |
| GroupId | `net.sphuta.arka.email` |
| Java package | `net.sphuta.arka.email` |
| Spring property prefix | `arka.email` |
| Module prefix | `sphuta-arka-email-{module}` |
| GitHub | `Pommala-LLC/sphuta-arka-email` |

### WhatsApp

| Layer | Value |
|---|---|
| Parent POM | `sphuta-arka-whatsapp` |
| GroupId | `net.sphuta.arka.whatsapp` |
| Java package | `net.sphuta.arka.whatsapp` |
| Spring property prefix | `arka.whatsapp` |
| Module prefix | `sphuta-arka-whatsapp-{module}` |
| GitHub | `Pommala-LLC/sphuta-arka-whatsapp` |

---

## 26. Repository structure

Each channel is a standalone repository with the following structure:

```
sphuta-arka-{channel}/
├── pom.xml                              ← parent POM
├── README.md                            ← this document (channel-specific)
├── CODING_STANDARDS.md                  ← coding standards (shared baseline)
├── sphuta-arka-{channel}-core/          ← contracts only
├── sphuta-arka-{channel}-service/       ← send orchestration
├── sphuta-arka-{channel}-validation/    ← startup validation
├── sphuta-arka-{channel}-provider-yaml/ ← YAML flow source
├── sphuta-arka-{channel}-provider-db/   ← DB flow source (PaaS)
├── sphuta-arka-{channel}-provider-hybrid/ ← hybrid resolution (PaaS)
├── sphuta-arka-{channel}-async/         ← async dispatch (PaaS)
├── sphuta-arka-{channel}-retry-fixed/   ← fixed retry (PaaS)
├── sphuta-arka-{channel}-circuit-breaker/ ← circuit breaker (PaaS)
├── sphuta-arka-{channel}-observability/ ← metrics (PaaS)
├── sphuta-arka-{channel}-health/        ← health indicators (PaaS)
├── sphuta-arka-{channel}-security/      ← auth and RBAC (PaaS)
├── sphuta-arka-{channel}-audit/         ← audit trail (PaaS)
├── sphuta-arka-{channel}-outbox/        ← transactional outbox (PaaS)
├── sphuta-arka-{channel}-idempotency/   ← duplicate prevention (PaaS)
├── sphuta-arka-{channel}-web/           ← REST API (PaaS)
├── sphuta-arka-{channel}-web-console/   ← admin UI (PaaS)
├── sphuta-arka-{channel}-tenant/        ← multi-tenancy (SaaS)
├── sphuta-arka-{channel}-dlq/           ← dead letter queue (SaaS)
├── sphuta-arka-{channel}-scheduler/     ← scheduled dispatch (SaaS)
├── sphuta-arka-{channel}-client/        ← typed client (SaaS)
├── sphuta-arka-{channel}-starter/       ← OSS composition
├── sphuta-arka-{channel}-starter-enterprise/ ← PaaS composition
├── sphuta-arka-{channel}-starter-full/  ← SaaS composition
├── ... channel-specific edge modules ...
└── sphuta-arka-{channel}-demo/          ← demo applications (never published)
```

---

## License

Apache 2.0 (OSS) · Commercial License (Enterprise and Cloud)

© 2026 Pommala LLC
