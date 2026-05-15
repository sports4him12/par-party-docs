# Design: C2 Adopt @ The Federal Club — First Hosted Tournament

_Captured 2026-05-15. Tournament date: September 2026. Format: Captain's Choice scramble, 2 flights (AM/PM). First paying GolfSync customer for tournament hosting._

## What this is (and isn't)

**This is**: the productization of the tournament demo trifecta. Same surfaces (public microsite + admin command center + mobile day-of), but every fabricated thing becomes real:

- Cardinal Cup → C2 Adopt
- The fake registration form → actual Stripe + offline payment intake
- The fake sponsor inquiry → actual sponsor signup with multiple tiers + payment methods
- The fake admin tab → an actual command center with reconciliation tools
- The fake mobile day-of demo → actual live scoring for AM + PM flights independently

**This is not** a one-off custom build for C2 Adopt. Every piece needs to generalize to the next tournament — the natural unit is a `HostedTournament` entity that any organizer can set up. **C2 Adopt is the v1 reference customer; the architecture supports v2 and beyond.**

## Confirmed requirements (organizer interview, 2026-05-15)

- **Format**: Captain's Choice (4-person scramble — every player tees, captain picks best ball, everyone plays from there)
- **Flights**: AM and PM on the same day. Different tee times. Same course. Same format. **Separate fields, separate winners, separate teams, separate leaderboards.**
- **Payment methods needed**: credit card, check by mail, ACH/bank transfer, invoice/PO, cash at the event
- **Pain points**: BOTH intake (forms hard to configure) AND reconciliation (who paid, who's in which flight, who's on which team) — workflow start to finish
- **Pre-launch**: site stays gated until organizer flips to public

## Architectural decisions

### 1. Hosted tournament reuses `LeagueTournament` (with extensions)

The existing `LeagueTournament` lives inside a `League` and assumes players are league members. C2 Adopt is the opposite: players don't exist yet when registration opens, and they're not joining a league afterward.

**Decision: Reuse `LeagueTournament` under a synthetic "C2 Adopt 2026" league.** Lightest engineering. Every tournament still needs a league shell; registrants get auto-created as league members on signup; mobile players install the app and find the league. The registration model gets extensions for hosted-specific fields (sponsor company, t-shirt size, dinner, dietary). When this proves out across 3-5 hosted tournaments, refactor to a dedicated `HostedTournament` entity as v2.

### 2. Two flights = two `LeagueTournament` rows with a sibling link

Captain's Choice with AM and PM flights is genuinely two tournaments on the same day at the same venue with separate fields, separate winners, separate teams.

**Decision: Two `LeagueTournament` rows in the same league, linked via a new `sibling_tournament_id` foreign key.** Admin creates "C2 Adopt - AM Flight" and "C2 Adopt - PM Flight" and links them. Registrants pick AM or PM at registration time. Public microsite shows both as one event but registration form has a flight picker. Leaderboards are independent.

Alternative considered (one tournament + flight column on registrations/teams): rejected because the leaderboard service would have to grow flight-aware in 12+ query paths. Sibling tournaments is less invasive.

### 3. Role-gated pre-launch

**Generic `TOURNAMENT_ORGANIZER` role + per-tournament grant table.** Not a hardcoded `C2_ORGANIZER` role — generalize so Customer 2's organizer is a 2-second config change.

The role tells Spring Security "this user can access organizer surfaces." A `tournament_organizers` table maps (user, tournament) and decides which specific tournament. Microsite route is 404 to anyone without admin role OR a grant for that tournament — until `is_publicly_listed` is flipped true.

## The four surfaces, by priority

### 1. Public microsite — `/tournaments/c2-adopt-2026`

Replaces `/demo/tournament`. Sections mostly map from what was built for the demo, but real:

