# XanePay — working rules

## RULE 1 — TEST FIRST. ALWAYS. NO EXCEPTIONS.

**Write the test before the code. Run it. See it FAIL. Only then implement.**

This is not a preference or a style. It is the rule this codebase is built on, and it exists because
this project has repeatedly shipped bugs that a green test suite was actively hiding:

- A 537-green suite hid delivered crypto never being debited from the user's balance. The books
  claimed we owed every customer an asset already sent, and `reconcileLedgerBalances()` reported
  `balanced: true` throughout — because the books *were* internally consistent and had merely
  stopped describing the world.
- A 608-green suite hid revenue being booked from a fee the engine never asked the rail to collect.
  The test that should have caught it asserted `@treasury === FEE` where `FEE` was the very number
  under suspicion — **the test used the bug as its own answer key.**

**Internal consistency is not truth.** A test written after the code tends to assert what the code
already does, which is worth nothing. A test written first has to describe what the code *should*
do, and it is the RED run that proves the test is capable of failing at all.

### The procedure, every time

1. Write the test.
2. **Run it. Watch it fail.**
3. **Read the failure message.** Confirm it fails for the *intended* reason — not a typo, not a
   missing import, not an unrelated error. A test that fails for the wrong reason proves nothing.
4. Implement the smallest change that makes it pass.
5. Run it again. Green.

### When the behaviour already exists (characterization tests)

Pinning existing behaviour still requires a RED run. **Mutate the implementation deliberately** —
break the thing the test claims to protect — confirm the test goes RED, then restore the code
exactly. If the test stays green under mutation, **the test is worthless and must be rewritten.**

This is not optional busywork. It caught a real defect in this repo: a fixture-driven test claimed to
pin rate *rounding*, but the recorded rate rounded and truncated to the same integer, so
`Math.round` → `Math.trunc` left it green. The comment asserting the guarantee was simply false.

### Never

- **Never** write implementation before its test.
- **Never** report a test as passing without having seen it fail first.
- **Never** write a test whose expected value is read from the code path under test — that is the
  answer-key bug above. Derive expectations independently (from raw recorded bytes, from the spec,
  from arithmetic done by hand).
- **Never** edit an existing test's harness to accommodate code not yet written, then wave away the
  resulting failures. Those failures are information.
- **Never** silence a failing test. Fix the code, or fix the test's premise and say which.

---

## RULE 2 — Money paths get proven against the real stack

Mocks agree with our misunderstandings. Every probe of the real system in this project has found
something a green unit suite was hiding, so prefer driving the real thing:

- Integration/e2e run against **Testcontainers Postgres + Redis** (`npm run test:integration`).
  Docker Desktop must be running.
- Provider response mapping is checked against **golden fixtures recorded from the live sandbox**
  (`npm run record:fixtures`), never against bodies we wrote from our reading of the docs.
  The recorder captures the **raw wire response** — never the adapter's own output, which would only
  prove the adapter agrees with itself.
- Fake external rails are faked at the **socket**, not at the port, so our own adapter code still
  executes (float→integer conversion, auth headers, error mapping).

---

## RULE 3 — Repo conventions

- `XaneApp/` is a **nested git repo**. The engine lives on the **`engine` branch**.
- Commit and push to **`private`** only. **Never `origin`.**
- No Claude co-author trailer on commits.
- `npm run verify` (typecheck → lint → unit → integration) must be clean at every commit.
- `npm run test:load` must be re-run if anything on the ledger path changes. Throughput peaks at
  `BULLMQ_CONCURRENCY=5` and gets **worse** above it — never raise it without re-measuring.

---

## RULE 4 — Read the migration, don't guess the schema

Each of these cost a debugging cycle:

- `executions` keys on **`quote_id`**. There is no `conversion_id` column.
- `fee_config` uses **`fee_type`**, not `conversion_type`.
- **Provider secrets live in ENV, NEVER in the DB.** A `providers` row carries only non-secret
  routing metadata (enabled, priority, `api_base_url`). The service key is read from env at call time
  (`ProviderFactory.resolveApiKey` — e.g. `ONSWITCH_SERVICE_KEY`), the same way the webhook path reads
  it. `ProviderRepository` **drops** any secret on write (`api_key_encrypted`/`webhook_secret_encrypted`
  are forced NULL) and returns NULL for them on read, so a leftover value in a legacy row can never be
  consumed. Do **not** re-add secret persistence, an encrypt-at-rest step, or a reveal endpoint — those
  were removed on purpose. Seed via `scripts/seed-onswitch.ts` (repository, metadata only).
- The fee is **always** in the quote's **source** currency. `FeeService.applyFees()` hard-errors
  otherwise. Never default it to NGN — `CRYPTO_TO_FIAT` fees are seeded in USDT.
- `bigint` columns arrive as **strings** from node-postgres. `'100' + 5` is `'1005'`.

---

## RULE 5 — Docs lie; verify against code

Tracker docs in `docs/` have repeatedly been stale in ways that misled audits — findings listed as
open that were fixed, and behaviour described that a later rewrite made false. Before trusting any
claim in `docs/` or `engine/docs/FINDINGS_REGISTER.md`, **check it against the code on the `engine`
branch.** When you find a stale claim, fix the doc in the same commit as the code.
