# SPHUTA Arka — Package and Artifact Matrix by Tier

SPHUTA Arka follows a **shared engine + channel edges** architecture.

- **Shared engine artifacts** use `sphuta-arka-*`
- **Channel edge artifacts** use `sphuta-arka-{channel}-*`
- **Channel business configuration remains channel-scoped**
  - `arka.email.*`
  - `arka.whatsapp.*`
  - `arka.slack.*`
  - `arka.sms.*`

Shared modules must be truly channel-neutral. Channel-specific transport, message format, templates, runtime surfaces, and configuration models belong in channel edge modules.

---

## 1. Shared Engine — OSS

| Module | Artifact ID | Java Package | Spring Prefix |
|---|---|---|---|
| Core | `sphuta-arka-core` | `net.sphuta.arka.core` | — |
| Service | `sphuta-arka-service` | `net.sphuta.arka.service` | `arka` |
| Validation | `sphuta-arka-validation` | `net.sphuta.arka.validation` | `arka.validation` |
| Provider YAML | `sphuta-arka-provider-yaml` | `net.sphuta.arka.provider.yaml` | `arka.provider.yaml` |

## 2. Shared Engine — PaaS

| Module | Artifact ID | Java Package | Spring Prefix |
|---|---|---|---|
| Async | `sphuta-arka-async` | `net.sphuta.arka.async` | `arka.async` |
| Retry Fixed | `sphuta-arka-retry-fixed` | `net.sphuta.arka.retry.fixed` | `arka.retry` |
| Provider DB | `sphuta-arka-provider-db` | `net.sphuta.arka.provider.db` | `arka.provider.db` |
| Provider Hybrid | `sphuta-arka-provider-hybrid` | `net.sphuta.arka.provider.hybrid` | `arka.provider.hybrid` |
| Circuit Breaker | `sphuta-arka-circuit-breaker` | `net.sphuta.arka.circuitbreaker` | `arka.circuit-breaker` |
| Observability | `sphuta-arka-observability` | `net.sphuta.arka.observability` | `arka.observability` |
| Health | `sphuta-arka-health` | `net.sphuta.arka.health` | `arka.health` |
| Security | `sphuta-arka-security` | `net.sphuta.arka.security` | `arka.security` |
| Audit | `sphuta-arka-audit` | `net.sphuta.arka.audit` | `arka.audit` |
| Outbox | `sphuta-arka-outbox` | `net.sphuta.arka.outbox` | `arka.outbox` |
| Idempotency | `sphuta-arka-idempotency` | `net.sphuta.arka.idempotency` | `arka.idempotency` |
| Web | `sphuta-arka-web` | `net.sphuta.arka.web` | `arka.web` |
| Web Console | `sphuta-arka-web-console` | `net.sphuta.arka.web.console` | `arka.console` |

## 3. Shared Engine — SaaS

| Module | Artifact ID | Java Package | Spring Prefix |
|---|---|---|---|
| Tenant | `sphuta-arka-tenant` | `net.sphuta.arka.tenant` | `arka.tenant` |
| DLQ | `sphuta-arka-dlq` | `net.sphuta.arka.dlq` | `arka.dlq` |
| Scheduler | `sphuta-arka-scheduler` | `net.sphuta.arka.scheduler` | `arka.scheduler` |
| Client | `sphuta-arka-client` | `net.sphuta.arka.client` | `arka.client` |

---

## 4. Channel Edge Matrix

### Email — Frozen Edge Set

| Tier | Module | Artifact ID | Java Package | Spring Prefix |
|---|---|---|---|---|
| OSS | Email Core | `sphuta-arka-email-core` | `net.sphuta.arka.email` | `arka.email` |
| OSS | SMTP Transport | `sphuta-arka-email-transport-javamail` | `net.sphuta.arka.email.transport.javamail` | `arka.email.transport` |
| OSS | Recipient | `sphuta-arka-email-recipient` | `net.sphuta.arka.email.recipient` | `arka.email.recipient` |
| OSS | Message | `sphuta-arka-email-message` | `net.sphuta.arka.email.message` | `arka.email.message` |
| OSS | Template Thymeleaf | `sphuta-arka-email-template-thymeleaf` | `net.sphuta.arka.email.template.thymeleaf` | `arka.email.template` |

### WhatsApp — Frozen Edge Set

| Tier | Module | Artifact ID | Java Package | Spring Prefix |
|---|---|---|---|---|
| OSS | WhatsApp Core | `sphuta-arka-whatsapp-core` | `net.sphuta.arka.whatsapp` | `arka.whatsapp` |
| OSS | Meta Transport | `sphuta-arka-whatsapp-transport-meta` | `net.sphuta.arka.whatsapp.transport.meta` | `arka.whatsapp.transport` |
| OSS | Template Meta | `sphuta-arka-whatsapp-template-meta` | `net.sphuta.arka.whatsapp.template.meta` | `arka.whatsapp.template` |
| PaaS | Webhook Status | `sphuta-arka-whatsapp-webhook-status` | `net.sphuta.arka.whatsapp.webhook.status` | `arka.whatsapp.webhook` |
| PaaS | WhatsApp Web Console | `sphuta-arka-whatsapp-web-console` | `net.sphuta.arka.whatsapp.web.console` | `arka.whatsapp.console` |

### Slack — Reserved Future Edge Pattern

