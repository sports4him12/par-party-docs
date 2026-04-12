# BEFORE MEMBERSHIPS ARE ENFORCED

Items that must be completed before the membership enforcement date is announced and goes live.
Work through each section top to bottom before deploying the enforcement date to production.

---

## 1. Finalize the Enforcement Date

- [ ] Decide the official date memberships are required (billing start date)
- [ ] Update the hardcoded "free through May 31, 2026" copy in `golfsync-web/app/membership/page.tsx` (lines 189–202) to reflect the real date
- [ ] Update `golfsync-docs/prod-go-live-checklist.md` billing start date (line 3) to match

---

## 2. Re-enable Paused Drip Emails

Three emails in [TrialDripService.java](../golfsync-api/src/main/java/com/golfsync/service/TrialDripService.java) were commented out because the enforcement date was not finalized. Uncomment all three before go-live.

- [ ] **Day 0 — Trial Welcome** (lines 78–83): fired the morning after signup via the 9 AM cron. Currently this is the only welcome email — with it paused, new users receive **no email at all** until Day 3.
- [ ] **Day 21 — Trial Ending Soon** (lines 94–99): fires when 9 days remain in the trial
- [ ] **Day 27 — Trial Last Chance** (lines 100–108): final personalized nudge with 3 days left

> **Note on Day 0:** Because it fires via the daily cron (not immediately on registration), new users will not receive the welcome email until the morning after they sign up. This is the existing behavior and is expected.

---

## 3. Stripe — Create Products and Prices

These do not exist yet. The backend throws `IllegalStateException` at checkout if they are not configured.

- [ ] In the [Stripe Dashboard](https://dashboard.stripe.com/products), create a **Monthly Membership** recurring price at $9.99/month
- [ ] Create an **Annual Membership** recurring price at $79.99/year
- [ ] Copy both `price_...` IDs — they are needed in the next section

---

## 4. Stripe — Configure Keys and Secrets

All three Stripe secrets are currently set to placeholder strings in Secrets Manager. The backend logs a warning and disables payments if any are missing.

- [ ] Rotate/set `STRIPE_SECRET_KEY` in Secrets Manager to the production `sk_live_...` key
- [ ] Set `STRIPE_PUBLISHABLE_KEY` in Secrets Manager to the production `pk_live_...` key
- [ ] Set `STRIPE_MONTHLY_PRICE_ID` env var to the monthly `price_...` ID from step 3
- [ ] Set `STRIPE_ANNUAL_PRICE_ID` env var to the annual `price_...` ID from step 3
- [ ] Add both price ID vars to `.env.prod.example` (test values are fine there)

---

## 5. Stripe — Register the Webhook Endpoint

The webhook code is ready but the endpoint must be manually registered in the Stripe Dashboard.

- [ ] In the Stripe Dashboard → Developers → Webhooks, add endpoint: `https://golfsync.io/api/stripe/webhook`
- [ ] Select these events: `checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`
- [ ] Copy the `whsec_...` signing secret and set `STRIPE_WEBHOOK_SECRET` in Secrets Manager
- [ ] Fix the wrong path in `golfsync-docs/admin-runbook.md` line 30 — it currently says `/api/payments/webhook` but the correct path is `/api/stripe/webhook`

---

## 6. Trial Period Alignment

Every user gets a 30-day trial on registration. Users who signed up more than 30 days before the enforcement date will already have expired trials when billing goes live — they will be locked out immediately on day one. Decide how to handle this:

- [ ] **Option A (recommended):** Run a one-time DB migration before go-live that sets `trial_expires_at` to the enforcement date for all users whose trial would otherwise expire before it
- [ ] **Option B:** Change `hasFullAccess()` in `User.java` to also return `true` if the current date is before the enforcement date (removes trial logic entirely until that date)
- [ ] **Option C:** Accept staggered enforcement — document the decision

---

## 7. End-to-End Checkout Smoke Test

- [ ] Run through the full subscription flow in dev: sign up → trial expires → visit `/membership` → select monthly plan → complete Stripe checkout → confirm membership activates
- [ ] Verify the Stripe webhook fires and membership status updates in the DB
- [ ] Verify the dunning emails fire correctly for a simulated failed payment

---

## 8. Deploy to Prod

- [ ] Deploy via `./scripts/deploy.sh prod` (never `npx cdk deploy` directly)
- [ ] Confirm ECS tasks pick up the new Stripe secrets (secrets are read at container start — a rolling restart is required after setting them)
- [ ] Confirm `/payments/config` API returns `"enabled": true` in production
