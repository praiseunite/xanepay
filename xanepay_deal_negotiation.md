# XanePay — What to Demand If You're Building This

## The Situation

They want you to build an institutional-grade financial engine. They can't pay cash. They're promising "benefits when revenue comes." That's a classic sweat equity deal — and it can be fair, **but only if you structure it right.** Without a proper structure, "benefits when revenue comes" means nothing.

> [!CAUTION]
> **"We'll take care of you when we make money" is NOT a deal.** It is a verbal promise with zero legal weight. If you build this without a written agreement, you have ZERO leverage once the code is delivered. They can ghost you, bring in another developer, or simply never pay. You must get everything in writing before you write a single line of code.

---

## The Core Principle: You Are Not a $200 Contractor

If you're building the entire engine — the thing that actually moves the money — you are not a freelancer. You are a **technical co-founder** or, at minimum, a **founding engineer.**

The value you're bringing:

| What You Bring | What It's Worth |
|---|---|
| Architecture design (already done) | $3,000 - $5,000 worth of consulting |
| System documentation (6 docs, production-grade) | $2,000 - $4,000 worth of technical writing |
| Full engine build (onramp + offramp) | $15,000 - $35,000 in development |
| Deployment + infrastructure | $5,000 - $12,000 |
| Ongoing maintenance + on-call | $3,000 - $7,000/month |
| **Domain expertise** (fintech, blockchain, ledger design) | You can't put a price on this — they can't hire this easily |

**Total value you'd be contributing: $25,000 - $56,000+ in the first engagement alone.**

They are offering $200 cash + vague future promises. The gap is enormous. Your deal structure must reflect that.

---

## What to Request: Two-Part Deal

### Part A — Equity (Ownership Stake)

This is the "benefits when the company makes money" part, but formalized.

#### How Much Equity?

| Your Role | Typical Equity Range | Recommended for You |
|---|---|---|
| Freelance dev working cheap | 1-3% | ❌ Too low for what you're doing |
| Founding engineer (first technical hire) | 5-15% | ⚠️ Reasonable if there's a CTO above you |
| Technical co-founder (sole builder) | 15-30% | ✅ **This is you** — you're building the entire revenue engine |
| Equal co-founder | 33-50% | Only if you're a true co-founder from day one |

> [!IMPORTANT]
> **If you are the only person building the product that generates revenue, you should be requesting 15-25% equity minimum.** Without your engine, XanePay is a business plan on paper. You are converting it into a functioning business.

#### Equity Must Be Vested

Never take equity without a **vesting schedule**. This protects both sides:

```
Standard 4-Year Vesting with 1-Year Cliff:

Year 0-1:  Building the engine. If you leave before 12 months, you get 0%.
           This protects THEM — you can't take equity and disappear.

Month 12:  25% of your equity vests (the "cliff").
           You now own that portion regardless of what happens.

Month 13-48: Remaining 75% vests monthly (equal portions).
             Every month you stay, you earn more.

Example with 20% equity:
  - Month 12: 5% vests (25% of 20%)
  - Month 13-48: ~0.42% per month
  - Month 48: Fully vested at 20%
```

**Why this works:**
- They're protected: if you quit early, they don't lose a huge chunk of equity
- You're protected: once vested, they can't take it away
- It aligns incentives: you're both committed long-term

---

### Part B — Revenue Share (Cash Flow Before Exit)

Equity only pays out if the company is sold or raises funding. That could be **years** — or never. You need a revenue share that pays you **as the system generates money.**

#### What to Request

