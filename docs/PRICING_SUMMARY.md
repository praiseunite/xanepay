# XanePay — Pricing Summary
## Onramp & Offramp Engine — Scope & Cost Reference

> This document provides a consolidated pricing reference for all
> engagement models. Use this alongside the detailed requirements
> documents to clearly define scope before any agreement is made.
>
> - Detailed onramp scope → `docs/ONRAMP_REQUIREMENTS.md`
> - Detailed offramp scope → `docs/OFFRAMP_REQUIREMENTS.md`
> - Full technical architecture → `docs/PRD.md`

---

## Why $100 Is Not a Valid Price for This Work

Before the pricing tables, it is important to establish context.

The XanePay engine is not a simple API integration project.
It is a **financial orchestration system** with real money,
real users, and real consequences for failures.

The following is what this work actually involves:

| Component | What It Requires |
|---|---|
| Double-entry ledger | Financial accounting expertise, ACID database design |
| Webhook security pipeline | HMAC signature verification, replay attack prevention |
| Transaction state machine | Formal state design, all failure paths explicitly handled |
| Idempotency system | Every operation safe to retry — prevents double charges |
| On-chain monitoring (offramp) | Blockchain node interaction, confirmation counting |
| Dynamic Provider Registry | SaaS-grade admin system — providers managed without code |
| Retry engine | Distributed job queues, exponential backoff, DLQ handling |
| Reconciliation engine | Nightly financial reconciliation against external providers |
| Deployment + infrastructure | Cloud setup, CI/CD, secrets management, SSL, monitoring |

Every one of these components must work correctly with real money.
The cost of a bug is not a broken UI — it is lost funds.

> **$100 buys approximately 1 hour of a junior developer's time.**
> This project requires a minimum of 6–16 weeks of senior
> full-stack + blockchain engineering, depending on scope.
> The prices below reflect that reality.

---

## Pricing Tables

### Option A — Onramp Engine Only

*User sends NGN → receives Crypto in wallet*

| Scope | Time Estimate | Price (USD) | Price (NGN) |
|---|---|---|---|
| **Starter:** 1 PSP + 1 Conversion Provider + Basic routing | 6–8 weeks | $8,000 – $12,000 | ₦6M – ₦9M |
| **Standard:** 2 PSPs + 2 Providers + Dynamic Registry + Observability | 10–12 weeks | $15,000 – $20,000 | ₦11M – ₦15M |
| **Complete:** All features per ONRAMP_REQUIREMENTS.md | 12–14 weeks | $18,000 – $25,000 | ₦13M – ₦18M |
| + Deployment add-on | +2–3 weeks | +$5,000 – $8,000 | +₦3.5M – ₦6M |

---

### Option B — Offramp Engine Only

*User sends Crypto → receives NGN in bank account*

> The offramp is priced higher than the onramp due to:
> on-chain deposit monitoring, a two-step async flow,
> bank payout failure handling, and partial-amount edge cases.

| Scope | Time Estimate | Price (USD) | Price (NGN) |
|---|---|---|---|
| **Starter:** 1 Conversion Provider + 1 PSP Payout + On-chain monitoring | 7–10 weeks | $10,000 – $15,000 | ₦7.5M – ₦11M |
| **Standard:** 2 Providers + 2 PSPs + Dynamic Registry + Observability | 12–14 weeks | $18,000 – $24,000 | ₦13M – ₦17M |
| **Complete:** All features per OFFRAMP_REQUIREMENTS.md | 14–16 weeks | $22,000 – $30,000 | ₦16M – ₦22M |
| + Deployment add-on | +2–4 weeks | +$6,000 – $10,000 | +₦4.5M – ₦7.5M |

---

### Option C — Onramp + Offramp (Combined)

*Full bidirectional engine — NGN ↔ Crypto*

> Combined pricing is more cost-efficient than doing each separately
> because the Ledger System, Routing Engine, State Machine, Dynamic
> Provider Registry, Observability Console, and all infrastructure
> are built once and shared by both directions.

| Scope | Time Estimate | Price (USD) | Price (NGN) |
|---|---|---|---|
| **MVP:** 1 PSP + 1 Conversion Provider, both directions | 10–12 weeks | $15,000 – $20,000 | ₦11M – ₦15M |
| **Standard:** 2 PSPs + 2–3 Providers + Dynamic Registry + Observability | 14–16 weeks | $25,000 – $35,000 | ₦18M – ₦26M |
| **Complete:** All features, both directions, full admin system | 16–20 weeks | $32,000 – $45,000 | ₦23M – ₦33M |
| + Deployment add-on | +3–4 weeks | +$7,000 – $12,000 | +₦5M – ₦9M |

---

### Option D — Monthly Retainer

*Best for: ongoing development, iterative delivery, product ownership role*

