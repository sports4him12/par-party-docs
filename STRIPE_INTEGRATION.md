# Stripe Integration Plan

Two distinct integration tracks, both flowing through Stripe but solving different problems:

1. **Platform Stripe** — GolfSync charges users for membership tiers (already live).
2. **Stripe Connect** — users (typically league owners) collect money *from other users* on the platform: tournament entry fees, dues, side-pot buy-ins. GolfSync takes an application fee. Not yet built.

This doc captures what's live, what's needed for Connect, and the gotchas.

---

## Status as of 2026-04-28

| Feature | Status | Notes |
|---|---|---|
| Membership tiers (Organizer $4.99/mo, League Pro $14.99/mo) | **Live** | Checkout Sessions + customer portal + webhook. Monthly billing only. Trial through 2026-05-31; no upgrade enforcement until then. |
| Stripe Connect (Express accounts for users) | **Not built** | This doc's primary subject. |
| Tournament entry-fee collection | **Manual today** | League owner uses Venmo + a screenshot. Once Connect lands, switch to Stripe-mediated. |
| Side-pot / skins buy-ins | **Manual** | Same as above. |
| Refunds (membership) | **Manual via dashboard** | No in-app UI; engineer-triggered. |

---

## Track 1: Platform Stripe (membership) — what exists

**Files**
- [`StripeConfig.java`](../golfsync-api/src/main/java/com/golfsync/config/StripeConfig.java) — sets `Stripe.apiKey` from `STRIPE_SECRET_KEY` env var on boot.
- [`MembershipService.java`](../golfsync-api/src/main/java/com/golfsync/service/MembershipService.java) — creates Checkout Sessions, opens billing portal sessions, handles tier transitions on webhook events.
- [`StripeWebhookController.java`](../golfsync-api/src/main/java/com/golfsync/controller/StripeWebhookController.java) — verifies HMAC signature with `STRIPE_WEBHOOK_SECRET`, dispatches to MembershipService.

**Env vars (already set in prod)**
```
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_ORGANIZER_PRICE_ID=price_...
STRIPE_LEAGUE_PRO_PRICE_ID=price_...
```

**Flow**
1. Web client hits `POST /api/membership/checkout` with chosen tier.
2. API creates a Checkout Session with `success_url` + `cancel_url` pointing back to `/membership/success` and `/membership`.
3. User pays on Stripe-hosted page.
4. Stripe sends `checkout.session.completed` → our webhook flips the user's tier and persists `stripe_customer_id` + `stripe_subscription_id`.
5. Recurring invoices fire `invoice.payment_succeeded` (no-op currently — tier already set) and `invoice.payment_failed` (downgrade after grace period — TODO, see "Open items" below).

**Webhook events we listen for**
- `checkout.session.completed` — initial activation
- `customer.subscription.updated` — tier change via portal
- `customer.subscription.deleted` — cancellation
- `invoice.payment_failed` — *currently logged only*; should downgrade to FREE after retries exhaust

---

## Track 2: Stripe Connect (user payouts) — what's needed

The pitch: a league owner who runs an 8-event Spring season can collect $25 dues × 24 players = $4,800 through GolfSync, and the platform takes a 3% application fee. Today they Venmo, the money arrives unevenly, and 2 of the 24 chase them for two weeks.

### Account type: **Express**

| Type | Onboarding | Dashboard | KYC handled by | Use when |
|---|---|---|---|---|
| Standard | Stripe-hosted, full | Full Stripe dashboard | Stripe | User already has a Stripe account |
| **Express** | **Stripe-hosted, ~3 minutes** | **Limited Stripe dashboard for refunds/payouts** | **Stripe** | **Default for marketplaces — pick this** |
| Custom | API-only, ~50 fields | We build it | Us | Heavily branded UX (not worth the work) |

**Express** is the right call: Stripe collects KYC and bank info on their hosted page (lower compliance burden on us), the user gets a basic dashboard for refunds and payout history, and the app integration is small.

### Capabilities to request

For collecting golf-event entry fees in the US:
- `transfers` — required to send funds to the connected account
- `card_payments` — required to charge cards on their behalf

EU/UK/CA users add country-specific requirements (typically just an address + bank account in local currency). MVP = US only; flag international users as "coming soon" in the onboarding UI.

### Schema additions

