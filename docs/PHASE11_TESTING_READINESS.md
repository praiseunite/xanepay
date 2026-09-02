# XanePay Engine — Phase 11 Testing Readiness & Architecture Decisions

> ## ⚠️ SUPERSEDED — DO NOT PLAN AGAINST THIS FILE
>
> **Retired 2026-07-22.** Every gate below is framed around **JuicyWay**, whose API is dead and which
> can never be integrated. The live rail is **OnSwitch** (deposit-based, not custody), so the whole
> 11-C/11-D/11-E sequence describes a system that no longer exists. The unchecked ☐ boxes do not mean
> the work is outstanding — most of it shipped, against a different rail.
>
> Genuinely still true, and the only part worth carrying forward:
> - **No conversion has ever reached `COMPLETED` against a real rail.** The OnSwitch sandbox returns
>   a placeholder deposit address (`0x…sandbox`) that cannot receive a transfer. A **funded staging
>   key** remains the hard gate on proving the money path with real money.
> - The "Architecture Principles to Preserve" section at the bottom still holds — with one addition
>   from the 2026-07-22 audit: **one number, one source.** Two independently-edited values describing
>   the same money will drift, and nothing will notice (see M1).
>
> Current state: `ENGINE_MONEY_AUDIT_2026-07.md` and `XaneApp/engine/docs/FINDINGS_REGISTER.md`.

**Author:** System Architect  
**Status:** SUPERSEDED — historical record only  
**Last Updated:** 2026-05-16 (superseded 2026-07-22)

---

## How to read this document

Each phase has a **Gate Checklist** — every item must be ✅ before you proceed to the next phase.
Items are split into: **You own this** (code/config you control) vs **External dependency** (requires a third party or real money).

---

## Phase 11-A: Unit Tests (Zero External Dependencies)

> Run entirely in memory. No database, no Redis, no JuicyWay account needed.

### Gate Checklist

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | Jest configured (`jest.config.ts`, `ts-jest`) | You | ☐ |
| 2 | `fee-rules.ts` — test BPS math, overflow edge cases | You | ☐ |
| 3 | `money.ts` value object — add, subtract, multiply, currency mismatch guard | You | ☐ |
| 4 | `conversion.ts` state machine — all valid transitions, all invalid transitions | You | ☐ |
| 5 | `applyFeeToRate` — test with BTC max rate (8×10⁹), 0 bps, 10000 bps | You | ☐ |
| 6 | `hmac-auth.middleware.ts` — valid sig, expired timestamp, tampered body | You | ☐ |
| 7 | `webhook-validator.middleware.ts` — bad IP, bad checksum, duplicate, first-time | You | ☐ |
| 8 | `error-handler.middleware.ts` — confirm DB errors are masked, validation errors are not | You | ☐ |
| 9 | Result monad helpers (`ok`, `err`, `isOk`, `isErr`, `map`, `flatMap`) | You | ☐ |

### What you need

- Nothing. Pure TypeScript. Run with `npm test`.

---

## Phase 11-B: Integration Tests (Local Infrastructure)

> Real PostgreSQL + Redis running locally via Docker. No JuicyWay, no real money.

### Gate Checklist

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | `docker-compose.yml` with postgres + redis services | You | ☐ |
| 2 | `.env.test` pointing to test DB + Redis | You | ☐ |
| 3 | DB migrations run clean on fresh test database | You | ☐ |
| 4 | `ConversionRepository` — create, findById, idempotency key unique constraint | You | ☐ |
| 5 | `LedgerRepository` — double-entry write, balance query | You | ☐ |
| 6 | `AuditLogRepository` — append-only, query by actor/feature/date range | You | ☐ |
| 7 | `RedisCacheRepository` — set, get, setIfNotExists (lock), increment + expire | You | ☐ |
| 8 | `InitiateConversionUseCase` with a mock provider (returns fixed rate) | You | ☐ |
| 9 | `ExecuteSwapUseCase` — success path, duplicate lock rejection (concurrent calls) | You | ☐ |
| 10 | `WebhookHandlerService` — success event updates status + ledger, duplicate skipped | You | ☐ |

### What you need

- Docker Desktop installed and running
- `docker-compose up -d` to spin up postgres + redis
- No JuicyWay credentials. Use a mock `IFiatCryptoProviderPort` that returns canned responses.

---

## Phase 11-C: JuicyWay Sandbox Integration

> Real JuicyWay API calls but in their sandbox/staging environment. No real money moves.

