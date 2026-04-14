# XanePay – Onramp/Offramp Engine: Implementation Plan

## Overview

XanePay's onramp/offramp engine is a **financial orchestration service** that abstracts
multiple external liquidity providers behind a single, clean API. It handles receiving
fiat (NGN), routing to the best crypto conversion provider, executing the swap, and
settling to the user's wallet or bank — while tracking every naira through a
double-entry ledger.

The architecture uses **Hexagonal Architecture (Ports & Adapters)** to ensure that:
- No business logic depends on any specific provider
- Adding a new provider = writing one new Adapter class, zero core changes
- Providers can eventually be toggled on/off from an admin config panel

---

## Rewritten PRD: XanePay Onramp/Offramp Engine (MVP)

### Product Goal
Build an MVP onramp/offramp engine that handles:
1. **Onramp:** User pays NGN → receives Crypto in their wallet
2. **Offramp:** User sends Crypto → receives NGN in their bank or XanePay balance

### MVP Scope (Phase 1)
- **1 Fiat PSP:** Paystack (virtual accounts / direct charge)
- **1 Conversion Provider:** BVNK
- **1 Currency Pair:** NGN ↔ ETH (via USDT stablecoin path)
- **Basic Routing:** Single provider, best quote wins
- **Internal Ledger:** Double-entry, PostgreSQL
- **Webhooks:** Paystack + BVNK webhook receivers

### What is Explicitly OUT of Scope for MVP
- User authentication/registration (provided by XaneApp)
- Frontend UI (provided by XaneApp team)
- DEX/Uniswap integration
- Smart routing algorithm (multi-provider scoring)
- Admin dashboard (Phase 2)
- KYC provider integration (Phase 2, pluggable)
- Treasury/liquidity management

### Success Criteria for MVP
- A user can deposit NGN and receive ETH in their wallet end-to-end
- A user can send ETH and receive NGN in their bank account end-to-end
- All transactions are tracked with full audit trail
- No money can be lost silently — every failure is logged and recoverable
- A second provider (e.g. Juicyway) can be added by writing ONE new class + config

---

## Architecture: Hexagonal (Ports & Adapters)

```
┌─────────────────────────────────────────────┐
│              PRIMARY ADAPTERS               │
│  (things that CALL our system)              │
│                                             │
│  REST API  │  Webhook Receiver  │  CLI Jobs │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│              APPLICATION CORE               │
│  (Pure business logic — no dependencies)    │
│                                             │
│  ┌─────────────┐  ┌──────────────────────┐  │
│  │  Use Cases  │  │   Domain Entities    │  │
│  │             │  │                      │  │
│  │ - OnrampUC  │  │ - Transaction        │  │
│  │ - OfframpUC │  │ - LedgerEntry        │  │
│  │ - QuoteUC   │  │ - Quote              │  │
│  │ - StatusUC  │  │ - Provider           │  │
│  └─────────────┘  └──────────────────────┘  │
│                                             │
│              PORTS (Interfaces)             │
│  ┌──────────────────────────────────────┐   │
│  │ IFiatPaymentPort                     │   │
│  │ ICryptoConversionPort                │   │
│  │ ILedgerPort                          │   │
│  │ ITransactionRepository               │   │
│  │ INotificationPort                    │   │
│  │ IProviderRegistryPort                │   │
│  └──────────────────────────────────────┘   │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│             SECONDARY ADAPTERS              │
│  (things our system CALLS)                  │
│                                             │
│  ┌────────────┐  ┌────────────┐             │
│  │  Paystack  │  │    BVNK    │  ← Phase 1  │
│  │  Adapter   │  │  Adapter   │             │
│  └────────────┘  └────────────┘             │
│  ┌────────────┐  ┌────────────┐             │
│  │  Monnify   │  │ Juicyway   │  ← Phase 2  │
│  │  Adapter   │  │  Adapter   │  (no core   │
│  └────────────┘  └────────────┘   changes)  │
│  ┌────────────┐  ┌────────────┐             │
│  │ PostgreSQL │  │   Redis    │  ← Infra     │
│  │  Adapter   │  │  Adapter   │             │
│  └────────────┘  └────────────┘             │
└─────────────────────────────────────────────┘
```

### Why Hexagonal Architecture Works Here

