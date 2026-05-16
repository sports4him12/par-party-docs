# Tournament Format Refactor — three-axis model

**Status**: RFC — design only, no service code touched yet
**Author**: Ryan + Claude (pair design session)
**Date**: 2026-05-16
**Migration**: 151 (schema + backfill, additive)
**Inspiration**: <https://golfcollege.edu/popular-golf-tournament-formats/> (18 formats)

---

## 1 · Why

Today, "what kind of tournament is this" is spread across **five**
overlapping dimensions that don't compose cleanly:

| Column | Today's values | Meaning |
|---|---|---|
| `tournament_type` | `INDIVIDUAL` / `TEAM_MATCH_PLAY` | Whether `LeagueTournamentTeam` rows exist (Ryder Cup) |
| `team_scoring_mode` | `STROKE_TOTAL` / `MATCH_PLAY` / `BEST_K_BY_HOLE_RANGE` | Only meaningful when `tournament_type=TEAM_MATCH_PLAY` |
| `scoring_mode` (enum `ScoringMode`) | `INDIVIDUAL` / `SCRAMBLE` / `BEST_BALL` / `ALTERNATE_SHOT` / `MATCH_PLAY` | "v1 only branches on this for UI labelling" per the Javadoc — wired through `Round` (casual rounds) but the LeagueTournament column is largely cosmetic |
| `scoring_system` (enum `ScoringSystem`) | `STROKE` / `STABLEFORD` / `MODIFIED_STABLEFORD` / `CHICAGO` | NULL = STROKE default; drives `ScoringStrategy` selection |
| `custom_stableford_points` | JSON `{eagle,birdie,par,bogey,double,triple}` | Owner override of the Stableford table |

The five-way split has real costs:
- **`scoring_mode` and `team_scoring_mode` collide.** Both have `MATCH_PLAY`. Both partially encode "team vs. individual." The Ryder Cup work added the second one because the first one didn't compose with the existing Stableford path.
- **MVP restriction in code today**: `RoundService` line 69 — *"if (scoringSystem != STROKE && scoringMode != INDIVIDUAL) throw"*. We literally banned team × Stableford because the data model can't represent it.
- **`scoring_mode` is "for UI labelling"** on `LeagueTournament` — it doesn't drive the leaderboard math. The math reads `tournament_type` + `team_scoring_mode`. Two parallel encodings of the same concept.

Adding **Scramble** today means: which enum do you write to?
`scoring_mode=SCRAMBLE` is the existing label, but the leaderboard math
ignores it. So you'd also have to invent a `team_scoring_mode=SCRAMBLE`
value AND flip `tournament_type` to something new. None of those scale.

The PGCC reference page lists 18 common formats. After mapping them
to the existing column dimensions, the pattern is clear: **today's
columns conflate three independent concerns**:

1. **How players group up** (1 / 2 / 3 / 4 / multi-team)
2. **How the team's ball(s) get played** (own-ball / scramble / shamble / alternate)
3. **How a hole turns into a score** (stroke / best-K / match / Stableford / skins / BBB / flags)

Most formats are an (axis1, axis2, axis3) triple plus a small param
bag. Ryder Cup is the only composite case — it's a sequence of
triples (one per session) which we already handle via the existing
match table.

---

## 2 · The three-axis model

### Axis 1 — `team_shape` (VARCHAR(24) NOT NULL)

How players group up.

| Value | Meaning | Today's equivalent |
|---|---|---|
| `INDIVIDUAL` | 1 player per scorecard | `tournament_type=INDIVIDUAL` |
| `TWO_PLAYER` | Pair | (new) |
| `THREE_PLAYER` | Trio | (new) |
| `FOUR_PLAYER` | Foursome | (new) |
| `MULTI_TEAM_PAIRINGS` | N teams of M players each, with sub-pairings per session | `tournament_type=TEAM_MATCH_PLAY` |

### Axis 2 — `ball_rule` (VARCHAR(24) NOT NULL)

