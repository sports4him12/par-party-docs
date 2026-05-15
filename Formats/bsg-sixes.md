# Design: BSG Sixes

_Captured 2026-05-15 from a BSG-provided format infographic ("BSG Sixes: Four Players. Three Rotations. One Champion.")._

## What the format actually is

4 players (A, B, C, D) play one round together. **Initial partner draw is random.** Pairs rotate every 6 holes so every player partners with every other player exactly once across the round:

| Holes | Teams |
|---|---|
| 1–6 | **A & B** vs. **C & D** |
| 7–12 | **A & C** vs. **B & D** |
| 13–18 | **A & D** vs. **B & C** |

**Per-hole scoring** is "Combined Net Score": each pair adds their two net scores together. **Lowest combined total wins the hole.**

> Example: A net 3 + B net 4 = Team A&B total 7. C net 5 + D net 6 = Team C&D total 11. Team A&B wins the hole.

**Per-hole points distribution** (per player):

| Outcome | Points to each player on winning team | Points to each player on losing team |
|---|---|---|
| **Outright win** | 3 | 0 |
| **Tie** | 1 | 1 (all 4 players) |
| **Loss** | 0 | — |

Points sum to **6 per hole** when there's a winner (3+3), or **4 per hole** when tied (1×4). Tie is slightly underweighted vs. zero-sum — by design, "nobody wins getting zero."

**Cumulative leaderboard** after 18 holes: 4 individual point totals. **Higher wins.** Range is 0–54 (perfect: 18 holes × 3 points), realistic 15–30.

**The POPS Protocol** — distinctive social rule, also load-bearing for the scoring:

- Every player must know **every other player's** handicap strokes (pops) on the current hole.
- **Before anyone tees**, the pops for that hole get announced clearly: "POPS ON 7: A=1, B=0, C=2, D=1."
- **Silence Penalty**: failure to communicate → "relentless berating." The card-keeper has no excuse for not knowing the state of play.

## What architecturally maps to what we already built

| Concept | Existing mechanism | Fit |
|---|---|---|
| 4 players in one foursome | `LeagueTournamentPairing` (foursome groupings exist) | ✅ exact reuse |
| Net per-hole math per player | `LiveScoringService.computePerHoleNet` | ✅ already computes net-per-player-per-hole |
| Designated scorer (one player enters for the group) | `team.scorer_user_id` + `submitHoleScore(targetUserId)` | ✅ exact reuse — BSG Sixes needs a scorer per foursome same as 4-3-2 |
| Per-hole partner rotation | _new_ | 🚫 brand-new concept |
| Per-hole 3/1/0 points distribution | _new_ | 🚫 the engine math is novel |
| Per-player cumulative points leaderboard | `TeamLeaderboardResponse.matchPoints` field already exists for MATCH_PLAY | ⚠ same shape, different population logic |
| Pre-tee POPS announcement banner | _none_ | 🚫 new mobile UI element |

## The honest gap: this is "pairs-rotate-by-block + points-per-hole"

Every other format in the codebase computes either:
- **Per-player gross/net** (STROKE, Stableford, individual scoring) — no team grouping
- **Team total** that aggregates 2–4 players' scores into one number (BSG 4-3-2, MATCH_PLAY, Three-Headed Monster)
- **Per-team match points** that accrue across holes (MATCH_PLAY 1/0.5/0)

BSG Sixes is **a hybrid**: per-hole it's team-vs-team (sum of 2 nets), but **the teams change every 6 holes**, and the leaderboard is **per-player points** (not per-team). The points awarded depend on which team you were on for that hole.

It's closest to a **Stableford team variant** but with:
- Dynamic teaming (B is on A's team for holes 1-6, on D's team for holes 7-12)
- A 3/1/0 grid instead of standard Stableford points

## Two implementation options

### Option A — New scoring mode: `SIXES_ROTATING_PAIRS`

A new `teamScoringMode` value with engine logic that:
1. Reads a per-tournament "rotation table" (3 rows: hole-from, hole-to, team-mapping)
2. For each hole, looks up which 2 players are on each side
3. Sums each pair's net scores, awards 3/1/0 per player based on which side won
4. Cumulative per-player points = leaderboard, descending

The "rotation table" is a small new table:

```sql
CREATE TABLE league_tournament_sixes_rotations (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tournament_id BIGINT NOT NULL,
    group_pairing_id BIGINT NOT NULL,  -- which foursome
    block_number INT NOT NULL,         -- 1, 2, 3 (the 3 rotations)
    from_hole INT NOT NULL,
    to_hole INT NOT NULL,
    team_a_user_ids JSON NOT NULL,     -- [userIdA, userIdB]
    team_b_user_ids JSON NOT NULL,     -- [userIdC, userIdD]
    UNIQUE KEY uq_sixes_block (tournament_id, group_pairing_id, block_number),
    INDEX idx_sixes_tournament (tournament_id)
);
```

**Pros**: explicit data model. Rotation is set at tournament-create time (admin or random-draw button), then immutable for the round. Engine has one job: read the rotation, compute points per hole.

**Cons**: more schema work than reusing existing tables.

### Option B — Reuse `LeagueTournamentHoleGrouping` rows with embedded team partnerships

Stuff the 3 rotation blocks into existing hole-groupings rows: `{ fromHole: 1, toHole: 6, formatMode: 'SIXES', teamAJson: '[1,2]', teamBJson: '[3,4]' }`.

**Pros**: zero new tables.

**Cons**: overloads the hole-groupings concept. BSG 4-3-2 hole-groupings is "how many of N nets count" — Sixes is "which 2 of 4 are partners." Different concept, awkwardly compressed.

**Recommendation: Option A.** The rotation table is clean and lets the admin pre-compute the random draw into stable data. Future BSG-Sixes variants (e.g., 5-player Sixes with a sit-out rotation) can extend the same table.

## Migration sketch

```sql
-- (TBD migration number, will depend on what ships first.)

-- 1. New rotation table for Sixes-style pair shuffling.
CREATE TABLE league_tournament_sixes_rotations (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tournament_id BIGINT NOT NULL,
    group_pairing_id BIGINT NOT NULL,
    block_number INT NOT NULL,
    from_hole INT NOT NULL,
    to_hole INT NOT NULL,
    team_a_user_ids JSON NOT NULL,
    team_b_user_ids JSON NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uq_sixes_block (tournament_id, group_pairing_id, block_number),
    INDEX idx_sixes_tournament (tournament_id),
    FOREIGN KEY (tournament_id) REFERENCES league_tournaments(id) ON DELETE CASCADE,
    FOREIGN KEY (group_pairing_id) REFERENCES league_tournament_pairings(id) ON DELETE CASCADE
);

-- 2. New tournament-level config knobs.
ALTER TABLE league_tournaments
    ADD COLUMN sixes_win_points INT NULL,    -- default 3
    ADD COLUMN sixes_tie_points INT NULL,    -- default 1
    ADD COLUMN sixes_loss_points INT NULL;   -- default 0
```

(Use the actual table name `league_tournaments` — plural — per the verify-table-names-in-migrations memory.)

## What the scorer interaction looks like on mobile

Reuses 95% of BSG 4-3-2 scorer flow:

1. **Tournament home → scorer screen** identical to BSG: stepper per player, foursome scorer model unchanged.
2. **Per-hole top banner** changes contextually: "Hole 7 · A & C vs. B & D · POPS: A=1, B=0, C=2, D=1." The pops are auto-derived from each player's course handicap + the hole's stroke index.
3. **Live points readout** below the steppers, updates as strokes change: "A&C team total 8 · B&D team total 9 → A&C wins → A +3, C +3, B 0, D 0."
4. **Per-hole-3/1/0 column on the leaderboard** instead of "+/- to par."

POPS banner is **new UI** but trivially derived from already-existing data (player handicap + per-tee SI). Pre-tee surfacing is the only novel mobile work.

## What the leaderboard shows

- 4 players, descending by total points across 18 holes
- Highlighting the current leader visually
- Per-hole breakdown: "Hole 7 · A&C 8 vs B&D 9 · winners +3, losers 0"
- A "Partner history" view: "You played with B (1-6), C (7-12), D (13-18)" so the player can replay their own arc

## What's missing from the source image (questions for the organizer)

The infographic is dense but a few load-bearing details are absent:

1. **Random draw rule**: who decides A/B/C/D? Sorted by handicap? Drawn from a hat? App-generated random? Just "names listed alphabetically"?
2. **Tiebreaker** when 2 players end the round on the same point total: USGA-standard back-9/back-6/back-3 chain? Sudden-death extra hole? Lower combined-team-margin across the 18 holes?
3. **Stroke index source**: per-tee SI (recently shipped via the per-league-course-overrides flow) or flat tournament SI? Important because mixed-tee groups will have different "pops on hole X" per player.
4. **Group composition**: 16 players in 4 groups (per parent BSG Majors series). Are points compared **within each group of 4 only**, or **across the field**? Cross-group points have a different statistical meaning since each player is in a different competitive frame.
5. **Does the silence penalty enforce anything in the app**? Probably no — it's a social rule. But if the app surfaces the POPS announcement automatically, maybe the "silence penalty" disappears as a social construct? Worth asking whether the BSG group _wants_ that.

## Can a live v1.3.2 mobile player participate today (if we shipped web+API now)?

**No, not as a full player experience.** Per the analysis in the parent conversation: v1.3.2 lacks the format badge, format explainer, per-hole subtitle, foursome scorer mode, team/points leaderboard, and read-only banner — all of which shipped in v1.3.3 or v1.3.4. The score-entry mechanics work on v1.3.2 (steppers + save), and the server can still compute the Sixes leaderboard, but the player has no way to see who they're partnered with this hole, who's winning, or what the live points are.

**Recommendation**: don't run BSG Sixes for tournaments where players are on v1.3.2. Wait for the v1.3.4 store release, or treat the May 2 event as a paper-card backup with the app as scorecard-only.

The POPS pre-announcement banner is novel and ships in **v1.3.5 or later** — none of the existing builds have a place for it. For early Sixes events, the POPS protocol stays a verbal social rule.

## Scope of work to implement

| Piece | Estimate | Required for first BSG Sixes event |
|---|---|---|
| Migration: `league_tournament_sixes_rotations` table + 3 points columns | 30 min | Yes |
| `SixesRotatingPairsScoring` pure-function engine + tests | 2-3 hr | Yes |
| Service wire: `RyderCupService.buildSixesLeaderboard` | 1.5 hr | Yes |
| New tournament-type / scoring-mode enum value `SIXES_ROTATING_PAIRS` | 30 min | Yes |
| Random-draw UI (admin / tournament creator generates the 3 rotation blocks for each foursome) | 2 hr | Yes |
| Web leaderboard view (per-player points, partner history, per-hole breakdown) | 2 hr | Yes |
| Mobile: per-hole top banner with current pair + POPS readout | 2 hr | Yes |
| Mobile: live points readout in scorer mode | 1 hr | Yes |
| Mobile: leaderboard mirror (per-player points, descending) | 1 hr | Yes |
| Per-tournament 3/1/0 point config UI | 30 min | Nice-to-have (defaults work for BSG) |
| POPS pre-announcement banner (auto-derived from handicap + SI) | 2 hr | **Maybe** — could ship as v1 without if verbal protocol stays the rule |
| Season aggregation hooks (3 best of 5 normalization) | 0 hr in Sixes-specific scope | Shared problem with Three-Headed Monster + Wolf |

**Bottom line: ~12-13 hours of focused work** to ship BSG Sixes. The 3/1/0 math is simpler than Wolf's payout grid, and the rotation is more constrained than Three-Headed Monster's multi-format game — but the **POPS banner is novel and the random-draw UI is novel**. Both are small but new.

## Open decisions

- **Option A (new rotation table) vs. B (reuse hole-groupings)**. Default to **Option A** — cleaner separation, and BSG-Sixes-specific data deserves its own table.
- **Does the POPS banner ship in v1 or v2?** Default to v2 — the format works without it (BSG players already announce verbally), and the banner is the only piece that doesn't have an existing UI surface to extend.
- **Within-group vs. cross-group leaderboard**: same question as Wolf. Per-group is the safer default; cross-group ranking requires a normalization decision.
- **POPS auto-derivation**: per-tee SI vs. flat SI. Defer until per-tee SI is universal on baseline courses (currently 3 of 4 BSG courses have it).

## Cross-references

- BSG 4-3-2 engine: `golfsync-api/src/main/java/com/golfsync/service/scoring/BestKOfNPerHoleTeamScoring.java`
- Sibling design docs:
  - `golfsync-docs/Formats/three-headed-monster.md`
  - `golfsync-docs/Formats/wolf.md`
- LeagueTournamentPairing (4-player groupings the rotation reads from): exists today
- Per-hole net engine: `LiveScoringService.computePerHoleNet` — already handles allowance % + cap mode
- Per-tee SI: migration 124 + per-league overlay migration 131
- Foursome scorer flow: migration 135
