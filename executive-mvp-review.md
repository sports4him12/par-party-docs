# GolfSync — Executive MVP Review
**Prepared:** 2026-04-06 (CFO/COO/CMO) · Updated 2026-04-07 (CTO) · Updated 2026-04-09 (full refresh)
**Perspective:** CFO · COO · CMO · CTO
**Purpose:** Identify what to change, cut, or double down on before scaling

> **Legend:** ✅ Done · 🔶 Partial · ⬜ Open

---

## Summary Assessment

| Dimension | Rating | Notes |
|---|---|---|
| **Technical foundation** | ★★★★☆ | Security hardening, CORS fixes, webhook infra, legal compliance all solid |
| **Product-market fit** | ★★★★☆ | Core social loop complete; tee-time booking still mocked |
| **Monetization** | ★★★☆☆ | Infrastructure ready; billing deliberately paused until June 1, 2026 launch period |
| **Legal & compliance** | ★★★★☆ | Terms, Privacy, Cookie consent, CAN-SPAM opt-out, GolfNow non-affiliation all addressed |
| **Growth engine** | ★★★☆☆ | Trial drip + invite links live; referral and tournaments hidden pending launch |
| **Operations readiness** | ★★★★☆ | ~$150–190/month compute savings shipped; CI/CD still manual |
| **MVP scope fit** | ★★★★☆ | Conversion funnel built; launch trial period (5/31) is a deliberate business decision |

**Bottom line:** The platform is functionally complete for an invite-based soft launch. The deliberate 5/31 free trial decision de-risks early user acquisition — everyone gets full access with no payment friction through May 31, 2026. The most critical remaining gap is a real tee-time data source (GolfNow integration or pivot). Legal compliance (Terms, Privacy, Cookie consent, email opt-out, GolfNow disclaimer) has been addressed. Onboarding is live on web and mobile. Referral and Tournament features are built but held behind "Coming this Summer" teasers pending polish and national data coverage.

---

## Business Context — Launch Trial Decision

**Decision (2026-04-09):** All features are free through May 31, 2026. No credit card is collected until June 1st.

**Rationale:**
- Removes all payment friction during the critical early-user acquisition window
- Allows word-of-mouth growth before billing creates churn risk
- Gives time to negotiate a real GolfNow API deal or pivot the positioning
- Stripe infrastructure is live — flipping the switch on June 1 requires only confirming live keys and setting `STRIPE_ENABLED=true`

**Implication:** Monthly/Annual plan cards are shown on the landing page and membership page with accurate, identical feature lists. Pricing is honest; no fake plan differentiation exists.

---

## Priority Action Plan — Open Items

### Immediate (before June 1, 2026)

| # | Action | Owner |
|---|---|---|
| 1 | **Activate billing** — confirm Stripe live keys in Secrets Manager, set `STRIPE_ENABLED=true` in prod, register webhook endpoint in Stripe dashboard, send pre-billing email to all users | CFO/CTO |
| 2 | **Decide GolfNow integration or reposition** — negotiate GolfNow/Supreme Golf/TeeOff API, or rebrand as "round management" | COO |

### Near-Term (post-June 1)

| # | Action | Owner |
|---|---|---|
| 3 | **Launch Tournament Discovery** — restore feature when national data pipeline is ready | COO/CMO |
| 4 | **Launch Refer a Friend** — restore dashboard widget and landing page CTA | CMO |
| 5 | **Course leaderboard** — top scores per course | CMO |
| 6 | **Shareable scorecard** — PNG export for Instagram/X | CMO |
| 7 | **Seasonal pricing evaluation** — $29.99/6 months, per-event $4.99, family/group plan | CFO |

### Medium-Term (90–180 days)

| # | Action | Owner |
|---|---|---|
| 11 | Tournament entry fee commission (5–10%) | CFO |
| 12 | Handicap calculation from round scores | CMO |
| 13 | Web push notifications | CMO |
| 14 | PWA with home screen install | CTO |
| 15 | Seasonal pricing ($29.99 for 6 months) | CFO |
| 16 | B2B pilot: sell to 2–3 golf clubs | CFO |

