# League Tournament Dues — Stripe Connect, surcharge model, 1% platform fee

**Status:** RFC — design only, no code touched yet
**Date:** 2026-05-16
**Replaces:** "Informational only — payment collection coming soon" stub at `app/league/[id]/manage/page.tsx:3042`
**Driver:** 2026-05-16 audit ranked this #1 next value-add. Stableford-based first paying customer is already in-market weekly; trial cliff at 2026-06-30. Founder direction: owner covers Stripe fees, 1% of every transaction to GolfSync.

---

## 1 · Money flow at a glance

```
Player pays $51.84 ──▶ Stripe ──┬─▶ $0.50  → GolfSync (1% platform fee)
                                ├─▶ $1.84  → Stripe (2.9% + $0.30)
                                └─▶ $49.50 → League owner's connected account

(Player owed $50 in dues; surcharge line "$1.84 covers processing" added at checkout)
```

The numbers above use the standard Stripe domestic card pricing. International, AMEX, and disputed charges all change the fee structure — see §6.

---

## 2 · The Apple App Store + Google Play nuance (CRITICAL)

This is the single most important constraint and must be wired correctly from day 1 or the iOS app gets rejected.

### What's allowed (no IAP, no 30% cut)

**Real-world peer-to-peer payments between humans for real-world activity.** Apple Guideline 3.1.5(a) and Google's equivalent policy explicitly allow:
- Cash-app-style transfers (Venmo, Zelle)
- Event ticketing (Eventbrite-style)
- Crowdfunding (GoFundMe-style)
- Service marketplaces (TaskRabbit, Uber Eats)
- **League dues, tournament entry fees, hosted-tournament registration** — these are real-world cash that happens to flow through the app

Stripe is the explicitly-allowed processor for all of the above. Apple does NOT take a cut.

We already use Stripe Elements + Apple Pay / Google Pay on the hosted-tournament public registration page (Phase 4, shipped) under this exact rule. Same rule applies to league dues.

### What's NOT allowed (must use IAP, Apple takes 30%)

**Digital content unlocks, subscriptions to access app features.** Apple Guideline 3.1.1:
- In-app purchase of premium features
- Unlocking app capabilities (e.g. "$2.99 to enable advanced stats")
- Subscriptions to GolfSync itself (membership Pro tier)

This is why our $19/mo Organizer subscription has an IAP path on iOS (`com.golfsync.mobile.organizer.monthly`) and the Phase 4 Apple/Google Pay buttons are reserved for **event payments** only.

### What this means for dues

Dues = real-world peer-to-peer payment = **Stripe path, no Apple IAP**.

But mobile UI matters. The dues-payment surface must:
- Never describe the payment as "unlocking" anything
- Never bundle dues with a subscription upsell
- Frame the payment as "pay your league dues" (real-world activity), not "pay GolfSync for ..."
- The 1% platform fee MUST be invisible to Apple's reviewer — they'd see it as us collecting a digital-services fee for app usage, which crosses into IAP territory

**Resolution:** The 1% platform fee uses Stripe Connect's `application_fee_amount` server-side. Player NEVER sees a "GolfSync platform fee" line item. The player sees only dues + processing surcharge. The 1% comes out of the league owner's share (technically the owner sees "$0.50 GolfSync fee" in their Stripe payout breakdown, but the *player checkout* is clean).

This is the standard Stripe Connect marketplace model and is App-Store-compliant. Patreon, Substack, Eventbrite all operate this way.

### Risk

If we ever ship a "GolfSync convenience fee" line that the **player** sees on iOS, Apple Review can reject the app on guideline 3.1.5(a) ambiguity. The split must stay server-side.

---

## 3 · Stripe Connect onboarding (one-time per league owner)

To collect payments on behalf of a league owner, they connect their Stripe account to ours. Two flavors:

### Option A — Stripe Connect Express (Recommended)

- Owner clicks "Set up payments" in league settings
- Redirected to a Stripe-hosted onboarding form (5 minutes)
- Stripe collects: legal name, DOB, last 4 SSN (US), bank account for payouts, tax ID for 1099-K
- Stripe handles KYC, fraud screening, tax reporting
- Stripe-branded dashboard for the owner to view their payouts
- GolfSync gets a `stripe_account_id` (`acct_xxx`) we store and use as the destination for `application_fee_amount` charges

**Pros:** ~5min to onboard, Stripe handles all compliance, owner gets their own payout dashboard, we have no money-services-business exposure.

**Cons:** Owner sees a Stripe-branded UI mid-flow (minor branding break). Stripe Connect fee structure adds 0.25% + $2/month per active connected account.

### Option B — Stripe Connect Standard