> This is the recommended model when the developer takes on
> full product ownership — building, deploying, maintaining,
> and iterating on the engine over time.

| Engagement Type | USD / Month | NGN / Month |
|---|---|---|
| **Development only** (no infra responsibility) | $3,000 – $5,000 | ₦2.2M – ₦3.7M |
| **Development + Deployment** (infra managed) | $4,500 – $7,000 | ₦3.3M – ₦5.1M |
| **Full Product Ownership** (build + deploy + maintain + on-call) | $6,000 – $10,000 | ₦4.4M – ₦7.3M |

*Minimum engagement: 3 months for retainer model.*

---

## Deployment Scope — What Is Included

When deployment is in scope, the following is covered
(in addition to all development deliverables):

| Category | What Is Delivered |
|---|---|
| **Cloud Infrastructure** | Server/container provisioning, database setup, Redis, networking |
| **On-Chain Infrastructure** (Offramp) | RPC node setup (Infura / Alchemy / self-hosted), address monitoring |
| **Security** | SSL certificates, secrets manager configuration, IP allowlists |
| **CI/CD Pipeline** | Automated test + deploy on push to `develop` and `main` |
| **Monitoring** | Error tracking (Sentry), uptime monitoring, log aggregation |
| **Alerting** | Operational alerts: provider failure, stuck transactions, DLQ overflow |
| **Go-Live Support** | Sandbox → production cutover, first live transactions monitored |
| **Runbook** | Documented procedures for common operational issues |

---

## What Affects the Final Price

The following factors move the price up or down within the ranges:

| Factor | Price Impact |
|---|---|
| Number of providers to integrate | +$2,000 – $4,000 per additional provider |
| KYC provider integration | +$3,000 – $5,000 |
| DEX / on-chain swap integration (Uniswap etc.) | +$5,000 – $8,000 |
| Admin dashboard UI (frontend) | +$5,000 – $12,000 |
| Multi-currency support beyond NGN/ETH | +$3,000 per additional pair |
| SLA / on-call requirement | +20–30% to monthly retainer |
| Expedited timeline (faster than estimates) | +25–40% premium |
| Provider sandbox access not provided | +1–2 weeks (cost to investigate undocumented APIs) |

---

## How to Scope an Engagement

Before agreeing a price, these items must be confirmed in writing:

### 1. Scope Definition
- [ ] Which direction(s): Onramp only / Offramp only / Both?
- [ ] Which providers are in scope (named, with sandbox access promised)?
- [ ] Is the Dynamic Provider Registry (admin management) included?
- [ ] Is the Observability/Resolution Console included?
- [ ] Is deployment in scope?
- [ ] Is KYC integration in scope?
- [ ] What is the target go-live date?

### 2. Access & Dependencies
- [ ] Sandbox API credentials provided by client before development begins
- [ ] A domain/server ready for webhook URL registration
- [ ] Cloud provider account access (if deployment is in scope)
- [ ] KYC provider API access (if KYC is in scope)
- [ ] XaneApp frontend team contact for API integration alignment

### 3. Responsibilities Outside This Scope
The following are the client's responsibility unless explicitly included:
- User authentication and registration (XaneApp)
- Frontend mobile/web UI (XaneApp)
- KYC provider selection and contracting
- Bank account compliance and regulatory licensing
- Provider commercial agreements (Paystack, BVNK, etc.)

### 4. Payment Terms (Recommended)
| Milestone | Payment |
|---|---|
| Agreement signed | 30% upfront |
| Phase 1 complete (core engine working in sandbox) | 30% |
| Phase 2 complete (first full flow working end-to-end) | 20% |
| Production deployment and go-live | 20% |

---

## Quick Reference Card

```
ONRAMP ONLY
  Starter (1 provider, no deployment):    $8K  – $12K
  Standard (2 providers + admin):         $15K – $20K
  + Deployment:                           +$5K – $8K

OFFRAMP ONLY
  Starter (1 provider, no deployment):    $10K – $15K
  Standard (2 providers + admin):         $18K – $24K
  + Deployment:                           +$6K – $10K

BOTH DIRECTIONS (Combined)
  MVP (1 provider each, no deploy):       $15K – $20K
  Standard (2-3 providers + admin):       $25K – $35K
  + Deployment:                           +$7K – $12K

MONTHLY RETAINER
  Dev only:                               $3K  – $5K /mo
  Dev + Deploy:                           $4.5K – $7K /mo
  Full Product Ownership:                 $6K  – $10K /mo

WHAT $100 GETS YOU:  ~1 hour of junior developer time.
                     Not a financial orchestration engine.
```

---

*All figures are in USD. NGN equivalents calculated at approximately ₦1,550/USD.*
*Figures reflect fair market rates for senior full-stack + blockchain engineering.*
*Prices are estimates; final scope document required before a fixed-price agreement.*
