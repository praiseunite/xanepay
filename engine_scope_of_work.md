# XanePay Engine — Scope of Work (Appendix to Services Agreement)

**Developer:** Praise Ubong (Alpha Sight Ltd, RC1965156)
**Project:** XanePay Fiat-to-Crypto / Crypto-to-Fiat Orchestration Engine
**Date:** April 20, 2026

---

## 1. What This Engine Is

XanePay's orchestration engine is the core financial processing layer that:
- Receives fiat (NGN) from users via a Payment Service Provider (PSP)
- Routes the conversion to the best crypto liquidity provider
- Executes the swap (NGN → Crypto or Crypto → NGN)
- Tracks every unit of value through an internal double-entry ledger
- Settles the result to the user's wallet or bank account

The engine is a **backend API service** — it has no frontend, no user interface, and no user-facing screens. It is consumed by the XaneApp frontend via REST API calls.

---

## 2. What Is IN Scope (Engine Deliverables)

### 2.1 Core Engine Components

| # | Component | Description |
|---|---|---|
| 1 | **Double-Entry Ledger** | Append-only, ACID-compliant internal accounting system. Every naira/crypto movement is recorded as a debit + credit pair. This is XanePay's source of truth — independent of any provider's records. |
| 2 | **Transaction State Machine** | Enforces the strict lifecycle of every transaction: INITIATED → PAYMENT_PENDING → PAYMENT_CONFIRMED → CONVERSION_PROCESSING → SETTLEMENT_PENDING → COMPLETED. Invalid transitions are rejected. Every state change is logged. |
| 3 | **Routing Service** | Selects the best conversion provider for each transaction based on rate, availability, and health status. Reads provider configuration from the database — no code changes needed to switch providers. |
| 4 | **Retry Engine** | Handles transient failures with exponential backoff (1 min → 5 min → 15 min). Classifies errors as retryable vs. terminal. Failed retries go to a Dead Letter Queue (DLQ) for manual review. |
| 5 | **Idempotency System** | Prevents double-processing of API calls and webhooks. Every transaction uses a unique idempotency key. Duplicate webhooks are detected and discarded. |

### 2.2 Provider Integrations

| # | Provider | Type | What Gets Built |
|---|---|---|---|
| 1 | **Paystack** | Fiat PSP (receives NGN) | `PaystackAdapter` — deposit initiation, webhook signature verification (HMAC-SHA512), webhook event parsing, fallback status polling |
| 2 | **BVNK** | Crypto Conversion Provider | `BVNKAdapter` — rate-locked quote fetching, quote execution, webhook signature verification, webhook event parsing, fallback status polling |

Each adapter implements a standardised **Port interface** — meaning a new provider can be added in the future by writing one new adapter file + one database insert, with zero changes to core business logic.

### 2.3 Port Interfaces (Abstractions)

| # | Interface | Purpose |
|---|---|---|
| 1 | `IFiatPaymentPort` | Contract that every fiat PSP adapter must implement (initiateDeposit, verifyWebhook, parseEvent, getStatus) |
| 2 | `ICryptoConversionPort` | Contract that every conversion provider adapter must implement (getQuote, executeQuote, verifyWebhook, parseEvent, getStatus) |
| 3 | `ILedgerPort` | Contract for ledger operations (recordDoubleEntry, getBalance, getEntriesByTransaction) |
| 4 | `ITransactionRepository` | Contract for transaction persistence (create, updateStatus, findById, findStuck) |
| 5 | `INotificationPort` | Contract for system alerts (Slack/email notification on failures) |

### 2.4 API Endpoints (Engine-Specific)

**Onramp (NGN → Crypto):**

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/v1/onramp/initiate` | Start conversion — returns payment link or virtual account |
| `GET` | `/v1/onramp/quote` | Get rate-locked quote from best provider |
| `POST` | `/v1/onramp/confirm` | Execute the quote after user confirms |

**Offramp (Crypto → NGN):**

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/v1/offramp/initiate` | Start offramp — returns crypto deposit address |
| `GET` | `/v1/offramp/quote` | Get rate-locked quote for crypto → NGN |
| `POST` | `/v1/offramp/confirm` | Execute the quote after user confirms |

**Transaction Status:**

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/v1/transactions/:id/status` | Check current state of any transaction (with event history) |

**Webhooks (Provider → Engine):**

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/v1/webhooks/paystack` | Receives and validates Paystack payment events |
| `POST` | `/v1/webhooks/bvnk` | Receives and validates BVNK conversion events |

**System:**

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Health check endpoint (infra + database + Redis) |

**Total: 10 API endpoints**

### 2.5 Database Tables (Engine-Specific)

| # | Table | Purpose |
|---|---|---|
| 1 | `transactions` | Core transaction records — status, amounts, currencies, provider reference, idempotency key, wallet/bank details, retry count |
| 2 | `ledger_entries` | Append-only double-entry records — debit/credit, amount, currency, linked to transaction and account |
| 3 | `accounts` | Chart of accounts — user wallets, system treasury, fee revenue, provider escrow, pending settlement |
| 4 | `providers` | Provider registry — name, type (FIAT_PSP / CRYPTO_CONVERSION), active status, priority, config reference |
| 5 | `transaction_events` | Full audit trail — every state transition with timestamp, trigger source, and raw payload |

**Total: 5 database tables + indexes**

### 2.6 Background Jobs

| # | Job | Frequency | Purpose |
|---|---|---|---|
| 1 | `StuckTransactionJob` | Every 5 minutes | Scans for transactions stuck in intermediate states beyond timeout — polls provider for status or escalates to DLQ |
| 2 | `ReconciliationJob` | Nightly | Compares internal ledger totals against provider-reported positions — flags discrepancies above threshold |

