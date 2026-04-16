# SPHUTA Platform — Coding Standards Implementation Guide

**Platform:** SPHUTA — Enterprise Integration Platform
**Baseline:** Java 21+ / Spring Framework 7.x / Spring Boot 4.x
**Status:** Finalized and locked. Deviations require explicit review.

> This guide operationalizes the General Coding Standards document.
> It defines concrete templates, enforcement rules, and tooling gates
> for every area the standards document leaves underspecified.
> All SPHUTA platform modules must comply with this guide.

---

## Contents

1. [Java Version Baseline](#1-java-version-baseline)
2. [Module Architecture](#2-module-architecture)
3. [Package Structure](#3-package-structure)
4. [Import Discipline](#4-import-discipline)
5. [Interface and Implementation Design](#5-interface-and-implementation-design)
6. [Null Handling and Optional](#6-null-handling-and-optional)
7. [Validation Layers](#7-validation-layers)
8. [Application Codes](#8-application-codes)
9. [Exception Model](#9-exception-model)
10. [Exception Handler](#10-exception-handler)
11. [Transport and Timeout Handling](#11-transport-and-timeout-handling)
12. [Retry Policy](#12-retry-policy)
13. [Resilience](#13-resilience)
14. [Async and Threading](#14-async-and-threading)
15. [Multi-Tenancy](#15-multi-tenancy)
16. [Logging Standards](#16-logging-standards)
17. [Tracking IDs and Correlation](#17-tracking-ids-and-correlation)
18. [Code Comments and Javadoc](#18-code-comments-and-javadoc)
19. [DB and Persistence](#19-db-and-persistence)
20. [Business Rules, Validation, and Conditions in Code](#20-business-rules-validation-and-conditions-in-code)
21. [Tooling Enforcement Summary](#21-tooling-enforcement-summary)

---

## 1. Java Version Baseline

**Minimum required: Java 21**

No SPHUTA module is compatible with Java 17 or below.

### Features actively used across the platform

| Version | Features |
|---|---|
| Java 8 | Lambdas, method references, streams, `CompletableFuture`, `Objects.requireNonNull`, defensive copying |
| Java 11+ | `String.isBlank()`, `List.of/copyOf`, `Map.of/copyOf`, `String.formatted()`, `var` |
| Java 17 | Records for DTOs and immutable value objects, switch expressions with `case ->` and `yield` |
| Java 21 | Virtual threads via `VirtualThreadTaskExecutor`, `SequencedSet`, `List.getFirst()` |

### Features deliberately not used platform-wide

| Feature | Decision |
|---|---|
| Pattern matching for `instanceof` | Use only where cast chains exist and it makes code meaningfully clearer |
| Sealed hierarchies | Use only for genuine algebraic types — not for open SPIs or strategy patterns |
| Record patterns | Records are used for construction and accessor access only |
| Scoped values | Not appropriate for production starters |
| Structured concurrency | Virtual threads alone are sufficient for current platform needs |

**Rule:** Do not add a Java feature to satisfy a checklist. Add it only when it makes the code meaningfully clearer.

### Compiler flags — mandatory

| Flag | Purpose | Gate |
|---|---|---|
| `-Xlint:all` | Enable all compiler warnings | CI build failure |
| `-Werror` | Promote warnings to errors | CI build failure |
| `-Xlint:-processing` | Suppress annotation-processor warnings (false positives with APT) | Exemption only |

These flags apply to all handwritten source sets (main and test). Generated sources are exempt.

---

## 2. Module Architecture

### Core principles

```
core owns contracts only.
every module owns exactly one capability.
every capability is replaceable via @ConditionalOnMissingBean.
no module imports another module except through core SPIs.
composition modules wire things together but do not own business rules.
```

### Dependency direction

Dependencies flow inward toward contracts and core. No circular dependencies. No sideways coupling between implementation modules unless explicitly justified and documented.

```
demo → starter-full → starter → core ← all implementation modules
```

Core has no outgoing dependencies except Java itself (and approved Spring utilities from `spring-core` where documented).

### Module activation

Module presence on the classpath is the activation signal. There are no `enabled: true/false` flags for core modules. Modules requiring schema changes or data migrations are the only exceptions requiring explicit opt-in.

### Module ownership

Every module must document a module ownership table:

| Column | Purpose |
|---|---|
| Module | Artifact name |
| Owns | The concrete classes and capabilities this module provides |
| Replace by providing | The SPI bean(s) a consumer provides to replace this module's default |

### SPI interfaces in core

All SPI interfaces live in core. Implementation modules implement them. Core never imports an implementation module. Every module that provides a default implementation must back off via `@ConditionalOnMissingBean` when a consumer provides their own bean.

### Bean composition rule

There must be exactly one **effective** bean per primary SPI at injection time. Wrapper modules decorate the selected effective bean — they must not register themselves as competing beans alongside it.

```
base implementation selected first
       ↓
optional decorators wrap the selected base
       ↓
downstream services depend only on the final effective bean
```

---

## 3. Package Structure

### Naming conventions

Every module must define and lock its naming conventions:

| Layer | Convention |
|---|---|
| Parent POM | `sphuta-{domain}-{subdomain}` |
| Module artifact IDs | `sphuta-{domain}-{subdomain}-{module}` |
| Base Java package | `net.sphuta.{domain}.{subdomain}` |
| Spring property prefix | `sphuta.{domain}.{subdomain}` |
| Auto-configuration package | `net.sphuta.{domain}.{subdomain}.{module}.autoconfigure` |

### Core package map template

```
net.sphuta.{domain}.{subdomain}
├── api              ← public consumer contracts only
├── spi              ← framework extension points only
├── model
│   ├── {domain}     ← internal resolved models
│   ├── enums        ← all enum types in separate files
│   └── builder      ← builders for model records
├── code             ← ApplicationCode interface + all code enums
├── exception        ← exception hierarchy (single dedicated package)
└── validation       ← validators
```

### Builder placement — separate files in `builder` subpackages

Builders are never inner classes. Every record with 4+ fields or a mix of required and nullable fields gets a companion builder in a `builder` subpackage.

```
model/
├── SomeRecord.java
├── AnotherRecord.java
├── builder/
│   ├── SomeRecordBuilder.java
│   └── AnotherRecordBuilder.java
```

**Builder rules:**

| Rule | Detail |
|---|---|
| One builder per file | Follows "one public type per file" standard |
| Builder naming | `{RecordName}Builder` |
| Builder location | `{domain-package}.builder` subpackage |
| Builder responsibility | Mutable construction only. No business logic. |
| Record compact constructor ownership | Owns all invariants — `Objects.requireNonNull`, `List.copyOf()`, null-to-default |
| Builder `build()` delegation | Calls the record constructor. Never reflection. Never direct field assignment. |
| Cross-reference Javadoc | Record `@see`s the builder. Builder `@see`s the record. |

**When a builder is needed:**

| Condition | Builder? |
|---|---|
| Record has 4+ fields | Yes |
| Record has a mix of required and nullable fields | Yes |
| Record has collection fields requiring defensive copies | Yes |
| Record has 1-3 fields, all required, no collections | No — direct constructor is clear |
| Record has a factory method or sentinel (`UNKNOWN`, `EMPTY`) | Factory method may suffice; evaluate case by case |

### Exception placement — single dedicated `exception` package

All exceptions for a module live in a single dedicated `exception` package as a top-level package. Not nested under any parent package.

**Exception package rules:**

| Rule | Detail |
|---|---|
| Single package per module | All exceptions in `exception`. No splits. No scattering across domain packages. |
| Import direction | `exception` may import from `code` (for `ApplicationCode` types). `code` must never import from `exception`. |
| No exception class outside this package | Enforced by ArchUnit test in CI |

### Hard rules

- One public class per file — no hidden helper classes inside service or controller files
- No nested DTOs inside controllers
- No mixing DTOs and entities in the same package
- No inner builder classes anywhere
- `api` contains only what application code imports and uses directly
- `spi` contains only framework extension points — application code must never import from `spi`
- Demo and sample applications belong in a dedicated `demo` module — never inside any starter source tree
- No vague catch-all packages (`common`, `utils`, `support`) — every class must have a precise, responsibility-aligned home
- Generated and handwritten sources must never share the same package
- Package with more than 10 public types must be evaluated for splitting by sub-responsibility

### Generated vs handwritten source separation

| Source type | Location |
|---|---|
| Handwritten | `src/main/java/` |
| Generated | `build/generated/sources/{tool}/` |

No handwritten `.java` file may be placed in any package that also contains generated `.java` files, and vice versa. Build tooling should fail on violations.

### Library module API surface

SPHUTA platform modules are primarily libraries and tooling, not web services.

| Web-app concept | Library module equivalent |
|---|---|
| Controller | Public API types in `api` package — delegation and translation only, no business logic |
| Service interface | SPI interfaces in `spi` package — extension points for framework integrations |
| DTO | Records in `model` subpackages |
| Exception handler | Exception hierarchy in `exception` package + centralized handler where web layer exists |
| Validation layer | Validators in `validation` package + extension SPI |

---

## 4. Import Discipline

### Import ordering convention

Standard block order with blank line separators between groups:

```java
// Block 1: Java standard library
import java.time.Duration;
import java.util.List;
import java.util.Objects;

// Block 2: Jakarta
import jakarta.validation.constraints.NotBlank;

// Block 3: Third-party libraries
import org.jspecify.annotations.NullMarked;
import org.slf4j.Logger;

// Block 4: Spring Framework
import org.springframework.stereotype.Component;
import org.springframework.util.Assert;

// Block 5: SPHUTA platform-internal
import net.sphuta.moduleA.api.SomeContract;
import net.sphuta.moduleB.code.SomeCode;
```

Within each block, imports are sorted alphabetically. No exceptions.

### Wildcard import ban

Wildcard imports (`import java.util.*`) are prohibited in all handwritten source files.

**Exception:** Generated code may use wildcard imports. Applies only to files under the generated sources directory.

### Unused import removal — build gate

Unused imports are build failures, not style preferences.

| Enforcement | Mechanism |
|---|---|
| Primary | Spotless auto-removal on every build |
| Secondary | Checkstyle `UnusedImports` with severity = error |

CI must reject any code containing unused imports.

### Static import policy

| Context | Rule |
|---|---|
| Test classes | Static imports allowed for AssertJ (`assertThat`), Mockito (`mock`, `when`, `verify`), JUnit 5 assertions |
| Production — enum switches | Static imports allowed for enum constants in exhaustive switch blocks |
| Production — all other cases | Prefer qualified access |

Static imports must not cross the dependency direction.

### Tooling

| Tool | Role | Gate |
|---|---|---|
| Spotless (Gradle) | Format enforcement, import ordering, unused import removal | `./gradlew spotlessCheck` in CI |
| Checkstyle | `CustomImportOrder`, `AvoidStarImport`, `UnusedImports` | CI build failure |
| `.editorconfig` | IDE-level consistency for indent, charset, trailing whitespace | Committed to repo root |

---

## 5. Interface and Implementation Design

### Placement rules

| Type | Package |
|---|---|
| Public API interfaces | `api` |
| Framework extension point interfaces | `spi` |
| Default implementations | Module-specific packages (`service`, `provider`, `transport`, etc.) |

### Do not extract interfaces unnecessarily

Only extract an interface when there is a genuine extension point or testability need. Keep internal concrete classes concrete when no extension is needed. Downstream code depends on the effective contract, not specific implementations.

### Repository rule

Modules that are read-only must expose only `findBy*` methods on repositories. No `save`, `delete`, `update`, or mutation methods.

### SPI Javadoc requirements

Every SPI interface must document its contract type, thread safety, mutability, registration mechanism, and all behavioral invariants. See [Section 18.5](#185-spi-interface-javadoc) for the required template.

---

## 6. Null Handling and Optional

### Decision per case

| Pattern | Use case |
|---|---|
| `Objects.requireNonNull(...)` | Required collaborators, constructors, mandatory arguments |
| Null-to-default normalization | Record compact constructors, DTO normalization, config normalization |
| Plain `if (x == null)` check | Internal branching logic in providers, mappers, validators, factories |
| `Optional<T>` return type | Repository lookups, parsers, find-or-miss operations, genuine absence |
| `Optional<T>` for optional bean | Spring auto-configuration, optional collaborators |

### What must never happen

| Anti-pattern | Why |
|---|---|
| `Optional` field in JPA entity | JPA does not map Optional fields cleanly |
| `Optional` field in DTO or config property | Hurts serialization and binding clarity |
| `Optional` as a method parameter | Burdens the caller — bad API design |
| `Optional` return from every internal helper | Over-engineers simple code |
| `Optional.get()` without presence check | Null danger in a different form |
| `Optional<Collection<T>>` | Prefer empty collection |

---

## 7. Validation Layers

### Multi-layer validation model

Validation must be split by responsibility. Each layer owns exactly one responsibility. No layer duplicates another.

| Layer | Trigger | Owns |
|---|---|---|
| Input shape | Per HTTP request or API call | Presence and blank checks only (annotations on DTOs) |
| Command invariants | At command/record construction | Non-null/non-blank fields, defensive copies |
| Startup config | Application startup | Configuration correctness, reference integrity, syntax validation under policy toggles |
| Runtime policy | Per operation | Domain-specific normalization, conditional enforcement, deduplication |
| Final invariants | Pre-side-effect | Last-chance guards before dispatch or persistence |

### Critical rules

- Startup validation collects **all** errors before throwing — operator sees the complete picture in one failure
- In multi-tenant mode, validation runs for **all** loaded tenants and fails globally if any is misconfigured
- `@Email` and similar annotations must **not** appear on config property classes when policy toggles exist — they bypass toggle logic and fail at bind time
- Runtime validation must enforce final safety before side effects happen

### Validation behavior matrix template

Every module must document which constraints are always-enforced vs toggle/mode-dependent.

#### Always enforced (not mode/toggle-dependent)

| Constraint Type | Trigger | Behavior |
|---|---|---|
| Missing required field | Null or blank on required field | Error — cannot be suppressed |
| Forbidden field present | Prohibited field detected in input | Error — cannot be suppressed |
| Invalid enum value | Value not in declared constant set | Error — cannot be suppressed |
| Constraint violation (min/max/length/pattern) | Value outside declared bounds | Error — cannot be suppressed |
| One-of violation | Zero or 2+ fields present in exclusive group | Error — cannot be suppressed |
| Mutually exclusive violation | Both fields in exclusive pair present | Error — cannot be suppressed |
| Conditional required violation | Discriminator has activating value but target is null | Error — cannot be suppressed |
| Final invariant failure | Pre-side-effect guard fails | Error — cannot be suppressed |

#### Mode/toggle-dependent

| Constraint Type | When active | Behavior when disabled |
|---|---|---|
| Unknown field present | Strictness mode is STRICT or COMPATIBLE | Ignored in LENIENT |
| Deprecated feature used | Strictness mode is STRICT or COMPATIBLE | Silently accepted in LENIENT |
| Optional syntax validation | Policy toggle is enabled | Skipped — trust runtime behavior |

**Key rule:** Structural constraint violations are always errors. Only advisory diagnostics are mode-dependent.

---

## 8. Application Codes

### Two-layer code model

| Layer | Purpose | Appears in |
|---|---|---|
| Internal application code | Logs, exceptions, metrics, audit records, tracing, support diagnostics | All internal infrastructure |
| External/API code | API responses only | Error response payloads |

Internal codes must never be replaced by HTTP status alone. External codes must never be used as internal diagnostic identifiers.

### Code families

Each meaningful failure or business condition gets a typed application code. One enum per code family. Never mix prefixes from multiple families into the same enum.

**Naming rule:** `{DOMAIN}-{LAYER}-{number}` — stable, grep-friendly, dashboard-friendly. Do not rename codes once released.

### Response-producing vs observability-only codes

Every code must be explicitly classified.

| Classification | Appears in |
|---|---|
| Response-producing | Error response `internalCode` field + logs + metrics |
| Observability-only | Logs and metrics only — never surfaced to API consumers |

**Rule:** Every enum constant's Javadoc must state its classification. See [Section 18.4](#184-enum-constant-javadoc).

### Category classification on code enums

When a single exception type must map to multiple external codes based on the nature of the failure, add a classification method (e.g., `configCategory()`) to the code enum. The handler switches on the classification — never string-prefix matching, never if-else chains. The switch must be exhaustive.

```java
// Category classification: exhaustive switch.
// Never string-prefix matching. Never if-else chains.
private HttpCode mapConfigurationCode(ValidationCode code) {
    return switch (code.configCategory()) {
        case SCHEMA     -> HttpCode.SCHEMA_CONFIGURATION_INVALID;
        case SECURITY   -> HttpCode.SECURITY_CONFIGURATION_INVALID;
        case GENERAL    -> HttpCode.GENERAL_CONFIGURATION_INVALID;
    };
}
```

### Sentinel codes

When framework exceptions bypass the domain exception hierarchy (e.g., `MethodArgumentNotValidException`, `ConstraintViolationException`), use a reserved sentinel code so that the internal code field is always populated — never pass `null` where a code is expected.

### External/API code carries embedded `HttpStatus`

The external code enum carries its HTTP status directly so the handler never maintains a separate mapping table:

```java
GATEWAY_TIMEOUT("APP-HTTP-5040", HttpStatus.GATEWAY_TIMEOUT, "Downstream gateway timed out"),
```

---

## 9. Exception Model

### Hierarchy pattern

```
ApplicationException (abstract — carries ApplicationCode)
├── InputValidationException              → 400
├── BusinessRuleViolationException        → 422
├── ResourceNotFoundException             → 404
├── ConfigurationException                → 500
├── TransportException                    → 502
│   └── TransportTimeoutException         → 504
├── AsyncTimeoutException                 → 504
├── TaskInterruptedException             → 503
├── TaskCancelledException               → 503
└── DispatcherUnavailableException       → 503
```

Every module adapts this pattern to its domain. Subtype relationships must reflect genuine is-a relationships.

### Constructor discipline — mandatory for all exception classes

The abstract base exception must define three constructors. All concrete exceptions delegate to these same three signatures.

| Constructor | Signature | Behavior |
|---|---|---|
| 1-arg | `(ApplicationCode code)` | Uses `code.defaultMessage()` as the exception message |
| 2-arg | `(ApplicationCode code, String message)` | Uses the explicit message |
| 3-arg | `(ApplicationCode code, String message, Throwable cause)` | Uses the explicit message and wraps the cause |

```java
/**
 * Abstract base for all domain exceptions in this module.
 *
 * <p>Every concrete exception carries a precise {@link ApplicationCode} —
 * never thrown without a code.
 *
 * <p>Three constructors per platform standard §13:
 * 1-arg (code, default message), 2-arg (code + custom message),
 * 3-arg (code + message + cause).
 */
public abstract class ApplicationException extends RuntimeException {

    private final ApplicationCode code;

    protected ApplicationException(ApplicationCode code) {
        super(code.defaultMessage());
        this.code = Objects.requireNonNull(code, "code must not be null");
    }

    protected ApplicationException(ApplicationCode code, String message) {
        super(message);
        this.code = Objects.requireNonNull(code, "code must not be null");
    }

    protected ApplicationException(ApplicationCode code, String message, Throwable cause) {
        super(message, cause);
        this.code = Objects.requireNonNull(code, "code must not be null");
    }

    public ApplicationCode code() { return code; }
}
```

### Exception placement rules

| Rule | Detail |
|---|---|
| All exceptions live in `exception` package | Single dedicated package, no splits |
| `exception` may import `code` | For `ApplicationCode` types |
| `code` must never import `exception` | One-way dependency |
| `exception` must never import web-layer code packages | Internal exceptions never reference HTTP-facing codes |
| Every translator exit returns typed exception | Never a raw `RuntimeException` |

### `InterruptedException` rule — mandatory, non-negotiable

```java
// CORRECT: always restore interrupt flag, always wrap in typed exception with code
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new TaskInterruptedException(AsyncCode.TASK_INTERRUPTED, "Task was interrupted", e);
}

// WRONG — never do this
catch (InterruptedException e) {
    log.warn("interrupted");   // interrupt status lost forever
}
```

### Throwing correctly

```java
// Correct — 1-arg, uses code's default message
throw new ConfigurationException(ValidationCode.MISSING_REQUIRED_CONFIG);

// Correct — 2-arg, specific context
throw new ResourceNotFoundException(ProviderCode.NOT_FOUND, "Resource not found: " + key);

// Correct — 3-arg, with cause chain
throw new TransportException(TransportCode.AUTH_FAILED, "Authentication failed for: " + target, ex);

// Wrong — no typed exception, no internal code
throw new RuntimeException("not found");
```

---

## 10. Exception Handler

### Handler rules

- Handle both `MethodArgumentNotValidException` and `ConstraintViolationException` — they cover different binding paths
- Configuration or category-based exceptions mapped via exhaustive switch — never string matching, never if-else chains
- Catch-all `Exception` handler always present as the final safety net
- Never log the same exception twice — service logs root cause, handler returns response

### Handler logging rules by HTTP status range

| HTTP status range | Log level | Stack trace | Rationale |
|---|---|---|---|
| `400` client input errors | None | No | Client errors are not server problems |
| `404` not found | `DEBUG` | No | Useful for support but not actionable for operations |
| `422` business rule violation | `WARN` | No | May indicate upstream integration issue |
| `500` configuration errors | `ERROR` | Yes — if not already logged by service | Server-side misconfig |
| `502` transport/integration errors | `ERROR` | Yes — transport module logs root cause | Infrastructure failure |
| `503` temporarily unavailable | `WARN` | No | Transient, expected to self-resolve |
| `504` timeout | `WARN` | No | Transient, logged with detail at transport layer |
| Catch-all `Exception` | `ERROR` | Yes | Unknown failure — always log |

### Error response shape

Every module with an HTTP layer must use this standard shape:

```json
{
  "success": false,
  "code": "APP-HTTP-4002",
  "httpStatus": 400,
  "message": "Human-readable error description",
  "internalCode": "APP-VAL-4000",
  "details": [
    {
      "field": "targetField",
      "rejectedValue": "invalid-value",
      "message": "Field-specific error description"
    }
  ],
  "path": "/api/resource",
  "timestamp": "2026-03-21T10:15:30Z"
}
```

| Field | Type | Requirement |
|---|---|---|
| `success` | `boolean` | Always `false` for error responses |
| `code` | `String` | External/API-facing code |
| `httpStatus` | `int` | Numeric HTTP status |
| `message` | `String` | Human-readable |
| `internalCode` | `String` | Internal application code for support — included per security exposure policy |
| `details` | `List<FieldErrorDetail>` | Typed list, never `List<?>` |
| `path` | `String` | Request path |
| `timestamp` | `String` | ISO 8601 |

---

## 11. Transport and Timeout Handling

### Mandatory timeout configuration

A missing or default timeout is a production bug. Every external transport must explicitly configure all applicable timeouts.

```yaml
# Example — adapt to your transport
sphuta.{domain}.{subdomain}.transport:
  connection-timeout: 5s
  read-timeout: 10s
  write-timeout: 10s
```

### Transport exception translation — mandatory

All low-level infrastructure exceptions must be translated to typed internal exceptions with application codes before leaving the service layer. Raw framework exceptions (`MailSendException`, `SocketTimeoutException`, `HttpClientErrorException`, etc.) must never propagate upward.

Every exit from a translator must return a typed domain exception — never a raw `RuntimeException`.

```java
// Transport exception translation rule (§17 General Standards):
// All low-level exceptions translated to typed internal exceptions.
// Every exit returns a typed exception. Never a raw RuntimeException.
catch (SocketTimeoutException e) {
    throw new TransportTimeoutException(TransportCode.TIMEOUT, "Transport timed out", e);
}
catch (AuthenticationException e) {
    throw new TransportException(TransportCode.AUTH_FAILED, "Authentication failed", e);
}
catch (SSLException e) {
    throw new TransportException(TransportCode.TLS_FAILED, "TLS handshake failed", e);
}
catch (ConnectException e) {
    throw new TransportException(TransportCode.CONNECTION_REFUSED, "Connection refused", e);
}
catch (Exception e) {
    throw new TransportException(TransportCode.GENERIC_FAILURE, "Transport failed", e);
}
```

### Application-level deadline enforcement

Framework-level timeouts (socket, connection) and application-level deadlines (future timeouts) are both required. Missing either layer leaves gaps.

```java
// Both transport-level and application-level timeouts required.
// Per-operation deadline
CompletableFuture.supplyAsync(() -> executeOperation(...), executor)
    .orTimeout(properties.operationTimeout().toMillis(), TimeUnit.MILLISECONDS);

// Batch total deadline
CompletableFuture.allOf(futuresArray)
    .thenApply(ignored -> collectResults(futures))
    .orTimeout(properties.batchTimeout().toMillis(), TimeUnit.MILLISECONDS);
```

---

## 12. Retry Policy

### Retry SPI pattern

Every module with retriable operations should define a retry policy SPI:

```java
public interface RetryPolicy {
    boolean isRetriable(ApplicationException ex);
    int maxAttempts();
    Duration delay();
}
```

### Retriable vs non-retriable classification

| Failure type | Retriable | Reason |
|---|---|---|
| Timeout | Yes | Transient network issue |
| Connection refused | Yes | Transient network issue |
| Stale connection | Yes | Transient network issue |
| Generic transport failure | Configurable | Default false — avoids retrying permanent rejections |
| Authentication failure | Never | Permanent until config changes |
| TLS failure | Never | Permanent until config changes |
| Configuration problem | Never | Permanent until config changes |
| Business-rule violation | Never | Not a transport failure |
| Validation failure | Never | Not a transport failure |

**Rule:** Document which failure codes are retriable and which are not. Generic or ambiguous failures should be non-retriable by default with an explicit opt-in toggle.

### Default retry properties

| Property | Recommended default |
|---|---|
| `max-attempts` | 3 |
| `delay` | 1s — fixed, no backoff, no jitter |
| `retry-on-generic-failure` | `false` |

---

## 13. Resilience

### Capability stack

Each resilience capability has a clearly separated responsibility:

| Capability | Mechanism | Responsibility |
|---|---|---|
| Retry | `RetryPolicy` SPI | Retry transient failures only |
| Throttling/bulkhead | `Semaphore(maxConcurrent)` | Prevent unbounded resource bursts |
| Async timeout | `CompletableFuture.orTimeout(...)` | Enforce application-level deadlines |
| Circuit breaker | Resilience4j or equivalent (optional) | Short-circuit on repeated downstream failures |

### Fallback rules

- Resource not found in primary source → fall back to secondary source
- Primary source failure/timeout/unavailability → propagate immediately — **never silently fall back**

```java
// Resilience rule (§4 General Standards): do not hide infrastructure failures
// behind silent fallback. Infrastructure failures must surface.
```

### Circuit breaker — optional

Circuit breaking is always an optional module. Default starters do not include it. Consumers who need it add the dependency explicitly.

---

## 14. Async and Threading

### `ExecutionContextPropagator` SPI

Context propagation across async boundaries uses a shared SPI so tenant, tracing, MDC, and other modules can propagate without coupling.

```java
public interface ExecutionContextPropagator {
    Runnable wrap(Runnable task);
    <T> Callable<T> wrap(Callable<T> task);
}
```

### Correct propagator pattern

```java
@Override
public Runnable wrap(Runnable task) {
    SomeContext captured = SomeContext.current();   // capture on calling thread
    return () -> {
        SomeContext.set(captured);                   // restore on worker thread
        try { task.run(); }
        finally { SomeContext.clear(); }             // always clean up
    };
}
```

### Propagator independence rule

Each propagator captures and restores only its own context. No propagator assumes anything about what another captured. Order of wrapping does not matter for correctness.

### Throttling rule

Concurrency limits must be explicit. Semaphore acquired before task submission. Prevents unbounded resource bursts.

```java
// Throttling rule (§18): concurrency limits must be explicit.
semaphore.acquire();
try {
    executor.submit(wrappedTask);
} catch (RejectedExecutionException e) {
    semaphore.release();
    throw e;
}
```

### Rejected execution translation — mandatory

```java
// Async rule (§19): rejected execution must be translated into a typed
// exception with a code, not swallowed or re-wrapped generically.
catch (RejectedExecutionException e) {
    throw new DispatcherUnavailableException(
        AsyncCode.EXECUTOR_REJECTED, "Executor rejected task", e);
}
```

### `InterruptedException` rule

Always restore the interrupt flag. Never swallow. See [Section 9](#9-exception-model).

---

## 15. Multi-Tenancy

### Design principle

Tenancy is a **wrapper** around existing SPIs — core services are never rewritten for tenant logic. Isolation is applied at lookup, resolution, validation, and dispatch boundaries.

### Isolation boundaries

Every module that supports multi-tenancy must enforce composite-key lookups:

```java
// Multi-tenancy rule (§23): isolation at lookup boundaries.
// Composite keys, never global lookups.
findByTenantIdAndResourceKey(tenantId, resourceKey)
// Never: findByResourceKey(resourceKey) — ignores tenant isolation
```

### Startup validation in multi-tenant mode

Validation runs for **all** loaded tenants. Fails globally if any tenant is misconfigured. Error messages include `tenantId` for every failure.

```
tenant=acme, resource=config-A, code=APP-VAL-4001, message=Configuration X is missing
tenant=globex, resource=config-B, code=APP-VAL-4004, message=Reference Y could not be resolved
```

### Tenant resolution strategies

| Strategy | Use case |
|---|---|
| `HEADER` | API-driven — custom header |
| `JWT_CLAIM` | Spring Security — JWT token claim |
| `THREAD_LOCAL` | Internal async — programmatically set |

### Async propagation

A `TenantContextPropagator` implements `ExecutionContextPropagator` and is automatically applied by the async module. No direct `tenant → async` module dependency is needed.

### DB schema rule

All tenant-aware tables include `tenant_id`. Lookups are always composite.

---

## 16. Logging Standards

### Logger setup

Every concrete class gets its own SLF4J logger. Never share loggers across classes.

```java
private static final Logger log = LoggerFactory.getLogger(EnclosingClass.class);
```

### Log level rules

| Level | When to use |
|---|---|
| `DEBUG` | Detailed diagnostics — resolution details, provider chosen, fallback path, retry attempt detail, normalized inputs |
| `INFO` | One line at major operation start, one at successful completion |
| `WARN` | Recoverable anomaly — policy-driven drops, deduplication collapse, degraded behavior, missing tracking context |
| `ERROR` | Actual failure with stack trace — at the layer that owns the failure only |

### Duplicate logging rule

```
ServiceLayer                → ERROR with full stack trace (owns the failure)
DispatcherLayer             → WARN with message only (service already logged it)
ExceptionHandler            → returns response, does not re-log at ERROR
```

Do not log the same exception stack trace twice. The layer that owns the failure logs at `ERROR`. Upstream layers log at `WARN` with message only.

### PII handling

PII must be masked in `WARN` and `ERROR` logs. Full values at `DEBUG` only in non-production or tightly controlled support scenarios.

```java
// WARN / ERROR — always mask
log.warn("[{}] Dropping invalid input: {}", domainPrefix, mask(sensitiveValue));

private static String mask(String value) {
    if (value == null || value.length() <= 2) return "***";
    return value.charAt(0) + "***" + value.substring(value.length() - 1);
}
```

### Structured log fields

Every module must define its structured log fields:

| Field | When to include |
|---|---|
| `{resourceKey}` | All operations on the primary resource |
| `correlationId` | All single operations |
| `batchId` | All batch operations |
| `internalCode` | All failure log lines |
| `attempt` | During retry attempts |
| `elapsed` | On successful completion |
| `tenantId` | When tenant module is active |
| `traceId` | When observability module is active |

### Domain log prefix

All log lines use a domain prefix for easy filtering:

```
INFO  ServiceClass - [{domain}] Starting operation 'X' [correlationId=...]
INFO  ServiceClass - [{domain}] Operation 'X' completed [correlationId=..., elapsed=142ms]
ERROR ServiceClass - [{domain}] Transport failure [APP-TRN-5045] [correlationId=...]
WARN  DispatcherClass - [{domain}] Batch complete — 2/3 succeeded [batchId=...]
```

---

## 17. Tracking IDs and Correlation

### Ownership model

| Responsibility | Owner |
|---|---|
| Create `correlationId` | Web layer (controller or API filter) at request entry point |
| Create `batchId` | Batch dispatcher at batch creation (boundary exception) |
| Read tracking IDs | Every downstream module via `MDC.get()` |
| Generate tracking IDs | Nobody downstream — never `UUID.randomUUID()` in service, provider, transport, or validator code |
| Handle missing context | `"untraced"` + `log.warn()` — surfaces the gap, does not hide it |
| Cross-thread propagation | `ExecutionContextPropagator` beans |

### Missing context handling

```java
// Tracking ID rule (§20): downstream modules read from MDC, never generate.
// Missing context → "untraced" + WARN log. Never a fallback UUID.
var rawId = MDC.get("correlationId");
if (rawId == null || rawId.isBlank()) {
    log.warn("[{}] Missing correlationId in MDC — using untraced", domainPrefix);
}
final var correlationId = (rawId != null && !rawId.isBlank()) ? rawId : "untraced";
```

### Lambda capture pattern

When tracking IDs read from MDC are captured in lambdas, use the effectively-final pattern:

```java
// Effectively final — safe for lambda capture
final var resourceKey = command.resourceKey();
final var correlationId = resolveCorrelationId();

CompletableFuture.supplyAsync(() -> {
    log.debug("[{}] Processing '{}' [correlationId={}]", domainPrefix, resourceKey, correlationId);
    return executeOperation(command);
}, executor);
```

### Module audit rule

CI should include a check for tracking ID generation violations — no `UUID.randomUUID()`, no `MDC.put("correlationId", ...)`, and no tracking ID creation in downstream modules.

---

## 18. Code Comments and Javadoc

### 18.1 Governing principles

- Every public type gets a Javadoc — classes, interfaces, enums, and records.
- Comment the decision, not the syntax.
- Business rules must be commented near the enforcement point.
- When code changes, the comment must change in the same commit.
- Wrong comments are worse than no comments.
- Section banners only in genuinely long classes. Remove decorative banners where package structure and method visibility already communicate.

---

### 18.2 Record Javadoc template

```java
/**
 * [One sentence: what this record represents and its role in the system.]
 *
 * <p>All collection fields use {@code List.copyOf()} or {@code Map.copyOf()}.
 * This record is fully immutable after construction.
 *
 * <h3>Construction rules:</h3>
 * <ul>
 *   <li>{@code fieldName} — required, non-null. [Purpose.]</li>
 *   <li>{@code optionalField} — nullable. [When absent and what that means.]</li>
 *   <li>{@code items} — required, non-null. Defensively copied. Empty list when no items.</li>
 * </ul>
 *
 * <h3>Business rules enforced at construction:</h3>
 * <ul>
 *   <li>[List every invariant the compact constructor enforces.]</li>
 *   <li>[State what null collections normalize to.]</li>
 * </ul>
 *
 * @param fieldName     [description], never null
 * @param optionalField [description], nullable — null means [meaning]
 * @param items         [description], never null, immutable after construction
 * @see CompanionBuilder
 * @see PrimaryConsumer
 */
```

**Mandatory:** Immutability contract, per-field `@param`, construction rules, business rules at construction, `@see` to builder and consumer.

---

### 18.3 Enum type Javadoc template

```java
/**
 * [What this enum represents and where it is used.]
 *
 * <p>[Value-label enum (logging/diagnostics only) or behavior-carrying enum.]
 *
 * <h3>Design rules:</h3>
 * <ul>
 *   <li>Exhaustive switch required — never if-else chains, never string-prefix matching.</li>
 *   <li>Used in [list consumers].</li>
 *   <li>Codes must not be renamed once released.</li>
 *   <li>All constants prefixed with {@code {DOMAIN}-{LAYER}-}.</li>
 * </ul>
 *
 * @see ConsumingException
 * @see RetryPolicy
 */
```

---

### 18.4 Enum constant Javadoc

**Every enum constant gets a Javadoc. No exceptions.**

```java
/**
 * [When this constant is triggered / what condition causes it.]
 *
 * <p>[Context: severity, whether transient or permanent.]
 *
 * <p>Retriable: [yes/no/not applicable].
 * Response-producing: [yes/no — observability-only].
 */
SOME_FAILURE_CODE("APP-LAYER-5040", "Default message"),
```

---

### 18.5 SPI interface Javadoc

```java
/**
 * [What this SPI extension point does.]
 *
 * <p>Contract type: {@code spi} — framework extension point.
 * Application code must never import from {@code spi} directly.
 *
 * <h3>Implementation rules:</h3>
 * <ul>
 *   <li>Thread safety: [thread-safe / not thread-safe / caller must synchronize].</li>
 *   <li>Registration: Spring bean. Default backs off via
 *       {@code @ConditionalOnMissingBean} when consumer provides their own.</li>
 *   <li>Exactly one effective bean at injection time.</li>
 * </ul>
 *
 * <h3>Behavioral contract:</h3>
 * <ul>
 *   <li>[Every invariant implementors must honor.]</li>
 *   <li>[What must never throw.]</li>
 *   <li>[What must always return true/false for specific failure categories.]</li>
 * </ul>
 *
 * @see DefaultImplementation
 */
```

---

### 18.6 API interface Javadoc

```java
/**
 * [What this API contract does — primary entry point for consumers.]
 *
 * <p>Contract type: {@code api} — application code imports and calls this directly.
 *
 * <h3>Behavioral contract:</h3>
 * <ul>
 *   <li>[Resolution/orchestration steps.]</li>
 *   <li>[Invariants enforced before dispatch.]</li>
 *   <li>[What typed exceptions are thrown — never raw exceptions.]</li>
 * </ul>
 *
 * @see InputCommand
 * @see ApplicationException
 */
```

---

### 18.7 Sealed interface Javadoc template

```java
/**
 * [Role of this sealed interface.]
 *
 * <p>Permitted subtypes:
 * <ul>
 *   <li>{@link SubtypeA} — [what it represents]</li>
 *   <li>{@link SubtypeB} — [what it represents]</li>
 * </ul>
 *
 * <p>Exhaustive pattern matching switch is required.
 * No {@code default} branch needed when all subtypes are covered.
 *
 * @see SubtypeA
 * @see SubtypeB
 */
```

---

### 18.8 Method Javadoc rules

**Public API methods:** Always `@param`, `@return`, `@throws`.

```java
/**
 * [What this method does.]
 *
 * @param command  the input command, never null
 * @return the result, never null
 * @throws ResourceNotFoundException      if the resource key cannot be resolved
 * @throws BusinessRuleViolationException if final invariant check fails
 * @throws TransportException             if downstream transport fails after retry exhaustion
 * @throws ConfigurationException         if resolved configuration is invalid
 * @throws NullPointerException           if command is null
 */
Result execute(Command command);
```

**Internal methods:** One-liner only when the name does not explain the *why*.

**Builder methods and record accessors:** Exempt. Exception: builder methods enforcing validation must document the rule.

```java
/**
 * Sets the resource key. Required — must not be null or blank.
 *
 * @throws NullPointerException     if key is null
 * @throws IllegalArgumentException if key is blank
 */
public CommandBuilder resourceKey(String key) { ... }
```

---

### 18.9 Constants and crucial variables

```java
/** Domain log prefix for all log lines in this module. Used for log aggregation filtering. */
private static final String LOG_PREFIX = "[domain]";

/**
 * Sentinel value for missing tracking context.
 * Must trigger a WARN log when encountered — never silently accepted.
 * Never a fallback UUID.
 */
private static final String UNTRACED = "untraced";

/**
 * Compiled validation pattern. Thread-safe for concurrent use.
 * Compiled once at class load.
 */
private static final Pattern IDENTIFIER_PATTERN = Pattern.compile("^[a-zA-Z0-9-]{3,50}$");
```

---

### 18.10 Package-info Javadoc — required on every package

Every `package-info.java` carries `@NullMarked` and a two-sentence Javadoc: (1) what the package contains, (2) what it does not contain or where adjacent concerns live.

```java
/**
 * Application code enums for this module.
 *
 * <p>Contains all internal code families and the external/API code enum.
 * Exception classes that consume these codes live in the {@code exception} package.
 */
@NullMarked
package net.sphuta.domain.subdomain.code;
```

```java
/**
 * Exception hierarchy for this module.
 *
 * <p>All exception types live here — no exception classes exist outside this package.
 * Exceptions consume application codes from the {@code code} package.
 */
@NullMarked
package net.sphuta.domain.subdomain.exception;
```

```java
/**
 * SPI extension points for this module.
 *
 * <p>Application code must never import from this package. Implementation modules
 * register beans that satisfy these contracts.
 */
@NullMarked
package net.sphuta.domain.subdomain.spi;
```

```java
/**
 * Public API contracts for this module.
 *
 * <p>Contains interfaces and types that application code imports directly.
 * Internal implementation details live in module-specific packages.
 */
@NullMarked
package net.sphuta.domain.subdomain.api;
```

---

## 19. DB and Persistence

### Read-only boundary

Modules that consume data from a database but do not own the write path must enforce a read-only boundary. Write operations belong in a separate module.

### Repository rule

Read-only modules expose only `findBy*` methods. No `save`, `delete`, `update`, or mutation methods.

```java
// Read-only boundary rule (§22): no mutation methods in this module.
Optional<Entity> findByKey(String key);
Optional<Entity> findByTenantIdAndKey(String tenantId, String key);

// WRONG — no mutations in a read-only module
// Entity save(Entity entity);
```

### Client-facing error messages — never expose raw infrastructure details

Persistence errors must not leak raw SQL, JPA, or infrastructure details to callers.

| Scenario | Client message |
|---|---|
| Resource not found | "Resource not found" |
| DB unavailable | "Provider is temporarily unavailable" |
| DB read timeout | "Lookup timed out" |
| Mapper / data inconsistency | "Configuration could not be resolved" |

---

## 20. Business Rules, Validation, and Conditions in Code

### 20.1 Governing principle

Every `if`, guard clause, switch branch, or policy check that enforces a business rule, validation condition, or architectural constraint must have an inline comment citing the specific rule.

This is not optional. Every enforcement point gets a comment.

---

### 20.2 Comment patterns for common enforcement categories

#### Required field enforcement

```java
// Business rule: required fields always produce errors regardless of configuration mode.
// Cannot be suppressed by lenient or compatible settings.
if (value == null || value.isBlank()) {
    errors.rejectValue(fieldName, "MISSING_REQUIRED_FIELD",
        "Field '" + fieldName + "' is required");
}
```

#### Forbidden field enforcement

```java
// Business rule: forbidden fields produce errors and processing fails
// regardless of configuration mode. Cannot be suppressed.
if (forbiddenNames.contains(fieldName)) {
    errors.rejectValue(fieldName, "FORBIDDEN_FIELD",
        "Field '" + fieldName + "' is forbidden");
}
```

#### Mode-dependent behavior

```java
// Mode-dependent behavior:
// STRICT — unknown fields are errors.
// COMPATIBLE — unknown fields are warnings.
// LENIENT — unknown fields are silently ignored.
switch (context.mode()) {
    case STRICT -> errors.rejectValue(field, "UNKNOWN_FIELD", message);
    case COMPATIBLE -> context.addWarning(field, "UNKNOWN_FIELD", message);
    case LENIENT -> { /* intentionally ignored per lenient policy */ }
}
```

#### Policy-driven enforcement

```java
// Policy-driven: recipient validation failures on primary recipients abort the operation.
// Failures on secondary recipients are dropped by policy — non-fatal.
if (isPrimary && action == Action.REJECT) {
    throw new InputValidationException(Code.INVALID_PRIMARY, "Invalid primary input");
}
// Secondary: drop silently, log at WARN
log.warn("[{}] Dropping invalid secondary input: {}", domainPrefix, mask(value));
```

#### Deduplication and precedence rules

```java
// Business rule: dedupe precedence is primary > secondary > tertiary.
// A value in the primary set is never also in the secondary or tertiary set.
if (primarySet.contains(normalizedValue)) {
    log.debug("[{}] Value already in primary set, skipping from secondary", domainPrefix);
    continue;
}
```

#### Privacy and isolation rules

```java
// Business rule: one operation per recipient — batch operations never share
// recipient lists for privacy. Each recipient gets their own isolated operation.
for (String recipient : normalizedRecipients) {
    futures.add(submitSingle(context, recipient));
}
```

#### Startup validation — collect-all-before-fail

```java
// Business rule: startup validation is fail-fast with full collection.
// All errors collected before throwing — operator sees the complete picture.
var collector = new ArrayList<ValidationError>();
for (Config config : allConfigs) {
    collector.addAll(validate(config));
}
if (!collector.isEmpty()) {
    throw new ConfigurationException(Code.STARTUP_FAILED,
        "Startup validation failed with " + collector.size() + " error(s)");
}
```

#### Multi-tenant startup validation

```java
// Business rule: in multi-tenant mode, validation runs for ALL loaded tenants
// and fails globally if any tenant is misconfigured.
for (String tenantId : allTenantIds) {
    collector.addAll(validateTenant(tenantId));
}
```

#### Toggle-dependent validation

```java
// Toggle-dependent: this check runs only when the policy toggle is enabled.
// When disabled, skip the check entirely — trust runtime behavior.
if (policyToggles.isValidateResourceExists()) {
    probe.verifyExists(config.resourceName());
}
```

#### Provider fallback rules

```java
// Provider fallback rule (locked): primary not found → fall back to secondary.
// Primary failure/timeout/unavailability → propagate immediately.
// Never silently fall back on infrastructure failure.
try {
    var result = primaryProvider.resolve(key);
    if (result.isPresent()) return result;
    return secondaryProvider.resolve(key);
} catch (PersistenceException e) {
    // Infrastructure failure — propagate, do not fall back
    throw e;
}
```

#### Transport exception translation

```java
// Transport rule: all low-level exceptions translated to typed internal
// exceptions before leaving the service layer. Every exit returns a typed
// exception — never a raw RuntimeException.
catch (SocketTimeoutException e) {
    throw new TransportTimeoutException(TransportCode.TIMEOUT, "Timed out", e);
}
```

#### Async and threading rules

```java
// Async rule (§19): rejected execution translated to typed exception with code.
catch (RejectedExecutionException e) {
    throw new DispatcherUnavailableException(AsyncCode.REJECTED, "Executor rejected task", e);
}
```

```java
// InterruptedException rule (§13): never swallow.
// Always restore interrupt flag. Always wrap in typed exception with code.
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new TaskInterruptedException(AsyncCode.INTERRUPTED, "Task interrupted", e);
}
```

#### Dependency direction enforcement

```java
// Dependency direction rule (§4): upstream module must not depend on
// downstream module. This interface is generic specifically to avoid
// importing downstream types.
```

#### Security enforcement

```java
// XXE protection (mandatory for all XML parsing):
// Both settings mandatory — omitting either is a security vulnerability.
factory.setProperty(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false);
factory.setProperty(XMLInputFactory.SUPPORT_DTD, false);
```

```java
// YAML safe construction (mandatory for all YAML parsing):
// Prevents deserialization attacks.
new Yaml(new SafeConstructor(new LoaderOptions()));
```

#### Immutability enforcement

```java
// Immutability rule (§25): all collection fields defensively copied at construction.
this.items = List.copyOf(items);
```

```java
// Null collection normalization (§25): null → empty, never remain null.
this.items = items != null ? List.copyOf(items) : List.of();
```

#### Tracking ID enforcement

```java
// Tracking ID rule (§20): downstream modules never generate their own IDs.
// Missing context → "untraced" + WARN log — never a fallback UUID.
var rawId = MDC.get("correlationId");
if (rawId == null || rawId.isBlank()) {
    log.warn("[{}] Missing correlationId in MDC — using untraced", domainPrefix);
}
final var correlationId = (rawId != null && !rawId.isBlank()) ? rawId : "untraced";
```

#### Retry classification

```java
// Retry classification (§18): authentication, TLS, configuration, and
// business-rule failures are never retriable. Fail fast.
if (code.isAuthenticationFailure() || code.isConfigurationError()) {
    throw new NonRetriableException(code, message);
}
```

#### Bean registration discipline

```java
// Pluggability rule (§5): default backs off when consumer provides their own bean.
// Exactly one effective bean per primary contract at injection time.
@Bean
@ConditionalOnMissingBean(TargetContract.class)
public TargetContract defaultImplementation() { ... }
```

#### Rejected execution handling

```java
// Async rule (§19): rejected execution translated to typed exception.
catch (RejectedExecutionException e) {
    throw new DispatcherUnavailableException(AsyncCode.REJECTED, "Executor rejected", e);
}
```

---

### 20.3 When to comment vs when not to comment

| Situation | Comment? | Reason |
|---|---|---|
| Guard clause enforcing a business rule | Yes — always | The rule may not be obvious from code alone |
| `Objects.requireNonNull` in compact constructor | Yes — briefly | State which invariant: "required collaborator" or "mandatory field" |
| Switch on enum with different business meaning per branch | Yes — per branch | Each branch enforces a different policy |
| Catch block translating infrastructure exception | Yes | State what translation accomplishes and what code is assigned |
| `List.copyOf()` in record constructor | Yes — briefly | "Immutability rule (§25)" is sufficient |
| Obvious null check on method parameter | No | Self-evident from `@NullMarked` and parameter name |
| Private helper doing string formatting | No | No business rule involved |
| `@ConditionalOnMissingBean` on a bean method | Yes | State which SPI it backs off for |
| Toggle-dependent validation branch | Yes | State what toggle controls it and what happens when disabled |

---

## 21. Tooling Enforcement Summary

| Standard | Enforcement mechanism | Gate |
|---|---|---|
| Compiler warnings as errors | `javac -Xlint:all -Werror` | CI build failure |
| Import ordering | Spotless + google-java-format or Checkstyle `CustomImportOrder` | CI build failure |
| Unused imports | Spotless auto-removal or `javac -Xlint:unused -Werror` | CI build failure |
| Wildcard import ban | Checkstyle `AvoidStarImport` with severity = error | CI build failure |
| One public type per file | Checkstyle `OneTopLevelClass` | CI build failure |
| No exceptions outside `exception` package | ArchUnit test | CI build failure |
| No inner builder classes | ArchUnit test | CI build failure |
| Generated/handwritten source separation | Build script validation | CI build failure |
| Javadoc on all public types | Checkstyle `MissingJavadocType` | CI build failure |
| Javadoc on all public methods (API/SPI) | Checkstyle `MissingJavadocMethod` | CI warning → error after stabilization |
| `InterruptedException` not swallowed | ErrorProne `InterruptedExceptionSwallowed` or grep-based CI | CI build failure |
| Dependency direction | ArchUnit test per module | CI build failure |
| Tracking ID generation violations | Grep-based CI for `UUID.randomUUID()` in downstream modules | CI build failure |

---

*All decisions in this guide are locked.*
*Changes require explicit review and update to this document before implementation.*
*Code and documentation must move together — update both in the same commit.*
