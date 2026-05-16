# Stripe setup — what Ryan needs to do

**Last updated:** 2026-05-16
**Owner:** Ryan
**Why this exists:** Claude is building league-dues collection via Stripe Connect. To make any of that code actually charge a real card, the Stripe Dashboard needs to be configured first. This walks you step-by-step so you don't have to read the engineering RFC.

**Total time:** ~50 min of your active work, then 1-3 business days waiting on Stripe approval, then ~20 min of follow-up.

**When to do each step:** the order matters. Step A1 starts a Stripe approval clock that's 1-3 business days. Do A1 today. A2 through A4 can't happen until A1 approves.

---

## A1 — Activate Stripe Connect on your platform account (15 min, do today)

This is the only step that's truly blocking. Without Connect approved, none of the code can charge anything.

1. Go to **Stripe Dashboard** → make sure you're on the **Golf Sync LLC** account (not personal)
2. Sidebar: **Settings** (gear icon, bottom left)
3. Find **Connect** in the settings menu → click **Get started**
4. Stripe asks: "What's your platform?" → pick **Platform or marketplace**
5. Fill out the application:
   - **Platform name:** Golf Sync
   - **Platform website:** `https://golfsync.io`
   - **Business type:** Limited Liability Company (single-member) — Golf Sync LLC
   - **Country of operation:** United States
   - **Brief description of what your platform does (1-2 sentences):**

     > Golf Sync connects amateur golf leagues with payment infrastructure for tournament dues, hosted-tournament registrations, and sponsorships. League organizers connect their own Stripe account via Stripe Connect Express to receive payments directly from players and sponsors; Golf Sync facilitates the connection and collects a small platform fee.

   - **Industry / category:** Sports & Recreation (or similar)
   - **Logo upload:** use `golfsync-web/public/brand/golfsync-logo.svg` (or any current GolfSync logo PNG)
   - **Brand color:** Navy `#0F172A` (or whichever navy hex Ryan considers the official one — match the membership-billing brand for consistency)
6. When asked about onboarding type → pick **Express**
   - This is the version where Stripe handles the connected-account UI; owners see a Stripe-branded flow, get a Stripe Express dashboard for their payouts, and Stripe handles all KYC/tax stuff for them.
   - The alternative ("Standard") requires every owner to already have their own Stripe account — that's a non-starter for our user base.
7. Submit the application

**What happens next:** Stripe reviews this. Usually 1-3 business days. They may email you with follow-up questions ("describe a typical transaction in more detail," "how do you verify the identity of your platform users," etc.) — answer honestly and reference the RFC at `golfsync-docs/architecture/league-tournament-dues-2026-05-16.md` if useful. You'll get an email when approved.

✅ **Done with A1 when:** you can see "Connect" as an active menu item in your Stripe Dashboard sidebar.

---

## A2 — Configure Connect settings (15 min, AFTER A1 is approved)

This is where you tell Stripe the technical details of how Golf Sync will use Connect.

1. Stripe Dashboard → **Settings → Connect → Settings**
2. **Branding section:**
   - Upload the same logo from A1 (Stripe stores branding separately for the platform vs. for connected-account onboarding)
   - Brand color: same navy as A1
   - Add a one-paragraph "About Golf Sync" blurb that connected-account owners see during their Express onboarding. Suggested:

     > Connect your Stripe account to start collecting league dues, tournament fees, and sponsorships through Golf Sync. Funds go directly to your Stripe balance and pay out to your bank on Stripe's standard schedule. Golf Sync's 1% platform fee is itemized in your transaction breakdowns.

3. **Integration → Account links / Redirect URLs section:**
   - Add `https://golfsync.io/api/stripe/connect/callback` to the list of allowed return URLs
   - (For local dev testing later: also add `http://localhost:3001/api/stripe/connect/callback`)
