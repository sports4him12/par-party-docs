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

## 3. Stripe — Create Products and Prices ✅ DONE 2026-04-28

Monthly billing only — no annual variants.

- [x] Organizer recurring price at $4.99/month created in Stripe Dashboard (live mode)
- [x] League Pro recurring price at $14.99/month created (live mode)
- [x] Both `price_...` IDs captured and wired into CDK (`prodConfig.stripeOrganizerPriceId` / `stripeLeagueProPriceId` in `golfsync-cdk/bin/golfsync-cdk.ts`)

---

## 4. Stripe — Configure Keys and Secrets ✅ DONE 2026-04-28

- [x] `STRIPE_SECRET_KEY` set to live `sk_live_...` in Secrets Manager (`golfsync-prod/stripe-secret-key`)
- [x] `STRIPE_PUBLISHABLE_KEY` set to live `pk_live_...` in Secrets Manager (`golfsync-prod/stripe-publishable-key`)
- [x] `STRIPE_ORGANIZER_PRICE_ID` injected as plain env var on API ECS task via CDK
- [x] `STRIPE_LEAGUE_PRO_PRICE_ID` injected as plain env var on API ECS task via CDK
- [x] `.env.prod.example` updated to point at CDK source rather than Secrets Manager for the price IDs

---

## 5. Stripe — Register the Webhook Endpoint ✅ DONE 2026-04-28

- [x] Endpoint registered: `https://golfsync.io/api/stripe/webhook` (Your account, snapshot payload, API version 2026-04-22.dahlia)
- [x] Subscribed events: `checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`
- [x] `whsec_...` signing secret stored in Secrets Manager (`golfsync-prod/stripe-webhook-secret`)

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

## 8. Deploy to Prod ✅ DONE 2026-04-28

- [x] Deployed via `./scripts/deploy.sh prod --skip-cypress --skip-ci-check --yes` (release tag `prod-2026-04-29-0339`)
- [x] ECS tasks restarted, new secrets and price-ID env vars verified on running task definition
- [ ] Confirm `/payments/config` API returns `"enabled": true` in production (deferred — covered by smoke test in section 7)

---

## Completed

- 2026-04-28 — Live Stripe membership integration deployed to prod. Sections 3, 4, 5, 8 closed out: live keys + price IDs in place, webhook endpoint registered, ECS rollout verified. Customer Portal also configured in Stripe Dashboard (cancellations end-of-period, plan switching across both tiers, Terms/Privacy URLs at golfsync.io/terms + /privacy, support email support@golfsync.io). Remaining open items (1, 2, 6, 7) are the actual enforcement-cutover work — pick the date, re-enable drip emails, decide trial alignment policy, run the smoke test.
- 2026-04-18 — Fixed the wrong webhook path in `golfsync-docs/admin-runbook.md` (corrected from `/api/payments/webhook` to `/api/stripe/webhook`).