### 2.7 Webhook Security Pipeline

Every incoming webhook goes through a 5-step validation chain before processing:

1. **IP allowlisting** (if provider publishes IP ranges)
2. **Signature verification** (HMAC using provider's secret — each provider has a different algorithm)
3. **Payload parsing** into normalised internal format
4. **Idempotency check** — hash the payload, skip if already processed
5. **State machine transition** — only valid transitions are executed

### 2.8 Error Handling & Recovery

| Scenario | Handling |
|---|---|
| PSP webhook delayed or missing | Polling fallback via `getTransactionStatus()` on the provider adapter |
| Conversion provider fails mid-execution | Retry with exponential backoff; after max retries → DLQ + operations alert |
| Duplicate webhook received | Idempotency check rejects — returns HTTP 200, no reprocessing |
| Quote expires before user confirms | Transaction moves to QUOTE_EXPIRED → user must request new quote |
| Payout to bank fails | Retry up to 3x; if exhausted → credit NGN to user's XanePay balance + alert |

### 2.9 Technical Documentation

| Document | Content |
|---|---|
| API Documentation | All 10 endpoints — request/response schemas, error codes, examples |
| Architecture Overview | Hexagonal architecture diagram, port/adapter relationships, data flow |
| Provider Integration Guide | "How to add a new provider" — step-by-step (1 adapter file + 1 DB insert) |
| Database Schema Reference | All 5 tables with column definitions, constraints, indexes |
| Deployment Guide | Docker Compose setup, environment variables, health checks |

---

## 3. What Is NOT In Scope (XaneApp Team's Responsibility)

These items are explicitly **outside** the engine scope and belong to the XaneApp team:

| # | Item | Owner |
|---|---|---|
| 1 | User registration and authentication | XaneApp backend team |
| 2 | User profile management | XaneApp backend team |
| 3 | Frontend / mobile UI | XaneApp frontend team |
| 4 | KYC provider integration | XaneApp / separate engagement |
| 5 | Admin dashboard UI | Separate engagement (Phase 2) |
| 6 | Treasury management & replenishment engine | Separate engagement (Phase 2) |
| 7 | Additional provider integrations beyond Paystack + BVNK | Separate engagement |
| 8 | Smart routing algorithm (ML-based scoring) | Separate engagement (Phase 2) |
| 9 | Post-launch monitoring, on-call support, bug fixes | Separate retainer agreement |
| 10 | Hosting, server management, DevOps | XaneApp operations team |

---

## 4. Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| Language | TypeScript (Node.js) | Strong typing critical for financial logic |
| HTTP Framework | Fastify | Schema validation built-in, faster than Express |
| ORM | Prisma | Type-safe database queries, migration management |
| Database | PostgreSQL | ACID compliance, row-level locking for ledger integrity |
| Cache | Redis | Quote TTL caching, idempotency key store |
| Queue | BullMQ (Redis-backed) | Background job processing (stuck tx scanner, reconciliation) |
| Testing | Jest + Supertest | Unit + integration tests |
| Logging | Pino | Structured JSON logs with transaction correlation IDs |
| Containerisation | Docker + Docker Compose | Reproducible development and staging environments |

---

## 5. Implementation Timeline (2 Months)

### Milestone 0 — Foundation (Week 1)
- Project initialisation (TypeScript + Fastify + Prisma + Redis + BullMQ)
- Docker Compose setup (Postgres, Redis, app)
- All domain entities and enums defined
- All port interfaces defined
- Database migrations executed
- Base test setup (Jest + Supertest)

### Milestone 1 — Core Engine + Provider Integrations (Weeks 2–5)
- `LedgerService` — double-entry, atomic writes
- `TransactionStateMachine` — all state transitions with validation
- `RoutingService` — selects active provider from database config
- `RetryService` — exponential backoff, error classification, DLQ
- `PaystackAdapter` — full implementation of `IFiatPaymentPort`
- `BVNKAdapter` — full implementation of `ICryptoConversionPort`
- Webhook controllers for both providers (with signature verification)
- Redis quote caching
- Unit tests for all core services
- Integration tests against Paystack and BVNK sandbox environments

### Milestone 2 — Full Flow, Hardening & Handover (Weeks 6–8)
- All 10 REST API endpoints built and wired
- Dependency injection configuration
- `StuckTransactionJob` — cron-based scanner
- `ReconciliationJob` — nightly ledger vs. provider comparison
- Idempotency key enforcement on all endpoints
- Request rate limiting
- Structured logging with correlation IDs
- Health check endpoint
- End-to-end test: NGN → ETH (full onramp)
- End-to-end test: ETH → NGN (full offramp)
- Security review (no secrets in logs, no sensitive data in errors)
- Complete technical documentation (API docs, architecture, provider guide, deployment guide)
- Successful staging deployment

---

## 6. What the XaneApp Team Needs to Provide

Before development can begin, the following must be provided by the Company:

| # | Item | When Needed |
|---|---|---|
| 1 | Paystack sandbox API credentials | Before Week 3 |
| 2 | BVNK sandbox API credentials | Before Week 4 |
| 3 | Company GitHub repository with Developer write access | Before Week 1 |
| 4 | Webhook URL domain or ngrok setup for sandbox testing | Before Week 3 |
| 5 | Confirmation of currency pair (NGN ↔ ETH via USDT path) | Before Week 1 |
| 6 | Designated point of contact for technical decisions | Before Week 1 |

Delays in providing these items will extend the project timeline proportionally, as outlined in the amended Agreement.

---

*This document serves as the Technical Scope Appendix to the XanePay Services Agreement and replaces the Technical Milestone Checklist in Section 5 of the original Agreement.*