| Problem | How Hex Arch Solves It |
|---|---|
| BVNK goes down, need to switch to Juicyway | Write `JuicywayAdapter`, update config — core untouched |
| Paystack raises fees, switch to Monnify | Write `MonnifyAdapter` — same |
| Want to add DEX support | Write `UniswapAdapter` implementing `ICryptoConversionPort` |
| Want to test without hitting real APIs | Swap real adapters for mock adapters in tests |
| Add new provider from admin panel | Provider Registry reads from DB config, instantiates correct adapter |

---

## Project Structure

```
xanepay-engine/
├── src/
│   ├── domain/                        # Pure business logic
│   │   ├── entities/
│   │   │   ├── Transaction.ts         # Core transaction entity
│   │   │   ├── LedgerEntry.ts         # Double-entry ledger record
│   │   │   ├── Quote.ts               # Price quote from provider
│   │   │   └── Provider.ts            # Provider config entity
│   │   ├── enums/
│   │   │   ├── TransactionStatus.ts   # State machine states
│   │   │   ├── TransactionType.ts     # ONRAMP | OFFRAMP
│   │   │   └── Currency.ts            # NGN | ETH | USDT | USD
│   │   └── errors/
│   │       ├── InsufficientFundsError.ts
│   │       ├── ProviderUnavailableError.ts
│   │       └── QuoteExpiredError.ts
│   │
│   ├── ports/                         # Interfaces (the contracts)
│   │   ├── outbound/
│   │   │   ├── IFiatPaymentPort.ts    # PSP interface
│   │   │   ├── ICryptoConversionPort.ts # Conversion provider interface
│   │   │   ├── ILedgerPort.ts         # Ledger interface
│   │   │   ├── ITransactionRepository.ts
│   │   │   └── INotificationPort.ts
│   │   └── inbound/
│   │       ├── IOnrampUseCase.ts
│   │       └── IOfframpUseCase.ts
│   │
│   ├── application/                   # Use cases (orchestration)
│   │   ├── usecases/
│   │   │   ├── InitiateOnrampUseCase.ts
│   │   │   ├── ConfirmOnrampUseCase.ts
│   │   │   ├── InitiateOfframpUseCase.ts
│   │   │   ├── ConfirmOfframpUseCase.ts
│   │   │   ├── GetQuoteUseCase.ts
│   │   │   └── GetTransactionStatusUseCase.ts
│   │   └── services/
│   │       ├── RoutingService.ts      # Selects best provider
│   │       ├── LedgerService.ts       # Double-entry bookkeeping
│   │       └── RetryService.ts        # Retry engine
│   │
│   ├── adapters/                      # Concrete implementations
│   │   ├── primary/                   # Inbound adapters
│   │   │   ├── http/
│   │   │   │   ├── routes/
│   │   │   │   │   ├── onramp.routes.ts
│   │   │   │   │   ├── offramp.routes.ts
│   │   │   │   │   └── webhook.routes.ts
│   │   │   │   └── controllers/
│   │   │   │       ├── OnrampController.ts
│   │   │   │       ├── OfframpController.ts
│   │   │   │       └── WebhookController.ts
│   │   │   └── jobs/
│   │   │       ├── ReconciliationJob.ts  # Nightly reconciliation
│   │   │       └── StuckTransactionJob.ts # Scan + fix stuck txns
│   │   │
│   │   └── secondary/                 # Outbound adapters
│   │       ├── fiat/
│   │       │   ├── PaystackAdapter.ts  # Implements IFiatPaymentPort
│   │       │   └── MonnifyAdapter.ts   # Phase 2 — plug in, no core change
│   │       ├── crypto/
│   │       │   ├── BVNKAdapter.ts      # Implements ICryptoConversionPort
│   │       │   ├── JuicywayAdapter.ts  # Phase 2
│   │       │   └── YellowCardAdapter.ts # Phase 2
│   │       ├── persistence/
│   │       │   ├── PostgresLedgerAdapter.ts
│   │       │   └── PostgresTransactionRepository.ts
│   │       └── notification/
│   │           └── SlackNotificationAdapter.ts
│   │
│   └── infrastructure/
│       ├── database/
│       │   ├── migrations/            # DB schema migrations
│       │   └── seeds/
│       ├── queue/
│       │   └── BullMQQueue.ts         # Job queue for async ops
│       ├── cache/
│       │   └── RedisCache.ts          # Quote caching
│       └── config/
│           └── providers.config.ts    # Provider registry config
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docker-compose.yml
└── package.json
```

