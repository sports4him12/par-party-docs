# GolfSync — Executive MVP Review
**Prepared:** 2026-04-06 (CFO/COO/CMO) · Updated 2026-04-07 (CTO) · Progress update 2026-04-07  
**Perspective:** CFO · COO · CMO · CTO  
**Purpose:** Identify what to change, cut, or double down on before scaling

> **Legend (progress update):** ✅ Done · 🔶 Partial · ⬜ Open

---

## What We're Looking At

GolfSync is a **golf-centric social platform** combining tee-time booking, tournament discovery, round scoring, friend matchmaking, and payment tracking — all under a subscription model ($9.99/month, $79.99/year) with a 30-day free trial. The tech stack is solid (Spring Boot, Next.js, AWS CDK), the architecture is clean, and infrastructure-as-code is in place. On paper, the foundation is production-ready.

The honest read: **the bones are good, and the core loops are now in place.** The remaining gaps are activation, marketing infrastructure, real tee-time data, and cost optimization.

---

## CFO View — Revenue, Unit Economics & Monetization

### The Core Problem: We Don't Know Our Numbers

⬜ There is **no analytics, no cohort tracking, no revenue reporting**. A CFO cannot make capital allocation decisions blind. Before spending another dollar on features, instrument the basics:

- Trial-to-paid conversion rate
- Monthly churn by plan
- CAC by acquisition channel
- ARPU and LTV by cohort
- Average time to first paid action

Without these, every product decision is a guess.

### Subscription Model — ✅ Fixed

~~The current payment flow creates one-time Stripe payment intents with manual expiry tracking. There is **no auto-renewal, no webhooks, no dunning**.~~

**Resolved (2026-04-07):** Stripe subscription webhooks are now implemented and are the authoritative billing source:
- `checkout.session.completed` — provisions membership on successful checkout
- `invoice.paid` — extends access for the next billing cycle automatically
- `invoice.payment_failed` — sends payment-failed email with a link to update payment; access not immediately revoked (Stripe retries)
- `customer.subscription.updated` — syncs plan tier and expiry on upgrade/downgrade
- `customer.subscription.deleted` — immediately revokes access and sends cancellation email

Webhook handler at `POST /api/stripe/webhook` uses raw body HMAC signature verification (no body-parsing before the check). All five handlers are idempotent.

🔶 **Still open:**
- Full dunning sequence (payment failed → grace period email at day 3 → downgrade at day 7) — currently only the initial payment-failed email is sent; Stripe's retry logic handles day 3/7 retries but no second email is triggered
- Renewal reminder email 7 days before subscription expiry
- Stripe Subscription objects vs. one-time PaymentIntents for the checkout flow itself — the checkout currently creates a PaymentIntent, not a Subscription object; the webhook infrastructure is in place but the checkout needs to create a `Subscription` (or use Stripe Checkout in subscription mode) to feed `invoice.paid` events

### Revenue is One-Dimensional

⬜ The monetization strategy is still subscription-only. Revenue levers remain on the roadmap:

| Lever | Status |
|---|---|
| Tournament entry fee commission (5–10%) | ⬜ Open |
| GolfNow referral/affiliate | ⬜ Blocked (integration still mocked) |
| Premium tier | ⬜ Open |
| Course partnerships | ⬜ Open |
| B2B white-label | ⬜ Open |

### Free Tier Restructured — ✅ Done

~~The free tier is too restrictive in the wrong places: social features paywalled while low-value browsing is free.~~

**Resolved (2026-04-07):** Free tier eliminated entirely. Replaced with a **30-day free trial** — full access to every feature from day one, no credit card required. After 30 days, a paid subscription is required to keep access. The dashboard shows a live countdown (amber at ≤5 days, red on expiry). Users who hit a `TRIAL_EXPIRED` 403 are redirected to the subscription page with context. The activation problem — users hitting a paywall before experiencing social value — is resolved.

### Pricing Sanity Check

⬜ Seasonal and per-event pricing options remain open:
- **Seasonal plan**: $29.99 for 6 months (April–September)
- **Per-event pricing**: $4.99 to unlock for a single round
- **Family/group plan**: one subscription, up to 4 golfers

---