---

## CFO — Status

### ✅ Launch Trial Period (through May 31, 2026)
All users receive full access at no charge through May 31. Stripe subscription infrastructure is ready to activate. Membership page clearly communicates the June 1 billing start date with a green "Free access through May 31" banner.

### 🔶 Subscription Billing — Ready, Not Yet Active
- Stripe Checkout Session (subscription mode) — implemented ✅
- Webhook handlers for all lifecycle events — implemented ✅
- Live keys not yet in Secrets Manager — needed before June 1 ⬜
- Full dunning sequence (day 3, day 7 follow-ups) — not yet implemented ⬜
- Renewal reminder email (7 days before expiry) — not yet implemented ⬜

### Revenue Levers

| Lever | Status |
|---|---|
| Monthly/Annual subscriptions | 🔶 Infrastructure ready; billing paused until June 1 |
| Tournament entry fee commission | ⬜ Open |
| GolfNow referral/affiliate | ⬜ Blocked (integration still mocked) |
| Premium tier | ⬜ Open |
| B2B white-label | ⬜ Open |

### Pricing Alternatives Not Yet Evaluated
- **Seasonal plan**: $29.99 for 6 months (April–September)
- **Per-event pricing**: $4.99 to unlock for a single round
- **Family/group plan**: one subscription, up to 4 golfers

---

## COO — Status

### ⬜ BLOCKER: GolfNow Integration is Mocked
Tee-time data is still from `GolfNowClientMock`. Exit interstitial and non-affiliation disclaimer are now in place (legal exposure addressed), but the core booking feature is not real.

**Decision needed:** (a) negotiate GolfNow/Supreme Golf/TeeOff API partnership, (b) pivot to "round management" platform and remove tee-time promise.

### ✅ Legal & Compliance (2026-04-09)
- Terms of Service page (`/terms`) — 13 sections, non-endorsement clause for third parties
- Privacy & Cookie Policy page (`/privacy`) — GDPR rights, cookie table, opt-out instructions
- Cookie consent banner — `localStorage` key, essential-only cookies stated, links to Privacy page
- Email marketing opt-out — toggle on `/account`; PATCH `/api/users/me/email-preferences`; drip guard; CAN-SPAM footer on all marketing emails
- GolfNow exit interstitial — modal with explicit non-affiliation statement before navigation
- Global footer with legal links on every page

### ✅ GDPR Self-Deletion
DELETE `/api/users/me` cancels Stripe subscription, cascade-deletes all user data, clears auth cookie. `/account` page requires typed confirmation.

### 🔶 Tournament Database — No National Pipeline
Live discovery via Google CSE supplements east-coast seed data. No structured national pipeline. Tournament feature hidden pending this gap.

### 🔶 Observability Gaps
CloudWatch structured logging and error alarms in place. Still missing:
- Business metrics dashboard (signups/day, trial → paid conversion, round creation rate)
- RDS Performance Insights
- Distributed tracing / correlation IDs
- Route53 health checks and synthetic monitoring

### ⬜ Operational Runbook Gaps
- Canary deployment strategy (ECS blue/green)
- Database backup restore test
- Incident response playbook
- On-call rotation beyond a single SNS email

---

## CMO — Status

### 🔶 Activation Funnel

| Stage | Current State |
|---|---|
| **Awareness** | ⬜ No marketing infrastructure |
| **Activation** | 🔶 Full trial access from day one; no guided onboarding flow |
| **Retention** | 🔶 Availability polls, drip emails, leaderboard active |
| **Referral** | 🔶 Built but hidden — "Coming this Summer" |

**Highest-priority next investment:** 3-step onboarding flow (find home course → invite a friend → create first round).

### 🔶 Referral Program — Hidden
Referral codes, invite links, credit logic, and dashboard widget are all built and tested. Hidden behind "Coming this Summer" teaser on the landing page pending full launch readiness. Restore by uncommenting the dashboard widget and adding `/referral` to the landing page CTA.