| Tier | Module | Artifact ID | Java Package | Spring Prefix |
|---|---|---|---|---|
| OSS | Slack Core | `sphuta-arka-slack-core` | `net.sphuta.arka.slack` | `arka.slack` |
| OSS | Webhook Transport | `sphuta-arka-slack-transport-webhook` | `net.sphuta.arka.slack.transport.webhook` | `arka.slack.transport` |
| OSS | API Transport | `sphuta-arka-slack-transport-api` | `net.sphuta.arka.slack.transport.api` | `arka.slack.transport` |
| OSS | Block Kit Message | `sphuta-arka-slack-message-blockkit` | `net.sphuta.arka.slack.message.blockkit` | `arka.slack.message` |
| PaaS | Slack Web Console | `sphuta-arka-slack-web-console` | `net.sphuta.arka.slack.web.console` | `arka.slack.console` |

### SMS — Reserved Future Edge Pattern

| Tier | Module | Artifact ID | Java Package | Spring Prefix |
|---|---|---|---|---|
| OSS | SMS Core | `sphuta-arka-sms-core` | `net.sphuta.arka.sms` | `arka.sms` |
| OSS | Twilio Transport | `sphuta-arka-sms-transport-twilio` | `net.sphuta.arka.sms.transport.twilio` | `arka.sms.transport` |
| OSS | SNS Transport | `sphuta-arka-sms-transport-sns` | `net.sphuta.arka.sms.transport.sns` | `arka.sms.transport` |
| OSS | Segment Message | `sphuta-arka-sms-message-segment` | `net.sphuta.arka.sms.message.segment` | `arka.sms.message` |
| PaaS | SMS Webhook Status | `sphuta-arka-sms-webhook-status` | `net.sphuta.arka.sms.webhook.status` | `arka.sms.webhook` |

---

## 5. Starter Matrix

### Email Starters

| Tier | Artifact ID | Bundles |
|---|---|---|
| OSS | `sphuta-arka-email-starter` | OSS shared engine + Email OSS edge |
| PaaS | `sphuta-arka-email-starter-enterprise` | Email OSS starter + shared PaaS engine |
| SaaS | `sphuta-arka-email-starter-full` | Email enterprise starter + shared SaaS engine |

### WhatsApp Starters

| Tier | Artifact ID | Bundles |
|---|---|---|
| OSS | `sphuta-arka-whatsapp-starter` | OSS shared engine + WhatsApp OSS edge |
| PaaS | `sphuta-arka-whatsapp-starter-enterprise` | WhatsApp OSS starter + shared PaaS engine + WhatsApp PaaS edge |
| SaaS | `sphuta-arka-whatsapp-starter-full` | WhatsApp enterprise starter + shared SaaS engine |

### Slack Starters (Reserved)

| Tier | Artifact ID | Bundles |
|---|---|---|
| OSS | `sphuta-arka-slack-starter` | OSS shared engine + Slack OSS edge |
| PaaS | `sphuta-arka-slack-starter-enterprise` | Slack OSS starter + shared PaaS engine + Slack PaaS edge |
| SaaS | `sphuta-arka-slack-starter-full` | Slack enterprise starter + shared SaaS engine |

### SMS Starters (Reserved)

| Tier | Artifact ID | Bundles |
|---|---|---|
| OSS | `sphuta-arka-sms-starter` | OSS shared engine + SMS OSS edge |
| PaaS | `sphuta-arka-sms-starter-enterprise` | SMS OSS starter + shared PaaS engine + SMS PaaS edge |
| SaaS | `sphuta-arka-sms-starter-full` | SMS enterprise starter + shared SaaS engine |

---

## 6. Internal Package Templates

Standard internal package layouts aligned with the coding standards.

### Shared Core Module

```
net.sphuta.arka.core
├── api
├── spi
├── model
│   ├── flow
│   ├── enums
│   └── builder
├── code
├── exception
└── validation
```

### Channel Core Modules

```
net.sphuta.arka.{channel}
├── api
├── spi
├── model
│   ├── flow
│   ├── config
│   ├── enums
│   └── builder
├── code
├── exception
└── validation
```

### Shared Validation Module

```
net.sphuta.arka.validation
├── autoconfigure
├── properties
├── pipeline
├── rule
├── diagnostic
├── report
└── internal
```

### Shared YAML Provider Module

```
net.sphuta.arka.provider.yaml
├── autoconfigure
├── properties
├── loader
├── parser
├── mapper
├── source
└── internal
```

### Transport Modules

```
net.sphuta.arka.{channel}.transport.{provider}
├── autoconfigure
├── properties
├── translator
├── client
└── internal
```

### Message and Template Modules

```
net.sphuta.arka.{channel}.message.{variant}
net.sphuta.arka.{channel}.template.{variant}
```

### Web Console Modules

```
net.sphuta.arka.web.console
net.sphuta.arka.{channel}.web.console
```

---

## 7. Tier Summary

| Tier | Shared Engine Modules | Channel Edge Pattern |
|---|---|---|
| OSS | `core`, `service`, `validation`, `provider-yaml` | Channel core + transport / message / template edges |
| PaaS | `async`, `retry-fixed`, `provider-db`, `provider-hybrid`, `circuit-breaker`, `observability`, `health`, `security`, `audit`, `outbox`, `idempotency`, `web`, `web-console` | Channel runtime surfaces (webhooks, console) |
| SaaS | `tenant`, `dlq`, `scheduler`, `client` | Typically no new channel artifacts — SaaS value is shared engine + tenant-scoped runtime + managed operations |

---

## 8. Freeze Rule

Even with shared engine extraction, channel business configuration must remain channel-scoped:

- `arka.email.*`
- `arka.whatsapp.*`
- `arka.slack.*`
- `arka.sms.*`

Do not collapse channel flow definitions into a generic shared `arka.*` business tree. Shared engine modules stay neutral. Channel-specific configuration and semantics stay in channel edges.

---

*SPHUTA Arka — Package and Artifact Matrix — Frozen Standard — March 2026*
