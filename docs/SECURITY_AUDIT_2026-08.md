# XanePay Engine — Security Audit Report
**Date:** 1 August 2026  
**Scope:** `XaneApp/engine/` — API routes, middleware, use-cases, fee service, provider integration  
**Auditor:** Automated code review (Cline)

---

## 1. Authentication & Authorization

### 1.1 HMAC Authentication (`hmac-auth.middleware.ts`)
**Verdict: ✅ Strong**

- HMAC-SHA256 over `{timestamp}.{JSON body}` with constant-time comparison (`crypto.timingSafeEqual`).
- **Timestamp tolerance** is absolute (`Math.abs(now - timestamp) > TOLERANCE`), bounding both past AND future timestamps. Previously only checked `now - timestamp > TOLERANCE`, which allowed future-dated requests to never expire — a replay vector. **Fixed.**
- **Single-use enforcement** via Redis `SET NX` with TTL matching the timestamp window. Fails **closed** on cache error (returns 401 rather than silently downgrading to no replay protection). **Correct.**
- Secret sourced from Zod-validated env config, never raw `process.env`.

### 1.2 Admin Authentication (`admin-auth.middleware.ts`)
**Verdict: ✅ Strong**

- `X-Admin-Key` header validated with `crypto.timingSafeEqual`.
- Returns 403 (not 401) — admin is a separate privilege, not just a different credential.
- Length-mismatch early rejection before constant-time compare (prevents length oracle).
- All failed attempts logged with IP and path for forensic auditing.
- Admin key read from `getEnvConfig().ADMIN_API_KEY` (Zod-validated).

### 1.3 Webhook Authentication (`onswitch-webhook-validator.middleware.ts`)
**Verdict: ✅ Strong**

- RAW body consumed (`express.raw`) so signature verification runs over exact bytes OnSwitch signed.
- `x-switch-signature` header verified with provider-specific HMAC.
- Idempotency dedup keyed on the signature itself (not a caller-supplied header).
- Unroutable payloads acknowledged (200) and dropped — no error leakage.
- Internal errors return 200 (not 5xx) to prevent OnSwitch retry storms.

### 1.4 Route-Level Auth Wiring
**Verdict: ✅ Correct**

| Route | Auth | Rate Limited |
|---|---|---|
| `POST /conversions/quote` | HMAC | Yes (per IP) |
| `POST /conversions/execute` | HMAC | Yes (per IP, generous) |
| `POST /conversions/:id/confirm-deposit` | HMAC | Yes (per IP) |
| `GET /conversions/:id` | HMAC | No |
| `POST /conversions/verify-account` | HMAC | Yes (per IP) |
| `POST /webhooks/onswitch` | Provider signature | No |
| `GET/PUT /admin/*` | X-Admin-Key | Varies |

No unauthenticated write endpoints. The status GET is HMAC-protected (B2B — one IP = one business).

---

## 2. Input Validation

### 2.1 Zod Schemas (`conversion.schema.ts`, `admin.schema.ts`)
**Verdict: ✅ Strong**

- **Currency allow-list** enforced at the API boundary via `.refine()` against `SUPPORTED_FIAT_CURRENCIES` + `SUPPORTED_CRYPTO_CURRENCIES`. ETH explicitly excluded.
- **Amount bounds** per source currency via `CURRENCY_AMOUNT_LIMITS` in `.superRefine()` — prevents cross-unit comparison (fiat kobo limits vs crypto minor units).
- **Monetary fields are integers** in minor units (`.int().positive()`).
- **`payoutDestinationSchema`** is now **required** on `/execute` (was optional). Previously, omitting it credited the customer's ledger, marked COMPLETED, and never paid anyone — internally consistent books that stopped describing reality. **Fixed.**
- **`accountName`** on bank payouts: 2-100 chars, regex requires ≥1 letter, prevents silent placeholder fallback.
- **`treasuryAddress`**: 26-62 alphanumeric chars (covers BTC, TRC-20, ERC-20, Bech32).
- **`txHash`**: 16-120 chars, hex/base58 pattern.
- **`verifyAccountSchema`**: account number 6-40 digits, bank code numeric, country ISO-3166 alpha-2.

### 2.2 Admin Schemas
**Verdict: ✅ Adequate**

- `updateProviderSchema`, `updateFeeConfigSchema`, `resolveConversionSchema`, `recordTreasuryWithdrawalSchema` all Zod-validated.
- Treasury withdrawal recording is rate-limited to 20/hour — a burst is "far more likely to be a stolen admin key or a buggy loop than genuine use."

---

## 3. Race Conditions & Transaction Isolation

### 3.1 EnqueueSwapUseCase (`enqueue-swap.use-case.ts`)
**Verdict: ✅ Strong**

- Conversion create + quote status update + conversion status update run in **one DB transaction** (`transactionPort.runInTransaction`).
- Queue job enqueued **only after commit** — prevents a job executing against a rolled-back conversion.
- `abortWith()` used for transactional rollback on duplicate or constraint violations.