## COO View — Operations, Scalability & What's Not Production-Ready

### Blocker #1: GolfNow Integration is Mocked

⬜ The tee time booking feature remains mocked. The `GolfNowClient` returns hardcoded data. **This is the highest-priority unresolved blocker.** GolfSync cannot be marketed as a tee-time booking platform until this is real.

**Decision still needed:** (a) negotiate GolfNow API partnership, (b) pivot to Supreme Golf or TeeOff, or (c) reposition as a "round management" platform and remove the tee-time booking promise until the integration exists.

### Blocker #2: Subscription Lifecycle — ✅ Fixed

~~No automated membership management; customer support handles every renewal manually.~~

**Resolved (2026-04-07).** See CFO section. Webhook handlers now manage the full subscription lifecycle automatically.

### Tournament Database Doesn't Scale

🔶 **Partially addressed.** Live tournament discovery via Google Custom Search API now supplements the 12 seeded east-coast tournaments, with city/state forwarding, SGA domain queries, and a 2-hour result cache. Users can also register for discovered tournaments directly via a safe URL resolver.

⬜ Still open: no pipeline for keeping structured tournament data fresh nationally. Options remain:
- License data from USGA, PGA, Golf Genius
- User-submitted tournament board (crowdsourced via existing report workflow)
- State golf association partnerships

### AI Conversation Memory Doesn't Survive Server Restart

⬜ In-memory `HashMap` conversation history is unchanged. In a multi-task prod deployment, users hitting different ECS tasks get different memory. Redis is still needed before real user scale.

### Observability — 🔶 Partial

✅ CloudWatch JSON structured logging, error alarm (5 ERRORs/5 min → SNS), and log tail commands are in place.

⬜ Still missing:
- Business metrics (signups/day, trial → paid conversions, round creation rate)
- RDS Performance Insights
- Distributed tracing / correlation IDs
- Route53 health checks and synthetic monitoring

### Rate Limiting — 🔶 Partial

✅ Auth endpoints are rate-limited (10 req/min per IP, sliding window, implemented 2026-04-06).

⬜ AI chat, tournament search, and user search remain unprotected. A single user can spam the OpenAI integration and generate significant API costs. Add: AI chat (5 req/min), search (30 req/min), tee time search (10 req/min).

### Security — ✅ Multiple Fixes Shipped

All of the following were resolved (2026-04-06):
- ✅ JWT moved from localStorage to httpOnly cookie (XSS attack surface eliminated)
- ✅ IDOR on change-password — callers can only change their own password
- ✅ Email exposure — `/api/users/search` now returns `PublicUserResponse` (no email field)
- ✅ AI booking agent no longer leaks user emails in friend search results
- ✅ User search minimum 2-character validation (enumeration protection)

### GDPR / Data Rights Gap

⬜ No user self-deletion API. Admin hard-delete exists but users cannot delete their own accounts. Legal requirement in the EU; trust signal in the US.

### Session Architecture Will Break Under Load

⬜ AI conversation state is in-process memory, no caching on tournament or user data. Add Redis before any marketing campaigns that drive traffic spikes.

### Missing Operational Runbook Pieces

⬜ All still open:
- Canary deployment strategy (ECS blue/green or weighted routing)
- Database backup restore test
- Incident response playbook
- On-call rotation (SNS alarm goes to one email address)
- Database connection pool monitoring

---

## CMO View — Acquisition, Retention & Why Someone Would Actually Use This

### Registration Friction — ✅ Reduced

~~The flow is: register → browse courses → hit a paywall → leave.~~

**Resolved (2026-04-07):** Registration is now a single form with no plan selection. Users land directly in the product with full access. The 30-day trial removes the paywall entirely from the activation path.

### The Freemium Funnel — 🔶 Improving

| Stage | Previous State | Current State |
|---|---|---|
| **Awareness** | No marketing infrastructure | ⬜ Unchanged |
| **Acquisition** | Register → choose plan → paywall | ✅ Register → full trial access |
| **Activation** | Browse courses, hit paywall | 🔶 Full access, but no onboarding flow yet |
| **Retention** | No notifications or reminders | 🔶 Availability polls help coordination; no push notifications yet |
| **Revenue** | Paywall before social value | ✅ Trial delivers full value before conversion ask |
| **Referral** | Email invites only | ⬜ No shareable round links yet |

