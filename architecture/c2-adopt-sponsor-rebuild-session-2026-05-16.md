# C2 Adopt Sponsor Rebuild — Session Summary

**Date:** 2026-05-15 → 2026-05-16
**Outcome:** Full C2 Adopt 22-tier sponsor structure live in prod with end-to-end ticket-claim automation. ~4 hours of focused work delivered across 4 sequential PRs + one follow-on, all deployed and verified.

## What shipped (in deploy order)

| Release | What | Verified |
|---|---|---|
| `prod-2026-05-16-0145` | OG card iteration 2 (golf markers, par row, breakdown panel) | ✓ |
| `prod-2026-05-16-0236` | TournamentFormatPolicy refactor (TeamScoringModeRules + StablefordPointsCodec extracted from LeagueService) | ✓ |
| `prod-2026-05-16-0336` | Branding image upload (Logo + Hero); web sponsor-highlights preview block; CSP dev fix | ✓ |
| `prod-2026-05-16-0449` | **PR 1 + 2 + 3** of sponsor rebuild — migration 145 (22 real tiers split across AM/PM siblings, `included_tickets` column); migration 146 (credibility fields); admin Sponsor Tiers CRUD tab with drag-reorder + duplicate-to-flight; microsite hero with Nth Annual badge + cumulative-raised band + benefiting line + per-flight tier grouping + spots-remaining + past-sponsor wall + tasteful Powered-by footer | ✓ |
| `prod-2026-05-16-1034` | **PR 4** — sponsor ticket-claim workflow end-to-end (migration 147 sponsor_ticket_claims table, SponsorTicketClaimService with idempotent issuance + sibling routing + partial fills + 90-day expiry, public /api/sponsor-claim/{token} controller, EmailService.sendSponsorTicketClaimEmail wired into all 4 paid-state entry points, web /sponsor-claim/[token] branded claim page with N player rows + AM/PM picker for flight-agnostic tiers) | ✓ |
| (deploying) | Resend-claim admin button (recovery for "sponsor lost the email") | bsn5zegqk in flight |

## The wow feature shipped

**Sponsor pays → branded claim page in inbox within seconds → fills player names + emails → those become real registrations on the AM/PM tee sheet automatically.** Zero Becky manual work. Per the research, zero of 7 analyzed nonprofit golf microsites had anything like this.

Companion to the in-app sponsor preview block (the screenshots) that lets sponsors see what they get on player phones BEFORE they commit. Together: the "wow C2 Adopt so they refer" combination that motivated the whole session.

## Tests

| Layer | Count | Outcome |
|---|---|---|
| API unit + slice | 2,563 (+27 from PR 4: 19 service + 8 controller) | 0 failures |
| Web type-check | clean across all 5 deploys | — |
| Cypress | not run this session (per founder direction) | — |

## Documentation written

- [architecture/solid-audit-2026-05-15.md](solid-audit-2026-05-15.md) — service-layer SOLID audit (466 lines)
- [architecture/refactor-readiness-2026-05-15.md](refactor-readiness-2026-05-15.md) — test-health audit for 4 refactor targets (337 lines)
- [architecture/tournament-format-policy-refactor-design.md](tournament-format-policy-refactor-design.md) — design doc for the format-policy extraction (231 lines)
- [architecture/nonprofit-golf-microsite-research-2026-05-16.md](nonprofit-golf-microsite-research-2026-05-16.md) — pattern research across 7 live charity tournament sites (88 lines)
- **(this doc)** session summary

## What did NOT ship (waiting on data or pending decision)

- **C2 Adopt tournament metadata** (date, time, registration prices, sponsor tier capacities if any differ from Becky's list, mailing address for checks, EIN, etc.). Mirror from OneCause was attempted; the page is JS-rendered + WebFetch only saw a shell. Recommend Becky pastes the relevant fields directly; admin Branding tab now has fields for inceptionYear, cumulativeRaisedCents, benefitingOrgName, taxEin.
- **Hosted-tournament refactor pass** for 1→20 scaling. The PR 2 work (admin tier CRUD) already removes the biggest "per-customer SQL migration" pain point — future hosted tournaments self-serve their tier set. Other 1→20 concerns (capacity planning, perf, multi-customer admin views) not yet addressed; suggest queuing this when tournament #2 commits.

## Founder action items

1. **Visit https://golfsync.io/admin/hosted-tournaments/c2-adopt-2026/manage → Sponsor Tiers tab.** Confirm the 22 tiers landed correctly. Edit any tier descriptions/prices via the new UI as Becky confirms her final numbers.
2. **Visit https://golfsync.io/admin/hosted-tournaments/c2-adopt-2026/manage → Branding tab.** Fill in the new credibility fields when Becky has the data:
   - First year of the event (powers the Nth Annual badge)
   - Cumulative raised in USD (powers the $X raised since YYYY band — the single most credibility-building number per the research)
   - Beneficiary org name (already seeded "C2 Adopt")
   - 501(c)(3) EIN (powers the footer tax-deductibility block)
3. **Test the sponsor claim flow end-to-end** before launch:
   - Mark a test sponsor row as paid in the admin
   - Check the sponsor's contact email for the claim link
   - Visit the link, fill in 2 of 4 player slots, submit
   - Confirm 2 registrations appear on the admin Registrations tab
   - Visit the link again, fill in the remaining 2, submit
   - Confirm token shows "all claimed" + tournament has 4 sponsor-attached registrations
4. **Test "Resend claim" recovery** by clicking the button on a paid sponsor row and confirming the email re-sends.

## Process notes worth saving

- 4 deploys across the session. All clean Liquibase + image-hash-verified. The deploy script's image-hash verification block was the load-bearing safety net — without it, the wrapper exit-0 wouldn't have been trustworthy. Don't skip it.
- The research subagent's WebFetch permission blocked it; the main agent retried with explicit permission per URL. Pattern that's worth saving: when a research subagent gets blocked on a permission, the main agent should fall back to performing the same fetches itself rather than spawning a fresh subagent — the latter would hit the same block.
- 4-PR sequencing was the right call. Each PR had a single coherent commit message + a verified deploy. Total work was substantial (~4 hours of focused implementation) but every commit was independently shippable. If any PR's deploy had failed, the others would still be live.
- TestSecurityConfig drift: any new public endpoint must be added to BOTH the prod SecurityConfig AND the test TestSecurityConfig. Bit me once during PR 4 (8 controller-slice tests returned 403); the existing inline comment in TestSecurityConfig calling out this exact issue saved future-me from a head-scratch.
