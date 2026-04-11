# GolfSync — Requirements & To-Do

Living backlog of features, improvements, and fixes. Add items here and they'll be picked up and addressed in future sessions.

**Status legend:** `[ ]` open · `[~]` in progress · `[x]` done

---

## Features

<!-- New user-facing functionality to build -->

- [ ] **GolfNow cancellation handling** — Implement round cancellation according to GolfNow API requirements. Research GolfNow's cancellation policies (window, fees, confirmation flow) and mirror that logic in the `cancelRound` flow. Update the BPMN process and UI accordingly so users understand what happens when a booked tee time is cancelled.

- [x] **3-step onboarding flow** — After registration, guide new users through: (1) find/set home course, (2) invite a friend, (3) create first round. Currently users land on the dashboard with no guidance. This is the highest-leverage activation improvement available.

- [ ] **Tournament Discovery — live feature** — Tournament Discovery is currently hidden ("Coming this Summer" teaser on landing page). When ready to launch: set `TOURNAMENTS_ENABLED=true`, restore the feature card on the landing page, and ensure the tournament database has national coverage beyond the current east-coast seed data.

- [ ] **Refer a Friend — live feature** — Referral program is currently hidden ("Coming this Summer" teaser on landing page). The backend (referral codes, credit logic) is implemented. When ready: restore the dashboard widget, add referral code to account settings, and wire the landing page CTA.

- [ ] **Direct Tee Time Booking — live feature** — Direct booking is listed as "Coming this Summer" on the landing page. Requires a course partner API or proprietary booking layer that removes the GolfNow redirect. When ready: integrate booking API, surface the booking flow within the round creation UX, and remove the teaser card from the landing page.

- [ ] **Membership billing activation** — Stripe is integrated but billing does not begin until June 1, 2026. Before that date: confirm Stripe live keys are populated in Secrets Manager, switch `STRIPE_ENABLED=true` in prod, verify webhook endpoint is registered in the Stripe dashboard, and send a pre-billing heads-up email to all users. Full dunning sequence (day 3 and day 7 follow-up after failed payment) is not yet implemented.

---

## Improvements

<!-- Enhancements to existing features, UX polish, performance -->

- [ ] **Shareable scorecard** — PNG export of a round scorecard for sharing on Instagram/X. Highest-impact viral moment candidates.

- [ ] **Course leaderboard** — Top scores per course (complements the existing friend leaderboard by rounds played).

- [ ] **Web push notifications** — Alert players to round invites, poll votes, and message activity without requiring them to open the app.

- [ ] **PWA / home screen install** — `manifest.json` + service worker so users can install GolfSync on their phone home screen.

- [x] **Full dunning email sequence** — Currently only the initial payment-failed email is sent. Add day-3 and day-7 follow-up emails before membership is downgraded. Also add a renewal reminder email 7 days before subscription expiry.

- [x] **How-to guide and FAQ updates for legal/compliance pages** — Update `app/how-to/account/page.tsx` and the FAQ in `app/support/page.tsx` to reflect: (a) the 5/31 launch trial period, (b) the email opt-out toggle in account settings, (c) the Terms / Privacy / Cookie pages.

---

## Security & Infrastructure

<!-- Auth, secrets, AWS, hardening, compliance -->

- [ ] **MFA for sensitive operations** — Add multi-factor authentication challenges before high-risk actions: Stripe billing management, account deletion, and admin-panel access. Evaluate TOTP (Google Authenticator / Authy) vs. email OTP. Backend needs an `mfa_secret` column, a verify-code endpoint, and a step-up token pattern. Frontend needs a modal challenge before the sensitive action proceeds.

- [x] **Support email — permanent address** — `support@golfsync.io` is now set across `application.properties`, `EmailService.java`, all web pages (terms, privacy, account, support), and CDK config (`bin/golfsync-cdk.ts`).

- [ ] **CDK TODOs — Route53 hosted zone and ACM certificate** — `golfsync-cdk.ts` has two outstanding values in the prod stack that must be populated before prod HTTPS works: (1) `GOLFSYNC_HOSTED_ZONE_ID` — find via `aws route53 list-hosted-zones`; (2) `GOLFSYNC_CERT_ARN` — request via ACM in us-east-1. Document the values in `AWS_DeploymentGuide.md` once confirmed.

- [ ] **CDK alarm email env vars** — `GOLFSYNC_DEV_ALARM_EMAIL` and `GOLFSYNC_PROD_ALARM_EMAIL` must be set before deploying either stack. Add to `.env.prod.example` with instructions.

