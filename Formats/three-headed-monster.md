# Design: BSG Major #3 — "The Three-Headed Monster" (July 18, 2026)

_Captured 2026-05-15 from the BSG website ([event page](https://www.bhattalionswingers.com/bsg-summer-series-2026/the-majors/tbd-july-18) + [Majors parent page](https://www.bhattalionswingers.com/bsg-summer-series-2026/the-majors))._

## What the format actually is

Three different 2-man team formats across one round, one card:

- **Holes 1–6**: **2-Man Scramble** — both hit drives; team picks one; both play from that spot; both putt; lowest of the two = team score
- **Holes 7–12**: **Alternate Shot (Foursomes)** — both tee off; team picks one drive; **partners alternate every shot from there** through the hole; one ball, one score
- **Holes 13–18**: **Greensome / Switch & Scramble** — both tee off; **swap drives** (partner A plays B's drive; partner B plays A's drive); each plays their swapped drive into the hole independently; lowest of the two = team score

**Plus the season-level wrapper:** "count 3 best of 5 Majors" — each team's series total is the sum of their best 3 net scores across the 5 Majors. Drop-the-worst-2.

## What architecturally maps to what we already built

| Concept | Existing mechanism | Fit |
|---|---|---|
| 2-man team | `LeagueTournamentTeam` + `LeagueTournamentTeamMember` | ✅ exact reuse, just size 2 instead of 4 |
| Net per-hole math | `LiveScoringService.computePerHoleNet` | ✅ already does NET allocation per-hole with allowance + cap |
| Foursome scorer (one player enters for team) | `team.scorer_user_id` + `submitHoleScore(targetUserId)` | ✅ exact reuse |
| Per-hole team total (varies by hole range) | `BestKOfNPerHoleTeamScoring` engine + `LeagueTournamentHoleGrouping` rows | ⚠ close but not quite — see below |
| Season aggregation "best 3 of 5" | `LeagueSeason` + season standings | ⚠ need a new aggregation mode |

## The honest gap: the current engine is "best K of N **player nets**." This format is "different team-score function per hole range."

The BSG 4-3-2 engine asks: *on hole H, how many of the N team members' nets count for this hole?* That works because every hole is "everyone hits their own ball, then we take the best K."

**The Three-Headed Monster** doesn't work that way:

- **Scramble** (1–6): Players never hit their own ball after the tee. They both play from the same spot. There are **two posted scores per hole** (one per player) but they're playing the **same ball** for all shots after the drive pick. **The team score is one number** — the strokes it took to get that one ball in the hole. *Not* "best of the two nets."
- **Alternate Shot** (7–12): There's **one ball, one team score**. No per-player score even exists.
- **Greensome/Switch** (13–18): There ARE two balls (after the drive swap), but they're not independent strokes the way 4-3-2 assumes. Best of the two is the team score.

So this is **not "best K of N nets"**, it's **"team-of-1 score, format determines how that score is built."**

Three options for the implementation:

### Option A — New scoring engine: `MultiFormatRotationScoring`

A new pure-function engine sibling to `BestKOfNPerHoleTeamScoring`. Engine input:

- Hole groupings: `[{ fromHole, toHole, formatMode: 'SCRAMBLE'|'ALTERNATE_SHOT'|'GREENSOME' }]`
- Per-hole team posts: `Map<hole, { teamScore: int, perPlayerNotes?: ... }>`

The **scorer enters a single team score per hole**, not individual player scores. The engine just sums team scores per hole and applies net allocation to that team score (or to one designated "stroke-receiving player," see below).

**Pros**: clean. The mobile scorecard becomes one stepper per hole (team's strokes for the hole). UI is much simpler than the BSG foursome switcher.

**Cons**: handicap allocation in alternate-shot is actually ambiguous in real golf. Standard rule: the team gets **50% of the combined course handicap** as allocated strokes. We need a config knob for this.

### Option B — Reuse `BestKOfNPerHoleTeamScoring` but with K=1 + a "shared ball" flag

Force the data model to "each player still posts their own net" but with `bestK=1` everywhere, and add a flag that for alternate-shot holes the scorer enters **the same number twice** (both partners record the team's stroke count).

**Pros**: zero new engine code.

**Cons**: dishonest data model. Net allocation per player is meaningless on alternate-shot holes — the team got strokes via the combined-handicap rule, not via two independent player allocations. Aggregations and "per-hole breakdown" would lie.

### Option C — Don't ship as a configurable format. Ship as a hand-rolled tournament type with a dedicated controller/UI.

Like how `MATCH_PLAY` is its own scoring mode separate from BEST_K_BY_HOLE_RANGE.

**Pros**: clean separation. Future "Major #4 with a completely different rotation" can copy this and tweak.

**Cons**: more code; less leverage on the BSG infrastructure.

**Recommendation: Option A.** The format generalizes — there are dozens of "2-man rotation" tournaments at clubs across the country that combine 2-3 of these phases. A new engine + a new `teamScoringMode='ROTATION_BY_HOLE_RANGE'` puts you in a position to run any of them.

## Migration 138 sketch

```sql
-- Adds a per-hole-grouping format mode for the new rotation engine.
-- Existing hole_groupings rows continue to use bestK (BSG 4-3-2);
-- rotation rows set format_mode + leave bestK NULL.
ALTER TABLE league_tournament_hole_groupings
    ADD COLUMN format_mode VARCHAR(24) NULL,  -- 'BEST_K' | 'SCRAMBLE' | 'ALTERNATE_SHOT' | 'GREENSOME'
    MODIFY COLUMN best_k INT NULL;  -- was NOT NULL; rotation rows leave it null

-- New tournament-level toggle:
ALTER TABLE league_tournaments
    ADD COLUMN team_handicap_allowance_mode VARCHAR(32) NOT NULL DEFAULT 'PER_PLAYER';
    -- 'PER_PLAYER' = each player's full handicap applies on holes they play (current behavior)
    -- 'COMBINED_HALF' = standard alternate-shot/foursomes rule: team gets
    --                   (sum of both partners' course handicaps) / 2, allocated
    --                   against the team score
    -- 'LOW_PARTNER'   = greensome convention: 60% of low + 40% of high
```

## What the scorer interaction looks like on mobile

**During Scramble holes (1–6)**: One stepper per hole, labeled "Team score." Both partners can score (either is the team scorer, or rotates). The scorer chip is unchanged from BSG.

**During Alternate Shot holes (7–12)**: Same — one stepper per hole, labeled "Team score." A small inline footnote "Alternate shot — one ball" replaces the "Best 2 / Best 3" copy from the current BSG explainer.

**During Greensome holes (13–18)**: Two steppers per hole side-by-side ("Drive A → ball played by B" and "Drive B → ball played by A"), with team score = lower. Or, simpler: one stepper for "Team score (best of the two)" and the per-player detail collapses behind a toggle. **Recommend the simpler path for the player UX — both players still see one score per hole.**

## What the leaderboard shows

Identical shape to the current BSG leaderboard:

- Team total (cumulative net, lower wins)
- Per-hole breakdown that says "Hole 4 (Scramble): team 4 — Sam's drive selected, both played from 230" or similar
- Scoring-error banner reuses the existing `TeamSummary.scoringError` plumbing

## Season aggregation ("best 3 of 5 Majors")

Not currently in the codebase. **This is the bigger gap actually.** What exists today:

- `LeagueSeason` model + season standings (`SeasonStandingsResponse`)
- Today's aggregation: sum all tournament results

What's missing:

- "Best N of M" drop-the-worst aggregation
- New field on `LeagueSeason`: `aggregation_mode VARCHAR(24)` = 'SUM_ALL' | 'BEST_N_OF_M'
- New field: `best_count INT` (= 3 for BSG Majors)
- Standings query changes to sort + take top N before summing

**This is a real piece of work** — probably 4-6 hours backend + UI on the season settings page.

## What's missing from the source page (questions for the organizer)

Per what we extracted from the BSG site, these aren't published:

1. **Handicap allowance %**: 100%? 85%? 75% (USGA recommendation for these formats)?
2. **Alternate Shot handicap rule**: combined-half (standard) or per-player (non-standard)?
3. **Tee assignment**: same tee for both partners, or each plays their own?
4. **Tiebreaker procedure**: card playoff hole-by-hole back? Sudden-death? Lowest team handicap?
5. **Course / venue** (TBD per the page)
6. **Entry fee + payouts**
7. **How teams are formed** (random draw? self-pick? captain's choice?)

## Scope of work to implement

| Piece | Estimate | Required for July 18 |
|---|---|---|
| Migration 138 (format_mode column on hole_groupings) | 30 min | Yes |
| `MultiFormatRotationScoring` engine + tests | 2-3 hr | Yes |
| Service wire (`RyderCupService.buildRotationLeaderboard`) | 1 hr | Yes |
| Web UI: extend `BsgFormatSettings` to allow per-row format selection | 2 hr | Yes |
| Web UI: new format mode in EditTournamentModal radio | 30 min | Yes |
| Mobile: scorer UI tweak (team-score stepper instead of per-player) | 2 hr | Yes |
| API: `team_handicap_allowance_mode` toggle + allocation rewire | 2 hr | **Maybe** — depends on organizer's answer to question 2 |
| Season "best 3 of 5" aggregation | 4-6 hr | **No** — needed for end of August, not July 18 |
| Per-hole breakdown UI shows the format ("Scramble", "Alt Shot", "Greensome") | 30 min | Nice-to-have |

**Bottom line: ~10 hours of focused work** to ship a working Three-Headed Monster tournament by July 18. Most of the leverage from the existing BSG infrastructure (teams, scorer-mode, per-hole net, leaderboard plumbing) stays intact — only the per-hole-team-score function changes.

**Before any of this gets built, organizer answers to questions 1, 2, and 7 above are required.** Those choices materially affect the implementation. The rest can ship with sensible defaults that the admin can adjust later.

## Open decision: Option A vs. C

Defaulting to **Option A (new engine + new format mode)** unless the organizer answers reveal a need for tournament-specific logic that doesn't generalize. If Major #4 turns out to be a fundamentally different shape (skins game, modified Stableford with side bets, etc.), Option C becomes more attractive.

## Cross-references

- BSG 4-3-2 engine: `golfsync-api/src/main/java/com/golfsync/service/scoring/BestKOfNPerHoleTeamScoring.java`
- Hole-groupings table: migration 133 (`league_tournament_hole_groupings`)
- BSG pooling mode: migration 137 (`bsg_pooling_mode` column)
- Team handicap knobs: migration 136 (`low_handicap_reference`, `max_handicap_cap`)
- Team scorer: migration 135 (`scorer_user_id` on `league_tournament_teams`)
