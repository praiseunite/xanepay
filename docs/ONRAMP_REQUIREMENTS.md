# XanePay — Onramp Engine
## Requirements, Architecture & Scope Document

> **What is the Onramp Engine?**
> The Onramp Engine handles the flow of value entering the system —
> a user sends fiat (NGN) and receives crypto in their wallet.
> It covers everything from receiving the payment, routing to the
> best conversion provider, executing the swap, and settling crypto
> to the user's destination wallet.

---

## System Architecture

![XanePay System Architecture](../diagrams/xanepay_system_architecture.png)

*The Onramp Engine sits within the XanePay Engine layer, consuming the
Fiat PSP Layer (to receive NGN) and the Conversion Provider Layer (to
deliver crypto). The Operations Frontend connects to manage providers
and monitor transactions without code changes.*

---

## Hexagonal Architecture — Provider Independence

![Hexagonal Architecture](../diagrams/hexagonal_architecture_diagram.png)

*The Application Core has zero knowledge of any specific provider.
All providers plug in as Adapters implementing a Port interface.
Adding a new Fiat PSP or Conversion Provider requires writing one
new Adapter class and registering it from the admin panel — no
core logic changes.*

---

## Onramp Flow — UML Sequence Diagram

![Onramp Sequence Diagram](../diagrams/onramp_sequence_diagram.png)

*Full onramp lifecycle: from user deposit initiation through PSP
payment, quote routing, conversion execution, and crypto settlement.*

---

## Transaction State Machine

![Transaction State Machine](../diagrams/transaction_state_machine.png)

*Every onramp transaction follows this enforced lifecycle.
Invalid transitions are rejected at the application layer.
Every transition is logged with the full triggering payload.*

---

## Onramp Functional Requirements

### FR-ON-01 — Fiat Deposit Initiation
The engine must accept a deposit request from XaneApp and call the
active Fiat PSP to generate a payment link or virtual account number
for the user to pay into.

**Acceptance criteria:**
- Returns a unique `transactionId` and payment instruction to XaneApp
- Transaction recorded in state `PAYMENT_PENDING`
- Idempotency key enforced — duplicate requests return the same transaction

### FR-ON-02 — Payment Webhook Processing
The engine must receive, validate, and process inbound payment
confirmation webhooks from Fiat PSP providers.

**Acceptance criteria:**
- Webhook signature verified (HMAC) before any processing
- IP allowlist enforced — requests from unknown IPs rejected
- Timestamp validation — payloads older than 5 minutes rejected
- Duplicate webhook detection — same payload processed only once
- On confirmation: transaction moves to `PAYMENT_CONFIRMED`, ledger updated atomically

### FR-ON-03 — Conversion Quote
The engine must query all active conversion providers in parallel
and return the best available quote for the requested currency pair.

**Acceptance criteria:**
- All active providers queried simultaneously, not sequentially
- Quote includes: rate, `toAmount`, fee breakdown, expiry timestamp
- Quote cached in Redis for the duration of its lock window
- Expired quotes rejected — user must request a fresh quote

### FR-ON-04 — Provider Routing & Selection
The engine must apply a configurable scoring model to select the
optimal provider from all available quotes.

**Acceptance criteria:**
- Scoring weights (rate, speed, reliability, cost) configurable from admin panel
- Providers marked DEGRADED or DOWN are excluded from scoring
- Selected provider and quote recorded on the transaction

### FR-ON-05 — Conversion Execution
On user confirmation, the engine must execute the conversion with
the selected provider using the locked quote.

**Acceptance criteria:**
- Quote must not be expired at point of execution
- Provider called with destination wallet address
- Transaction moves to `CONVERSION_PROCESSING`
- Provider transaction ID recorded for tracking

### FR-ON-06 — Conversion Webhook Processing
The engine must receive and process inbound conversion result webhooks
from the conversion provider.