⬜ Still the most valuable next activation investment: a 3-step onboarding flow (find home course → invite a friend → create first round).

### Group Coordination — ✅ Solved

**Resolved (2026-04-07):** Availability Polls allow a booker to propose 2–4 date/time options, invite friends, and collect I'm In / Can't Make It votes before locking in a tee time. Dashboard widget shows active polls with "Vote needed" badge. This directly addresses the coordination pain the primary target market (recreational group golfers) experiences.

Additional booker tooling shipped:
- ✅ Add players to an existing round post-creation
- ✅ Nudge pending invitees (in-app reminder message per non-responding player)
- ✅ Per-player payment settlement (dues tracking, PAID/WAIVED status, booker controls)

### The "Aha Moment"

⬜ Still missing a viral moment. Candidates remain:
- Shareable round scorecard (PNG export for Instagram/X)
- Friend leaderboard (where do you rank this season?)
- Public round lobby (matchmaking for open rounds)

The friend score feed is live (paid feature on dashboard), but it's internal — not shareable externally.

### Tournament Discovery — ✅ Significantly Improved

**Resolved (2026-04-06/07):**
- ✅ Live tournament discovery via Google CSE with SGA site: queries, city/state forwarding, 2-hour cache
- ✅ Admin-featured tournaments appear at top of dashboard, location-filtered by haversine distance
- ✅ Safe registration URL resolver — clicking Register resolves the top organic CSE result with an adult-content filter, bypassing ads and search pages
- ✅ Tournament interest preferences (Scramble, Amateur, Charity, etc.) persist in localStorage and sort results

⬜ Still open: national coverage beyond the 12 seeded east-coast tournaments.

### Social Proof, Positioning, Referral

⬜ All open. No public profiles, no leaderboards, no shareable links, no referral program.

---

## CTO View — AWS Spend, Compute Architecture & Technical Risk

### Where the Money Goes Right Now

At current scale (dev + prod), the rough monthly AWS bill:

| Service | Dev | Prod | Notes |
|---|---|---|---|
| **ECS Fargate** (API) | ~$35 | ~$70–140 | 0.25 vCPU / 512 MB dev; 0.5 vCPU / 1 GB × 2 tasks prod |
| **ECS Fargate** (Camunda) | ~$35 | ~$70 | Always-on even when idle |
| **RDS MySQL** (db.t3.micro) | ~$15 | ~$30–60 | Prod Multi-AZ: $60 |
| **ALB** | ~$20 | ~$20 | Flat LCU base |
| **NAT Gateway** | ~$35 | ~$35 | Biggest hidden cost; charged per GB egress |
| **Secrets Manager** | ~$5 | ~$5 | Per-secret monthly fee |
| **CloudWatch Logs** | ~$5 | ~$10 | Ingestion + retention |
| **ECR** | ~$1 | ~$1 | Image storage |
| **Total** | **~$150/mo** | **~$220–320/mo** | All three cost wins below still open |

### CTO Priority List — Progress Update

| Priority | Action | Savings / Impact | Status |
|---|---|---|---|
| **Immediate** | Embed Camunda in-process (remove second ECS service) | ~$70–105/month | ⬜ Open |
| **Immediate** | Replace dev Fargate with EC2 t3.small | ~$50/month | ⬜ Open |
| **Near-term** | Replace NAT Gateway with VPC Endpoints | ~$30–35/month | ⬜ Open |
| **Near-term** | Serve Next.js static assets via CloudFront + S3 | ~$20/month + latency | ⬜ Open |
| **Near-term** | Run Liquibase as a pre-deploy ECS task (not at startup) | Reliability | ⬜ Open |
| **Near-term** | Add CI/CD pipeline (GitHub Actions → ECS deploy) | Velocity | ⬜ Open |
| **Medium-term** | Redis for AI conversation state + caching | Correctness at scale | ⬜ Open |
| **Medium-term** | Evaluate App Runner for stateless API services | Simplicity | ⬜ Open |

None of the compute cost wins have been implemented yet. These remain the highest-leverage infrastructure changes available.

### Technical Debt — 🔶 Some Addressed