4. **Webhook endpoints for Connect events** (this is a SEPARATE webhook from your existing membership webhook):
   - Stripe Dashboard → **Developers → Webhooks → Add endpoint**
   - **Endpoint URL:** `https://api.golfsync.io/api/webhooks/stripe/connect`
   - **Listen to:** events on **Connected accounts** (this is the toggle that makes this a Connect webhook, distinct from the membership one which listens on **Your account**)
   - **Events to send:**
     - `account.updated` (when a connected account's status changes — KYC complete, KYC failed, payouts enabled, charges disabled, etc.)
     - `account.application.deauthorized` (when an owner disconnects)
     - `payment_intent.succeeded` (already wired for membership; the Connect version is on the connected account)
     - `payment_intent.payment_failed`
     - `charge.refunded`
     - `charge.dispute.created`
   - **Save** — Stripe shows you a signing secret (`whsec_xxx`). **Copy this — you'll need it for A4.**

✅ **Done with A2 when:** you have a `whsec_xxx` for the Connect webhook AND you can see the redirect URL in your settings.

---

## A3 — Capture your Connect client ID (2 min, AFTER A2)

This is just locating an existing value, not setting anything up.

1. Stripe Dashboard → **Settings → Connect → Settings → Integration**
2. There's a section labeled **"Client ID"** with a value like `ca_AbCdEf1234567890...`
3. **Copy this value.** You'll need it for A4. There are usually TWO client IDs visible — one for **test mode** (`ca_test_...`) and one for **live mode** (`ca_live_...`). Copy both; you'll add both to the right places.

✅ **Done with A3 when:** you have both `ca_test_xxx` and `ca_live_xxx` written down.

---

## A4 — Add the new secrets to env files (5 min, after A2 + A3)

These are the values the code needs to actually talk to Stripe Connect. Two places they live:

### Local dev (`.env.dev`)

Open `/Users/ryanpick/ryanrpick/GolfSyncApplication/.env.dev` and add:

```bash
# Stripe Connect (added 2026-05-16 for league dues collection)
STRIPE_CONNECT_CLIENT_ID=ca_test_xxxxxxxxxxxxxxxxxxxxxxxx
STRIPE_CONNECT_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxxxxx
```

Use the **test mode** values for local dev.

### Production (AWS Secrets Manager)

You'll need to add two new secrets in the AWS console:

1. AWS console → **Secrets Manager** (region: us-east-1, profile: golfsync-prod)
2. Find the existing `golfsync-prod-stripe-secrets` secret (or whatever the current naming pattern is — Claude will know after one grep)
3. **Edit** → add two new key/value pairs:
   - `STRIPE_CONNECT_CLIENT_ID` → your `ca_live_xxx` value
   - `STRIPE_CONNECT_WEBHOOK_SECRET` → the `whsec_xxx` from A2
4. **Save**

After this is saved, when Claude redeploys, the new env vars will land on the running ECS tasks automatically. No restart needed beyond the deploy itself.

✅ **Done with A4 when:** `.env.dev` has the new lines AND Secrets Manager has the new keys.

---

## B1 — Lawyer review of `/terms/payments` (NOT blocking code; blocking public launch)

The starter draft at `https://golfsync.io/terms/payments` needs an attorney pass before the feature opens to the public.

- **Cost:** $500-2000 for ~1 hour with a small-business attorney familiar with US payments law and marketplace facilitator rules.
- **What to ask them to review specifically:**
  - Is Golf Sync LLC correctly framed as a "marketplace facilitator" (vs. money services business — your state may have specific definitions)
  - The surcharge language and any state-specific carve-outs (5 US states currently have surcharge restrictions: NY, CA, CO, OK, KS — last verified 2024; ask the attorney for current)
  - Dispute pass-through to the owner (is your liability properly limited)
  - The 30-day fee-change notice clause (some state consumer-protection laws have stricter notice requirements)
  - Whether the limitation-of-liability section needs to be added (the starter doesn't have one)
- **What this is NOT a substitute for:** generic legal advice. Get a real attorney.

When the review is complete:
- Update the `Effective date:` line at the top of the page
- Remove the amber "Starter draft — pending legal review" banner
- Commit the changes

✅ **Done when:** the amber banner is gone from `/terms/payments`.

---

## C1 — Reach out to the Stableford-based first-paying customer (15 min, do today)

He's going to be your dogfood for the first ~1 week after launch.

**Suggested message:**

> Hey [name],
>
> I'm building Stripe payment collection for league dues in Golf Sync — you'd be my first user. In about a week you'd be able to:
>
> - Connect your own Stripe account (takes ~5 minutes)
> - Pick which tournaments collect via card vs. cash/Venmo (per tournament)
> - Players can pay their dues with a credit card, Apple Pay, or Google Pay
> - Money goes directly to your account (Golf Sync takes 1%, players cover the card processing fee)
> - You can still mark anyone paid manually if they hand you cash at the course
>
> Want to be the first to try it? I'll walk you through Stripe setup on a 15-min call once it's ready.

If he says yes → you have a real test channel. If he says no → still ship it, but find someone else to dogfood with before broader rollout.

✅ **Done when:** you have a yes/no from him.

---

## Decisions you'll need to give Claude during the build

These don't block you from doing A1 today. They come up mid-build:

| Decision | Default if you don't reply | When Claude needs it |
|---|---|---|
| Payout schedule for connected accounts | Stripe default (rolling 2-day for US) | Before webhook code is wired |
| Minimum dues amount that allows Stripe collection | $1.00 (Stripe's hard minimum) | Before Stripe Elements UI |
| Do you want a "Coming soon" email to existing league owners when this ships? | No auto-send — your call | Before public ship |

Just answer these as questions come up in chat with Claude.

---

## Summary timeline

| What | When | Who | Time |
|---|---|---|---|
| A1 Activate Connect | Today | You | 15 min |
| C1 Reach out to first customer | Today | You | 15 min |
| Wait for Stripe approval | 1-3 business days | Stripe | — |
| A2 Configure Connect settings | After A1 approval | You | 15 min |
| A3 Capture client ID | After A2 | You | 2 min |
| A4 Add env vars + Secrets Manager | After A3 | You | 5 min |
| B1 Lawyer review | Before public launch (whenever) | You + attorney | ~$500-2000 |
| Claude builds in parallel | Throughout | Claude | 2-3 days |

Once A1 is submitted today, you can hand Claude the rest of the values as they become available and Claude keeps coding in parallel.