---

## Database Schema

### Core Tables

```sql
-- Provider Registry (editable from admin, no code change needed)
CREATE TABLE providers (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          VARCHAR(100) NOT NULL UNIQUE,  -- 'paystack', 'bvnk'
  type          VARCHAR(50) NOT NULL,          -- 'FIAT_PSP' | 'CRYPTO_CONVERSION'
  adapter_class VARCHAR(100) NOT NULL,         -- 'PaystackAdapter'
  is_active     BOOLEAN DEFAULT true,
  priority      INTEGER DEFAULT 1,            -- routing priority
  config        JSONB,                         -- API keys reference, not values
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Accounts (double-entry chart of accounts)
CREATE TABLE accounts (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id      UUID NOT NULL,               -- user_id or system
  owner_type    VARCHAR(50) NOT NULL,        -- 'USER' | 'SYSTEM' | 'PROVIDER'
  account_type  VARCHAR(50) NOT NULL,        -- 'NGN_WALLET' | 'TREASURY' | 'FEE_REVENUE'
  currency      VARCHAR(10) NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(owner_id, account_type, currency)
);

-- Ledger Entries (NEVER delete, NEVER update — only append)
CREATE TABLE ledger_entries (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id  UUID NOT NULL REFERENCES transactions(id),
  account_id      UUID NOT NULL REFERENCES accounts(id),
  entry_type      VARCHAR(10) NOT NULL CHECK (entry_type IN ('DEBIT', 'CREDIT')),
  amount          NUMERIC(20, 8) NOT NULL CHECK (amount > 0),
  currency        VARCHAR(10) NOT NULL,
  description     TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Transactions (state machine)
CREATE TABLE transactions (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key     VARCHAR(255) UNIQUE NOT NULL,  -- prevents double processing
  user_id             UUID NOT NULL,
  type                VARCHAR(20) NOT NULL,           -- 'ONRAMP' | 'OFFRAMP'
  status              VARCHAR(50) NOT NULL DEFAULT 'INITIATED',
  from_currency       VARCHAR(10) NOT NULL,
  to_currency         VARCHAR(10) NOT NULL,
  from_amount         NUMERIC(20, 8) NOT NULL,
  to_amount           NUMERIC(20, 8),                -- filled after quote confirmation
  fee_amount          NUMERIC(20, 8),
  provider_id         UUID REFERENCES providers(id),
  provider_txn_id     VARCHAR(255),                  -- provider's own reference
  quote_id            VARCHAR(255),                  -- locked quote reference
  quote_expires_at    TIMESTAMPTZ,
  user_wallet_address VARCHAR(255),                  -- destination for onramp
  user_bank_code      VARCHAR(20),                   -- destination for offramp
  user_bank_account   VARCHAR(20),
  retry_count         INTEGER DEFAULT 0,
  last_error          TEXT,
  metadata            JSONB DEFAULT '{}',
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- State Transition Log (full audit trail)
CREATE TABLE transaction_events (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id  UUID NOT NULL REFERENCES transactions(id),
  from_status     VARCHAR(50),
  to_status       VARCHAR(50) NOT NULL,
  triggered_by    VARCHAR(100),   -- 'webhook' | 'user' | 'system' | 'retry_job'
  payload         JSONB,          -- raw webhook payload or system data
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_transactions_user_id ON transactions(user_id);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_provider_txn_id ON transactions(provider_txn_id);
CREATE INDEX idx_ledger_entries_account_id ON ledger_entries(account_id);
CREATE INDEX idx_ledger_entries_transaction_id ON ledger_entries(transaction_id);
```

---

## Core Port Interfaces (TypeScript)

