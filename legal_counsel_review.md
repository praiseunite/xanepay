# LEGAL REVIEW MEMORANDUM

---

**PRIVILEGED & CONFIDENTIAL — FOR ADDRESSEE ONLY**

---

**TO:** [Your Full Name] ("Client")

**FROM:** [Counsel Name], Legal Adviser

**DATE:** April 20, 2026

**RE:** Review of XanePay Services Agreement dated April 18, 2026, between Xane LLC ("the Company") and Client ("the Developer")

**INSTRUCTION:** Client forwarded the above-referenced Agreement for independent legal review prior to execution. Client has requested an assessment of the Agreement's fairness, risk allocation, enforceability, and alignment with the verbal terms discussed between the parties.

---

## EXECUTIVE SUMMARY

Upon thorough review, **this Agreement is substantially one-sided in favour of the Company and exposes the Developer to disproportionate legal, financial, and personal risk that is not commensurate with the nature or value of the engagement.** The contract designates the Developer as "Technical Lead" with responsibility for "backend infrastructure, system architecture, and coordination with the team and founders" — functionally a founding-level technical leadership role — while compensating at a flat fee of ₦700,000, which represents approximately 1–3% of the market value of the deliverables demanded. Critically, Section 6 vests immediate and irrevocable ownership of all intellectual property in the Company from the moment of creation and prohibits the Developer from retaining any protective interest, while Section 5 grants the Company unilateral power to reject milestones with its decision being "final and binding" and with no independent recourse available to the Developer. Sections 9.5 and 15 together collect the Developer's government-issued identification and residential address and link them explicitly to a personal asset enforcement clause for refund disputes — a provision grossly disproportionate to a sub-₦1M engagement. **In its current form, this Agreement is not suitable for execution.**

---

## STRUCTURE OF THIS OPINION

| Part | Subject |
|---|---|
| A | Scope and Role: Misalignment Between Contract and Verbal Terms |
| B | Intellectual Property and Code Ownership: Elimination of Developer Leverage |
| C | Compensation and Milestone Structure: Unconscionable Risk Allocation |
| D | Technical Milestone Checklist: Deliverables Outside Agreed Scope |
| E | Termination and Refund Provisions: Internal Contradictions and Disproportionate Remedy |
| F | Personal Liability and Identity Verification: Oppressive Enforcement Provisions |
| G | Miscellaneous Provisions |
| H | Summary of Required Amendments |

---

## A. SCOPE AND ROLE — MISALIGNMENT WITH VERBAL TERMS

### Contract Language:

**Section 1 (Definition):**
> *"'XanePay' refers to all infrastructure, systems, and financial orchestration technology developed under the Company."*

**Section 2 (Role & Scope):**
> *"The Developer will act as **Technical Lead** for XanePay, responsible for **backend infrastructure, system architecture, and coordination with the team and founders**."*

**Signature Block:**
> *Title: **Backend Developer***

### Analysis:

Three distinct problems arise from these provisions:

**1. The Role Exceeds the Agreed Engagement.**

The Client was verbally engaged to build a **specific, self-contained financial orchestration engine** (fiat-to-crypto/crypto-to-fiat conversion), not to serve as "Technical Lead" responsible for the Company's entire backend infrastructure, system architecture, and team coordination. The Company represented that its existing backend was approximately 95% complete. The contractual role — Technical Lead with infrastructure, architecture, and coordination responsibilities — constitutes a **founding-level technical leadership position** that is materially different from a specialist engine build.

A Technical Lead with these responsibilities would command equity participation and monthly compensation in the range of ₦500,000–₦1,500,000/month or its equivalent, depending on stage. To compress this role into a ₦700,000 flat fee with no equity is commercially unreasonable.

**2. Section 1's Definition Is Overbroad.**

By defining "XanePay" as *"all infrastructure, systems, and financial orchestration technology,"* the Agreement captures the Developer's scope over the entire platform — not merely the engine component. Every obligation the Developer has "for XanePay" is therefore an obligation over the complete system, not merely the conversion engine.

**3. Title Contradiction.**

