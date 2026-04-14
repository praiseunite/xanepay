# XanePay Onramp & Offramp Engine — Implementation Plan
## MVP → Scalable Delivery Roadmap

> **Note:** This plan covers the onramp/offramp engine scope only.
> User authentication, frontend UI, and wallet management are separate XaneApp concerns.
> Full technical architecture is documented in [`docs/PRD.md`](./PRD.md).

---

## Delivery Philosophy

This engine is built in **phases**, where each phase produces a working, deployable system.
No phase requires ripping out the previous one — each layer builds on the last.

The architecture uses **Hexagonal (Ports & Adapters)** throughout, meaning:
- Adding providers never touches business logic
- Each phase can be tested in isolation with mock adapters
- The system is production-safe from Phase 1 onwards

---

## Phase 0 — Foundation
**Goal:** Project skeleton, contracts, and database ready. No external provider calls yet.

### Deliverables
- [ ] Project structure scaffolded (TypeScript / Node.js)
- [ ] All domain entities defined (`Transaction`, `LedgerEntry`, `Quote`, `Provider`)
- [ ] All port interfaces defined (`IFiatPaymentPort`, `ICryptoConversionPort`, `ILedgerPort`, `ITransactionRepository`, `IProviderRegistry`)
- [ ] All transaction status enums defined (`TransactionStatus`, `TransactionType`, `Currency`)
- [ ] PostgreSQL database schema created and migrated
  - `providers` table (dynamic registry)
  - `accounts` table (chart of accounts)
  - `ledger_entries` table (append-only, double-entry)
  - `transactions` table (state machine + idempotency key)
  - `transaction_events` table (full audit trail)
- [ ] Redis configured (quote caching + idempotency key store)
- [ ] BullMQ configured (background job queue)
- [ ] Docker Compose environment (Postgres + Redis + App)
- [ ] Base test framework configured (unit + integration)
- [ ] CI pipeline skeleton (lint + type-check + test on push)

**Exit Criteria:** `docker-compose up` starts a healthy app. Database migrates clean.

---

## Phase 1 — Core Engine (Business Logic)
**Goal:** The financial engine works correctly with mock adapters. No real providers needed yet.

### Deliverables
- [ ] **Ledger Service**
  - Double-entry write (`recordDoubleEntry`) wrapped in DB transaction
  - Balance calculation by account (`getBalance`)
  - Atomic: ledger entries + state change commit together or both roll back
  - Row-level locking on account reads (`SELECT FOR UPDATE`)
- [ ] **Transaction State Machine**
  - All valid state transitions implemented
  - Invalid transitions rejected (throw `InvalidTransitionError`)
  - Every transition appends to `transaction_events`
- [ ] **Routing Service (Mock)**
  - Reads active providers from `providers` table
  - Calls `getQuote()` on all active providers in parallel
  - Applies scoring model (weights from provider registry)
  - Returns selected provider and best quote
- [ ] **Retry Service**
  - Exponential backoff (configurable: delays, max attempts)
  - Classifies errors as retryable vs. terminal
  - On exhaustion: triggers DLQ + notification
- [ ] **Transaction Repository**
  - Create, read, update transaction records
  - Idempotency key enforcement (DB unique constraint + service check)
  - Query: find transactions by status, user, provider
  - Query: find "stuck" transactions (stale in non-terminal state > threshold)
- [ ] **Provider Registry Service**
  - Read active providers by type from DB
  - Health status read/write
  - Routing weight retrieval

**Exit Criteria:** Full onramp + offramp flow tested end-to-end with mock adapters. All money moves correctly through the ledger. Invalid operations rejected.

---

## Phase 2 — First Fiat PSP Integration
**Goal:** Real fiat deposits work. NGN enters the system from an actual payment provider.

### Deliverables
- [ ] **Fiat PSP A Adapter** (implements `IFiatPaymentPort`)
  - `initiateDeposit()` — creates payment link or virtual account
  - `verifyWebhookSignature()` — HMAC signature validation
  - `parseWebhookEvent()` — normalize raw webhook to standard event
  - `getTransactionStatus()` — polling fallback if webhook never arrives
- [ ] **Webhook Receiver Endpoint** (`POST /v1/webhooks/fiat-psp/{providerId}`)
  - IP allowlist enforcement
  - Signature verification (step 1 — reject before any processing)
  - Timestamp validation (replay attack prevention)
  - Idempotency check (payload hash — reject duplicates)
  - Schema validation
  - Route to `ConfirmOnrampUseCase`