### 🔶 Tournament Discovery — Hidden
Feature is built; displayed as a "Coming this Summer" card on the landing page. Blocked by national data pipeline gap. Restore when data coverage is sufficient.

### ⬜ Viral / "Aha" Moment
Shareable scorecard (PNG for Instagram/X) is the highest-potential viral feature. Friend leaderboard is live but internal.

### ✅ Social Loop Complete
- Friend network, round invites, group messaging, availability polls all live
- Friend leaderboard by rounds played
- Trial drip sequence (day 0, 3, 7, 21, 27)
- Shareable round invite links for non-members

---

## CTO — Status

### ⬜ No CI/CD Pipeline
Deployments are still manual `cdk deploy` + `docker compose build`. Every release requires developer intervention.

### ✅ Security & Auth
- JWT in httpOnly cookie (XSS hardened)
- CORS PATCH method added (was causing 403 on round edits)
- Rate limiting: auth (10/min/IP), AI (5/min/user), search (30/min/IP), tee-times (10/min/IP)
- IDOR on change-password fixed; email removed from user search responses

### ✅ Test Coverage (as of 2026-04-09)
- ≥80% unit test coverage maintained (JaCoCo enforced)
- All controller, service, and API-layer changes have corresponding unit tests
- Cypress E2E covers: auth, rounds, dashboard, account (including email opt-out toggle), referral widget hidden state

### ✅ Infrastructure Optimisations (savings vs. original)
| Service | Before | After | Saving |
|---|---|---|---|
| Camunda ECS service | ~$70–105/mo | $0 (in-process) | ~$70–105/mo |
| Dev Fargate → EC2 t3.small | ~$35/mo | ~$15/mo | ~$20/mo |
| Dev NAT Gateway → VPC Endpoints | ~$30–35/mo | $0 | ~$30–35/mo |
| CloudFront + S3 (prod static) | ~$20/mo | ~$3–5/mo | ~$15–17/mo |
| **Total monthly saving** | | | **~$135–177/mo** |

### ⬜ Single-Region Deployment
No multi-region or failover. An outage during peak booking hours (Friday morning) has no recovery path.

---

## Completed Items

| # | Action | Completed |
|---|---|---|
| — | GolfNow exit interstitial + non-affiliation disclaimers | ✅ 2026-04-09 |
| — | Legal compliance — Terms, Privacy, Cookie consent, CAN-SPAM email opt-out | ✅ 2026-04-09 |
| — | Launch trial period decision — free through 5/31, membership required from June 1 | ✅ 2026-04-09 |
| — | CDK renamed ParParty → GolfSync (all env vars, file names, stack names) | ✅ 2026-04-09 |
| — | CORS PATCH fix — round edit 403 resolved | ✅ 2026-04-09 |
| — | Round detail — inline edit, friends autocomplete, dues in dollars | ✅ 2026-04-09 |
| — | Dashboard AI navigation assistant | ✅ 2026-04-09 |
| — | Conversion funnel instrumentation + drip sequence | ✅ 2026-04-07 |
| — | Referral mechanism — codes, invite links, leaderboard, 30-day credit | ✅ 2026-04-07 |
| — | Stripe Checkout Session (subscription mode) | ✅ 2026-04-07 |
| — | Rate limiting — AI, search, tee-times | ✅ 2026-04-07 |
| — | Redis-backed AI conversation memory + ElastiCache CDK | ✅ 2026-04-07 |
| — | Shareable round invite links | ✅ 2026-04-07 |
| — | GDPR self-deletion | ✅ 2026-04-07 |
| — | Analytics dashboard (retention, churn, LTV) | ✅ 2026-04-07 |
| — | Free tier → 30-day free trial; full access from signup | ✅ 2026-04-07 |
| — | Security hardening — JWT httpOnly, IDOR fix, PublicUserResponse | ✅ 2026-04-06 |
| — | CDK dev/prod split (Gmail SMTP dev, SES prod, Route53 prod-only) | ✅ 2026-04-06 |