Section 2 assigns the role of "Technical Lead," but the signature block titles the Developer as "Backend Developer." These are different roles with different expectations. "Backend Developer" implies general backend work. "Technical Lead" implies leadership, architecture ownership, and coordination authority. Neither accurately describes the agreed-upon engagement (engine development). This inconsistency will create ambiguity in any future dispute about the Developer's scope of responsibility.

**Advisory:** Section 2 must be narrowed to: *"The Developer will act as Engine Developer for XanePay, responsible for the design, development, and delivery of the fiat-to-crypto/crypto-to-fiat orchestration engine as defined in the Technical Milestone Checklist."* The signature block title should read "Engine Developer." Section 1's definition should not expand the Developer's obligations beyond the engine scope.

---

## B. INTELLECTUAL PROPERTY AND CODE OWNERSHIP — ELIMINATION OF DEVELOPER LEVERAGE

### Contract Language (Section 6):

> *"All code, configurations, scripts, and work product created under this Agreement are the exclusive property of the Company **from the moment of creation**."*
>
> *"All development must occur in a **Company-controlled GitHub repository**; the Company must have **admin access at all times**."*
>
> *"The Developer must commit and push work to the repository at minimum **every 3 business days** of active development."*
>
> *"The Developer **shall not use personal or third-party repositories** for any primary development work."*
>
> *"The Developer shall not **withhold, encrypt, restrict, delete, or delay access** to the codebase **at any time**."*
>
> *"Violation of any code access obligation is grounds for **immediate termination without pro-rated compensation** for the current milestone."*

### Analysis:

This section creates a **structural power imbalance** that eliminates the Developer's primary commercial safeguard:

**1. The Developer generates value continuously; the Company captures it immediately.** All work product belongs to the Company from the moment it is typed — not upon delivery, not upon approval, not upon payment. The Developer builds; the Company owns. This is irreversible.

**2. The Developer is denied any protective interest.** Personal repositories are prohibited. Withholding access for any reason — including non-payment — is prohibited. The Developer has no mechanism to protect their position if the Company breaches its own obligations (e.g., failing to pay a milestone).

**3. The Company receives real-time work product; the Developer receives arrears-based payment.** Commits to the Company's repository are required every 3 business days. Payment occurs only after milestone completion and Company approval — which under Section 5 is "final and binding" with no appeal. The Developer therefore delivers code continuously but receives compensation intermittently and conditionally.

**4. Violation triggers disproportionate penalty.** Any code access violation — including a late commit due to illness, connectivity failure, or travel — triggers immediate termination **without pro-rated compensation for the current milestone.** This is a **forfeiture clause** with no proportionality requirement.

**In combination, these provisions mean:** The Company can, at any point during the engagement:
- Possess the full codebase built to date (via mandated commits)
- Reject a milestone, with its decision being "final and binding" (Section 5)
- Terminate the Agreement, citing any code access irregularity (Section 6) or broadly-defined "material breach" (Section 13.2)
- Retain all code already committed to the repository
- Owe the Developer nothing for the current milestone (Section 6 forfeiture clause) and potentially demand a refund of prior payments (Section 13.5)

**Legal Doctrine — Unconscionability:** A clause is unconscionable where the inequality of bargaining power results in terms so unreasonable that no informed person would voluntarily agree to them. The combination of immediate IP transfer, no protective retention, arrears-based payment, unilateral milestone rejection, and punitive forfeiture for minor infractions — all within a below-market-rate engagement — reaches this threshold.

**Advisory:** The IP and repository provisions must be rebalanced. Either:
- **(a)** Milestone payments must be restructured to **pay in advance** before each development phase begins, recognising that the Company receives code continuously; or
- **(b)** The Developer must be permitted to maintain a parallel repository with IP transfer upon corresponding milestone payment; or
- **(c)** At minimum: (i) "final and binding" milestone rejection must be replaced with independent adjudication, (ii) the forfeiture clause must be removed, and (iii) pro-rated compensation must be guaranteed for all verifiable work regardless of termination type.

---

## C. COMPENSATION AND MILESTONE STRUCTURE — UNCONSCIONABLE RISK ALLOCATION

### Contract Language (Section 3):

> *₦200,000 upon project commencement*
> *₦300,000 upon completion and Company approval of base infrastructure milestone*
> *₦200,000 upon full product completion, successful launch, and post-launch stability*