- [ ] **GolfNow API partnership or pivot** — GolfNow integration is currently mocked (`GolfNowClientMock`). Exit interstitial and non-affiliation disclaimers are in place. Decision needed: (a) negotiate GolfNow/Supreme Golf/TeeOff API partnership, or (b) reposition as "round management" platform and remove tee-time booking. Blocking real marketing.

---

## Bugs

<!-- Known defects to fix -->

- [ ]

---

## Tech Debt

<!-- Refactors, test gaps, dependency updates, cleanup -->

- [ ] **Migrate from Camunda 7 to FluxNova** — Camunda 7 reaches end of life; FluxNova (`org.finos.fluxnova.bpm.springboot`) is the target BPM runtime. Replace starters, update BPMN definitions, verify `golf-round-booking-process` and `tournament-report-process` workflows behave identically. Note: FluxNova starters are not on Maven Central — confirm artifact repository before starting.

---

## Business / Product

<!-- Pricing, partnerships, analytics, marketing, legal -->

- [ ] **Seasonal pricing evaluation** — Evaluate $29.99 for 6 months (April–September), per-event pricing ($4.99 for one round), and family/group plan (one subscription, up to 4 golfers) before June 1 billing launch.

- [ ] **B2B pilot** — Approach 2–3 local golf clubs about a white-label or club-management pilot. Existing admin panel and round management tooling provides a foundation.

- [ ] **Tournament entry fee commission** — 5–10% commission on tournament registrations processed through the platform. Requires real tournament registration API integration.

---

## Completed

<!-- Move items here (with date) when done -->