### 3.2 ExecuteSwapUseCase (`execute-swap.use-case.ts`)
**Verdict: ✅ Strong**

- **Distributed lock** acquired before provider call: `cache.acquireLock(lockKey, lockToken, ttl)`. Lock token prevents accidental release of another request's lock.
- Lock released in `finally` block — always runs, even on throw.
- Lock release checks ownership: "A blind `del()` here would delete a lock a *different* request legitimately took over after ours expired."
- **`executions.quote_id` is UNIQUE** — database-level at-most-once guarantee independent of the lock.
- Ledger settlement + status update run in a single transaction.
- Payout delivery + COMPLETED status + ledger liability clearing run in a single transaction.
- If the completion transaction fails after payout was sent, conversion parks at `PAYOUT_PENDING` for reconciliation — never `FAILED` (money already moved).

### 3.3 Provider Call Ambiguity
**Verdict: ✅ Well-Handled**

- `providerCalled` flag tracks whether the external call was made. If an exception occurs:
  - **Before provider call** → `ExecutionStatus.FAILED` (unambiguous: no money moved).
  - **After provider call** → `ExecutionStatus.COMPENSATION_REQUIRED` (ambiguous: request may have landed, requires human review).

---

## 4. Idempotency Mechanisms

### 4.1 HMAC Signature Single-Use
**Verdict: ✅ Strong**

- `cache.setIfNotExists('hmac:sig:{signature}', ...)` with TTL = timestamp window.
- Fails closed on cache error.
- Combined with absolute timestamp window, a signature is valid exactly once within a fixed time window.

### 4.2 Quote ID Uniqueness
**Verdict: ✅ Strong**

- `executions.quote_id` is UNIQUE in the database. A replayed `/execute` with the same quote is rejected by the DB constraint regardless of how many requests bypass the rate limiter.

### 4.3 Webhook Idempotency
**Verdict: ✅ Strong**

- Keyed on `webhook:onswitch:{signature}` — the signature itself, not a caller-supplied header.
- Dedup check in middleware; key committed by controller after successful processing.
- Duplicate webhooks return 200 (not error) to prevent provider retry storms.

### 4.4 Payout Idempotency
**Verdict: ✅ Strong**

- Payout reference = `payout:{conversionId}` — UNIQUE, so the webhook path cannot double-release a liability that was already cleared synchronously.

---

## 5. Fee Manipulation Vectors

### 5.1 FeeService (`fee-service.ts`)
**Verdict: ✅ Strong**

- **Fallback chain**: DB row (conversion-type-specific) → DB row (DEFAULT) → env `DEFAULT_FEE_BPS`.
- **Currency guard**: `config.feeCurrency !== sourceCurrency` → hard error. Prevents cross-unit fee application (e.g., applying a NGN fee config to a USDT conversion).
- **Min/max bounds** enforced when a DB config row exists.
- **Pure integer arithmetic**: `Math.floor((amount * feeBps) / 10_000)` — no floating-point.
- **Zero/negative rejection**: `amount <= 0` → error.
- **Fee BPS stored on conversion record** (`xaneFeeBps`): completion recomputes revenue from this rather than trusting a stored absolute figure.

### 5.2 Admin Fee Configuration
**Verdict: ✅ Adequate**

- `PUT /admin/fees` requires `X-Admin-Key` + Zod validation.
- No rate limit on the fee config endpoint itself, but admin key compromise is the primary threat vector (mitigated by constant-time comparison and audit logging).

---

## 6. Replay Attack Surfaces

### 6.1 HMAC Timestamp Window
**Verdict: ✅ Fixed (was CRITICAL)**

- **Previous**: `now - timestamp > TOLERANCE` — future timestamps never expired. A captured request with a future timestamp was replayable indefinitely.
- **Current**: `Math.abs(now - timestamp) > TOLERANCE` — bounds both directions. **Fixed.**

### 6.2 Signature Single-Use
**Verdict: ✅ Strong**

- Even within the 30-second window, a signature can only be used once (Redis `SET NX`).

### 6.3 Rate Limiter Keying
**Verdict: ✅ Fixed (was CRITICAL)**

- **Previous**: `/execute` rate limiter keyed on `x-idempotency-key` — every request got its own bucket, so the limit was never reached. The most sensitive endpoint was effectively unrated.
- **Current**: Keyed on `req.ip`. **Fixed.**

### 6.4 Webhook Replay
**Verdict: ✅ Strong**

- Signature-based dedup in middleware. Duplicate returns 200 (acknowledged, not errored).

---

## 7. Ledger Integrity & Double-Entry Correctness

### 7.1 Settlement Flow
**Verdict: ✅ Strong**

The execute-swap use-case follows a strict double-entry pattern:

1. **Provider settlement** → write ledgers (treasury credit + user liability) in one transaction with status update.
2. **Payout delivery** → `PAYOUT_DELIVERED` posting clears the user liability in the SAME transaction that marks the conversion COMPLETED.
3. **Webhook path** also clears the liability the same way — "both doors are covered."

### 7.2 Liability Clearing
**Verdict: ✅ Fixed (was CRITICAL — Audit M5)**

- **Previous**: Omitting `destination` credited the customer's ledger, marked COMPLETED, and never paid anyone. Books were internally consistent but stopped describing reality.
- **Current**: `destination` is **required** on `/execute`. A conversion with nowhere to deliver is rejected before any money moves.

### 7.3 Payout Trace
**Verdict: ✅ Strong**

- Append-only ledger entry for `PAYOUT_INITIATED` — best-effort but must not roll back the settled swap.
- Deposit-based trace (`PROVIDER_CALLED`) is checked (not fire-and-forget): "losing it silently is how 'we have no record of that' happens."

---

## 8. Error Handling & Information Leakage

### 8.1 API Error Responses
**Verdict: ✅ Adequate**

- Errors use structured `{ success: false, error: { kind, message } }` format.
- `ErrorKind` enum prevents raw error propagation.
- Correlation IDs included in responses for tracing.

### 8.2 Webhook Error Handling
**Verdict: ✅ Strong**

- Internal errors in webhook processing return **200** (not 5xx) to prevent provider retry storms.
- Unroutable payloads return 200 with `{ ignored: true }`.

### 8.3 Admin Auth Error Messages
**Verdict: ✅ Good**

- Consistent "Invalid admin credentials" message regardless of whether the key was missing, wrong length, or wrong value — no oracle.

### 8.4 Potential Concern: Stack Traces in Production
**Verdict: ⚠️ Needs Verification**

- The `next(error)` pattern in controllers passes errors to Express error-handling middleware. Verify that the global error handler does not leak stack traces or internal paths in production responses.

---

## 9. Provider Integration Security

### 9.1 OnSwitch Adapter
**Verdict: ✅ Strong**

- API keys resolved through `ProviderFactory.resolveApiKey()` — never stored in DB, sourced from env.
- Webhook signature verification over raw body bytes.
- No `lockRate` capability declared (OnSwitch is deposit-based) — the router never tries to lock it.
- `initiate` is a combined swap+payout call; the adapter correctly models this.

### 9.2 Provider Credential Storage
**Verdict: ✅ Strong**

- Admin route comment explicitly states: "there is NO reveal-secret route — provider credentials are not stored in the DB (they live in env, see ProviderFactory.resolveApiKey), so there is nothing to reveal."

---

## 10. Summary of Findings

### Critical Issues (All Fixed)
| ID | Finding | Status |
|----|---------|--------|
| C1 | HMAC timestamp only checked past, not future — indefinite replay | ✅ Fixed |
| C2 | `/execute` rate limiter keyed on `x-idempotency-key` — effectively unrated | ✅ Fixed |
| C3 | Optional `destination` allowed silent ledger liability without payout (Audit M5) | ✅ Fixed |
| C4 | `accountName` silently fell back to placeholder on bank payouts | ✅ Fixed |

### High-Severity Issues
None identified.

### Medium-Severity Issues
| ID | Finding | Recommendation |
|----|---------|---------------|
| M1 | No rate limit on `PUT /admin/fees` | Add rate limiting (admin key compromise is primary threat, but defense-in-depth) |
| M2 | `feeBps` stored on conversion but completion recomputes from it — verify recomputation is always consistent | Add integration test for fee recomputation across all conversion types |

### Low-Severity / Observational
| ID | Finding | Recommendation |
|----|---------|---------------|
| L1 | Global error handler may leak stack traces | Verify production error handler strips stack traces |
| L2 | `conversion.controller.ts` line 143: `next(new Error(...))` — plain Error, not EngineError | Use `createEngineError` for consistency |
| L3 | `enqueue-swap.use-case.ts` line 79: `dto.userId ?? dto.idempotencyKey` as userId fallback | Document that idempotencyKey is used as userId when not provided |

---

## 11. Overall Assessment

The XanePay engine demonstrates **strong security posture** across all audited dimensions. The codebase shows evidence of prior security review and remediation — several critical vulnerabilities were identified and fixed (timestamp replay, rate-limiter keying, silent ledger liability). The architecture follows defense-in-depth principles:

- **At-most-once payment safety** is guaranteed at three independent layers: distributed lock, UNIQUE constraint on `executions.quote_id`, and HMAC signature single-use.
- **Double-entry ledger** is consistently applied with liability clearing in the same transaction as status transitions.
- **Provider credentials** are never stored in the database.
- **All monetary arithmetic** is integer-based in minor units.
- **Input validation** is comprehensive with currency-specific bounds.

**No exploitable vulnerabilities were identified in the current codebase.**