```sql
-- 089-stripe-connect.sql (proposed)
ALTER TABLE users
    ADD COLUMN stripe_connect_account_id VARCHAR(40) NULL,
    ADD COLUMN stripe_connect_charges_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN stripe_connect_payouts_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN stripe_connect_onboarded_at TIMESTAMP NULL,
    ADD INDEX idx_users_stripe_connect (stripe_connect_account_id);

-- Per-tournament payment intent tracking
CREATE TABLE tournament_payments (
    id BIGINT NOT NULL AUTO_INCREMENT,
    tournament_id BIGINT NOT NULL,
    payer_user_id BIGINT NOT NULL,
    amount_cents INT NOT NULL,
    application_fee_cents INT NOT NULL,
    stripe_payment_intent_id VARCHAR(40) NOT NULL,
    status VARCHAR(20) NOT NULL,  -- REQUIRES_PAYMENT_METHOD / PROCESSING / SUCCEEDED / FAILED / REFUNDED
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    CONSTRAINT fk_tp_tournament FOREIGN KEY (tournament_id) REFERENCES league_tournaments(id) ON DELETE CASCADE,
    CONSTRAINT fk_tp_payer FOREIGN KEY (payer_user_id) REFERENCES users(id),
    CONSTRAINT uq_tp_intent UNIQUE (stripe_payment_intent_id)
);
```

Storing `charges_enabled` + `payouts_enabled` separately matters: a freshly onboarded account may have `charges_enabled=true` (can accept money) but `payouts_enabled=false` (Stripe is still verifying their bank). The tournament-create UI should let captains *configure* dues either way, but the registration "Pay" button must check `charges_enabled`.

### Onboarding flow ("are you sure you want to receive money on this internet")

The user-facing experience needs to communicate: *this is your money, GolfSync is just the rails, and Stripe will ask for your SSN/EIN because the IRS requires it.* The onboarding screen should walk them through it before the Stripe handoff so the SSN ask isn't a cold blast.

Proposed steps:

1. **Intro screen** ("Get paid through GolfSync")
   - Sets expectations: 3-step Stripe form, ~5 min, need bank account + SSN/EIN, 1099-K issued at year-end if they cross the IRS threshold.
   - Lists what they unlock: tournament entry collection, side-pot pots, dues auto-pay.
   - Single CTA: "Continue to Stripe."

2. **Account-link redirect**
   - API: `POST /api/stripe/connect/onboard` → creates the Connect account if missing, generates a one-time `AccountLink` (`type=account_onboarding`), returns the URL.
   - User redirects to Stripe's hosted form (legal name, DOB, SSN last-4, bank account or debit card).
   - Stripe sends them back to `/account/payouts?refresh=true` on success or expiry.

3. **Status screen** (back in our app)
   - Pulls the latest account status via `GET /api/stripe/connect/status` (server-side `Account.retrieve(...)`).
   - Renders one of: "All set — start collecting" / "Stripe is verifying your bank — you can collect now, payouts unlock in 2-3 days" / "Stripe needs more info" with a deep link back to the hosted form.
   - Status is also pushed via webhooks (`account.updated`) so the user doesn't have to refresh.

4. **First-payout celebration** (optional, recommended)
   - After their first successful tournament charge, fire an in-app banner + email: "Your $X is on its way to your bank, arrives [date]." Closes the loop on the "is this real?" anxiety that kills marketplace adoption.

### Charging flow (collecting tournament entry fees)

Two patterns; **Direct Charges with Application Fee** is the right default:

```
Player pays $25
  ├─ $24.25 lands in league owner's connected account (immediate, then payout per their Stripe schedule)
  ├─ $0.75 application fee → GolfSync platform account
  └─ Stripe takes their cut from the connected account's $24.25 (~$1.03 = 2.9% + 30¢)
                                                                  Owner nets ~$23.22
```

API shape:
```java
PaymentIntent intent = PaymentIntent.create(PaymentIntentCreateParams.builder()
    .setAmount(2500L)
    .setCurrency("usd")
    .setApplicationFeeAmount(75L)
    .setTransferData(TransferDataParams.builder()
        .setDestination(tournament.getLeague().getOwner().getStripeConnectAccountId())
        .build())
    .build());
```

**Application fee math** (tentative — needs a pricing call): start at **3%**. Stripe takes ~2.9% + 30¢ on the connected account. Total cost to the owner: ~6% + 30¢ on a $25 entry = ~$1.78 = ~7%. Defensible because the alternative (Venmo + spreadsheet) costs them an hour per tournament.