### Gate Checklist

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | JuicyWay sandbox account created | **JuicyWay** | ☐ |
| 2 | Sandbox `JUICYWAY_API_KEY` (≥32 chars) obtained | **JuicyWay** | ☐ |
| 3 | Sandbox `JUICYWAY_BUSINESS_ID` (≥32 chars) obtained | **JuicyWay** | ☐ |
| 4 | Sandbox `JUICYWAY_BASE_URL` confirmed (staging URL) | **JuicyWay** | ☐ |
| 5 | `GET /exchange/fx/rate` — confirm response shape matches `JuicyWayFxRateResponse` | You | ☐ |
| 6 | `POST /exchange/fx/rate/{id}/lock` — confirm rate lock works | You | ☐ |
| 7 | `POST /exchange/fx/convert` — confirm swap execution in sandbox | You | ☐ |
| 8 | JuicyWay webhook delivery confirmed in sandbox (IP whitelist may differ) | You + JuicyWay | ☐ |
| 9 | Sandbox webhook IPs added to `JUICYWAY_WEBHOOK_IPS` constant | You | ☐ |
| 10 | Float→integer conversion validated end-to-end (e.g. 0.6667 USDT = 666700 minor units) | You | ☐ |

### What you need

- JuicyWay developer/sandbox access (contact their team or use self-service portal)
- A public URL for webhook delivery (use `ngrok` or `cloudflared tunnel` for local dev)
- **You do NOT need real NGN** for sandbox — JuicyWay's sandbox simulates balances

---

## Phase 11-D: E2E Tests (Full Stack, Simulated Flow)

> Everything wired together. Uses sandbox credentials + local DB + Redis. No real money.

### Gate Checklist

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | All Phase 11-A, 11-B, 11-C gates cleared | You | ☐ |
| 2 | E2E test: POST /quote → POST /execute → GET /conversions/:id = COMPLETED | You | ☐ |
| 3 | E2E test: POST /execute with same idempotency key twice → second returns existing record | You | ☐ |
| 4 | **Chaos test: two identical /execute requests in parallel** → only one acquires Redis lock, other gets 409 | You | ☐ |
| 5 | E2E test: webhook delivery → conversion status updated → ledger entries written | You | ☐ |
| 6 | E2E test: rate limit hit (>10 requests/10s on /execute) → 429 returned | You | ☐ |
| 7 | E2E test: invalid HMAC signature → 401 | You | ☐ |
| 8 | E2E test: expired quote → 422 or 400 (not executed) | You | ☐ |
| 9 | Admin API: register provider, update fee, query audit log | You | ☐ |
| 10 | Reconciliation service: mark stuck conversions | You | ☐ |

### Chaos Test (Priority)

The most important test in this phase:

```typescript
// Prove the distributed lock works
it('should reject a duplicate concurrent execution', async () => {
  const [result1, result2] = await Promise.all([
    api.post('/execute', { idempotencyKey: 'same-key' }),
    api.post('/execute', { idempotencyKey: 'same-key' }),
  ]);
  const statuses = [result1.status, result2.status].sort();
  expect(statuses).toEqual([200, 409]); // one wins, one loses
});
```

---

## Phase 11-E: Production Readiness (Real Money)

> First real NGN flows through the system. This is your first live transaction.

### Gate Checklist — HARD GATES (do not skip)

| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | All previous phases cleared | You | ☐ |
| 2 | **JuicyWay production account fully KYC'd** | **JuicyWay / You** | ☐ |
| 3 | **NGN float funded in JuicyWay account** (minimum float for conversion liquidity) | **You (real money)** | ☐ |
| 4 | Production `JUICYWAY_API_KEY`, `BUSINESS_ID`, `BASE_URL` set in prod env | You | ☐ |
| 5 | **Fiat PSP integrated** (Paystack or Monnify) to receive NGN from end users | You | ☐ |
| 6 | Fiat PSP → JuicyWay funding flow documented and tested | You | ☐ |
| 7 | Production PostgreSQL (not local) — backups configured | DevOps | ☐ |
| 8 | Production Redis — persistence enabled (AOF or RDB) | DevOps | ☐ |
| 9 | SENTRY_DSN configured for error tracking | You | ☐ |
| 10 | Reconciliation cron running (detect stuck conversions) | You | ☐ |
| 11 | First live test: ₦500 → USDT (manual, monitored) | You | ☐ |
| 12 | Verify ledger entries, audit log, and conversion record after live test | You | ☐ |

---

## Architecture Decision: JuicyWay as Fiat Receiver

**Question you raised:** Can JuicyWay also *receive* the NGN from our users, instead of using a separate PSP like Paystack or Monnify?

### Current Architecture (Two-PSP Model)

```
User sends NGN
    │
    ▼
[Paystack / Monnify]  ← Fiat PSP receives NGN, issues webhook
    │ settlement
    ▼
[JuicyWay Float]      ← Pre-funded float converts NGN → USDT
    │ USDT
    ▼
[User Wallet]
```

