# Gap Report — CTO Audit findings vs. the REAL `engine` branch (v0.11.0)

> ## ⚠️ THIS FILE IS A SNAPSHOT OF 2026-07-11 AND IS NOW STALE ITSELF
>
> Verified against `engine` at `19a4fc3a` on **2026-07-22**. **Every finding below marked "STILL
> OPEN" has since been fixed:**
>
> | Finding | Says below | Actually |
> |---|---|---|
> | **B5** — payout amount from cache | ❌ still open | ✅ **fixed** — the quote is loaded from Postgres and Zod-validated; no `cache.get<QuoteRecord>` remains |
> | **B9** — lock TTL / ownership | ❌ still open | ✅ **fixed** — `releaseLock(key, token)`, a Lua compare-and-delete (`execute-swap.use-case.ts`) |
> | **B2** ledger schema, **B3** `/execute` gate, **B7** append-only, **B6b** balance recon | ❌ open | ✅ all built (migrations 12/13, triggers, hash chain) |
>
> The irony is the point: this file exists **because** the CTO audit was written against a stale
> checkout, and it then became stale in exactly the same way. The 2026-07-22 money audit re-derived
> B5 and B9 as open from reading this file, and both cost time to disprove.
>
> **Read the code, not the tracker.** For current state see `ENGINE_MONEY_AUDIT_2026-07.md` and
> `XaneApp/engine/docs/FINDINGS_REGISTER.md`. Everything below is preserved as the historical record
> of what was true on 2026-07-11.

