# GolfSync — Requirements & To-Do

Living backlog of features, improvements, and fixes. Add items here and they'll be picked up and addressed in future sessions.

**Status legend:** `[ ]` open · `[~]` in progress · `[x]` done

---

## Features

<!-- New user-facing functionality to build -->


- [x] **Membership selection during registration** — Free tier removed; replaced with a 30-day free trial (full access, no credit card). Plan selector removed from registration. Users subscribe Monthly or Annual after trial via /membership.

- [x] **Admin-promoted courses and tournaments** — Admins can flag specific courses or tournaments as "featured." Featured items appear as highlighted cards at the top of the relevant section on the user's dashboard, filtered to the user's current home location. Courses and tournaments outside the user's area should not be surfaced.

- [ ] **GolfNow cancellation handling** — Implement round cancellation according to GolfNow API requirements. Research GolfNow's cancellation policies (window, fees, confirmation flow) and mirror that logic in the `cancelRound` flow. Update the BPMN process and UI accordingly so users understand what happens when a booked tee time is cancelled.

---

## Improvements

<!-- Enhancements to existing features, UX polish, performance -->

- [ ] 

---

## Security & Infrastructure

<!-- Auth, secrets, AWS, hardening, compliance -->

- [ ] 

---

## Bugs

<!-- Known defects to fix -->

- [ ] 

---

## Tech Debt

<!-- Refactors, test gaps, dependency updates, cleanup -->

- [ ] **Migrate from Camunda 7 to FluxNova** — Camunda 7 reaches end of life; FluxNova (`org.finos.fluxnova.bpm.springboot`) is the target BPM runtime for long-term support. Replace the `org.camunda.bpm.springboot` starters, update BPMN process definitions as needed, and verify the `golf-round-booking-process` and `tournament-report-process` workflows behave identically. Note: FluxNova starters are not on Maven Central — confirm artifact repository before starting.

---

## Business / Product

<!-- Pricing, partnerships, analytics, marketing, legal -->

- [ ] 

---

## Completed

<!-- Move items here (with date) when done -->

| Date | Item |
|------|------|
| 2026-04-06 | JWT moved from localStorage to httpOnly cookie (XSS fix) |
| 2026-04-06 | ALB HTTPS via ACM — CDK conditional TLS listener on 443 |
| 2026-04-06 | CDK dev/prod environment split (Gmail SMTP dev, SES prod, Route53 prod-only) |
| 2026-04-06 | PublicUserResponse DTO — email no longer exposed in user search results |
| 2026-04-06 | Change-password IDOR fix — authenticated callers can only change their own password |
| 2026-04-06 | AI booking agent no longer leaks user emails in friend search results |
| 2026-04-06 | Rate limiting on auth endpoints (10 req/min per IP, sliding window) |
| 2026-04-06 | User search minimum 2-character validation |
| 2026-04-06 | Admin-promoted courses and tournaments with haversine location filtering |
| 2026-04-06 | Tournament type interest preferences (Scramble, Amateur, Charity, etc.) with localStorage persistence and preferred-first sorting |
| 2026-04-06 | Live tournament discovery via Google Custom Search API with SGA site: queries, city/state forwarding, and 2-hour cache |
| 2026-04-06 | Tournament self-registration ("I'm Going") with friend grouping — users mark themselves registered and add friends to the record; panel shows all upcoming registrations |
| 2026-04-06 | Free tier by default — new accounts limited to tee time booking; friends/messaging/round invitations gated behind paid membership; upgrade banner on dashboard and full-page gates on friends/messages pages |
| 2026-04-06 | Round scores — players can post gross stroke scores to booked rounds; friend score feed (paid feature) on dashboard |
| 2026-04-06 | Booker/Payment settlement — round creator identified as Booker; per-player dues (cents) and PAID/WAIVED status tracked for both rounds and tournament registrations; booker controls in round detail UI |
| 2026-04-07 | Availability Polls — booker creates date/time option polls, invites friends, friends vote I'm In/Can't Make It; dashboard widget shows active polls with "Vote needed" badge; new Liquibase migration 020 |
| 2026-04-07 | Add player post-creation — booker can invite additional players to an existing round from the Round detail page |
| 2026-04-07 | Nudge pending invitees — booker sends in-app reminder message to players who haven't responded (Remind button per INVITED player) |
| 2026-04-07 | Safe tournament registration URL resolver — GET /api/tournaments/register-url resolves top organic Google CSE result with adult-content safety filter; fallback to Google Search when blocked |
| 2026-04-07 | How To Guide restructured into role-based hub with sub-pages: Booker, Participant, Tournaments, Account |
| 2026-04-07 | CTO AWS assessment added to executive-mvp-review.md — spend breakdown, Fargate cost analysis, alternatives (Lambda, EC2, App Runner), three immediate cost-reduction recommendations |
| 2026-04-07 | Free tier replaced with 30-day free trial — all features unlocked on signup, trial countdown on dashboard, TRIAL_EXPIRED 403 redirects to /membership; Stripe subscription webhooks (checkout.session.completed, invoice.paid, invoice.payment_failed, customer.subscription.updated, customer.subscription.deleted) with HMAC signature verification; payment-failed and cancellation emails |
| 2026-04-07 | CTO infrastructure changes — (1) Camunda 7 embedded in-process via Spring Boot starter, removing separate Camunda ECS service (~$70-105/month savings); (2) EC2 t3.small replaces Fargate for dev ECS cluster (~$50/month savings); (3) VPC Interface Endpoints (ECR API/DKR, Secrets Manager, CloudWatch Logs) + S3 Gateway Endpoint replace NAT Gateway for dev (~$30-35/month savings); (4) CloudFront distribution with S3 origin for /_next/static/* offloads Next.js static assets from Fargate (prod only); (5) Liquibase runs as isolated pre-deploy ECS task (not at API startup), fixing startup race condition |
