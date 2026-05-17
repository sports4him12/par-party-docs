# Live Scorecard for Casual Round Bookings — Design

**Date:** 2026-05-16
**Status:** Draft (pre-coding); open questions in §F
**Owner:** Ryan

## Goal

Bring the league live-scorecard experience to regular (non-league) round bookings: when a member books a tee time in the app, anyone in the group can update hole-by-hole scores during the round, and the leaderboard updates for everyone. Includes non-app players (guests) — the booker can add guests by name and enter their scores.

## Locked decisions

| Decision | Choice | Notes |
|---|---|---|
| Permissions | **Self + booker edits all** | A player edits their own card; the round creator can edit anyone's. Other players cannot edit each other. |
| Guests | **Name-only on `RoundPlayer` (nullable `user_id`)** | First-class participants. `HoleScore` re-keyed to `round_player_id`. |
| Platform | **Mobile first**, web read-only this phase | On-course use case lives on a phone. |
| Course data | **Shared `Course` if available; otherwise typed-in or OCR'd, saved on the round** | Ad-hoc course data is saved permanently on the round for that round's reuse, but **never** promoted to the shared `Course` catalog automatically. To get a course into the shared catalog, the user requests it from a GolfSync admin (separate manual flow, out of scope here). |

## What's already there (don't rebuild)

- `Round`, `RoundPlayer` (already has `score`, `scorePostedAt`, `teePlayed`, `courseRating`, `courseSlope`, `playerHandicapIndex`), and `HoleScore` (already has strokes/par/GIR/putts/penalty/fairway/yardages/conceded)
- `RoundLeaderboardController` + `RoundLeaderboardResponse` with ETag caching; mobile already polls every 6s on `golfsync-mobile/app/rounds/[id]/leaderboard.tsx`
- OCR: `POST /api/scorecards/ocr` returns `ParsedScorecardResponse` (in-memory, not persisted); `PUT /api/rounds/{id}/scorecard` already persists per-hole strokes + pars
- `Course` + `CourseTee` + `CourseTeeHolePar` for shared per-hole pars/yardages/SI
- `SaveScorecardRequest.forUserId` — partial scorekeeper plumbing; today only checks both parties are participants, doesn't yet enforce self-or-booker

## A. Data model changes

Ad-hoc course data is saved on the **round**, not in `Course`. Read order for pars/SI:

1. `Round.course_id` set → `Course.hole_pars` / `CourseTeeHolePar`
2. Else → `Round.hole_pars_json` / `Round.hole_handicaps_json`
3. Else → `hole_scores.par` per row (last-resort)

### Migration 153 — `153-round-player-guests.sql`
```sql
ALTER TABLE round_players MODIFY user_id BIGINT NULL;
ALTER TABLE round_players ADD COLUMN guest_name VARCHAR(80) NULL;
```
App-level invariant: exactly one of `user_id` / `guest_name` is set per row.

### Migration 154 — `154-hole-scores-round-player-fk.sql` (additive, nullable)
```sql
ALTER TABLE hole_scores ADD COLUMN round_player_id BIGINT NULL;
CREATE INDEX idx_hole_scores_rp_hole ON hole_scores(round_player_id, hole_number);
UPDATE hole_scores hs
  JOIN round_players rp ON rp.round_id = hs.round_id AND rp.user_id = hs.user_id
  SET hs.round_player_id = rp.id;
ALTER TABLE hole_scores ADD CONSTRAINT fk_hole_scores_round_player
  FOREIGN KEY (round_player_id) REFERENCES round_players(id) ON DELETE CASCADE;
```
Leave `round_player_id` NULLABLE and keep the old `(round_id, user_id, hole_number)` unique index in place for this release.

### Migration 155 — `155-rounds-adhoc-course-data.sql`
```sql
ALTER TABLE rounds ADD COLUMN hole_pars_json       JSON NULL;
ALTER TABLE rounds ADD COLUMN hole_handicaps_json  JSON NULL;
ALTER TABLE rounds ADD COLUMN tees_json            JSON NULL;
```

