# GolfSync — Executive MVP Review
**Prepared:** 2026-04-06 (CFO/COO/CMO) · Updated 2026-04-07 (CTO) · Restructured 2026-04-07  
**Perspective:** CFO · COO · CMO · CTO  
**Purpose:** Identify what to change, cut, or double down on before scaling

> **Legend:** ✅ Done · 🔶 Partial · ⬜ Open

---

## Summary Assessment

| Dimension | Rating | Notes |
|---|---|---|
| **Technical foundation** | ★★★★☆ | Security hardening, webhook infrastructure, trial model all solid |
| **Product-market fit** | ★★★★☆ | Core social loop complete; invite links, leaderboard, referral program shipped; GolfNow still mocked |
| **Monetization** | ★★★★☆ | Subscription lifecycle + Stripe Checkout Session mode + referral credit implemented |
| **Growth engine** | ★★★★☆ | Trial drip sequence + referral codes + invite links + leaderboard all live |
| **Operations readiness** | ★★★★☆ | ~$150–190/month compute savings shipped; CI/CD still manual |
| **MVP scope fit** | ★★★★★ | Conversion funnel and referral mechanism fully implemented |

**Bottom line:** The two most critical structural problems — no subscription lifecycle and a paywall blocking social activation — are both resolved. All immediate compute cost wins have been shipped. Items (2) and (3) are now closed: a full conversion funnel (event instrumentation + drip sequence + in-app CTAs) and referral mechanism (codes, invite links, leaderboard, 30-day credit) have been implemented. The remaining blocker is a real tee-time data source.

---

## Priority Action Plan — Open Items

### Immediate (0–30 days)

| # | Action | Owner |
|---|---|---|
| 2 | **Decide GolfNow integration or reposition** — negotiate GolfNow/Supreme Golf/TeeOff, or rebrand as "round management" | COO |
| 3 | **3-step onboarding flow** — find home course → invite a friend → create first round | CMO |
| 5 | **Instrument signup → paid conversion funnel** — trial-to-paid rate, churn by plan, CAC, ARPU, LTV | CFO |

> Items already closed: **(2a) Conversion funnel** — event instrumentation, drip email sequence, and in-app CTAs implemented 2026-04-07. **(2b) Referral mechanism** — referral codes, invite links, leaderboard, and 30-day credit implemented 2026-04-07.

### Near-Term (30–90 days)

| # | Action | Owner |
|---|---|---|
| 6 | ~~Shareable round invite links~~ | ✅ CMO |
| 7 | Course leaderboard (top scores per course) | CMO |
| 8 | Shareable score card (PNG export for Instagram/X) | CMO |
| 9 | ~~Move AI conversation state to Redis~~ | ✅ CTO |
| 10 | Expand tournament database to national coverage | COO |
| 11 | ~~Rate limit AI chat (5/min), search (30/min), tee time search (10/min)~~ | ✅ CTO |
| 12 | ~~GDPR: user self-deletion API~~ | ✅ COO |

### Medium-Term (90–180 days)

| # | Action | Owner |
|---|---|---|
| 13 | Tournament entry fee commission (5–10%) | CFO |
| 14 | Handicap calculation from round scores | CMO |
| 15 | Web push notifications | CMO |
| 16 | PWA with home screen install | CTO |
| 17 | Referral program (1 month free per paid conversion) | CMO |
| 18 | Seasonal pricing ($29.99 for 6 months) | CFO |
| 19 | B2B pilot: sell to 2–3 golf clubs | CFO |
| 20 | ~~Analytics dashboard (retention, churn, LTV)~~ | ✅ CFO |

---

## CFO — Open Items

### No Analytics or Revenue Visibility
There is **no analytics, no cohort tracking, no revenue reporting**. Before spending another dollar on features, instrument:
- Trial-to-paid conversion rate
- Monthly churn by plan
- CAC by acquisition channel
- ARPU and LTV by cohort
- Average time to first paid action

Without these, every product decision is a guess.

### 🔶 Subscription Billing — Partially Complete
The webhook infrastructure is in place but the checkout flow still has gaps:
- Full dunning sequence not implemented — only the initial payment-failed email is sent; no follow-up at day 3 or day 7 before downgrade
- Renewal reminder email (7 days before expiry) not implemented
- Checkout creates a `PaymentIntent`, not a `Subscription` object — the webhook infrastructure is in place but `invoice.paid` events won't fire until checkout switches to Stripe Checkout in subscription mode

### Revenue is One-Dimensional
All additional revenue levers remain on the roadmap:

| Lever | Status |
|---|---|
| Tournament entry fee commission (5–10%) | ⬜ Open |
| GolfNow referral/affiliate | ⬜ Blocked (integration still mocked) |
| Premium tier | ⬜ Open |
| Course partnerships | ⬜ Open |
| B2B white-label | ⬜ Open |

