# C2 Adopt 2026 — Founder runbook

This is the operational checklist for getting C2 Adopt 2026 live in
production. Companion to [`c2-adopt-federal-2026.md`](./c2-adopt-federal-2026.md) (the design
doc). Written 2026-05-15.

> **What this covers**
>
> Step-by-step tasks YOU (the founder) need to do outside of code:
> Stripe Dashboard config, Apple Pay domain verification, Google Pay /
> Web Payment Request setup, environment variables, customer info to
> collect from C2 Adopt, and the order in which everything has to
> happen before public launch.
>
> **What this does NOT cover**
>
> Internal architecture (see the design doc). Code-level details for
> the existing Phase 1-4 backend (see the api repo's
> `feature/hosted-tournaments-*` work and recent commits tagged
> `feat(hosted-tournaments)`).

---

## 0 · Status at time of writing (2026-05-15)

Shipped + live in prod:
- ✅ Hosted-tournament backend (read, intake, payment state machine, webhook routing, emails)
- ✅ Admin command center (5 tabs: Registrations / Sponsors / Reconcile / Prices / Branding / Setup)
- ✅ Federal Club course seeded (migration 142)
- ✅ Per-tournament branding (logo + colors + tagline, with admin Branding tab + live preview)
- ✅ C2 Adopt 2026 tournament row seeded in PREVIEW mode (migration 144)
- ✅ Hosted Tournaments shortcut tab on `/admin`

Still on the runway:
- ⏳ `<PaymentElement>` + Apple Pay / Google Pay buttons on the public registration form (backend ready, frontend pending)
- ⏳ C2 Adopt customer info collection (date, capacity, tier names/prices, refund policy, etc.)
- ⏳ Stripe Dashboard configuration (this doc)
- ⏳ Apple Pay domain verification (this doc)
- ⏳ Final flip from `is_publicly_listed=FALSE` to `TRUE` on launch day

---

## 1 · Customer info to collect from C2 Adopt (Becky's team)

Send Becky a Google Form / email with these questions. Until these
land, the placeholder values from migration 144 stay in the admin UI
and you can edit each via the Branding / Prices / Sponsors tabs:

1. **Tournament date** — exact day in September 2026
2. **Flight tee times** — what time does AM tee off? PM?
3. **Per-flight capacity** — 72/72 is the placeholder; can be different
4. **Registration prices** — confirm or revise:
   - Foursome: $550 (current placeholder)
   - Individual: $150
   - Dinner-only: $50
5. **Sponsor tier prices + benefit copy** — current placeholders:
   - Title: $10,000
   - Eagle: $5,000
   - Birdie: $2,500
   - Hole: $750
6. **Refund policy** — cancellation deadline? Partial refunds OK? No refunds after X?
7. **Stripe processing fee** — eat it or pass to registrant ("Add 3% to cover processing")?
8. **Mail-to address for checks** — exact mailing address C2 Adopt wants printed in confirmation emails
9. **Bank info for ACH/wire** — routing + account, or do they handle this manually?
10. **Existing invoicing tool** — QuickBooks? FreshBooks? GolfSync generates invoices, or just tracks status against external ones?
11. **Becky's email + any other staff** who need TOURNAMENT_ORGANIZER access
12. **Logo handoff** — we have the SVG from c2adopt.org pre-seeded; do they want a different version for the microsite hero? (Higher-res PNG, white-on-transparent variant for dark backgrounds, etc.)
13. **Mission statement / tagline** — current placeholder: "Uniting children with forever families. Join us at The Federal Club this September." OK or revise?

---

## 2 · Stripe Dashboard configuration (founder action)

Done once per Stripe account. Most of these may already be set from
the membership flow; double-check each before launching C2 Adopt.

### 2.1 Webhook endpoint

The hosted-tournament card flow relies on `payment_intent.succeeded`
webhooks. The existing membership endpoint already subscribes to
several events — make sure it includes:

- ✅ `payment_intent.succeeded` (NEW for hosted tournaments — confirm subscribed)
- `checkout.session.completed` (membership)
- `invoice.paid` (membership)
- `invoice.payment_failed` (membership)
- `customer.subscription.updated` (membership)
- `customer.subscription.deleted` (membership)
- `charge.refunded` (membership + future hosted-refund work)
- `charge.dispute.created` (membership)

**Where**: Stripe Dashboard → Developers → Webhooks → your prod endpoint
→ "Listen to events on your account" → ensure `payment_intent.succeeded`
is checked.

