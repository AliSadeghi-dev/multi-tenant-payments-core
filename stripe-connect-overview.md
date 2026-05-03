# Stripe Connect — Product & Architecture Overview

**Audience:** stakeholders reviewing whether Stripe Connect fits our payment platform.  
**Goal:** a single, factual document so others can understand the feature, tradeoffs, and what we would need to build—then give informed feedback.

This document is **not** implementation guidance for our codebase; it describes **Stripe’s product** and how it maps to common platform models.

---

## 1. What problem does Connect solve?

### Without Connect (single Stripe account)

- All charges settle into **one** Stripe account (typically the platform company’s account).
- You separate tenants in **your own database** (e.g. `projectId`, metadata on Checkout Sessions).
- **Good fit** when: every “project” is really **your** product line, or all money is **yours** and you settle with partners offline/contractually.

### With Connect

- **Multiple real businesses** can each have (or receive payouts to) **their own** Stripe account, while your platform software still creates charges, subscriptions, etc. **on their behalf** (subject to Connect type and compliance).
- **Good fit** when: marketplace, multi-vendor payouts, “money should land with the seller, not the platform,” or regulatory/KYC must sit with each seller.

**Connect does not replace** tenant isolation in your app. It answers a **different** question: *who is the legal/payout recipient on Stripe’s side.*

---

## 2. Core concepts (vocabulary)

| Term | Meaning |
|------|--------|
| **Platform** | Your company’s Stripe account + your software (the “hub”). |
| **Connected account** | A separate Stripe account linked to your platform via Connect (the “spoke”). Identified by an account id (e.g. `acct_…`). |
| **Direct charge** | Charge created on the connected account; funds flow per Connect configuration. |
| **Destination charge / separate charges & transfers** | Patterns where the platform charges and then moves funds (details depend on integration and account type). |
| **Application fee** | Optional platform fee taken on a charge (business decision, not mandatory). |

You generally **do not** store sellers’ Stripe **secret API keys**. You store the **connected account id** after onboarding, and use the **platform secret key** + Connect APIs / headers to act on behalf of that account (per Stripe’s model for your chosen account type).

---

## 3. How do external parties “connect” to us?

Typical flow (high level):

1. In your **dashboard or onboarding UI**, the seller clicks **“Connect payments”** (wording varies).
2. Your backend asks Stripe for an **Account Link** or uses **OAuth** (depends on Connect type and product choice).
3. The seller completes **Stripe-hosted onboarding** (identity, bank details, agreements). Stripe—not your UI—collects most sensitive KYC data.
4. Stripe notifies your backend (often via **webhook**) that the account is ready or needs more information.
5. You persist **`connected_account_id`** (and onboarding state) on your `Project` / `Seller` entity.

After that, payment APIs include the **connected account context** so Stripe knows which account the money event belongs to.

**Important:** “Connect to us” means **Connect onboarding to your platform as a Stripe Connect platform**, not “paste a secret key into a text box.” Secret keys remain **server-side secrets** for the relevant Stripe account(s).

---

## 4. Connect account types (short comparison)

Stripe supports **Standard**, **Express**, and **Custom** connected accounts. They differ in:

- Who owns the **Stripe Dashboard** experience for the seller  
- **KYC / compliance** burden split between Stripe, platform, and seller  
- **Flexibility vs operational load**

**Product/legal should choose** the type; engineering implements the chosen path. Switching later can be painful—decide early with counsel if payouts/KYC are non-trivial.

---

## 5. Benefits (why teams adopt Connect)

- **Correct payout routing:** funds can settle with the **seller’s** business, not always the platform’s balance.
- **Marketplace economics:** optional **application fees** or revenue splits (if that’s the business model—not required).
- **Stripe-hosted onboarding:** reduces what your UI must collect for identity/banking (compared to DIY).
- **Clear separation** for tax/KYC when each seller is a real merchant.

---

## 6. Costs and responsibilities (often underestimated)

