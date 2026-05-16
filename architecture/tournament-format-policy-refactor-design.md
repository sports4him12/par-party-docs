# TournamentFormatPolicy Refactor — Design Doc

**Date:** 2026-05-15
**Status:** Proposed (awaiting approval)
**Companion to:** `solid-audit-2026-05-15.md`, `refactor-readiness-2026-05-15.md`
**Scope:** A scoped refactor of `LeagueService`'s format-driven branching to defang the OCP pressure as new tournament formats land. NOT the full `LeagueTournamentService` extraction.

## The problem (restated from the founder's framing)

> "I'm personally most concerned about the leagues as I foresee more and more formats to come and it getting messier without a refactor."

Concrete: the [BSG Sixes / Three-Headed Monster / Wolf format docs](../Formats/) point at additional formats incoming. Each format that lands today edits `LeagueService` directly — adding a new branch to `forceShowNetForFormat`, a new gate in `requireBestKEntitlement`, a new normalizer if it has its own pooling mode, a new persistence wrinkle if it has tournament-specific config (like Stableford's `customStablefordPoints`).

The SOLID audit named this as the largest OCP smell in the layer.

## What the SOLID audit assumed vs. what the code actually is

The audit sketched:

> Extract `LeagueTournamentFormatPolicy` — strategy interface + per-format impls (Stroke / Stableford / BSG / RyderCup) absorbing `forceShowNetForFormat`, `requireBestKEntitlement`, `normalizeBsgPoolingMode`, `serializeCustomStablefordPoints`. ~150 LOC across impls.

That assumes a **single dispatch axis** — the tournament's "format" picks one strategy. The actual code has **four dispatch axes** that intersect:

1. `format` (string, free-form-ish: e.g. `INDIVIDUAL`, `SCRAMBLE`, `BEST_BALL`, `ALT_SHOT`, `MATCH_PLAY`)
2. `scoringSystem` (enum: STROKE / STABLEFORD / MODIFIED_STABLEFORD / CHICAGO)
3. `scoringMode` (enum: GROSS / NET)
4. `teamScoringMode` (string: STROKE_TOTAL / MATCH_PLAY / BEST_K_BY_HOLE_RANGE)

The format-specific behaviors don't dispatch on a single axis:

| Behavior | Triggered by |
|---|---|
| `requireBestKEntitlement` | `teamScoringMode == BEST_K_BY_HOLE_RANGE` |
| `forceShowNetForFormat` | `teamScoringMode == BEST_K_BY_HOLE_RANGE` |
| `normalizeBsgPoolingMode` | Always called on the request; only meaningful when teamScoringMode is BSG-shaped |
| `normalizeHandicapAllowancePct`, `normalizeHoleScoreCapMode`, `normalizeMaxHandicapCap` | Always called; meaningful for BSG team play but stored on every tournament |
| `serializeCustomStablefordPoints` | Triggered by **presence of `customStablefordPoints` in the request** (implicit Stableford signal); persisted regardless of `scoringSystem` value |
| `lowHandicapReference` | Stored only; consulted by team-tournament scoring strategies elsewhere |

So a clean Strategy-pattern interface like `interface TournamentFormatPolicy { boolean forceShowNet(); ... } @Component class StablefordPolicy implements TournamentFormatPolicy { ... }` would have to **multi-dispatch** on `(format, scoringSystem, teamScoringMode)` together. There is no single key. Worse, some behaviors (`serializeCustomStablefordPoints`) trigger on **request shape**, not on a stored format value.

A single-axis Strategy pattern here would be an abstraction fighting the data.

## The real shape of the code (what the four axes actually do)

After reading [LeagueService.java:1010-1100](../../golfsync-api/src/main/java/com/golfsync/service/LeagueService.java) (createTournament) and [1620-1715](../../golfsync-api/src/main/java/com/golfsync/service/LeagueService.java) (updateTournament) plus the format helpers at [1987-2206](../../golfsync-api/src/main/java/com/golfsync/service/LeagueService.java), the format-related methods cluster into **three cohesive groups**, each with its own dispatch shape:

### Group A — Generic input parsers (NOT format-specific)
Methods that just translate wire strings into enums. Used by every tournament regardless of format.

- `parseScoringSystem(String)` → `ScoringSystem` enum or null
- `parseScoringMode(String)` → `ScoringMode` enum or null
- `parseHandicapAllowance(String)` → `HandicapAllowance` enum (default NET)
- `normalizeTournamentType(String)` → INDIVIDUAL / TEAM_MATCH_PLAY string
- `normalizeTeamScoringMode(String)` → STROKE_TOTAL / MATCH_PLAY / BEST_K_BY_HOLE_RANGE string

**Verdict:** These don't belong in a "format policy" — they're request-shape validators. Leave them in `LeagueService` (or extract to a smaller `TournamentRequestParser` later if size warrants it).

### Group B — BSG-shaped rules (cohesive cluster around `BEST_K_BY_HOLE_RANGE`)
Methods that all relate to the BSG (Best K of N team scoring) format family — entitlement gate, default forcing, pooling mode, BSG-specific cap fields.

- `requireBestKEntitlement(Long leagueId, String teamScoringMode)` — gates feature flag
- `forceShowNetForFormat(request, rawTeamMode)` — BSG forces showNet=true
- `normalizeBsgPoolingMode(String)` — BSG-specific column
- `normalizeHandicapAllowancePct(Integer)` — BSG-literal default 100
- `normalizeHoleScoreCapMode(String)` — BSG-literal default NONE
- `normalizeMaxHandicapCap(Integer)` — BSG-shaped (1..54 USGA range)
- `resolveTeamScoringMode(leagueId, raw)` — convenience: normalize + entitlement gate

**Verdict:** This IS a cohesive module — the methods all relate to the team-scoring-mode dimension. **Extract as `TeamScoringModeRules` (or `BsgTeamRules`).** Adding the next BSG variant (BSG Sixes per the docs) becomes:
- New enum value on `BsgPoolingMode` (or a new field) — single edit
- Possibly extend `requireBestKEntitlement` to gate the new variant — single edit
- All in one cohesive file, **not in `LeagueService`**.

### Group C — Stableford persistence helpers (cohesive cluster around custom point tables)
Methods that translate the wire `StablefordPointsInput` to/from the stored JSON column.

- `serializeCustomStablefordPoints(StablefordPointsInput)` → JSON string or null (with all-zero-collapses-to-null business rule)
- `toCustomStablefordPointsView(String)` → response DTO or null

**Verdict:** This IS a cohesive module — wire ↔ JSON translation for one feature. **Extract as `StablefordPointsCodec` (or similar).** Adding a Modified Stableford custom-point variant in the future becomes editing this one file.

## The proposed refactor

Two new Spring `@Component` beans, each cohesive and single-purpose:

### 1. `TeamScoringModeRules`
```java
package com.golfsync.service.tournament;

@Component
public class TeamScoringModeRules {
    private final FeatureEntitlementService featureEntitlementService;

    public TeamScoringModeRules(FeatureEntitlementService featureEntitlementService) { ... }

    /** Validate wire string, default to STROKE_TOTAL on null/blank. */
    public String normalize(String raw) { ... }

    /** Throw IAE if BSG-gated mode used by an unentitled league. No-op otherwise. */
    public void requireEntitlement(Long leagueId, String normalizedMode) { ... }

    /** Convenience: normalize + entitlement gate in one call. */
    public String resolve(Long leagueId, String raw) { ... }

    /** True iff this mode requires net scoring to be meaningful. */
    public boolean forcesShowNet(String normalizedMode) { ... }

    /** Apply showNet defaulting to a create request based on the mode. */
    public boolean resolveShowNet(String normalizedMode, Boolean requestShowNet) { ... }

    /** Validate + default the BSG pooling mode column. */
    public BsgPoolingMode normalizePoolingMode(String raw) { ... }

    /** Validate + default the handicap allowance percentage (1..100, default 100). */
    public Integer normalizeHandicapAllowancePct(Integer raw) { ... }

    /** Validate + default the hole-score cap mode. */
    public HoleScoreCapMode normalizeHoleScoreCapMode(String raw) { ... }

    /** Validate the max-handicap cap (1..54, null = no cap). */
    public Integer normalizeMaxHandicapCap(Integer raw) { ... }
}
```

**LOC estimate:** ~120 LOC + ~60 LOC unit test. Methods just lift verbatim from LeagueService.

### 2. `StablefordPointsCodec`
```java
package com.golfsync.service.tournament;

@Component
public class StablefordPointsCodec {
    /** Wire input → JSON for persistence. NULL/all-zero → NULL (defaults apply). */
    public String serialize(CreateLeagueTournamentRequest.StablefordPointsInput in) { ... }

    /** Persisted JSON → response view. NULL preserved. */
    public LeagueTournamentResponse.StablefordPointsView toView(String json) { ... }
}
```

**LOC estimate:** ~50 LOC + ~40 LOC unit test.

### What changes in LeagueService

Two new fields injected. Method bodies in `createTournament` and `updateTournament` become:

```java
.teamScoringMode(teamScoringModeRules.resolve(leagueId, request.getTeamScoringMode()))
.showNet(teamScoringModeRules.resolveShowNet(
    teamScoringModeRules.normalize(request.getTeamScoringMode()),
    request.getShowNet()))
.bsgPoolingMode(teamScoringModeRules.normalizePoolingMode(request.getBsgPoolingMode()))
.handicapAllowancePct(teamScoringModeRules.normalizeHandicapAllowancePct(request.getHandicapAllowancePct()))
.holeScoreCapMode(teamScoringModeRules.normalizeHoleScoreCapMode(request.getHoleScoreCapMode()))
.maxHandicapCap(teamScoringModeRules.normalizeMaxHandicapCap(request.getMaxHandicapCap()))
.customStablefordPoints(stablefordPointsCodec.serialize(request.getCustomStablefordPoints()))
```

The 9 helper methods migrate out of `LeagueService` entirely (~140 LOC removed). LeagueService net LOC change: **-100** (removed methods exceed added field declarations + delegation lines).

### What does NOT change

- `parseScoringSystem`, `parseScoringMode`, `parseHandicapAllowance`, `normalizeTournamentType` stay in LeagueService — they're generic parsers, not format policy.
- `validateTournamentBody` stays — generic body validation, multi-course rules.
- `forceShowNetForFormat` (the original method) is **deleted** — replaced by `teamScoringModeRules.resolveShowNet`.
- `resolveTeamScoringMode` (the convenience method) is **deleted** — replaced by `teamScoringModeRules.resolve`.
- `LeagueTournament` entity unchanged.
- `CreateLeagueTournamentRequest` DTO unchanged.
- All controller endpoints unchanged.
- All response DTOs unchanged.

## What this DOESN'T do

This refactor is **deliberately scoped** to the format-branching problem:

- ❌ Does NOT extract `LeagueTournamentService` from `LeagueService` (that's the audit's bigger refactor #3, deferred)
- ❌ Does NOT introduce a polymorphic `TournamentFormatPolicy` interface with per-format implementations (the code's dispatch shape doesn't support it cleanly)
- ❌ Does NOT touch `LiveScoringService`, `RyderCupService`, scoring strategies, or DTOs
- ❌ Does NOT change behavior — every method moves verbatim

If a future format DOES warrant a polymorphic strategy (e.g. multiple BSG variants needing different `forceShowNet` rules), `TeamScoringModeRules` is the right home to grow it inside — adding a `Map<String, BsgVariantPolicy>` for instance — without disturbing `LeagueService`.

## Why this is the right shape

- **Solves the founder's stated concern.** Adding a new BSG variant becomes editing `TeamScoringModeRules` (cohesive, ~120 LOC) instead of `LeagueService` (3,201 LOC, 30+ injected deps).
- **Honest to the data.** Two cohesive clusters (BSG-shaped rules + Stableford persistence) become two cohesive classes. No invented polymorphism.
- **Low risk.** Pure code movement. No behavior change, no DTO change, no DB change. Existing `LeagueServiceTest` tests should continue to pass without modification (LeagueService still wires the calls; the methods just live elsewhere).
- **Testability win.** `TeamScoringModeRules` can be unit-tested without 23 mock dependencies. The 7 minimum-safety tests called for in `refactor-readiness-2026-05-15.md` (D.1 #1, #2) become tests against the new beans, not against a behemoth class.
- **Sets a precedent.** When the next cluster of format-related methods accumulates (e.g. "round-format rules" for Round side), there's a pattern to follow: `RoundFormatRules` bean.

## Test plan (to be executed AFTER design approval)

Run the existing 182-test `LeagueServiceTest` BEFORE refactor — establishes baseline (must all pass).

Then write **before** the production code changes:

| Test class | Tests | Effort |
|---|---|---|
| `TeamScoringModeRulesTest` | 8-10 small tests covering each method's defaults, throws, and BSG entitlement gate | small |
| `StablefordPointsCodecTest` | 4-5 tests covering serialize round-trip, all-zero collapse, null preservation, view conversion | small |

Then execute the refactor (move methods, add fields, update delegations).

Then verify:
- Every `LeagueServiceTest` still passes — proves no behavior regression
- Every new test passes
- `mvn test` (full suite) clean

If any existing test breaks, **stop and ask** — that's a real behavior delta, not a refactor mechanic.

## Effort + risk

- **Design approval review:** ~10 min (this doc)
- **Write minimum-safety tests:** ~1 hour (12-15 small tests against the to-be-extracted beans, exercising current methods first to lock in behavior)
- **Execute the refactor:** ~1 hour (mechanical move of 9 methods)
- **Verify + commit:** ~30 min
- **Total:** ~3 hours, single PR.

**Risk:** Low. Pure code movement, no behavior change. The only real failure mode is a test that depended on a `private` method via reflection — none observed in the existing test files.

## Decision

Approve this design? Two options if not:

1. **Smaller still:** Extract only `TeamScoringModeRules`; leave `serializeCustomStablefordPoints` in LeagueService for now.
2. **Different shape:** Reject these two beans in favor of something else (e.g. enrich `LeagueTournament` entity with the policy methods, or create a `TournamentFactory` that owns the entire builder chain).

---

*Design produced by Claude (Opus 4.7, 1M context) during a 2026-05-15 working session, after reading the actual format-branching surface to ground the proposal in the code's real shape.*