- [ ] **Deposit Initiation Endpoint** (`POST /v1/onramp/deposit/initiate`)
  - Idempotency-Key header required
  - KYC pre-check gate
  - Returns: payment URL / virtual account / reference
- [ ] **StuckTransactionJob** (first version)
  - Scans for transactions in `PAYMENT_PENDING` older than TTL
  - Polls PSP for status
  - Updates state machine based on result
- [ ] Sandbox integration test: full deposit webhook flow

**Exit Criteria:** User initiates deposit → receives virtual account → sends money → webhook fires → ledger credits user balance. All verified in sandbox.

---

## Phase 3 — First Conversion Provider Integration
**Goal:** Fiat-to-crypto and crypto-to-fiat flows work end-to-end with a real conversion provider.

### Deliverables
- [ ] **Conversion Provider A Adapter** (implements `ICryptoConversionPort`)
  - `getQuote()` — fetch rate-locked quote
  - `executeQuote()` — execute conversion with destination details
  - `getTransactionStatus()` — polling fallback
  - `verifyWebhookSignature()` + `parseWebhookEvent()`
- [ ] **Quote Caching** (Redis)
  - Cache quote by `quoteId` with TTL matching `expiresAt`
  - Return cached quote on confirm — no second provider call
  - Reject confirmation if quote expired
- [ ] **Onramp Conversion Endpoints**
  - `GET /v1/onramp/quote` — fetch + cache quote
  - `POST /v1/onramp/confirm` — lock quote, begin execution
- [ ] **Offramp Endpoints**
  - `POST /v1/offramp/initiate` — get deposit address for crypto
  - `GET /v1/offramp/quote`
  - `POST /v1/offramp/confirm`
  - `POST /v1/offramp/cashout`
- [ ] **Conversion Webhook Receiver** (`POST /v1/webhooks/conversion/{providerId}`)
  - Same security pipeline as PSP webhooks
  - Routes to appropriate use case based on event type
- [ ] **StuckTransactionJob** (conversion states)
  - Scan for transactions stuck in `CONVERSION_PROCESSING` > threshold
  - Poll conversion provider for status
  - Advance or fail state machine accordingly
- [ ] Sandbox integration test: full NGN → Crypto flow
- [ ] Sandbox integration test: full Crypto → NGN flow

**Exit Criteria:** Full onramp (NGN → ETH) and offramp (ETH → NGN) work end-to-end in sandbox. Ledger balanced at every step.

---

## Phase 4 — Operations API & Admin Capabilities
**Goal:** All transaction management and provider configuration possible from an API (no server access needed).

### Deliverables
- [ ] **Provider Management API**
  - `GET /admin/providers` — list all providers with health + metrics
  - `POST /admin/providers` — register new provider (credentials → secrets manager)
  - `PATCH /admin/providers/:id` — update config, weights, limits
  - `PATCH /admin/providers/:id/toggle` — enable / disable
  - `GET /admin/providers/:id/health` — live health check
- [ ] **Transaction Operations API**
  - `GET /admin/transactions` — filterable list (status, user, provider, date, type)
  - `GET /admin/transactions/:id` — full detail: overview + state timeline + ledger entries
  - `POST /admin/transactions/:id/retry` — re-queue failed step
  - `POST /admin/transactions/:id/resolve` — mark manually resolved (with audit note)
  - `POST /admin/transactions/:id/reverse` — initiate ledger reversal + user notification
- [ ] **Metrics API**
  - `GET /admin/metrics` — volume, revenue, success rates (period-selectable)
  - `GET /admin/reconciliation/reports` — daily reconciliation results
- [ ] **Dynamic Adapter Factory**
  - Reads `adapter_class` from `providers` table
  - Instantiates correct adapter at runtime
  - No restart required when new provider registered
- [ ] **Health Check Endpoint** (`GET /health`) — for load balancer / uptime monitoring
- [ ] **Rate Limiting** — per-user and per-endpoint request limits

**Exit Criteria:** A new provider can be registered and activated entirely through API calls. A stuck transaction can be found, inspected, and resolved without touching the server.

---

## Phase 5 — Reconciliation & Hardening
**Goal:** The system is production-safe. Money discrepancies are caught before they become problems.