### Analysis:

**1. Third Milestone — "Post-Launch Stability" Is an Illusory Condition.**

The phrase "post-launch stability" is not defined anywhere in the Agreement. There is no specification of:
- What metrics constitute "stability" (uptime percentage, error rate, response time?)
- What timeframe applies (1 week? 1 month? 6 months?)
- Whether instability caused by factors outside the Developer's control (hosting failures, PSP outages, provider downtime, user load spikes) exempts the Developer from this condition

In contract law, a condition that one party has sole and undefined discretion to declare satisfied or unsatisfied is an **illusory condition.** It renders the obligation to pay illusory as well — the Company need never pay the ₦200,000 by perpetually asserting that "stability" has not been achieved.

**2. ₦200,000 (28.6% of compensation) Is Held Hostage to an Open-Ended Condition.**

The Developer must complete all work, deploy the system, and achieve an undefined state of "stability" before receiving the final payment. Post-launch stability monitoring and support is a **separate engagement** (typically compensated on a monthly retainer basis), not a condition precedent to payment of development fees.

**3. Market Value Disparity.**

The deliverables enumerated in Section 5 — including backend infrastructure, authentication, double-entry ledger, full API suite, database schema, provider integration framework, transaction lifecycle, webhook handling, error recovery, full simulation, and complete handover documentation — would command a market rate of **$15,000–$35,000 USD** (₦22M–₦52M) from a mid-to-senior financial engineer. The total compensation of ₦700,000 represents approximately 1.3%–3.1% of fair market value.

While the Client has accepted a below-market rate as a professional arrangement, the Agreement's punitive provisions (personal asset liability, unlimited liability for broadly-defined "cause," refund demands that bypass dispute resolution) are structured as if full market-rate compensation were being paid. **The risk exposure is not commensurate with the compensation.**

**Advisory:** The third milestone must be tied to an objective, verifiable event (e.g., successful deployment to staging/production environment with passing automated test suite). "Post-launch stability" must be either: (a) defined with specific metrics, specific timeframes, and exclusions for third-party-caused instability, or (b) removed entirely and replaced with a separate post-launch support agreement.

---

## D. TECHNICAL MILESTONE CHECKLIST — DELIVERABLES OUTSIDE AGREED SCOPE

### Contract Language (Section 5):

> The following deliverables are listed as required:
> - *Functional backend infrastructure*
> - *User authentication system*
> - *Double-entry ledger implementation*
> - *API endpoints (user, balance, transactions)*
> - *Database schema and setup*
> - *Provider integration framework*
> - *Transaction lifecycle system*
> - *Webhook handling*
> - *Error handling and retry logic*
> - *Full transaction flow simulation*
> - *Technical documentation and deployment readiness*

Additionally:
> *"The Company's milestone review decision is **final and binding**."*

### Analysis:

**1. Foreign Deliverables.**

The following items were **not** part of the agreed engine scope and appear in the checklist without the Developer's consent:

- **"Functional backend infrastructure"** — The Developer was engaged to build the conversion engine, not the Company's entire backend. The Company represented that the backend was 95% complete.
- **"User authentication system"** — The Company's own project documentation (PRD v3.0, Section "What is Explicitly OUT of Scope for MVP") expressly lists *"User authentication/registration (provided by XaneApp)"* as out of scope. This deliverable directly contradicts the Company's own technical specifications.

These constitute **unilateral scope enlargement by contract drafting.** The Developer was solicited for engine work and is now being contractually bound to deliver the full backend platform.

**2. Ambiguous Deliverables.**

- **"API endpoints (user, balance, transactions)"** — Does "user" refer to the engine's transaction-related endpoints, or the broader platform's user management endpoints (/users, /profiles, /auth)? The distinction is material to the scope of work.
- **"Database schema and setup"** — Engine database tables (transactions, ledger entries, provider registry, transaction events), or the entire platform database (users, authentication, app settings)?

**3. "Final and Binding" Milestone Rejection — No Independent Recourse.**

Section 5 states the Company's milestone review decision is "final and binding." Section 3 provides that milestone assessment (for pro-rated pay on termination) is conducted by *"a third-party developer designated by the Company"* — not a mutually agreed assessor.