**Date:** 2026-07-11
**Why this exists:** The CTO audit (`CTO_AUDIT_AND_BUILD_PLAN.md`) was written against a **stale
checkout** — the detached-HEAD commit `42c8da35` (tag **v0.8.0**, "Phase 8 Business Logic
Orchestration"). That commit is on **no branch** and is a **divergent dead-end**.

The real engine lives on the **`engine` branch (v0.11.0)**. Both lines forked from `ab249893`
("Phase 7 Complete"). Since the fork:

| Line | Commits since fork | Test files |
| --- | --- | --- |
| `engine` (real, v0.11.0) | 16 | 61 (~458 tests) |
| detached HEAD (v0.8.0) — what the audit read | 1 | 27 |

> ⚠️ The `v0.8.0` tag was *created* on 2026-06-22, **later in wall-clock time** than `v0.11.0`
> (2026-06-16), even though it is far behind in content. That is the trap: it looks newest by date.
> Always `git switch engine` before assessing engine state.
>
> The audit's own text contains the tell: *"docs claim min-32 was applied; the code says otherwise."*
> That is the signature of reading stale code alongside the real branch's docs.

**Consequence:** several audit findings are **already fixed** on `engine`. Others are **genuinely
still open**. This report is the corrected, evidence-based status. Trust THIS file over the audit's
line numbers.

---

## Status of each finding on the REAL `engine` branch

Legend: ✅ already fixed · ⚠️ partially fixed · ❌ genuinely still open

### ✅ B1 — "The distributed lock is not a lock"  → **ALREADY FIXED**
`execute-swap.use-case.ts:111` already uses `this.cache.setIfNotExists(lockKey, {...}, 30_000)` —
a real Redis `SET NX PX` — and returns `DUPLICATE_REQUEST` when the lock is held (`:119`).
**The audit's single most dangerous finding does not apply to the real branch.**

### ❌ B9 — Lock TTL and ownership → **STILL OPEN**
The lock is *acquired* atomically but *released* with `await this.cache.del(lockKey)`
(`execute-swap.use-case.ts:391`) — **no ownership token**. If our lock expires (30s TTL) and another
process legitimately acquires it, our `finally` block deletes **their** lock. Needs the Lua
compare-and-delete release.

### ⚠️ B2 — "The double-entry ledger is neither double-entry nor atomic" → **HALF FIXED**
- **Atomicity: ✅ fixed.** `ILedgerPort.recordEntryPair(debit, credit, ctx)` — *"both entries commit
  or neither does"* — participates in the caller's transaction, and the post-provider commit **does
  check the Result** (`if (isErr(pairResult)) throw` → rolls the transaction back).
- **Schema: ❌ still broken.** `04_create_ledger_entries.ts` is still
  `entry_type VARCHAR(50)` + `data JSONB`. There is **no direction, account, amount, or currency
  column**. So the one query a double-entry ledger exists to answer —
  `SUM(debits) = SUM(credits)` — **still cannot be run**. No balances, no reconstruction, no hash
  chain, no tamper-evidence.

**The core complaint ("books that cannot prove anything") STANDS.**

### ⚠️ B3 — DB-level idempotency + state machine → **HALF FIXED**
- ✅ `conversions.idempotency_key` is **UNIQUE NOT NULL** (`03_create_conversions.ts:12`) — a real
  database gate. But it guards **conversion creation** (`/initiate`), keyed by a caller-supplied key.
- ✅ Statuses use the `ConversionStatus` enum, not the raw string literals of the stale tree.
- ❌ There is **no `executions` table** and **no insert-first gate on the `/execute` path keyed by
  quote**. Nothing rejects a conversion already in `PROVIDER_CALLED`/`COMPLETED` *before* calling the
  provider. The only protection at `/execute` is the Redis lock — which expires after 30s and is not
  token-safe (B9).

**The audit's crash-retry-repays scenario still stands** (narrower than on the stale tree, but real).

### ✅ B4 — "Chain hashes are fabricated" → **APPEARS FIXED**
The `transaction_hash || json.data.id` fabrication is **gone** from the JuicyWay adapter on this
branch, and migration `11_add_payout_destination.ts` exists. Worth one more read when porting, but
the fabrication pattern is absent.

### ❌ B5 — "The payout amount comes from a cache" → **STILL OPEN**
`execute-swap.use-case.ts:68` still reads the quote from Redis **first**
(`await this.cache.get<QuoteRecord>('quote:' + quoteId)`), falling back to Postgres only on a miss
(`:72`). Money amounts are still sourced from a volatile, unauthenticated cache. (It is typed as
`QuoteRecord` rather than `any` — marginally better than the stale tree — but there is still no
schema validation at the money boundary.)

### ⚠️ B6 — Reconciliation → **HALF EXISTS**
- ✅ **Stuck-transaction / provider-poll recovery is BUILT**: `reconciliation.service.ts`
  (`runNightlyReconciliation()`), `reconciliation.processor.ts` (BullMQ), and migration
  `08_create_reconciliation_runs.ts`. It is provider-aware and resolves stuck conversions forward
  (`PENDING` → FAILED; `PROCESSING` → check ledger then poll; `PROVIDER_CALLED` → poll → complete /
  fail on 404 / skip on 5xx).
- ❌ **Balance reconciliation is absent**: no balance reconstruction, no drift tolerance against the
  provider's reported balance — **because the ledger schema (B2) cannot compute a balance.**

### ❌ B7 — "Append-only is a comment, not a constraint" → **STILL OPEN**
**No append-only trigger and no `REVOKE` exists in any migration on this branch.** Any code path or
compromised credential can still `UPDATE`/`DELETE` ledger rows and rewrite history.

### ⚠️ B8 — Unchecked Results on the money path → **MIXED**
- ✅ The **critical** post-provider ledger write **is** checked (`isErr(pairResult)` → throw →
  rollback), and the file explicitly documents *"NEVER MARK FAILED AFTER PROVIDER SETTLES"*.
- ❌ Pre-provider writes remain unchecked: `quoteRepo.updateStatus` (`:131`),
  `conversionRepo.updateStatus` (`:134`), `audit.log` (`:137`), and `:164` (`PROVIDER_CALLED`).
  Lower severity (no money has moved yet), but still the B8 class of bug.

---

## Bottom line

| Finding | Real branch status | Is the Day-1 work I built still needed? |
| --- | --- | --- |
| B1 lock acquire | ✅ fixed | **No** — redundant |
| B4 hash fabrication | ✅ fixed | **No** — redundant |
| B6 stuck-txn / poll recovery | ✅ exists | **No** — redundant |
| B2 ledger **atomicity** | ✅ fixed | No |
| B3 **conversion-creation** idempotency | ✅ exists | No |
| **B2 ledger SCHEMA** (prove the books) | ❌ open | **YES** — two-table ledger, balances, hash chain, reconstruction |
| **B3 `/execute` at-most-once gate** | ❌ open | **YES** — `executions` table + insert-first gate |
| **B5 DB-first + validation** | ❌ open | **YES** |
| **B7 append-only** | ❌ open | **YES** — triggers + REVOKE |
| **B9 token-safe release** | ❌ open | **YES** — Lua compare-and-delete |
| B8 pre-provider writes | ⚠️ partial | **YES** (partial) |
| B6 **balance** reconciliation | ❌ open | **YES** — unlocked by the B2 schema |

Roughly **60% of the Day-1 work is still genuinely needed** on the real branch. The redundant 40%
(B1 acquire, B4, B6 stuck-txn) should be **dropped, not ported**.

## Porting caveat — this is a re-implementation, not a cherry-pick

My code was written against the stale tree's *simple* `ExecuteSwapUseCase` (~130 lines: quoteId +
treasuryAddress). The real one is **394 lines** with a `conversionId`, a **payout delivery leg**
(`PAYOUT_PENDING`, `sendPayout`), transaction contexts (`ctx`), a circuit breaker, and HMAC/rate
limiting around it. The *migrations also collide*: my `08_rebuild_ledger` / `09_create_executions`
clash with the branch's existing `08_create_reconciliation_runs` / `09_add_fee_currency` and must be
renumbered to **`12_` / `13_`**.

**Recommended sequence on `engine`:**
1. `12_rebuild_ledger.ts` — two-table ledger + accounts + append-only triggers (B2 schema + B7).
2. `LedgerService` + routes-as-data + hash chain + `verifyChain()` / `reconstructBalances()`.
3. `13_create_executions.ts` + insert-first gate on `/execute` (B3).
4. Token-safe `releaseLock` (B9); DB-first + validated quote load (B5); check pre-provider Results (B8).
5. Extend the **existing** `ReconciliationService` with balance reconstruction + drift tolerance (B6
   second half) — do **not** rebuild the stuck-transaction half.
6. Re-baseline against the branch's ~458-test suite.

## Repo hygiene (confirmed)
- `private` → `https://github.com/praiseunite/xanepay_private.git` ← **push here only**
- `origin` → `https://github.com/xaneApp/XaneApp.git` ← **never push here**
- **Nothing has been pushed.** The Day-1 work is preserved uncommitted on local branch
  `engine-money-integrity-wip` (based on the dead-end v0.8.0 commit).
- The dev Postgres volume was reset during this session; re-run migrations on `engine` to rebuild it.