**Acceptance criteria:**
- Webhook signature verified before processing
- On success: transaction moves to `CONVERSION_COMPLETED` then `SETTLEMENT_PENDING`
- On failure: retry engine engaged (max attempts configurable)
- On max retries exhausted: transaction moves to `REVERSED`, ledger reversal entries created

### FR-ON-07 — Crypto Settlement
The engine must record and confirm the settlement of crypto to the
user's destination wallet address.

**Acceptance criteria:**
- On-chain transaction hash (`txHash`) recorded on the transaction
- Transaction moves to `COMPLETED`
- XaneApp notified via event/webhook
- Ledger finalized with all entries balanced

### FR-ON-08 — Ledger Integrity
Every money movement during the onramp flow must be recorded in the
double-entry ledger atomically with its corresponding state change.

**Acceptance criteria:**
- Ledger entries and state changes committed in single DB transaction
- Balances computed from ledger — no stored balance field
- Account row locked before any debit operation
- No ledger entry ever updated or deleted

---

## Onramp Non-Functional Requirements

### NFR-ON-01 — Availability
The onramp API must target 99.9% uptime.
Provider downtime must not cause system downtime — failover must be automatic.

### NFR-ON-02 — Latency
Quote response (all providers queried) must return within 3 seconds.
Deposit initiation must return a payment instruction within 2 seconds.

### NFR-ON-03 — Security
All inbound webhooks must pass the 5-step security pipeline
(IP allowlist → signature verification → timestamp validation →
idempotency check → schema validation) before any processing.

### NFR-ON-04 — Auditability
Every state transition must be logged with timestamp, triggering event,
and actor (webhook / user / system / admin).
Logs must be queryable from the operations interface without server access.

### NFR-ON-05 — Recoverability
A transaction stuck in any non-terminal state for longer than a configured
threshold must be automatically detected and recovered by background jobs,
without manual intervention.

---

## Onramp API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/v1/onramp/deposit/initiate` | Initiate fiat deposit via active PSP |
| `GET` | `/v1/onramp/quote` | Fetch best conversion quote (NGN → Crypto) |
| `POST` | `/v1/onramp/confirm` | Confirm and lock quote, begin execution |
| `GET` | `/v1/transactions/:id` | Get transaction status + full event timeline |
| `POST` | `/v1/webhooks/fiat-psp/{providerId}` | Receive PSP payment events |
| `POST` | `/v1/webhooks/conversion/{providerId}` | Receive conversion provider events |

---

## Onramp Critical Requirements (Missing from Original PRD)

These are not optional enhancements.
They are prerequisites for safe onramp operation in production.

| # | Requirement | Risk if Missing |
|---|---|---|
| 1 | Idempotency on all endpoints and webhooks | Duplicate webhook = user receives crypto without paying |
| 2 | Webhook security pipeline (HMAC + IP + timestamp) | Fake webhook = free crypto for attacker |
| 3 | Atomic ledger writes (ACID) | Partial failure = money debited but conversion never starts |
| 4 | Append-only ledger with computed balances | Balance manipulation, undetectable discrepancies |
| 5 | Row-level locking on account reads | Race condition = account overdraft |
| 6 | Dynamic Provider Registry (DB-driven) | Provider changes require code deployment and downtime |
| 7 | Stuck transaction background jobs | PSP webhook fails = transaction stuck forever |
| 8 | Secrets management (no credentials in DB) | Database breach = all provider API keys exposed |
| 9 | Transaction observability and admin resolution | No way to investigate or fix failures without server access |

---

## Onramp Deployment Scope

When the developer also takes on deployment responsibility,
the following is added to the onramp scope:

### Infrastructure Setup
- [ ] Cloud environment provisioning (VPS or managed cloud: AWS / GCP / DigitalOcean)
- [ ] PostgreSQL database provisioning and hardening (backups, access control)
- [ ] Redis instance setup (quote cache + job queue)
- [ ] Secrets Manager setup and initial credential loading
- [ ] Docker containerisation of the application

