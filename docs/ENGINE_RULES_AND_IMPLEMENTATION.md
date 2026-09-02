# XanePay Engine — Rules, TDD/BDD Standards & Implementation Phases

**Owner:** Praise Udoh  
**Architect:** Claude (Sonnet 4.6)  
**Started:** 2026-05-17  
**Status Legend:** 🔴 PENDING | 🟡 IN PROGRESS | ✅ DONE

> This is a living document. When a phase is completed, update its status to ✅ DONE with the completion date. Never skip a phase. Never mark DONE unless all tests are green and the live gate (where applicable) has been cleared.

---

## PART 1 — ARCHITECTURE RULES (Non-Negotiable)

These rules are enforced by code structure and must never be broken. Any PR that violates them is rejected.

### R1 — No Floats in the Domain
All monetary values are integers in their smallest unit (kobo for NGN, minor units for USDT/crypto).
Floats exist ONLY inside `JuicywayAdapter` at the external boundary, nowhere else.
```
✅  amountSource: 5000000  (50,000 NGN in kobo)
❌  amountSource: 50000.00
```

### R2 — Hexagonal Architecture: Domain Never Imports Adapters
The domain layer (`src/domain/`) must never import anything from `src/adapters/` or `src/infrastructure/`.
Domain depends on nothing. Adapters depend on domain. Never the reverse.
```
✅  domain/rules/fee-rules.ts  → imports nothing
❌  domain/rules/fee-rules.ts  → imports JuicywayAdapter
```

### R3 — Result Monad for All Errors (No Thrown Exceptions in Business Logic)
Every function that can fail returns `Result<T, EngineError>`. Exceptions are only caught at the adapter boundary and converted to Results.
```typescript
✅  async execute(): Promise<Result<QuoteResponse, EngineError>>
❌  async execute(): Promise<QuoteResponse>  // throws on failure
```

### R4 — No Hardcoded Secrets or Addresses
All secrets, API keys, wallet addresses, and URLs come from validated env config. Engine refuses to start if any are missing.
```
✅  getEnvConfig().JUICYWAY_API_KEY
❌  const API_KEY = "jw_live_abc123..."
```

### R5 — State Machine Controls All Status Transitions
`ConversionStatus` transitions are gated by `Conversion.transition()`. Nothing writes status directly to the DB without going through the state machine value object.
```typescript
✅  const result = conversion.transition(ConversionStatus.LEG1_PENDING)
❌  await db('conversions').update({ status: 'leg1_pending' })
```

### R6 — Redis Lock Before Any Provider Call
Any use case that calls an external provider (JuicyWay, smart contract) MUST acquire a Redis distributed lock keyed on `conversionId` before making the call. Released in `finally`.
```typescript
✅  const lock = await cache.setIfNotExists(`lock:${id}`, '1', 30000)
    try { await provider.executeSwap(...) } finally { await cache.delete(lock) }
❌  await provider.executeSwap(...)  // no lock
```

### R7 — Double-Entry Ledger Always
Every money movement produces exactly two ledger entries: one DEBIT and one CREDIT. If you write one, you write two. No exceptions.

### R8 — Idempotency at DB Level
The `UNIQUE(idempotency_key)` constraint on the `conversions` table is the last line of defence. Never remove it. Application-level idempotency checks are in addition to, not instead of, the DB constraint.

### R9 — Error Masking at the API Boundary
`DATABASE_ERROR`, `CACHE_ERROR`, and `INTERNAL` errors are masked to "An internal server error occurred" before reaching the client. Only `VALIDATION_FAILED`, `UNAUTHORIZED`, `RATE_LIMIT_EXCEEDED` expose their message.

### R10 — Audit Log All Admin and Financial Actions
Every admin action and every financial state change must produce an audit log entry. Append-only. Never update or delete audit logs.