### IFiatPaymentPort — PSP Contract
```typescript
export interface IFiatPaymentPort {
  /**
   * Create a payment request (virtual account or payment link)
   * Returns a reference the user uses to make payment
   */
  initiateDeposit(params: {
    userId: string;
    amount: number;        // in kobo (NGN smallest unit)
    currency: 'NGN';
    reference: string;     // your idempotency key
    callbackUrl: string;
  }): Promise<{
    paymentReference: string;
    paymentUrl?: string;
    virtualAccountNumber?: string;
    virtualAccountBank?: string;
    expiresAt: Date;
  }>;

  /**
   * Verify a webhook payload is authentic (HMAC signature check)
   */
  verifyWebhookSignature(payload: string, signature: string): boolean;

  /**
   * Parse a raw webhook into a normalized event
   */
  parseWebhookEvent(payload: unknown): NormalizedPaymentEvent;

  /**
   * Directly query provider for transaction status (webhook fallback)
   */
  getTransactionStatus(providerReference: string): Promise<NormalizedPaymentStatus>;
}
```

### ICryptoConversionPort — Conversion Provider Contract
```typescript
export interface ICryptoConversionPort {
  /**
   * Get a rate-locked quote for a currency pair
   */
  getQuote(params: {
    fromCurrency: string;    // 'NGN'
    toCurrency: string;      // 'ETH'
    fromAmount?: number;
    toAmount?: number;
  }): Promise<{
    quoteId: string;
    fromAmount: number;
    toAmount: number;
    rate: number;
    fee: number;
    expiresAt: Date;         // rate lock expiry (30-60 seconds)
    providerName: string;
  }>;

  /**
   * Execute a previously obtained quote
   */
  executeQuote(params: {
    quoteId: string;
    destinationAddress?: string;  // for onramp: user wallet
    destinationBank?: string;     // for offramp: user bank
    destinationAccount?: string;
    reference: string;
  }): Promise<{
    providerTransactionId: string;
    status: 'PENDING' | 'PROCESSING';
  }>;

  /**
   * Poll transaction status (webhook fallback)
   */
  getTransactionStatus(providerTransactionId: string): Promise<{
    status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED';
    completedAt?: Date;
    txHash?: string;       // on-chain hash for crypto side
  }>;

  /**
   * Verify webhook payload authenticity
   */
  verifyWebhookSignature(payload: string, signature: string): boolean;

  /**
   * Parse raw webhook to normalized format
   */
  parseWebhookEvent(payload: unknown): NormalizedConversionEvent;
}
```

---

## Transaction State Machine

```
INITIATED
  │
  ├─→ PAYMENT_PENDING        (PSP payment link/VA created, waiting for user)
  │       │
  │       ├─→ PAYMENT_CONFIRMED   (Paystack webhook: charge.success)
  │       │       │
  │       │       ├─→ QUOTE_REQUESTED       (calling BVNK for quote)
  │       │       │       │
  │       │       │       ├─→ QUOTE_LOCKED         (quote confirmed by user)
  │       │       │       │       │
  │       │       │       │       ├─→ CONVERSION_PROCESSING  (BVNK executing)
  │       │       │       │       │       │
  │       │       │       │       │       ├─→ SETTLEMENT_PENDING  (crypto sent)
  │       │       │       │       │       │       │
  │       │       │       │       │       │       └─→ COMPLETED ✅
  │       │       │       │       │       │
  │       │       │       │       │       └─→ FAILED → RETRY (max 3) → REVERSED ❌
  │       │       │       │       │
  │       │       │       │       └─→ QUOTE_EXPIRED (user took too long → restart)
  │       │       │       │
  │       │       │       └─→ QUOTE_FAILED (provider error → try next provider)
  │       │       │
  │       │       └─→ LEDGER_CREDIT_PENDING  (money received, ledger update)
  │       │
  │       └─→ PAYMENT_EXPIRED (user never paid → CANCELLED)
  │
  └─→ FAILED (catastrophic error → manual review)
```

---

## API Contract (Endpoints)

### Onramp Endpoints
```
POST /v1/onramp/initiate
  Body: { userId, fromAmount, fromCurrency, toCurrency, walletAddress }
  Returns: { transactionId, paymentUrl, virtualAccount, expiresAt }

GET  /v1/onramp/quote
  Query: ?fromAmount=500000&fromCurrency=NGN&toCurrency=ETH
  Returns: { quoteId, rate, toAmount, fee, expiresAt, providerName }

POST /v1/onramp/confirm
  Body: { transactionId, quoteId }
  Returns: { transactionId, status }

GET  /v1/transactions/:id/status
  Returns: { transactionId, status, fromAmount, toAmount, txHash?, events[] }
```