How balls get played on each hole.

| Value | Meaning |
|---|---|
| `OWN_BALL` | Every player plays their own ball all 18 (default; covers four-ball/best-ball/match-singles) |
| `SCRAMBLE` | All tee off → pick one → everyone plays from there |
| `SHAMBLE` | Scramble off the tee → own-ball from the fairway in |
| `ALTERNATE_SHOT` | True foursomes: 1 ball, alternate shots |
| `CHAPMAN` | Both tee → swap balls for shot 2 → pick best → alternate after |

### Axis 3 — `scoring_method` (VARCHAR(24) NOT NULL)

How a hole turns into a score that feeds the leaderboard.

| Value | Meaning | Today's equivalent |
|---|---|---|
| `STROKE_TOTAL` | Sum of strokes; lowest wins | `team_scoring_mode=STROKE_TOTAL` |
| `BEST_K_OF_N_PER_HOLE` | Best K of N net scores per hole; today's BSG | `team_scoring_mode=BEST_K_BY_HOLE_RANGE` |
| `MATCH_PLAY` | Head-to-head per hole, points to team/player | `team_scoring_mode=MATCH_PLAY` |
| `STABLEFORD` | Points table per hole (parametric) | `stableford_points_json` on INDIVIDUAL |
| `SKINS` | Per-hole pot; ties carry over | (new) |
| `BINGO_BANGO_BONGO` | 3 points/hole (first to green / closest / first in) | (new) |
| `FLAGS` | Stroke-budget against distance | (new) |
| `QUOTA` | Target-points (Quota/Chicago); Stableford variant w/ HCP-driven target | (new — could be a STABLEFORD param) |

### Axis 4 — `format_params_json` (JSON NULL)

A single JSON blob for per-format extras. Examples:

```json
// STABLEFORD
{ "pointsTable": { "doubleBogey": 0, "bogey": 1, "par": 2,
                   "birdie": 3, "eagle": 4, "doubleEagle": 5 } }

// SKINS
{ "skinValueCents": 500, "carryover": true,
  "bonusSkins": ["greenies", "sandies"] }

// BEST_K_OF_N_PER_HOLE (replaces today's standalone columns)
{ "k": 2, "n": 4, "poolingMode": "BY_HOLE_NUMBER",
  "holeRanges": [...] }

// MONEY_BALL (a BEST_K variant)
{ "k": 2, "n": 4, "designatedRotation": "ROUND_ROBIN" }

// FLAGS
{ "startingStrokes": 91, "handicapPctApplied": 100 }

// QUOTA / CHICAGO
{ "target": 36, "negativeStart": false }
```

**Why JSON not columns:** Columns are right for things you want to
filter/sort by (`status`, `start_time`, `tournament_type`). Per-format
config is opaque to SQL — `pointsTable.par` doesn't get a WHERE clause.
JSON keeps the schema flat and the rare params don't bloat every row.

The handful of existing single-purpose columns (`bsg_pooling_mode`,
`hole_score_cap_mode`, `handicap_allowance_pct`, `off_the_low`) stay
as columns for now to avoid a big-bang rewrite — they'll migrate into
`format_params_json` in a follow-up once everything reads through the
strategy layer.

---

## 3 · The 18-format decomposition