### R11 — Fee Always From Engine, Never From Caller
The fee is always derived from the engine's own `fee_config` table or env fallback. The calling app (XanePay frontend, Xane Wallet) cannot override, inject, or suggest a fee. The engine is the single source of truth.

### R12 — Webhook Idempotency Key Committed After Processing
The Redis idempotency key for a webhook is committed AFTER `processEvent()` completes, not before. This ensures a process crash does not silently consume an event.

### R13 — BigInt Values Validated on DB Read
All BIGINT columns read from PostgreSQL must pass through `parseSafeBigInt()` which throws if the value exceeds `Number.MAX_SAFE_INTEGER`. Silent precision loss is a critical financial bug.

---

## PART 2 — TDD/BDD STANDARDS

### The Cycle (Never Break This Order)
```
1. RED    → Write the test. It describes the behaviour you want.
             Run it. Confirm it FAILS. If it passes without implementation, the test is wrong.
2. GREEN  → Write the minimum implementation to make the test pass.
             No more, no less.
3. REFACTOR → Clean up. The tests still pass.
4. REPEAT
```

### BDD Naming Convention
Tests read like sentences. A non-technical person should understand what each test does.
```typescript
// ✅ Correct
describe('FeeService', () => {
  describe('when no specific fee config exists for a conversion type', () => {
    it('falls back to the DEFAULT row in the database')
    it('falls back to env DEFAULT_FEE_BPS when no DEFAULT row exists')
  })
})

// ❌ Wrong
describe('FeeService', () => {
  it('test1')
  it('handles error case')
})
```

### Test File Structure
```
tests/
  unit/           ← pure logic, zero infrastructure, fast (<1s per file)
    domain/
    application/
    adapters/
  integration/    ← real postgres + redis via Docker, no external APIs
  e2e/            ← full stack, sandbox credentials required
    juicyway/     ← live tests against JuicyWay sandbox
    chaos/        ← concurrent requests, failure injection
```

### Test Rules
1. Unit tests must never touch the DB, Redis, or any network
2. Integration tests use `.env.test` pointing to Docker test containers
3. E2E tests use `.env.sandbox` with real JuicyWay sandbox credentials
4. Every new feature gets unit tests + integration tests minimum
5. Every live-gate feature gets an E2E test
6. A phase is NOT done until `npm test` exits with 0 failures

### Mocking Rules
- Unit tests: mock all ports (ICachePort, IConversionRepoPort, etc.) using `jest.fn()`
- Integration tests: use real DB + Redis, mock only external HTTP (JuicyWay API)
- E2E tests: mock nothing — real JuicyWay sandbox, real DB, real Redis

---

## PART 3 — IMPLEMENTATION PHASES

---

### ✅ PRE-WORK — Bugs Fixed (2026-05-17)

| Fix | File | Status |
|-----|------|--------|
| Weak API key validation (min 1 → min 32 chars) | `infrastructure/config/env.ts` | ✅ DONE |
| Unsafe `Number()` on DB BigInt columns | `conversion.repository.ts` | ✅ DONE |
| Webhook idempotency key committed before processing | `webhook-validator.middleware.ts` + `webhook.controller.ts` | ✅ DONE |
| Test mocks missing `feeCurrency` field | `orchestration.test.ts` + `admin.controller.test.ts` | ✅ DONE |

---

### 🔴 PHASE 1 — Fee Architecture Refactor

**Goal:** Fee is percentage-based (0.5% default), conversion-type aware, configurable from admin without code changes.

**Why first:** Every conversion in every phase depends on correct fee logic. Nothing downstream works correctly without this.

**Live gate:** None. Fully testable offline.

#### Tests to Write First

