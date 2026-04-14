# XanePay — Treasury & Custody Model
## Internal Ledger Control, Treasury Architecture & Fund Flow

> **Document Status:** Core Architecture — Required Reading Before Implementation
>
> This document defines how XanePay owns, controls, and accounts for all
> financial positions. It resolves the custody question, defines the treasury
> layer architecture, and establishes the ledger as the absolute source of
> truth — independent of any third-party provider's records.

---

## Table of Contents

1. [The Core Principle — XanePay Controls Its Own Books](#1-the-core-principle)
2. [Non-Custody Model — What the Ledger Actually Represents](#2-non-custody-model)
3. [The Two Treasury Layers](#3-the-two-treasury-layers)
   - 3.1 Fiat Treasury (NGN)
   - 3.2 Crypto Treasury (ETH / USDT)
   - 3.3 NGN Treasury — Three Realistic Control Models
4. [Ledger Accounts — Full Definitions](#4-ledger-accounts--full-definitions)
5. [Execution Paths — Provider vs. Treasury](#5-execution-paths)
6. [New Architecture — Treasury Control Layer](#6-treasury-control-layer-architecture)
7. [ITreasuryPort — The New Required Interface](#7-itreasuryport-interface)
8. [Replenishment Engine](#8-replenishment-engine)
9. [Wallet Custody Infrastructure](#9-wallet-custody-infrastructure)
10. [Impact on Implementation Plan](#10-impact-on-implementation-plan)
11. [Open Questions Before Build](#11-open-questions-before-build)

---

## 1. The Core Principle

> **XanePay controls its own ledger. XanePay controls its own treasury.
> No third party is ever the source of truth for XanePay's financial position.**

This is not a preference — it is a foundational architectural decision with
legal, operational, and technical consequences.

### What This Means in Practice

| Aspect | What XanePay Controls |
|---|---|
| Ledger | XanePay's internal double-entry ledger is the MASTER record. Provider statements are verified against it — not the other way around. |
| Fiat Treasury | XanePay maintains its own NGN working capital in its own designated settlement account. PSPs are collection agents only. |
| Crypto Treasury | XanePay holds its own ETH/USDT inventory in wallets where XanePay controls the keys. |
| Settlement | XanePay settles users from its own inventory. Providers replenish that inventory in the background. |
| Provider Relationship | Providers are **replenishment vendors and collection agents** — not ledger keepers or settlement operators. |

### What XanePay Rejects

| What Is Rejected | Why |
|---|---|
| Provider as source of truth | If BVNK reports a different balance than XanePay's ledger, BVNK is wrong until proven otherwise |
| Provider-controlled settlement timing | Users should not wait for BVNK to decide when ETH leaves |
| Funds held at provider's discretion | A PSP freeze should not block XanePay from operating |
| Ledger outsourced to third party | Any SaaS ledger or provider-managed accounting breaks this principle |

---

## 2. Non-Custody Model — What the Ledger Actually Represents

XanePay does **not** custody user funds in the legal sense.
When a user deposits ₦100,000, that NGN does not belong to XanePay —
XanePay has an **obligation** to that user for ₦100,000.

The ledger records this obligation, not physical possession of the funds.

### The Obligation Chain

```
User deposits ₦500,000
│
├─→ PSP (Paystack/Monnify) physically receives ₦500,000
│   Acts as collection agent. Holds until swept.
│
├─→ XanePay Ledger immediately records:
│   DEBIT  → User Pending Account      ₦500,000
│   CREDIT → System Treasury NGN       ₦500,000
│   (Ledger reflects the obligation before NGN even sweeps)
│
└─→ NGN sweeps from PSP to XanePay's settlement fiat account
    Ledger entry remains unchanged — the obligation was already recorded
```

This matters because:
- The ledger is updated the moment the webhook is verified — not when money physically arrives
- XanePay's books are always current, regardless of PSP processing delays
- If a PSP sweeps slowly, XanePay still knows exactly what it owes users

---

## 3. The Two Treasury Layers

XanePay maintains **two distinct treasury positions** at all times.
Both are reflected in the internal ledger. Both are controlled entirely by XanePay.

### 3.1 Fiat Treasury (NGN)

```
┌──────────────────────────────────────────────────────┐
│                  FIAT TREASURY                       │
│                                                      │
│  Physical Location:                                  │
│    XanePay's corporate NGN settlement account        │
│    (bank account or PSP sub-account designated       │
│     for sweeps — separate from operating funds)      │
│                                                      │
│  Source of Funds:                                    │
│    - Daily/realtime sweeps from PSPs (Paystack,      │
│      Monnify) after user deposit collection          │
│    - Offramp recovery: ETH received → sold for NGN   │
│    - Fee revenue accumulation                        │
│                                                      │
│  Used For:                                           │
│    - Paying out users on offramp (ETH → NGN)         │
│    - Covering fee revenue position                   │
│    - Funding bulk crypto purchases from providers    │
│                                                      │
│  Ledger Account: System Treasury NGN                 │
│  Controlled By: XanePay operations team             │
└──────────────────────────────────────────────────────┘
```

### 3.2 Crypto Treasury (ETH / USDT)

```
┌──────────────────────────────────────────────────────┐
│                  CRYPTO TREASURY                     │
│                                                      │
│  Physical Location:                                  │
│    XanePay-controlled hot wallet(s)                  │
│    Private keys held by XanePay (via Fireblocks,     │
│    BitGo, MPC solution, or self-managed HSM)         │
│                                                      │
│  Source of Funds:                                    │
│    - Bulk purchases from conversion providers        │
│      (BVNK, Juicyway) triggered by replenishment     │
│    - Onramp recovery: ETH received on offramp        │
│      that gets redistributed on next onramp          │
│                                                      │
│  Used For:                                           │
│    - Sending ETH/USDT to users on onramp             │
│      (instant settlement from own inventory)         │
│    - On-chain transactions for offramp receipt       │
│                                                      │
│  Ledger Accounts:                                    │
│    System Treasury ETH                               │
│    System Treasury USDT                              │
│  Controlled By: XanePay (key custody)                │
└──────────────────────────────────────────────────────┘
```

---

### 3.3 NGN Treasury — Three Realistic Control Models

> [!IMPORTANT]
> This section is critical. "XanePay controls its NGN treasury" does not mean
> the same thing in all situations. The level of control depends entirely on
> XanePay's company structure and banking relationships.
> **This is an alternative model decision — not a software decision.**

Because NGN is a regulated fiat currency, it must always physically sit with
a licensed third party (a bank or PSP). The question is: **which third party,
and how much control does XanePay retain over it?**

The three models below are alternatives. XanePay must choose one before build.

---

#### Model A — PSP Balance as Treasury *(MVP / Current Reality)*

```
How it works:
  User pays ₦500,000 via Paystack
      │
      ▼
  Paystack receives and holds the NGN
      │
      ▼
  XanePay's PAYSTACK DASHBOARD shows +₦500,000
  (This is a credit on Paystack's platform — not XanePay's bank)
      │
      ▼
  For offramp payout: XanePay calls Paystack Transfer API
  Paystack executes the bank transfer on XanePay's behalf

Controlled by XanePay:
  ✅  Internal ledger (XanePay's own Postgres DB — fully independent)
  ✅  Who gets paid (XanePay initiates transfers via API key)
  ✅  When payment is initiated
  ⚠️  Physical NGN (sits at Paystack — Paystack is still the custodian)
  ⚠️  Paystack's platform is still the truth for the NGN balance

Risk:
  - Paystack can freeze the account
  - Paystack settling delays affect payout liquidity
  - XanePay's NGN position is only as reliable as Paystack's uptime

Requires:
  - Paystack business account (already exists for most startups)
  - No bank account needed
  - No additional setup beyond existing PSP relationship

Best for: MVP — before company is registered / bank account opened
```

---

#### Model B — Corporate Bank Account *(Full Control — Recommended)*

```
How it works:
  User pays ₦500,000 via Paystack
      │
      ▼
  Paystack collects (collection agent only)
      │  Paystack settles to XanePay's bank account (T+0 / T+1 / T+2)
      ▼
  XanePay's GTBank / Access / Zenith corporate account
  (XanePay is the account holder — true ownership)
      │
      ▼
  For offramp payout: XanePay initiates bank transfer
  directly from their bank (via bank API or PSP Transfers)

Controlled by XanePay:
  ✅  Internal ledger
  ✅  Physical NGN (it's in XanePay's own bank account)
  ✅  Initiating transfers (bank API or Paystack used as transfer tool only)
  ✅  Independence from Paystack — if Paystack is down, NGN is safe in bank
  ✅  No single PSP can freeze XanePay's operating capital

Risk:
  - Settlement lag: Paystack typically settles T+1
    (NGN available next business day, not instantly)
  - Bank account can be frozen by bank or regulator
    (but this requires a formal legal order — harder than PSP freeze)

Requires:
  - CAC-registered Nigerian company
  - Corporate bank account (GTBank, Access, UBA, Zenith, Sterling, etc.)
  - Commercial agreement with Paystack to settle to that bank account
  - Settlement schedule configured (daily, real-time, or per-batch)

Best for: Post-registration, production operations
```

---

#### Model C — Banking-as-a-Service (BaaS) *(Practical Middle Ground)*

```
How it works:
  XanePay opens virtual bank accounts via a BaaS provider:
    - Anchor (anchor.co)
    - Bloc (blochq.io)
    - Bankly
    - Mono

  Each user or flow gets a dedicated virtual account number
  (real NIP/NIBSS account — money goes directly into XanePay's
   virtual account, not into Paystack's pool)
      │
      ▼
  NGN lands in XanePay's virtual sub-account (API-managed)
      │
      ▼
  XanePay initiates transfers via BaaS provider's API

Controlled by XanePay:
  ✅  Internal ledger
  ✅  NGN in dedicated virtual accounts (not pooled with other businesses)
  ✅  Transfer initiation via API
  ✅  No Paystack balance dependency
  ⚠️  BaaS provider is still the underlying custodian
      (but dedicated accounts = more control than a shared PSP pool)

Risk:
  - BaaS provider can freeze or suspend accounts
  - Newer providers — due diligence on financial stability required

Requires:
  - CAC registration (required by all BaaS providers)
  - Compliance onboarding with BaaS provider (KYB process)
  - Integration with BaaS provider's API (instead of or alongside Paystack)

Best for: Startups past registration stage wanting more control
          before setting up a full bank relationship
```

---

#### Summary Comparison

| Factor | Model A: PSP Balance | Model B: Corporate Bank | Model C: BaaS |
|---|---|---|---|
| NGN ownership | Paystack holds it | XanePay's own bank | BaaS provider holds it |
| Control level | Partial ⚠️ | Full ✅ | Good ✅ |
| Freeze risk | PSP decision alone | Legal order required | BaaS decision |
| Setup required | None | CAC + bank account | CAC + KYB onboarding |
| Settlement speed | T+1 (Paystack schedule) | Configurable | Near real-time |
| Recommended phase | MVP only | Production | Pre-bank transition |
| Ledger independence | ✅ Always — regardless of model chosen |

> [!NOTE]
> **The internal ledger is fully XanePay-controlled in ALL three models.**
> XanePay's Postgres database is the master truth for every user obligation
> regardless of where the physical NGN sits. What changes between models is
> the level of control over the physical movement of NGN — not the accounting.

---

## 4. Ledger Accounts — Full Definitions

Every account in the double-entry ledger has a precise real-world meaning.
These definitions are authoritative — they cannot be changed without a
formal architecture review.

### 4.1 User-Side Accounts (Per User)

| Account Name | Type | Real-World Meaning |
|---|---|---|
| `User Pending NGN` | Per-user, per-currency | NGN deposited but not yet confirmed by PSP webhook |
| `User NGN Wallet` | Per-user | Full confirmed NGN balance XanePay owes this user |
| `User ETH Wallet` | Per-user | ETH balance XanePay has committed to deliver (if held on behalf) |

> **Note:** If XanePay does not custody crypto on behalf of users (user has their own external wallet), the `User ETH Wallet` account is not needed. ETH delivery is final when `txHash` is confirmed on-chain. This must be determined and documented before build.

### 4.2 System Treasury Accounts (XanePay-Owned)

| Account Name | Type | Real-World Meaning |
|---|---|---|
| `System Treasury NGN` | System | XanePay's own operational NGN capital — sitting in XanePay's settlement bank account |
| `System Treasury ETH` | System | ETH inventory in XanePay's controlled hot wallet |
| `System Treasury USDT` | System | USDT inventory in XanePay's controlled hot wallet |
| `Fee Revenue NGN` | System | Earned spread/margin in NGN — profit available for withdrawal |
| `Fee Revenue USD` | System | Earned spread/margin in USD equivalent |

### 4.3 Operational / Transit Accounts

| Account Name | Type | Real-World Meaning |
|---|---|---|
| `Pending Settlement` | System | Funds received at PSP but not yet swept into XanePay's settlement account |
| `Provider Replenishment Escrow` | Per-provider | Funds sent *to* a provider specifically for a treasury replenishment operation (not user transactions) |
| `In-Transit ETH` | System | ETH that has been sent from XanePay's wallet, not yet confirmed on-chain |

### 4.4 The Golden Rule

> Every debit has an equal credit. The sum of all account balances is always zero.
> If it does not balance, there is a bug. Not a rounding error. A bug.

---

## 5. Execution Paths

The Treasury-Controlled model defines **two execution paths** for all transactions.
The Routing Engine selects the path automatically based on treasury state.

### Path A — Treasury Path (PRIMARY — Preferred)

Used when: XanePay's treasury holds sufficient inventory for the requested currency.

```
ONRAMP — NGN to ETH (Treasury Path):

  Step 1: User requests ₦500,000 → ETH conversion
  Step 2: XanePay receives NGN via PSP webhook (verified)
  Step 3: Ledger: DEBIT User Pending NGN → CREDIT System Treasury NGN
  Step 4: Treasury check: "Do we have ≥ 0.119 ETH in the hot wallet?"
          YES → Proceed on Treasury Path
  Step 5: XanePay sends 0.119 ETH from its own wallet to user's address
  Step 6: Ledger: DEBIT System Treasury ETH → CREDIT In-Transit ETH
  Step 7: On-chain confirmation received
  Step 8: Ledger: DEBIT In-Transit ETH → balance zeroed (obligation fulfilled)
  Step 9: Background: Replenishment engine evaluates ETH treasury level
          Below threshold → trigger replenishment buy

Settlement time: < 60 seconds (XanePay's own wallet — no provider wait)


OFFRAMP — ETH to NGN (Treasury Path):

  Step 1: User sends ETH to XanePay's deposit address
  Step 2: On-chain monitor confirms receipt
  Step 3: Ledger: DEBIT System Treasury ETH (received) → CREDIT User NGN (obligation)
  Step 4: Treasury check: "Do we have ≥ ₦500,000 NGN in settlement account?"
          YES → Proceed on Treasury Path
  Step 5: XanePay sends NGN from own settlement account to user's bank
  Step 6: Payout webhook confirms
  Step 7: Ledger: DEBIT System Treasury NGN → obligation fulfilled
  Step 8: Background: XanePay now holds new ETH — decides to sell via provider
          to replenish NGN treasury as needed

Settlement time: Bank transfer speed (minutes to hours — same day)
```

### Path B — Provider Path (FALLBACK)

Used when: Treasury balance is insufficient, treasury is paused, or amount
exceeds treasury threshold (very large transactions).

```
ONRAMP — NGN to ETH (Provider Path):

  Step 1–3: Same as Treasury Path (PSP collection and ledger credit)
  Step 4: Treasury check: "Do we have sufficient ETH?"
          NO → Use Provider Path
  Step 5: Route to best conversion provider (BVNK / Juicyway — scored)
  Step 6: Provider converts NGN → ETH and sends to user's wallet directly
  Step 7: Provider webhook confirms delivery
  Step 8: Ledger: DEBIT Provider Replenishment Escrow → closed
  Step 9: No treasury replenishment needed (provider handled end-to-end)

Settlement time: 5–40 minutes (provider's processing + blockchain finality)
```

### Routing Decision Logic

```
function selectExecutionPath(currency, amount):

  1. Is treasury monitoring ACTIVE?
     NO → Fallback to Provider Path

  2. Does treasury have balance ≥ amount + safety buffer?
     NO → Fallback to Provider Path

  3. Is amount within treasury transaction limit?
     (e.g., Treasury Path capped at 5 ETH per transaction)
     NO → Fallback to Provider Path (very large amounts go directly to provider)

  4. Is treasury path configured for this currency pair?
     NO → Fallback to Provider Path

  5. All checks pass → USE TREASURY PATH
```

---

## 6. Treasury Control Layer Architecture

The Treasury Control Layer sits alongside the existing Provider Abstraction Layer.
It is not a replacement — it is an additional execution path that the Routing Engine
can choose.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        XANEPAY ENGINE                                │
│                                                                      │
│  ┌────────────┐   ┌──────────────────────────────────────────────┐  │
│  │    API     │   │             APPLICATION CORE                 │  │
│  │   Layer    │──▶│                                              │  │
│  └────────────┘   │  Use Cases ──▶ Routing Engine                │  │
│                   │                    │                          │  │
│  ┌────────────┐   │                    ├──▶ Treasury Path         │  │
│  │  Webhooks  │──▶│                    │         │                │  │
│  └────────────┘   │                    │         ▼                │  │
│                   │                    │   ITreasuryPort          │  │
│                   │                    │         │                │  │
│                   │                    └──▶ Provider Path         │  │
│                   │                              │                │  │
│                   │                              ▼                │  │
│                   │                     ICryptoConversionPort     │  │
│                   └──────────────────────────────────────────────┘  │
│                                                                      │
├──────────────────────────────────────────────────────────────────────┤
│                    TREASURY CONTROL LAYER                            │
│                                                                      │
│  ┌────────────────────┐    ┌──────────────────────────────────────┐ │
│  │  Crypto Treasury   │    │         Fiat Treasury                │ │
│  │                    │    │                                      │ │
│  │  Hot Wallet        │    │  Settlement Account                  │ │
│  │  (XanePay keys)    │    │  (XanePay's bank/PSP sub-account)   │ │
│  │                    │    │                                      │ │
│  │  ETH position      │    │  NGN position                        │ │
│  │  USDT position     │    │  Swept from PSP payins               │ │
│  └────────────────────┘    └──────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │               REPLENISHMENT ENGINE                           │   │
│  │                                                              │   │
│  │  Monitor ETH balance → below threshold → buy from provider   │   │
│  │  Monitor NGN balance → below threshold → alert + action      │   │
│  │  Monitor USDT balance → rebalance as needed                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 7. ITreasuryPort Interface

`ITreasuryPort` is now a **required interface** — not optional.
It is the contract that all treasury wallet implementations must satisfy.

```typescript
export interface ITreasuryPort {

  /**
   * Get XanePay's current treasury balance for a given currency.
   * This queries the actual wallet/account — not just the ledger.
   * Used to reconcile ledger vs. actual positions.
   */
  getBalance(currency: 'NGN' | 'ETH' | 'USDT'): Promise<{
    ledgerBalance: number;    // what the ledger says we hold
    actualBalance: number;    // what the wallet/account actually holds
    discrepancy: number;      // flag if > 0 for reconciliation
  }>;

  /**
   * Check whether the treasury can cover a given amount.
   * Includes safety buffer check (configurable %).
   */
  hasSufficientBalance(params: {
    currency: 'NGN' | 'ETH' | 'USDT';
    amount: number;
    includeSafetyBuffer?: boolean;   // default true
  }): Promise<boolean>;

  /**
   * Send crypto from XanePay's own hot wallet to a destination address.
   * Called during onramp when using Treasury Path.
   */
  sendCrypto(params: {
    toAddress: string;
    amount: number;
    currency: 'ETH' | 'USDT';
    reference: string;         // links to transaction ID
    transactionId: string;
  }): Promise<{
    txHash: string;
    estimatedConfirmationTime: number;   // seconds
  }>;

  /**
   * Send NGN from XanePay's settlement account to a user's bank.
   * Called during offramp when using Treasury Path.
   */
  sendFiat(params: {
    bankCode: string;
    accountNumber: string;
    accountName: string;
    amount: number;            // in kobo
    reference: string;
    narration: string;
    transactionId: string;
  }): Promise<{
    payoutReference: string;
    status: 'PENDING' | 'PROCESSING';
  }>;

  /**
   * Get the deposit address for receiving on-chain crypto.
   * Used in offramp — this is where users send their ETH.
   * Address is deterministic per user to enable monitoring.
   */
  getDepositAddress(params: {
    userId: string;
    currency: 'ETH' | 'USDT';
  }): Promise<{
    address: string;
    network: string;           // 'ethereum', 'polygon', etc.
  }>;

  /**
   * Confirm whether an on-chain transaction has been received
   * and reached minimum confirmation count.
   */
  confirmOnChainReceipt(params: {
    txHash: string;
    expectedAmount: number;
    currency: 'ETH' | 'USDT';
    minConfirmations?: number;   // default: 12 for ETH
  }): Promise<{
    confirmed: boolean;
    confirmations: number;
    actualAmount: number;
  }>;

  /**
   * Trigger a replenishment buy from a named conversion provider.
   * Called by the Replenishment Engine when treasury drops below threshold.
   * This uses the PROVIDER PATH specifically for treasury management —
   * not for user-facing transactions.
   */
  triggerReplenishment(params: {
    currency: 'ETH' | 'USDT';
    targetAmount: number;      // buy this much
    provider: string;          // 'BVNK' | 'Juicyway'
    maxSlippage?: number;      // reject if rate worse than this % from quote
  }): Promise<{
    replenishmentTransactionId: string;
    estimatedDelivery: Date;
  }>;
}
```

---

## 8. Replenishment Engine

The Replenishment Engine is a background service that ensures the treasury
never runs dry. It is separate from the transaction processing pipeline.

### Replenishment Triggers

```
Trigger 1 — Threshold Alert (Automatic)
  Cron: Every 5 minutes
  Check: Is ETH balance < REPLENISHMENT_THRESHOLD_ETH?
  Action: Trigger replenishment buy for (TARGET_ETH_BALANCE - current)
  
Trigger 2 — Post-Transaction Evaluation
  After every successful onramp via Treasury Path:
  Check: Is remaining ETH balance < SAFETY_BUFFER_ETH?
  Action: Immediate replenishment (don't wait for cron)

Trigger 3 — Manual Trigger (Admin API)
  Operations team can force an immediate replenishment
  via: POST /admin/treasury/replenish
```

### Replenishment Flow

```
1. Engine detects ETH balance below threshold
2. Fetch quotes from all active conversion providers in parallel
3. Select cheapest quote within slippage tolerance
4. Execute bulk buy (provider path — NGN → ETH, treasury-to-treasury)
5. Record replenishment in a dedicated replenishment_events table
6. On provider webhook: ETH arrives in XanePay's hot wallet
7. Update System Treasury ETH ledger account
8. Alert operations: "Treasury replenished: +X ETH via BVNK"
```

### Configurable Thresholds (Admin-Managed, No Code Change)

| Parameter | Default | Description |
|---|---|---|
| `ETH_REPLENISHMENT_THRESHOLD` | 1.0 ETH | Buy more when balance drops below this |
| `ETH_TARGET_BALANCE` | 5.0 ETH | Fill up to this amount when replenishing |
| `ETH_SAFETY_BUFFER` | 0.5 ETH | Never let treasury go below this |
| `NGN_REPLENISHMENT_THRESHOLD` | ₦5,000,000 | Alert when NGN drops below this |
| `NGN_TARGET_BALANCE` | ₦20,000,000 | Target operating capital |
| `MAX_SLIPPAGE_PERCENT` | 0.5% | Reject replenishment if rate is this worse than quote |
| `TREASURY_PATH_MAX_TXN_ETH` | 2.0 ETH | Above this, always use Provider Path |

---

## 9. Wallet Custody Infrastructure

This is the highest-risk component of the treasury architecture.
The private key is the treasury. If it is lost, the funds are gone.
If it is stolen, the funds are gone.

### Custody Options (Decision Required)

| Option | Security | Complexity | Cost |
|---|---|---|---|
| **Fireblocks** | Institutional grade (MPC) | Low — managed service | High ($1,000+/mo) |
| **BitGo** | Institutional grade (multi-sig) | Low — managed service | High |
| **Self-managed MPC wallet** | High if implemented correctly | Very high | Engineering time |
| **Simple HD wallet (self-managed)** | Acceptable for MVP | Medium | Low |

> [!IMPORTANT]
> For MVP, a simple HD wallet (BIP-44 derivation, keys in secrets manager)
> is acceptable IF transaction sizes are small and treasury balances are low.
> As volumes grow, migration to institutional custody (Fireblocks/BitGo) is
> STRONGLY recommended. This migration MUST be planned before go-live.

### Key Management Requirements (Non-Negotiable)

```
- Private keys NEVER stored in the database
- Private keys NEVER logged
- Private keys stored in a dedicated secrets manager (AWS KMS / HashiCorp Vault)
- Hot wallet holds only operating inventory — excess moved to cold storage
- Multi-signature required for transactions above configurable threshold
- All wallet operations logged with operator identity and timestamp
- Wallet addresses generated deterministically (HD wallet) — single seed backup
```

### On-Chain Monitoring Infrastructure

For offramp (user sends crypto → XanePay receives it), XanePay needs to
monitor the blockchain for incoming deposits:

```
Options:
  A. Blockchain node provider: Infura / Alchemy / QuickNode
     → Subscribe to address-level event webhooks
     → Low cost, managed, reliable

  B. Self-hosted node (geth / nethermind)
     → Full control, no third-party dependency
     → High operational cost, requires DevOps expertise

Recommendation: Start with Alchemy/Infura (Option A).
Self-hosted node is Phase 3+ when volumes justify it.
```

---

## 10. Impact on Implementation Plan

This treasury architecture changes the existing implementation plan.
The following updates must be made:

### What Moves from Optional to Required

| Item | Previous Status | New Status |
|---|---|---|
| `ITreasuryPort` interface | Optional (Phase 3+) | **Required — Phase 1** |
| `System Treasury NGN` ledger account | Existed but undefined | **Formally defined — Phase 0** |
| `System Treasury ETH/USDT` ledger accounts | Did not exist | **Required — Phase 0** |
| Wallet custody decision | Not mentioned | **Required before Phase 2** |
| On-chain monitoring | Optional | **Required for Offramp** |
| Replenishment Engine | Not mentioned | **Required — Phase 4** |

### New Phase Added — Phase 0.5: Treasury Foundation

Between Phase 0 (Foundation) and Phase 1 (Core Engine), a new sub-phase
is needed:

```
Phase 0.5 — Treasury Foundation

- [ ] Choose and configure wallet custody solution (Fireblocks / HD wallet)
- [ ] Generate and secure treasury wallet (ETH/USDT hot wallet)
- [ ] Configure designated NGN settlement account with banking partner
- [ ] Set up on-chain monitoring (Alchemy/Infura webhook subscriptions)
- [ ] Define all treasury ledger accounts in schema
- [ ] Define ITreasuryPort interface (full TypeScript contract)
- [ ] Write TreasuryWalletAdapter (implements ITreasuryPort)
- [ ] Write ReplenishmentEngine (cron + threshold evaluation)
- [ ] Configure all replenishment thresholds in providers config
- [ ] Integration test: send test ETH from treasury wallet + confirm receipt
- [ ] Integration test: PSP fiat sweep into settlement account
```

### Updated Routing Engine

The Routing Engine must be updated to evaluate execution path first:

```typescript
async function route(transaction: Transaction): Promise<ExecutionPlan> {
  // Step 1: Check treasury path eligibility
  const treasuryEligible = await treasuryPort.hasSufficientBalance({
    currency: transaction.toCurrency,
    amount: transaction.toAmount,
    includeSafetyBuffer: true,
  });

  if (treasuryEligible && transaction.toAmount <= TREASURY_PATH_MAX_LIMIT) {
    return { path: 'TREASURY', executor: treasuryPort };
  }

  // Step 2: Fall back to provider scoring
  const quotes = await fetchQuotesFromAllProviders(transaction);
  const bestProvider = scoreProviders(quotes);
  return { path: 'PROVIDER', executor: bestProvider };
}
```

---

## 11. Open Questions Before Build

> [!IMPORTANT]
> These must be answered and documented before Phase 0.5 begins.
> Building without answers to these is building on sand.

| # | Question | Who Decides | Impact |
|---|---|---|---|
| 1 | **Which NGN Treasury Model?** PSP Balance (Model A), Corporate Bank (Model B), or BaaS (Model C)? See Section 3.3. | XanePay founders + legal | Changes the entire fiat treasury architecture and what `ITreasuryPort.sendFiat()` calls |
| 2 | **Is XanePay CAC-registered?** If not, Models B and C are unavailable — MVP must use Model A. | XanePay founders | Determines which NGN model is available at launch |
| 3 | **What wallet custody solution?** Fireblocks, BitGo, or self-managed HD wallet? | XanePay founders + engineering | Determines Phase 0.5 crypto treasury architecture |
| 4 | **What is the initial treasury capitalisation?** How much ETH and NGN will XanePay pre-fund at launch? | XanePay finance | Determines replenishment thresholds and Treasury Path availability at launch |
| 5 | **What is the Treasury Path transaction size limit?** Above what ETH amount does the system always use Provider Path? | XanePay risk team | Routing Engine configuration |
| 6 | **Does XanePay hold crypto on behalf of users?** Or does crypto always go immediately to the user's external wallet? | XanePay product team | Determines whether `User ETH Wallet` ledger account is needed |
| 7 | **Which blockchain(s)?** ETH mainnet only, or also Polygon/Base for lower gas? | XanePay product + engineering | On-chain monitoring infrastructure, gas management |
| 8 | **Who has authority to trigger manual replenishment?** What approval is required for large treasury buys? | XanePay operations | Admin API access controls |
| 9 | **Regulatory position:** Has legal counsel reviewed holding crypto as operating inventory in this jurisdiction? | XanePay legal | May affect treasury limits and disclosure requirements |

---

*This document is the authoritative definition of XanePay's treasury and custody model.*
*It supersedes any vague references to "treasury" in the PRD or implementation plan.*
*All phase planning and interface design must be consistent with this document.*

*Related documents:*
- *Full technical architecture: `docs/PRD.md`*
- *Onramp scope: `docs/ONRAMP_REQUIREMENTS.md`*
- *Offramp scope: `docs/OFFRAMP_REQUIREMENTS.md`*
- *Implementation phases: `docs/IMPLEMENTATION_PLAN.md`*