In combination: the Company can reject a milestone, and the Developer has no right of appeal, no independent assessment, and no recourse. The Company retains all code already committed, withholds payment, and the Developer's only option is to accept or abandon the engagement.

**Advisory:**
- Remove "functional backend infrastructure" and "user authentication system" from the checklist. Replace with engine-specific deliverables (onramp/offramp flows, provider adapters, webhook handlers, engine-internal ledger, engine API endpoints). An appendix listing exact endpoints and database tables should be attached.
- Replace "final and binding" with: *"If the Developer disputes a milestone rejection, the matter shall be referred to a third-party developer **mutually agreed** by both parties for independent assessment."*

---

## E. TERMINATION AND REFUND — INTERNAL CONTRADICTIONS AND DISPROPORTIONATE REMEDY

### Contract Language (Sections 3, 4, 9, 13):

**Pro-Rated Compensation — Direct Contradiction:**

- **Section 3** states: *"the Developer shall be compensated on a pro-rated basis for verifiable, committed work toward that milestone."*
- **Section 13.5** states: *"The Developer **waives the right to pro-rated compensation** in such cases."* (for termination under Section 13.2)

These provisions are **mutually contradictory.** Under the principle of **contra proferentem** — that ambiguity in a contract is construed against the drafting party — Section 3's guarantee should prevail. However, the Company may argue that Section 13.5 is a specific exception overriding the general rule, leaving the Developer exposed.

**Compounding Mechanism:**

Section 13.2 grants the Company broad immediate termination rights for "material breach" of Sections 6, 7, 9, or 10. Under Section 6, a failure to commit code within the prescribed 3-business-day window could be classified as a material breach. The Company could therefore:

1. Classify a late commit as material breach of Section 6
2. Terminate immediately under Section 13.2
3. Invoke Section 13.5's waiver of pro-rated compensation
4. Retain all code already committed to the Company repository
5. Demand a refund of prior milestone payments under Section 9

Every step in this chain is supported by explicit contract language.

**Abandonment — Section 9.4: 7 Business Days.**

Section 9.4 defines abandonment as 7 consecutive business days of unresponsiveness without prior written notice. This is approximately 9–10 calendar days. This threshold does not account for medical emergencies, bereavement, sustained power/internet outages, or other force majeure events. Upon abandonment:
- All unpaid milestones are cancelled
- Full refund demanded for undelivered work
- Developer forfeits all pro-rated compensation claims

Note that Section 4 separately references a 14-day delay threshold for abandonment. The coexistence of a 7-day threshold (Section 9.4) and a 14-day threshold (Section 4) creates further ambiguity.

**Refund Demands Bypass Dispute Resolution — Section 14:**

Section 14 establishes a dispute resolution process: 14-day negotiation → mediation → courts. However, the same section explicitly exempts refund demands: *"refund demands are immediately enforceable upon written notice."*

This exemption means the Company can demand money back without any intervening period for the Developer to respond, present evidence, or contest the demand. It defeats the purpose of having a dispute resolution mechanism by carving out the most financially consequential category of dispute.

**Advisory:**
- Resolve the Section 3 / Section 13.5 contradiction. Pro-rated compensation for verifiable work must apply in all termination scenarios. The waiver in Section 13.5 should be deleted. The only exception should be proven fraud (Developer received payment and performed demonstrably zero work).
- Extend the abandonment threshold in Section 9.4 to 14 business days, consistent with Section 4. Add explicit exemptions for documented medical emergencies, infrastructure failures, and Company-caused delays (failure to provide credentials, sandbox access, or specifications).
- Remove the dispute resolution exemption for refund demands. All disputes, including refund claims, should follow the Section 14 process.

---

## F. PERSONAL LIABILITY AND IDENTITY VERIFICATION — OPPRESSIVE ENFORCEMENT

### Contract Language:

**Section 9.5 (Refund Enforcement):**
> *"The Developer consents to **personal liability** for any such debt, enforceable against their **identity and assets** as verified under Section 15."*

**Section 15 (Identity Verification):**
> The Developer must provide: full legal name, current residential address, NIN (National Identification Number) or government-issued ID, active phone number, and email.
>
> *"This information is held strictly confidential by the Company and used solely for payment processing, legal compliance, and **enforcement of obligations** under this Agreement."*

