# Design: BSG Major #1 + #4 — "Wolf Hunt @ Sycamore Creek" (May 2 + Aug 15, 2026)

_Captured 2026-05-15 from the BSG website ([event page](https://www.bhattalionswingers.com/bsg-summer-series-2026/the-majors/sycamore-creek-may-2-aug-15) + [Majors parent page](https://www.bhattalionswingers.com/bsg-summer-series-2026/the-majors))._

## What the format actually is

Standard rotating Wolf — 4 players per group, the "Wolf" rotates every hole, plays 2-vs-2 or 1-vs-3:

- **Group of 4** players. Same 4-some plays the whole round.
- **The Wolf rotates every hole**: Player 1 is Wolf on Hole 1, Player 2 on Hole 2, etc. After 4 holes the rotation resets (P1 again on H5, P2 on H6...). On an 18-hole round the rotation goes through 4 full cycles + 2 extra (P1/P2 get one extra Wolf turn).
- **Wolf always tees off LAST.** As each non-Wolf player drives, the Wolf has 3 seconds to either **PICK** that player as partner (becomes 2v2 for the hole) or **PASS** (waits for next drive).
- **Lone Wolf**: if the Wolf hates everyone's drive, they can declare "Lone Wolf" before their second shot — they play **1-vs-3** for the hole.
- **Net scoring**, with strokes announced verbally on each hole ("I'm a Net 4 on this hole, got a stroke on the Wolf").
- **Payout grid** for the hole is NOT published — the page literally says "Scoring (The Spreadsheet Logic): USE THE APP." Implementation has to pick a convention or surface it as configurable.

**Two event dates, same course, same format:** Major #1 (May 2) and Major #4 (Aug 15). The "May 2 – Aug 15" in the URL slug spans **both** dates, not a qualifying window. They are two discrete tournament days that each count toward the season "3 best of 5" aggregation.

**Field:** 16 players in 4 groups of 4. Named groups are published (Left-Right Express, Iceman, Lieutenant Pearson, Par Insurance).

**Fee:** $100/event.

## What architecturally maps to what we already built

| Concept | Existing mechanism | Fit |
|---|---|---|
| Group of 4 players | `LeagueTournamentPairing` (foursomes already exist for tee-time grouping) | ⚠ pairing exists, but per-hole-points game state doesn't |
| Net per-hole math | `LiveScoringService.computePerHoleNet` | ✅ each player's net stroke per hole is already computed; Wolf math reads those |
| Cumulative team-style points | `TeamLeaderboardResponse.matchPoints` (MATCH_PLAY) | ⚠ analogous shape but Wolf isn't team-vs-team across the field — it's points per player accumulating |
| Per-hole interactive picks ("PICK / PASS / Lone Wolf") | **Nothing today** | 🚫 brand-new state machine |
| Designated-scorer flow (one player enters for foursome) | `team.scorer_user_id` + `submitHoleScore(targetUserId)` | ✅ exact reuse — Wolf groups need a single scorer per group, same as BSG foursomes |

## The honest gap: Wolf is **per-player points**, not team scoring

Every other format in the codebase (BSG, Match Play, Stableford team rollups, the upcoming Three-Headed Monster) computes a **team** total across players. Wolf computes a **per-player points balance** that accumulates over 18 holes — the leaderboard is **4 players within each group**, then optionally **all 16 players across the field** if cross-group comparison is meaningful.

Wolf's state per hole is also non-trivial:

- **Who's the Wolf** (rotation)
- **Wolf's decision on each non-Wolf drive**: PICK (with which player) or PASS
- **Lone Wolf declaration** (before second shot)
- **Per-player net score on the hole** (the existing engine handles this)
- **Point distribution** based on who won the hole vs. the Wolf's team configuration

Without an interactive PICK/PASS state machine, the app can't honestly run Wolf. **The scorer would need to enter each hole's outcome retroactively** ("Hole 3: Sam was Wolf, picked Pat, Sam+Pat beat Drew+Andy with net 4 vs 5 → Sam +2, Pat +2, Drew -2, Andy -2").

## Three implementation options

### Option A — Retroactive hole-outcome entry (lightest)

The scorer enters: net score per player + "who was the Wolf" + "who did the Wolf pick (or Lone Wolf)" + the system computes points using a fixed payout grid. No real-time PICK/PASS button.

**Pros**: smallest build (~1 weekend of work). Reuses the existing scorer model + net computation. Mobile UI is one screen per hole with one extra dropdown ("Who did the Wolf pick?") on top of the existing per-player stepper.

**Cons**: doesn't capture the *live game* feeling. The 3-second PICK/PASS timer is a real social dynamic that the app can't reproduce after-the-fact. Lone Wolf bonus math also typically depends on WHEN the Lone Wolf was declared (before any drive vs. after seeing N drives) — retroactive entry collapses that.

**Best fit if**: the BSG group just wants the scoreboard to compute and Charlie's foursome-scorer-on-paper experience already works socially.

### Option B — Live PICK/PASS UI (heaviest)

The scorer's app shows the rotation, lets them tap "Wolf is Sam" → after each drive a button appears "Sam picks Pat?" / "Sam passes" / "Sam goes Lone Wolf." Real-time state machine, designated scorer drives it.

**Pros**: matches the actual on-course interaction. Could even surface PASSED-on-three-already messages as warnings.

**Cons**: significant build (~3 weekends). State machine + persistence + concurrency (two scorers in two groups must not race). Real WebSocket-style updates aren't there yet (project memory `project_websocket_backlog.md` defers WebSockets until concurrent users >50).

**Best fit if**: BSG wants to grow Wolf into a flagship product feature; or if Charlie's group asks "can I see the picks history at the end of the round?"

### Option C — Just compute the leaderboard from per-player nets + a per-hole "Wolf config" JSON

The scorer's existing flow doesn't change at all — they enter each player's net score per hole. A separate "Wolf Hunt config" form (admin-side or per-tournament) lets the organizer enter, after the round, who was Wolf and who they picked for each of 18 holes. The system computes points and renders the leaderboard.

**Pros**: zero changes to the player scoring flow. Reuses the existing scoring path 100%. The "who was Wolf / who got picked" data entry can happen post-round on a single web form by the organizer.

**Cons**: organizer has to do post-round bookkeeping. Players in the round can't see live points — leaderboard only computes after the organizer fills the Wolf config.

**Best fit if**: minimum viable product to satisfy "the app does scoring." Defer the live-PICK UX to a future build once Wolf has proved itself.

**Recommendation: Option A.** It's the middle ground. The scorer enters live (matching their existing BSG-foursome muscle memory), the per-hole Wolf decision is one tap, and the points compute immediately. Defer Option B's live PICK/PASS to a v2 unless the group asks for it.

## Payout grid — pick a default + make it configurable

Standard Wolf payouts vary by club. Most common shape:

| Outcome | Wolf gets | Wolf's partner gets | Each pack member gets |
|---|---|---|---|
| Wolf+partner WIN | +2 | +2 | −1 each |
| Pack WINS | −2 | −2 | +1 each |
| **Lone Wolf** WINS | +4 (or +3) | — | −1 each (or −2 each) |
| **Lone Wolf** LOSES | −2 (or −3) | — | +1 each (or +2 each) |
| Tie | 0 | 0 | 0 |

Net result: points sum to zero per hole. Cumulative balance over 18 holes is the leaderboard. **Make this a per-tournament config** with a "BSG default" preset.

## Migration 139 sketch

```sql
-- Per-tournament Wolf config. Only meaningful when tournament_type =
-- WOLF (new value); otherwise ignored.
ALTER TABLE league_tournaments
    ADD COLUMN wolf_lone_win_payout INT NULL,        -- default 4
    ADD COLUMN wolf_lone_loss_payout INT NULL,       -- default -2
    ADD COLUMN wolf_partner_win_payout INT NULL,     -- default 2 (each on winning side)
    ADD COLUMN wolf_pack_win_payout INT NULL,        -- default 1 (each pack member)
    ADD COLUMN wolf_strokes_for_lone_wolf BOOLEAN NULL DEFAULT TRUE;
    -- whether the Lone Wolf still gets their handicap strokes; some
    -- variants say "Lone Wolf plays gross" — config knob for both.

-- New table: per-hole Wolf decision + outcome. One row per hole per
-- tournament per group; carries who was Wolf, who they picked
-- (or null for Lone Wolf), and the resulting per-player points delta.
CREATE TABLE league_tournament_wolf_holes (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tournament_id BIGINT NOT NULL,
    group_pairing_id BIGINT NOT NULL,                -- which foursome
    hole_number INT NOT NULL,
    wolf_user_id BIGINT NOT NULL,
    pick_user_id BIGINT NULL,                        -- null = Lone Wolf
    wolf_won BOOLEAN NULL,                           -- null until hole settles
    points_json JSON NOT NULL,                       -- { "userId": delta, ... }
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uq_wolf_hole (tournament_id, group_pairing_id, hole_number),
    INDEX idx_wolf_tournament (tournament_id),
    FOREIGN KEY (wolf_user_id) REFERENCES users(id),
    FOREIGN KEY (pick_user_id) REFERENCES users(id)
);
```

The `points_json` blob is denormalized for fast read; a fully normalized "wolf_hole_points" child table would be more correct but adds latency to every leaderboard read. Given groups are 4 players and rounds are 18 holes, the blob stays small (<1KB per row).

## New tournament type vs. extending existing types

This needs to be a **new `tournament_type` value: `WOLF`** because:

- It's neither `INDIVIDUAL` (no field-wide stroke total) nor `TEAM_MATCH_PLAY` (teams rotate every hole, not fixed)
- The leaderboard is per-player points, sorted descending (higher wins) — opposite of stroke play
- Per-hole state shape (Wolf, pick, points delta) doesn't fit any existing scoring table

Add `WOLF` as the third option in the tournament-type enum + matching `teamScoringMode = 'WOLF_POINTS'`.

## What the scorer UI looks like on mobile

Same scorer-mode infrastructure as BSG foursome, but with one extra interaction per hole:

1. **Hole header**: "Hole 3 · Par 4 · **Sam is the Wolf**" (rotation is fixed at tournament start)
2. **Stepper per player** to enter strokes (existing pattern from ScorecardPanel)
3. **Wolf decision picker** (a third widget below the steppers): three buttons "Wolf picked Pat", "Wolf picked Drew", "Wolf picked Andy", "Lone Wolf"
4. **Live points readout** below the picker: "Sam +2, Pat +2, Drew −1, Andy −1" updates as strokes change
5. **Save** the whole hole as a unit (steppers + decision) — atomic write

The scorer-chip / read-only banner from BSG (`scorerMode === 'readOnly'` + "Sam is scoring") reuses unchanged.

## What the leaderboard shows

Two views, toggle between them:

1. **Group leaderboard** (default during the round): 4 rows for the group, cumulative points balance, sorted descending. "Sam +6, Pat +4, Drew −3, Andy −7."
2. **Field leaderboard** (for cross-group comparison): all 16 players, points balance summed across each group separately, sorted descending. This works because Wolf is zero-sum per group — comparing across groups is meaningful only if the BSG organizer treats it as "best individual session" rather than "won the group."

Per-hole breakdown: "Hole 3 (Sam was Wolf, picked Pat, won) → Sam +2 Pat +2 Drew −1 Andy −1."

## Tiebreakers + payouts

**Tiebreakers**: not published. Default the implementation to "back-9 points, then back-6, then back-3, then last hole" — matches the existing scorecard tiebreak ladder in `LiveScoringService.tiebreakKey`.

**Payouts**: not published in dollar amounts. The $100 fee covers green fees + cart + range + swag + junk bets + scoring system (per parent Majors page). No payout split published. Defer payout-distribution UI; the leaderboard answers "who won the day."

## Season aggregation (still TBD)

Same dependency as Three-Headed Monster: BSG counts "3 best of 5" across the season. Need new `LeagueSeason.aggregation_mode` + `best_count` columns. **Wolf's two events (May 2 + Aug 15) both feed the same season standing**, so the aggregation has to:

1. Convert each Major's player-level result into a comparable "Major score" (for Wolf: cumulative points balance for the round; for BSG 4-3-2: team net total; for Three-Headed Monster: team net total). Different scoring units across majors — need a normalization rule.
2. Drop each player's worst 2 of 5 Major scores.
3. Sum the remaining 3.

**This is the bigger architecture question.** A Wolf player ends the round at "+8 points" while a BSG 4-3-2 team ends at "Net 67" — those don't add together meaningfully. The season-aggregation layer needs a **per-Major normalization step**: convert each player's Major result to a 0-100 percentile-of-field score, then "3 best" is well-defined.

That's a real design conversation worth having before any of the 5 Majors finishes, but Wolf itself can ship without it — the per-event leaderboard works standalone.

## What's missing from the source page (questions for the organizer)

Per what was extracted from the BSG site:

1. **Per-hole payout grid**: what does Lone Wolf win/lose? What's the standard PICK win/loss? (Page defers to "the app" — organizer needs to pick the values)
2. **Does Lone Wolf get strokes?** Common variant: no, Lone Wolf plays gross to balance the 1-vs-3.
3. **Tee order within the group**: who's Player 1 / 2 / 3 / 4 in the rotation? Random? Captain assigns? Sorted by handicap?
4. **Tiebreaker procedure** (within a group + season-wide)
5. **Junk Bets integration**: closest-to-pin, longest drive, skins — are those tracked in the same scoreboard or separate?
6. **Cross-group comparison**: does the field leaderboard matter, or is each group its own standalone match?
7. **Season normalization**: how does a Wolf "+8" stack against a BSG 4-3-2 "Net 67" in the 3-best-of-5 calculation?

## Scope of work to implement

| Piece | Estimate | Required for May 2 |
|---|---|---|
| Migration 139 (Wolf config columns + wolf_holes table) | 1 hr | Yes |
| `WolfScoring` pure-function engine (compute per-hole points from inputs) + tests | 3-4 hr | Yes |
| Tournament-type enum: `WOLF` + `teamScoringMode = 'WOLF_POINTS'` | 30 min | Yes |
| WolfController: GET/PUT per-hole decisions + per-group leaderboard | 3 hr | Yes |
| Web UI: tournament create form Wolf preset (payout grid editable) | 2 hr | Yes |
| Web UI: group assignment editor (4 players per group, ordered) | 2 hr | Yes |
| Web UI: per-group + field leaderboard views | 2 hr | Yes |
| Mobile: hole-by-hole Wolf scorer UI (steppers + decision picker + live points readout) | 4 hr | Yes |
| Tiebreaker logic | 1 hr | Nice-to-have for May 2; required for the season finale Oct 3 |
| Junk Bets (CTP, LD, skins) | 4-6 hr | **No** — separate ask, separate ship |
| Season aggregation (3 best of 5 + cross-Major normalization) | 8-12 hr | **No** — needed by Oct 3 Finale, not by May 2 |

**Bottom line: ~18-20 hours of focused work** to ship Wolf by May 2. The Wolf engine + the per-hole scorer interaction are the two heaviest pieces; everything else is wiring through existing infrastructure.

**Before any of this gets built, organizer answers to questions 1, 2, 3 above are required.** The payout grid and Lone-Wolf-strokes rule materially change the math; tee order changes the rotation logic.

## Open decisions

- **Option A (retroactive entry) vs. Option B (live PICK/PASS UI) vs. Option C (post-round Wolf-config form)**. Default to **Option A** unless the organizer wants the on-course interactive flow.
- **Whether to ship as `tournament_type = WOLF` or piggyback on `TEAM_MATCH_PLAY` with a new `teamScoringMode`**. Default to new tournament type — Wolf's state machine genuinely doesn't fit the team-match-play shape.
- **Cross-Major normalization rule** for the "3 best of 5" season standings. **Defer to a separate design conversation** — affects all 5 Majors, not just Wolf.

## Cross-references

- BSG 4-3-2 engine: `golfsync-api/src/main/java/com/golfsync/service/scoring/BestKOfNPerHoleTeamScoring.java`
- Three-Headed Monster design doc: `THREE_HEADED_MONSTER_2026-05-15.md`
- Existing scorer permission model: migration 135 (`team_scorer.sql`) + `LiveScoringService.requireDesignatedScorer`
- LeagueTournamentPairing (foursome grouping that Wolf rotation reads from): exists today for tee-time groupings
- Season standings (where 3-best-of-5 lands): `SeasonStandingsResponse` — needs aggregation_mode field