### Deliverables
- [ ] **ReconciliationJob** (nightly cron)
  - Fetch all completed transactions for the period from internal DB
  - Call provider reporting APIs to confirm amounts and statuses
  - Diff internal ledger vs. provider records
  - Flag discrepancies above threshold
  - Generate reconciliation report (stored in DB, exposed via API)
- [ ] **Structured Logging** (JSON, with correlation IDs)
  - Every log line: `transactionId`, `userId`, `provider`, `action`
  - Zero sensitive data in logs (no amounts in debug, no account numbers)
- [ ] **Error Monitoring** (Sentry or equivalent)
  - Uncaught exceptions automatically reported
  - Financial errors (ledger failures, state machine violations) flagged as critical
- [ ] **Alerting**
  - "No successful transactions in 15 minutes" → operations alert
  - "Provider X has failed 5 consecutive transactions" → operations alert
  - "DLQ has > N items" → operations alert
  - "Reconciliation found discrepancy" → operations alert
- [ ] **Security Hardening Review**
  - Webhook IP allowlists verified per provider
  - No secrets in logs confirmed (automated check)
  - HMAC verification tested with tampered payloads
  - Idempotency tested with duplicate API calls
  - Concurrent balance writes tested (race condition validation)
- [ ] Load test: ledger write path under concurrent load
- [ ] Full end-to-end test suite (all happy paths + all failure paths)

**Exit Criteria:** System handles 100 concurrent transactions without ledger inconsistency. All failure scenarios tested. Monitoring and alerts active.

---

## Phase 6 — Second Provider (Extensibility Proof)
**Goal:** Prove that adding a new provider requires zero changes to business logic.

### Deliverables
- [ ] **Fiat PSP B Adapter** (implements `IFiatPaymentPort`)
  - Full implementation: deposit, webhook verification, status polling
- [ ] **Conversion Provider B Adapter** (implements `ICryptoConversionPort`)
  - Full implementation: quote, execute, webhook, status polling
- [ ] Register both in `providers` table via Admin API
- [ ] Routing Engine automatically includes both in scoring
- [ ] Integration test: disable Provider A in admin panel → traffic routes to Provider B automatically
- [ ] **Failover test**: Provider A webhook times out → system retries on Provider B
- [ ] Document: "How to add a new provider" (1 page, referenced in README)

**Exit Criteria:** Second providers live in production with zero changes to any core business logic file. Failover demonstrated end-to-end.

---

## Technology Decisions

| Concern | Decision | Rationale |
|---|---|---|
| Language | TypeScript (Node.js) | Type safety critical for financial logic |
| HTTP Framework | Fastify | Built-in schema validation, faster than Express |
| ORM | Prisma | Type-safe migrations + queries |
| Database | PostgreSQL | Full ACID compliance, row-level locking |
| Cache | Redis | Quote TTL, idempotency key store |
| Job Queue | BullMQ (Redis) | Retry engine, background jobs |
| Logging | Pino | Structured JSON, minimal overhead |
| Secrets | Secrets Manager (cloud) | Credentials never in code, DB, or logs |
| Containerisation | Docker + Compose | Reproducible environment |

---

## Security Contract (Non-Negotiable)

These properties must hold throughout all phases:

| Property | How It Is Enforced |
|---|---|
| Money never disappears silently | Every state change + ledger entry in one atomic DB transaction |
| No double-credits | Idempotency key on every transaction. Payload hash on every webhook. |
| No race conditions on balances | `SELECT ... FOR UPDATE` row locking before every debit |
| No credentials exposed | Secrets manager only. Reference keys in DB, never values. |
| No fake webhooks processed | HMAC signature verification before any processing. IP allowlist. |
| Full audit trail | `transaction_events` is append-only. Every action logged with actor. |

---

## What Is Out of Scope (this engine)

| Concern | Owner |
|---|---|
| User registration / authentication | XaneApp |
| Frontend / mobile UI | XaneApp |
| KYC provider integration | XaneApp or shared service |
| Wallet custody / key management | XaneApp or dedicated custody service |
| Notification delivery (SMS, push) | XaneApp (XanePay fires events, XaneApp delivers) |
| DEX / on-chain swap execution | Phase 3+ optional, separate adapter |

---

*See [`docs/PRD.md`](./PRD.md) for full technical architecture, database schema, port interfaces, and flow diagrams.*
