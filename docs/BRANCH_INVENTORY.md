# XanePay — Full Branch Inventory

_Generated 2026-07-23. Enumerated across both repos and both remotes (`origin` = `github.com/xaneApp/XaneApp`, `private` = `github.com/praiseunite/xanepay_private`) after `git fetch --all --prune`._

There are **two git repos**:

1. **Root repo** `c:\Projects\XanePay` → remote `origin = github.com/praiseunite/xanepay.git` (docs/planning wrapper).
2. **Nested repo** `XaneApp/` → the actual product. Remotes: `origin` (team) + `private` (Praise's mirror). This is where all the real branches live.

---

## Root repo (`praiseunite/xanepay`)

| Branch | Last commit | Date | Author | Subject |
|---|---|---|---|---|
| `main` (local) | `4c99637` | 2026-04-14 | Praise Udoh | Merge branch 'main' … |
| `origin/main` | `32e7ecc` | 2026-04-14 | Praise Udoh | Delete pricing details … |

Only a `main`. This repo just wraps planning docs.

---

## XaneApp repo — REMOTE branches

Sorted newest-first by last commit. Language counts are source files (ts/tsx/js/jsx/sol) in the branch tree.

| Branch | Last commit | Date | Author | Languages | Notes |
|---|---|---|---|---|---|
| `private/engine` | `19a4fc3a` | 2026-07-21 | Praise Udoh | **468 ts**, 15 tsx, 7 js, 22 sol | Praise's engine mirror — most advanced TS engine |
| `origin/backend` | `c5e1a2bb` | 2026-07-21 | vida | (JS backend) | Active team backend, JS |
| `origin/staging-backend` | `80773570` | 2026-07-21 | vida | 111 js, 21 ts, 39 tsx | Staging of team backend (JS + some frontend) |
| `origin/frontend` | `6f8a1eec` | 2026-07-20 | theloneson | 113 js, 21 ts, 35 tsx, 1 sol | **Mobile app (Expo/React Native)** — mostly JS |
| `origin/main` | `ab11ef24` | 2026-07-01 | Victor | 167 ts, 15 tsx, 5 js, 22 sol | Team mainline — mostly TS |
| `origin/revert-8-blockchain` | `484293c8` | 2026-07-01 | Victor | 167 ts, 15 tsx, 5 js, 22 sol | Revert branch |
| `origin/blockchain` | `7fd42fff` | 2026-07-01 | Victor | 182 ts, 15 tsx, 6 js, 22 sol | Blockchain/contracts work (TS + Solidity) |
| `origin/victor-backend` | `788f33f6` | 2026-06-30 | Victor Mbah | **69 js, 0 ts** | **Pure-JS backend rewrite** |
| `origin/engine` | `42c8da35` | 2026-06-22 | Praise Udoh | 167 ts, 15 tsx, 5 js, 22 sol | Team-side engine (TS) — older than private/engine |
| `origin/staging` | `29d50b4f` | 2026-06-17 | Udoh, Idopise | 99 ts, 15 tsx, 5 js, 22 sol | Staging (TS) |
| `private/engine-buildout` | `939891c3` | 2026-06-13 | Praise Udoh | 385 ts, 15 tsx, 7 js, 22 sol | Engine buildout mirror (TS) |
| `origin/stage_zero` | `46ba423e` | 2026-05-26 | Praise Udoh | **57 js, 0 ts** | JS backend snapshot |
| `origin/temp-phone-auth-fixes` | `b33cd09e` | 2026-05-22 | vida | **57 js, 0 ts** | JS backend, phone-auth fixes |
| `private/blockchain` | `77c199df` | 2026-04-19 | icekidtech | — | Older blockchain mirror |
| `private/frontend` | `481aa54f` | 2026-04-19 | victor mbah | — | Older frontend mirror |
| `private/backend` | `45c8d9e8` | 2026-04-06 | victor mbah | — | Older backend mirror (JS) |
| `origin/xanpay` | `245c8561` | 2026-02-24 | forrune | 39 js, 39 tsx, 16 ts, 1 sol | Early app ("Add Fund Card") |
| `private/xanpay` | `245c8561` | 2026-02-24 | forrune | same | Mirror of above |
| `origin/xanpay-backend` | `a88094ec` | 2026-02-16 | forrune | **32 js, 2 ts** | **Earliest backend — JS (Paystack)** |
| `private/xanpay-backend` | `a88094ec` | 2026-02-16 | forrune | same | Mirror |
| `private/main` | `f8cf2809` | 2026-02-13 | Udoh, Idopise | — | Old mainline mirror |
| `private/staging` | `0ff56a69` | 2026-01-14 | Udoh, Idopise | — | Old staging mirror |
| `origin/develop` | `b0ee4e03` | 2025-09-16 | theloneson | 29 ts, 15 tsx, 3 js | Oldest — monorepo setup |
| `private/develop` | `b0ee4e03` | 2025-09-16 | theloneson | same | Mirror |

---

## XaneApp repo — LOCAL branches (on this machine)

| Branch | Last commit | Date | Author | Subject |
|---|---|---|---|---|
| `engine` (checked out) | `0ef8290a` | 2026-07-23 | Praise Udoh | test(engine): OnSwitch deposit-based e2e money-path |
| `engine-money-integrity-wip` | `eb4bfc48` | 2026-07-11 | Praise Udoh | wip(engine): money-integrity work |
| `engine-buildout` | `0e80f609` | 2026-06-16 | Praise Udoh | chore(engine): security review (OWASP API Top 10) |
| `origin-sync` | `fbff4bdb` | 2026-05-15 | Praise Udoh | feat: Phase 4 — Error handler middleware |
| `engine-architecture` | `5626e042` | 2026-05-08 | Praise Udoh | feat: Phase 3 — Logging system |
| `engine-phase-1` | `1e107d77` | 2026-04-29 | Praise Udoh | feat: Phase 6 redis caching layer |
| `main` | `f8cf2809` | 2026-02-13 | Udoh, Idopise | Merge PR #7 |

Local `engine` (`0ef8290a`, 2026-07-23) is **ahead of** `private/engine` (`19a4fc3a`, 2026-07-21) — unpushed work.

---

## What this says about the TypeScript → JavaScript question

The branches make the real history legible:

- **The "backend" splits into two lineages.** A **JavaScript** lineage — `xanpay-backend` (earliest, forrune, Feb) → `stage_zero` / `temp-phone-auth-fixes` (57 JS) → `victor-backend` (69 JS, Victor Mbah) → `origin/backend` (vida, current) — and a **TypeScript** lineage — `engine` / `engine-*` (Praise, up to 468 TS on `private/engine`).
- **The mobile app is `origin/frontend`: Expo / React Native**, and it carries `ts`/`tsx` files — so **TypeScript already runs the Android side.** This directly contradicts "TS can't be deployed to Android."
- The JS backend is a **separate lineage authored by different people** (forrune → Victor → vida), not a migration forced by Android. The Node backend never runs on the phone regardless of language; the app reaches it over HTTP.

**Conclusion:** the JS backend is a team/authorship split, not a technical necessity. "Couldn't deploy a TS backend to Android" is not supported by the repo — Android (frontend) is itself TS-capable, and the backend is a hosted Node service independent of the app.