### Refund flow

`Refund.create(...)` against the original PaymentIntent + `reverse_transfer=true` so the application fee comes back too. Owner-initiated only (refunding random members would surprise them).

### Webhook events to add

- `account.updated` — sync `charges_enabled` + `payouts_enabled` on the user row
- `account.application.deauthorized` — user disconnected Stripe; clear stored fields, hide pay buttons across the app
- `payment_intent.succeeded` — flip `tournament_payments.status` to SUCCEEDED, mark registration paid
- `payment_intent.payment_failed` — flag the registration, email player, give them a "Try again" CTA
- `charge.refunded` — flip status to REFUNDED, email both parties
- `payout.paid` — useful for the "$X arrived in your bank" notification (Connect-account-scoped event, not platform)

Connect-account events arrive at the same webhook endpoint with `account` field set; route by presence of that field rather than separate URLs.

---

## What changes for the engineering org

### New env vars
```
STRIPE_CONNECT_CLIENT_ID=ca_...           # from Settings → Connect
STRIPE_PLATFORM_ACCOUNT_ID=acct_...        # our platform account
STRIPE_CONNECT_REFRESH_URL=https://golfsync.io/account/payouts?refresh=true
STRIPE_CONNECT_RETURN_URL=https://golfsync.io/account/payouts?return=true
```

### Stripe Dashboard setup (one-time)
1. **Settings → Connect** → enable Express
2. **Settings → Connect → Branding** — upload GolfSync logo, set support email, brand color (the navy used elsewhere)
3. **Settings → Connect → Onboarding** — set the live redirect URLs above
4. **Webhooks** → add a *Connect* endpoint (separate from the platform endpoint we already have) at `/api/stripe/connect-webhook` and subscribe to the events listed above
5. **Tax forms** — enable 1099-K issuance for US accounts (Stripe handles delivery)

### Compliance one-pager (do before launch)
- **Terms update**: add a section on third-party payment processing, that GolfSync is not a party to the underlying transaction, and that disputes go to the connected account holder.
- **Privacy update**: disclose that opting into payouts shares PII (name, address, SSN-or-EIN) with Stripe.
- **Marketplace rules**: not a money transmitter (we're a marketplace facilitating a service), but check the FinCEN guidance for the specific structure annually.

---

## Open items / known limitations

**Membership track**
- `invoice.payment_failed` doesn't downgrade. Today it logs and waits for the user to fix billing manually. Add a retry-then-downgrade after 3 failures.
- No in-app refund UI — engineers do it from the Stripe dashboard. Low priority.
- No usage-based billing; flat tier pricing only. Fine for v1.

**Connect track (when we build it)**
- International support — US-only at launch. EU adds VAT collection complexity; Canada adds CAD payouts.
- Payouts to the *player* (e.g. CTP winner gets paid out automatically). Today the side-pot UI tracks the winner but doesn't move money. Future work: a second `Transfer` from the league owner's connected account to the winner's connected account once both are onboarded. **Don't ship this until both sides are confirmed onboarded** — half-onboarded transfers fail in non-obvious ways.
- Disputes/chargebacks land on the connected account, not us. Owners need education on this in the onboarding flow.
- Tax form distribution: Stripe handles 1099-K, but the user has to know to expect it. Add a "Year-end tax info" line to the onboarding intro screen.

---

## Sequencing recommendation

1. **Now** — keep Connect on the backlog; membership Stripe is fine.
2. **Trigger to build Connect** = first 5 league owners ask for it OR a real-money tournament loss happens (one player ghosts, owner eats it).
3. **Build order when triggered**:
   1. Schema + onboarding screens (intro → AccountLink redirect → status)
   2. `account.updated` webhook handler so the status syncs without polling
   3. Tournament dues collection (Direct Charges + Application Fee), starting with one league as a beta
   4. Refund flow
   5. Side-pot winner payouts (only after #3 is stable for ~30 days)
4. **Don't build first**: international, custom-branded onboarding, recurring dues subscriptions on the Connect account. Each of these is its own sprint and not core to the wedge.

---

## References

- Connect overview: https://stripe.com/docs/connect
- Express accounts: https://stripe.com/docs/connect/express-accounts
- Direct charges with application fee: https://stripe.com/docs/connect/direct-charges
- Account links (the onboarding redirect): https://stripe.com/docs/api/account_links
- Webhook events for Connect: https://stripe.com/docs/connect/webhooks