| PGCC Format | team_shape | ball_rule | scoring_method | format_params |
|---|---|---|---|---|
| Stroke Play | INDIVIDUAL | OWN_BALL | STROKE_TOTAL | — |
| Match Play | INDIVIDUAL | OWN_BALL | MATCH_PLAY | — |
| Better Ball (stroke) | TWO_PLAYER..FOUR_PLAYER | OWN_BALL | BEST_K_OF_N_PER_HOLE | `{k:1,n:N}` |
| Four Ball (match) | TWO_PLAYER | OWN_BALL | MATCH_PLAY | `{ballSelectionMode:"BEST"}` |
| Scramble | TWO_PLAYER..FOUR_PLAYER | SCRAMBLE | STROKE_TOTAL | — |
| Shamble | TWO_PLAYER..FOUR_PLAYER | SHAMBLE | STROKE_TOTAL or BEST_K | — |
| Alternate Shot | TWO_PLAYER | ALTERNATE_SHOT | STROKE_TOTAL or MATCH_PLAY | — |
| Chapman | TWO_PLAYER | CHAPMAN | STROKE_TOTAL or MATCH_PLAY | — |
| Ryder Cup | MULTI_TEAM_PAIRINGS | (per-session) | MATCH_PLAY | `{sessions:[...]}` |
| Skins | INDIVIDUAL (or team) | OWN_BALL | SKINS | `{skinValueCents,carryover}` |
| Stableford | INDIVIDUAL (or team) | OWN_BALL | STABLEFORD | `{pointsTable}` |
| Modified Stableford | INDIVIDUAL | OWN_BALL | STABLEFORD | `{pointsTable: {...custom}}` |
| Bingo Bango Bongo | INDIVIDUAL | OWN_BALL | BINGO_BANGO_BONGO | — |
| Flags | INDIVIDUAL | OWN_BALL | FLAGS | `{startingStrokes, hcpPct}` |
| Money Ball | FOUR_PLAYER | OWN_BALL | BEST_K_OF_N_PER_HOLE | `{k:2,n:4,designatedRotation}` |
| Quota | INDIVIDUAL (or team) | OWN_BALL | QUOTA | `{target,negativeStart:false}` |
| Chicago | INDIVIDUAL | OWN_BALL | QUOTA | `{negativeStart:true}` |
| Peoria | INDIVIDUAL | OWN_BALL | STROKE_TOTAL | `{handicapSystem:"PEORIA"}` — handicap engine concern, not scoring |

**Observation:** every PGCC format fits the triple cleanly. No format
needs a fourth axis. Even Ryder Cup decomposes — its sessions are
sub-tournaments with their own (shape, ball, scoring) triple.

---

## 4 · Migration plan (additive)

### Migration 151 (this PR)

