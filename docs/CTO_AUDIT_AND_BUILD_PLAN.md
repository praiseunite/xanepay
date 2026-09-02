# XanePay Engine — CTO Audit, Money-Integrity Solutions, Pluggable Providers & 4-Day Ship Plan

> **Permanent reference file.** Items get ticked ✅ as completed. Progress is tracked in the
> [Execution Tracker](#execution-tracker) at the bottom.

---

## THE PROBLEM CASE WE ARE SOLVING

JuicyWay is ending API support. Direction chosen: **hybrid** — rent a payout rail now
(Rapyd primary, Thunes standby), keep self-custody treasury as a future drop-in adapter behind
the same port. Engine stays a separate, self-contained microservice.

A CTO-level scan of `XaneApp/engine/` found **critical money-integrity bugs**, each explained
below line by line with the financial consequence and the exact remediation. The NombaVault PRD
(built on Formance, Blnk, Midaz — three production open-source ledgers) supplies proven patterns
adopted as solutions.

---

## Context

JuicyWay is ending API support. Direction chosen: hybrid — rent a payout rail now (Rapyd primary,
Thunes standby), keep self-custody treasury as a future drop-in adapter behind the same port.
Engine stays a separate, self-contained microservice.

---

## PART 1 — AUDIT FINDINGS: PROBLEM EXPLAINED LINE BY LINE → WHAT MUST BE DONE

### B1 (CRITICAL) — The "distributed lock" is not a lock → the same conversion can pay out twice

**The problem, line by line.** `src/application/use-cases/execute-swap.use-case.ts`:

- Line 53: `const lockKey = 'lock:' + quoteId;` — one lock key per quote. Correct intent.
- Line 54: `const lockResult = await this.cache.set(lockKey, { status: 'processing' }, 30);` — this
  is where it breaks. It calls `set`, and in `redis-cache.repository.ts:26-41` `set()` executes a
  plain Redis `SET`/`SETEX`. A plain `SET` always succeeds — it doesn't care whether the key
  already exists; it just overwrites it and returns OK.
- Lines 55-57: `if (isErr(lockResult)) return err(... 'Transaction already in progress' ...)` —
  this error branch only fires if Redis itself is down, never because another request holds the
  lock. The code believes a successful set means "I acquired the lock." It actually means "Redis
  is alive." Those are completely different guarantees.
- A grep of the entire `src/` tree confirms `setIfNotExists` / `SETNX` / `NX` appears nowhere. The
  atomic check-and-claim primitive that makes a lock a lock was never built.

**Why this loses money — concrete scenario.** A user's app double-taps "Convert" (or a mobile
network retry fires the same request twice). Both requests hit `/execute` for quote Q1
(₦500,000 → USDT) within the same 50 milliseconds:

- Request A runs `SET lock:Q1` → Redis replies OK → A believes it holds the lock.
- Request B runs `SET lock:Q1` → Redis replies OK (it just overwrote A's value) → B also believes
  it holds the lock.
- Both proceed to line 74 and both call `providerPort.executePayout(...)`.
- The provider receives two payout instructions and sends $650 worth of USDT twice. The user
  received ₦500,000 worth of service and got ₦1,000,000 of crypto. That ₦500,000 is gone.

At 100 conversions/day and even a 1% duplicate-request rate, this is a guaranteed, silent, daily
loss. This is the single most dangerous line in the codebase.

**What must be done (as the financial expert):**

- Add two methods to `ICachePort`: `setIfNotExists(key, value, ttlMs): Result<boolean>` implemented
  as Redis `SET key value NX PX ttlMs` — atomic "claim only if free" — and
  `releaseLock(key, token): Result<boolean>` implemented as a Lua script that deletes the key only
  if its value equals our token (so we can never delete a lock another process now legitimately
  holds after ours expired).
- In `ExecuteSwapUseCase`: generate a random `lockToken` (crypto-random UUID), call
  `setIfNotExists`. If it returns false, return `err(DUPLICATE_REQUEST)` → HTTP 409. Do not
  proceed. In `finally`, call `releaseLock(lockKey, lockToken)`.
- Write the test FIRST (RED): fire two concurrent acquisitions of the same key — assert exactly one
  wins. This chaos test stays in CI forever; if anyone ever regresses the lock, the build fails.
- Understand the layering: the Redis lock is the fast guard. It is not the last guard — that's B3's
  database constraint. Financial systems never rely on a single guard for a double-spend.

### B2 (CRITICAL) — The "double-entry ledger" is neither double-entry nor atomic → books that cannot prove anything

**The problem, line by line.** `execute-swap.use-case.ts`, after the payout succeeds:

- Lines 89-96: first `this.ledgerPort.recordEntry({... entryType: PROVIDER_CALLED, data: { currency, amount, quoteId } ...})`
  — the "debit". Note: `await` with no check of the returned Result. If this insert fails, the code
  doesn't know and doesn't care.
- Lines 99-106: second `recordEntry({... entryType: TREASURY_CREDITED ...})` — the "credit". Same:
  result ignored.
- These are two separate database inserts with no transaction around them
  (`ledger.repository.ts:18-36` — each `recordEntry` is its own standalone
  `db('ledger_entries').insert(...)`).
- Line 109: `await this.quoteRepo.updateStatus(quoteId, 'COMPLETED');` — runs regardless of whether
  either ledger insert succeeded.

Now the schema, `migrations/04_create_ledger_entries.ts`:

- Line 13: `entry_type VARCHAR(50)` — a label, but no direction column (DEBIT/CREDIT), no account
  column (whose money?), no amount column, no currency column.
- Line 14: `data JSONB` — the amount lives inside a free-form JSON blob.

**Why this destroys the books — concrete scenario.** Double-entry accounting has one job: at any
moment, `SUM(debits) = SUM(credits)` per currency, provable by one SQL query. This schema cannot
run that query at all — the amounts are inside JSONB with no enforced shape (and indeed
`check-ledger.js` already reads a JSONB field, `providerFeeMinorUnits`, that the use-case never
writes — the drift has already happened). Worse, the writes aren't atomic: the process crashes
(deploy, OOM, power) between line 96 and line 99 → the debit exists, the credit doesn't. Money left
the books on one side and never arrived on the other. In an audit, a dispute, or a reconciliation
against the provider, you cannot answer "where is the user's ₦500,000?" — and a payment company
that cannot answer that question is finished. And because the Results are unchecked, the most
dangerous case is silent: payout succeeded, both ledger inserts failed, status says COMPLETED —
real money moved with zero record.

**What must be done:**

- Rebuild the ledger on the NombaVault/Midaz two-table model (migration `08_rebuild_ledger.ts`):
  - `ledger_transactions` — the parent financial event: `reference UNIQUE NOT NULL` (idempotency —
    the same event can never post twice), `type`, `status`, `metadata JSONB`.
  - `ledger_operations` — the legs: `transaction_id FK`, `account_alias VARCHAR NOT NULL`,
    `direction CHAR NOT NULL CHECK (direction IN ('DEBIT','CREDIT'))`,
    `amount_minor BIGINT NOT NULL CHECK (amount_minor > 0)`, `currency`, `balance_before BIGINT`,
    `balance_after BIGINT`, `hash`, `prev_hash`. Integer minor units per rule R1 — never floats.
- Seed system accounts so every unit of value always has an address: `@external/NGN` (the outside
  world), `@provider:rapyd:USDT` (our funded balance at the provider — the "treasury account"),
  `@user:{id}:{ccy}` (our obligation to a user), `@fees:{ccy}` (our revenue), `@suspense:{ccy}`
  (anything unmatched — money NEVER disappears; it goes to suspense and waits for a human).
- Seed `transaction_routes` — every posting type defined as data upfront (PROVIDER_PAYOUT:
  DEBIT `@provider:X` / CREDIT `@user:{id}`; FEE_COLLECTED; REVERSAL; REFUND; SUSPENSE_IN/OUT).
  Model refunds and reversals on day one: books go unbalanced the first time reality produces a
  refund you never modeled.
- One `LedgerService.post(routeName, params)` that inserts parent + all legs + updates balances
  inside a single Knex transaction — all or nothing. Callers must check the Result: if the payout
  succeeded but the ledger write failed, the conversion enters COMPENSATION_REQUIRED and pages a
  human. It is never silently COMPLETED.
- Balance reconstruction endpoint (Blnk): recompute `SUM(credits) − SUM(debits)` per account from
  raw operations and compare to the stored balance → `{stored, computed, difference, flagged}`.
  This is the "prove the books" button — for you, an auditor, or an investor.

### B3 (CRITICAL) — No database-level idempotency and no state machine on execution → a crash-retry pays the user again

**The problem, line by line.**

- `i-quote-repo.port.ts:14`: `updateStatus(id: QuoteId, status: string)` — status is a free string.
  Any code can write any status in any order.
- `execute-swap.use-case.ts:61`: `updateStatus(quoteId, 'PENDING_EXECUTION');` line 81: `'FAILED'`;
  line 109: `'COMPLETED'` — raw literals. The carefully-built `Conversion.transition()` state
  machine (`domain/value-objects/conversion.ts`) is never called in any use-case (grep-verified).
  Rule R5 exists on paper only.
- There is no `executions` table, and no `UNIQUE` constraint anywhere that says "this quote has been
  executed once." The only guard is the Redis lock — which B1 showed is broken, and which expires
  after 30 seconds anyway.

**Why this loses money — concrete scenario.** Execute runs for quote Q1: provider payout succeeds
at line 74… and the process crashes at line 85 (deploy restart). The quote sits in
PENDING_EXECUTION. The user (or a retry job, or support staff clicking "retry") re-fires `/execute`
for Q1. The Redis lock from the first attempt has long expired. Nothing in the database says "Q1 was
already paid" — so the whole flow runs again, and the provider pays again. Rule R8's own words: the
DB constraint is the last line of defence. Right now there is no last line — only a broken first
line.

**What must be done:**

- Migration `09_create_executions.ts`: an `executions` table with `quote_id UNIQUE NOT NULL`,
  `idempotency_key`, `provider_reference`, `on_chain_tx_hash NULLABLE`, `status`, timestamps.
- Insert-first gate (NombaVault pattern): INSERT the execution row before calling the provider. A
  second attempt violates the unique constraint → catch it → return 409. This is atomic at the
  database; no application race can beat it.
- Send an idempotency key to the provider derived from `execution_id` (Rapyd supports this). Then
  even a crash mid-provider-call can't double-send — the provider dedupes on their side. Three
  independent guards: Redis lock (fast), DB unique (authoritative), provider idempotency (external).
- Route every status change through the state machine. PENDING_EXECUTION → COMPLETED only via
  `transition()`; illegal jumps (e.g. FAILED → COMPLETED) rejected in one place.

### B4 (HIGH) — Chain hashes are fabricated → the ledger's "proof of payment" can be a lie

**The problem, line by line.** `juicyway.adapter.ts:59`:
`providerTxHash: json.data.transaction_hash || json.data.id` — if the provider response has no
`transaction_hash`, the code silently substitutes the provider's order ID and passes it up as a
blockchain transaction hash. `execute-swap.use-case.ts:103` then writes it into the ledger as
`txHash`. No format validation, no confirmation check, no distinction between "provider accepted the
order" and "the crypto actually moved on-chain."

**Why this matters financially.** The tx hash is your proof of delivery — the one identifier that
lets you (or a disputing user, or an auditor) independently verify on a public blockchain that the
asset arrived. If the ledger contains order IDs masquerading as hashes, you cannot distinguish
"delivered" from "provider said OK." When a user claims "I never received my USDT," you have no
evidence. When you reconcile against the chain, nothing matches.

**What must be done — three identifiers, never confused:**

- `provider_reference` — the provider's own ID for the payout. Always present, stored on the
  execution row. Used for querying provider status.
- `on_chain_tx_hash` — the real blockchain hash. Nullable. Set ONLY when the provider/webhook
  supplies one (Rapyd returns `crypto_payout_hash`). Validated by format per chain (`0x` + 64 hex
  for EVM; 64 hex for Tron). If absent, it stays NULL — an honest "not yet confirmed" beats a
  fabricated value every time. Ledger entries carry UNVERIFIED → VERIFIED states.
- Internal ledger hash chain (Blnk): every ledger operation stores
  `hash = SHA256(id + account_alias + direction + amount + created_at + prev_hash)`, chaining to
  the previous operation on that account. Anyone who edits history breaks every hash after the edit.
  `GET /admin/ledger/verify` re-walks the chain and reports breaks — cryptographic tamper-evidence
  for our own books, independent of any blockchain. This runs nightly and in CI.

### B5 (HIGH) — The payout amount comes from a cache → Redis becomes the source of truth for money

**The problem, line by line.** `execute-swap.use-case.ts:34-41`: line 34 reads the quote from Redis
(`cache.get<any>` — note `any`: zero validation); lines 37-40 fall back to Postgres only when the
cache is empty. Then line 48 checks expiry against the cached `expiresAt`, and line 74-78 sends
`quote.amountTarget` and `quote.providerReference` — cached values — to the provider.

**Why this is wrong financially.** A ledger database is durable, transactional, access-controlled. A
cache is fast, volatile, and (here) unauthenticated — anyone or anything that can write to Redis can
change the amount of money the engine sends. Even without an attacker: a stale cache after an admin
correction, a serialization bug, a TTL edge case — and the engine pays the wrong amount with no
trace of why. The rule in every payment system: read fast from cache, but move money only from the
system of record.

**What must be done:**

- At execution time, ALWAYS load the quote from Postgres. The cache accelerates pre-execution reads
  (quote display) only.
- Zod-validate the loaded record (amounts are integers, currencies supported, reference non-empty)
  before any provider call — even DB data gets shape-checked at the money boundary.

### B6 (HIGH) — The reconciliation engine does not exist → nobody is checking the books against reality

**The problem.** Glob for reconciliation code in `src/`: nothing. The docs promise a nightly
`ReconciliationJob` and a 5-minute `StuckTransactionJob`; neither is built. The one ad-hoc script
(`check-ledger.js`) reads a JSONB field the code never writes. Practical consequence: a conversion
stuck in PENDING_EXECUTION (crash, provider timeout) is invisible forever — the user paid, nothing
arrived, and no system notices until the user complains publicly.

**Why reconciliation is not optional.** Your ledger says what should be true. The provider's balance
and the blockchain say what is true. Reconciliation is the daily act of proving they agree — it's
how you detect provider errors, missed webhooks, internal bugs, and fraud before they compound.
Every serious money business runs it daily; most discover their first real incident through it.

**What must be done — three jobs (BullMQ, patterns from NombaVault):**

- `StuckTransactionJob` (every 5 min): find executions in non-terminal states older than the
  timeout → poll provider status → resolve forward (complete/fail via state machine) or park in DLQ
  + alert a human.
- `ProviderPollRecoveryJob` (every 15 min): webhooks are best-effort — if the engine was down when
  the provider called, that event is gone. Poll the provider's payout list for the last window; feed
  anything unseen through the same handler. The B3 idempotency gate makes replays harmless by
  construction.
- `ReconciliationJob` (nightly): reconstruct every account balance from raw ledger operations (B2's
  endpoint); compare `@provider:rapyd:USDT` against the provider's reported balance with drift
  tolerance (Blnk pattern — e.g. 1%, because provider fees make exact matching fail); any
  discrepancy beyond tolerance → BALANCE_DRIFT event + alert. Books must balance internally AND
  against the outside world.

### B7 (MEDIUM) — "Append-only" ledger is a comment, not a constraint

**The problem.** The migration comment says "No UPDATE operations will ever run on this table."
Nothing enforces it — any code path, migration mistake, or compromised credential can UPDATE/DELETE
ledger rows and rewrite history. **What must be done:** Postgres triggers that RAISE EXCEPTION on
UPDATE/DELETE on `ledger_transactions`, `ledger_operations`, `audit_logs`; REVOKE UPDATE/DELETE from
the app's DB role; repository exposes only `save()`. Corrections are new REVERSAL postings — standard
double-entry: you never erase a mistake, you post its mirror. Combined with the B4 hash chain,
history becomes both immutable and tamper-evident.

### B8 (MEDIUM) — Unchecked Results throughout the money path

**The problem.** `execute-swap.use-case.ts` lines 61, 81, 109 (`updateStatus`) and 62-71, 111-120
(`audit.log`): all awaited, none checked. The engine's whole error discipline is the Result monad —
and the most critical file ignores it. A failed status write means the DB says PENDING_EXECUTION
while the flow continues to completion. **What must be done:** check every Result in the money path;
a failed write after external money movement → COMPENSATION_REQUIRED + alert; enable
`@typescript-eslint/no-floating-promises` and a must-use-Result lint so this class of bug can't
re-enter.

### B9 (MEDIUM) — Lock TTL and ownership

**The problem.** 30s TTL (line 54) vs a payout that can take longer (adapter timeout alone is 5s/call,
multi-step flows more) — the lock can expire mid-flight, letting a second request in; and
`cache.del(lockKey)` in `finally` (line 125) deletes whoever's lock, including a newer legitimate
one. **What must be done:** token compare-and-delete release (B1); TTL sized to payout timeout +
margin; extend (heartbeat) for long-running calls.

---

## PART 2 — SECURITY (this engine records movement of money)

### Threat model — asset → threat → control

| Asset | Threat | Control |
| --- | --- | --- |
| Funds at provider (our Rapyd/Thunes balance) | stolen API keys → attacker drains balance via payouts to their own wallet | keys in secrets manager only; provider-side IP allowlist; payout velocity limits in-engine (max/txn + max/hour, admin-configurable, breach → halt + alert); alert on any provider webhook for a payout we didn't initiate |
| The ledger | insider/attacker rewrites history to hide theft | append-only triggers + revoked grants (B7) + hash chain + nightly verify + reconstruction (B2/B4) |
| Provider credentials | DB dump exposes `api_key_encrypted` | prove AES-256-GCM with a decrypt-fixture test (docs' own P1 flag — currently unproven); rotation runbook |
| Execute path | replay/duplicate/concurrent → double payout | Redis NX lock + DB unique gate + provider idempotency key (B1/B3) — three independent guards |
| Inbound webhooks | forged "payout completed" flips states; forged deposits credit users | 5-step chain: IP allowlist → HMAC signature → schema parse → DB idempotency gate (UNIQUE(event_id, provider)) → state machine; idempotency committed AFTER processing (R12) |
| API surface | unauthenticated execute; DoS driving provider cost | HMAC auth mounted on every route; rate limiting with `X-RateLimit-*` headers; strict Zod; correlation IDs |
| Logs/errors | provider error bodies with secrets leak into logs (`juicyway.adapter.ts:104-112`); internals leak to clients | field-whitelist scrubbing into `EngineError.details`; R9 masking; Pino redact paths |
| Config | `JUICYWAY_API_KEY: min(1)` at `env.ts:47` — docs claim min-32 was applied; the code says otherwise; Redis with no auth requirement holds the payment locks | min(32) all provider secrets; Redis password+TLS required by prod schema; fail-closed startup |
| Admin actions | rogue admin refunds to self | separate admin secret (cryptographic separation — NombaVault pattern); every admin action append-only audit-logged before returning; dual control on manual payouts |

### New non-negotiable rules (append to ENGINE_RULES as R14–R18)

- **R14**: every payout carries an idempotency key derived from `execution_id`; passed to providers
  that support it.
- **R15**: ledger tables physically append-only (trigger + grants); hash chain verifies clean
  nightly and in CI.
- **R16**: `on_chain_tx_hash` is never inferred or fabricated — NULL until a validated hash arrives.
- **R17**: payout velocity limits enforced in-engine; breach halts payouts and alerts.
- **R18**: secrets live only in env/secrets-manager; any DB-stored secret must be provably
  AES-256-GCM (test decrypts a fixture).

---

## PART 3 — WHAT WE ADOPT FROM NOMBAVAULT (and skip)

XanePay is a conversion engine, not a virtual-account platform: take the financial core, skip the
platform.

| ✅ Adopt | Lands where |
| --- | --- |
| Two-table ledger + routes-as-data (Midaz) | B2 solution |
| System account aliases incl. `@suspense` (Formance) | B2 — money never disappears |
| `balance_before/after` per operation (Midaz) | B2 — instant audit of any movement |
| SHA-256 hash chain + `ledger/verify` (Blnk) | B4/B7 — tamper-evident books |
| DB UNIQUE idempotency gate, insert-first (NombaVault webhook handler) | B3 + webhook chain |
| Balance reconstruction `{stored, computed, drift}` (Blnk) | B2/B6 — the "prove the books" button |
| Drift-tolerant reconciliation rules (Blnk) | B6 — provider fees break exact matching |
| Provider polling recovery job | B6 — webhooks are best-effort |
| Optimistic-lock version on balances (Midaz) | B2 balances table |
| Transactional outbox | later, when the engine sends webhooks to XaneApp |
| Sandbox simulator endpoint | Day 3 — test recon without provider creds |

**❌ Skip:** NUBAN/customer identity, KYC tiers, developer platform/dashboards, SDK/docs-site, PDF
statements — XaneApp-side, not engine scope.

---

## PART 4 — HOW ADDITIONS CAN'T BREAK THE ENGINE

- One port, many adapters, zero core edits — new rail = 1 adapter + 1 factory case + 1 providers row.
- Shared contract test suite every adapter must pass (integer minor units, ErrorKind mapping, never
  throws, timeouts).
- CI-enforced hexagonal boundaries — dependency-cruiser: domain imports nothing; application never
  imports adapters.
- DB constraints as last line — unique gates, append-only triggers, CHECKs: buggy code physically
  cannot double-pay or unbalance books.
- TDD RED-first + CI gate: full suite + migration dry-run + hash-chain verify + chaos test on every
  PR.

---

## PART 5 — THE 4-DAY BUILD & DEPLOY MAP

Ships a deployed, secured engine on provider sandbox. Rapyd/Thunes KYB is an external clock — start
onboarding Day 1 morning; flip `is_enabled` when live creds arrive. TDD RED-first throughout.

### DAY 1 — Money integrity

- **Block 1: real lock (B1)** — RED test (2 concurrent acquires → one winner) → `setIfNotExists` +
  token `releaseLock` → wire into ExecuteSwap.
- **Block 2 (biggest): ledger rebuild (B2/B4/B7)** — migration `08_rebuild_ledger.ts` (two tables,
  accounts, routes, seeds, append-only triggers, revoked grants) → `LedgerService.post()`
  single-transaction → hash chain + `verify()` + reconstruction → rewire ExecuteSwap with checked
  Results and COMPENSATION_REQUIRED.
- **Block 3: execution idempotency (B3/B8)** — migration `09_create_executions.ts` (UNIQUE(quote_id))
  → insert-first gate → provider idempotency key → state-machine-gated transitions → all Results
  checked.
- **Block 4: truth sources (B5) + identifier split (B4)** — DB-first quote load + Zod;
  `provider_reference` vs validated `on_chain_tx_hash`, UNVERIFIED→VERIFIED.
- **Exit criteria:** full suite green (existing ~315 + new); chaos test green; `ledger/verify` clean;
  deliberate tamper detected.

### DAY 2 — Pluggability + the new rail

`ProviderFactory`; `RapydAdapter` (POST `/v1/payouts`, `xx_stablecoin_ewallet`, HMAC signing,
idempotency key, `crypto_payout_hash` → `on_chain_tx_hash`); `ThunesAdapter` skeleton
(quotation → transaction); rename `treasuryAddress` → `destinationAddress`; env generic + V-fixes
(min-32, optional `JUICYWAY_*`); seed provider rows (juicyway off / rapyd on / thunes standby);
contract suite passes for all adapters.

### DAY 3 — API surface, security, jobs

Composition root + Express: `/quote`, `/execute`, `/conversions/:id`, `/health`,
`/webhooks/:provider` behind HMAC + Zod + rate limiter + correlation IDs + R9 masking; webhook
5-step chain with DB idempotency gate; sandbox simulator endpoint; BullMQ `StuckTransactionJob` +
`ProviderPollRecoveryJob` + `ReconciliationJob`; AES-256-GCM proven by test; velocity limits (R17).

### DAY 4 — Prove it, then deploy

Chaos suite (concurrent executes; kill-between-payout-and-ledger → COMPENSATION_REQUIRED); Rapyd
sandbox E2E (quote → execute → Complete Payout simulation → webhook → COMPLETED + valid hash);
ledger proof (debits=credits; UPDATE throws; verify clean); Docker → deploy (Railway/Render/VPS) +
Sentry + `/health`; provider swap drill; 1-page runbook.

---

## PART 6 — TREASURY RECOMMENDATION

Rent now, own later. Rapyd first (self-serve, USDC/USDT to wallets + fiat, real chain hashes),
Thunes standby (Circle-backed, 130+ countries incl. Nigeria; enterprise onboarding — start paperwork
early). In ledger terms the rented rail IS the treasury: the funded provider balance is
`@provider:rapyd:USDT`, reconciled nightly. Build the self-custody treasury (own hot wallet + KMS;
USDT on Polygon/Base ≈ $0.01 gas) as a third adapter when provider fees (~0.5–2%/txn) exceed
own-treasury run cost — the Day-2 factory makes it a drop-in.

> ⚠️ During KYB: Rapyd stablecoin protocols are ERC20/BEP20/Polygon — confirm TRC20 if mandatory
> (Thunes fallback). NGN collection (user pay-in) still needs a local PSP
> (Paystack/Monnify/Interswitch) — the rail replaces delivery, not collection.

### Appendix — Cost of building our OWN self-custody treasury

Build: adapter (ethers/tronweb) 4–6d; KMS key mgmt 2–3d; monitoring 2–3d; factory (shared) 2–3d;
tests+hardening 6–9d → ~16–24 dev-days (3–5 wks solo), ≈$3k–10k contracted. Run: $50–300/mo
self-managed (KMS + RPC + monitoring). Managed custody (Fireblocks/BitGo) $1.5k–3k+/mo — only at
scale. Gas/send (USDT): Polygon/Base ~$0.001–0.05 ✅ · Tron $1–8 (≈$0 with TRX staking) ⚠️ · ETH L1
$1–20 ❌. Float (the real money): daily onramp volume × 1–3 days buffer. ~₦50m/day ($33k) → hold
$35k–100k USDT; replenish via exchange/OTC (0.1–0.5% spread). Compliance/risk: NG legal review
~$500–5k; key loss = total loss → backups, rotation, multisig before volume. Crossover: own it when
monthly provider fees > own-treasury run+ops cost.

---

## VERIFICATION (definition of done)

- `cd XaneApp/engine && npm test` — all green incl. lock/ledger/factory/contract/chaos; existing
  ~315 unbroken.
- Chaos: 10 concurrent `/execute` on one quote → exactly 1 provider call, 1 balanced posting,
  9× 409.
- Ledger proof: `SUM(amount_minor)` debits = credits per currency; UPDATE on ledger tables throws;
  `ledger/verify` clean; deliberate tamper detected.
- Reconstruction: stored == computed for every account after the full test run.
- Rapyd sandbox E2E: quote → execute → Complete Payout simulation → webhook → COMPLETED with
  format-valid `on_chain_tx_hash`.
- Provider swap drill passes with zero code changes.
- Deployed `/health` green publicly; Sentry test event received; velocity-limit breach halts and
  alerts.

---

## EXECUTION TRACKER

Status legend: ⬜ not started · 🟡 in progress · ✅ done

### Day 1 — Money integrity ✅ COMPLETE (2026-07-11, on the `engine` branch)

> **Read `ENGINE_BRANCH_GAP_REPORT.md` first.** This audit was written against a **stale checkout**
> (detached HEAD `42c8da35`, tag v0.8.0 — a divergent dead-end). The real engine is the **`engine`
> branch (v0.11.0)**, which had already fixed several findings on its own. Day 1 was therefore
> re-executed against `engine`, porting only the ~60% that was genuinely still missing. Migrations
> are numbered **12/13** here (not 08/09) because `engine` already had 08–11.

Baseline on `engine` before this work: **466 passing / 3 skipped / 0 failing**. TDD RED-first
throughout. Run with `cd XaneApp/engine && npx jest --runInBand` (integration suites share one test
DB, so they must not run in parallel workers).

Already correct on `engine` before this session — **no change needed**: B1 (lock *acquire* is a real
`SET NX PX`), B4 (the `transaction_hash || id` hash fabrication was already gone), B6-stuck-txn
(poll/recovery half of reconciliation), B2-atomicity (`recordEntryPair` in one transaction), and
B3-conversion-creation idempotency.

Built this session:

- ✅ **B2 (schema)** — Ledger rebuilt on the two-table model. Migration `12_rebuild_ledger.ts`
  (`ledger_accounts` / `ledger_transactions` with UNIQUE `reference` / `ledger_operations` with
  direction+amount CHECKs), seeded system accounts incl. `@suspense`, routes-as-data
  (`transaction-routes.ts`), and `LedgerService.post()` writing parent + legs + balances in ONE
  Knex transaction. Unbalanced postings are rejected before touching the DB. Additive: legacy
  `ledger_entries` is untouched, so every existing path keeps working.
  Proof: `tests/integration/ledger/ledger-service.test.ts` (13 tests).
- ✅ **B3** — `executions` table (migration `13_create_executions.ts`) with `quote_id UNIQUE`, the
  insert-first gate in `ExecutionRepository.insertPending` (SQLSTATE 23505 → `DUPLICATE_REQUEST`),
  and an `ExecutionStatus` state machine (`domain/value-objects/execution.ts`) — illegal jumps
  rejected in one place. Wired into `ExecuteSwap`: the row is claimed **before** the provider is
  called, so the database — not application logic — decides who may pay out.
  Proof: `tests/integration/persistence/execution.repo.test.ts` — 10 concurrent inserts → 1 winner,
  9 × 409; plus `tests/unit/domain/execution.test.ts`.
- ✅ **B4 (identifiers)** — `provider_reference` vs nullable `on_chain_tx_hash`, validated by
  `parseOnChainTxHash` (EVM `0x`+64hex / Tron 64hex). A provider order id yields NULL — an honest
  "not yet confirmed" beats a plausible lie in the ledger.
  Proof: `tests/unit/domain/on-chain-tx-hash.test.ts`.
- ✅ **B5** — Execution loads the quote from **Postgres only** (never the cache) and Zod-validates it
  (`ExecutableQuoteSchema`: integer minor units, supported currencies, non-empty reference) before
  any provider call. Proven with a poisoned cache entry (100× the amount) that the engine ignores.
- ✅ **B7** — Append-only enforced by Postgres triggers (RAISE EXCEPTION on UPDATE/DELETE of
  `ledger_transactions`, `ledger_operations`, `audit_logs`) + `REVOKE UPDATE, DELETE`. Corrections
  are `REVERSAL` postings. SHA-256 per-account hash chain + `verifyChain()` detects tampering even
  when the triggers are bypassed (`session_replication_role = replica`).
- ✅ **B8** — Every pre-provider Result on the money path is checked. If we cannot record our intent
  to pay, we do not pay. And if the payout succeeded but the ledger write failed, the execution
  becomes **`COMPENSATION_REQUIRED`** — never silently `COMPLETED`. A throw *after* the provider
  call is treated as ambiguous (may have landed) and also goes to `COMPENSATION_REQUIRED`.
- ✅ **B9** — Token-safe lock release: `releaseLock(key, token)` is a Lua compare-and-delete, so a
  stale `finally` block can never delete a lock another process legitimately holds.
  Proof: `tests/integration/cache/redis-lock-release.test.ts`.
- ✅ **B6b (balance reconciliation)** — `ReconciliationService.reconcileLedgerBalances()` re-derives
  every balance from the immutable operations, compares against the stored projection, and re-walks
  the hash chain. Drift is flagged + audited (`ledger_drift_detected` / `ledger_chain_broken`) and
  deliberately **not** auto-corrected — a balance that disagrees with its own history is evidence,
  not a bug to patch.
  Proof: `tests/unit/application/services/ledger-reconciliation.test.ts`.

**Ledger "prove the books" surface** (used by the reconciliation job and the admin endpoint):
`LedgerService.verifyChain()`, `.reconstructBalances()` → `{stored, computed, difference, flagged}`,
`.getBalance()`, `.reverse()`.

### Still open

- ⬜ **B6 (jobs)** — The BullMQ `ReconciliationJob` that *calls* `reconcileLedgerBalances()` on a
  schedule, and the admin endpoint that exposes it. The service method and its ledger primitives are
  built and tested; only the scheduling/HTTP wrapper is missing (Day 3).
### ✅ Ledger cutover — the money path now writes the double-entry books (2026-07-11)

Until this step the books were **empty**: `ExecuteSwap` only wrote the legacy `ledger_entries` blob,
so `@fees:*` never earned, `@provider:*` float never moved, and B6b's `reconcileLedgerBalances()` was
reconciling nothing and would always report "balanced". A clean bill of health over an empty ledger
is worse than no report at all.

- ✅ **`LedgerService.post(route, params, ctx)`** now takes an optional transaction context and joins
  the caller's transaction instead of opening its own. Without this the conversion could commit
  COMPLETED while the ledger posting rolled back — the exact atomicity bug the ledger exists to
  prevent. Proven by two tests: rolling the caller back rolls the posting back; committing commits.
- ✅ **`CONVERSION_SETTLED` fixed to model the WHOLE movement.** It previously posted only the payout
  and fee legs — which balances per-currency (so the tests passed) but debits a fee from a user
  source-currency account that was **never credited**, driving every `@user:{id}:NGN` permanently
  negative. It now books the money arriving (`@external` → `@user`), our fee (`@user` → `@fees`), the
  float that funds the swap (`@user` → `@provider`), and the delivery (`@provider` → `@user`).
- ✅ **Dual-write, one transaction.** The money path writes both the legacy event log (*what
  happened* — still read by `ReconciliationService` and `WebhookHandlerService`) and the double-entry
  books (*how much*), inside the single existing `runInTransaction`. A failed ledger posting aborts
  the transaction → `COMPENSATION_REQUIRED`, never a silent `COMPLETED`.
- ✅ **Fee currency is the SOURCE currency** — not inferred, not defaulted to NGN. `FeeService`
  ([fee-service.ts:60](../XaneApp/engine/src/application/services/fee-service.ts#L60)) hard-errors on
  any fee config whose currency differs from the source, so this is an enforced invariant. Defaulting
  to NGN would have mis-booked CRYPTO_TO_FIAT/CRYPTO_TO_CRYPTO fees, which are seeded as USDT.

Proof: `tests/integration/ledger/money-path-ledger.test.ts` — runs a real conversion through
`ExecuteSwapUseCase` against Postgres, then asserts the user was credited, we earned our fee, float
moved, debits === credits per currency, `reconstructBalances()` flags nothing, and `verifyChain()` is
clean. It asserts the ledger is **non-empty**, which is the assertion that would have caught this.

### ✅ END-TO-END PROOF — a real conversion over HTTP (2026-07-11)

The suite was 526-green and still could not answer the only question that matters commercially: *has
a customer's naira ever actually become USDT with the books balanced?* It couldn't, because:

- `tests/e2e/` drove real HTTP but was wired to `InMemoryCache`/`InMemoryQuoteRepo` — no Postgres, no
  ledger. It proved routing, not money.
- The provider was a `jest.fn()` in **every** test, so `JuicywayAdapter`'s float→integer conversion,
  auth headers, error mapping and idempotency-key handling were never once executed. The code most
  likely to lose money was the code least tested.
- `conversion.api.test.ts` / `health.api.test.ts` were `describe.skip` placeholders asserting
  `expect(true).toBe(true)`. The 3 "skipped" tests were exactly the ones that would have proven this.

Now: **`tests/e2e/money-path.e2e.test.ts`** drives a conversion through the real stack —

    HTTP (real HMAC, rate limiter, Zod) → real CompositionRoot → real Postgres → real Redis
      → real BullMQ queue AND worker → real JuicywayAdapter over a real socket → fake rail
      → THE BOOKS BALANCE

Nothing is a `jest.fn()`. Only the far side of the network is faked (`tests/e2e/helpers/fake-rail.ts`)
— unavoidable, since JuicyWay's API is dead, and precisely why it is faked at the **socket** rather
than at the port: everything we wrote still executes. The fake rail answers in **floats**
(`0.645161 USDT`) like the real API, so the assertion `@user:…:USDT === 645_161` passes only if the
adapter's float→integer conversion is correct.

It also proves the double-spend guard **through the front door**: two API requests against the same
quote produce exactly one `executions` row — the database refuses the second claim regardless of what
the application does.

`tests/helpers/real-app.ts` is the reusable fixture (the "Phase 11 CompositionRoot fixture" the API
placeholders were waiting for). The 3 skipped placeholders are now **9 real tests** covering HMAC
auth (401 unsigned / 401 wrong secret), integer-only amounts (400 on a float — R1), unsupported
currencies, and a live health check.

### 🔴 PRODUCTION HAZARD FOUND BY RUNNING THE REAL STACK — Redis eviction policy

BullMQ warned on boot: `Eviction policy is allkeys-lru. It should be "noeviction"`.

Redis was configured `--maxmemory 128mb --maxmemory-policy allkeys-lru`. **Redis is not just a cache
here — BullMQ stores queued jobs in it.** Under memory pressure, Redis was free to evict the job for
a conversion the API had already accepted (customer holding a `202`): money in, nothing delivered, no
error raised anywhere. Fixed in `XaneApp/engine/docker-compose.yml` → `--maxmemory-policy noeviction`.
A failed enqueue we can see and retry beats a vanished payout we cannot.

**A doc note is not a control.** Managed Redis (ElastiCache, Upstash, Redis Cloud) frequently defaults
to `allkeys-lru`, which would silently reintroduce the bug on the day it matters. So the engine now
**refuses to boot** against any eviction policy:

- `src/infrastructure/startup/redis-eviction-guard.ts` — reads `CONFIG GET maxmemory-policy`;
  anything other than `noeviction` is a hard `process.exit(1)` with an explicit message.
- Called from `src/index.ts` **before any worker starts**, so the hazard cannot reach production.
- Degrades to a loud warning (not a crash-loop) where the provider forbids `CONFIG GET` — being
  undeployable on such a provider would be a worse failure than an unverified policy.

The hazard is now *undeployable* rather than merely documented. No mock would ever have surfaced it;
it took booting the actual queue.

### 🔴 THE PHANTOM LIABILITY — found by writing the delivery-path e2e (2026-07-11)

**The books were wrong about reality, and reconciliation would never have told us.**

`ExecuteSwap` posts `CONVERSION_SETTLED`, which CREDITS `@user:{id}:{tgt}` — correct at that instant,
because the swap has settled and we genuinely owe the customer the asset. But **nothing ever debited
it back out when the asset was actually delivered.** The webhook that confirms delivery only flipped
the conversion status and wrote a *legacy* `ledger_entries` row; the synchronous-success path just
marked it COMPLETED. So after genuinely sending a customer their USDT on-chain, the double-entry
books still recorded that **we owe them that USDT** — a liability compounding with every single
payout.

And `reconcileLedgerBalances()` would have reported **`balanced: true`** the entire time, because the
books *were* internally consistent. They had simply stopped describing the world. **Internal
consistency is not truth** — which is the whole reason this needed an end-to-end test rather than
another unit test.

Fixed with a `PAYOUT_DELIVERED` route (`DEBIT @user:{id}:{ccy}` / `CREDIT @external:{ccy}`), posted
in the **same transaction** as the COMPLETED flip, on **both** doors a conversion can complete
through:
- the webhook confirmation (`WebhookHandlerService`), and
- the synchronous `payout.status === 'success'` path (`ExecuteSwapUseCase`) — which the first fix
  missed entirely, and which only the e2e caught.

> 🔴 **"Both" was wrong. There were FOUR** (found 2026-07-22, audit M5/M6):
> - `ExecuteSwapUseCase`'s **no-destination** path — posted `CONVERSION_SETTLED`, marked COMPLETED,
>   never paid anyone or cleared the credit. Reachable from the public API, since `destination` was
>   `optional()`.
> - `WebhookHandlerService`'s **generic success** path — any conversion not exactly `PAYOUT_PENDING`
>   walked PROVIDER_CONFIRMED → COMPLETED writing nothing to the hash-chained ledger.
>
> The lesson is not "we miscounted." **Enumerating the doors is not the same as enforcing the
> invariant** — each new completion path had to remember the clearing leg, and two did not. Both are
> now *closed* rather than patched: a destination is required, and there is exactly one webhook
> completion path. See `ENGINE_MONEY_AUDIT_2026-07.md`.

The rule is now: **COMPLETED means DELIVERED means the liability is cleared.** `reference` is
`payout:{conversionId}` and UNIQUE, so a replayed webhook (providers retry) cannot release the asset
twice and drive the customer's balance negative.

Proof: `money-path.e2e.test.ts` → after real delivery, `@user:{id}:USDT === 0`, a `PAYOUT_DELIVERED`
transaction exists, debits still equal credits per currency, and reconciliation is clean.

### ✅ The smoke detector now has a battery

`reconcileLedgerBalances()` was built, tested — and **wired into nothing**. No job, no endpoint, no
schedule. It now runs on the `ReconciliationProcessor` cron (drift → CRITICAL log + Sentry, never
auto-corrected) and via `POST /api/v1/admin/reconciliation/trigger`, which now returns
`{ conversions, ledger, balanced }`.

### ✅ Redis hazard is now UNDEPLOYABLE, not merely documented

Changing `docker-compose.yml` was a dev-only fix; managed Redis defaults to `allkeys-lru`. The engine
now **refuses to boot** against any eviction policy
(`src/infrastructure/startup/redis-eviction-guard.ts`, called from `src/index.ts` before any worker
starts). It degrades to a loud warning — not a crash-loop — where the provider forbids `CONFIG GET`.

### ✅ Hermetic suite (Testcontainers)

`tests/helpers/global-setup.ts` now starts Postgres + Redis programmatically and
`global-teardown.ts` stops them. No `docker compose up -d`; the suite runs on a cold machine with
only Docker present, and CI is reproducible. Redis starts with `--maxmemory-policy noeviction` so the
tests exercise the production configuration rather than a different system.
Escape hatch: `USE_TESTCONTAINERS=false` reuses an existing compose stack.

### Still open on testing

- ✅ **Testcontainers** — done (see above).
- ✅ **Concurrent chaos through the API** — 10 *simultaneous* `POST /conversions/execute` against one
  quote → exactly one `executions` row and exactly one rail swap call.
- ✅ **True replay** — the identical request (same idempotency key) sent twice creates one conversion,
  not two.
- ✅ **Every endpoint now covered against the real stack** — `/health`, `/quote`, `/conversions/quote`,
  `/conversions/execute`, `/conversions/:id`, `/webhooks/juicyway`, `/admin/*`.
- ⚠️ **The fake rail is our reading of JuicyWay, not JuicyWay's.** It was written from the adapter's
  own type definitions — so if the adapter misunderstands the real contract, the fake agrees with the
  bug. **This is not closable now the API is dead** and remains an accepted risk. The mitigation, on
  Day 2: a provider **contract suite** every adapter must pass, plus **golden fixtures recorded from
  Rapyd's live sandbox** when KYB credentials land.

### Follow-up: retire the legacy ledger

Dual-write is deliberate and temporary — the new ledger has never seen production traffic, so the
proven event log stays until B6b has run clean against real data. Retiring it means moving
`ReconciliationService`'s COMPLETED-entry check and `WebhookHandlerService` onto `ledger_operations`,
then dropping `recordEntryPair` from the money path. **This needs an owner, or "temporary" becomes
permanent.**

### Day 2 — Pluggability + new rail

- ⬜ `ProviderFactory` + `RapydAdapter` + `ThunesAdapter` skeleton + contract suite
- ⬜ `treasuryAddress` → `destinationAddress`; env V-fixes (min-32, optional JUICYWAY_*)
- ⬜ Seed provider rows (juicyway off / rapyd on / thunes standby)

### Day 3 — API surface, security, jobs

- ⬜ Composition root + Express routes behind HMAC + Zod + rate limiter + correlation IDs
- ⬜ Webhook 5-step chain with DB idempotency gate
- ⬜ BullMQ `StuckTransactionJob` + `ProviderPollRecoveryJob` + `ReconciliationJob`
- ⬜ AES-256-GCM decrypt-fixture test; velocity limits (R17)

### Day 4 — Prove it, then deploy

- ⬜ B6 reconciliation proven end-to-end; chaos suite; Rapyd sandbox E2E
- ⬜ Docker + deploy + Sentry + `/health`; provider swap drill; 1-page runbook