### Migration 156 — next release after backend dual-write
```sql
ALTER TABLE hole_scores MODIFY round_player_id BIGINT NOT NULL;
DROP INDEX uk_hole_scores_round_user_hole ON hole_scores;
ALTER TABLE hole_scores ADD CONSTRAINT uk_hole_scores_rp_hole
  UNIQUE (round_player_id, hole_number);
```

### Migration 157 — release after that (≥30 days), zero readers of `user_id`
```sql
ALTER TABLE hole_scores DROP FOREIGN KEY fk_hole_scores_user;
ALTER TABLE hole_scores DROP COLUMN user_id;
```

Memory rules satisfied:
- All migrations registered in `db.changelog-master.yaml`
- No MySQL 8.0 reserved words (`guest_name`, `round_player_id`, `hole_pars_json`, `hole_handicaps_json`, `tees_json` all safe)
- Table names verified against entity singular vs. table plural
- `NOT NULL` flip staged across releases to avoid the 2026-05-10 deploy-ordering trap

## B. Backend API

### Permission rule (centralized in `ScorecardService`)
```
isSelf  = target.user != null && target.user.id == caller.id
isBooker = round.creator.id == caller.id
require(isSelf || isBooker)
```
Guests have `user == null` → only booker can edit them.

### Endpoints

| Method | Path | Purpose | Auth |
|---|---|---|---|
| `POST` | `/api/rounds/{roundId}/players/guest` | Add guest by name. Body `{ "guestName": "Dad" }`. | Booker only |
| `DELETE` | `/api/rounds/{roundId}/players/{roundPlayerId}` | Re-key existing remove endpoint from `userId` to `roundPlayerId` so it works for guests. | Booker or self |
| `PUT` | `/api/rounds/{roundId}/scorecard` | Extend `SaveScorecardRequest` with optional `forRoundPlayerId` (preferred over `forUserId`). Old clients keep working. | Self + booker rule |
| `PUT` | `/api/rounds/{roundId}/course-data` | Save ad-hoc course info on the round: `{ holePars, holeHandicaps, tees, courseName }`. | Booker only |

### OCR flow (no new OCR endpoint)
1. Mobile → `POST /api/scorecards/ocr` with photo
2. Server returns `ParsedScorecardResponse` (in-memory)
3. Mobile shows review screen; user confirms/edits
4. Mobile → `PUT /api/rounds/{roundId}/course-data` to persist course info on the round
5. Scorecard reads pull pars/SI from `rounds.hole_pars_json` thereafter