### CI/CD Pipeline
- [ ] Git repository setup with branch protection rules
- [ ] CI pipeline: lint + type-check + unit tests on every push
- [ ] CD pipeline: automated deployment to staging on merge to `develop`
- [ ] CD pipeline: gated deployment to production on merge to `main`
- [ ] Environment variable management (staging vs. production)

### Webhook Infrastructure
- [ ] Public HTTPS endpoint for receiving provider webhooks
- [ ] SSL certificate provisioning and renewal (Let's Encrypt or ACM)
- [ ] Webhook URL registration with each provider (PSP + conversion)

### Monitoring & Alerting
- [ ] Application error monitoring (Sentry or equivalent)
- [ ] Uptime monitoring with alerting
- [ ] Log aggregation setup
- [ ] Alerts: no successful transactions in 15 min, provider degraded, DLQ overflow

### Post-Launch
- [ ] Sandbox to production cutover coordination
- [ ] Provider sandbox → production credential rotation
- [ ] First live transaction monitoring (on-call availability)
- [ ] Documented runbook for common operational issues

---

## Onramp Deliverables Summary

| Deliverable | Description |
|---|---|
| Fiat PSP Adapter(s) | Integration with active Fiat PSP(s) — deposit initiation, webhook processing, status polling |
| Conversion Provider Adapter(s) | Integration with active Conversion Provider(s) — quote, execute, webhook, status polling |
| Routing Engine | Parallel quote fetching + configurable scoring model |
| Transaction State Machine | Full onramp lifecycle with all transitions and failure paths |
| Double-Entry Ledger | Atomic, append-only, ACID-compliant financial ledger |
| Webhook Receiver | Secured endpoint with 5-step validation pipeline |
| Background Jobs | Stuck payment scanner, stuck conversion scanner |
| Onramp REST API | All endpoints for XaneApp to consume |
| Dynamic Provider Registry | DB-driven provider config — add/configure from admin panel |
| Transaction Observability API | Full event timeline per transaction; admin resolution actions |
| Secrets Management | Provider credentials in secrets manager, referenced not stored |
| Deployment (if in scope) | Full infra setup, CI/CD, monitoring, go-live support |

---

## Onramp Pricing

### Development Only (No Deployment)

| Scope | Estimated Time | Fair Price (USD) | Fair Price (NGN) |
|---|---|---|---|
| 1 PSP + 1 Conversion Provider (basic routing) | 6–8 weeks | $8,000 – $12,000 | ₦6M – ₦9M |
| 2 PSPs + 2 Conversion Providers + Dynamic Registry | 10–12 weeks | $15,000 – $20,000 | ₦11M – ₦15M |
| Full onramp (all features in this document) | 12–14 weeks | $18,000 – $25,000 | ₦13M – ₦18M |

### Development + Deployment

| Scope | Estimated Time | Fair Price (USD) | Fair Price (NGN) |
|---|---|---|---|
| Basic onramp + full infra setup + CI/CD | 10–12 weeks | $13,000 – $18,000 | ₦9.5M – ₦13M |
| Full onramp + deployment + monitoring setup | 14–16 weeks | $22,000 – $32,000 | ₦16M – ₦23M |

### Monthly Retainer (Ongoing Development + Maintenance)

| Engagement | Monthly Rate (USD) | Monthly Rate (NGN) |
|---|---|---|
| Development only | $3,000 – $5,000/mo | ₦2.2M – ₦3.7M/mo |
| Development + deployment + on-call | $4,500 – $7,000/mo | ₦3.3M – ₦5.1M/mo |

> **Note on $100 offer:** The onramp engine as described here is
> a minimum of 6 weeks of senior full-stack + blockchain engineering work.
> $100 is approximately 0.8% of the fair minimum price for this scope.
> It does not reflect the complexity, the financial risk, or the
> level of expertise required to build this safely.

---

*This document covers the XanePay Onramp Engine scope only.*
*For Offramp scope and pricing, see: `docs/OFFRAMP_REQUIREMENTS.md`*
*For combined pricing, see: `docs/PRICING_SUMMARY.md`*
