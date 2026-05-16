# Nonprofit Charity Golf Microsite — Pattern Research

**Date:** 2026-05-16
**Sources analyzed (7 sites):**
- [Halliburton Charity Golf](https://www.halliburtoncharitygolf.org/) — 25-year, $35M+ raised
- [AGAT Foundation Charity Classic](https://charityclassic.agatfoundation.com/) — 44-year, $13.9M raised
- [First Tee Hampton Roads](https://firstteehamptonroads.org/annual-golf-tournament/)
- [Bright Horizons Foundation](https://brighthorizonsfoundation.org/ways-to-give/bhgolftournament)
- [Veterans Inc. Best Ball Classic](https://www.veteransinc.org/events/best-ball-charity-golf-classic/) — 29-year
- [First Tee Howard County](https://firstteehowardcounty.org/tournament/) — 25-year
- [The Georgia Club Foundation](https://thegeorgiaclubfoundation.com/charity-golf-sponsorship/)

**Audience:** golfsync founder, designing the C2 Adopt 2026 microsite (first of an expected 20 hosted tournaments).

---

## A. Table-stakes patterns (every site does this)

These show up on virtually every charity golf microsite. If C2 Adopt's microsite is missing them, it'll feel sub-par.

- **Date front-and-center.** Hero leads with date (often ahead of even the event name). Format varies: "Save the Date: October 22, 2025" / "Monday, October 27, 2025" / "August 20, 2026." Not buried.
- **Venue named in hero.** Course name + city. Even when no map/photo is shown.
- **"Benefiting [mission]" line.** Single sentence in the hero linking the dollar to the cause. Halliburton: "Since 1993… raised more than $35 million." Veterans Inc: "Your support helps Veterans and their families access critical programs."
- **501(c)(3) tax-deductibility callout.** Every site mentions it, usually in the footer or near the donate CTA. Required for sponsor finance teams.
- **Past sponsor logo wall.** Every site featured 4–100+ named past sponsors as social proof. Even Veterans Inc's "Thank you to our 2026 Major Sponsors!" with 30+ logos.
- **Event-day timeline.** Registration time → shotgun start → reception/awards. Brief but present (Veterans Inc: "7:30 AM registration, 9:00 AM shotgun, 3:00 PM reception"). Bright Horizons: meal schedule called out.
- **Contact person.** Name + email + phone for sponsorship inquiries. Always at least one human, usually two for redundancy.
- **Multiple sponsor tiers** (vs. one flat ask). Range from 4 (First Tee Howard County) to 12+ (Halliburton). Common ladder shape: 3-4× from top to next-down.
- **What's included with each registration.** Foursome details: lunch, beverages, putting contest, dinner, giveaways. Always itemized.

## B. WOW-factor moves (rare but memorable)

Patterns that appeared on only 1-2 sites but felt high-leverage. Each gets an effort estimate against the GolfSync stack.

- **Cumulative dollar raised in hero** (AGAT: $13,953,697; Halliburton: $35M+). It's the single most credibility-building number on the page. Forces the visitor to take the event seriously. **Effort: trivial** — admin entry field on the tournament, render in hero.
- **Live countdown timer** (AGAT: Days/Hours/Minutes/Seconds). Adds urgency without being tacky. **Effort: ~1hr** — pure client-side calc from the tournament_date field.
- **Anniversary milestone branding** ("25th Annual" / "29th Annual Best Ball" / "44 years of history"). Telegraphs "this is real, not a one-off." **Effort: trivial** — `years_running` field on tournament, render as "Nth Annual" badge.
- **Specific impact stat tied to dollar raised** (Bright Horizons: "Sponsors create entirely new Bright Space™" — donor sees the THING their dollar built). Pure-play intangible "supports our mission" loses to a tangible artifact. **Effort: ~30 min** — admin tagline / impact-statement field on each tier; we already have `description`.
- **Capacity / scarcity messaging** (Bright Horizons: "Close to reaching maximum capacity… spots often become available — join waitlist"). The only site that did this. Honest scarcity is more persuasive than urgency timers. **Effort: ~2hr** — "X of N spots remaining" computed from registrations vs. flight_capacity, shown on registration card. We already have the data.
- **Niche / experiential sponsor tiers** that aren't just dollar amounts (First Tee Howard County: "Beverage Cart Sponsorship — cart branding"). Small-dollar tiers ($250-$500) that feel personal vs. transactional. **Effort: free** — C2 Adopt already has this (Goodies, Cigar, Koozie, Putting, Meal sponsors).
- **Past-year photo gallery** (Veterans Inc: "Photos from 2025 event"). Proves the event actually happens and looks fun. **Effort: ~3hr** — admin uploads a small batch of images, microsite renders a horizontal scroll/grid.
- **Participant testimonial quote** (First Tee Howard County: parent quote with name attribution). Quotes humanize the impact. **Effort: ~1hr** — admin field for 1-3 testimonials, microsite renders as cards.
- **Bifurcated giving paths** (Bright Horizons separates "golf sponsorships" from "non-golf sponsorships"). Acknowledges some donors don't want to play. **Effort: free** — already supported by our 0-ticket "Hole Sponsor" tier pattern.
- **Hospitality tent / brand activation as separate purchase** (Halliburton). Premium add-on for sponsors who want physical presence. **Effort: future PR** — would need an "add-ons" model on top of tiers.

---

## C. Concrete C2 Adopt microsite recommendations

Ranked by impact-to-effort. Anything ✓ is already on the GolfSync stack today; the rest go into PR 3 / PR 4 / future work.

### Top 5 wins for C2 Adopt launch

1. **"Nth Annual" + "Benefiting Children with Forever Families" line in hero.** Anniversary milestone + mission-link in one band. Trivial. *Already partial — we have `tagline`. Add a `yearsRunning` admin field.*
2. **Cumulative-raised counter** (e.g. "$XXX raised since YYYY"). Single most credibility-building number per the research. Add `cumulativeRaisedCents` + `inceptionYear` to tournament admin. ~1hr. **Highest leverage move I'd make.**
3. **Live event-day timeline** strip ("8:00 AM Check-in · 9:00 AM Shotgun · 3:00 PM Awards"). Becky already gave us "8:00 AM" / "1:30 PM" placeholders. Render them as a polished horizontal strip on the microsite. ~30 min, free.
4. **Past sponsor logo wall** below the tier picker. We're already collecting sponsor applications + capturing `companyName`; render the past-paid roster with company names (logos when uploaded). ~2hr, huge credibility boost.
5. **"X spots remaining" badges** on registration tier cards (we have flight_capacity + registered count via FlightSummary). Honest scarcity. ~1hr.

### Should-haves for PR 3 (microsite tier rendering)

6. **Per-tier "what you get" expandable benefits.** Several sites used bulleted benefit lists; ours has free-form description. Switch to bulleted parsing of `description` (split on `|` or newlines) for visual hierarchy. ~1hr.
7. **Group tiers by flight visually** (we already have `flightLabel` from PR 1). Add a flight-tab toggle on the public microsite OR section headers. **This was already in PR 3's plan.** ~2hr.
8. **Ticket-bundle badge** on each tier ("Foursome included" / "Signage only"). Maps directly to `included_tickets`. ~30 min.

### Wow-factor moves for PR 4 (or near-term follow-ups)

9. **Past-year photo gallery** — admin uploads 4-12 images, microsite renders horizontal scroll above the registration section. ~3hr.
10. **Sponsor testimonial cards** — 2-3 quoted testimonials with name + company. Admin field. ~1hr per quote.
11. **Countdown timer** — to tournament date. Drops once you cross zero. ~1hr.
12. **Tax-deductibility footer block** — 501(c)(3) status, EIN, "all donations tax-deductible." Admin fields, microsite renders. **Required for big-sponsor finance teams.** ~30 min.
13. **"Bright Space-style" tangible-impact bullets per tier.** Becky might say "Title Sponsor funds the C2 Adopt Family Liaison program for one year." Admin field, renders inside the tier card hover. ~30 min plus copy from Becky.

### Things I'd skip (low ROI for first hosted customer)

- Hospitality tent add-ons (Halliburton) — overcomplex for v1. Tier IS the activation level.
- Email-gated pricing (Georgia Club) — anti-pattern; we're transparent.
- "Charity application" sub-flow — Halliburton lets nonprofits APPLY to be a beneficiary; not relevant for C2 Adopt.

### Patterns we ALREADY beat the field on

- **Live in-app sponsor preview** (the "What sponsors get on player phones" block we shipped today). **Zero of the 7 sites** showed what the sponsorship actually LOOKS like in the player experience. This is genuinely differentiating — most charity golf microsites are static text + sponsor logos. The screenshots make C2 Adopt's microsite credibly *next-generation*.
- **Stripe + Apple Pay / Google Pay payment** (we have this already). Most analyzed sites used external platforms (BirdEase, GolfGenius, SGA Software, OneCause) — meaning a redirect away from the microsite. Inline payment keeps the conversion in our brand.
- **22 distinct tiers with rich descriptions + images.** Most sites had 4-7 tiers and Halliburton's 12+ was text-only. C2 Adopt + GolfSync = 22 cohesive tiers + ticket bundles + flight-aware presentation, all admin-editable.

---

*Synthesized 2026-05-16 from WebFetch of 7 live nonprofit golf microsites. ACF Charity Classic was the 8th target but had a TLS cert mismatch and was skipped. Each per-site analysis is bullet-summarized above; raw analysis lives in the chat transcript.*