**File:** `tests/unit/application/services/fee-service.spec.ts`
```
describe('FeeService')
  describe('conversion type derivation')
    it('derives FIAT_TO_CRYPTO from NGN source and USDT target')
    it('derives FIAT_TO_CRYPTO from NGN source and SOL target')
    it('derives CRYPTO_TO_FIAT from USDT source and NGN target')
    it('derives CRYPTO_TO_FIAT from SOL source and NGN target')
    it('derives CRYPTO_TO_CRYPTO from USDT source and SOL target')
    it('derives CRYPTO_TO_CRYPTO from SOL source and BTC target')

  describe('fee lookup with DB rows')
    it('returns the exact fee_percent for a matching conversion_type row')
    it('returns DEFAULT row fee when no specific conversion_type row exists')
    it('returns a different fee for FIAT_TO_CRYPTO vs CRYPTO_TO_CRYPTO')

  describe('fee lookup fallback chain')
    it('falls back to env DEFAULT_FEE_BPS when no DB rows exist at all')
    it('never returns a float — always returns integer basis points')
    it('converts 0.50 percent to 50 basis points correctly')
    it('converts 1.00 percent to 100 basis points correctly')

  describe('fee bounds enforcement')
    it('returns minFeeAmount when calculated fee is below minimum')
    it('returns maxFeeAmount when calculated fee exceeds maximum')
    it('returns calculated fee when within min/max bounds')
```

**File:** `tests/unit/domain/rules/fee-rules.spec.ts`
```
describe('applyFeeToRate')
  it('applies 50 bps (0.5%) correctly to a standard NGN rate')
  it('applies 0 bps correctly — returns original rate unchanged')
  it('floors the result — never rounds up (protects user)')
  it('handles BTC max rate (8×10⁹) without exceeding MAX_SAFE_INTEGER')
  it('handles 10000 bps (100%) without overflow')
```

**File:** `tests/integration/fee-config.integration.spec.ts`
```
describe('FeeConfigRepository')
  it('inserts a new FIAT_TO_CRYPTO fee config row')
  it('retrieves active fee config by conversion_type')
  it('returns null when no active row exists for a type')
  it('retrieves DEFAULT fallback row when specific type not found')
  it('marks old row inactive when a new one is inserted for same type')
```

#### Implementation Steps

1. **Migration `10_refactor_fee_config.ts`:**
   - Add `conversion_type` VARCHAR(30) column
   - Add `fee_percent` NUMERIC(5,2) column  
   - Keep `fee_bps` as computed/derived (or remove and compute at read time)
   - Remove old `currency` column
   - Seed DEFAULT row: `{ conversion_type: 'DEFAULT', fee_percent: 0.50 }`
   - Add index on `(conversion_type, is_active)`

2. **`domain/types/enums.ts`:** Add `ConversionType` enum

3. **`domain/rules/conversion-type.ts`:** Pure function `deriveConversionType(source, target)`

4. **`FeeConfigRecord`** in `records.ts`: Add `conversionType` and `feePercent` fields

5. **`FeeConfigRepository`:** Update queries to use new columns, add `getByConversionType()`

6. **`FeeService`:** Rewrite `getActiveFeeConfig()` to use fallback chain lookup

7. **`env.ts`:** Change `DEFAULT_FEE_BPS` default from `200` to `50`

8. **Admin schema `admin.schema.ts`:** Accept `{ conversionType, feePercent }` for `POST /admin/fees`

9. **`AdminController`:** Convert `feePercent` → bps on write, bps → `feePercent` on read

#### Phase 1 Done When
- [ ] All fee-service.spec.ts tests GREEN
- [ ] All fee-rules.spec.ts tests GREEN  
- [ ] All fee-config.integration.spec.ts tests GREEN
- [ ] `npm test` exits 0
- [ ] Admin can POST `{ conversionType: "FIAT_TO_CRYPTO", feePercent: 0.5 }` and engine uses it on next quote

**STATUS: 🔴 PENDING**

---

### 🔴 PHASE 2 — Destination + Deposit Fields + JuicyWay Full Extension