- **Operational:** onboarding states (“restricted”, “pending”, “rejected”), document requests, support when sellers fail KYC.
- **Dashboard / UX:** sellers may need a **seller-facing** area (even minimal) for status, bank updates, tax forms—depending on account type.
- **Webhooks:** you may need to listen to events **across** connected accounts; routing and idempotency become more complex than a single-account webhook inbox.
- **Compliance:** platform responsibilities vary by Connect type and region; **legal review** is advised before promising a model to customers.

---

## 7. UI & dashboards — what do we actually need?

There is **no single answer**; it depends on Connect type and how much you delegate to Stripe.

| Layer | Typical contents |
|-------|------------------|
| **Platform admin** | List connected accounts, onboarding status, risk flags, support actions, fee configuration (if any). |
| **Seller / merchant UI** (may be minimal for Express) | “Start/continue onboarding”, payout status, maybe links out to Stripe-hosted components. |
| **End-customer checkout** | Usually still **your** or Stripe’s hosted checkout; branding and who appears on the bank statement depend on integration choices. |

**Multiple dashboards** are common in marketplace products (platform ops vs seller portal). For **internal-only multi-project** billing (all money is yours), you often need **only** your platform admin—**Connect adds seller-facing surfaces** when real third-party merchants exist.

---

## 8. What Connect does *not* magically solve

- **Your** multi-tenant data model (`Project`, `Product`, `Payment`, etc.) is still yours to design and scope.
- **Disputes, refunds, and support workflows** still need product decisions.
- **“Can we white-label and resell this payment service?”** can be solved **without** Connect by **per-deployment** Stripe keys (each customer runs their own instance/secrets). Connect is for **many merchants on one deployment**, not the only B2B distribution model.

---

## 9. Decision matrix (for stakeholder discussion)

| If our situation is… | Likely direction |
|----------------------|------------------|
| Several **internal** apps; all revenue is **our** company’s | **Single Stripe account** + strong tenant isolation in our DB. **No Connect required** to start. |
| We operate a **marketplace**; sellers must receive payouts as **their** business | **Connect** (choose account type with legal). |
| We sell **hosted software** to other companies; each wants **their own** Stripe | Often **separate deployment or env per customer** with their Stripe keys; Connect only if you truly need **one SaaS deployment** onboarding **many unrelated Stripe businesses**. |

---

## 10. Open questions (please answer / debate)

Use this as a checklist for the review meeting:

1. **Money flow:** Should settlement always hit **our** Stripe balance, or **sellers’** balances?  
2. **Legal entity:** Who is the merchant of record for receipts/invoices?  
3. **Fees:** Do we charge a **platform fee** per transaction, a **SaaS subscription only**, or both?  
4. **Regions & methods:** Which countries and payment methods must we support in v1?  
5. **Account type:** Standard vs Express vs Custom—who owns KYC/support burden?  
6. **Webhooks:** Single endpoint with account attribution—do we have the engineering capacity for the extra edge cases?  
7. **Roadmap:** Can we **ship MVP without Connect** and add Connect only when a signed customer requires marketplace payouts?

---

## 11. Recommendation for *our* stated original goal

If the primary goal is: **one internal payment service shared by multiple projects we own**, with **one Stripe account**, then **Stripe Connect is optional and adds complexity**.

Treat Connect as a **phase-2 / separate product track** unless we have a concrete marketplace or multi-merchant payout requirement.

---

## 12. References (official)

- Stripe docs: **Stripe Connect** (overview, account types, onboarding, charges).  
- Search within Stripe’s site for: *Connect account types*, *Connect onboarding*, *Application fees*, *Webhooks Connect*.

*(Links intentionally omitted here to avoid stale URLs; use the current Stripe documentation index.)*

---

## Feedback requested

Please reply with:

- **Yes / no / later** on adopting Connect for v1, and why.  
- Which **account type** you assume if “yes.”  
- Any **must-have** country or payout scenario we did not mention.

---

*Document version: 1.0 — internal planning draft.*