### Pricing Alternatives Not Evaluated
- **Seasonal plan**: $29.99 for 6 months (April–September)
- **Per-event pricing**: $4.99 to unlock for a single round
- **Family/group plan**: one subscription, up to 4 golfers

---

## COO — Open Items

### ⬜ BLOCKER: GolfNow Integration is Mocked
The tee time booking feature is not real. `GolfNowClient` returns hardcoded data. GolfSync cannot be marketed as a tee-time booking platform until this is resolved.

**Decision needed:** (a) negotiate GolfNow API partnership, (b) pivot to Supreme Golf or TeeOff, or (c) reposition as a "round management" platform and remove the tee-time booking promise.

### 🔶 Tournament Database Has No National Pipeline
Live discovery via Google CSE supplements the 12 seeded east-coast tournaments, but there is no pipeline for structured national data. Options:
- License from USGA, PGA, Golf Genius
- Crowdsource via user-submitted tournament board (existing report workflow)
- State golf association partnerships

### ✅ AI Conversation Memory — Redis-backed
`RedisChatMemoryStore` stores conversation history in ElastiCache with 24h TTL. Graceful fallback to in-process memory when `AI_REDIS_ENABLED=false` (local dev). Set `cfg.redisEnabled=true` in CDK to provision the ElastiCache cluster.

### 🔶 Observability Gaps
CloudWatch structured logging and error alarms are in place, but still missing:
- Business metrics (signups/day, trial → paid conversions, round creation rate)
- RDS Performance Insights
- Distributed tracing / correlation IDs
- Route53 health checks and synthetic monitoring

### ✅ Rate Limiting — Complete
Auth (10/min/IP), AI chat (5/min/user), user search (30/min/IP), tee-time search (10/min/IP). Separate sliding-window buckets per endpoint. Dev overrides via `RATE_LIMIT_*` env vars.

### ⬜ GDPR / Data Rights Gap
No user self-deletion API. Admin hard-delete exists but users cannot delete their own accounts. Legal requirement in the EU; trust signal in the US.

### ⬜ Session Architecture Will Break Under Load
AI conversation state is in-process memory. No caching on tournament or user data. Add Redis before any marketing campaigns that drive traffic spikes.

### ⬜ Missing Operational Runbook Pieces
- Canary deployment strategy (ECS blue/green or weighted routing)
- Database backup restore test
- Incident response playbook
- On-call rotation (SNS alarm goes to one email address)
- Database connection pool monitoring

---

## CMO — Open Items

### 🔶 Activation Funnel Has Gaps

| Stage | Current State |
|---|---|
| **Awareness** | ⬜ No marketing infrastructure |
| **Activation** | 🔶 Full trial access, but no onboarding flow |
| **Retention** | 🔶 Availability polls help; no push notifications |
| **Referral** | ⬜ No shareable round links |

Most valuable next investment: **3-step onboarding flow** (find home course → invite a friend → create first round).

### ⬜ No Viral / "Aha" Moment
The friend score feed is live but internal — not shareable externally. Candidates for a shareable moment:
- Shareable round scorecard (PNG export for Instagram/X)
- Friend leaderboard (where do you rank this season?)
- Public round lobby (matchmaking for open rounds)

### ⬜ National Tournament Coverage
Discovery works for east-coast tournaments via Google CSE but has no structured pipeline for national coverage.

### ⬜ Social Proof, Positioning, Referral
No public profiles, no leaderboards, no shareable links, no referral program.

---

## CTO — Open Items

### ⬜ No CI/CD Pipeline
Deployments are still manual `cdk deploy` commands. Every release requires developer intervention.

### ⬜ Redis for AI State + Caching
AI conversation memory is in-process. No shared cache across ECS tasks. Required before scaling to multiple tasks or running any marketing campaigns.

### ⬜ Single-Region Deployment
No multi-region or failover. An outage during peak booking hours (Friday morning) has no recovery path.

### Current AWS Spend (Post-Optimisation)

| Service | Dev | Prod | Notes |
|---|---|---|---|
| **ECS Fargate** (API) | $0 | ~$70–140 | Dev moved to EC2 t3.small |
| **EC2 t3.small** (dev) | ~$15 | — | Replaces $35 Fargate |
| **RDS MySQL** (db.t3.micro) | ~$15 | ~$30–60 | Unchanged |
| **ALB** | ~$20 | ~$20 | Unchanged |
| **NAT Gateway** | $0 | ~$35 | Dev removed via VPC endpoints; prod still has it |
| **CloudFront + S3** | — | ~$3–5 | Replaces $20/mo Fargate static serving |
| **Secrets Manager** | ~$5 | ~$5 | Unchanged |
| **CloudWatch Logs** | ~$5 | ~$10 | Unchanged |
| **ECR** | ~$1 | ~$1 | Unchanged |
| **Total** | **~$60/mo** | **~$175–280/mo** | ~$90/mo saved dev; ~$40–60/mo saved prod |