### Offramp Endpoints
```
POST /v1/offramp/initiate
  Body: { userId, fromAmount, fromCurrency, toCurrency, bankCode, accountNumber }
  Returns: { transactionId, depositAddress, expiresAt }
  Note: depositAddress is where user sends their crypto

GET  /v1/offramp/quote
  Query: ?fromAmount=0.01&fromCurrency=ETH&toCurrency=NGN

POST /v1/offramp/confirm
  Body: { transactionId, quoteId }
```

### Webhook Endpoints (Internal — provider calls these)
```
POST /v1/webhooks/paystack    (Paystack events)
POST /v1/webhooks/bvnk        (BVNK events)
POST /v1/webhooks/monnify     (Phase 2)
```

### Provider Management (Admin — enables adding providers without code)
```
GET  /admin/providers               (list all registered providers)
POST /admin/providers               (register a new provider config)
PATCH /admin/providers/:id/toggle   (enable/disable a provider)
GET  /admin/providers/:id/health    (check provider live status)
```

---

## Implementation Phases

### Phase 0 — Project Foundation (Week 1)
- [ ] Initialize TypeScript project (Node.js + Fastify)
- [ ] Set up PostgreSQL + Prisma ORM
- [ ] Set up Redis for quote caching
- [ ] Set up BullMQ for job queues
- [ ] Set up Docker Compose (Postgres, Redis, app)
- [ ] Define all domain entities and enums
- [ ] Define all Port interfaces
- [ ] Run database migrations
- [ ] Write base test setup (Jest + Supertest)

### Phase 1 — Core Engine (Weeks 2–3)
- [ ] Build `LedgerService` (double-entry, atomic writes)
- [ ] Build `TransactionStateMachine` (all state transitions)
- [ ] Build `RoutingService` (basic: returns only active provider)
- [ ] Build `RetryService` (exponential backoff, max 3 attempts)
- [ ] Write unit tests for all core services
- [ ] Build `TransactionRepository` (CRUD + status queries)

### Phase 2 — Paystack Integration (Week 3–4)
- [ ] Implement `PaystackAdapter` (implements `IFiatPaymentPort`)
  - [ ] `initiateDeposit()` — create payment link / charge
  - [ ] `verifyWebhookSignature()` — HMAC-SHA512 validation
  - [ ] `parseWebhookEvent()` — normalize charge.success event
  - [ ] `getTransactionStatus()` — fallback polling
- [ ] Build `WebhookController` for Paystack
- [ ] Integration test: full Paystack webhook flow in sandbox

### Phase 3 — BVNK Integration (Weeks 4–5)
- [ ] Implement `BVNKAdapter` (implements `ICryptoConversionPort`)
  - [ ] `getQuote()` — fetch rate-locked quote
  - [ ] `executeQuote()` — execute the conversion
  - [ ] `getTransactionStatus()` — poll for status
  - [ ] `verifyWebhookSignature()` + `parseWebhookEvent()`
- [ ] Build `WebhookController` for BVNK
- [ ] Implement Redis quote caching (TTL = quote expiry)
- [ ] Integration test: full BVNK quote → execute flow in sandbox

### Phase 4 — Full Flow Wire-Up (Week 5–6)
- [ ] Build all REST API endpoints (Onramp + Offramp)
- [ ] Wire use cases to adapters via dependency injection
- [ ] Build `StuckTransactionJob` (cron: every 5 mins, poll stuck txns)
- [ ] Build `ReconciliationJob` (cron: nightly, compare ledger vs provider)
- [ ] End-to-end test: NGN → ETH (Paystack → BVNK → wallet)
- [ ] End-to-end test: ETH → NGN (BVNK → Paystack → bank)

### Phase 5 — Hardening (Week 6–7)
- [ ] Add idempotency key enforcement on all endpoints
- [ ] Add request rate limiting
- [ ] Add webhook signature validation for all providers
- [ ] Add structured logging (Pino) with transaction correlation IDs
- [ ] Add Sentry for error alerting
- [ ] Add health check endpoint (`/health`)
- [ ] Load test the ledger write path
- [ ] Security audit: no secrets in logs, no sensitive data in errors