- **Hero + identity**: C2 Adopt logo, "21st Annual C2 Adopt Golf Tournament", Sept date, The Federal Club, "Powered by GolfSync" footer
- **Mission section**: pulled from C2 Adopt's about page
- **Schedule**: real schedule (registration table, breakfast, shotgun start, lunch turn, dinner + auction)
- **Register section**:
  - Player type picker: **Foursome** ($X) / **Individual** ($Y) / **Dinner-only** ($Z)
  - Flight picker (AM / PM) with capacity counts ("38 / 72 spots filled")
  - Player details (name, email, phone, handicap, t-shirt size, dietary)
  - **Payment method picker**: credit card / check / ACH / invoice / pay at event
  - Conditional payment UI: card → Stripe; check → "Mail to..." instructions; ACH → routing info; invoice → company + billing email; in-person → "Pay at registration table"
  - Submit → confirmation page + email
- **Sponsorship section**: tiered cards (Title, Premier, Hole, Banquet, In-Kind) with prices + benefits
  - Tap a tier → application form (company info, contact, payment method same as above, optional logo upload)
  - Submit → confirmation + admin email
- **In-event experience preview**: marketing copy for what players get on the day (live leaderboard, mobile scorecard, etc.)
- **Contact / FAQ**

**No login required when `is_publicly_listed = true`. Until then, 404 unless you're admin or an organizer grant exists.**

### 2. Payment handling — four methods + admin comp

Each registration / sponsor application carries:

- `paymentStatus`: `pending_payment` / `paid` / `comped` / `refunded`
- `paymentMethod`: `card` / `check` / `ach` / `invoice` / `in_person` / `comp`
- `paidAt` timestamp + `paidBy` (admin who marked it)
- `paymentReference`: check number, ACH confirmation, PO number, etc.

**Credit card flow** (Stripe, already wired in the codebase for membership):

- Charge on submit → status `paid` immediately
- Failure → registration enters `pending_payment` status with error captured

**Offline flows** (check, ACH, invoice, in-person):

- Submit puts registration in `pending_payment` immediately
- Confirmation email tells the registrant: "Your spot is held pending payment. Send your check to [address] referencing your confirmation #..."
- Admin gets an email too
- Spot is **held but not confirmed** until admin marks paid