**Verify**: After saving, hit "Send test webhook" with a
`payment_intent.succeeded` event. Check the prod API log for a `200`
response with body `{"received":"true"}`. Look for the log line
`Stripe webhook received: type=payment_intent.succeeded id=evt_...`
in `/golfsync-prod/api`.

### 2.2 Test vs. Live mode keys

You'll already have these set from the membership rollout. Confirm:

- `STRIPE_SECRET_KEY` — starts with `sk_live_` in prod, `sk_test_` everywhere else
- `STRIPE_PUBLISHABLE_KEY` — `pk_live_` / `pk_test_`
- `STRIPE_WEBHOOK_SECRET` — `whsec_…` — MUST be set in prod or the API refuses to start (StripeConfig.init fail-fast as of 2026-05-09)

**Where**: AWS Secrets Manager → `golfsync-prod/stripe/*`. Update there, then trigger a rolling ECS deploy so tasks pick up the new values (`./scripts/deploy.sh prod`).

### 2.3 Apple Pay domain verification

Required for the Apple Pay button to appear in Safari / iOS on the
public registration form.

**Steps:**
1. Stripe Dashboard → Settings → Payment methods → Apple Pay → "Add a new domain"
2. Enter `golfsync.io` (and `staging.golfsync.io` if you want it on staging too)
3. Stripe gives you a verification file — `apple-developer-merchantid-domain-association`
4. Place that file at `golfsync-web/public/.well-known/apple-developer-merchantid-domain-association` (Next.js serves `/public/*` at root, so it'll resolve at `https://golfsync.io/.well-known/apple-developer-merchantid-domain-association`)
5. Click "Verify" in the Stripe Dashboard — it'll fetch the file and confirm

**Verify**: `curl https://golfsync.io/.well-known/apple-developer-merchantid-domain-association` should return the file contents (a long hex string). If 404, the file wasn't deployed — check Next.js public/ directory.

### 2.4 Google Pay / Web Payment Request

No domain verification needed for Google Pay — Stripe handles the
merchant identifier server-side. The Google Pay button appears
automatically on Chrome / Android via the Web Payment Request API
once Stripe.js is on the page.

**Verify** (after Stripe Elements is wired in — see §4):
1. Open the public registration form in Chrome on Android (or DevTools mobile emulation)
2. Select "Credit card" payment method
3. Confirm a Google Pay button renders above the regular card form

### 2.5 Stripe Connect (sponsor pass-through payouts — OPTIONAL)

Not needed for C2 Adopt v1. Their tournament uses a single merchant
account (yours) and they reconcile with you off-platform. If a future
customer wants direct deposits to THEIR bank, that requires Stripe
Connect — not in scope.

---

## 3 · Pre-launch verification checklist

Run through these before flipping `is_publicly_listed=TRUE`. Use the
admin command center's **Setup** tab — every item there is
data-driven so it tells you exactly what's missing.

| # | Check | Where to look |
|---|---|---|
| 1 | Tournament name + date set | Admin command center → Setup tab |
| 2 | Course location captured | Same (already set: The Federal Club, Glen Allen, VA) |
| 3 | At least one flight configured | Same (AM + PM both seeded) |
| 4 | Registration prices configured | Admin → Prices tab (3 types seeded; edit if needed) |
| 5 | Sponsor tiers configured | Admin → Sponsors tab (4 tiers seeded) |
| 6 | Branding applied | Admin → Branding tab (logo + colors pre-filled; check live preview) |
| 7 | Stripe `payment_intent.succeeded` webhook firing | Stripe Dashboard → recent events |
| 8 | Apple Pay domain verified | Stripe Dashboard → Apple Pay domains |
| 9 | Test registration end-to-end | Self-register with a test card on `/tournaments/c2-adopt-2026` (you're authorized to see it in PREVIEW mode) |
| 10 | Test sponsor application | Same; pick a tier + submit |
| 11 | Confirmation emails arriving | Check the test address inbox; check SES dashboard for delivery |
| 12 | Becky has TOURNAMENT_ORGANIZER access | Admin → Tournaments tab → Manage organizers → grant by email |

### Test card numbers (Stripe test mode only)

- `4242 4242 4242 4242` — successful charge
- `4000 0000 0000 9995` — insufficient funds, declined
- `4000 0027 6000 3184` — 3D Secure 2 authentication required

Any future date for expiry, any 3-digit CVC, any 5-digit ZIP.

---

## 4 · Outstanding code work before public launch

Two pieces of code are not yet shipped. Both are required before
C2 Adopt's card-paying registrants can actually pay end-to-end:

### 4.1 Stripe Elements + Apple Pay / Google Pay on the public form

**Status**: Backend fully ready. Public registration form has the
"credit card" payment method radio, and when you submit with `card`
selected the API creates a real Stripe PaymentIntent and returns a
`clientSecret`. But the form has nowhere to confirm the card with that
secret — there's no `<PaymentElement>` mounted, no Stripe.js loaded.

**What needs to happen**:
1. Load `loadStripe(publishableKey)` once on `/tournaments/[slug]/page.tsx`
2. When `paymentMethod === "card"` is selected, wrap the submit button
   in `<Elements stripe={stripePromise} options={{ clientSecret }}>`
   after the registration POST returns the secret
3. Mount `<PaymentElement />` inside — Stripe.js's "Payment Element"
   automatically surfaces Apple Pay on Safari/iOS + Google Pay on
   Chrome/Android via the Web Payment Request API
4. Call `stripe.confirmPayment({ elements, confirmParams: { return_url } })`
   on submit
5. On the return URL, show the confirmation panel (existing component)

**Reference implementation**: see `/app/membership/page.tsx` — same
pattern, different intent shape.

**Estimate**: ~2-3 hours.

### 4.2 Phase 6 customer-specific tweaks

Once C2 Adopt answers the questions in §1:
1. Edit the AM + PM tournament rows to set the real date + times via the league tournament page
2. Edit Sponsor tiers in the admin Sponsors tab (or run a follow-up migration)
3. Edit registration prices in the admin Prices tab
4. Edit refund policy copy in the description field
5. Grant Becky TOURNAMENT_ORGANIZER role (admin → Tournaments → Manage organizers → by email)

**Estimate**: ~30 min once info arrives.

---

## 5 · Launch-day procedure

1. Run through §3 checklist one final time
2. Hit `/admin/hosted-tournaments/c2-adopt-2026/manage` → Setup tab → confirm all checks green
3. From the league tournament admin page, flip `is_publicly_listed` from FALSE to TRUE on **both** AM and PM rows
4. Verify `https://golfsync.io/tournaments/c2-adopt-2026` loads for an anonymous (logged-out) browser
5. Post the URL to C2 Adopt's email list / website / social channels
6. Monitor `/admin/hosted-tournaments/c2-adopt-2026/manage` registrations tab through the day for inbound

---

## 6 · Day-of operations

The OneCause differentiator. C2 Adopt's main pain with their previous
platform was the setup-to-day-of workflow. Your moat:

- **Bulk mark-paid**: Admin command center → Reconcile tab. Tick N pending rows, enter a deposit reference ("Truist deposit 9/14"), one click marks them all paid.
- **Live registration table**: Registrations tab refreshes via the Refresh button; counts in the top bar (Paid / Pending / Outstanding $) recompute.
- **Sponsor signage list**: Sponsors tab; export by copy-paste for the printer.
- **Setup checklist**: Data-driven, not manually ticked — admins can't accidentally check off something that isn't actually configured.

---

## 7 · Post-event cleanup

1. Export registrations CSV (TODO: not built yet — copy-paste the table for now, or hit the admin API directly with `curl`)
2. Reconcile final payments (Reconcile tab)
3. Flip `is_publicly_listed=FALSE` if you want the URL to stop responding to public traffic
4. Archive the league or leave it as-is for next year (re-using the same slug means next year's tournament can edit in place)

---

## 8 · Where everything lives

- **Public microsite**: `https://golfsync.io/tournaments/c2-adopt-2026`
- **Admin command center**: `https://golfsync.io/admin/hosted-tournaments/c2-adopt-2026/manage`
- **Admin Hosted Tournaments list**: `https://golfsync.io/admin` → "Hosted Tournaments" tab
- **League management** (date/time/capacity edits): `https://golfsync.io/league/{league_id}/manage` → Tournaments tab → click the C2 Adopt row
- **Stripe Dashboard**: dashboard.stripe.com → Payments / Webhooks / Settings
- **Prod API logs**: `aws --profile golfsync-prod logs tail /golfsync-prod/api`
- **Migrations**: `golfsync-api/src/main/resources/db/changelog/changes/14[2-4]-*.sql`
- **Brand assets**: logo SVG at `https://c2adopt.org/wp-content/uploads/2023/12/logo.svg` (referenced directly by the seed); colors `#006E9E` (primary) + `#FCB525` (accent)