### Phase 6 — Provider Extensibility Demo (Week 7)
- [ ] Build `JuicywayAdapter` stub to prove extensibility
- [ ] Update `providers` table with Juicyway record
- [ ] Update `RoutingService` to read active providers from DB
- [ ] Demo: disable BVNK in DB → system auto-routes to Juicyway
- [ ] Document: "How to add a new provider" (1 file + 1 DB insert)

---

## Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| Language | TypeScript (Node.js) | Strong typing critical for financial logic |
| HTTP Framework | Fastify | Faster than Express, schema validation built-in |
| ORM | Prisma | Type-safe DB queries, migration management |
| Database | PostgreSQL | ACID compliance, row-level locking |
| Cache | Redis | Quote TTL caching, idempotency key store |
| Queue | BullMQ (Redis) | Retry jobs, background processing |
| Testing | Jest + Supertest | Unit + integration tests |
| Logging | Pino | Structured JSON logs, fast |
| Error Tracking | Sentry | Real-time error alerts |
| Secrets | Environment + Vault ref | API keys never in code or logs |
| Containerization | Docker + Docker Compose | Reproducible environments |

---

## Key Design Decisions

### 1. Idempotency Keys
Every transaction creation endpoint requires an `Idempotency-Key` header.
The key is stored in the `transactions.idempotency_key` column with a `UNIQUE` constraint.
If the same key is sent twice, the second request returns the existing transaction — no duplicate.

### 2. Ledger is Append-Only
`ledger_entries` rows are **never updated or deleted**.
Balance = `SUM(credits) - SUM(debits)` for any account.
This gives a perfect audit trail by design.

### 3. Webhooks Must Be Idempotent
Each webhook payload is hashed and stored in `transaction_events.payload`.
Before processing, check if this event hash already exists — if yes, return 200 and skip.

### 4. Money Never Moves Without a Ledger Entry
The `LedgerService.recordDoubleEntry()` method is called inside a PostgreSQL transaction alongside every state change. If the ledger write fails, the state change rolls back. Money and state stay in sync.

### 5. Provider Registry is Database-Driven
The `providers` table controls which adapters are active.
The `RoutingService` reads from this table at runtime.
Adding a new provider = write a new Adapter class + INSERT into `providers` table.
No restart required if you use dynamic adapter loading.

---

## What You Deliver to XaneApp (The API Consumer)

XaneApp calls your engine as a black box:

```
POST /v1/onramp/initiate     → "Here is a payment link"
GET  /v1/onramp/quote        → "Here is a rate"  
POST /v1/onramp/confirm      → "Executing now"
GET  /v1/transactions/:id    → "Here is the current status"
[Webhook to XaneApp]         → "Transaction completed/failed"
```

XaneApp never knows which PSP or conversion provider was used.
That is the entire point of the abstraction.

---

## Verification Plan

### Automated Tests
- Unit tests for `LedgerService` — balance calculations, double-entry invariants
- Unit tests for `TransactionStateMachine` — invalid transition rejection
- Unit tests for `RoutingService` — provider selection logic
- Integration tests for `PaystackAdapter` — against Paystack sandbox
- Integration tests for `BVNKAdapter` — against BVNK sandbox
- E2E test: full onramp flow (mock webhooks)
- E2E test: full offramp flow (mock webhooks)

### Manual Verification
- Test with real Paystack sandbox credentials
- Simulate a webhook arriving twice — verify no double credit
- Simulate BVNK going down mid-transaction — verify retry and rollback
- Verify `StuckTransactionJob` recovers an orphaned transaction
- Confirm second provider (Juicyway stub) routes correctly when BVNK disabled

---

## Open Questions for XanePay Team

> [!IMPORTANT]
> These must be answered before development begins

1. **Who holds the crypto treasury wallet private key** for DEX/on-chain operations?
2. **Which KYC provider** will be used (Smile Identity, Dojah, Youverify)?
3. **Does BVNK require KYC data** passed through at execute time, or do they manage it independently?
4. **What is the webhook URL scheme?** Do you have a domain ready, or are we using ngrok for MVP?
5. **Who manages the Paystack + BVNK accounts?** Sandbox credentials should be shared before Week 3.
6. **What notification format does XaneApp expect?** (webhook back to XaneApp, or polling?)
