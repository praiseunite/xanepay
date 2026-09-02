# Engine Money-Integrity Audit — 2026-07-22

**Scope:** `XaneApp/engine`, branch `engine`, HEAD `19a4fc3a`. Working tree clean.
**Method:** trace the value, not the call graph. Read every path money can take, then ask of each
green test what it would *still* pass with.

> ## ✅ ALL TEN FIXED — 2026-07-22
>
> Every finding below was fixed the same day it was written up, each **test-first with an observed
> RED run**. Final state: **646 unit/security/bdd/contract + 124 integration/e2e green, lint 0,
> tsc 0.** The per-fix write-ups, with the trap that made each possible, are in
> `XaneApp/engine/docs/FINDINGS_REGISTER.md` under "the money-integrity audit, M1–M10".
>
> Three RED runs worth keeping, because they *are* the bugs:
> - **M1** — completion booked `999999` (a stale stored amount) where the rail would deduct `15000`.
> - **M2** — books holding ₦50,000 against a silent wallet returned `flagged: []`, `balanced: true`.
> - **M4** — a swap the rail reported as `pending` came back `conversionsFixed: 1`.
>
> **Two caveats that survive the fixes** and must not be read as closed:
> 1. **Nothing here has been proven with real money.** The OnSwitch sandbox's placeholder deposit
>    address cannot receive a transfer, so no conversion has ever reached `COMPLETED` against a real
>    rail. What is proven is that the number on the wire and the number in the books are now the
>    same number, end to end through the real stack.
> 2. **The JuicyWay half of M9/O4 is unclosable** — that API is dead, so no fixture can ever be
>    recorded from it.
>
> The findings are preserved below exactly as written, as the record of what was wrong and why.

---

## The one-line summary

The engine's books are internally immaculate and, on a default deployment, **describe revenue that
was never collected**. The control designed to catch exactly that — nightly treasury drift detection
— is structurally blind to it, and reports `balanced: true` while doing nothing at all.

That is the same failure mode as the phantom-liability incident this project already survived once:
*internal consistency is not truth.* It has recurred in a new place.

---

## Findings, ranked by money consequence

| # | Finding | Severity | Live today? |
|---|---|---|---|
| **M1** | Revenue is booked from a fee the engine never asks the rail to collect | **CRITICAL** | **Yes** |
| **M2** | Treasury drift detection iterates the wallet's currencies, so it cannot see M1 | **CRITICAL** | **Yes** |
| **M3** | The customer is quoted a rate markup that cannot be realised, and a fee that is never deducted | **HIGH** | **Yes** |
| **M4** | Reconciliation marks a **pending** custody swap as `COMPLETED` | **HIGH** | Latent |
| **M5** | The no-destination execute path completes without clearing the user liability | **HIGH** | Latent |
| **M6** | Webhook generic-success path completes without `PAYOUT_DELIVERED` | **MEDIUM** | Latent |
| **M7** | `AWAITING_DEPOSIT` status-write failure orphans a claimed execution | **MEDIUM** | **Yes** |
| **M8** | Unchecked `Result`s inside the reconciliation transaction | **MEDIUM** | **Yes** |
| **M9** | O4 is only half-closed — the golden fixtures were never recorded | **MEDIUM** | **Yes** |
| **M10** | `TREASURY_BALANCES_JSON` parsed unguarded at composition | **LOW** | **Yes** |

"Latent" = the code is reachable through the public API, but only with a **custody** rail. JuicyWay
is dead and OnSwitch is deposit-based, so nothing exercises it *today*. Each one arms itself the day
a second custody rail is registered — which is the stated plan.

---

## M1 — CRITICAL — We book revenue we never asked to be paid

**There are two entirely separate fee numbers and nothing reconciles them.**

