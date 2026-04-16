#  Arka — OSS Tier Complete Reference

> **Scope:** All application codes, exceptions, business rules, and validations for the OSS tier across all channels
> **Architecture:** Shared engine + channel edges
> **Modules covered:** 4 shared engine + per-channel OSS edges + per-channel composition starters
> **Channels:** Email (frozen), WhatsApp (frozen), Slack (reserved), SMS (reserved)
> **Status:** Aligned with frozen 33-artifact architecture, strict fail-fast activation, and tier isolation rule
>
> **Tier isolation rule (locked):** OSS core owns only OSS contracts, codes, and exceptions.
> PaaS modules own PaaS contracts. SaaS modules own SaaS contracts. No cross-tier contract leakage.
>
> **Activation rule (locked):** Classpath presence expresses intent. Missing hard prerequisites are configuration errors — startup fails. Properties tune behavior only. No enabled flags anywhere.

---

## Contents

1. [OSS Tier Module Inventory — All Channels](#1-oss-tier-module-inventory--all-channels)
2. [Shared Engine — Application Codes](#2-shared-engine--application-codes)
3. [Shared Engine — Exception Hierarchy](#3-shared-engine--exception-hierarchy)
4. [Shared Engine — Per-Module Reference](#4-shared-engine--per-module-reference)
5. [Email Channel — OSS Edge Reference](#5-email-channel--oss-edge-reference)
6. [WhatsApp Channel — OSS Edge Reference](#6-whatsapp-channel--oss-edge-reference)
7. [Slack Channel — OSS Edge Reference (Reserved)](#7-slack-channel--oss-edge-reference-reserved)
8. [SMS Channel — OSS Edge Reference (Reserved)](#8-sms-channel--oss-edge-reference-reserved)
9. [Cross-Channel Validation Layer Matrix](#9-cross-channel-validation-layer-matrix)
10. [Cross-Channel Retriability Classification](#10-cross-channel-retriability-classification)
11. [Cross-Channel Business Rules](#11-cross-channel-business-rules)
12. [What OSS Core Does NOT Contain](#12-what-oss-core-does-not-contain)
13. [Starter Composition](#13-starter-composition)
14. [Maven Coordinates](#14-maven-coordinates)

---

## 1. OSS Tier Module Inventory — All Channels

OSS is the minimal self-managed runtime. Synchronous single-send. No async, no retry, no DB/hybrid providers, no batch dispatch, no security, no observability.

### Shared engine — OSS (4 modules)

| Module | Artifact ID | Java Package | Role |
|---|---|---|---|
| Core | `-arka-core` | `net..arka.core` | Shared SPIs, base exceptions, ApplicationCode, ConfigCategory |
| Service | `-arka-service` | `net..arka.service` | Generic send orchestration engine |
| Validation | `-arka-validation` | `net..arka.validation` | Generic startup validation engine |
| Provider YAML | `-arka-provider-yaml` | `net..arka.provider.yaml` | YAML-backed flow resolution engine |

### Email edge — OSS (5 modules)

| Module | Artifact ID | Java Package | Role |
|---|---|---|---|
| Email Core | `-arka-email-core` | `net..arka.email` | Email contracts, models, EMAIL-* codes, exceptions, adapter beans |
| SMTP Transport | `-arka-email-transport-javamail` | `net..arka.email.transport.javamail` | SMTP transport with typed exception translation |
| Recipient | `-arka-email-recipient` | `net..arka.email.recipient` | Recipient normalization, validation, deduplication |
| Message | `-arka-email-message` | `net..arka.email.message` | MIME message construction and final message assembly |
| Template Thymeleaf | `-arka-email-template-thymeleaf` | `net..arka.email.template.thymeleaf` | Thymeleaf-based template rendering |

### WhatsApp edge — OSS (3 modules)

| Module | Artifact ID | Java Package | Role |
|---|---|---|---|
| WhatsApp Core | `-arka-whatsapp-core` | `net..arka.whatsapp` | WhatsApp contracts, models, WA-* codes, exceptions, adapter beans |
| Meta Transport | `-arka-whatsapp-transport-meta` | `net..arka.whatsapp.transport.meta` | Meta Cloud API send transport with typed exception translation |
| Template Meta | `-arka-whatsapp-template-meta` | `net..arka.whatsapp.template.meta` | Meta Template Management API — sync, approval check, category rules |

### Slack edge — OSS reserved (4 modules)

| Module | Artifact ID | Java Package | Role |
|---|---|---|---|
| Slack Core | `-arka-slack-core` | `net..arka.slack` | Slack contracts, models, SLACK-* codes, exceptions, adapter beans |
| Webhook Transport | `-arka-slack-transport-webhook` | `net..arka.slack.transport.webhook` | Slack Incoming Webhooks transport |
| API Transport | `-arka-slack-transport-api` | `net..arka.slack.transport.api` | Slack Web API transport (chat.postMessage) |
| Block Kit Message | `-arka-slack-message-blockkit` | `net..arka.slack.message.blockkit` | Block Kit message construction |

### SMS edge — OSS reserved (4 modules)

| Module | Artifact ID | Java Package | Role |
|---|---|---|---|
| SMS Core | `-arka-sms-core` | `net..arka.sms` | SMS contracts, models, SMS-* codes, exceptions, adapter beans |
| Twilio Transport | `-arka-sms-transport-twilio` | `net..arka.sms.transport.twilio` | Twilio SMS transport |
| SNS Transport | `-arka-sms-transport-sns` | `net..arka.sms.transport.sns` | AWS SNS SMS transport |
| Segment Message | `-arka-sms-message-segment` | `net..arka.sms.message.segment` | SMS segment splitting and encoding |

### Composition starters

| Channel | Artifact ID | Bundles |
|---|---|---|
| Email | `-arka-email-starter` | 4 shared OSS engine + 5 Email OSS edge |
| WhatsApp | `-arka-whatsapp-starter` | 4 shared OSS engine + 3 WhatsApp OSS edge |
| Slack | `-arka-slack-starter` | 4 shared OSS engine + 4 Slack OSS edge |
| SMS | `-arka-sms-starter` | 4 shared OSS engine + 4 SMS OSS edge |

### OSS artifact counts

| Layer | Capability modules | Starters | Total |
|---|---|---|---|
| Shared engine | 4 | — | 4 |
| Email edge | 5 | 1 | 6 |
| WhatsApp edge | 3 | 1 | 4 |
| Slack edge (reserved) | 4 | 1 | 5 |
| SMS edge (reserved) | 4 | 1 | 5 |

---

## 2. Shared Engine — Application Codes

Shared engine codes live in `-arka-core` in the `net..arka.core.code` package. Every code implements `ApplicationCode` with `code()` + `defaultMessage()`.

### `ArkaCoreCode` — `ARKA-CORE-*`

Generic service lifecycle events. Used by the shared service engine.

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `ARKA-CORE-1000` | `SEND_STARTED` | Send workflow started | Observability-only |
| `ARKA-CORE-1001` | `SEND_SUCCEEDED` | Send workflow completed | Observability-only |
| `ARKA-CORE-1002` | `FLOW_RESOLUTION_USED` | Flow resolution used | Observability-only |
| `ARKA-CORE-5000` | `SEND_FAILED` | Send workflow failed | Response-producing |
| `ARKA-CORE-5001` | `UNEXPECTED_ERROR` | Unexpected error | Response-producing |

### `ArkaValidationCode` — `ARKA-VAL-*`

Shared startup validation codes.

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `ARKA-VAL-4000` | `VALIDATION_FAILED` | Startup validation failed | Response-producing |
| `ARKA-VAL-4001` | `FLOW_KEY_MISSING` | Flow key is missing | Response-producing |

### `ArkaProviderCode` — `ARKA-PROV-*`

Provider resolution codes.

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `ARKA-PROV-4000` | `PROVIDER_AMBIGUOUS` | Ambiguous provider selection | Response-producing |
| `ARKA-PROV-4001` | `PROVIDER_DUPLICATE` | Duplicate provider candidates | Response-producing |
| `ARKA-PROV-4002` | `FLOW_NOT_FOUND` | Flow not found in any provider | Response-producing |

### `ArkaClientCode` — `ARKA-CLIENT-*`

Client descriptor uniqueness codes.

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `ARKA-CLIENT-4000` | `DESCRIPTOR_DUPLICATE` | Duplicate ChannelClientDescriptor for same channel key | Response-producing |

### Shared engine code count

| Family | Prefix | Codes |
|---|---|---|
| Core | `ARKA-CORE-*` | 5 |
| Validation | `ARKA-VAL-*` | 2 |
| Provider | `ARKA-PROV-*` | 3 |
| Client | `ARKA-CLIENT-*` | 1 |
| **Total in shared core** | | **11** |

---

## 3. Shared Engine — Exception Hierarchy

All shared exceptions live in `net..arka.core.exception` within `-arka-core`. Every exception carries an `ApplicationCode`. Three constructors per platform standard.

```
ArkaApplicationException (abstract — carries ApplicationCode)
│
├── ArkaConfigurationException            → 500
├── ArkaTransportException                → 502 / 503 (code-based)
│   └── ArkaTransportTimeoutException     → 504
├── UnknownFlowException                  → 404
├── BusinessRuleViolationException        → 422
├── ArkaAsyncTimeoutException             → 504
├── ArkaTaskInterruptedException          → 503
├── ArkaTaskCancelledException            → 503
└── BatchDispatcherUnavailableException   → 503
```

Channel exceptions extend these base types:

```
EmailApplicationException extends ArkaApplicationException
WhatsAppApplicationException extends ArkaApplicationException
SlackApplicationException extends ArkaApplicationException
SmsApplicationException extends ArkaApplicationException
```

---

## 4. Shared Engine — Per-Module Reference

### 4.1 `-arka-core`

**Role:** Pure contracts — all shared SPIs, base exception hierarchy, `ApplicationCode`, `ConfigCategory`, generic adapter contracts.

**Rule:** No class in core may import anything outside `java.*` (plus JSpecify annotations).

**Shared SPIs owned:**

| SPI | Purpose |
|---|---|
| `SendCommand` | Marker interface for all channel send commands |
| `FlowService<C extends SendCommand>` | Primary send orchestration |
| `BatchDispatcher<C extends SendCommand>` | Batch dispatch (PaaS, but contract in core for type safety) |
| `FlowProvider<F>` | Flow resolution by key |
| `FlowConfigMapper<F>` | Raw config to typed resolved flow |
| `Transport<M>` | Send final message via channel protocol |
| `MessageBuilder<F, C, M>` | Construct send-ready message |
| `FlowValidator<F>` | Startup validation |
| `RetryFailureClassifier` | Retriable/non-retriable classification |
| `SendObservationConvention<C>` | Channel metric vocabulary |
| `ArkaHealthContributor` | Health probes (additive) |
| `StartupReadinessGate` | Readiness checks (additive) |
| `AuthorizationPolicy` | Channel authorization decisions |
| `AuthorizationTarget` | Resource being accessed |
| `SecuritySubject` | Authenticated principal |
| `OutboxPayloadSerializer<C>` | Outbox serialization |
| `OutboxDispatcher<C>` | Outbox replay dispatch |
| `DlqPayloadHandler<C>` | DLQ entry handling |
| `SchedulerPayloadHandler<C>` | Scheduler entry handling |
| `ChannelClientDescriptor<C, R>` | Client endpoint paths and DTOs |
| `ExecutionContextPropagator` | Thread-local context across virtual threads |
| `ApplicationCode` | `code()` + `defaultMessage()` |
| `ConfigCategory` | Exhaustive switch categories |

**Business rules enforced in core:**

| Rule | Location | Enforcement |
|---|---|---|
| `ApplicationCode` never null on exceptions | `ArkaApplicationException` constructor | `Objects.requireNonNull` |
| 3-constructor discipline | All concrete exceptions | 1-arg, 2-arg, 3-arg |
| Exhaustive switch on `ConfigCategory` | Required by handler implementations | Compiler-enforced switch |

### 4.2 `-arka-service`

**Role:** Generic send orchestration engine. Delegates to channel adapters for flow resolution, destination handling, message building, and transport.

**OSS send flow (synchronous, single-attempt):**

```
send(SendCommand)
  1. SendInterceptor.beforeSend() [additive — all beans called]
  2. FlowProvider.resolve(flowKey)
  3. Channel-specific destination handling
  4. MessageBuilder.build(flow, command)
  5. Transport.send(message)
  6. SendInterceptor.afterSuccess() / afterFailure()
```

No retry loop. No circuit breaker. No async timeout. No idempotency check. Those activate when PaaS modules are on classpath.

### 4.3 `-arka-validation`

**Role:** Generic startup validation engine. Collects all errors before throwing — operator sees the complete picture.

**Business rules enforced:**

| Rule | Enforcement |
|---|---|
| Collect ALL errors before throwing | Error list accumulation → single throw |
| Delegates to channel `FlowValidator<F>` | Channel provides rules |

### 4.4 `-arka-provider-yaml`

**Role:** YAML-backed flow resolution engine. Reads YAML properties, delegates to channel `FlowConfigMapper<F>`.

**Business rules enforced:**

| Rule | Enforcement |
|---|---|
| Flow key must exist in YAML | Key lookup → `UnknownFlowException` |
| Delegates mapping to channel mapper | `FlowConfigMapper<F>` SPI |
| YAML safe construction | `SafeConstructor` always used |

---

## 5. Email Channel — OSS Edge Reference

### 5.1 Email Application Codes

All Email OSS codes live in `-arka-email-core` in `net..arka.email.code`.

#### `EmailCoreCode` — `EMAIL-CORE-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `EMAIL-CORE-1000` | `SEND_STARTED` | Send workflow started | Observability-only |
| `EMAIL-CORE-1001` | `SEND_SUCCEEDED` | Send workflow completed | Observability-only |
| `EMAIL-CORE-1002` | `FLOW_RESOLUTION_USED` | Flow resolution used | Observability-only |
| `EMAIL-CORE-5000` | `SEND_FAILED` | Send workflow failed | Response-producing |
| `EMAIL-CORE-5001` | `UNEXPECTED_ERROR` | Unexpected error | Response-producing |

#### `EmailValidationCode` — `EMAIL-VAL-*`

Each carries a `ConfigCategory`.

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

#### `EmailRecipientCode` — `EMAIL-RCP-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `EMAIL-RCP-4000` | `INVALID_ADDRESS_FORMAT` | Recipient address format is invalid | Response-producing |
| `EMAIL-RCP-4001` | `NO_VALID_PRIMARY_RECIPIENT` | No valid primary recipient after policy resolution | Response-producing |
| `EMAIL-RCP-4002` | `DUPLICATE_COLLAPSED` | Duplicate recipient collapsed | Observability-only |

#### `EmailProviderCode` — `EMAIL-PROV-*`

| Code | Constant | Default message | Category |
|---|---|---|---|
| `EMAIL-PROV-4002` | `FLOW_NOT_FOUND_ANYWHERE` | Flow not found in any provider | — |
| `EMAIL-PROV-4003` | `SENDER_REF_MISSING` | Sender reference missing in flow | `SENDER` |
| `EMAIL-PROV-4004` | `SENDER_REF_UNRESOLVABLE` | Sender reference could not be resolved | `SENDER` |
| `EMAIL-PROV-4005` | `ATTACHMENT_REF_UNRESOLVABLE` | Attachment reference could not be resolved | `ATTACHMENT` |

Note: `HYBRID_FALLBACK` (`EMAIL-PROV-5001`) lives in the PaaS hybrid provider module, not in core.

#### `EmailTransportCode` — `EMAIL-TRN-*`

| Code | Constant | Default message | Retriable | HTTP |
|---|---|---|---|---|
| `EMAIL-TRN-5020` | `SMTP_SEND_FAILED` | SMTP send failed | Conditional | 502 |
| `EMAIL-TRN-5031` | `SMTP_SUBSYSTEM_ERROR` | Mail subsystem unavailable | No | 503 |
| `EMAIL-TRN-5040` | `SMTP_TIMEOUT` | SMTP timeout | Yes | 504 |
| `EMAIL-TRN-5043` | `SMTP_AUTH_FAILED` | SMTP auth failed | No | 502 |
| `EMAIL-TRN-5044` | `SMTP_TLS_FAILED` | SMTP TLS failed | No | 502 |
| `EMAIL-TRN-5045` | `SMTP_CONNECTION_REFUSED` | SMTP connection refused | Yes | 502 |
| `EMAIL-TRN-5046` | `SMTP_SESSION_INIT_FAILED` | SMTP session init failed | No | 502 |
| `EMAIL-TRN-5047` | `SMTP_STALE_CONNECTION` | Stale or broken connection | Yes | 502 |

Note on `EMAIL-TRN-5040`: Single code for connect, read, and write timeouts. JavaMail does not reliably distinguish them.

Note on `EMAIL-TRN-5031`: Maps to 503 (temporarily unavailable), not 502.

#### `EmailTemplateCode` — `EMAIL-TPL-*`

| Code | Constant | Default message | Category |
|---|---|---|---|
| `EMAIL-TPL-5000` | `TEMPLATE_RENDER_FAILED` | Template rendering failed | `TEMPLATE` |
| `EMAIL-TPL-5001` | `TEMPLATE_NOT_RESOLVABLE` | Template not resolvable | `TEMPLATE` |
| `EMAIL-TPL-5002` | `TEMPLATE_VARIABLE_ERROR` | Template variable error | `TEMPLATE` |

#### `EmailMessageCode` — `EMAIL-MSG-*`

| Code | Constant | Default message | Category |
|---|---|---|---|
| `EMAIL-MSG-5000` | `MESSAGE_BUILD_FAILED` | Message build failed | `MESSAGE` |
| `EMAIL-MSG-5001` | `ATTACHMENT_LOAD_FAILED` | Attachment load failed | `MESSAGE` |
| `EMAIL-MSG-5002` | `MIME_CONSTRUCTION_FAILED` | MIME construction failed | `MESSAGE` |
| `EMAIL-MSG-5003` | `SENDER_FORMAT_INVALID` | Sender format invalid | `MESSAGE` |

#### Email `ConfigCategory`

| Constant | Scope |
|---|---|
| `FLOW` | Flow definition: template, subject, flow key |
| `SENDER` | Sender identity: from, reply-to, sender-ref |
| `ATTACHMENT` | Attachment references |
| `TEMPLATE` | Template rendering |
| `MESSAGE` | Message construction |

#### Email code count — OSS only

| Family | Prefix | Codes |
|---|---|---|
| Core | `EMAIL-CORE-*` | 5 |
| Validation | `EMAIL-VAL-*` | 9 |
| Recipient | `EMAIL-RCP-*` | 3 |
| Provider | `EMAIL-PROV-*` | 4 |
| Transport | `EMAIL-TRN-*` | 8 |
| Template | `EMAIL-TPL-*` | 3 |
| Message | `EMAIL-MSG-*` | 4 |
| **Total** | | **36** |

### 5.2 Email Exception Hierarchy

```
EmailApplicationException extends ArkaApplicationException
│
├── InvalidEmailRequestException          → 400
├── InvalidRecipientException             → 400
├── BusinessRuleViolationException        → 422
├── UnknownEmailFlowException             → 404
├── EmailConfigurationException           → 500
├── EmailTransportException               → 502 / 503
│   └── EmailTransportTimeoutException    → 504
```

7 exception types. All OSS-active.

### 5.3 Email Exception-to-HTTP Status Mapping

#### Direct exception-type mapping

| Exception type | HTTP | HTTP code |
|---|---|---|
| `InvalidEmailRequestException` | 400 | `EMAIL-HTTP-4000` |
| `InvalidRecipientException` | 400 | `EMAIL-HTTP-4001` |
| `UnknownEmailFlowException` | 404 | `EMAIL-HTTP-4002` |
| `BusinessRuleViolationException` | 422 | `EMAIL-HTTP-4003` |
| `EmailTransportTimeoutException` | 504 | `EMAIL-HTTP-5040` |
| `EmailTransportException` | code-based | see below |

#### Transport code-based HTTP mapping

| Transport code | HTTP | HTTP code | Rationale |
|---|---|---|---|
| `EMAIL-TRN-5031` | 503 | `EMAIL-HTTP-5031` | Subsystem temporarily unavailable |
| `EMAIL-TRN-5040` | 504 | `EMAIL-HTTP-5040` | Handled by timeout subtype |
| All other `EMAIL-TRN-*` | 502 | `EMAIL-HTTP-5020` | Gateway/upstream SMTP failure |

#### Configuration category-based mapping

| ConfigCategory | HTTP code | HTTP |
|---|---|---|
| `FLOW` | `EMAIL-HTTP-5004` | 500 |
| `SENDER` | `EMAIL-HTTP-5002` | 500 |
| `ATTACHMENT` | `EMAIL-HTTP-5003` | 500 |
| `TEMPLATE` | `EMAIL-HTTP-5005` | 500 |
| `MESSAGE` | `EMAIL-HTTP-5005` | 500 |

### 5.4 Email Per-Module Reference

#### `-arka-email-core`

**Role:** Email contracts, models, codes, exceptions, and all adapter beans for shared engines.

**Adapter beans provided:**

| Shared engine | Email adapter |
|---|---|
| `service` | `EmailFlowService` |
| `validation` | `EmailFlowValidator` |
| `provider-*` | `EmailFlowConfigMapper` |
| `retry-fixed` | `EmailRetryFailureClassifier` |
| `observability` | `EmailObservationConvention` |
| `health` | `SmtpHealthContributor`, `EmailReadinessGate` |
| `security` | `EmailAuthorizationPolicy` |
| `outbox` | `EmailOutboxSerializer`, `EmailOutboxDispatcher` |
| `dlq` | `EmailDlqPayloadHandler` |
| `scheduler` | `EmailSchedulerPayloadHandler` |
| `client` | `EmailChannelClientDescriptor` |

**Owns:**

| Package | Contents |
|---|---|
| `api` | `EmailFlowService`, `EmailSendCommand`, `RecipientSendResult` |
| `api.builder` | `EmailSendCommandBuilder` |
| `spi` | `EmailFlowProvider`, `MailTransport`, `EmailMessageBuilder`, `RecipientPolicyHandler`, `EmailFlowValidator`, `TemplateRenderer`, `TemplateProbe`, `EmailSendInterceptor` |
| `model.flow` | `ResolvedEmailFlow`, `ResolvedSender`, `ResolvedAttachment`, `FinalEmailMessage`, `NormalizedRecipients` |
| `model.enums` | `FlowSourceType`, `RecipientPolicyMode` |
| `code` | 7 code enums (36 codes total) |
| `exception` | `EmailApplicationException` (abstract), 7 concrete types |

#### `-arka-email-transport-javamail`

**Role:** SMTP transport with typed exception translation.

**Exception translation table:**

| Root cause | Code | Exception type | HTTP |
|---|---|---|---|
| `SocketTimeoutException` | `EMAIL-TRN-5040` | `EmailTransportTimeoutException` | 504 |
| `MailAuthenticationException` | `EMAIL-TRN-5043` | `EmailTransportException` | 502 |
| `SSLException` | `EMAIL-TRN-5044` | `EmailTransportException` | 502 |
| `ConnectException` / `UnknownHostException` | `EMAIL-TRN-5045` | `EmailTransportException` | 502 |
| Session init failure | `EMAIL-TRN-5046` | `EmailTransportException` | 502 |
| Stale connection | `EMAIL-TRN-5047` | `EmailTransportException` | 502 |
| Generic `MailSendException` | `EMAIL-TRN-5020` | `EmailTransportException` | 502 |
| Subsystem unavailable | `EMAIL-TRN-5031` | `EmailTransportException` | 503 |

**Hard prerequisite:** JavaMail on classpath. Startup fails if absent.

**Configuration:**

```yaml
arka:
  email:
    transport:
      connection-timeout: 5s
      read-timeout: 10s
      write-timeout: 10s
```

#### `-arka-email-recipient`

**Role:** Recipient normalization, validation, and deduplication.

**Normalization pipeline:**

```
Input (to, cc, bcc)
  1. Normalize: trim, extract from display-name format, lowercase domain
  2. Validate: RFC-compliant syntax check per address
     valid → pass through
     invalid → apply RecipientAction (FAIL / DROP_WARN / DROP_SILENT)
  3. Merge: flow-level cc/bcc + runtime cc/bcc → combined lists
  4. Deduplicate: to > cc > bcc precedence
     collapse → log WARN per collapsed address
  5. Build NormalizedRecipients (List.copyOf on all three lists)
```

**Stored vs canonical form:**

| Input | Stored form | Canonical key |
|---|---|---|
| `John Doe <John@Example.COM>` | `John@example.com` | `john@example.com` |
| `JOHN@EXAMPLE.COM` | `JOHN@example.com` | `john@example.com` |

**Configuration:**

```yaml
arka:
  email:
    policy:
      runtime-validation:
        invalid-to-action: FAIL
        invalid-cc-action: DROP_WARN
        invalid-bcc-action: DROP_WARN
      recipients:
        dedupe-enabled: true
```

#### `-arka-email-message`

**Role:** MIME message construction and final message assembly.

**Business rules:**

| Rule | Code |
|---|---|
| Template rendered via `TemplateRenderer` if present | — |
| Static flow CC/BCC merged with runtime CC/BCC | — |
| Sender display name applied | `EMAIL-MSG-5003` if format invalid |
| Attachment resources resolved | `EMAIL-MSG-5001` if load fails |
| MIME construction failures translated | `EMAIL-MSG-5002` |
| `FinalEmailMessage` is fully immutable | `List.copyOf()` |

#### `-arka-email-template-thymeleaf`

**Role:** Thymeleaf-based template rendering with template existence probing.

**Business rules:**

| Rule | Code |
|---|---|
| Template existence probing at startup | `EMAIL-TPL-5001` |
| Template rendering failures translated | `EMAIL-TPL-5000` |
| Template variable resolution failures | `EMAIL-TPL-5002` |

**Hard prerequisite:** Thymeleaf on classpath. Startup fails if absent.

### 5.5 Email Validation Rules

| Rule | Check | Code | Toggle |
|---|---|---|---|
| Every flow must have a template | Null/blank | `EMAIL-VAL-4001` | Always |
| Every flow must have a subject | Null/blank | `EMAIL-VAL-4002` | Always |
| Every flow must declare sender-ref | Null/blank | `EMAIL-VAL-4003` | Always |
| Sender-ref must resolve | Lookup | `EMAIL-VAL-4004` | Always |
| Sender from must be valid email | Syntax | `EMAIL-VAL-4005` | `validate-from` |
| Reply-to must be valid email | Syntax | `EMAIL-VAL-4006` | `validate-reply-to` |
| Attachment-ref must resolve | Lookup | `EMAIL-VAL-4007` | Always |
| Template must exist at startup | Probe | `EMAIL-VAL-4008` | `validate-template-exists` |
| Collect ALL errors before throwing | Accumulation | `EMAIL-VAL-4000` | Always |

---

## 6. WhatsApp Channel — OSS Edge Reference

### 6.1 WhatsApp Application Codes

All WhatsApp OSS codes live in `-arka-whatsapp-core` in `net..arka.whatsapp.code`.

#### `WhatsAppCoreCode` — `WA-CORE-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-CORE-1000` | `SEND_STARTED` | Send workflow started | Observability-only |
| `WA-CORE-1001` | `SEND_COMPLETED` | Send workflow completed | Observability-only |
| `WA-CORE-1002` | `SEND_FAILED` | Send workflow failed | Observability-only |

#### `WhatsAppValidationCode` — `WA-VAL-*`

Each carries a `ConfigCategory`.

| Code | Constant | Default message | Category |
|---|---|---|---|
| `WA-VAL-0000` | `FRAMEWORK_VALIDATION` | Framework validation error | `FLOW` |
| `WA-VAL-4000` | `FLOW_KEY_MISSING` | Flow key is missing | `FLOW` |
| `WA-VAL-4001` | `FLOW_TEMPLATE_REF_MISSING` | Flow template reference is missing | `TEMPLATE` |
| `WA-VAL-4002` | `FLOW_SENDER_REF_MISSING` | Flow sender reference is missing | `SENDER` |
| `WA-VAL-4003` | `SENDER_REF_UNRESOLVED` | Sender reference could not be resolved | `SENDER` |
| `WA-VAL-4004` | `TEMPLATE_REF_UNRESOLVED` | Template reference could not be resolved | `TEMPLATE` |
| `WA-VAL-4005` | `MEDIA_REF_UNRESOLVED` | Media reference could not be resolved | `MEDIA` |
| `WA-VAL-4006` | `PHONE_NUMBER_ID_MISSING` | Sender phone-number-id is missing | `SENDER` |
| `WA-VAL-4007` | `LANGUAGE_CODE_MISSING` | Template language code is missing | `TEMPLATE` |
| `WA-VAL-4008` | `DELIVERY_MODE_MISSING` | Flow delivery-mode is missing | `DELIVERY_MODE` |
| `WA-VAL-4009` | `DELIVERY_MODE_TEMPLATE_NO_REF` | Delivery mode requires template but no template-ref | `DELIVERY_MODE` |
| `WA-VAL-4010` | `META_TEMPLATE_NAME_MISSING` | Meta template name is missing in template definition | `TEMPLATE` |

#### `WhatsAppDestinationCode` — `WA-DST-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-DST-4000` | `DESTINATION_INVALID` | Destination phone number is invalid | Response-producing |
| `WA-DST-4001` | `DESTINATION_NOT_E164` | Destination is not in E.164 format | Response-producing |
| `WA-DST-4002` | `DESTINATION_OPT_IN_MISSING` | Destination has not opted in | Response-producing |
| `WA-DST-4003` | `DESTINATION_BLOCKED` | Destination is on the blocked number list | Response-producing |
| `WA-DST-4004` | `NO_VALID_DESTINATION` | No valid destination after policy resolution | Response-producing |
| `WA-DST-4005` | `ALL_DESTINATIONS_FILTERED` | All destinations were filtered by policy | Response-producing |

#### `WhatsAppProviderCode` — `WA-PROV-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-PROV-4000` | `FLOW_NOT_FOUND_YAML` | Flow not found in YAML provider | Observability-only when fallback exists |
| `WA-PROV-4001` | `FLOW_NOT_FOUND_DB` | Flow not found in DB provider | Observability-only when fallback exists |
| `WA-PROV-4002` | `FLOW_NOT_FOUND` | Flow not found in any provider | Response-producing |

#### `WhatsAppTransportCode` — `WA-TRN-*`

| Code | Constant | Default message | Retriable | HTTP |
|---|---|---|---|---|
| `WA-TRN-5020` | `API_SEND_FAILED` | Meta API send failed | Configurable | 502 |
| `WA-TRN-5031` | `API_UNAVAILABLE` | Meta API unavailable — circuit breaker open | No | 503 |
| `WA-TRN-5040` | `API_TIMEOUT` | Meta API request timed out | Yes | 504 |
| `WA-TRN-5043` | `API_AUTH_FAILED` | Meta API authentication failed | Never | 502 |
| `WA-TRN-5044` | `API_TLS_FAILED` | Meta API TLS handshake failed | Never | 502 |
| `WA-TRN-5045` | `API_CONNECTION_REFUSED` | Meta API connection refused | Yes | 502 |
| `WA-TRN-5046` | `API_SESSION_INIT_FAILED` | Meta API client initialization failed | Never | 502 |
| `WA-TRN-5050` | `API_RATE_LIMITED` | Meta API rate limit exceeded | Yes | 429 |
| `WA-TRN-5051` | `API_INVALID_PHONE` | Meta API rejected — invalid phone number | Never | 502 |
| `WA-TRN-5052` | `API_TEMPLATE_NOT_FOUND` | Meta API rejected — template not found | Never | 502 |
| `WA-TRN-5053` | `API_SESSION_EXPIRED` | Meta API rejected — session window expired | Never | 502 |

#### `WhatsAppTemplateCode` — `WA-TPL-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-TPL-4000` | `TEMPLATE_NOT_APPROVED` | Meta template is not in APPROVED status | Response-producing |
| `WA-TPL-4001` | `TEMPLATE_LANGUAGE_MISMATCH` | Template language code does not match | Response-producing |
| `WA-TPL-5000` | `VARIABLE_BINDING_FAILED` | Template variable binding failed | Response-producing |
| `WA-TPL-5001` | `TEMPLATE_RENDER_FAILED` | Template rendering failed | Response-producing |

#### `WhatsAppMessageCode` — `WA-MSG-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-MSG-5000` | `MESSAGE_BUILD_FAILED` | Message construction failed | Response-producing |
| `WA-MSG-5001` | `MEDIA_BINDING_FAILED` | Media attachment binding failed | Response-producing |
| `WA-MSG-5002` | `SENDER_RESOLUTION_FAILED` | Sender resolution failed | Response-producing |
| `WA-MSG-5003` | `INTERACTIVE_BUILD_FAILED` | Interactive message component build failed | Response-producing |

#### `WhatsAppBatchCode` — `WA-BATCH-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-BATCH-1000` | `BATCH_STARTED` | Batch dispatch started | Observability-only |
| `WA-BATCH-1001` | `BATCH_COMPLETED` | Batch dispatch completed | Observability-only |
| `WA-BATCH-1002` | `BATCH_PARTIAL` | Batch dispatch completed with partial success | Response-producing |
| `WA-BATCH-4000` | `BATCH_EMPTY` | Batch destination list is empty | Response-producing |
| `WA-BATCH-4001` | `BATCH_NULL` | Batch destination list is null | Response-producing |
| `WA-BATCH-5001` | `RECIPIENT_FAILED` | Per-recipient send failed | Response-producing |

#### `WhatsAppAsyncCode` — `WA-ASYNC-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-ASYNC-5000` | `DISPATCH_REJECTED` | Dispatch rejected by executor | Response-producing |
| `WA-ASYNC-5001` | `TASK_INTERRUPTED` | Task interrupted | Response-producing |
| `WA-ASYNC-5002` | `TASK_CANCELLED` | Task cancelled | Response-producing |
| `WA-ASYNC-5040` | `SINGLE_SEND_TIMEOUT` | Single send timed out | Response-producing |
| `WA-ASYNC-5041` | `BATCH_TIMEOUT` | Batch dispatch timed out | Response-producing |
| `WA-ASYNC-5042` | `RECIPIENT_TIMEOUT` | Recipient task timed out | Response-producing |

#### `WhatsAppPersistenceCode` — `WA-PERSIST-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-PERSIST-5000` | `DB_READ_FAILED` | DB read failed | Response-producing |
| `WA-PERSIST-5001` | `DB_READ_TIMEOUT` | DB read timed out | Response-producing |
| `WA-PERSIST-5002` | `MAPPER_FAILED` | Mapper conversion failed | Response-producing |
| `WA-PERSIST-5003` | `DB_UNAVAILABLE` | DB unavailable | Response-producing |
| `WA-PERSIST-5004` | `DB_DATA_INCONSISTENT` | DB data inconsistent | Response-producing |

#### `WhatsAppWindowCode` — `WA-WINDOW-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-WINDOW-1000` | `WINDOW_ACTIVE` | 24-hour session window is active | Observability-only |
| `WA-WINDOW-1001` | `WINDOW_EXPIRED` | 24-hour session window has expired | Observability-only |
| `WA-WINDOW-5000` | `WINDOW_CHECK_FAILED` | Conversation window check failed | Response-producing |
| `WA-WINDOW-5001` | `WINDOW_DATA_UNAVAILABLE` | No inbound message data available | Response-producing |

#### `WhatsAppWebhookCode` — `WA-WEBHOOK-*`

| Code | Constant | Default message | Classification |
|---|---|---|---|
| `WA-WEBHOOK-1000` | `WEBHOOK_RECEIVED` | Webhook event received | Observability-only |
| `WA-WEBHOOK-1001` | `STATUS_DELIVERED` | Message delivery confirmed | Observability-only |
| `WA-WEBHOOK-1002` | `STATUS_READ` | Message read confirmed | Observability-only |
| `WA-WEBHOOK-1003` | `STATUS_FAILED` | Message delivery failed | Response-producing |
| `WA-WEBHOOK-4000` | `SIGNATURE_INVALID` | Webhook signature verification failed | Response-producing |
| `WA-WEBHOOK-4001` | `PAYLOAD_MALFORMED` | Webhook payload could not be parsed | Response-producing |

#### WhatsApp `ConfigCategory`

| Constant | Scope |
|---|---|
| `FLOW` | Flow definition: flow key, delivery mode |
| `SENDER` | Sender identity: phone-number-id, business-account-id |
| `TEMPLATE` | Template: template-ref, meta-template-name, language-code |
| `MEDIA` | Media: media-ref |
| `DESTINATION` | Destination phone number |
| `DELIVERY_MODE` | Delivery mode: template vs session strategy |

#### WhatsApp code count — all families

| Family | Prefix | Codes |
|---|---|---|
| Core | `WA-CORE-*` | 3 |
| Validation | `WA-VAL-*` | 12 |
| Destination | `WA-DST-*` | 6 |
| Provider | `WA-PROV-*` | 3 |
| Transport | `WA-TRN-*` | 11 |
| Template | `WA-TPL-*` | 4 |
| Message | `WA-MSG-*` | 4 |
| Batch | `WA-BATCH-*` | 6 |
| Async | `WA-ASYNC-*` | 6 |
| Persistence | `WA-PERSIST-*` | 5 |
| Window | `WA-WINDOW-*` | 4 |
| Webhook | `WA-WEBHOOK-*` | 6 |
| **Total** | | **70** |

### 6.2 WhatsApp Exception Hierarchy

```
WhatsAppApplicationException extends ArkaApplicationException
│
├── WhatsAppConfigurationException extends ArkaConfigurationException
├── WhatsAppTransportException extends ArkaTransportException
│   └── WhatsAppTransportTimeoutException extends ArkaTransportTimeoutException
├── UnknownWhatsAppFlowException extends UnknownFlowException
├── WhatsAppTemplateException
├── WhatsAppSessionExpiredException
├── WhatsAppAsyncTimeoutException extends ArkaAsyncTimeoutException
├── WhatsAppTaskInterruptedException extends ArkaTaskInterruptedException
├── WhatsAppTaskCancelledException extends ArkaTaskCancelledException
└── BatchDispatcherUnavailableException extends ArkaApplicationException
```

### 6.3 WhatsApp Per-Module Reference

#### `-arka-whatsapp-core`

**Role:** WhatsApp contracts, models, WA-* codes, exception hierarchy, all adapter beans. Includes `WhatsAppConversationWindowService` (WhatsApp-only, not in shared core).

**Adapter beans provided:**

| Shared engine | WhatsApp adapter |
|---|---|
| `service` | `WhatsAppFlowService` |
| `validation` | `WhatsAppFlowValidator` |
| `provider-*` | `WhatsAppFlowConfigMapper` |
| `retry-fixed` | `WhatsAppRetryFailureClassifier` |
| `observability` | `WhatsAppObservationConvention` |
| `health` | `MetaTransportHealthContributor`, `WhatsAppReadinessGate` |
| `security` | `WhatsAppAuthorizationPolicy` |
| `outbox` | `WhatsAppOutboxSerializer`, `WhatsAppOutboxDispatcher` |
| `dlq` | `WhatsAppDlqPayloadHandler` |
| `scheduler` | `WhatsAppSchedulerPayloadHandler` |
| `client` | `WhatsAppChannelClientDescriptor` |

#### `-arka-whatsapp-transport-meta`

**Role:** Meta Cloud API send transport with typed exception translation.

**Exception translation table:**

| Root cause | Code | Exception type |
|---|---|---|
| `SocketTimeoutException` | `WA-TRN-5040` | `WhatsAppTransportTimeoutException` |
| `ConnectException` | `WA-TRN-5045` | `WhatsAppTransportException` |
| `HttpClientErrorException` (401) | `WA-TRN-5043` | `WhatsAppTransportException` |
| `HttpClientErrorException` (429) | `WA-TRN-5050` | `WhatsAppTransportException` |
| Meta error 131026 (session expired) | `WA-TRN-5053` | `WhatsAppTransportException` |
| Meta error 132000 (template not found) | `WA-TRN-5052` | `WhatsAppTransportException` |
| `HttpServerErrorException` (5xx) | `WA-TRN-5020` | `WhatsAppTransportException` |

**Hard prerequisite:** `arka.whatsapp.transport.access-token` must be configured. Startup fails if missing.

#### `-arka-whatsapp-template-meta`

**Role:** Meta Template Management API — sync, approval check, category-aware rate limit awareness.

**Business rules:**

| Rule | Enforcement |
|---|---|
| Sync templates from Meta at startup | `sync-on-startup` (default: true) |
| Template must be APPROVED before send | `fail-on-unapproved` (default: true) |
| Category-aware rate limits | UTILITY, MARKETING, AUTHENTICATION |
| Local cache with TTL | `cache-ttl` (default: 15m) |

**Configuration:**

```yaml
arka:
  whatsapp:
    template:
      sync-on-startup: true
      sync-interval: 1h
      fail-on-unapproved: true
      cache-ttl: 15m
```

### 6.4 WhatsApp Validation Rules

| Rule | Check | Code | Always? |
|---|---|---|---|
| Flow must have sender-ref | Null/blank | `WA-VAL-4002` | Yes |
| Sender must have phone-number-id | Null/blank | `WA-VAL-4006` | Yes |
| Flow must have delivery-mode | Null | `WA-VAL-4008` | Yes |
| Template-ref required for template modes | Delivery mode check | `WA-VAL-4009` | Yes |
| Meta template name present | Null/blank | `WA-VAL-4010` | Yes |
| Template language code present | Null/blank | `WA-VAL-4007` | Yes |
| Collect ALL errors before throwing | Accumulation | — | Yes |

### 6.5 WhatsApp-Specific Concepts

| Concept | Description |
|---|---|
| 24-hour conversation window | Determines template vs session message type |
| Delivery modes | `TEMPLATE_ONLY`, `SESSION_ONLY`, `TEMPLATE_PREFERRED_WITH_SESSION_FALLBACK`, `SESSION_PREFERRED_WITH_TEMPLATE_FALLBACK` |
| Meta-managed templates | Pre-registered, APPROVED status required, category-aware |
| E.164 phone number | International phone number format required for all destinations |
| `WA-WINDOW-*` | Separate code family (not merged into `WA-CORE-*` or `WA-TRN-*`) |

---

## 7. Slack Channel — OSS Edge Reference (Reserved)

Slack edge modules are reserved. Package patterns and artifact names are locked. Implementation is deferred until business priority.

### 7.1 Slack Application Codes (Reserved Pattern)

| Family | Prefix | Domain |
|---|---|---|
| Core | `SLACK-CORE-*` | Service lifecycle |
| Validation | `SLACK-VAL-*` | Startup validation |
| Channel | `SLACK-CH-*` | Channel/DM routing |
| Transport | `SLACK-TRN-*` | Webhook/API transport |
| Message | `SLACK-MSG-*` | Block Kit message construction |
| Webhook | `SLACK-WEBHOOK-*` | Event API callbacks |

### 7.2 Slack Exception Hierarchy (Reserved)

```
SlackApplicationException extends ArkaApplicationException
├── SlackConfigurationException extends ArkaConfigurationException
├── SlackTransportException extends ArkaTransportException
│   └── SlackTransportTimeoutException extends ArkaTransportTimeoutException
├── UnknownSlackFlowException extends UnknownFlowException
└── SlackMessageException
```

### 7.3 Slack Per-Module Reference (Reserved)

#### `-arka-slack-core`

**Role:** Slack contracts, models, SLACK-* codes, exceptions, adapter beans.

**Adapter beans (will provide):**

| Shared engine | Slack adapter |
|---|---|
| `service` | `SlackFlowService` |
| `validation` | `SlackFlowValidator` |
| `provider-*` | `SlackFlowConfigMapper` |
| `retry-fixed` | `SlackRetryFailureClassifier` |
| `observability` | `SlackObservationConvention` |
| `health` | `SlackTransportHealthContributor`, `SlackReadinessGate` |
| `security` | `SlackAuthorizationPolicy` |
| `outbox` | `SlackOutboxSerializer`, `SlackOutboxDispatcher` |
| `dlq` | `SlackDlqPayloadHandler` |
| `scheduler` | `SlackSchedulerPayloadHandler` |
| `client` | `SlackChannelClientDescriptor` |

#### `-arka-slack-transport-webhook`

**Role:** Slack Incoming Webhooks transport. Simple HTTPS POST to webhook URL.

**Hard prerequisite:** Webhook URL configured.

#### `-arka-slack-transport-api`

**Role:** Slack Web API transport (chat.postMessage). Supports channel routing, DM routing, thread replies.

**Hard prerequisite:** Slack Bot Token configured.

#### `-arka-slack-message-blockkit`

**Role:** Slack Block Kit message construction. Sections, actions, context, dividers, images, buttons.

### 7.4 Slack-Specific Concepts

| Concept | Description |
|---|---|
| Block Kit | Structured message format with sections, actions, context blocks |
| Channel routing | Messages routed to channels (#general) or DMs (@user) |
| Thread replies | Messages can be sent as thread replies to existing messages |
| Bot Token | OAuth-based authentication for Web API transport |
| Webhook URL | Pre-configured URL for Incoming Webhooks transport |
| Rate limiting | Slack enforces per-workspace and per-method rate limits |

---

## 8. SMS Channel — OSS Edge Reference (Reserved)

SMS edge modules are reserved. Package patterns and artifact names are locked. Implementation is deferred until business priority.

### 8.1 SMS Application Codes (Reserved Pattern)

| Family | Prefix | Domain |
|---|---|---|
| Core | `SMS-CORE-*` | Service lifecycle |
| Validation | `SMS-VAL-*` | Startup validation |
| Destination | `SMS-DST-*` | Phone number validation |
| Transport | `SMS-TRN-*` | Twilio/SNS transport |
| Message | `SMS-MSG-*` | Segment splitting, encoding |
| Status | `SMS-STATUS-*` | Delivery status callbacks |

### 8.2 SMS Exception Hierarchy (Reserved)

```
SmsApplicationException extends ArkaApplicationException
├── SmsConfigurationException extends ArkaConfigurationException
├── SmsTransportException extends ArkaTransportException
│   └── SmsTransportTimeoutException extends ArkaTransportTimeoutException
├── UnknownSmsFlowException extends UnknownFlowException
└── SmsSegmentException
```

### 8.3 SMS Per-Module Reference (Reserved)

#### `-arka-sms-core`

**Role:** SMS contracts, models, SMS-* codes, exceptions, adapter beans.

**Adapter beans (will provide):**

| Shared engine | SMS adapter |
|---|---|
| `service` | `SmsFlowService` |
| `validation` | `SmsFlowValidator` |
| `provider-*` | `SmsFlowConfigMapper` |
| `retry-fixed` | `SmsRetryFailureClassifier` |
| `observability` | `SmsObservationConvention` |
| `health` | `SmsTransportHealthContributor`, `SmsReadinessGate` |
| `security` | `SmsAuthorizationPolicy` |
| `outbox` | `SmsOutboxSerializer`, `SmsOutboxDispatcher` |
| `dlq` | `SmsDlqPayloadHandler` |
| `scheduler` | `SmsSchedulerPayloadHandler` |
| `client` | `SmsChannelClientDescriptor` |

#### `-arka-sms-transport-twilio`

**Role:** Twilio SMS transport. Sends via Twilio Messages API.

**Hard prerequisite:** Twilio Account SID + Auth Token configured.

**Configuration:**

```yaml
arka:
  sms:
    transport:
      twilio:
        account-sid: ${TWILIO_ACCOUNT_SID}
        auth-token: ${TWILIO_AUTH_TOKEN}
        from-number: ${TWILIO_FROM_NUMBER}
```

#### `-arka-sms-transport-sns`

**Role:** AWS SNS SMS transport. Sends via AWS SNS Publish API.

**Hard prerequisite:** AWS credentials + region configured.

**Configuration:**

```yaml
arka:
  sms:
    transport:
      sns:
        region: us-east-1
        sender-id: 
        message-type: Transactional
```

#### `-arka-sms-message-segment`

**Role:** SMS segment splitting and encoding. Handles GSM-7 vs UCS-2 encoding, segment counting, multipart concatenation.

### 8.4 SMS-Specific Concepts

| Concept | Description |
|---|---|
| E.164 phone number | International format required (shared validation with WhatsApp) |
| Segment splitting | Messages over 160 chars (GSM-7) or 70 chars (UCS-2) are split into segments |
| GSM-7 vs UCS-2 | Character encoding affects segment size |
| Sender ID | Alphanumeric sender identification (country-dependent) |
| Opt-in/opt-out | Regulatory compliance for SMS marketing |
| Delivery receipts | Carrier-level delivery confirmations |

---

## 9. Cross-Channel Validation Layer Matrix

Five layers, each with exactly one responsibility. No layer duplicates another. Applied identically across all channels.

| Layer | Scope | Trigger | Module |
|---|---|---|---|
| 1. HTTP shape | DTO presence/blank | Per HTTP request | web (PaaS) |
| 2. Command invariants | Non-null flowKey, defensive copies | At construction | Channel core |
| 3. Startup config | Flow config correctness | Application startup | validation |
| 4. Destination policy | Parse, validate, normalize | Per send | Channel-specific |
| 5. Send invariants | At least one valid destination | Pre-dispatch | service |

**Critical rules:**

- Startup validation collects ALL errors before throwing
- The empty-destination guard belongs in service (layer 5), not in destination handler (layer 4)
- `@Email` / `@Pattern` must NOT appear on config property classes — bypasses policy toggles

---

## 10. Cross-Channel Retriability Classification

In OSS, send is single-attempt — no retry loop. This classification applies when the PaaS retry module is present.

### Email retriability

| Code | Failure | Retriable | Reason |
|---|---|---|---|
| `EMAIL-TRN-5040` | SMTP timeout | Yes | Transient |
| `EMAIL-TRN-5045` | Connection refused | Yes | Transient |
| `EMAIL-TRN-5047` | Stale connection | Yes | Transient |
| `EMAIL-TRN-5020` | Generic SMTP | Configurable (default: no) | Avoid retrying permanent rejections |
| `EMAIL-TRN-5043` | Auth failed | Never | Permanent |
| `EMAIL-TRN-5044` | TLS failed | Never | Permanent |
| `EMAIL-TRN-5046` | Session init | Never | Permanent |
| `EMAIL-TRN-5031` | Subsystem unavailable | Never | Infrastructure |
| `EMAIL-VAL-*` | Validation | Never | Configuration |
| `EMAIL-RCP-*` | Recipient | Never | Data |
| `EMAIL-MSG-*` | Message | Never | Configuration |
| `EMAIL-TPL-*` | Template | Never | Configuration |

### WhatsApp retriability

| Code | Failure | Retriable | Reason |
|---|---|---|---|
| `WA-TRN-5040` | API timeout | Yes | Transient |
| `WA-TRN-5045` | Connection refused | Yes | Transient |
| `WA-TRN-5050` | Rate limited | Yes | With backoff |
| `WA-TRN-5020` | Generic API failure | Configurable (default: no) | Avoid retrying permanent |
| `WA-TRN-5043` | Auth failed | Never | Permanent |
| `WA-TRN-5044` | TLS failed | Never | Permanent |
| `WA-TRN-5046` | Session init | Never | Permanent |
| `WA-TRN-5031` | Circuit breaker open | Never | Infrastructure |
| `WA-TRN-5051` | Invalid phone | Never | Data |
| `WA-TRN-5052` | Template not found | Never | Configuration |
| `WA-TRN-5053` | Session expired | Never | Caller handles fallback |
| `WA-VAL-*` | Validation | Never | Configuration |
| `WA-DST-*` | Destination | Never | Data |
| `WA-MSG-*` | Message | Never | Configuration |
| `WA-TPL-*` | Template | Never | Configuration |

### Slack retriability (reserved)

| Code | Failure | Retriable | Reason |
|---|---|---|---|
| `SLACK-TRN-5040` | API timeout | Yes | Transient |
| `SLACK-TRN-5045` | Connection refused | Yes | Transient |
| `SLACK-TRN-5050` | Rate limited | Yes | With backoff |
| `SLACK-TRN-5043` | Auth failed | Never | Permanent |
| `SLACK-TRN-50xx` | Other transport | Configurable | Per-code decision |

### SMS retriability (reserved)

| Code | Failure | Retriable | Reason |
|---|---|---|---|
| `SMS-TRN-5040` | API timeout | Yes | Transient |
| `SMS-TRN-5045` | Connection refused | Yes | Transient |
| `SMS-TRN-5050` | Rate limited | Yes | With backoff |
| `SMS-TRN-5043` | Auth failed | Never | Permanent |
| `SMS-TRN-50xx` | Other transport | Configurable | Per-code decision |

---

## 11. Cross-Channel Business Rules

### Always-enforced (all channels)

| Rule | Where | What happens |
|---|---|---|
| `flowKey` not null or blank | SendCommand constructor | `NullPointerException` / `IllegalArgumentException` |
| `ApplicationCode` not null on exceptions | ArkaApplicationException constructor | `NullPointerException` |
| All collection fields defensively copied | All record constructors | `List.copyOf()` / `Map.copyOf()` |
| Null collections normalize to empty | Record compact constructors | `null -> List.of()` |
| Flow must exist | FlowProvider.resolve() | `UnknownFlowException` |
| At least one valid destination | Service pre-dispatch guard | `BusinessRuleViolationException` |
| Transport exceptions always translated | Exception translator | Typed exception with code, never raw |
| `InterruptedException` always restores flag | All catch blocks | `Thread.currentThread().interrupt()` before typed re-throw |
| Collect all validation errors before throwing | FlowValidator | Accumulate then single throw |
| No `UUID.randomUUID()` downstream | Platform-wide tracking ID rule | Missing context = `"untraced"` + WARN |
| YAML safe construction | YAML providers | `SafeConstructor` always used |

### Email-specific rules

| Rule | Where |
|---|---|
| Template + subject required per flow | Email validator |
| Sender-ref must resolve | Email provider/validator |
| Recipient dedupe: to > cc > bcc | Email recipient module |
| RecipientAction: FAIL / DROP_WARN / DROP_SILENT | Email recipient module |
| MIME construction failures translated | Email message module |

### WhatsApp-specific rules

| Rule | Where |
|---|---|
| Delivery mode required per flow | WhatsApp validator |
| Template-ref required for template modes | WhatsApp validator |
| Phone-number-id required per sender | WhatsApp validator |
| Meta template must be APPROVED | WhatsApp template-meta |
| E.164 format required for destinations | WhatsApp destination handler |
| 24-hour session window enforcement | WhatsApp conversation window service |
| SESSION_ONLY requires active window | WhatsApp message builder |

### Slack-specific rules (reserved)

| Rule | Where |
|---|---|
| Channel or DM target required | Slack validator |
| Bot token or webhook URL required | Slack transport |
| Block Kit structure validation | Slack message module |

### SMS-specific rules (reserved)

| Rule | Where |
|---|---|
| E.164 format required | SMS destination handler |
| Segment count calculated | SMS message module |
| Sender ID country compliance | SMS transport |
| Opt-in verification | SMS destination handler |

---

## 12. What OSS Core Does NOT Contain

Per the tier isolation rule, the following are NOT in any OSS core module. Each is owned by its respective PaaS or SaaS module.

### Contracts owned by PaaS modules (shared)

| Contract | Lives in shared module |
|---|---|
| `RetryPolicy` (engine) | `-arka-retry-fixed` |
| `CircuitBreaker` (engine) | `-arka-circuit-breaker` |
| Health aggregation | `-arka-health` |
| Observability engine | `-arka-observability` |
| Security framework | `-arka-security` |
| Audit engine | `-arka-audit` |
| Outbox engine | `-arka-outbox` |
| Idempotency engine | `-arka-idempotency` |
| DB provider engine | `-arka-provider-db` |
| Hybrid provider engine | `-arka-provider-hybrid` |
| REST API surface | `-arka-web` |
| Web console shell | `-arka-web-console` |
| Typed client | `-arka-client` |

### Contracts owned by SaaS modules (shared)

| Contract | Lives in shared module |
|---|---|
| Tenant engine | `-arka-tenant` |
| DLQ engine | `-arka-dlq` |
| Scheduler engine | `-arka-scheduler` |

### Channel-specific PaaS contracts

| Contract | Lives in channel module |
|---|---|
| WhatsApp webhook status | `-arka-whatsapp-webhook-status` |
| WhatsApp console screens | `-arka-whatsapp-web-console` |
| Slack console screens | `-arka-slack-web-console` (reserved) |
| SMS webhook status | `-arka-sms-webhook-status` (reserved) |

---

## 13. Starter Composition

| Channel | Starter | Shared OSS | Channel OSS edge | Total |
|---|---|---|---|---|
| Email | `-arka-email-starter` | core, service, validation, provider-yaml | email-core, email-transport-javamail, email-recipient, email-message, email-template-thymeleaf | 9 |
| WhatsApp | `-arka-whatsapp-starter` | core, service, validation, provider-yaml | whatsapp-core, whatsapp-transport-meta, whatsapp-template-meta | 7 |
| Slack | `-arka-slack-starter` | core, service, validation, provider-yaml | slack-core, slack-transport-webhook, slack-transport-api, slack-message-blockkit | 8 |
| SMS | `-arka-sms-starter` | core, service, validation, provider-yaml | sms-core, sms-transport-twilio, sms-transport-sns, sms-message-segment | 8 |

Enterprise starters (PaaS):

| Channel | Starter | Adds |
|---|---|---|
| Email | `-arka-email-starter-enterprise` | OSS starter + all shared PaaS modules |
| WhatsApp | `-arka-whatsapp-starter-enterprise` | OSS starter + all shared PaaS modules + WhatsApp PaaS edge |
| Slack | `-arka-slack-starter-enterprise` | OSS starter + all shared PaaS modules + Slack PaaS edge |
| SMS | `-arka-sms-starter-enterprise` | OSS starter + all shared PaaS modules + SMS PaaS edge |

Full starters (SaaS):

| Channel | Starter | Adds |
|---|---|---|
| Email | `-arka-email-starter-full` | Enterprise starter + all shared SaaS modules |
| WhatsApp | `-arka-whatsapp-starter-full` | Enterprise starter + all shared SaaS modules |
| Slack | `-arka-slack-starter-full` | Enterprise starter + all shared SaaS modules |
| SMS | `-arka-sms-starter-full` | Enterprise starter + all shared SaaS modules |

---

## 14. Maven Coordinates

All artifacts share GroupId `net..arka`.

### Shared Engine — OSS (4 capability modules)

| ArtifactId | Java Package | Role |
|---|---|---|
| `-arka-core` | `net..arka.core` | Shared SPIs, base exceptions, ApplicationCode |
| `-arka-service` | `net..arka.service` | Generic send orchestration engine |
| `-arka-validation` | `net..arka.validation` | Generic startup validation engine |
| `-arka-provider-yaml` | `net..arka.provider.yaml` | YAML-backed flow resolution engine |

### Email — OSS (5 capability + 1 starter)

| ArtifactId | Java Package | Role |
|---|---|---|
| `-arka-email-core` | `net..arka.email` | Email contracts, models, codes, exceptions, adapter beans |
| `-arka-email-transport-javamail` | `net..arka.email.transport.javamail` | SMTP transport, typed exception translation |
| `-arka-email-recipient` | `net..arka.email.recipient` | Recipient normalization, dedupe, RecipientAction policy |
| `-arka-email-message` | `net..arka.email.message` | MIME construction, attachment binding |
| `-arka-email-template-thymeleaf` | `net..arka.email.template.thymeleaf` | Thymeleaf rendering, template probe |
| `-arka-email-starter` | — | Composition: shared OSS engine + email OSS edge |

### WhatsApp — OSS (3 capability + 1 starter)

| ArtifactId | Java Package | Role |
|---|---|---|
| `-arka-whatsapp-core` | `net..arka.whatsapp` | WhatsApp contracts, models, codes, exceptions, adapter beans |
| `-arka-whatsapp-transport-meta` | `net..arka.whatsapp.transport.meta` | Meta Cloud API send transport, exception translation |
| `-arka-whatsapp-template-meta` | `net..arka.whatsapp.template.meta` | Meta Template Management API, sync, approval check |
| `-arka-whatsapp-starter` | — | Composition: shared OSS engine + whatsapp OSS edge |

### Slack — OSS Reserved (4 capability + 1 starter)

| ArtifactId | Java Package | Role | Status |
|---|---|---|---|
| `-arka-slack-core` | `net..arka.slack` | Slack contracts, models, codes, exceptions, adapter beans | Reserved |
| `-arka-slack-transport-webhook` | `net..arka.slack.transport.webhook` | Slack Incoming Webhooks transport | Reserved |
| `-arka-slack-transport-api` | `net..arka.slack.transport.api` | Slack Web API transport (chat.postMessage) | Reserved |
| `-arka-slack-message-blockkit` | `net..arka.slack.message.blockkit` | Block Kit message construction | Reserved |
| `-arka-slack-starter` | — | Composition: shared OSS engine + slack OSS edge | Reserved |

### SMS — OSS Reserved (4 capability + 1 starter)

| ArtifactId | Java Package | Role | Status |
|---|---|---|---|
| `-arka-sms-core` | `net..arka.sms` | SMS contracts, models, codes, exceptions, adapter beans | Reserved |
| `-arka-sms-transport-twilio` | `net..arka.sms.transport.twilio` | Twilio SMS transport | Reserved |
| `-arka-sms-transport-sns` | `net..arka.sms.transport.sns` | AWS SNS SMS transport | Reserved |
| `-arka-sms-message-segment` | `net..arka.sms.message.segment` | SMS segment splitting, GSM-7/UCS-2 encoding | Reserved |
| `-arka-sms-starter` | — | Composition: shared OSS engine + sms OSS edge | Reserved |

### OSS Artifact Totals

| Channel | Capability | Starters | Total |
|---|---|---|---|
| Shared engine | 4 | — | 4 |
| Email | 5 | 1 | 6 |
| WhatsApp | 3 | 1 | 4 |
| Slack (reserved) | 4 | 1 | 5 |
| SMS (reserved) | 4 | 1 | 5 |
| **All channels** | **20** | **4** | **24** |

---

* Arka — OSS Tier Complete Reference — All Channels — Frozen Standard — March 2026*
*Tier isolation rule: OSS core owns only OSS contracts. PaaS modules own PaaS contracts. SaaS modules own SaaS contracts.*
*All decisions in this document are locked. Changes require explicit review.*