| Date | Item |
|------|------|
| 2026-04-11 | Friend search now matches on name field in addition to username — "Sutton" finds users whose display name is Sutton even if their username differs |
| 2026-04-11 | Friend request pending badge — clicking "Add Friend" on the Friends page now shows a "✓ Pending" badge inline instead of removing the user from results |
| 2026-04-11 | Friends Recent Scores "Unknown Course" bug fixed — score feed now falls back to courseNameText when no linked Course entity exists |
| 2026-04-11 | Friends Recent Scores cards made non-interactive — removed link wrapper; non-participants get 403 on round detail, so cards now display score info only |
| 2026-04-11 | Onboarding reduced to 2 steps — Home Course step hidden (not yet relevant); flow is now: Invite a Friend → Book Your First Round |
| 2026-04-11 | AI Booking Assistant date bug fixed — added getCurrentDate() tool so agent computes relative dates correctly instead of guessing |
| 2026-04-11 | www.golfsync.io CORS fix — ALB redirect rule added (priority 5) to redirect www → apex before CORS filter runs |
| 2026-04-11 | Full dunning email sequence — day-3 and day-7 follow-up emails after failed payment (DunningService, bitmask pattern); renewal reminder 7 days before expiry; MembershipService idempotency guard; DunningServiceTest (9 tests) |
| 2026-04-09 | 3-step onboarding wizard — `/onboarding` page (find home course → invite friend → create first round, each skippable); registration redirects to `/onboarding`; `POST /api/users/me/complete-onboarding`; `onboarding_completed` column (migration 037) |
| 2026-04-09 | Feedback section — `user_feedback` table (migration 038); `POST /api/feedback` (upsert per user), `GET /api/feedback/me`, `GET /api/feedback` (admin); feedback widget on account page; Feedback tab in admin portal |
| 2026-04-09 | Booker auto-accepted on round creation — creator now added as ACCEPTED RoundPlayer when creating a round |
| 2026-04-09 | "Golf Sync" branding — renamed GolfSync → Golf Sync across all TSX/TS files (29 files) including metadata title/description |
| 2026-04-09 | Accept Invite moved to top of round detail page — prominent navy banner below navbar when status === INVITED |
| 2026-04-09 | Friends Recent Scores moved to left sidebar — now appears under Friend Leaderboard card on dashboard |
| 2026-04-09 | Handicap editing on account page — number input with Update button calling PUT /api/users/me |
| 2026-04-09 | Booking Test #2 bug — root cause was Recent Rounds capped at 5; increased to 10 and created /rounds listing page with status filter tabs |
| 2026-04-09 | Friend autocomplete on new round form — tag-pill system with friend suggestions on the invitees field when attesting a booking |
| 2026-04-09 | How-to guide + FAQ updated — launch trial period (5/31), email opt-out, Terms/Privacy/Cookie links |
| 2026-04-09 | GolfNow exit interstitial + non-affiliation disclaimers — `ExternalLinkModal` shown before opening GolfNow; inline disclaimer on booking page; Terms of Service non-endorsement clause strengthened |
| 2026-04-09 | "Coming this Summer" teaser on landing page — Tournament Discovery and Refer a Friend shown as dashed-border coming-soon cards in the features section |
| 2026-04-09 | Membership page messaging updated — green launch-period banner; both plans show identical feature lists; no fake plan differentiation; "no charge until June 1, 2026" messaging throughout |
| 2026-04-09 | Landing page pricing section rewritten — removed fake Member-only features (Priority AI, Advanced management); Monthly and Annual cards with accurate, identical feature lists |
| 2026-04-09 | Contact Us → support form — Footer "Contact Us" now links to `/support#contact`; `id="contact"` anchor added to support page form |
| 2026-04-09 | Support email updated to ryanrpick@gmail.com — across application.properties, EmailService, all web pages (support, account, terms, privacy) |
| 2026-04-09 | Email marketing opt-out — `email_marketing_opt_out` column (migration 035); PATCH `/api/users/me/email-preferences`; toggle on account settings page; drip email guard; CAN-SPAM unsubscribe footer on all marketing emails |
| 2026-04-09 | Legal compliance pages — Terms of Service (`/terms`), Privacy & Cookie Policy (`/privacy`), global Footer with legal links, cookie consent banner (localStorage, essential cookies only) |
| 2026-04-09 | Dashboard: referral widget hidden (Coming this Summer); AI navigation assistant deployed |
| 2026-04-09 | Round detail page — inline edit form (max players, date, time, course, notes); friends autocomplete dropdown on invite input; dues input in dollars; "Completed Booking" status label |
| 2026-04-09 | CORS PATCH fix — added `PATCH` to `WebConfig.java` allowed methods; was causing 403 on round edits |
| 2026-04-09 | CDK renamed from ParParty → GolfSync — env vars, file names, stack class names updated throughout `golfsync-cdk/` |
| 2026-04-09 | Test coverage — TrialDripServiceTest (opt-out guard), UserControllerTest (PATCH email-preferences), UserServiceTest (setEmailMarketingOptOut), api.test.ts (updateEmailPreferences, deleteMyAccount), account.cy.ts (email toggle E2E), dashboard.cy.ts (referral widget hidden) |
| 2026-04-08 | Landing page redesign — full feature showcase, pricing section, Member card |
| 2026-04-08 | CuratedTournamentRefreshService unit test coverage — 95.2% instruction coverage |
| 2026-04-08 | Availability poll sharing — all accepted friends pre-selected; Select All / Deselect All |
| 2026-04-08 | Nullable entry fee and format "Unavailable" sentinel (migration 033) |
| 2026-04-08 | Tournament location radius filter — only curated tournaments with real coordinates |
| 2026-04-08 | Tournament date corrections and page scraping |
| 2026-04-08 | Signup → paid conversion funnel (CFO item #5) |
| 2026-04-07 | Analytics dashboard — AnalyticsService; GET /api/admin/analytics |
| 2026-04-07 | GDPR self-deletion — DELETE /api/users/me; /account page |
| 2026-04-07 | Shareable round invite links (migration 029) |
| 2026-04-07 | AI chat error handling + conversationId |
| 2026-04-07 | Redis-backed AI conversation memory + ElastiCache CDK |
| 2026-04-07 | Extended rate limiting — AI (5/min/user), search (30/min/IP), tee-times (10/min/IP) |
| 2026-04-07 | Stripe Checkout Session subscription mode |
| 2026-04-07 | Friend leaderboard — /leaderboard |
| 2026-04-07 | Round invite for non-members — UUID token; /invite/[token] |
| 2026-04-07 | Referral program — unique 8-char code; 30-day credit on conversion |
| 2026-04-07 | Trial drip email sequence — 5-step, bitmask per user |
| 2026-04-07 | Conversion funnel instrumentation — user_events table |
| 2026-04-07 | CTO infrastructure — Camunda in-process; EC2 t3.small dev; VPC endpoints; CloudFront; Liquibase pre-deploy |
| 2026-04-07 | Free tier → 30-day free trial |
| 2026-04-07 | How To Guide restructured (Booker, Participant, Tournaments, Account) |
| 2026-04-07 | Availability Polls (migration 020) |
| 2026-04-07 | Booker/Payment settlement — per-player dues, PAID/WAIVED |
| 2026-04-07 | Round scores — post gross scores; friend score feed |
| 2026-04-07 | Tournament self-registration |
| 2026-04-07 | Live tournament discovery via Google CSE |
| 2026-04-06 | Admin-promoted courses and tournaments |
| 2026-04-06 | Rate limiting on auth endpoints |
| 2026-04-06 | Security fixes — JWT httpOnly cookie, IDOR, PublicUserResponse, email leak |
| 2026-04-06 | CDK dev/prod split (Gmail SMTP dev, SES prod, Route53 prod-only) |