```
Revenue Share Structure:

  XanePay earns revenue from the FX spread on every transaction.
  (Your docs show: 0.5% - 1.5% spread = the company's gross margin)

  Your cut: X% of NET transaction revenue until a cap is reached.

  Option A — Percentage of Revenue (Recommended):
    You receive 10-20% of XanePay's net spread revenue
    Until either:
      (a) You've received $Y total (e.g., $50,000 — roughly fair market value), OR
      (b) The company raises a funding round and pays you a lump sum buyout, OR
      (c) You transition to a salaried CTO role with market-rate compensation

  Option B — Per-Transaction Fee:
    You receive ₦X per completed transaction (onramp or offramp)
    Example: ₦500 per transaction
    At 100 transactions/day = ₦50,000/day = ₦1.5M/month (~$1,000/month)
    This scales with usage — fair for both sides

  Option C — Tiered Revenue Share:
    0-₦10M monthly volume    → 20% of spread revenue to you
    ₦10M-₦50M monthly volume → 15% of spread revenue to you
    ₦50M+ monthly volume     → 10% of spread revenue to you
    This rewards early risk-taking with higher share, then scales down
```

> [!TIP]
> **Option C (tiered) is the most sophisticated and fair.** You get a higher percentage when the revenue is small (because you took the risk), and it gradually decreases as volume grows (because the business needs more operating capital). Investors also find this structure reasonable.

---

## The Non-Negotiables — Your Protection Checklist

These are things you **must** have before writing any code. Not optional. Not "we'll sort it out later."

### 1. Written Agreement (Signed by Both Parties)

```
The agreement must include:
  ✅ Your equity percentage and vesting schedule
  ✅ Revenue share terms (percentage, cap, payment frequency)
  ✅ Your title and role (CTO / Technical Co-Founder / Lead Engineer)
  ✅ Definition of "revenue" (gross vs net, what's included)
  ✅ Payment schedule for revenue share (monthly, quarterly)
  ✅ What happens if they raise funding (does your share get diluted? by how much?)
  ✅ What happens if they sell the company (your equity payout terms)
  ✅ What happens if they bring in another developer
  ✅ IP assignment clause (code ownership — see below)
  ✅ Termination terms (what you keep if either side walks away)
```

> [!CAUTION]
> **No written agreement = no work. Period.** A verbal promise is worth the paper it's printed on. If they're serious about the business, they'll sign. If they won't sign, they're not serious — and you should walk.

### 2. Code Ownership / IP Protection

This is your single biggest leverage point. Structure it carefully:

```
Option A — Code Held in Escrow (Recommended):
  You build. Code lives in YOUR repository.
  They get access to a deployed, running instance.
  Full source code is transferred ONLY when:
    (a) Equity agreement is signed and filed, AND
    (b) First revenue share payment is made, AND
    (c) All terms of the agreement are met

  If they breach the agreement → you retain the code.

Option B — IP Licensed, Not Transferred:
  You retain IP ownership of the engine.
  XanePay gets an exclusive license to use it.
  License is conditional on ongoing revenue share payments.
  If payments stop → license revoked.

Option C — IP Transferred on Vesting:
  IP transfers gradually as your equity vests.
  At 0 months: you own 100% of the code
  At 12 months (cliff): 25% of IP transfers
  At 48 months: full transfer complete
```

> [!IMPORTANT]
> **DO NOT transfer full code ownership on day one.** The code is your leverage. Once they have the code and deploy it, your negotiating position drops to zero. Keep the code in your control until the deal is formalized.

### 3. Access to Business Operations

If you're an equity holder, you have a right to transparency:

```
You should have:
  ✅ Read access to transaction volume and revenue dashboards (you're building these anyway)
  ✅ Monthly revenue reports (so you can verify your revenue share)
  ✅ Notification of any funding rounds, investor meetings, or sale discussions
  ✅ A seat at technical decision-making (you ARE the technical decision-maker)
  ✅ Veto power on technical hires that affect the engine you built
```

### 4. Your Role and Title

```
Your title should be one of:
  - CTO (Chief Technology Officer)
  - Technical Co-Founder
  - Founding Engineer / Lead Architect

NOT:
  - "Freelance Developer"
  - "Contractor"
  - "Backend Developer"

Why it matters:
  - Your LinkedIn, your portfolio, your future career
  - Co-founders and CTOs get equity. Contractors get paid.
  - If they're not willing to give you a title that matches
    your contribution, they don't value your contribution.
```