### Analysis:

The Company collects verified personal identification data (including government-issued ID and residential address) and expressly links it to an **asset enforcement mechanism** for refund disputes.

The practical effect: If the Company unilaterally rejects a milestone (Section 5 — "final and binding"), then demands a refund under Section 9 (immediately enforceable — no negotiation period), and the Developer does not refund within 7 days, the Company claims the right to pursue the Developer's personal assets using the identity information collected at onboarding.

For a contract valued at ₦700,000 — already a fraction of market rate — **personal asset seizure provisions are grossly disproportionate.** Such clauses are typically reserved for commercial agreements in the millions, secured lending, or fiduciary positions. Applying them to a sub-₦1M services engagement where the Developer is already working at a substantial discount constitutes an **oppressive term** that exploits the bargaining imbalance between the parties.

**Advisory:** Personal liability under Section 9.5 must be limited to cases of:
- Proven fraud (Developer received payment and performed demonstrably zero work)
- Deliberate misappropriation of Company data, funds, or credentials

Standard milestone disputes, delivery timing disagreements, quality objections, or scope disagreements must not trigger personal asset enforcement. Identity verification under Section 15 is acceptable for payment processing and KYC purposes but must be **decoupled** from the enforcement language in Section 9.5.

---

## G. MISCELLANEOUS PROVISIONS

### G.1: Scope Expansion Without Compensation (Section 2)

> *"Scope expansions do not automatically entitle the Developer to additional compensation unless separately agreed in writing."*

This clause places the burden on the Developer to negotiate compensation for every scope addition, while the Company can request additional work freely. Given the Developer is already working at a significant discount, this is commercially unreasonable.

**Advisory:** Amend to: *"Any scope expansion requires a written agreement on additional compensation before additional work commences. No work outside the defined scope will be performed without a signed amendment."*

### G.2: Pre-Existing IP (Section 6)

> *"Undisclosed pre-existing IP incorporated into deliverables is deemed assigned to the Company."*

This must explicitly exclude third-party open-source libraries and frameworks governed by their own public licences (MIT, Apache, ISC, etc.). Such software is publicly licensed and cannot be "assigned" to a private entity.

### G.3: Deliverables — "Production-Ready Backend Infrastructure" (Section 8)

Section 8 states the Developer will deliver *"Production-ready backend infrastructure."* The agreed deliverable is a production-ready orchestration engine — a component within the existing backend, not the backend itself.

**Advisory:** Change to: *"Production-ready fiat-to-crypto/crypto-to-fiat orchestration engine."*

### G.4: Confidentiality Breach — Full Refund of ALL Payments (Section 10)

> *"Any breach of confidentiality is grounds for immediate termination **and full refund of all payments made**, in addition to all other remedies."*

This means even if the Developer has completed two milestones, delivered verified working code, and been paid for that work — a single confidentiality breach obligates the Developer to refund **everything**, including pay for completed and accepted work. This is a **penalty clause** (refund of fully-earned milestone payments that bear no relationship to the confidentiality breach) rather than a genuine pre-estimate of loss.

**Advisory:** Confidentiality remedies should include termination and damages for the actual harm caused — not automatic forfeiture of all previously earned compensation for completed work.

### G.5: Replacement Readiness as Milestone Gate (Section 11)

Replacement readiness is evaluated at every milestone. Failure means the milestone is not complete *"regardless of other technical functionality."* Combined with the Company's "final and binding" rejection power, this creates an additional subjective rejection mechanism. Code can be technically perfect and fully functional, but the milestone is rejected because documentation is deemed "insufficient" — with no independent appeal.

### G.6: Credential Transfer Without Timeframe (Section 7)

> *"Failure to transfer credentials **upon request** is a material breach."*

"Upon request" has no defined compliance window. This should specify 48–72 business hours.

### G.7: Survival Clauses — Indefinite (Section 13.6)

Sections 6, 9, 10, 12, and 15 survive *"indefinitely."* Confidentiality and IP clauses surviving indefinitely is standard. However, refund demands (Section 9) and liability claims (Section 12) surviving indefinitely is excessive. Industry standard is 12–24 months post-termination.