- Owner connects their own pre-existing Stripe account (must already have one)
- Less hand-holding, fewer fees, but most league commissioners don't have a Stripe account

We use **Express** — universal-onboarding is non-negotiable for organic adoption.

### Schema needed

```sql
ALTER TABLE leagues
  ADD COLUMN stripe_account_id        VARCHAR(64) NULL,
  ADD COLUMN stripe_account_status    VARCHAR(20) NULL,   -- pending / active / restricted / disabled
  ADD COLUMN stripe_account_connected_at TIMESTAMP NULL,
  ADD COLUMN stripe_charges_enabled   BOOLEAN NOT NULL DEFAULT FALSE,
  ADD COLUMN stripe_payouts_enabled   BOOLEAN NOT NULL DEFAULT FALSE;
```

We listen for `account.updated` webhooks to keep `*_enabled` flags fresh — Stripe can disable a connected account at any time for KYC reasons and we need to surface that to the owner.

---

## 4 · Per-registrant dues + payment flow

Schema today (`LeagueTournamentRegistration`) already has the columns we need from prior hosted-tournament work:

| Column | Use |
|---|---|
| `amount_cents` | What the player owes (defaults from `tournament.duesAmountCents`) |
| `payment_status` | `pending_payment` / `paid` / `comped` / `refunded` |
| `payment_method` | `card` / `cash` / `venmo` / `check` / `comp` |
| `payment_reference` | Free-text reference for non-card payments |
| `paid_at` / `paid_by_user_id` | Audit |

**No schema migration needed** for the per-registrant data — we reuse existing columns.

The **league** needs the Stripe Connect columns (§3) and one new column:

```sql
ALTER TABLE league_tournaments
  ADD COLUMN dues_collection_mode VARCHAR(16) NOT NULL DEFAULT 'INFORMATIONAL';
  -- INFORMATIONAL (today's default — display amount, no collection)
  -- COLLECT_STRIPE (player pays via card on tournament page; manual mark-paid still available as a fallback)
  -- COLLECT_MANUAL (no Stripe; owner marks paid as cash/Venmo/check flows in)
```

**Offline mark-paid is a first-class capability in BOTH non-INFORMATIONAL modes.** Owners always have a "Mark paid (offline)" path on every registrant row regardless of whether the tournament is set up for Stripe collection. This covers the universal case: player handed the owner cash at the course, Venmoed them directly, mailed a check, etc. Reusing the existing `PaymentIntakeDialog` shipped in P0-4 (cash / Venmo / check / wire / card-external / other picker) so the UX is consistent with the hosted-tournament admin.

When `dues_collection_mode = COLLECT_STRIPE`, the registration form on the tournament page **also** wires Stripe Elements with `application_fee_amount` for the 1% + the surcharge math. Stripe is the player's self-service path; offline mark-paid is the owner's backup.

### Payment flow (COLLECT_STRIPE)

1. Player visits `/league/[id]/tournament/[tid]` and clicks Register
2. If dues > 0 and mode is `COLLECT_STRIPE`, the registration POST creates a Stripe PaymentIntent with:
   - `amount`: dues + surcharge (player-visible total)
   - `application_fee_amount`: 1% of dues
   - `transfer_data.destination`: league's `stripe_account_id`
   - `on_behalf_of`: league's `stripe_account_id`
3. Frontend mounts `<PaymentElement>` (reuses the hosted-tournament pattern)
4. On `payment_intent.succeeded` webhook: server flips registration's `payment_status` to `paid`, stamps `paid_at`, records the Stripe charge id in `payment_reference`
5. Confirmation email goes out

### Payment flow (COLLECT_MANUAL) — also the offline-mark-paid path

No Stripe involvement. Owner marks rows paid via the same `PaymentIntakeDialog` that hosted-tournament admin uses (picker: cash / Venmo / check / wire / card-external / other + optional notes). The payment-method string + notes get composed into `payment_reference` so audit + reconciliation remain searchable.

**This path is also available as a fallback inside COLLECT_STRIPE tournaments.** Real-world example: player insists on paying cash at the course on tournament morning, doesn't want to fill out the Stripe form. Owner opens that registrant row → "Mark paid (offline)" → records cash + optional note. Stripe is never invoked, no platform fee, no processing surcharge — just a status flip with provenance. The 1% platform fee is only collected on transactions that actually flow through Stripe.

### Payment flow (INFORMATIONAL)

Today's behavior. Dues display only, no collection. Owners who keep this mode get the offline-mark-paid path too — INFORMATIONAL just means "we don't add a Pay Now button on the public registration form."

### Hosted tournaments — parity