*What we book:*
[`deposit-completion.service.ts:80-100`](../XaneApp/engine/src/application/services/deposit-completion.service.ts#L80-L100)
posts `REVENUE_EARNED` for `conversion.feeAmount` — computed by
[`FeeService.applyFees()`](../XaneApp/engine/src/application/services/fee-service.ts#L72) from the
`fee_config` table, or from `DEFAULT_FEE_BPS` (**default 50 bps**) when no row exists.

*What we actually collect:*
[`onswitch.adapter.ts:115-121`](../XaneApp/engine/src/adapters/driven/providers/onswitch/onswitch.adapter.ts#L115-L121):

```ts
private developerFeeFields(): Record<string, unknown> {
  if (this.developerFeeBps <= 0 || !this.developerRecipient) return {};
  return { developer_fee: this.developerFeeBps / 100, developer_recipient: this.developerRecipient };
}
```

`developerFeeBps` is `ONSWITCH_DEVELOPER_FEE_BPS` — [`env.ts:60`](../XaneApp/engine/src/infrastructure/config/env.ts#L60),
**default `0`**. `developerRecipient` is `ONSWITCH_DEVELOPER_RECIPIENT` — **optional, no default**.

So on a default deployment `developerFeeFields()` returns `{}`, the `initiate` call carries **no fee
at all**, OnSwitch collects nothing for us — and completion still books `REVENUE_EARNED` at 50 bps
of every conversion.

**Money consequence.** `@treasury:onswitch:{ccy}` grows by a fee that never arrived, on every single
conversion, forever. This is not a rounding drift; it is the entire fee line. Every downstream
number built on it — the treasury snapshots, the period reports produced by differencing them, any
revenue figure shown to a board or an auditor — is fiction. Nothing crashes, nothing logs, nothing
fails a test.

**The two numbers can also disagree without being zero.** Even correctly configured, `fee_config.fee_bps`
and `ONSWITCH_DEVELOPER_FEE_BPS` are edited in different places by different people (one is a DB row
behind an admin endpoint, the other an env var behind a redeploy). Nothing asserts they match. The
divergence is silent in both directions.

**Aggravating detail:** `FeeService` clamps to `minFeeAmount`/`maxFeeAmount`
([fee-service.ts:74-75](../XaneApp/engine/src/application/services/fee-service.ts#L74-L75)) and
`developer_fee` is a flat percentage with no equivalent. Even with the bps aligned, any conversion
that hits a min or max floor books a different number from the one collected.

**Test that would have caught it:** none exists. `onswitch-deposit-lifecycle.test.ts:171-172`
asserts `@treasury:onswitch:USDC === FEE` and `@revenue:USDC === -FEE`, where `FEE` is
`conversion.feeAmount` — the test takes the booked number as its own oracle and never checks that
OnSwitch was asked for it. It proves the books are self-consistent about a number describing money
we never received. **This is exactly the vacuity class that hid the phantom liability.**

**Proposed fix:** one source of truth. Derive the `developer_fee` sent on the wire from the same
`FeeService` result that gets booked, passed per-call rather than baked into the adapter's
constructor. Until that lands, a boot-time assertion that `ONSWITCH_DEVELOPER_FEE_BPS` equals the
active `fee_config` bps **and** that `ONSWITCH_DEVELOPER_RECIPIENT` is set — refusing to boot
otherwise, in the manner of `redis-eviction-guard.ts`. A note in a doc is not a control.

---

## M2 — CRITICAL — The drift detector cannot see M1

`reconcileTreasuryRevenue` is described in the code as *"the check that catches the two failures that
actually cost us money: a fee we booked but never received, and profit that left the wallet without
the engine knowing"* ([reconciliation.processor.ts:117-119](../XaneApp/engine/src/adapters/driven/queue/processors/reconciliation.processor.ts#L117-L119)).

It cannot catch the first one. [`reconciliation.service.ts:215`](../XaneApp/engine/src/application/services/reconciliation.service.ts#L215):

```ts
for (const [currency, actual] of Object.entries(actualBalances)) {
```

It iterates **the currencies the wallet reports**, not the currencies the books have booked. A
treasury with booked revenue and a wallet that reports nothing produces an empty loop, `flagged: []`,
and **`balanced: true`**.

**That is the default configuration.** [`composition-root.ts:285-291`](../XaneApp/engine/src/infrastructure/composition-root.ts#L285-L291):

```ts
const balances = env.TREASURY_BALANCES_JSON ? JSON.parse(...) : {};
return new TreasuryRegistry([new ConfigTreasurySource(env.ONSWITCH_TREASURY_ID, balances)]);
```

`TREASURY_BALANCES_JSON` is optional with no default, so `balances` is `{}`. Trace the nightly sweep:

1. `getBalances()` returns `ok({})` — **success**, so the unreadable-treasury branch at
   [reconciliation.processor.ts:130](../XaneApp/engine/src/adapters/driven/queue/processors/reconciliation.processor.ts#L130) is skipped.
2. `reconcileTreasuryRevenue(id, {})` → zero iterations → `balanced: true`.
3. `writeSnapshot(id, {})` → [treasury-reporting.service.ts:61-63](../XaneApp/engine/src/application/services/treasury-reporting.service.ts#L61-L63):
   `currencies = Object.keys({}) = []` → **zero rows written**.

So the nightly treasury job reports a clean bill of health, writes no history at all, and raises
nothing — while `@treasury:onswitch:NGN` accrues fictitious revenue on every conversion.

**The asymmetry is the bug.** `writeSnapshot` handles `null` (unreadable) correctly, falling back to
`bookedCurrencies(treasuryId)`. It does not handle `{}` (readable but empty) the same way — and `{}`
is the shipping default. The doc's own principle — *"'unknown' and 'empty' must not look the
same"* — is honoured for `null` and violated for `{}`.

**Test that would have caught it:** none. Every case in `treasury-revenue-recon.test.ts` supplies the
currency under test. The test at line 99 is even *named* **"checks every currency it is given"** —
the blind spot is stated in the test title and was never read as a gap.

**Proposed fix:** reconcile over the **union** of booked currencies and wallet-reported currencies.
A currency with a booked balance and no wallet figure is drift of the full booked amount, not
silence. Treat an empty balance map from a treasury with booked revenue as unreadable, not empty.

---

## M3 — HIGH — The quote promises two charges, and collects neither

[`initiate-conversion.use-case.ts`](../XaneApp/engine/src/application/use-cases/initiate-conversion.use-case.ts)
applies `feeBps` **twice, in two different ways**:

- **as a rate markup** — `computeCustomerRate({ ..., feeBps })` → `applyFeeToRate` →
  `floor(rate × (10_000 + feeBps) / 10_000)` ([fee-rules.ts:29](../XaneApp/engine/src/domain/rules/fee-rules.ts#L29)).
  `amountTarget` is derived from that marked-up rate (line 158-160), so the customer is quoted **less
  crypto**.
- **as a separate `feeAmount`** — returned to the caller as `feeAmountMinorUnits` (line 193) and
  stored on the quote (line 175).

Neither is ever collected:

- The markup cannot be realised. `execute-swap.use-case.ts:236` sends
  `amountSourceMinorUnits: quote.amountSource` — **the full amount** — and OnSwitch converts at *its*
  rate and delivers straight to the customer's own wallet. We never hold the float, so there is no
  spread to keep. The findings register already states this (*"A rate markup CANNOT pay us here"*);
  the quote path was never changed to match.
- `feeAmount` is never deducted from what is sent, and never appears on the wire.

**Money consequence.** The customer is quoted a worse rate than they receive and a fee they are never
charged — a quote that does not describe the transaction. We collect nothing from either. Not a loss
of funds, but every quote the engine issues is inaccurate in a way a customer or a regulator could
read as a misrepresentation of price.

**Related:** the discrepancy assertion at `execute-swap.use-case.ts:276` compares
`swap.sourceAmountMinorUnits` against `quote.amountSource` and only `logger.warn`s on a mismatch. It
is skipped entirely on the deposit-based path (the branch returns at line 271-273, before line 276),
so the one place amounts are compared against the rail does not run for the only live rail.

---

## M4 — HIGH — A pending swap is marked COMPLETED

[`reconciliation.service.ts:497`](../XaneApp/engine/src/application/services/reconciliation.service.ts#L497):

```ts
if (swap.status === 'success' || swap.status === 'pending') {
  const result = await this.atomicCompleteFromReconciliation(conversion, swap);
  ...
  return ok('completed');
}
```

A rail that reports the swap is **still pending** causes the conversion to be marked `COMPLETED` and
`TREASURY_CREDITED` ledger entries to be written.

The irony is fourteen lines up. The deposit-based branch at 483-495 handles all three outcomes
correctly, under a comment that reads: *"Only 'pending' skips: the deposit may still land, and we
never mark delivered before the rail says so."* That discipline was applied to the new branch and
never back-ported to the custody branch immediately below it.

**Money consequence.** We tell the customer their conversion is done, and book it, on a swap the rail
has not completed. If it subsequently fails, the books record a delivery that never happened and no
process reverses it — the conversion is now in a terminal state that reconciliation skips
(`default: return ok('skipped')`, line 436-437).

---

## M5 — HIGH — The no-destination path completes while still owing the customer

[`execute-swap.use-case.ts:386-399`](../XaneApp/engine/src/application/use-cases/execute-swap.use-case.ts#L386-L399),
the "LEGACY PATH: no delivery target":

```ts
if (!destination) {
  await this.transactionPort.runInTransaction(async (ctx) => {
    await this.conversionRepo.updateStatus(conversionId, ConversionStatus.PROVIDER_CONFIRMED, undefined, ctx);
    await writeLedgers(ctx);                       // posts CONVERSION_SETTLED → CREDITs @user:{id}:{tgt}
    await this.conversionRepo.updateStatus(conversionId, ConversionStatus.FEE_COLLECTED, undefined, ctx);
    await this.conversionRepo.updateStatus(conversionId, ConversionStatus.COMPLETED, undefined, ctx);
    ...
```

`CONVERSION_SETTLED` credits `@user:{id}:{target}` — a real liability — and the conversion is marked
`COMPLETED` **without any `PAYOUT_DELIVERED` clearing leg** and without any payout being attempted.

`destination` is `optional()` in
[`conversion.schema.ts:86`](../XaneApp/engine/src/adapters/driving/api/schemas/conversion.schema.ts#L86)
and the columns are nullable (migration 11), so this path is reachable from the public API by simply
omitting the field.

This is the **third door** onto the phantom liability. The register records the fix as covering "both
doors a conversion can complete through" — the webhook and the synchronous payout success. This one
was not counted, because it is the path where no payout happens at all.

**It may be intentional** for a custody model where we hold the asset on the user's behalf — in which
case the liability is correct and the bug is only that the conversion says `COMPLETED`. That
ambiguity is itself the finding: **the code does not say which model it is implementing**, and the
two readings differ by an unbounded, compounding liability. It needs a decision, then an assertion
that encodes it.

---

## M6 — MEDIUM — Webhook generic-success completes without the clearing leg

[`webhook-handler.service.ts:206-251`](../XaneApp/engine/src/application/services/webhook-handler.service.ts#L206-L251):
a success webhook for a conversion that is **not** `PAYOUT_PENDING` and **not** deposit-based falls
through to a generic path that writes `PROVIDER_CONFIRMED`, then `COMPLETED`, with **no
`PAYOUT_DELIVERED` posting**. Same phantom-liability shape as M5, reached via the webhook door.

Also here: the idempotency guard at line 104 only short-circuits on `COMPLETED`. A replayed webhook
against a conversion in any other state re-runs the whole path. The ledger's UNIQUE `reference`
catches the double-post on the `PAYOUT_PENDING` branch — but the generic path posts nothing to the
hash-chained ledger, so it has no such backstop.

---

## M7 — MEDIUM — A failed AWAITING_DEPOSIT write orphans a claimed execution

[`execute-swap.use-case.ts:604-605`](../XaneApp/engine/src/application/use-cases/execute-swap.use-case.ts#L604-L605):

```ts
const awaiting = await this.conversionRepo.updateStatus(conversionId, ConversionStatus.AWAITING_DEPOSIT);
if (isErr(awaiting)) return awaiting;
```

It returns without closing or failing the `executions` row already claimed at the top of the use
case. Compare the deposit-persist failure twelve lines above (596-602), which correctly fails **both**
the conversion and the execution.

**Consequence.** The OnSwitch order is live and the customer has been given a deposit instruction,
but the conversion stays at `PROVIDER_CALLED` and the execution stays `PENDING` forever. Because
`executions.quote_id` is UNIQUE, the quote is permanently claimed by an attempt that half-happened —
the exact condition `abortBeforeProvider` exists to prevent, on the one branch that does not use it.

Two more silent exits in the same function:

- Line 608 — `recordEntry` for the audit trace, `Result` discarded.
- Line 653-656 — `completeExecution` returns silently if `transitionExecution` errors, leaving the
  execution `PENDING` after a successful initiate.

---

## M8 — MEDIUM — Unchecked writes inside the reconciliation transaction

[`reconciliation.service.ts:549`](../XaneApp/engine/src/application/services/reconciliation.service.ts#L549):

```ts
await this.conversionRepo.updateStatus(conversion.id, ConversionStatus.COMPLETED, undefined, ctx);
```

Inside `runInTransaction`, `Result` discarded. The `recordEntryPair` above it uses
`abortWith(pairResult.error)` correctly; the status write does not. If it fails, the transaction
commits the ledger entries anyway and reconciliation returns `ok('completed')` for a conversion still
sitting in its old status — which the next sweep will pick up and complete again.

The same pattern appears at `enqueue-swap.use-case.ts:106-107` (pre-provider, so lower severity — no
money has moved) and at `execute-swap.use-case.ts:390-394` and `403-406`.

---

## M9 — MEDIUM — O4 is half-closed

Commit `19a4fc3a` ships the contract battery as closing O4. It closes the *invariant* half only.
[`tests/contract/provider-contract.ts`](../XaneApp/engine/tests/contract/provider-contract.ts) is
hermetic with `global.fetch` stubbed, and says so: *"Happy-path response mapping is covered by each
adapter's own tests and, later, golden fixtures."*

`tests/contract/fixtures/` contains **only a README**. `scripts/record-onswitch-fixtures.ts` exists
and appears never to have been run, though the sandbox is reachable and the key is in `.env`.

So the risk O4 was created to close — *"if the adapter misunderstands the real contract, the fake
agrees with the bug"* — **remains fully open for OnSwitch**, the only live rail. The engine's
float→minor-unit conversion, status mapping and error mapping are still checked only against our own
reading of the docs.

This is the cheapest item on the list to fix: run the recorder.

---

## M10 — LOW — Unguarded JSON parse at composition

[`composition-root.ts:287-288`](../XaneApp/engine/src/infrastructure/composition-root.ts#L287-L288)
calls `JSON.parse(env.TREASURY_BALANCES_JSON)` with no try/catch and no schema validation. Malformed
JSON throws during composition — a boot crash, so it is loud rather than silent, but it bypasses the
Zod validation every other env var gets. A syntactically valid but wrong-shaped value (e.g.
`{"NGN":"500"}`, a string) passes silently and produces string-vs-number comparisons in the drift
check, where `500 === "500"` is `false` and every run reports drift.

---

## What is genuinely solid — verified, not assumed

The audit found no fault with these, and several stale docs claim otherwise:

- **HMAC auth** ([hmac-auth.middleware.ts](../XaneApp/engine/src/adapters/driving/api/middleware/hmac-auth.middleware.ts))
  — absolute ±30s window in both directions, constant-time compare, single-use signatures via Redis
  `SET NX`, **fails closed** on cache error. Correctly wired with the cache on every route.
- **B5 (cache-sourced money) is FIXED.** `ENGINE_BRANCH_GAP_REPORT.md` lists it open; the quote is
  loaded from Postgres and Zod-validated. No `cache.get<QuoteRecord>` remains.
- **B9 (lock ownership) is FIXED.** Same report lists it open; line 559 calls
  `releaseLock(lockKey, lockToken)` — the Lua compare-and-delete.
- **The hash-chained ledger** — append-only triggers, per-account SHA-256 chain, balance
  reconstruction. `verifyChain()` and `reconstructBalances()` do what they claim.
- **`onswitch-deposit-lifecycle.test.ts`** is a genuinely good test — specific expected values,
  real Postgres, drift injection, idempotency by operation count. Its only flaw is M1's oracle.
- **Lint discipline holds.** `no-floating-promises` is **not** in the `tests/**` override block, so a
  missing `await` in a test still fails lint.

---

## Docs that are now wrong (re-baseline before trusting)

| Doc | Stale claim |
|---|---|
| `ENGINE_BRANCH_GAP_REPORT.md` | Lists B5 and B9 as open. Both are fixed. |
| `FINDINGS_REGISTER.md` (2026-07-19 entry) | Says deposit completion posts `CONVERSION_SETTLED` + `PAYOUT_DELIVERED`. The 2026-07-20 pass-through rewrite made this false; it now posts `REVENUE_EARNED` only. |
| `FINDINGS_REGISTER.md` | Phantom liability described as fixed on "both doors". There are at least four doors; M5 and M6 are uncovered. |
| `FINDINGS_REGISTER.md` (O4) | Implied closed by `19a4fc3a`. Half-closed — see M9. |
| `PHASE11_TESTING_READINESS.md` | Entirely JuicyWay-framed with every gate unchecked. Superseded by OnSwitch; needs rewriting or retiring. |
| `CTO_AUDIT_AND_BUILD_PLAN.md` | Day 2/3/4 trackers all ⬜ though much of the work shipped. |

---

## Recommended fix order

1. **M1 + M2 together.** They are one story: stop booking uncollected revenue, and make the detector
   able to see it if we ever do again. Fixing M2 alone would surface M1 as a daily drift alert, which
   is a legitimate interim step if a fix must ship tonight.
2. **M9** — run `npm run record:fixtures`. Cheapest real risk reduction available, and it may itself
   surface adapter-mapping bugs that invalidate assumptions elsewhere in this report.
3. **M3** — decide what we actually charge, then make the quote say it.
4. **M4, M5, M6** — before any custody rail is registered. M5 needs a product decision first.
5. **M7, M8, M10** — routine hardening.

Each fix gets a test written **RED first**, per standing project discipline. For M1 and M2 the test
must fail against today's code for the right reason — a treasury that booked revenue and a wallet
reporting nothing must be flagged, not reported balanced.

---

## Unproven — stated so it is not mistaken for cleared

- **Nothing here was executed.** These findings come from reading. The M1/M2 chain is a code-path
  trace, not an observed production drift; it should be confirmed by a probe against Testcontainers
  before the fix, so the fix has a witness.
- **No conversion has ever reached `COMPLETED` against a real rail.** The OnSwitch sandbox returns a
  placeholder deposit address (`0x…sandbox`) that cannot receive a transfer, so the entire
  completion path — including every assertion in M1 — has only ever run against our own fakes. A
  funded staging key remains the gate on proving any of this end to end.
- **Load and concurrency numbers are unretested this round.** The register's figures stand; nothing
  in this audit re-measured them.