**Goal:** Engine knows where to send money (user's wallet or bank) and how to receive crypto (deposit address). JuicyWay adapter can generate deposit addresses and initiate payouts.

**Why second:** Without destination fields, a completed swap has nowhere to send the output. This is the minimum for a real end-to-end flow.

**Live gate:** Full NGN → USDT → user TRC20 wallet, live against JuicyWay sandbox. Do not proceed to Phase 3 until this passes live.

#### Tests to Write First

**File:** `tests/unit/adapters/driven/providers/juicyway/juicyway-deposit.spec.ts`
```
describe('JuicywayAdapter.generateDepositAddress')
  it('calls POST /payment-sessions with correct params')
  it('returns a deposit address on success')
  it('returns the chain alongside the address')
  it('returns an expiry time from the response')
  it('returns PROVIDER_ERROR when JuicyWay API is unreachable')
  it('returns PROVIDER_ERROR when JuicyWay returns a non-200 status')
```

**File:** `tests/unit/adapters/driven/providers/juicyway/juicyway-payout.spec.ts`
```
describe('JuicywayAdapter.sendPayout')
  describe('crypto wallet payout')
    it('calls POST /payouts with type crypto_address')
    it('includes the correct destination address and chain')
    it('sends the amount in minor units')
    it('returns the payout reference on success')
  
  describe('bank account payout')
    it('calls POST /payouts with type bank_account')
    it('includes account number and bank code')
    it('sends the correct NGN amount in kobo')
    it('returns the payout reference on success')
  
  describe('error handling')
    it('returns PROVIDER_ERROR when payout is rejected')
    it('returns PROVIDER_ERROR on network timeout')
```

**File:** `tests/unit/application/use-cases/initiate-conversion.spec.ts`
```
describe('InitiateConversionUseCase')
  it('stores user_destination in the conversion record')
  it('validates destination is present before creating quote')
  it('returns VALIDATION_FAILED when destination is missing')
  it('sets conversion_type on the conversion record')
```

**File:** `tests/integration/payout-flow.integration.spec.ts`
```
describe('Payout flow')
  it('conversion record stores destination address after creation')
  it('payout reference is recorded after sendPayout is called')
  it('conversion status moves to PAYOUT_PENDING after payout initiated')
  it('conversion status moves to COMPLETED after payout webhook received')
```

#### Implementation Steps

1. **Migration `11_add_destination_and_payout_fields.ts`:**
   ```
   conversions table — add columns:
     conversion_type       VARCHAR(30)  NOT NULL
     user_destination      JSONB        NOT NULL
       -- stores: { type: 'crypto_wallet', address: '...', chain: 'TRX' }
       -- OR:    { type: 'bank_account', accountNumber: '...', bankCode: '...' }
     deposit_reference     VARCHAR(255) NULLABLE  (JuicyWay payment session ID)
     payout_reference      VARCHAR(255) NULLABLE  (JuicyWay payout ID)
   ```

2. **`ConversionRecord`** in `records.ts`: Add `conversionType`, `userDestination`, `depositReference`, `payoutReference`

3. **`IFiatCryptoProviderPort`:** Add `generateDepositAddress()` and `sendPayout()` method signatures

4. **`JuicywayAdapter`:** Implement both new methods

5. **`ConversionRepository`:** Update `create()` and `mapToDomain()` for new fields

6. **`InitiateConversionUseCase`:** Validate and store `userDestination` and `conversionType`

7. **`ExecuteSwapUseCase`:** After swap confirmed → deduct fee → call `sendPayout()` → update `payoutReference`

8. **`WebhookHandlerService`:** Handle payout completion webhook → mark `COMPLETED`

9. **Extend `ConversionStatus`:** Add `PAYOUT_PENDING` state

10. **Extend `VALID_TRANSITIONS`:** Add `PROVIDER_CONFIRMED → PAYOUT_PENDING → COMPLETED`

#### Phase 2 Done When
- [ ] All juicyway-deposit.spec.ts tests GREEN
- [ ] All juicyway-payout.spec.ts tests GREEN
- [ ] All payout-flow.integration.spec.ts tests GREEN
- [ ] `npm test` exits 0
- [ ] **LIVE GATE:** POST /quote + POST /execute against JuicyWay sandbox → USDT arrives in test TRC20 wallet ✅

**STATUS: 🔴 PENDING**

---

### 🔴 PHASE 3 — Conversion Routing (Auto-detect Provider)

**Goal:** Engine automatically detects which provider to use based on source/target currencies. XanePay and Xane Wallet both call the same engine — it routes correctly for each.

**Live gate:** USDT → NGN → user's bank account, live against JuicyWay sandbox.

#### Tests to Write First

**File:** `tests/unit/application/services/provider-orchestrator.spec.ts`
```
describe('ProviderOrchestrator routing')
  it('routes FIAT_TO_CRYPTO to JuicyWay adapter')
  it('routes CRYPTO_TO_FIAT to JuicyWay adapter')
  it('routes CRYPTO_TO_CRYPTO to XaneContract adapter')
  it('returns PROVIDER_UNAVAILABLE when selected provider is UNHEALTHY')
  it('selects fallback provider when primary is UNKNOWN')
```

**File:** `tests/unit/domain/rules/conversion-type.spec.ts`
```
describe('deriveConversionType')
  it('returns FIAT_TO_CRYPTO for NGN → USDT')
  it('returns FIAT_TO_CRYPTO for NGN → SOL')
  it('returns FIAT_TO_CRYPTO for NGN → BTC')
  it('returns CRYPTO_TO_FIAT for USDT → NGN')
  it('returns CRYPTO_TO_FIAT for SOL → NGN')
  it('returns CRYPTO_TO_CRYPTO for USDT → SOL')
  it('returns CRYPTO_TO_CRYPTO for SOL → BTC')
  it('throws if both currencies are fiat (not supported)')
```

#### Implementation Steps

1. **`domain/rules/conversion-type.ts`:** `deriveConversionType(source, target)` pure function

2. **`ProviderOrchestrator`:** Add routing logic by `ConversionType`

3. **`InitiateConversionUseCase`:** Call `deriveConversionType` at quote time, store result

4. **`ExecuteSwapUseCase`:** Read `conversionType` from record, select correct adapter

#### Phase 3 Done When
- [ ] All provider-orchestrator.spec.ts routing tests GREEN
- [ ] All conversion-type.spec.ts tests GREEN
- [ ] `npm test` exits 0
- [ ] **LIVE GATE:** USDT → NGN → user's Nigerian bank account live on JuicyWay sandbox ✅

**STATUS: 🔴 PENDING**

---

### 🔴 PHASE 4 — Multi-hop State Machine

**Goal:** Engine handles 2-leg conversions (e.g. NGN → USDT → SOL). Leg 2 only starts after Leg 1 is confirmed on-chain. Leg 2 failure does not lose user funds.

**Live gate:** NGN → SOL full multi-hop flow live on sandbox.

#### Tests to Write First

**File:** `tests/unit/domain/value-objects/conversion-multihop.spec.ts`
```
describe('ConversionStatus multi-hop transitions')
  it('allows QUOTED → LEG1_PENDING')
  it('allows LEG1_PENDING → LEG1_CONFIRMED')
  it('allows LEG1_CONFIRMED → LEG2_PENDING')
  it('allows LEG2_PENDING → PAYOUT_PENDING')
  it('allows LEG2_PENDING → LEG2_FAILED')
  it('allows LEG2_FAILED → LEG2_PENDING (retry)')
  it('allows LEG2_FAILED → REVERSED (after max retries)')
  it('rejects LEG1_PENDING → COMPLETED (must go through LEG2)')
  it('rejects LEG2_PENDING → LEG1_PENDING (no going back)')
  it('rejects any transition FROM a terminal state (COMPLETED, REVERSED)')
```

**File:** `tests/unit/application/services/webhook-handler-multihop.spec.ts`
```
describe('WebhookHandlerService multi-hop')
  it('triggers Leg 2 execution when LEG1_CONFIRMED webhook received')
  it('enqueues Leg 2 with correct source amount from Leg 1 output')
  it('does NOT trigger Leg 2 for single-hop conversions')
  it('marks LEG2_FAILED and enqueues retry on DEX failure')
  it('marks REVERSED after max retries exceeded')
```

#### Implementation Steps

1. **`domain/types/enums.ts`:** Add new status values: `LEG1_PENDING`, `LEG1_CONFIRMED`, `LEG2_PENDING`, `LEG2_FAILED`, `REVERSED`

2. **`domain/value-objects/conversion.ts`:** Extend `VALID_TRANSITIONS` map

3. **Migration `12_extend_conversion_status.ts`:** Update DB status constraint

4. **Migration `13_add_multihop_fields.ts`:**
   ```
   conversions table — add:
     leg1_provider_reference  VARCHAR(255) NULLABLE
     leg2_provider_reference  VARCHAR(255) NULLABLE
     is_multihop              BOOLEAN NOT NULL DEFAULT false
   ```

5. **`WebhookHandlerService`:** On `LEG1_CONFIRMED` → check `is_multihop` → enqueue Leg 2

6. **`ConversionProcessor`:** Handle `LEG2_FAILED` retry with BullMQ exponential backoff

#### Phase 4 Done When
- [ ] All conversion-multihop.spec.ts tests GREEN
- [ ] All webhook-handler-multihop.spec.ts tests GREEN
- [ ] `npm test` exits 0
- [ ] **LIVE GATE:** NGN → SOL multi-hop completes live, SOL arrives in test wallet ✅

**STATUS: 🔴 PENDING**

---

### 🔴 PHASE 5 — Smart Contract DEX Adapter (Xane Wallet)

**Goal:** Engine calls the already-deployed XanePay smart contract for crypto-to-crypto swaps. Contract finds best price, executes, splits fee to treasury in one transaction.

**Live gate:** USDT → SOL via smart contract on testnet.

#### Tests to Write First

**File:** `tests/unit/adapters/driven/contracts/xane-contract.spec.ts`
```
describe('XaneContractAdapter')
  describe('getBestPrice')
    it('calls the contract price discovery function with correct params')
    it('returns a DexQuote with source, target, rate, and estimated gas')
    it('returns PROVIDER_ERROR when contract call reverts')

  describe('executeSwap')
    it('submits transaction to contract with correct amounts')
    it('includes treasury address as fee recipient')
    it('includes 0.5% fee amount in the call')
    it('returns the tx hash on success')
    it('returns PROVIDER_ERROR when transaction fails')
    it('returns PROVIDER_ERROR when gas estimation fails')
```

**File:** `tests/unit/application/use-cases/execute-swap-dex.spec.ts`
```
describe('ExecuteSwapUseCase with DEX provider')
  it('acquires Redis lock before calling smart contract')
  it('releases Redis lock in finally block even on failure')
  it('records Leg 2 tx hash in conversion record')
  it('writes double-entry ledger entries for DEX swap')
  it('rejects duplicate concurrent execution (409)')
```

#### Implementation Steps

1. **`application/ports/driven/i-dex-provider.port.ts`:** Define `IDexProviderPort`

2. **`adapters/driven/contracts/xane-contract.adapter.ts`:** Implement adapter calling your deployed contract

3. **`env.ts`:** Add `XANE_CONTRACT_ADDRESS` and `XANE_CONTRACT_RPC_URL`

4. **`CompositionRoot`:** Register `XaneContractAdapter` as `IDexProviderPort`

5. **`ProviderOrchestrator`:** Wire `CRYPTO_TO_CRYPTO` → `XaneContractAdapter`

#### Phase 5 Done When
- [ ] All xane-contract.spec.ts tests GREEN
- [ ] All execute-swap-dex.spec.ts tests GREEN
- [ ] `npm test` exits 0
- [ ] **LIVE GATE:** USDT → SOL via Xane smart contract on testnet ✅

**STATUS: 🔴 PENDING**

---

### 🔴 PHASE 6 — Chaos Tests + Full E2E Suite

**Goal:** Prove the system is safe under adversarial conditions. Duplicate requests, network failures, Leg 2 failures, stranded funds.

#### Tests to Write

**File:** `tests/e2e/chaos/duplicate-execution.chaos.spec.ts`
```
describe('Chaos: Duplicate concurrent execution')
  it('only one of two simultaneous /execute calls acquires the Redis lock')
  it('the losing request receives 409 DUPLICATE_REQUEST')
  it('exactly one conversion record exists after both calls complete')
  it('the ledger has exactly two entries (one DEBIT, one CREDIT)')
```

**File:** `tests/e2e/chaos/leg2-failure.chaos.spec.ts`
```
describe('Chaos: Leg 2 DEX failure recovery')
  it('USDT remains in treasury when Leg 2 fails')
  it('conversion status is LEG2_FAILED, not COMPLETED')
  it('retry succeeds on second attempt after mock DEX is restored')
  it('REVERSED status is set after max retries (3) are exhausted')
  it('ledger entries prove USDT was never lost')
```

**File:** `tests/e2e/juicyway/full-onramp.e2e.spec.ts`
```
describe('E2E: Full NGN → USDT → TRC20 wallet (JuicyWay sandbox)')
  it('quote returns a locked rate with expiry time')
  it('execute initiates the JuicyWay swap')
  it('webhook marks conversion PROVIDER_CONFIRMED')
  it('payout sends USDT to test TRC20 wallet')
  it('payout webhook marks conversion COMPLETED')
  it('ledger entries are balanced (DEBIT = CREDIT + FEE)')
  it('audit log records all state transitions')
```

#### Phase 6 Done When
- [ ] All chaos tests GREEN
- [ ] All e2e tests GREEN against JuicyWay sandbox
- [ ] `npm test` exits 0
- [ ] Ledger is balanced on every test run

**STATUS: 🔴 PENDING**

---

### 🔴 PHASE 7 — Production Hardening

**Goal:** Harden all remaining audit findings before go-live.

| Item | Priority | File | Status |
|------|----------|------|--------|
| Circuit breaker (HEALTHY→DEGRADED→DOWN→HALF_OPEN) | P1 | `provider-orchestrator.ts` | 🔴 |
| Verify `apiKeyEncrypted` is AES-256-GCM not plaintext | P1 | `provider.repository.ts` | 🔴 |
| Rate limit response headers (RateLimit-Limit, Remaining, Reset) | P2 | `rate-limiter.middleware.ts` | 🔴 |
| Correlation ID null guard in controllers | P2 | All controllers | 🔴 |
| DEGRADED health state (latency threshold) | P2 | `health-check.service.ts` | 🔴 |
| ETH BigInt support | P3 | `constants.ts` + `money.ts` | 🔴 |

**STATUS: 🔴 PENDING**

---

## PART 4 — EXTERNAL DEPENDENCIES TRACKER

These are NOT code tasks. They must be resolved in parallel with coding.

| # | Item | Who | Needed For | Status |
|---|------|-----|------------|--------|
| E1 | JuicyWay sandbox API key (≥32 chars) | JuicyWay | Phase 2 live gate | ✅ HAVE IT |
| E2 | JuicyWay sandbox Business ID (≥32 chars) | JuicyWay | Phase 2 live gate | ✅ HAVE IT |
| E3 | JuicyWay sandbox base URL | JuicyWay | Phase 2 live gate | ✅ HAVE IT |
| E4 | JuicyWay webhook URL set in sandbox dashboard | You | Phase 2 live gate | 🔴 PENDING |
| E5 | ngrok/cloudflared running for local webhook receipt | You | Phase 2 live gate | 🔴 PENDING |
| E6 | Test TRC20 wallet address (Tron) | You | Phase 2 live gate | 🔴 PENDING |
| E7 | XanePay smart contract address + ABI + RPC URL | You | Phase 5 | 🔴 PENDING |
| E8 | JuicyWay production KYC approved | JuicyWay | Go-live | 🔴 PENDING |
| E9 | NGN float funded in JuicyWay production | You (real ₦) | Go-live | 🔴 PENDING |

---

## PART 5 — NGROK WEBHOOK SETUP (How-To)

### Step 1 — Install ngrok
```bash
# Option A: npm
npm install -g ngrok

# Option B: Download from https://ngrok.com/download
# Extract and move to PATH
```

### Step 2 — Authenticate ngrok (free account required)
```bash
# Sign up at https://ngrok.com (free)
# Get your authtoken from the dashboard
ngrok config add-authtoken YOUR_AUTH_TOKEN
```

### Step 3 — Start your engine locally
```bash
cd XaneApp/engine
npm run dev
# Engine starts on port 3001
```

### Step 4 — Start ngrok tunnel
```bash
ngrok http 3001
# Output will show something like:
# Forwarding  https://abc123def456.ngrok-free.app -> http://localhost:3001
```

### Step 5 — Set the webhook URL in JuicyWay dashboard
```
Your full webhook URL:
https://abc123def456.ngrok-free.app/api/v1/webhooks/juicyway

Go to JuicyWay sandbox dashboard → Settings → Webhooks → Add URL
Paste the above URL and save
```

### Step 6 — Add JuicyWay's sandbox IP to whitelist
JuicyWay sandbox likely uses different IPs than production. Ask JuicyWay support:
> "What are the outbound IP addresses for your sandbox webhook delivery?"
Add those to `JUICYWAY_WEBHOOK_IPS` in `infrastructure/config/constants.ts`

### Important: ngrok URL Changes on Restart (Free Plan)
Every time you restart ngrok, you get a new URL. You will need to update the JuicyWay dashboard each time.

**To avoid this:** Use ngrok's paid static domain feature, OR use cloudflared which gives a permanent tunnel:
```bash
# Cloudflared (free, permanent URL)
# Install: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
cloudflared tunnel --url http://localhost:3001
# First run creates a permanent subdomain at trycloudflare.com
```

---

## PART 6 — TEST WALLET SETUP (TRC20 / Tron)

For Phase 2 live gate you need a Tron wallet address to receive test USDT.

### Option A — TronLink Browser Extension (Recommended)
1. Install TronLink from Chrome Web Store
2. Create a new wallet (save seed phrase securely — do NOT use a real wallet with funds)
3. Your address starts with `T` — e.g. `TRWBqiqoFZysoAeyR1J35ibuyc8EvhUAoY`
4. This is the address you pass as `userDestination.address` in test requests

### Option B — TronScan Quick Address
Visit https://tronscan.org and generate a one-time test address. Only use for testing.

### Note on JuicyWay Sandbox
JuicyWay sandbox does NOT send real crypto to real addresses. The sandbox simulates the conversion and fires webhook events. You will see the conversion marked COMPLETED but no actual USDT arrives at the TRC20 address. Real USDT only moves in production with real funds.

---

## PROGRESS SUMMARY

```
PRE-WORK  ✅ DONE     (2026-05-17) — 4 bugs fixed
PHASE 1   🔴 PENDING  — Fee architecture refactor
PHASE 2   🔴 PENDING  — JuicyWay full extension + destination fields
PHASE 3   🔴 PENDING  — Conversion routing
PHASE 4   🔴 PENDING  — Multi-hop state machine
PHASE 5   🔴 PENDING  — Smart contract DEX adapter
PHASE 6   🔴 PENDING  — Chaos tests + E2E suite
PHASE 7   🔴 PENDING  — Production hardening
```