**Problem:** This requires maintaining a pre-funded float in JuicyWay, manual top-up operations, and a separate Paystack/Monnify integration just for the deposit step.

### Proposed Architecture (JuicyWay-First Model)

**If JuicyWay supports NGN collection accounts (virtual accounts):**

```
User sends NGN
    │
    ▼
[JuicyWay Virtual Account]  ← JuicyWay issues a virtual account per user
    │ credit event (webhook)
    ▼
[Engine: Webhook Handler]   ← Triggers conversion automatically on credit
    │
    ▼
[JuicyWay Conversion API]   ← Same API we already use
    │ USDT
    ▼
[User Wallet]
```

**Advantages:**
- Single vendor for both fiat receipt and conversion (fewer integration points)
- No pre-funded float management — JuicyWay uses the received NGN directly
- Simpler reconciliation (one ledger per provider)
- Fewer moving parts = fewer failure modes

**Risks & Open Questions:**
1. Does JuicyWay offer NGN virtual accounts for business clients? **→ Must confirm with JuicyWay**
2. What is JuicyWay's uptime SLA? Single-vendor dependency is a concentration risk
3. Regulatory: receiving NGN from retail users requires CBN licensing — does JuicyWay's license cover this?
4. If JuicyWay goes down, both fiat receipt AND conversion fail simultaneously

**Recommendation:**
Start with JuicyWay-first if they support it — it is architecturally simpler for the MVP. Add Paystack/Monnify as a fallback PSP in Phase 2. The engine's `IFiatCryptoProviderPort` abstraction already supports swapping providers without changing business logic.

### Action Required

- [ ] Email JuicyWay: "Do you provide NGN virtual accounts or collection accounts for business clients? We want users to send NGN to a JuicyWay-issued account and trigger automatic conversion."
- [ ] If yes: design a new `IFiatDepositPort` in the engine (separate from `IFiatCryptoProviderPort`)
- [ ] If no: proceed with Paystack/Monnify integration for fiat receipt

---

## Outstanding Code Fixes (Completed This Session)

These bugs were identified in the Phase 10.5 audit and fixed:

| Bug | File | Severity | Fixed |
|-----|------|----------|-------|
| Weak API key env validation (min 1 char) | `infrastructure/config/env.ts` | HIGH | ✅ Increased to min 32 chars |
| Unsafe `Number()` on DB BigInt columns | `conversion.repository.ts` | HIGH | ✅ `parseSafeBigInt()` with `isSafeInteger` guard |
| Webhook idempotency key committed before processing | `webhook-validator.middleware.ts` + `webhook.controller.ts` | HIGH | ✅ Key now committed after processing in controller |

---

## Remaining Open Items (Not Yet Fixed)

These are architectural improvements that require more design work:

| Item | Priority | Description |
|------|----------|-------------|
| Circuit breaker for JuicyWay | P1 | `HEALTHY → DEGRADED → DOWN → HALF_OPEN` based on consecutive failures |
| Rate limit response headers | P2 | Add `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` to 429 responses |
| Provider credential encryption audit | P1 | Verify `apiKeyEncrypted` and `webhookSecretEncrypted` columns are actually AES-256-GCM encrypted, not plaintext |
| Correlation ID null guard | P2 | Validate `getCorrelationId()` is non-null before casting in controllers |
| ETH BigInt support | P3 | Implement `BigInt` arithmetic to unblock ETH/wei conversions |
| Fiat PSP integration | P1 | Paystack or Monnify for NGN receipt (or confirm JuicyWay handles it) |
| DEGRADED health state detection | P2 | JuicyWay health check only knows HEALTHY vs DOWN — add latency threshold for DEGRADED |

---

## Architecture Principles to Preserve

As the system grows, these invariants must never be broken:

1. **No floats in the domain.** Every amount is an integer in minor units. Floats exist only in `JuicywayAdapter` at the external boundary.
2. **State machine only.** `ConversionStatus` transitions are gated by `Conversion.transition()`. Nothing writes status directly to the DB without going through the state machine.
3. **Lock before act.** Any operation that calls an external provider MUST hold the Redis lock for that `conversionId`.
4. **Double-entry always.** Every debit has a credit. If you write one ledger entry, you write two.
5. **Idempotency at DB level.** The `UNIQUE(idempotency_key)` constraint is the last line of defence. Never remove it.
6. **Error masking at the boundary.** DB and cache errors must never leak to the client. The `ErrorKind` enum drives masking in `error-handler.middleware.ts`.
7. **Adapters are replaceable.** Business logic depends on ports, not adapters. Adding Monnify or Paystack should require zero changes to any use case.