```sql
ALTER TABLE league_tournaments
  ADD COLUMN team_shape         VARCHAR(24) NULL,
  ADD COLUMN ball_rule          VARCHAR(24) NULL,
  ADD COLUMN scoring_method     VARCHAR(24) NULL,
  ADD COLUMN format_params_json JSON        NULL;

-- Backfill team_shape from the 2 dimensions that encode "team-ness":
--   tournament_type=TEAM_MATCH_PLAY → MULTI_TEAM_PAIRINGS (Ryder Cup)
--   scoring_mode IN ('SCRAMBLE','BEST_BALL','ALTERNATE_SHOT') →
--     team event; default to FOUR_PLAYER (most common) — owners can
--     refine via the new admin field once read paths cut over
--   everything else → INDIVIDUAL
UPDATE league_tournaments SET team_shape = CASE
  WHEN tournament_type = 'TEAM_MATCH_PLAY' THEN 'MULTI_TEAM_PAIRINGS'
  WHEN scoring_mode IN ('SCRAMBLE','BEST_BALL','ALTERNATE_SHOT') THEN 'FOUR_PLAYER'
  ELSE 'INDIVIDUAL'
END;

-- Backfill ball_rule from scoring_mode (the existing enum already
-- encodes this axis cleanly — INDIVIDUAL/BEST_BALL/MATCH_PLAY all
-- map to OWN_BALL because they play their own balls)
UPDATE league_tournaments SET ball_rule = CASE
  WHEN scoring_mode = 'SCRAMBLE' THEN 'SCRAMBLE'
  WHEN scoring_mode = 'ALTERNATE_SHOT' THEN 'ALTERNATE_SHOT'
  ELSE 'OWN_BALL'  -- INDIVIDUAL, BEST_BALL, MATCH_PLAY, NULL
END;

-- Backfill scoring_method. Priority: Ryder Cup team_scoring_mode wins
-- (it's the most-recently-added column and the most specific), then
-- scoring_system for individual Stableford/Chicago, then
-- scoring_mode=MATCH_PLAY for casual head-to-head, else STROKE_TOTAL.
UPDATE league_tournaments SET scoring_method = CASE
  WHEN tournament_type = 'TEAM_MATCH_PLAY' AND team_scoring_mode = 'MATCH_PLAY'
    THEN 'MATCH_PLAY'
  WHEN tournament_type = 'TEAM_MATCH_PLAY' AND team_scoring_mode = 'BEST_K_BY_HOLE_RANGE'
    THEN 'BEST_K_OF_N_PER_HOLE'
  WHEN scoring_system = 'STABLEFORD' OR scoring_system = 'MODIFIED_STABLEFORD'
    THEN 'STABLEFORD'
  WHEN scoring_system = 'CHICAGO'
    THEN 'QUOTA'
  WHEN scoring_mode = 'MATCH_PLAY'
    THEN 'MATCH_PLAY'
  ELSE 'STROKE_TOTAL'  -- includes scoring_system=STROKE and NULL
END;

-- Backfill format_params_json — only Stableford has a populated param
-- bag today (the custom point table). Chicago/Modified Stableford
-- have defaults in code so an empty NULL is correct; the strategy
-- reads the ScoringSystem variant from the (now-deprecated) column
-- during cutover and from format_params_json post-cutover.
UPDATE league_tournaments SET format_params_json = CASE
  WHEN custom_stableford_points IS NOT NULL THEN
    JSON_OBJECT(
      'pointsTable', custom_stableford_points,
      'variant', scoring_system  -- STABLEFORD vs MODIFIED_STABLEFORD
    )
  WHEN scoring_system = 'MODIFIED_STABLEFORD' THEN
    JSON_OBJECT('variant', 'MODIFIED_STABLEFORD')
  WHEN scoring_system = 'CHICAGO' THEN
    JSON_OBJECT('target', 39, 'negativeStart', false)
  ELSE NULL
END;

-- After backfill, lock the 3 enum columns as NOT NULL
ALTER TABLE league_tournaments
  MODIFY COLUMN team_shape     VARCHAR(24) NOT NULL,
  MODIFY COLUMN ball_rule      VARCHAR(24) NOT NULL,
  MODIFY COLUMN scoring_method VARCHAR(24) NOT NULL;
```

**Old columns stay** (`tournament_type`, `team_scoring_mode`,
`scoring_mode`, `scoring_system`, `custom_stableford_points`).
Nothing reads the 4 new columns yet. This migration is safely
reversible (drop the 4 new columns).

**Important — `Round.scoring_mode` is NOT touched.** This refactor is
scoped to `league_tournaments`. Casual rounds (`Round` entity) keep
their existing `scoring_mode` column unchanged; that's a separate
surface with its own evolution path.

### Subsequent migrations (NOT this PR)

- **Mig 152+**: progressively migrate `LeagueService` / `RyderCupService` / `TeamScoringModeRules` to read from the new columns. Each PR cuts over one read path; the corresponding write path duplicates to both old + new columns for the duration of the cutover.
- **Mig N (later sprint)**: drop the old columns once nothing reads them. Probably 3-6 weeks after mig 151.

---

## 5 · What this enables

Adding **Scramble** post-refactor:

1. No schema change.
2. Admin UI: dropdown gains "Scramble" → writes `(team_shape=FOUR_PLAYER, ball_rule=SCRAMBLE, scoring_method=STROKE_TOTAL)`.
3. New `ScrambleScoringStrategy` bean (or reuse `StrokeTotalScoringStrategy` since the scoring math is identical to stroke — the *ball rule* changes, but the per-hole stroke count still aggregates the same way).
4. Pairings page gets one new info card ("scramble: tee, pick best, all play next from there"); leaderboard is unchanged.

Adding **Skins**:

1. New `SkinsScoringStrategy` bean. Reads `format_params_json.skinValueCents` and `format_params_json.carryover`.
2. New `SkinsLeaderboardResponse` DTO (different shape than stroke — shows skins-per-hole, carryover state).
3. No `LeagueService` changes; the strategy bean encapsulates everything.

The point of the refactor isn't "we can ship Scramble next week" — it's that **future formats are additive (one bean + one row in a dropdown), not surgical (modify LeagueService + add a column + migrate every consumer)**.

---

## 6 · What breaks if we ship just mig 151

Nothing. Every existing read path uses one of the 5 legacy columns
(`tournament_type`, `team_scoring_mode`, `scoring_mode`,
`scoring_system`, `custom_stableford_points`). The 4 new columns are
populated but unread.

Risks to watch during the multi-PR cutover:
1. **`Round.scoring_mode` confusion** — same column name on a different
   table. The refactor is `league_tournaments`-scoped; `RoundService`
   should be left alone in mig 152+.
2. **Stableford variant fidelity** — the backfill stuffs the variant
   into `format_params_json.variant`. Cutover PR has to read it back
   correctly or Modified-Stableford rounds will silently downgrade to
   standard Stableford.
3. **C2 Adopt is live in 5 weeks** (2026-09-14). The 4-3-2-1-1-1 BSG
   pipeline + Stableford pipeline both run real tournaments today.
   Cut over the BSG read path LAST and only after a full BSG-cluster
   test pass on a staging dataset.

---

## 7 · What the code-level refactor looks like (post-mig 151)

`TournamentFormatStrategy` bean per format (or per scoring method):

```java
public interface TournamentFormatStrategy {
  ScoringMethod scoringMethod();   // which scoring_method values this handles
  LeaderboardResponse buildLeaderboard(LeagueTournament t, List<Scorecard> cards);
  ValidationResult validateParams(JsonNode formatParams);
}
```

Existing `TeamScoringModeRules` becomes `BestKOfNScoringStrategy`
(it's already cohesive per the 2026-05-13 refactor — line 1139 of
BACKLOG completed work). `StablefordPointsCodec` moves inside
`StablefordScoringStrategy`.

Selection at runtime: `formatStrategyRegistry.forMethod(t.getScoringMethod()).buildLeaderboard(...)`.

---

## 8 · Resolved design decisions

Originally drafted as open questions; resolved 2026-05-16 per the leans below.

1. **Handicap engine columns stay as columns.** Peoria, off-the-low, `handicap_allowance_pct`, `hole_score_cap_mode` apply across formats — moving them into `format_params_json` would force every Stableford row to duplicate them and would lose SQL-level filtering. They stay on `LeagueTournament` as first-class columns. The format-strategy layer reads them directly.

2. **Ryder Cup sessions inherit + override.** A session row carries only the axes that differ from its parent tournament. The session DTO computes the effective triple as `(parent.team_shape, session.ball_rule ?? parent.ball_rule, session.scoring_method ?? parent.scoring_method)`. Keeps the data model flat — no triple-duplication per session — and matches how `LeagueTournamentMatch` already inherits most context from its parent.

3. **`STABLEFORD` is one scoring method; Quota and Chicago are param flavors.** All three share the same per-hole points math. The leaderboard "wins" rule differs: Stableford = highest total wins, Quota = first to hit `target` wins, Chicago = vs.-target ranking. Encode as: `scoring_method='STABLEFORD'`, `format_params_json={pointsTable, target?, negativeStart?, variant?}`. One strategy bean handles all three. `MODIFIED_STABLEFORD` is also a Stableford with a custom `pointsTable` — same method, different params.

4. **Customer-facing format label = derived function + optional override column.** A `displayFormatName(team_shape, ball_rule, scoring_method, format_params_json)` helper returns sane defaults ("4-Player Scramble", "Stableford Foursomes"). When the auto-name reads awkwardly, owners override via a new optional `display_format_name VARCHAR(64) NULL` column on `LeagueTournament`. **Not in scope for mig 151** — added in a later migration alongside the first format that needs an override.