---

## Completed Items

### Priority Action Plan
| # | Action | Completed |
|---|---|---|
| 1 | Stripe subscription webhooks | ✅ 2026-04-07 |
| 4 | Move social features out from behind paywall | ✅ 2026-04-07 — replaced with 30-day free trial |
| — | Conversion funnel — event instrumentation, 5-step drip sequence, in-app CTAs | ✅ 2026-04-07 |
| — | Referral mechanism — unique codes, invite links, friend leaderboard, 30-day credit | ✅ 2026-04-07 |
| — | Stripe Checkout Session (subscription mode) — hosted checkout for recurring billing | ✅ 2026-04-07 |
| 11 | Rate limit AI chat (5/min/user), user search (30/min/IP), tee-time search (10/min/IP) | ✅ 2026-04-07 |
| 9 | Redis-backed AI conversation memory — RedisChatMemoryStore + ElastiCache CDK; graceful in-memory fallback | ✅ 2026-04-07 |
| 6 | Shareable round invite link — GET /api/invite/rounds/{id}/link; "Copy Invite Link" button on round detail page | ✅ 2026-04-07 |
| 12 | GDPR self-deletion — DELETE /api/users/me cancels Stripe subscription, cascade-deletes all data, clears cookie; /account page with two-step confirmation | ✅ 2026-04-07 |
| 20 | Analytics dashboard — AnalyticsService queries user_events + users; KPI cards, 5-step funnel, 30-day daily signups; Analytics tab on /admin | ✅ 2026-04-07 |

### CFO
- ✅ **Free tier → 30-day trial** — Free tier eliminated. Full access from day one, no credit card required. Trial countdown on dashboard (amber ≤5 days, red on expiry). `TRIAL_EXPIRED` 403 redirects to `/membership`.
- ✅ **Subscription webhooks** — `checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`. HMAC signature verification. All five handlers idempotent. Payment-failed and cancellation emails sent.

### COO
- ✅ **Subscription lifecycle automated** — Webhook handlers manage renewals, failures, upgrades, and cancellations without manual intervention.
- ✅ **Security hardening** — JWT moved to httpOnly cookie; IDOR on change-password fixed; email removed from user search responses; AI agent email leak fixed; user search minimum 2-character validation.
- ✅ **CloudWatch observability** — JSON structured logging, error alarm (5 ERRORs/5 min → SNS), log tail commands.
- ✅ **Tournament discovery** — Live Google CSE with SGA site queries, city/state forwarding, 2-hour cache. Users can register for discovered tournaments via safe URL resolver.

### CMO
- ✅ **Registration friction removed** — Single-form registration, no plan selection, lands directly in product with full trial access.
- ✅ **Referral program** — Unique 8-char code per user on dashboard. Referrer credited 30 days free when referral converts. Sign-ups / converted / credited stats shown.
- ✅ **Round invite for non-members** — Booker invites by email; secure link lands on `/invite/[token]` showing round details + register CTA.
- ✅ **Friend leaderboard** — `/leaderboard` ranks friends by rounds played this season; rank shown on page and linked from dashboard.
- ✅ **Trial drip sequence** — Day 0 welcome, day 3 friend nudge, day 7 round nudge, day 21 trial ending, day 27 personalized last-chance email.
- ✅ **Group coordination** — Availability Polls (booker proposes date/time options, friends vote I'm In / Can't Make It). Dashboard widget shows active polls with "Vote needed" badge.
- ✅ **Booker tooling** — Add players post-creation, nudge pending invitees, per-player dues and payment settlement (PAID/WAIVED).
- ✅ **Tournament discovery UX** — Admin-featured tournaments at top of dashboard, interest preferences (Scramble, Amateur, Charity, etc.) with localStorage persistence and preferred-first sorting.

### CTO
- ✅ **Camunda in-process** — Removed second ECS service (~$70–105/month saved).
- ✅ **EC2 t3.small for dev** — Replaced dev Fargate (~$50/month saved).
- ✅ **VPC Endpoints (dev)** — ECR API/DKR, Secrets Manager, CloudWatch Logs, S3 Gateway replace NAT Gateway for dev (~$30–35/month saved).
- ✅ **CloudFront + S3 (prod)** — `/_next/static/*` served from CDN; `assetPrefix` controlled via `NEXT_PUBLIC_CDN_URL` (~$20/month saved, lower latency).
- ✅ **Liquibase pre-deploy task** — Migrations run as isolated ECS task before API starts; race condition on multi-task deployments eliminated.
- ✅ **Liquibase migrations 021–022** — `trial_expires_at`, `stripe_customer_id`, `stripe_subscription_id` columns with indexes; backfill for pre-migration users.