### Leaderboard (`RoundLeaderboardService`)
- Group hole scores by `round_player_id`, not `user_id`
- Guest rows: `displayName = guestName`, no `@username`
- New optional Entry fields: `roundPlayerId` (stable key), `isGuest: boolean`
- Caller participation gate unchanged (guests have no account → can't call)
- ETag hash stays stable; new fields are append-only

### Repository changes
- `HoleScoreRepository` — add `findByRoundPlayerId…` variants; deprecate `…AndUserId…` until migration 157
- `RoundPlayerRepository` — add `findByRoundIdAndId(roundId, roundPlayerId)` for permission lookups

## C. Mobile UI

### New screen — `golfsync-mobile/app/rounds/[id]/scorecard.tsx`
- Hole-by-hole editable grid; defaults to viewer's own card
- `?rpid=N` switches to teammate/guest editing (only navigable if viewer is booker)
- Tee selector, par row (editable when no Course attached), strokes row, optional stats
- Debounced autosave per cell
- "Scan card" → existing `ocrScorecard()` → review → `PUT /course-data` then strokes
- "Manual entry": tap "no course data yet" banner → type pars per hole

### Extended screen — `golfsync-mobile/app/rounds/[id]/leaderboard.tsx`
- Pencil icon per row when `canEdit(viewer, round, entry)` is true
- Tap → routes to `/rounds/[id]/scorecard?rpid=…`
- "Add guest" CTA at the bottom (booker only) → name modal → `addGuestToRound()`
- Guest rows render `Name · guest`

### New helpers — `golfsync-mobile/lib/api.ts`
- `addGuestToRound(roundId, guestName)`
- `removeRoundPlayer(roundId, roundPlayerId)`
- `saveRoundCourseData(roundId, payload)`
- Extend `saveScorecard()` inputs with `forRoundPlayerId?: number`

### Permission helper — `golfsync-mobile/lib/permissions.ts` (new)
Single source of truth for `canEditScorecard(viewerId, round, entry)`. Mirrors backend rule. Imported by both leaderboard and scorecard screens.

## D. Rollout sequencing

1. **Release N** — migrations 153 + 154 + 155 (all additive, all nullable). Old code keeps working. **SHIPPED** in api `aa853a0` (2026-05-17), pending prod deploy.
2. **Release N+1** — backend reads/writes `round_player_id`, dual-writes `user_id`. New endpoints (POST guest, DELETE player by `roundPlayerId`, PUT course-data, extended PUT scorecard with `forRoundPlayerId`). Verify zero NULLs in `hole_scores.round_player_id` in prod before proceeding. **SHIPPED** in api `44a20a0` (2026-05-17), pending prod deploy.
3. **Release N+2** — migration 156 tightens NOT NULL + swaps unique index. Web read-only scorecard view ships. Mobile UI ships **bundled with voice commands** in the next EAS build (see "Cross-platform release coordination" below).
4. **Release N+3** — migration 157 drops `hole_scores.user_id`. Defer at least 30 days after N+2 to give all readers (and any background jobs / analytics) time to migrate off `user_id`.

### Cross-platform release coordination (2026-05-17)

- **API + web** can ship Releases N → N+3 at their own pace; both are free + immediate. No App Review delay, no $ cost.
- **Mobile** bundles the casual-scorecard UI with voice commands into a single EAS build + a single App Store / Play submission. Doing two separate mobile releases would burn $4 of EAS credit and double the App Review surface area for no user-visible benefit.
- This means mobile users won't see the new scorecard UI until BOTH the casual-scorecard mobile screen AND voice commands are ready. API endpoints are exercised by web tests + automated curl in the meantime, so server bugs surface independently of the mobile bundle.

Operational rules per memory: `git stash` any WIP before each prod deploy; `./scripts/deploy.sh prod` only; verify each deploy actually rolled out (image hash), not just CFN `UPDATE_COMPLETE`.

## E. Out of scope / follow-ups

- Web editable scorecard UI (phase 2)
- Admin "promote round course data → shared `Course` catalog" flow (request channel TBD — Slack/email vs. lightweight `CourseRequest` entity)
- WebSocket transport (>50 concurrent viewers threshold per BACKLOG)
- Guest → registered-user claim flow (verification email, history merge)
- Per-hole undo / version history on `hole_scores`
- Drop `hole_scores.user_id` (migration 157, ≥30 days after N+2)

## F. Open questions

1. **Edit window after `RoundStatus.COMPLETED`** — booker still able to edit a finalized round? Today it's editable indefinitely. Keep that?
2. **Booker editing teammate hole on BEST_BALL / ALTERNATE_SHOT** — replicate to team score the same way self-edits do today? Assumed yes.
3. **Guests in `findRecentScoredByUserIds`** (friends-recently-played projection) — exclude guests? Defaulting to yes-exclude.
4. **Mobile feature-flag plumbing** — confirm existing mechanism in `golfsync-mobile`, or add one.
5. **Per-guest `teePlayed`** — each guest has their own tee, not inheriting the booker's. Confirm.

## G. Risks

- **Deploy-ordering** (memory: 2026-05-10 outage): all three release-N migrations are strictly additive and nullable. Backend dual-writes through release N+1. Only release N+2 tightens to `NOT NULL`, after backfill is proven.
- **`HoleScore.user_id` callers** — must inventory all readers before migration 157. Inventory step blocks 157, not 156.
- **ETag cache invalidation** — new optional `Entry` fields must not be included in the volatile-fields mask, or the cache thrashes.
- **Permission bypass via `forUserId` legacy field** — both old and new fields must run through the same permission gate in `ScorecardService`; don't gate only the new one.
