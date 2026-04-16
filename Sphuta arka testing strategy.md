#  Arka — Testing Strategy (Full Scope)

> **Scope:** Complete testing strategy across all tiers (OSS / PaaS / SaaS) and all channels (Email, WhatsApp, Slack, SMS)
> **Architecture:** Modular, classpath-activated, SPI-driven, shared engine + channel adapter
> **Status:** Aligned with frozen 33-artifact architecture, strict fail-fast activation, and coding standards

---

## Contents

1. [Full-Scope Tier Map](#1-full-scope-tier-map)
2. [Common Test Families Across All Channels](#2-common-test-families-across-all-channels)
3. [Channel-by-Channel Full Scope](#3-channel-by-channel-full-scope)
4. [Cross-Tier Cross-Channel Mandatory Test Categories](#4-cross-tier-cross-channel-mandatory-test-categories)
5. [Full-Scope Coverage Map](#5-full-scope-coverage-map)

---

## 1. Full-Scope Tier Map

### Tier 1 — OSS / Base

The channel runtime foundation.

**Applicable test scope:**

- Core contract tests
- Model / builder / invariant tests
- Code enum / application code tests
- Exception hierarchy tests
- Validation-layer tests
- Provider tests
- Message construction tests
- Transport tests
- Async tests
- Retry tests
- Starter / auto-config backoff tests
- Send-flow integration tests

The base platform already includes contracts-only core, service orchestration, validation, transport, async, retry, and send-flow semantics.

### Tier 2 — Enterprise / PaaS

Adds operational hardening and managed platform behavior.

**Applicable test scope (adds to OSS):**

- DB provider tests
- Hybrid provider fallback tests
- Circuit-breaker tests
- Observability tests
- Health/readiness tests
- Security tests
- Web/client contract tests
- Operational integration tests

Observability, health, security, DB/hybrid provider, and circuit-breaker are explicit modules above the base starter path.

### Tier 3 — Cloud / SaaS

Adds platform-governed reliability, tenancy, and managed delivery guarantees.

**Applicable test scope (adds to Enterprise):**

- Audit persistence tests
- Outbox recovery tests
- Idempotency replay tests
- Tenant isolation tests
- Tenant async-propagation tests
- Multi-tenant startup validation tests
- Cross-tenant security tests
- Cloud-operability tests

Audit, outbox, idempotency, and tenant are part of the full enterprise/platform capability and production readiness path.

---

## 2. Common Test Families Across All Channels

These apply to Email, WhatsApp, Slack, and SMS because the architecture intentionally reuses the same modular pattern across channels.

| Test family | OSS | Enterprise/PaaS | Cloud/SaaS |
|---|---|---|---|
| Core contracts / SPIs | Yes | Yes | Yes |
| Model invariants / builders | Yes | Yes | Yes |
| Application codes / exception model | Yes | Yes | Yes |
| Architecture / boundary rules | Yes | Yes | Yes |
| Auto-configuration / bean backoff | Yes | Yes | Yes |
| Validation layers | Yes | Yes | Yes |
| Provider resolution | Yes | Yes | Yes |
| Transport translation | Yes | Yes | Yes |
| Async / throttling / timeout | Yes | Yes | Yes |
| Retry logic | Yes | Yes | Yes |
| Circuit breaker | Optional | Yes | Yes |
| Web/API contract | Optional | Yes | Yes |
| Security | No | Yes | Yes |
| Observability / health | No | Yes | Yes |
| Audit | No | Optional | Yes |
| Outbox | No | Optional | Yes |
| Idempotency | No | Optional | Yes |
| Tenant isolation | No | Optional | Yes |
| End-to-end system tests | Yes | Yes | Yes |

---

## 3. Channel-by-Channel Full Scope

### A. Email

The Email channel has the fullest documented module set: provider, message, transport, template, async, retry, circuit breaker, web, client, observability, health, security, audit, outbox, idempotency, tenant.

#### Email — OSS / Base

- Core/API/SPI tests
- `EmailSendCommand` invariant tests
- Validation-layer tests
- Recipient normalization and dedupe tests
- Message builder tests
- YAML provider tests
- JavaMail transport tests
- Timeout translation tests
- Async timeout and semaphore tests
- Retry classification tests
- Send-flow integration tests

#### Email — Enterprise / PaaS (adds)

- DB provider repository tests
- Hybrid fallback tests
- Web controller + exception handler tests
- JWT/API-key/role tests
- Micrometer metrics tests
- Health/readiness tests
- Circuit-breaker state tests
- Client contract tests

#### Email — Cloud / SaaS (adds)

- Audit append-only tests
- Outbox transaction + restart recovery tests
- Idempotency replay/TTL tests
- Tenant isolation tests
- Tenant-aware provider tests
- Async tenant propagation tests
- Multi-tenant startup validation tests

### B. WhatsApp

The shared test backbone applies, plus WhatsApp-specific channel rules. The channel-extension guide defines Meta transport concerns such as the 24-hour session window, approved templates, E.164 numbers, and rate limits.

#### WhatsApp — OSS / Base

- Core/SPI/model tests
- Command invariant tests
- Transport request/response mapping tests
- Phone validation tests
- Template payload rendering tests
- YAML/provider tests
- Retry classification for Meta errors
- Async fan-out tests if batch sending exists

#### WhatsApp — Enterprise / PaaS (adds)

- Webhook / delivery-status tests if exposed
- Security tests if web surface exists
- Observability metrics/traces for send outcomes
- Rate-limit handling and backoff integration tests

#### WhatsApp — Cloud / SaaS (adds)

- Tenant isolation by phone-number / template namespace
- Idempotent send-key behavior
- Audit of outbound message + status transitions
- Outbox recovery for delayed template sends
- Cross-tenant session-window enforcement tests

#### WhatsApp-Specific Mandatory Cases

| Case | What to test |
|---|---|
| 24-hour conversation window | Enforcement, active vs expired, fallback behavior |
| Approved-template enforcement | APPROVED status check, rejection on unapproved |
| Session vs template routing | Delivery mode resolution per flow config |
| E.164 normalization | Format validation, rejection of non-E.164 |
| Meta rate-limit retry | Backoff behavior on 429 response |
| Non-retriable template-not-approved | Immediate failure, no retry |

### C. Slack

Same backbone, plus Slack-specific transport/API concerns: webhook vs Web API, Block Kit payloads, bot/channel authorization, and `Retry-After` handling.

#### Slack — OSS / Base

- Core/SPI/model tests
- Slack payload builder tests
- Block Kit serialization tests
- Webhook transport tests
- API transport tests
- Retry classification for network vs permanent Slack failures
- Async delivery tests

#### Slack — Enterprise / PaaS (adds)

- Auth/token tests
- Rate-limit integration tests using `Retry-After`
- Observability tests
- Web/admin surface tests if exposed

#### Slack — Cloud / SaaS (adds)

- Tenant/workspace isolation tests
- Audit of posted message and channel identity
- Outbox recovery for delayed post attempts
- Idempotent suppression of duplicate posts
- Per-workspace security-policy tests

#### Slack-Specific Mandatory Cases

| Case | What to test |
|---|---|
| Webhook vs API route selection | Correct transport used per flow config |
| Block Kit validation | Payload structure correctness |
| `ratelimited` handling | Backoff behavior with `Retry-After` header |
| `channel_not_found` classification | Non-retriable, immediate failure |
| `token_revoked` classification | Non-retriable, permanent until config change |
| Bot invited/not-invited behavior | Authorization failure paths |

### D. SMS

Same backbone, plus SMS-specific concerns: E.164 numbers, segmentation, encoding, carrier filtering, and opt-out rules.

#### SMS — OSS / Base

- Core/SPI/model tests
- Phone normalization tests
- Message segmentation tests
- GSM-7 vs UCS-2 encoding tests
- Twilio/SNS transport tests
- Retry classification tests
- Async bulk-send tests

#### SMS — Enterprise / PaaS (adds)

- Delivery callback/status mapping tests
- Observability tests
- Security tests if HTTP/admin surface exists
- Rate-limit and provider-failure integration tests

#### SMS — Cloud / SaaS (adds)

- Tenant isolation by sender/profile
- Audit logging for sends and delivery states
- Idempotent duplicate-SMS suppression
- Outbox recovery for queued sends
- Per-tenant opt-out isolation and enforcement

#### SMS-Specific Mandatory Cases

| Case | What to test |
|---|---|
| Segment counting | Correct split at 160 (GSM-7) / 70 (UCS-2) boundaries |
| GSM-7/UCS-2 boundary behavior | Encoding detection, fallback to UCS-2 on special chars |
| E.164 enforcement | Format validation, rejection of non-E.164 |
| Carrier rejection classification | Retriable vs permanent carrier errors |
| STOP/opt-out handling | Compliance enforcement if implemented |
| Transactional vs promotional routing | Correct routing per message category |

---

## 4. Cross-Tier Cross-Channel Mandatory Test Categories

These are the categories that should exist across the entire platform.

### 4.1 Unit Tests

Records, builders, enums, codes, exceptions, translators, utilities.

### 4.2 Architecture Tests

- Core purity (no forbidden imports, `java.*` only)
- Module boundary enforcement (one capability per module)
- Dependency direction (no shared module imports channel)
- No `common`, `utils`, or `support` packages

### 4.3 Auto-Configuration Tests

- Activate on classpath
- Back off on user bean (`@ConditionalOnMissingBean`)
- Decorator/wrapper becomes single effective bean
- Additive SPI collection behavior
- Fail-fast when prerequisites missing

### 4.4 Validation Tests

- Input shape (DTO level)
- Command invariants (construction time)
- Startup fail-fast with collect-all errors
- Runtime destination policy
- Final pre-dispatch invariants (empty-destination guard)

### 4.5 Provider Tests

- YAML / DB / hybrid resolution
- Not-found vs infrastructure failure behavior
- Hybrid: DB not found → YAML fallback; DB error → propagate immediately
- Tenant-scoped lookup where applicable

### 4.6 Transport Tests

- Request building
- Exception translation (every raw exception → typed with code)
- Timeout handling (connect, read, write)
- Auth and connectivity failures
- No raw framework exceptions escaping

### 4.7 Async Tests

- Virtual-thread executor selection
- Semaphore throttle acquisition/release
- Per-item timeout
- Batch timeout
- `InterruptedException` restoration + typed wrap
- `RejectedExecutionException` → `BatchDispatcherUnavailableException`
- Context propagation (MDC, tenant, security)

### 4.8 Retry / Resilience Tests

- Retriable vs non-retriable classification per code
- Max attempts / delay enforcement
- No retry when module absent (single-attempt)
- Circuit breaker: OPEN / HALF_OPEN / CLOSED transitions
- Fast fail when circuit open
- Interaction with async throttling

### 4.9 Web/API Tests

- Endpoint contract (send, batch, health)
- DTO validation at controller level
- Typed error response shape
- HTTP status mapping from internal exceptions
- 400 / 404 / 422 / 500 / 502 / 503 / 504 responses

### 4.10 Security Tests

- Unauthenticated → 401
- Wrong role → 403
- Per-flow role override
- API key auth
- JWT auth
- Combined tenant + security path

### 4.11 Observability Tests

- Counters/timers emitted with correct tags
- Trace bridge creates child span
- MDC enrichment visible in async logs
- Channel observation convention applied (metric prefix, tag schema)

### 4.12 Persistence Tests

- Repositories (read-only for providers, read-write for outbox/DLQ/scheduler)
- Audit append-only behavior
- Outbox state machine (NEW → DISPATCHING → SENT / FAILED / DEAD)
- Idempotency store semantics (key lookup, TTL expiry)

### 4.13 Tenant Tests

- Lookup isolation (tenant A cannot resolve tenant B flows)
- Security isolation (cross-tenant access blocked)
- Async propagation (tenant context preserved in virtual threads)
- Validation across all tenants at startup
- Resolution strategy tests (THREAD_LOCAL, HEADER, JWT_CLAIM)

### 4.14 End-to-End System Tests

- Full single-send path (service → validation → transport → success)
- Full batch/fan-out path (dispatcher → per-recipient → aggregate)
- Provider + message + transport + retry + async interaction
- Failure-path system tests (retry exhaustion → outbox → DLQ)

### 4.15 Durability Tests

- Crash-before-send (outbox entry survives restart)
- Restart recovery (pending entries dispatched after restart)
- Duplicate request replay (idempotency key returns cached outcome)
- Eventual cleanup (completed entries purged per retention policy)

---

## 5. Full-Scope Coverage Map

### By Scope Area and Tier

| Scope area | OSS | Enterprise/PaaS | Cloud/SaaS |
|---|---|---|---|
| Core/unit | Required | Required | Required |
| Architecture rules | Required | Required | Required |
| Auto-config | Required | Required | Required |
| Validation | Required | Required | Required |
| Provider | Required | Required | Required |
| Transport | Required | Required | Required |
| Async | Required | Required | Required |
| Retry | Required | Required | Required |
| Circuit breaker | Optional | Required | Required |
| Web/API | Optional | Required | Required |
| Security | No | Required | Required |
| Observability | No | Required | Required |
| Health/readiness | No | Required | Required |
| DB persistence | No | Required | Required |
| Audit | No | Optional | Required |
| Outbox | No | Optional | Required |
| Idempotency | No | Optional | Required |
| Tenant isolation | No | Optional | Required |
| Channel-specific rules | Required | Required | Required |
| Full E2E | Required | Required | Required |

### By Channel and Tier — Test Case Count Estimates

| Channel | OSS cases | PaaS adds | SaaS adds | Total |
|---|---|---|---|---|
| Email | ~80 | ~50 | ~40 | ~170 |
| WhatsApp | ~60 | ~35 | ~30 | ~125 |
| Slack (reserved) | ~50 | ~30 | ~25 | ~105 |
| SMS (reserved) | ~50 | ~30 | ~25 | ~105 |
| Shared engine | ~40 | ~30 | ~20 | ~90 |
| **Total** | **~280** | **~175** | **~140** | **~595** |

### Tools by Test Layer

| Test layer | Tools |
|---|---|
| Unit tests | JUnit 5, AssertJ |
| Architecture tests | ArchUnit |
| Auto-config tests | `ApplicationContextRunner` |
| Validation tests | JUnit 5, custom assertion helpers |
| Provider tests | `@DataJpaTest`, H2 |
| Transport tests (Email) | GreenMail |
| Transport tests (WhatsApp) | `MockRestServiceServer` |
| Transport tests (Slack) | `MockRestServiceServer` |
| Transport tests (SMS) | `MockRestServiceServer` |
| Async tests | `CountDownLatch`, `CompletableFuture`, mock clock |
| Security tests | `@WithMockUser`, `MockMvc` |
| Persistence tests | `@DataJpaTest`, Testcontainers |
| E2E tests | `@SpringBootTest`, Testcontainers |
| Load tests | JMH, Gatling |

---

## Summary

The full-scope testing strategy follows one principle: **test what the module owns, at the tier where it activates, for the channel where it applies.**

- Common platform tests cover every channel
- Tier-specific extensions add as modules enter the classpath
- Channel-specific rule tests validate transport physics, validation rules, and message format contracts that differ between Email, WhatsApp, Slack, and SMS

The test pyramid applies at every level: unit tests are always required, integration tests are required for every runtime path, and end-to-end tests are required before production across all tiers.

---

* Arka — Testing Strategy (Full Scope) — All Tiers, All Channels — Frozen Standard — March 2026*