---

## The Minimum Deal You Should Accept

If you decide to do this, here is the **floor** — the absolute minimum:

```
MINIMUM ACCEPTABLE DEAL:

  1. Equity: 15% with standard 4-year vesting, 1-year cliff
  
  2. Revenue share: 15% of net spread revenue, paid monthly,
     until you've received $40,000 total OR are converted to
     a salaried role at market rate ($3,000+/mo)
  
  3. Title: CTO or Technical Co-Founder
  
  4. Written agreement: Signed before any code is written
  
  5. Code escrow: Source code in your repo until agreement is
     executed and first revenue share is paid
  
  6. Anti-dilution: Your equity cannot be diluted below 10%
     without your written consent
  
  7. Monthly reporting: Access to revenue and transaction data
```

---

## The Ideal Deal (What to Push For)

```
STRONG DEAL:

  1. Equity: 20-25% with 4-year vesting, 1-year cliff
  
  2. Revenue share: Tiered
     - 0-₦10M/mo volume: 20% of spread
     - ₦10M-₦50M/mo: 15%
     - ₦50M+/mo: 10%
     No cap — runs until you're bought out or salaried
  
  3. Title: CTO and Co-Founder
  
  4. Board seat or observer rights (if company formalizes a board)
  
  5. First right of refusal on all technical hires
  
  6. Liquidation preference: In a sale, your equity pays out
     before common shareholders (after investors)
  
  7. Acceleration clause: If the company is sold within 2 years,
     100% of your unvested equity immediately vests
  
  8. Minimum monthly after revenue: Once XanePay generates
     ₦5M+/month in spread, you receive a minimum of ₦300,000/month
     regardless of revenue share calculation
```

---

## Red Flags — Walk Away If You See These

| Red Flag | What It Means |
|---|---|
| "We'll figure out the details later" | They won't. You'll build, they'll ghost. |
| "We don't want to do paperwork yet" | They don't take the business seriously enough to protect either party. |
| "Just trust us" | Trust is great. Contracts are better. |
| "We'll give you equity but can't say how much yet" | They want to decide your share AFTER you've already built it — when your leverage is gone. |
| "We own the idea so we should get more" | Ideas are worth $0. Execution is everything. You ARE the execution. |
| "We'll bring in a real CTO later" | They see you as temporary labor, not a partner. |
| Unwilling to sign anything | **The single biggest red flag.** Run. |

---

## Conversation Script — How to Present This

Here's how to frame it when you talk to them:

> *"I'm interested in building this, and I believe in the product. But let's be real — what I'm building is the entire revenue engine. Without this engine, XanePay doesn't make money. So I'm not a $200 contractor — I'm a co-builder.*
>
> *Here's what I need to move forward:*
>
> *1. A formal agreement that gives me [X]% equity with vesting*
> *2. A revenue share of [X]% of spread revenue once we go live*
> *3. My title as CTO / Technical Co-Founder*
> *4. The source code stays in my control until the agreement is signed*
>
> *I've already built the full architecture and documentation — that's my proof of commitment. Now I need yours. If we're partners, let's make it official. If not, I understand — but I can't build a financial system for free with no protection."*

---

## Final Thought

> [!IMPORTANT]
> **The documentation you've already created is your proof of competence AND your negotiating leverage.** Any developer they hire next would need weeks just to understand what you've already designed. You're not starting from zero — you've already invested significant value. Make sure the deal reflects that.

The question isn't whether the work is worth doing. The question is whether THESE founders will respect your contribution with a fair deal. The answer to that question is in whether they'll sign an agreement.

If they sign → build it. You could end up as CTO of a profitable fintech.
If they won't sign → walk. Your skills will find a better home.
