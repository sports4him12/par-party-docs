# Par Party — Requirements & To-Do

Living backlog of features, improvements, and fixes. Add items here and they'll be picked up and addressed in future sessions.

**Status legend:** `[ ]` open · `[~]` in progress · `[x]` done

---

## Features

<!-- New user-facing functionality to build -->

- [ ] **Free tier by default** — New accounts are free and limited to tee time booking only. Tournament discovery and all friend/social features (friend requests, messaging, round invitations) are gated behind a paid membership. The free experience should make the value of upgrading obvious without being hostile.

- [ ] **Membership selection during registration** — Integrate plan selection (Free / Monthly / Annual) directly into the account creation flow with Stripe payment inline. Users should not have to navigate to a separate `/membership` page after registering; the choice and payment (if upgrading) happen as part of sign-up.

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