**Capacity counting**: a `pending_payment` registration counts against the flight cap (so AM flight doesn't oversell), but capacity displays both "62/72 registered" and "8 pending payment" so the admin sees the risk.

**Payment data visibility for organizers**: organizers see status + amount + method (e.g. "paid via check ref #1234"). **Never sees credit card details, including last-4.** Reduces PCI scope. Stripe's dashboard remains the source for card details (admin-only).

### 3. Admin command center

Lives at `/tournaments/c2-adopt-2026/admin`. **Organizers landing on login go straight here** (skip the regular GolfSync home / leagues / mobile feed). If they have multiple tournament grants, they get a chooser.

**Registrations tab** (the big one):

- Table: name, email, phone, flight (AM/PM), payment status, amount, paid date, actions
- Filters: flight, payment status, registration type
- Bulk actions: "Mark these 8 as paid" (one click after a check batch arrives), "Resend confirmation"
- Per-row actions:
  - **Switch flight** (AM↔PM, with confirmation modal "Mike has paid; we're moving them from AM to PM")
  - Mark paid / refund / cancel
  - Edit (fix typo, update t-shirt size)
  - Assign to team
- Inline `Add registration` for the org admin adding a paper-form signup

**Sponsors tab**:

- Same structure for sponsors: company, tier, contact, status, amount, logo
- Bulk-export logo files for the on-course signage prep
- Per-row mark-paid + edit

**Payments tab**:

- Reconciliation view: every payment with method + reference
- Filter "pending payment for >7 days" → quick chase list
- "Send reminder email to N pending registrants" bulk action

**Teams tab**:

- Per-flight team rosters
- Auto-assign button: "Pair foursomes by handicap" / "Random pairs from singles" / "Manual drag-drop"
- Edit team name, captain, color

**Day-of tab**:

- Tee sheet (printable)
- Cart signs (printable)
- Live leaderboard preview
- Push announcement composer (already built in the demo)

**Setup checklist** at the top — the "cakewalk" promise:

- ✅ Registration window open
- ✅ Sponsor tiers configured
- ⚠ Stripe webhook untested — [Send test charge]
- ⚠ 14 days until event — 32 of 72 AM spots filled, 18 of 72 PM spots filled
- ❌ Hole pars not configured for The Federal Club — [Course Editor]
- ❌ Teams not assigned — [Assign now]
- ❌ Site not yet public — [Publish microsite]

Each item links to where to fix it.

### 4. Mobile day-of — flight-aware

The existing scorer + leaderboard infrastructure works. Two changes:

- **Mobile home / tournament list**: the LIVE banner has to know which flight a player is in and surface that flight's tournament — not the AM if they're in PM
- **Leaderboard**: each flight has its own. Optional "Field" view shows both side by side if the org wants cross-flight context

**Captain's Choice scoring is just one of the existing scoring modes (SCRAMBLE / BEST_BALL).** Team's score is the single ball the captain played. Already supported by `LiveScoringScorecard` — one stepper per hole, team total. **No new mobile state machine** — it's the simplest team format because there's literally one ball.

## Role gating & access control (P1)

### `TOURNAMENT_ORGANIZER` role + `tournament_organizers` grant table

Capabilities map:

| Capability | USER | TOURNAMENT_ORGANIZER (granted for tournament X) | ADMIN |
|---|---|---|---|
| View `/admin` | ❌ | ❌ | ✅ |
| View `/tournaments/X` microsite | only if `is_publicly_listed` | ✅ | ✅ |
| Edit X's registrations | ❌ | ✅ | ✅ |
| Edit X's sponsors | ❌ | ✅ | ✅ |
| Switch a player AM↔PM | ❌ | ✅ | ✅ |
| Mark a registration paid | ❌ | ✅ | ✅ |
| Other admin tabs (users, leagues, etc.) | ❌ | ❌ | ✅ |
| See other customers' tournaments | ❌ | ❌ | ✅ |
| See credit card details (last-4 etc.) | ❌ | ❌ | ✅ |
| Grant organizer role to others | ❌ | ❌ | ✅ |

### Three gating layers

1. **Public microsite**: gated by role + per-tournament grant. Not signed in → redirect to login. Signed in but not granted → 404 (don't reveal the tournament exists). ADMIN sees everything.
2. **"Flip to public" toggle**: `is_publicly_listed` on the tournament. False = gated. True = anyone with the URL can see + register. One click, intentional.
3. **Registration window**: even when public, registration may be closed (event happened, capacity reached, paused). Independent of the above gating.

### Inviting an organizer

Since Becky probably doesn't have a GolfSync account yet:

1. ADMIN goes to tournament's admin page → "Manage organizers"
2. Enters Becky's email → "Invite as organizer"
3. System creates pending invite (or auto-creates her user in `PENDING` state) + emails her a signup link
4. She clicks → completes account → lands on C2 Adopt admin with `TOURNAMENT_ORGANIZER` role + grant already applied

Mirrors the existing league invite pattern — same template, different role grant on accept.

## Data model additions

```sql
-- Migrations (exact #s pending — verify table names per
-- feedback_verify_table_names_in_migrations.md):

-- 1. Hosted-tournament-specific registration fields.
ALTER TABLE league_tournament_registrations
    ADD COLUMN registration_type VARCHAR(24) NULL,  -- 'foursome' | 'individual' | 'dinner_only'
    ADD COLUMN flight VARCHAR(8) NULL,              -- 'AM' | 'PM' | NULL for non-flighted
    ADD COLUMN tshirt_size VARCHAR(8) NULL,
    ADD COLUMN dietary_restrictions VARCHAR(255) NULL,
    ADD COLUMN payment_method VARCHAR(16) NULL,
    ADD COLUMN payment_status VARCHAR(16) NOT NULL DEFAULT 'pending_payment',
    ADD COLUMN payment_reference VARCHAR(128) NULL,
    ADD COLUMN paid_at DATETIME NULL,
    ADD COLUMN paid_by_user_id BIGINT NULL,
    ADD COLUMN amount_cents INT NULL,
    ADD COLUMN confirmation_number VARCHAR(16) NULL;

-- 2. Flight pairing + public listing on the tournament.
ALTER TABLE league_tournaments
    ADD COLUMN sibling_tournament_id BIGINT NULL,    -- links AM ↔ PM
    ADD COLUMN flight_label VARCHAR(16) NULL,
    ADD COLUMN flight_capacity INT NULL,
    ADD COLUMN public_slug VARCHAR(64) NULL,         -- 'c2-adopt-2026'
    ADD COLUMN is_publicly_listed BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN organizer_invite_email VARCHAR(128) NULL;

-- 3. Sponsor entity.
CREATE TABLE tournament_sponsors (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tournament_id BIGINT NOT NULL,
    company_name VARCHAR(128) NOT NULL,
    contact_name VARCHAR(128) NOT NULL,
    contact_email VARCHAR(128) NOT NULL,
    contact_phone VARCHAR(32) NULL,
    tier VARCHAR(32) NOT NULL,
    amount_cents INT NULL,
    payment_method VARCHAR(16) NULL,
    payment_status VARCHAR(16) NOT NULL DEFAULT 'pending_payment',
    payment_reference VARCHAR(128) NULL,
    paid_at DATETIME NULL,
    paid_by_user_id BIGINT NULL,
    logo_url VARCHAR(512) NULL,
    notes TEXT NULL,
    confirmation_number VARCHAR(16) NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_tournament_sponsors_tournament (tournament_id),
    FOREIGN KEY (tournament_id) REFERENCES league_tournaments(id) ON DELETE CASCADE
);

-- 4. Sponsor tiers — per-tournament config.
CREATE TABLE tournament_sponsor_tiers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tournament_id BIGINT NOT NULL,
    tier_key VARCHAR(32) NOT NULL,
    display_name VARCHAR(64) NOT NULL,
    description TEXT NULL,
    price_cents INT NOT NULL,
    benefits_json JSON NULL,
    display_order INT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE KEY uq_sponsor_tier (tournament_id, tier_key),
    INDEX idx_tier_tournament (tournament_id)
);

-- 5. Organizer grant table — generic, per-tournament.
CREATE TABLE tournament_organizers (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tournament_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    granted_by_user_id BIGINT NOT NULL,
    granted_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    revoked_at DATETIME NULL,
    notes VARCHAR(255) NULL,
    UNIQUE KEY uq_organizer (tournament_id, user_id),
    INDEX idx_organizer_user (user_id),
    FOREIGN KEY (tournament_id) REFERENCES league_tournaments(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (granted_by_user_id) REFERENCES users(id)
);
```

Also: extend the `UserRole` enum with `TOURNAMENT_ORGANIZER`. Spring Security uses this to allow access to the organizer surfaces; the per-tournament grant table decides which tournament.

## What can be reused vs. built fresh

| Surface | Reuse | Build fresh |
|---|---|---|
| Public microsite layout | `/demo/tournament` page | Real data binding + payment forms + gating |
| Stripe wiring | Existing membership Stripe flow | Tournament-payment intent shape |
| Confirmation emails | Existing `EmailService` | New templates per payment method |
| Mobile scorer (Captain's Choice) | `ScorecardPanel` + one-ball-per-hole | Nothing — Captain's Choice = simplest case |
| Leaderboard | Existing tournament leaderboard | Flight filter + sibling-tournament link |
| Team assignment | Existing Ryder Cup team UI | Sibling-tournament aware (AM ≠ PM teams) |
| Admin command center | Admin panel structure | Hosted-tournament-specific tabs |
| Course data (The Federal) | Admin Course Editor (just shipped) | Seed The Federal in production |
| Organizer invite flow | League invite-by-email pattern | New role grant on accept |

## Scope of work

Split into phases — too much to ship safely at once:

### Phase 1: Foundation (~15-20 hr)

- Migrations: registration extensions + sponsor tables + organizer grant
- Sponsor + extended registration entities/repos/services
- Public microsite: real data binding (replaces demo), reading from a `HostedTournamentService` that wraps `LeagueTournament` with the hosted-specific fields
- Sibling tournament linking + flight capacity counting
- Confirmation number generation + lookup

### Phase 2: Role gating + organizer invite (~6-8 hr)

- `TOURNAMENT_ORGANIZER` role in `UserRole` enum + Spring Security wiring
- `tournament_organizers` table + service + repo
- Microsite route gate (404 if not granted + not public)
- Admin UI: "Manage organizers" panel on tournament (grant/revoke by email)
- `is_publicly_listed` toggle in admin command center
- Organizer landing redirect (Becky → C2 Adopt admin on login)
- Tests for the gate (unauthed → 404, granted organizer → 200, admin → 200, ungranted user → 404)

### Phase 3: Payment intake (~12-15 hr)

- Stripe charge flow for tournament registrations (extending existing membership Stripe integration)
- Offline payment flows: confirmation pages per method, email templates
- Payment status state machine + admin reconciliation endpoints
- Receipt PDFs / invoices for sponsors who need an AP-friendly invoice

### Phase 4: Admin command center (~12-15 hr)

- Registrations table view + filters + bulk actions
- Sponsors table view
- Switch-flight modal
- Payments reconciliation tab
- Setup checklist with deep-link CTAs

### Phase 5: Day-of polish (~6-8 hr)

- Flight-aware mobile home banner
- Flight filter on the leaderboard
- Tee sheet print view (admin)
- Captain's Choice scoring is already supported; verify

### Phase 6: C2 Adopt specifics (~4-6 hr)

- Seed The Federal Club via the admin Course Editor (already shipped!)
- Configure C2 Adopt tournament + sibling AM/PM flights
- Configure sponsor tiers per C2 Adopt's spec
- Brand the microsite (logo, colors, copy)
- Test full registration → payment → admin reconciliation flow end-to-end

**Total: ~58-73 hours of focused engineering.** Spread over 3-4 weeks for a September event leaves comfortable margin.

## Open questions for C2 Adopt before code

1. **Sponsor tiers**: exact names, prices, benefits per tier? (Title $X, Premier $Y, Hole $Z, etc.)
2. **Registration prices**: foursome vs. individual vs. dinner-only?
3. **Flight capacity**: how many spots AM vs. PM?
4. **ACH / invoice flow specifics**: preferred bank for ACH? Existing invoicing tool (QuickBooks, Stripe Invoicing, FreshBooks)? Whether GolfSync generates invoices or just tracks payment status against external invoices?
5. **Mail-to address for checks**?
6. **Stripe processing fee**: C2 Adopt eats it or pass to registrant ("Add 3% to cover processing")?
7. **Refund policy**: cancellation deadlines, partial refunds, no refunds after X?
8. **Dinner-only and individual registrants**: source into flights' player count, or separate?
9. **Branding**: logo + brand colors? Or design the microsite header from scratch?
10. **Timeline**: registration opens when? Closes when? Day-of is September WHAT exactly?
11. **Organizer to grant**: Becky's email? Anyone else from their staff?

## What to build first, before all answers

Phase 1 (foundation) + Phase 2 (role gating) are mostly format-agnostic and unblock everything else. Start there in parallel with collecting C2 Adopt's answers. Phase 6 (their actual data) is the last thing to slot in.

## Cross-references

- Existing tournament demo (lives at `/demo/tournament`, will be replaced by `/tournaments/c2-adopt-2026`): `golfsync-web/app/demo/tournament/page.tsx`
- Admin Course Editor (just shipped 2026-05-15): use it to seed The Federal Club
- Membership Stripe flow (template for tournament Stripe wiring): `StripeService` and `membership_*` endpoints
- League invite-by-email pattern (template for organizer invite): existing league member invite
- Memory: `project_first_paid_customer.md` (Stableford league) and the BSG project memory — both inform the conventions for paying customers
- Sibling docs:
  - Format designs at `golfsync-docs/Formats/` (BSG Sixes, Three-Headed Monster, Wolf)
  - Once Customer 2 signs on, add `golfsync-docs/Customers/customer-2-name.md` using this as the template