The hosted-tournament admin already has the offline-mark-paid path (shipped 2026-05-16 P0-4 via `PaymentIntakeDialog`). League dues v1 reuses the same component + the same payment-method enum so Becky's mental model and a league commissioner's mental model are identical.

---

## 5 · UI surfaces

### League owner — `/league/[id]/manage` Tournaments tab

- Replace "Informational only — payment collection coming soon" with a real picker per tournament:
  - **Display amount only** (default — preserves today's behavior)
  - **Track manually** (mark-paid + waive + per-registrant amounts)
  - **Collect via Stripe** (requires connected account; greyed-out CTA if not connected)
- Each registrant row gains: Amount owed · Status badge · [Mark Paid] / [Waive] / [Refund] / [Send Reminder] buttons
- "Connect Stripe" CTA in league settings tab if not yet connected

### League owner — Settings tab (new "Payments" section)

- Big "Connect Stripe" button when `stripe_account_id` is null
- Status panel when connected: balance · last payout · disabled-account banner if KYC issue
- Link to Stripe Express dashboard

### Player — `/league/[id]/tournament/[tid]` registration form

- When `COLLECT_STRIPE` + dues > 0: Stripe Elements card form + Apple/Google Pay buttons (reuses Phase 4 hosted-tournament pattern)
- Surcharge line: "$50.00 dues · $1.84 covers processing · $51.84 total"
- "Why am I being charged this fee?" tooltip linking to a short help-page entry

### Player — `/account` page

- Payment history (your tournament payments, with download-receipt link)

---

## 6 · Edge cases worth pre-deciding

1. **AMEX charges** (3.5% + $0.30): surcharge becomes $2.05 instead of $1.84. Stripe Connect calculates this automatically — we ask for the *fee-inclusive* amount, Stripe returns the actual fee, and our `application_fee_amount` math always uses the same 1% of dues (NOT 1% of fee-inclusive total — that would compound)
2. **Refunds**: when admin refunds a paid registration, Stripe refunds the player's full amount, and the league owner's account is debited; the 1% platform fee STAYS WITH GOLFSYNC unless we explicitly refund it (`refund_application_fee` flag). Standard practice: keep the platform fee on refunds for fraud-discouragement
3. **Disputed charges**: Stripe charges $15 dispute fee. By default falls on the connected account (league owner). Surface this in onboarding terms
4. **Mobile in-browser Stripe**: works on mobile Safari/Chrome the same as desktop. Apple Pay button auto-shows on iOS Safari, Google Pay on Android Chrome. Same as Phase 4
5. **Connect account disabled mid-tournament**: webhook updates `stripe_charges_enabled = false`. New registrations fall back to `COLLECT_MANUAL` (waiver/cash) until owner resolves. Existing paid registrations are unaffected (money already settled)
6. **Tax handling**: GolfSync issues 1099-K via Stripe Connect Express automatically for owners crossing the $5K/year IRS threshold. No work for us. Owner sees the 1099 in their Stripe Express dashboard

---

## 7 · What ships in v1

1. Migration: `leagues` Stripe Connect columns + `league_tournaments.dues_collection_mode`
2. `StripeConnectService` (onboarding link generation, account.updated webhook handler)
3. `LeagueDuesController` (mark paid / waive / refund / set custom amount — manual path)
4. Stripe PaymentIntent creation with `application_fee_amount` + `transfer_data.destination` (Stripe path)
5. Webhook routing additions: `account.updated`, `payment_intent.succeeded` (already wired for hosted; extend the routing)
6. League settings tab → "Payments" section with Connect CTA + status
7. Manage page → Tournament card payment-mode picker + per-registrant payment UI
8. Tournament page → Stripe Elements when `COLLECT_STRIPE` is set + surcharge math

**Effort estimate (honest):** ~2-3 days end-to-end.

## 8 · What's explicitly NOT in v1

- Annual reporting / accountant exports (Stripe Express dashboard handles this)
- Multi-currency (US/CAD only at launch)
- Payment plans / installments
- Refund-platform-fee toggle (always keep the 1%)
- Mobile-native registration flow (web only for v1; mobile uses web view embedded in app — already the hosted-tournament pattern)

---

## 9 · Open questions before implementation

1. **Stripe Connect terms of service** — owners need to agree to GolfSync's connected-account terms during Express onboarding. Does Ryan have a written terms page or do we draft one? (Stripe supplies most of the language; we add platform-specific clauses)
2. **1% rate written in product** — surface this rate in the league owner's onboarding flow so it's transparent? Or just disclose in terms?
3. **First-customer dogfood** — Stableford commissioner is the obvious first user. Coordinate the launch so he tests it before broader rollout?
