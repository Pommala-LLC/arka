# SPHUTA Arka Email — OSS Tier Complete Reference

> **Scope:** All application codes, exceptions, business rules, and validations for the OSS tier
> **Modules covered:** 8 capability modules + 1 composition starter = 9 Maven artifacts
> **Status:** Aligned with canonical tier freeze, coding standards implementation guide, and tier isolation rule
>
> **Tier isolation rule (locked):** OSS core owns only OSS contracts, codes, and exceptions.
> PaaS modules own PaaS contracts. SaaS modules own SaaS contracts. No cross-tier contract leakage.

---

## Contents

1. [OSS Tier Module Inventory](#1-oss-tier-module-inventory)
2. [Application Codes — Complete Registry](#2-application-codes--complete-registry)
3. [Exception Hierarchy](#3-exception-hierarchy)
4. [Exception-to-HTTP Status Mapping](#4-exception-to-http-status-mapping)
5. [Per-Module Reference](#5-per-module-reference)
6. [Validation Layer Matrix](#6-validation-layer-matrix)
7. [Business Rules — Complete Registry](#7-business-rules--complete-registry)
8. [Retriability Classification](#8-retriability-classification)
9. [What Core Does NOT Contain](#9-what-core-does-not-contain)

---

## 1. OSS Tier Module Inventory

OSS is the pure synchronous core. No async, no retry, no DB/hybrid providers, no batch dispatch.

### Shared capabilities (4 modules)

| Module | Artifact ID | Role |
|---|---|---|
| Core | `sphuta-arka-email-core` | Contracts, SPIs, models, codes, exceptions — OSS scope only |
| Service | `sphuta-arka-email-service` | Default send orchestration (single-send, synchronous) |
| Validation | `sphuta-arka-email-validation` | Startup flow configuration validation |
| Provider YAML | `sphuta-arka-email-provider-yaml` | YAML-backed flow resolution |

### Email edge capabilities (4 modules)

| Module | Artifact ID | Role |
|---|---|---|
| Transport JavaMail | `sphuta-arka-email-transport-javamail` | SMTP transport with typed exception translation |
| Recipient | `sphuta-arka-email-recipient` | Recipient normalization, validation, deduplication |
| Message | `sphuta-arka-email-message` | MIME message construction and final message assembly |
| Template Thymeleaf | `sphuta-arka-email-template-thymeleaf` | Thymeleaf-based template rendering |

### Composition (1 module)

| Module | Artifact ID | Role |
|---|---|---|
| Starter | `sphuta-arka-email-starter` | Composition POM — bundles all 8 OSS modules. Owns no code. |

---

## 2. Application Codes — Complete Registry

All OSS code enums live in `sphuta-arka-email-core` in the `net.sphuta.arka.email.code` package. Every code implements `ApplicationCode`.

Core contains **only** codes for OSS-active capabilities. PaaS modules own their own code families.

---

### `EmailCoreCode` — `EMAIL-CORE-*`

Service lifecycle events. Used by `DefaultEmailFlowService`.

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `EMAIL-CORE-1000` | `SEND_STARTED` | Send workflow started | Observability-only |
| `EMAIL-CORE-1001` | `SEND_SUCCEEDED` | Send workflow completed | Observability-only |
| `EMAIL-CORE-1002` | `FLOW_RESOLUTION_USED` | Flow resolution used | Observability-only |
| `EMAIL-CORE-5000` | `SEND_FAILED` | Send workflow failed | Response-producing |
| `EMAIL-CORE-5001` | `UNEXPECTED_ERROR` | Unexpected error | Response-producing |

---

### `EmailValidationCode` — `EMAIL-VAL-*`

Startup validation codes. Used by `DefaultEmailFlowValidator`. Each carries a `ConfigCategory`.

| Code | Constant | Default message | Category |
|---|---|---|---|
| `EMAIL-VAL-4000` | `VALIDATION_FAILED` | Startup validation failed | `FLOW` |
| `EMAIL-VAL-4001` | `TEMPLATE_MISSING` | Flow template is missing | `FLOW` |
| `EMAIL-VAL-4002` | `SUBJECT_MISSING` | Flow subject is missing | `FLOW` |
| `EMAIL-VAL-4003` | `SENDER_REF_MISSING` | Sender reference is missing | `SENDER` |
| `EMAIL-VAL-4004` | `SENDER_REF_UNRESOLVABLE` | Sender reference could not be resolved | `SENDER` |
| `EMAIL-VAL-4005` | `SENDER_FROM_INVALID` | Sender from address is invalid | `SENDER` |
| `EMAIL-VAL-4006` | `REPLY_TO_INVALID` | Reply-to address is invalid | `SENDER` |
| `EMAIL-VAL-4007` | `ATTACHMENT_REF_UNRESOLVABLE` | Attachment reference could not be resolved | `ATTACHMENT` |
| `EMAIL-VAL-4008` | `TEMPLATE_NOT_RESOLVABLE` | Template could not be found | `FLOW` |

---

### `EmailRecipientCode` — `EMAIL-RCP-*`

Recipient normalization codes. Used by `DefaultRecipientNormalizer`.

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `EMAIL-RCP-4000` | `INVALID_ADDRESS_FORMAT` | Recipient address format is invalid | Response-producing |
| `EMAIL-RCP-4001` | `NO_VALID_PRIMARY_RECIPIENT` | No valid primary recipient after policy resolution | Response-producing |
| `EMAIL-RCP-4002` | `DUPLICATE_COLLAPSED` | Duplicate recipient collapsed | Observability-only |

---

### `EmailProviderCode` — `EMAIL-PROV-*`

Provider resolution codes. Used by YAML provider. Sender/attachment resolution failures carry `ConfigCategory`.

| Code | Constant | Default message | Category |
|---|---|---|---|
| `EMAIL-PROV-4002` | `FLOW_NOT_FOUND_ANYWHERE` | Flow not found in any provider | — |
| `EMAIL-PROV-4003` | `SENDER_REF_MISSING` | Sender reference missing in flow | `SENDER` |
| `EMAIL-PROV-4004` | `SENDER_REF_UNRESOLVABLE` | Sender reference could not be resolved | `SENDER` |
| `EMAIL-PROV-4005` | `ATTACHMENT_REF_UNRESOLVABLE` | Attachment reference could not be resolved | `ATTACHMENT` |

Note: `HYBRID_FALLBACK` (`EMAIL-PROV-5001`) lives in `sphuta-arka-email-provider-hybrid` (PaaS), not in core.

---

### `EmailTransportCode` — `EMAIL-TRN-*`

SMTP transport codes. Used by `JavaMailTransport` and `MailExceptionTranslator`.

| Code | Constant | Default message | Retriable | HTTP status |
|---|---|---|---|---|
| `EMAIL-TRN-5020` | `SMTP_SEND_FAILED` | SMTP send failed | Conditional | 502 |
| `EMAIL-TRN-5031` | `SMTP_SUBSYSTEM_ERROR` | Mail subsystem unavailable | No | **503** |
| `EMAIL-TRN-5040` | `SMTP_TIMEOUT` | SMTP timeout | Yes | 504 |
| `EMAIL-TRN-5043` | `SMTP_AUTH_FAILED` | SMTP auth failed | No | 502 |
| `EMAIL-TRN-5044` | `SMTP_TLS_FAILED` | SMTP TLS failed | No | 502 |
| `EMAIL-TRN-5045` | `SMTP_CONNECTION_REFUSED` | SMTP connection refused | Yes | 502 |
| `EMAIL-TRN-5046` | `SMTP_SESSION_INIT_FAILED` | SMTP session init failed | No | 502 |
| `EMAIL-TRN-5047` | `SMTP_STALE_CONNECTION` | Stale or broken connection | Yes | 502 |

Note on `EMAIL-TRN-5040`: Single code for connect, read, and write timeouts. JavaMail does not reliably distinguish them at the exception chain level.

Note on `EMAIL-TRN-5031`: Maps to **503** (temporarily unavailable), not 502. The subsystem is expected to recover — this is not a gateway error, it is a service readiness problem.

---

### `EmailTemplateCode` — `EMAIL-TPL-*`

Template rendering codes. Used by Thymeleaf renderer. All carry `ConfigCategory.TEMPLATE`.

| Code | Constant | Default message | Category |
|---|---|---|---|
| `EMAIL-TPL-5000` | `TEMPLATE_RENDER_FAILED` | Template rendering failed | `TEMPLATE` |
| `EMAIL-TPL-5001` | `TEMPLATE_NOT_RESOLVABLE` | Template not resolvable | `TEMPLATE` |
| `EMAIL-TPL-5002` | `TEMPLATE_VARIABLE_ERROR` | Template variable error | `TEMPLATE` |

---

### `EmailMessageCode` — `EMAIL-MSG-*`

Message composition codes. Used by `DefaultEmailMessageBuilder`. All carry `ConfigCategory.MESSAGE`.

| Code | Constant | Default message | Category |
|---|---|---|---|
| `EMAIL-MSG-5000` | `MESSAGE_BUILD_FAILED` | Message build failed | `MESSAGE` |
| `EMAIL-MSG-5001` | `ATTACHMENT_LOAD_FAILED` | Attachment load failed | `MESSAGE` |
| `EMAIL-MSG-5002` | `MIME_CONSTRUCTION_FAILED` | MIME construction failed | `MESSAGE` |
| `EMAIL-MSG-5003` | `SENDER_FORMAT_INVALID` | Sender format invalid | `MESSAGE` |

---

### `ConfigCategory` enum

Maps validation and configuration codes to HTTP error response categories. Used by the exception handler's exhaustive switch.

| Constant | Scope |
|---|---|
| `FLOW` | Flow definition: template, subject, flow key |
| `SENDER` | Sender identity: from, reply-to, sender-ref |
| `ATTACHMENT` | Attachment references |
| `TEMPLATE` | Template rendering |
| `MESSAGE` | Message construction |

---

### Code count summary — core only

| Family | Prefix | Codes |
|---|---|---|
| Core | `EMAIL-CORE-*` | 5 |
| Validation | `EMAIL-VAL-*` | 9 |
| Recipient | `EMAIL-RCP-*` | 3 |
| Provider | `EMAIL-PROV-*` | 4 |
| Transport | `EMAIL-TRN-*` | 8 |
| Template | `EMAIL-TPL-*` | 3 |
| Message | `EMAIL-MSG-*` | 4 |
| **Total in core** | | **36** |

---

## 3. Exception Hierarchy

All OSS exceptions live in `net.sphuta.arka.email.exception` within core. Every exception carries an `ApplicationCode` via `internalCode()`. Three constructors per platform standard.

Core contains **only** exceptions thrown by OSS modules.

```
EmailApplicationException (abstract — carries ApplicationCode)
│
├── InvalidEmailRequestException          → 400
├── InvalidRecipientException             → 400
├── BusinessRuleViolationException        → 422
├── UnknownEmailFlowException             → 404
├── EmailConfigurationException           → 500
├── EmailTransportException               → 502 / 503 (code-based)
│   └── EmailTransportTimeoutException    → 504
```

**7 exception types in core.** All are OSS-active.

### Per-exception detail

| Exception | HTTP | Typical code families | Thrown by |
|---|---|---|---|
| `InvalidEmailRequestException` | 400 | `EMAIL-RCP-*` | recipient |
| `InvalidRecipientException` | 400 | `EMAIL-RCP-*` | recipient |
| `BusinessRuleViolationException` | 422 | `EMAIL-RCP-4001` | service |
| `UnknownEmailFlowException` | 404 | `EMAIL-PROV-4002` | provider-yaml |
| `EmailConfigurationException` | 500 | `EMAIL-VAL-*`, `EMAIL-PROV-400x`, `EMAIL-MSG-*`, `EMAIL-TPL-*` | validation, provider-yaml, message, template-thymeleaf |
| `EmailTransportException` | 502 / 503 | `EMAIL-TRN-*` (except 5040) | transport-javamail |
| `EmailTransportTimeoutException` | 504 | `EMAIL-TRN-5040` | transport-javamail |

### PaaS exceptions — NOT in core

These exceptions are defined in their respective PaaS modules, not in core. They extend `EmailApplicationException` via normal inward dependency.

| Exception | Defined in module | HTTP |
|---|---|---|
| `EmailAsyncTimeoutException` | `sphuta-arka-email-async` | 504 |
| `EmailTaskInterruptedException` | `sphuta-arka-email-async` | 503 |
| `EmailTaskCancelledException` | `sphuta-arka-email-async` | 503 |
| `BatchDispatcherUnavailableException` | `sphuta-arka-email-async` | 503 |

---

## 4. Exception-to-HTTP Status Mapping

### Direct exception-type mapping

| Exception type | HTTP status | HTTP code |
|---|---|---|
| `InvalidEmailRequestException` | 400 | `EMAIL-HTTP-4000` |
| `InvalidRecipientException` | 400 | `EMAIL-HTTP-4001` |
| `UnknownEmailFlowException` | 404 | `EMAIL-HTTP-4002` |
| `BusinessRuleViolationException` | 422 | `EMAIL-HTTP-4003` |
| `EmailTransportTimeoutException` | 504 | `EMAIL-HTTP-5040` |
| `EmailTransportException` | **code-based** | see below |

### Transport exception — code-based HTTP mapping

`EmailTransportException` maps to different HTTP statuses based on the internal code. This resolves the 502/503 split:

```java
private HttpCode mapTransportCode(EmailTransportCode code) {
    return switch (code) {
        case SMTP_SUBSYSTEM_ERROR -> HttpCode.SERVICE_UNAVAILABLE;   // 503
        case SMTP_TIMEOUT         -> HttpCode.GATEWAY_TIMEOUT;       // 504 (via subtype)
        default                   -> HttpCode.BAD_GATEWAY;           // 502
    };
}
```

| Transport code | HTTP status | HTTP code | Rationale |
|---|---|---|---|
| `EMAIL-TRN-5031` | **503** | `EMAIL-HTTP-5031` | Subsystem temporarily unavailable — expected to self-recover |
| `EMAIL-TRN-5040` | **504** | `EMAIL-HTTP-5040` | Handled by `EmailTransportTimeoutException` subtype |
| All other `EMAIL-TRN-*` | **502** | `EMAIL-HTTP-5020` | Gateway/upstream SMTP failure |

### Configuration exception — category-based mapping

`EmailConfigurationException` maps via exhaustive switch on `configCategory()`:

| ConfigCategory | HTTP code | HTTP status |
|---|---|---|
| `FLOW` | `EMAIL-HTTP-5004` | 500 |
| `SENDER` | `EMAIL-HTTP-5002` | 500 |
| `ATTACHMENT` | `EMAIL-HTTP-5003` | 500 |
| `TEMPLATE` | `EMAIL-HTTP-5005` | 500 |
| `MESSAGE` | `EMAIL-HTTP-5005` | 500 |

### Framework exception mapping

| Framework exception | HTTP status | Sentinel code |
|---|---|---|
| `MethodArgumentNotValidException` | 400 | `EMAIL-HTTP-4000` |
| `ConstraintViolationException` | 400 | `EMAIL-HTTP-4000` |
| `Exception` (catch-all) | 500 | `EMAIL-HTTP-5099` |

### Handler logging rules by HTTP status range

| HTTP range | Log level | Stack trace |
|---|---|---|
| 400 | None | No |
| 404 | `DEBUG` | No |
| 422 | `WARN` | No |
| 500 | `ERROR` | Yes (if not already logged by service) |
| 502 | `ERROR` | Yes (transport module logs root cause) |
| 503 | `WARN` | No (transient, expected to self-resolve) |
| 504 | `WARN` | No (transient, logged at transport layer) |
| Catch-all | `ERROR` | Yes |

---

## 5. Per-Module Reference

---

### 5.1 `sphuta-arka-email-core`

**Role:** Pure contracts — APIs, SPIs, models, codes, exceptions. OSS scope only. Zero implementations.

**Rule:** No class in core may import anything outside `java.*`.

**Owns:**

| Package | Contents |
|---|---|
| `api` | `EmailFlowService`, `EmailSendCommand`, `RecipientSendResult` |
| `api.builder` | `EmailSendCommandBuilder` |
| `spi` | `EmailFlowProvider`, `MailTransport`, `EmailMessageBuilder`, `RecipientPolicyHandler`, `EmailFlowValidator`, `TemplateRenderer`, `TemplateProbe`, `EmailSendInterceptor` |
| `model.flow` | `ResolvedEmailFlow`, `ResolvedSender`, `ResolvedAttachment`, `FinalEmailMessage`, `NormalizedRecipients` |
| `model.enums` | `FlowSourceType`, `RecipientPolicyMode` |
| `code` | `ApplicationCode`, `ConfigCategory`, 7 code enums (36 codes total) |
| `exception` | `EmailApplicationException` (abstract), 7 concrete exception types |

**What is NOT in core** (tier isolation rule):

| Type | Lives in | Reason |
|---|---|---|
| `EmailBatchDispatcher` | async module | Batch is PaaS |
| `BatchSendResult` | async module | Batch is PaaS |
| `EmailRetryPolicy` | retry-fixed module | Retry is PaaS |
| `EmailCircuitBreaker` | circuit-breaker module | Circuit breaker is PaaS |
| `ExecutionContextPropagator` | async module | Context propagation is PaaS |
| `EmailThrottlePolicy` | async module | Throttling is PaaS |
| `TenantResolver` | tenant module | Tenancy is SaaS |
| `EmailAsyncCode` (6 codes) | async module | Async is PaaS |
| `EmailBatchCode` (6 codes) | async module | Batch is PaaS |
| `EmailPersistenceCode` (5 codes) | provider-db module | DB provider is PaaS |
| 4 PaaS exception types | async module | Thrown only by PaaS modules |

**Business rules enforced in core models:**

| Rule | Location | Enforcement |
|---|---|---|
| `flowKey` not null or blank | `EmailSendCommand` constructor | `Objects.requireNonNull` + blank check |
| All collection fields defensively copied | All record compact constructors | `List.copyOf()` |
| Null collections normalize to empty | Record compact constructors | `null → List.of()` |
| `ApplicationCode` never null on exceptions | `EmailApplicationException` constructor | `Objects.requireNonNull(internalCode)` |

---

### 5.2 `sphuta-arka-email-service`

**Role:** Default send orchestration. Synchronous single-send in OSS.

**Owns:** `DefaultEmailFlowService`

**Auto-configuration:** `EmailServiceAutoConfiguration`

**Activation:** `@ConditionalOnMissingBean(EmailFlowService.class)`

**Codes used:** `EMAIL-CORE-1000`, `EMAIL-CORE-1001`, `EMAIL-CORE-5000`, `EMAIL-CORE-5001`, `EMAIL-RCP-4001`

**Exceptions thrown:** `BusinessRuleViolationException` (empty-to guard)

**Business rules enforced:**

| Rule | Enforcement | Code |
|---|---|---|
| At least one valid `to` after normalization | `if (normalized.to().isEmpty())` → throw | `EMAIL-RCP-4001` |
| Flow must resolve | Delegates to `EmailFlowProvider.resolve()` | `EMAIL-PROV-4002` (via provider) |
| Send interceptors called in order | `beforeSend` → send → `afterSuccess`/`afterFailure` | — |

**OSS send flow (synchronous, single-attempt):**

```
send(EmailSendCommand)
  ├── [1] Send interceptors — beforeSend (if any registered)
  ├── [2] Flow resolution — EmailFlowProvider.resolve(flowKey)
  ├── [3] Recipient normalization — RecipientPolicyHandler.normalize(to, cc, bcc)
  ├── [4] Empty-to guard — throw if normalized.to().isEmpty()
  ├── [5] Message construction — EmailMessageBuilder.build(flow, command)
  ├── [6] Transport send — MailTransport.send(message)
  └── [7] Send interceptors — afterSuccess / afterFailure
```

No retry loop. No circuit breaker check. No async timeout. No idempotency check. Those capabilities activate only when their PaaS modules are on the classpath.

---

### 5.3 `sphuta-arka-email-validation`

**Role:** Startup flow configuration validation.

**Owns:** `DefaultEmailFlowValidator`, `NoopTemplateProbe`

**Auto-configuration:** `EmailValidationAutoConfiguration`

**Activation:** `@ConditionalOnMissingBean(EmailFlowValidator.class)` for validator; `@ConditionalOnMissingBean(TemplateProbe.class)` for `NoopTemplateProbe`

**Codes used:** All `EMAIL-VAL-*` codes

**Exceptions thrown:** `EmailConfigurationException`

**Business rules enforced:**

| Rule | Check | Code | Always? |
|---|---|---|---|
| Every flow must have a template | Null/blank template field | `EMAIL-VAL-4001` | Yes |
| Every flow must have a subject | Null/blank subject field | `EMAIL-VAL-4002` | Yes |
| Every flow must declare sender-ref | Null/blank sender-ref | `EMAIL-VAL-4003` | Yes |
| Sender-ref must resolve | Lookup in sender map | `EMAIL-VAL-4004` | Yes |
| Sender from must be valid email | Syntax validation | `EMAIL-VAL-4005` | Toggle |
| Reply-to must be valid email | Syntax validation | `EMAIL-VAL-4006` | Toggle |
| Attachment-ref must resolve | Lookup in attachment map | `EMAIL-VAL-4007` | Yes |
| Template must exist | `TemplateProbe.exists()` | `EMAIL-VAL-4008` | Toggle |
| Collect ALL errors before throwing | Error list accumulation | `EMAIL-VAL-4000` | Yes |

**Policy toggles:**

```yaml
arka.email.policy.startup-validation:
  validate-template-exists: true
  validate-flow-cc: true
  validate-flow-bcc: true
  validate-from: true
  validate-reply-to: true
```

No master switch. Each toggle is an individually visible, reviewable decision.

---

### 5.4 `sphuta-arka-email-provider-yaml`

**Role:** YAML-backed flow resolution.

**Owns:** `YamlEmailFlowSource`, `YamlFlowParser`

**Auto-configuration:** `EmailYamlProviderAutoConfiguration`

**Activation:** `@ConditionalOnMissingBean(EmailFlowProvider.class)`

**Codes used:** `EMAIL-PROV-4002`, `EMAIL-PROV-4003`, `EMAIL-PROV-4004`, `EMAIL-PROV-4005`

**Exceptions thrown:** `UnknownEmailFlowException`, `EmailConfigurationException`

**Business rules enforced:**

| Rule | Enforcement | Code |
|---|---|---|
| Flow key must exist in YAML | Key lookup | `EMAIL-PROV-4002` |
| Sender-ref must resolve to sender entry | Map lookup | `EMAIL-PROV-4003` / `EMAIL-PROV-4004` |
| Attachment-ref must resolve | Map lookup | `EMAIL-PROV-4005` |
| YAML safe construction | `new Yaml(new SafeConstructor(new LoaderOptions()))` | — |
| `FlowSourceType.YAML` returned from `sourceType()` | Enum return | — |

**Configuration:**

```yaml
arka.email:
  flows:
    enrollment-confirmation:
      template: enrollment/confirmation
      subject: "Welcome aboard"
      sender-ref: default-sender
  senders:
    default-sender:
      from: noreply@example.com
      display-name: "Acme Corp"
```

---

### 5.5 `sphuta-arka-email-transport-javamail`

**Role:** JavaMail SMTP transport with typed exception translation.

**Owns:** `JavaMailTransport`, `MailExceptionTranslator`, `TransportProperties`

**Auto-configuration:** `EmailJavaMailAutoConfiguration`

**Activation:** `@ConditionalOnMissingBean(MailTransport.class)`

**Codes used:** All `EMAIL-TRN-*` codes

**Exceptions thrown:** `EmailTransportException`, `EmailTransportTimeoutException`

**Exception translation table:**

| Root cause | Internal code | Exception type | HTTP |
|---|---|---|---|
| `SocketTimeoutException` | `EMAIL-TRN-5040` | `EmailTransportTimeoutException` | 504 |
| `MailAuthenticationException` | `EMAIL-TRN-5043` | `EmailTransportException` | 502 |
| `SSLException` | `EMAIL-TRN-5044` | `EmailTransportException` | 502 |
| `ConnectException` / `UnknownHostException` | `EMAIL-TRN-5045` | `EmailTransportException` | 502 |
| Session init failure | `EMAIL-TRN-5046` | `EmailTransportException` | 502 |
| Stale connection | `EMAIL-TRN-5047` | `EmailTransportException` | 502 |
| Generic `MailSendException` | `EMAIL-TRN-5020` | `EmailTransportException` | 502 |
| Subsystem unavailable | `EMAIL-TRN-5031` | `EmailTransportException` | **503** |

**Business rules enforced:**

| Rule | Enforcement |
|---|---|
| All low-level exceptions translated to typed exceptions | `MailExceptionTranslator` — every exit returns typed exception, never raw `RuntimeException` |
| Timeout properties wired into `JavaMailSenderImpl` | `TransportProperties` bound at startup |
| Missing timeout is a production bug | Explicit defaults in properties |

**Configuration:**

```yaml
arka.email.transport:
  connection-timeout: 5s
  read-timeout: 10s
  write-timeout: 10s
```

---

### 5.6 `sphuta-arka-email-recipient`

**Role:** Recipient normalization, validation, and deduplication.

**Owns:** `DefaultRecipientNormalizer`, typed `NormalizedRecipients` return

**Auto-configuration:** `EmailRecipientAutoConfiguration`

**Activation:** `@ConditionalOnMissingBean(RecipientPolicyHandler.class)`

**Codes used:** `EMAIL-RCP-4000`, `EMAIL-RCP-4002`

**Exceptions thrown:** `InvalidRecipientException` (when `RecipientAction.FAIL`)

**Business rules enforced:**

| Rule | Enforcement | Code |
|---|---|---|
| Parse and validate each address | RFC-compliant parse | `EMAIL-RCP-4000` |
| Invalid `to` → action per policy | `FAIL` throws, `DROP_WARN` logs and drops, `DROP_SILENT` drops silently | `EMAIL-RCP-4000` |
| Invalid `cc`/`bcc` → `DROP_WARN` | Log `WARN`, remove from list | `EMAIL-RCP-4000` |
| Dedup precedence: `to` > `cc` > `bcc` | Address in higher-priority list wins | `EMAIL-RCP-4002` |
| Log `WARN` on every dedup collapse | WARN with masked address | `EMAIL-RCP-4002` |
| Stored form preserves case, canonical key lowercases domain | RFC 5321 compliance | — |
| Empty collection, never null | `List.of()` for absent lists | — |

**Normalization pipeline:**

```
Input (to, cc, bcc)
  ├── [1] Normalize: trim, extract from display-name format, lowercase domain
  ├── [2] Validate: RFC-compliant syntax check per address
  │       ├── valid → pass through
  │       └── invalid → apply RecipientAction (FAIL / DROP_WARN / DROP_SILENT)
  ├── [3] Merge: flow-level cc/bcc + runtime cc/bcc → combined lists
  ├── [4] Deduplicate: to > cc > bcc precedence
  │       └── collapse → log WARN per collapsed address
  └── [5] Build NormalizedRecipients (List.copyOf on all three lists)
```

**Stored vs canonical form:**

| Input | Stored form | Canonical key |
|---|---|---|
| `John Doe <John@Example.COM>` | `John@example.com` | `john@example.com` |
| `JOHN@EXAMPLE.COM` | `JOHN@example.com` | `john@example.com` |
| `  user@example.com  ` | `user@example.com` | `user@example.com` |

---

### 5.7 `sphuta-arka-email-message`

**Role:** MIME message construction and final message assembly.

**Owns:** `DefaultEmailMessageBuilder`, `FinalEmailMessageFactory`, `MimeMessageBuilder`

**Auto-configuration:** `EmailMessageAutoConfiguration`

**Activation:** `@ConditionalOnMissingBean(EmailMessageBuilder.class)`

**Codes used:** All `EMAIL-MSG-*` codes

**Exceptions thrown:** `EmailConfigurationException`

**Business rules enforced:**

| Rule | Enforcement | Code |
|---|---|---|
| Template rendered via `TemplateRenderer` if present | Optional bean dependency | — |
| Static flow CC/BCC merged with runtime CC/BCC | List merge in builder | — |
| Sender display name applied | `ResolvedSender.displayName()` | `EMAIL-MSG-5003` if format invalid |
| Attachment resources resolved | Classpath or filesystem lookup | `EMAIL-MSG-5001` |
| `FinalEmailMessage` is fully immutable | `List.copyOf()` on all collections | — |
| MIME construction failures translated | Catch `MessagingException` → typed exception | `EMAIL-MSG-5002` |

---

### 5.8 `sphuta-arka-email-template-thymeleaf`

**Role:** Thymeleaf-based template rendering with template existence probing.

**Owns:** `ThymeleafTemplateRenderer`, `ThymeleafTemplateProbe`

**Auto-configuration:** `EmailTemplateAutoConfiguration`

**Activation:** `@ConditionalOnClass(SpringTemplateEngine.class)` + `@ConditionalOnMissingBean(TemplateRenderer.class)`

**Codes used:** All `EMAIL-TPL-*` codes

**Exceptions thrown:** `EmailConfigurationException`

**Business rules enforced:**

| Rule | Enforcement | Code |
|---|---|---|
| Template existence probing at startup | `ThymeleafTemplateProbe.exists()` checks engine | `EMAIL-TPL-5001` |
| Template rendering failures translated | Catch Thymeleaf exceptions → typed exception | `EMAIL-TPL-5000` |
| Template variable resolution failures | Variable binding errors caught | `EMAIL-TPL-5002` |

---

### 5.9 `sphuta-arka-email-starter`

**Role:** Composition POM only. Owns no code.

**Bundles:** core + service + validation + provider-yaml + transport-javamail + recipient + message + template-thymeleaf

---

## 6. Validation Layer Matrix

Five layers, each with exactly one responsibility. No layer duplicates another.

| Layer | Class | Trigger | Owns | Module |
|---|---|---|---|---|
| 1. HTTP shape | DTOs (PaaS — web module) | Per HTTP request | Presence and blank checks only | web (PaaS) |
| 2. Command invariants | `EmailSendCommand` | At construction | Non-null flowKey, defensive copies | core |
| 3. Startup config | `DefaultEmailFlowValidator` | Application startup | Flow config correctness, syntax under toggles | validation |
| 4. Recipient policy | `DefaultRecipientNormalizer` | Per send | Parse, validate, deduplicate, apply RecipientAction | recipient |
| 5. Send invariants | `DefaultEmailFlowService` | Pre-dispatch | At least one valid `to` after normalization | service |

**Critical rules (per coding standards):**

- `@Email` must NOT appear on config property classes — it bypasses policy toggles and fails at bind time
- Startup validation collects ALL errors before throwing — operator sees the complete picture in one failure
- The empty-to guard belongs in service (layer 5), not in recipient normalizer (layer 4)

---

## 7. Business Rules — Complete Registry

### Always-enforced (cannot be suppressed)

| Rule | Where | What happens |
|---|---|---|
| `flowKey` not null or blank | `EmailSendCommand` constructor | `NullPointerException` / `IllegalArgumentException` |
| `ApplicationCode` not null on exceptions | `EmailApplicationException` constructor | `NullPointerException` |
| All collection fields defensively copied | All record constructors | `List.copyOf()` applied |
| Null collections normalize to empty | Record compact constructors | `null → List.of()` |
| Flow must exist | `EmailFlowProvider.resolve()` | `UnknownEmailFlowException` |
| At least one valid `to` after normalization | `DefaultEmailFlowService` step 4 | `BusinessRuleViolationException` |
| No duplicate recipients | `DefaultRecipientNormalizer` | Dedup with precedence `to > cc > bcc` |
| Transport exceptions always translated | `MailExceptionTranslator` | Typed exception with code, never raw |
| `InterruptedException` always restores interrupt flag | All catch blocks platform-wide | `Thread.currentThread().interrupt()` before re-throw |
| Template, subject, sender-ref present per flow | `DefaultEmailFlowValidator` | `EmailConfigurationException` |
| Sender-ref resolvable | `DefaultEmailFlowValidator` | `EmailConfigurationException` |
| Collect all validation errors before throwing | `DefaultEmailFlowValidator` | Accumulate → single throw |
| YAML safe construction | `YamlEmailFlowSource` | `SafeConstructor` always used |
| No `UUID.randomUUID()` downstream | Platform-wide tracking ID rule | Missing context → `"untraced"` + WARN |

### Toggle-dependent (can be suppressed via policy)

| Rule | Toggle | When disabled |
|---|---|---|
| Sender from syntax valid | `validate-from` | Skip check, trust runtime |
| Reply-to syntax valid | `validate-reply-to` | Skip check, trust runtime |
| Flow CC syntax valid | `validate-flow-cc` | Skip check, trust runtime |
| Flow BCC syntax valid | `validate-flow-bcc` | Skip check, trust runtime |
| Template exists at startup | `validate-template-exists` | Skip existence probe |

---

## 8. Retriability Classification

In OSS, send is single-attempt — no retry loop. This classification applies when the PaaS retry module is present, but is also relevant for consumers implementing their own retry.

| Code | Failure | Retriable | Reason |
|---|---|---|---|
| `EMAIL-TRN-5040` | SMTP timeout | Yes | Transient network issue |
| `EMAIL-TRN-5045` | SMTP connection refused | Yes | Transient network issue |
| `EMAIL-TRN-5047` | Stale connection | Yes | Transient network issue |
| `EMAIL-TRN-5020` | Generic SMTP failure | Configurable (default: no) | Avoids retrying permanent rejections |
| `EMAIL-TRN-5043` | SMTP auth failed | Never | Permanent until config change |
| `EMAIL-TRN-5044` | SMTP TLS failed | Never | Permanent until config change |
| `EMAIL-TRN-5046` | Session init failed | Never | Permanent until config change |
| `EMAIL-TRN-5031` | Subsystem unavailable | Never | Infrastructure problem |
| `EMAIL-VAL-*` | Startup validation | Never | Configuration problem |
| `EMAIL-PROV-4002` | Flow not found | Never | Configuration problem |
| `EMAIL-RCP-*` | Recipient issues | Never | Data problem |
| `EMAIL-MSG-*` | Message construction | Never | Configuration problem |
| `EMAIL-TPL-*` | Template rendering | Never | Configuration problem |

---

## 9. What Core Does NOT Contain

Per the tier isolation rule, the following are **not** in `sphuta-arka-email-core`. Each is owned by its respective PaaS or SaaS module.

### Contracts owned by PaaS modules

| Contract | Lives in module |
|---|---|
| `EmailBatchDispatcher` (API) | `sphuta-arka-email-async` |
| `BatchSendResult` (API) | `sphuta-arka-email-async` |
| `EmailRetryPolicy` (SPI) | `sphuta-arka-email-retry-fixed` |
| `EmailCircuitBreaker` (SPI) | `sphuta-arka-email-circuit-breaker` |
| `ExecutionContextPropagator` (SPI) | `sphuta-arka-email-async` |
| `EmailThrottlePolicy` (SPI) | `sphuta-arka-email-async` |

### Contracts owned by SaaS modules

| Contract | Lives in module |
|---|---|
| `TenantResolver` (SPI) | `sphuta-arka-email-tenant` |

### Code families owned by PaaS modules

| Family | Codes | Lives in module |
|---|---|---|
| `EmailAsyncCode` (`EMAIL-ASYNC-*`) | 6 | `sphuta-arka-email-async` |
| `EmailBatchCode` (`EMAIL-BATCH-*`) | 6 | `sphuta-arka-email-async` |
| `EmailPersistenceCode` (`EMAIL-PERSIST-*`) | 5 | `sphuta-arka-email-provider-db` |
| `HYBRID_FALLBACK` (`EMAIL-PROV-5001`) | 1 | `sphuta-arka-email-provider-hybrid` |

### Exceptions owned by PaaS modules

| Exception | Lives in module |
|---|---|
| `EmailAsyncTimeoutException` | `sphuta-arka-email-async` |
| `EmailTaskInterruptedException` | `sphuta-arka-email-async` |
| `EmailTaskCancelledException` | `sphuta-arka-email-async` |
| `BatchDispatcherUnavailableException` | `sphuta-arka-email-async` |

---

## Appendix — Maven Coordinates

```xml
<!-- OSS starter — single dependency for consumers -->
<dependency>
    <groupId>net.sphuta.arka.email</groupId>
    <artifactId>sphuta-arka-email-starter</artifactId>
    <version>${arka.email.version}</version>
</dependency>
```

**All 9 OSS artifacts:**

| GroupId | ArtifactId |
|---|---|
| `net.sphuta.arka.email` | `sphuta-arka-email-core` |
| `net.sphuta.arka.email` | `sphuta-arka-email-service` |
| `net.sphuta.arka.email` | `sphuta-arka-email-validation` |
| `net.sphuta.arka.email` | `sphuta-arka-email-provider-yaml` |
| `net.sphuta.arka.email` | `sphuta-arka-email-transport-javamail` |
| `net.sphuta.arka.email` | `sphuta-arka-email-recipient` |
| `net.sphuta.arka.email` | `sphuta-arka-email-message` |
| `net.sphuta.arka.email` | `sphuta-arka-email-template-thymeleaf` |
| `net.sphuta.arka.email` | `sphuta-arka-email-starter` |

---

*All decisions in this document are locked.*
*Tier isolation rule: core owns only OSS contracts. PaaS modules own PaaS contracts. SaaS modules own SaaS contracts.*
*Changes require explicit review and update before implementation.*