### G.8: Timeline Extension Without Compensation (Section 4)

> *"Timeline extensions must be agreed in writing and **do not automatically increase compensation**."*

If the timeline extends due to Company-caused delays (late credentials, changed requirements, delayed feedback), the Developer absorbs the additional time at no extra cost. This is unfair. Timeline extensions caused by the Company should either increase compensation or extend the milestone payment schedule proportionally.

---

## H. SUMMARY OF REQUIRED AMENDMENTS

| # | Section | Issue | Required Amendment |
|---|---|---|---|
| 1 | §1 | Definition of "XanePay" is overbroad — captures entire platform | Narrow scope to the orchestration engine |
| 2 | §2 | Role "Technical Lead" with backend/architecture/coordination exceeds agreed scope | Change to "Engine Developer" — remove backend, architecture, coordination |
| 3 | §2 | Scope expansion clause places burden on Developer | Require written compensation agreement BEFORE any additional work |
| 4 | Sig. | Title "Backend Developer" entrenches wrong scope | Change to "Engine Developer" |
| 5 | §3 | Third milestone tied to undefined "post-launch stability" | Define with metrics and timeframe, or tie to staging deployment |
| 6 | §3 vs §13.5 | Direct contradiction on pro-rated compensation | Section 3 must govern; delete §13.5 waiver |
| 7 | §5 | Wrong deliverables: "backend infrastructure," "user authentication" | Replace with engine-specific deliverables; attach detailed appendix |
| 8 | §5 | Milestone rejection "final and binding," no independent appeal | Replace with mutually agreed third-party assessment |
| 9 | §6 | Immediate IP transfer + no escrow + arrears payment = zero Developer leverage | Restructure: advance payments, or parallel repo, or guaranteed pro-rated compensation |
| 10 | §6 | Code access violation = immediate termination without pro-rated pay | Remove forfeiture; pro-rated pay must always apply for completed work |
| 11 | §6 | Undisclosed pre-existing IP auto-assigned | Exclude open-source libraries |
| 12 | §7 | Credential transfer "upon request" — no compliance window | Add 48–72 hour window |
| 13 | §8 | "Production-ready backend infrastructure" exceeds scope | Change to "production-ready orchestration engine" |
| 14 | §9.4 | Abandonment at 7 business days — unreasonably tight | Extend to 14 business days; add medical, force majeure, Company-delay exemptions |
| 15 | §9.5 | Personal asset liability for ₦700K contract | Limit to proven fraud/theft only; decouple from identity verification |
| 16 | §10 | Confidentiality breach triggers refund of ALL payments including completed work | Limit to actual damages, not forfeiture of earned compensation |
| 17 | §14 | Refund demands bypass dispute resolution | Subject all disputes to 14-day negotiation process |
| 18 | §13.6 | Refund (§9) and liability (§12) survive termination indefinitely | Add 12-month limitation period post-termination |
| 19 | §4 | Timeline extensions "do not automatically increase compensation" | Company-caused extensions must adjust compensation or timeline proportionally |

---

## CONCLUSION

This Agreement, in its present form, **is not suitable for execution.** It assigns the Developer the scope and accountability of a Technical Lead — inclusive of backend infrastructure, system architecture, and team coordination — while compensating at a fixed flat fee of ₦700,000, a fraction of the market value for the deliverables demanded.

The Agreement further eliminates the Developer's commercial safeguards by vesting immediate IP ownership in the Company, prohibiting any protective code retention, granting the Company unilateral milestone rejection with no independent appeal, establishing contradictory termination provisions that enable the Company to retain all work product while denying the Developer compensation, and attaching personal asset enforcement provisions that are wholly disproportionate to the value of the engagement.

**I strongly advise the Client to decline execution** until the amendments set out in Section H above have been incorporated into a revised Agreement. If the Company declines to amend the material provisions — particularly Sections 2, 3, 5, 6, 9.5, and 13.5 — I would advise the Client to withdraw from the engagement entirely.

I am available to review any revised draft the Company produces.

---

*This memorandum is provided for the private use of the addressee and does not constitute legal advice to any third party.*

---

**[Counsel Name]**
Legal Adviser
[Date]