✅ **Stripe one-time PaymentIntents → Subscription webhooks** — The billing infrastructure is now event-driven. The `checkout.session.completed` handler stores `stripeCustomerId` and `stripeSubscriptionId` on the user, enabling all downstream events to look up users by Stripe ID without relying on session context.

✅ **Liquibase schema drift risk reduced** — Migrations 021 and 022 add `trial_expires_at`, `stripe_customer_id`, and `stripe_subscription_id` columns with the correct indexes. The existing backfill in migration 021 ensures all pre-migration users receive a trial window from their `created_at` date.

⬜ **Liquibase race condition on startup** — Still open. Multi-task ECS deployments race to apply migrations. Move to a one-shot pre-deploy task.

⬜ **Single-region deployment** — No multi-region or failover. Friday morning downtime during peak booking hours remains a risk.

⬜ **No CDN on Next.js frontend** — Static assets still served from Fargate behind ALB on every request.

⬜ **No CI/CD pipeline** — Deployments are still manual `cdk deploy` commands.

---

## Priority Action Plan — Updated Status

### Immediate (0–30 days)

| # | Action | Status |
|---|---|---|
| 1 | Stripe subscription webhooks | ✅ Done (2026-04-07) |
| 2 | Decide GolfNow integration or reposition | ⬜ Decision still pending |
| 3 | Add 3-step onboarding flow (course → friend → round) | ⬜ Open |
| 4 | Move social features out from behind paywall | ✅ Done — free trial gives full access to all features from day one |
| 5 | Instrument signup → paid conversion funnel | ⬜ Open |

### Near-Term (30–90 days)

| # | Action | Status |
|---|---|---|
| 6 | Shareable round invite links | ⬜ Open |
| 7 | Course leaderboard (top scores per course) | ⬜ Open |
| 8 | Shareable score card (PNG export) | ⬜ Open |
| 9 | Move AI conversation state to Redis | ⬜ Open |
| 10 | Expand tournament database to 100+ nationally | 🔶 CSE discovery added; no structured pipeline |
| 11 | Rate limit AI and search endpoints | ⬜ Open (auth only) |
| 12 | GDPR: user self-deletion API | ⬜ Open |

### Medium-Term (90–180 days)

| # | Action | Status |
|---|---|---|
| 13 | Tournament entry fee commission (5–10%) | ⬜ Open |
| 14 | Handicap calculation from round scores | ⬜ Open |
| 15 | Web push notifications | ⬜ Open |
| 16 | PWA with home screen install | ⬜ Open |
| 17 | Referral program (1 month free per paid conversion) | ⬜ Open |
| 18 | Seasonal pricing ($29.99 for 6 months) | ⬜ Open |
| 19 | B2B pilot: sell to 2–3 golf clubs | ⬜ Open |
| 20 | Analytics dashboard (retention, churn, LTV) | ⬜ Open |

---

## Summary Assessment — Updated

| Dimension | Previous Rating | Current Rating | Notes |
|---|---|---|---|
| **Technical foundation** | ★★★★☆ | ★★★★☆ | Security hardening, webhook infrastructure, trial model all solid |
| **Product-market fit** | ★★☆☆☆ | ★★★☆☆ | Core social loop now complete; GolfNow still mocked |
| **Monetization** | ★★☆☆☆ | ★★★☆☆ | Structural flaw (no webhooks) fixed; one revenue stream remains |
| **Growth engine** | ★☆☆☆☆ | ★★☆☆☆ | Trial removes activation friction; referral/viral loop still absent |
| **Operations readiness** | ★★★☆☆ | ★★★☆☆ | Webhook lifecycle is now automated; compute costs unchanged |
| **MVP scope fit** | ★★★☆☆ | ★★★★☆ | Availability polls, booker tooling, and trial model close the core gaps |

**Bottom line:** The product is meaningfully further along than the initial assessment. The two most critical structural problems — no subscription lifecycle and a paywall blocking social activation — are both resolved. The remaining blockers are: (1) a real tee-time data source, (2) a conversion funnel you can see and optimize, and (3) a referral mechanism to make growth self-sustaining. The compute cost recommendations remain unimplemented and represent ~$150–190/month in savings with no product impact.
