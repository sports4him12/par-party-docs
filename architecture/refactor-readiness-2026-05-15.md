# Refactor-Readiness Test Audit — 4 SOLID Audit Targets

**Date:** 2026-05-15
**Scope:** Test health for `LeagueService`, `LiveScoringService`, `ScorecardService`, `EmailService` ahead of the refactor sketch in `golfsync-docs/architecture/solid-audit-2026-05-15.md`.
**Method:** Static read of test sources only. No `mvn`, no coverage runs.

## Headline summary

| Service | LOC (prod) | Public methods | Direct unit tests | Integration coverage | Verdict |
|---|---|---|---|---|---|
| `LeagueService` | 3,201 | **67** | **182** in `LeagueServiceTest` | 1 acceptance class + 2 controller slices | **Needs scaffolding** |
| `LiveScoringService` | 2,048 | **23** | **54** in `LiveScoringServiceTest` | 1 controller slice (27 tests) | **Needs scaffolding** |
| `ScorecardService` | 1,584 | **16** | **68 + 9** across two unit classes | 1 controller slice (12 tests) | **Ready** |
| `EmailService` | 1,419 | **42** | **108** in `EmailServiceTest` | none direct | **Needs scaffolding** |

The dominant pattern across all four: pure Mockito at the unit layer (`@ExtendWith(MockitoExtension.class)` + `@InjectMocks`), `@WebMvcTest` slice tests at the controller layer with `@MockitoBean`, and `@SpringBootTest` against H2 for the few acceptance tests. **No service-layer Testcontainers, no shared fixture builders, no MapStruct.** All four target classes have meaningful direct test coverage but with concentrated gaps that line up exactly with the proposed refactor seams.

---

## A. Per-class breakdown

### 1. LeagueService (3,201 LOC, 67 public methods)

#### A.1 Inventory of existing tests

**Unit tests (direct):**
- `golfsync-api/src/test/java/com/golfsync/service/LeagueServiceTest.java` — **182 `@Test` methods**, 3,594 LOC. Pure Mockito with `@InjectMocks`. Uses `ReflectionTestUtils.setField` to wire a real `LeaguePermissionService` (with mocked repos) so the owner/captain gating is exercised end-to-end (line 90-94).
- The test class declares **23 `@Mock` collaborators** (matches the prod constructor blob). Comments at lines 59-80 narrate three rounds of "this mock was added because Test X was NPEing" — the test setup is itself reflecting the SRP problem.

**Distinct method coverage in `LeagueServiceTest`:** 47 unique `leagueService.*` invocation targets out of 67 public methods (~70% surface touched at least once).

**Integration / acceptance tests:**
- `acceptance/CourseTournamentInheritanceAcceptanceTest.java` — `@SpringBootTest` with `@Autowired LeagueService leagueService` (line 56). Two tests, both around the course-pars → tournament-pars propagation contract (`tournamentCreatedBeforeCourseSetup_picksUpHCPsAfterCourseSave`, `manualSiOverride_isPreserved_neverOverwritten`). Uses real H2 + repos.
- `acceptance/ChatRedesignAcceptanceTest.java` — wires `LeagueService` privately to drive league creation as setup; not asserting on `LeagueService` itself.
- `acceptance/LeagueIdorAcceptanceTest.java` — 8+ tests against `/api/leagues/...` endpoints checking IDOR. Hits `LeagueService` via HTTP through `TestRestTemplate`.

**Controller `@WebMvcTest` slices (mock the service — useful as refactor reference):**
- `controller/LeagueControllerTest.java` — `@MockitoBean LeagueService leagueService`, **40 `@Test` methods**.
- `controller/LeagueAdminControllerTest.java` — `@MockitoBean LeagueService leagueService`, **8 `@Test` methods**.
- `controller/LeaguePairingsControllerTest.java` and `controller/LeagueAdminControllerTest.java` reference subset behaviors.

**Cypress E2E** that exercise league/tournament endpoints (against running stack):
- `golfsync-web/cypress/e2e/league-organizer-tools.cy.ts` (the heavy hitter — touches league CRUD, tournament setup, owner tools)
- `golfsync-web/cypress/e2e/tournaments.cy.ts`
- `golfsync-web/cypress/e2e/leaderboard.cy.ts`
- `golfsync-web/cypress/e2e/individual-flow.cy.ts`, `scramble-flow.cy.ts`, `match-play-flow.cy.ts`, `best-ball-flow.cy.ts`, `alt-shot-flow.cy.ts` — all play-format flows likely invoke `createTournament` + `getTournaments`.

**Counts:** 182 unit-test methods directly invoking `leagueService.*`; 40 + 8 = 48 controller-slice tests using a mocked `LeagueService` (those don't catch service regressions but do nail down the method shapes the controllers depend on); ~10 Cypress specs exercise the prod surface end-to-end.

#### B.1 Coverage estimate

Based on which `leagueService.*` calls appear in `LeagueServiceTest`:

| Bucket | Methods | Examples |
|---|---|---|
| **Strong** (≥3 tests, edge cases asserted) | ~22 | `createLeague` (3+), `createTournament` (15+), `createSeason` (3), `getSeasonStandings` (5), `joinByInviteCode` (3), `addPartnerToLeague` (5), `getMyNextUp` (4), `votePoll`, `nudgeNonVoters` |
| **Adequate** (1-2 happy-path tests) | ~25 | `updateLeague`, `setLeagueImage`, `clearLeagueImage`, `deleteLeague`, `deleteEvent`, `closePoll`, `setMemberRole`, `getLeagueDetailAdmin`, `markLeagueSeen`, `setRegistrationStatus`, `unregisterFromTournament`, `sendTournamentPush`, `adminDeleteLeague`, `adminDeleteMessage`, `getMessages`, `sendMessage` |
| **Thin** (transitively exercised; no direct assertion) | ~8 | `setLeagueDefaultTee`, `setMyPreferredTee`, `setMemberHandicap`, `clearMemberHandicap`, `setMemberFlight`, `getMyRoleInLeague` |
| **Untested** (no direct invocation) | **~12** | `getFlights`, `createFlight`, `updateFlight`, `deleteFlight`, `getHoleGroupings`, `setHoleGroupings`, `startTournament`, `closeTournament`, `getPublicLeagues`, `getMyLeagues`/list-mode variants, `getMembers` (only via slice) |

Roughly **47/67 (~70%)** of public methods have direct unit test coverage; ~20% are exercised only through controller/integration tests; ~12 methods (~18%) are essentially untested at the service layer. This is broadly consistent with a class hovering near but probably above CLAUDE.md's 80% line-coverage bar — JaCoCo would tell us for sure but the `target/site/jacoco/` directory doesn't exist (only `jacoco.exec`), so there's no recent HTML report.

#### C.1 The risky surfaces

| Method | Risk | Test today | Test that SHOULD exist |
|---|---|---|---|
| `createTournament` (lines 1012-1148) | Highest. ~120 lines, format-driven branching on Stableford/BSG/multi-course/team scoring, transactional fan-out (creator auto-register, hole-pars seed, push, email, analytics). | 15 tests covering BSG entitlement gate, par seed paths, scoring system parsing, push exclusion, prior-tournament inheritance. | Custom Stableford points round-trip (the `serializeCustomStablefordPoints` JSON write is **completely untested** — search shows zero hits for `customStableford`/`StablefordPoints` in the test). `holeScoreCapMode` + `handicapAllowancePct` normalization paths. `lowHandicapReference` + `maxHandicapCap` paths. Multi-course `persistTournamentCourses`. `forceShowNetForFormat` matrix. |
| `registerForTournament` (line 1257) | Mutates payments-adjacent state (`duesAmountCents`), unique-constraint behavior, called transactionally from inside `createTournament`. | 4 tests (member registers, idempotent, non-member 403, tournament-not-found 404). | Capacity-full path (`maxPlayers` enforcement). Registration-deadline-passed branch. Status-WITHDREW idempotency vs status-REGISTERED idempotency. |
| `setHoleGroupings` (line 1510) | Cross-aggregate write (tournament + groupings). | **None.** | Owner-only gate, idempotent set, capacity rejections. |
| `startTournament` / `closeTournament` (1390 / 1439) | State machine writes, broadcasts, must run before scorecard creation. | **None.** | Owner-only, idempotent close, status transitions guarded. |
| `notifyLeagueMembers` (private, called by 4+ public methods) | Email + push fan-out, swallows errors. | Indirectly tested via `createTournament_pushesToAllMembersExceptCreator` and similar. | Exception isolation (one bad email doesn't block the rest); creator-exclusion logic across all callers. |
| `getMyNextUp` | Dashboard loader; multi-course tournament window logic was the source of a 2026-05-11 prod bug per inline comments. | 4 tests including the multi-course window guard. | Solid here. |
| `serializeCustomStablefordPoints` (private, line 3075) | Writes JSON to a column. Per audit comment, "validation is at the DTO boundary" — but persistence is here. | **Zero.** No test mentions `customStableford*`. | Round-trip test: input `{eagle:8,birdie:5,...}` → JSON → DB → response DTO matches. All-zero collapses to NULL. |
| `resolveHolePars` (private, line 3125) | The four-source par grid resolution branch. | Indirectly via `getTournaments_includesHolePars_andRegistrants` (per-tournament path) and the inheritance acceptance test. | Per-tee path. Per-hole row-incomplete fallthrough. League override path (when added — currently dormant). |
| `getSeasonStandings` (line 2954) | Direction-aware drop-lowest math; Stableford league owner uses this every week per memory. | 6 tests covering ties, no-drop, in-progress filter, mixed direction. | Strong. |

#### D.1 Test gaps to fill BEFORE refactoring `LeagueService`

| # | Gap | Effort | Priority |
|---|---|---|---|
| 1 | `createTournament_customStablefordPoints_persistsJsonColumn` + `_allZeroInput_persistsNull` + `_roundtripsThroughResponse` | small | **Blocker** — splitting `LeagueTournamentService` and extracting `TournamentFormatPolicy` would touch this code and there's no safety net |
| 2 | `createTournament_handicapAllowancePctNormalization` + `_holeScoreCapModeNormalization` + `_bsgPoolingModeNormalization` (one test each, plus invalid-input throws) | small | **Blocker** — these normalizers are exactly the methods the refactor moves into format policies |
| 3 | `startTournament_*` and `closeTournament_*` — owner gate, idempotent close, status transition guard | small | **Blocker** — both are extracted to `LeagueTournamentService`; zero coverage today |
| 4 | `setHoleGroupings_*` — owner-only, capacity rejection, idempotent overwrite | small | High — pairings ride on these |
| 5 | `registerForTournament_capacityFull_throws` + `_deadlinePassed_throws` | small | High — payment-impacting |
| 6 | `persistTournamentCourses_multiCourseTournament_writesRowsInOrder` | small | High — used by every multi-course tournament; no direct assertion |
| 7 | `setMemberHandicap` + `clearMemberHandicap` + `setMemberFlight` direct assertions (currently tested only through `LeagueHandicapService`/`LeagueFlightService` mocks) | small | Medium |
| 8 | `getFlights` / `createFlight` / `updateFlight` / `deleteFlight` — at least one happy-path each | medium | Medium — flight CRUD survives the refactor unchanged but no service-layer test today |
| 9 | A `LeagueServiceTournamentTest` extracted file scoping every tournament-block test (currently sharing 23 mocks with the rest). Pre-create now to make the split obvious | medium | Medium — sets up the refactor's PR-by-PR boundary |
| 10 | One `@SpringBootTest` integration test creating a Stableford tournament end-to-end with the BSG entitlement, custom points, and capacity. Currently zero acceptance tests cover the Stableford league owner's actual flow. | large | High — first paid customer |

#### E.1 Verdict: **Needs scaffolding**

LeagueService has surprisingly broad coverage (182 tests, ~70% of public methods touched), but the highest-risk surfaces — `createTournament`'s format/policy branching and `serializeCustomStablefordPoints` — are the parts that the proposed `LeagueTournamentService` + `TournamentFormatPolicy` extraction touches. Refactor without filling gaps 1-3 and you'll silently break custom Stableford points (which the first paid customer probably relies on). With those gaps closed, the rest of the extraction (`LeagueAnnouncementService`, `LeagueEventService`, `LeaguePollService`) is comparatively safe because each concern is well-tested and 1:1 with a controller endpoint.

---

### 2. LiveScoringService (2,048 LOC, 23 public methods)

#### A.2 Inventory

**Unit tests:**
- `service/LiveScoringServiceTest.java` — **54 `@Test` methods**, 1,513 LOC. Pure Mockito; 16 `@Mock` collaborators including `ContentModerationService`. `ReflectionTestUtils.setField` to wire real `LeaguePermissionService` (line 56-58). Defines test helpers `stubRegistered` and `fullyConfiguredHolePars` to satisfy the gate-checks added in May 2026 — these are pseudo-fixtures.
- 10 unique `liveScoringService.*` invocation targets out of 23 public methods (~43% surface).

**Integration / acceptance:** None directly. No `@SpringBootTest` has `@Autowired LiveScoringService`.

**Scoring math tests (cover transitive behavior):**
- `service/scoring/StablefordScoringStrategyTest.java`, `ChicagoScoringStrategyTest.java`, `ModifiedStablefordScoringStrategyTest.java`, `BestKOfNPerHoleTeamScoringTest.java`, `LeagueTournamentScoringTest.java`, `ScoringSystemMessyDataTest.java` — together ~50 tests exercising the math layer that `LiveScoringService.recalculateTotals` and `computePerHoleNet` delegate to. **The math is well-covered separately**; the integration with `recalculateTotals` is the gap.

**Controller `@WebMvcTest`:** `controller/LiveScoringControllerTest.java` — `@MockitoBean LiveScoringService liveScoringService`, **27 `@Test` methods**. Notable: ETag/304 caching tests on `getLeaderboard`, override audit log, finalize 409 path.

**Cypress:** `leaderboard.cy.ts`, `individual-flow.cy.ts`, `match-play-flow.cy.ts`, `scramble-flow.cy.ts`, `best-ball-flow.cy.ts`, `alt-shot-flow.cy.ts`, `dashboard.cy.ts`, `league-organizer-tools.cy.ts`, `new-features-smoke.cy.ts` — ~9 specs end-to-end the scoring flows.

#### B.2 Coverage estimate

| Bucket | Methods | Examples |
|---|---|---|
| **Strong** | ~7 | `startScorecard` (8 tests covering both happy paths and 4 distinct gate-rejection branches), `submitHoleScore` (12 tests including Stableford net allocation, scorer/teammate writes, range validation), `ownerOverrideScore` (5), `setHolePars` (5), `getScorecardFor` (3 + foursome scorer matrix) |
| **Adequate** | ~5 | `getLeaderboard` (1 ranking test), `finalizeScorecard` (2), `ownerSetPlayerStatus` (2), `ownerOverrideTee` (4), `getScorecard` (3 stroke/show-net matrix) |
| **Thin** | ~3 | `setMarker` (none — just stubs in helpers), `confirmAsMarker` (none), `getHolePars` (indirectly via `setHolePars` save-then-read pattern) |
| **Untested** | **~8** | Three `startScorecard` overloads partially merged into one test set; `getLeaderboard(callerUserId, tournamentId)` permission-checked overload; **`buildLeaderboardCsv` ×2 — 0 tests**; **`getAuditLog` ×2 — 0 tests at the service layer** (controller slice has 2). The 4 par-resolution branches (`readParsFromTournament`, `readParsFromCourse`, `readParsFromSnapshot`, `parsForScorecard`) are private and only transitively covered. |

Roughly **10/23 (~43%)** public methods have direct unit tests; another ~6 are exercised via the controller slice. **The CSV export and audit-log read paths are zero-coverage at the service layer.**

#### C.2 Risky surfaces

| Method | Risk | Test today | Test that SHOULD exist |
|---|---|---|---|
| `getLeaderboard(tournamentId)` (line 956) | Comparator cascade + tiebreaker; per audit, must agree with `PayoutService.computeWinnersByPosition` and they live in different files. | One ranking test. | Tiebreak chain assertion — same total + diff thru + diff handicap → predictable order. The "must always agree with PayoutService" pact is one bug away from drifting. |
| `buildLeaderboardCsv` (×2 overloads, lines 1302, 1308) | Renders user data into a downloadable. CSV-injection (formula prefixes) hazard. | **Zero.** | Empty leaderboard → header-only. Field with `=` / `,` / `"` is escaped. Permission overload throws on non-owner. |
| `parsForScorecard` + the four `readParsFrom*` methods | Per audit, this is the OCP-pressure zone (4 par sources, 5th coming). The whole point of extracting `ParGridResolver`. | Indirect: `getMyScorecard_perTeeSiOverlaysFlatTable`, `_perTeeNullSiCellFallsThroughToFlat`, `_noTeeSet_usesFlatTableUnchanged`, `_mixedTeeGroup_strokesAllocateByPerTeeSi` — 4 tests covering the per-tee overlay branches. | Direct unit tests on each `readParsFrom*` source so the chain-of-responsibility extraction has 1:1 unit coverage of each link. |
| `computePerHoleNet` | Called by `RyderCupService` as a one-method dependency (the ISP smell). The exact method to extract first into `ScorecardScoringMath`. | Indirectly tested through Stableford net-allocation submitHoleScore tests. | Direct `computePerHoleNet_handicap12_givesOneStrokeOnFirstTwelveHoles` style tests at the API surface most callers use. |
| `getAuditLog` (×2) | Owner-or-self read of audit rows. Compliance/dispute use case. | 0 service-layer tests; 2 controller slice tests cover the 403 path. | Direct: caller-not-owner-not-self throws; rows return in chronological order; PII redaction (if any). |
| `ownerOverrideScore` + audit write | Mutates score, writes audit row in same tx, content-moderation gate on the reason. | 5 tests including moderation reject and blank reason. | Solid. |
| `finalizeScorecard` | Cross-aggregate write; downstream payout depends on totals. | 2 (happy + already-finalized 409). | Partial-round finalize behavior; recomputation invariance after finalize. |

#### D.2 Test gaps to fill BEFORE refactoring `LiveScoringService`

| # | Gap | Effort | Priority |
|---|---|---|---|
| 1 | `LiveScoringServiceTest.computePerHoleNet_*` direct tests (handicap=0, =12, =22 distributions matching `LeagueTournamentScoring.strokesReceived` semantics) | small | **Blocker** for extracting `ScorecardScoringMath` — RyderCup uses this method |
| 2 | `LiveScoringServiceTest.buildLeaderboardCsv_emptyLeaderboard_headerOnly` + `_csvInjectionPrefix_escaped` + `_permissionOverloadEnforcesOwner` | small | **Blocker** for extracting `LeaderboardCsvExporter` |
| 3 | `LiveScoringServiceTest.getAuditLog_*` — owner reads any, self reads own, non-owner-non-self 403, ordering | small | High — currently zero service-layer coverage |
| 4 | `LiveScoringServiceTest.parsForScorecard_*` — explicit branch test per par source (snapshot, league tee, per-tee, course default). Today they're entangled with `getMyScorecard_*` tests. | medium | **Blocker** for `ParGridResolver` extraction |
| 5 | `LiveScoringServiceTest.getLeaderboard_tiebreaker_*` — multiple assertions on the comparator cascade matching `PayoutService.computeWinnersByPosition` | small | High — protects the cross-file pact |
| 6 | `LiveScoringServiceTest.setMarker_*` and `confirmAsMarker_*` happy paths + permission checks | small | Medium |
| 7 | A `@SpringBootTest` covering the full lifecycle: createTournament → setHolePars → startScorecard → submitHoleScore × 18 → finalizeScorecard → getLeaderboard → buildLeaderboardCsv. Currently zero end-to-end test of this stack at the API layer. | large | High — Stableford league + C2 Adopt both ride this |
| 8 | Stateless extraction prep: pull `tournament(id, league, creator)` and `scorecard(id, t, u)` helpers in `LiveScoringServiceTest` into a shared `ScorecardFixtures` so the future split tests can share setup. | medium | Medium |

#### E.2 Verdict: **Needs scaffolding**

Strong direct coverage on the lifecycle methods (`startScorecard`, `submitHoleScore`, `ownerOverrideScore`) and good gate-rejection breadth, but **CSV export, audit-log reads, and direct `computePerHoleNet` tests are missing**. The audit's recommended first extractions (`ScorecardScoringMath`, then `LeaderboardCsvExporter`) are the exact methods with the thinnest direct coverage. Items 1-2 above are blockers; the rest are high-leverage but not strictly required.

---

### 3. ScorecardService (1,584 LOC, 16 public methods)

#### A.3 Inventory

**Unit tests:**
- `service/ScorecardServiceTest.java` — **68 `@Test` methods**, 1,545 LOC. Pure Mockito with manual `service = new ScorecardService(...)` constructor (line 42-45) — not `@InjectMocks`. Includes a `stubOpenAi(int status, String body)` helper at line 1380 that stands up an HTTP fake to test the OCR HTTP roundtrip end-to-end.
- `service/ScorecardServiceFirstTimerRobustnessTest.java` — **9 `@Test` methods**, 335 LOC. Focused on edge cases: 8 strokes on 18-hole round, all-null par, partial→full update, 0/3-image OCR error messages.
- All 10 distinct `service.*` invocation targets out of 16 public methods (~62%).

**Integration / acceptance:** None directly. No `@SpringBootTest` references `ScorecardService`.

**Controller `@WebMvcTest`:** `controller/ScorecardControllerTest.java` — `@MockitoBean ScorecardService`, **12 `@Test` methods**. Covers OCR upload, save, reset, share, sharedView.

**Cypress:** `scorecard-entry.cy.ts`, `course-setup-scan.cy.ts`, `shared-scorecard.cy.ts`, `rounds.cy.ts` — direct end-to-end coverage.

#### B.3 Coverage estimate

| Bucket | Methods | Examples |
|---|---|---|
| **Strong** | ~9 | `saveScorecard` (~25 tests covering scorekeeper, team formats, yardages, concession, FIR/GIR, alternate-shot, scramble, best-ball, all-null), `parseScorecardImage` (8 tests including HTTP roundtrip via `stubOpenAi`), `loadScorecard` (8 tests), `parseScorecardImages` (6), `mergeTwoPages` (5), `loadSharedScorecard` (3) |
| **Adequate** | ~5 | `loadScorecardFor` (3), `resetScorecard` (4), `createShareLink` (3), `listUserScorecards` (3), `parseCourseSetupImage`/`parseCourseSetupImages` (covered in `service/CourseSetupOcrParseTest.java` + `CourseSetupFromScanTest.java`) |
| **Thin** | ~2 | `parseCourseSetupImages` (only 2-3 tests in adjacent files) |
| **Untested** | 0 | All 16 public methods are touched at least once. |

Roughly **16/16 (100%)** of public methods have direct unit test coverage. This is by far the best-tested of the four. The OCR HTTP path is *unit-tested via a fake HTTP server*, which is unusual — most of the codebase mocks at the service boundary. Already a rough port pattern in disguise.

#### C.3 Risky surfaces

| Method | Risk | Test today | Test that SHOULD exist |
|---|---|---|---|
| `parseScorecardImage` HTTP roundtrip | OpenAI vendor lock-in; raw `HttpClient`. | 8 tests via `stubOpenAi`. **Already strong.** | None additional for the refactor — just preserve when extracting `VisionOcrPort`. |
| `saveScorecard` team-format fan-out | Cross-player writes (alt-shot replicates within team, scramble fans to roster, best-ball is per-player). | 6+ tests covering each branch explicitly. | Solid. |
| `loadScorecard` stat aggregation (FIR, GIR, putts, penalties) | Per-hole denominator logic; FIR-on-par-3-ignored is a real footgun | 6 tests including the FIR/par-3 trap. | Solid. |
| `createShareLink` short-code generation | Public URL exposure. | 3 tests. | Add a collision-handling test if you don't already have one (idempotent re-issue is tested; brand-new collision path isn't). |

#### D.3 Test gaps to fill BEFORE refactoring `ScorecardService`

| # | Gap | Effort | Priority |
|---|---|---|---|
| 1 | Confirm at least one test exists asserting that `parseScorecardImage` works against a `VisionOcrPort` *interface mock* (currently uses HTTP fake) — to give the extraction a 1:1 swap target. If absent, add `parseScorecardImage_mockedPort_returnsParsedJson`. | small | Medium — the refactor is mechanical anyway |
| 2 | `createShareLink_collisionHandling_retries` — ensures `generateShortCode` re-rolls on collision (assuming it does) | small | Low |
| 3 | A pair of `@SpringBootTest` integration tests (one for OCR-disabled fallback when no API key, one for the persistence side end-to-end). Currently zero integration tests for either half of this service. | medium | Low — unit coverage is already good |

#### E.3 Verdict: **Ready**

This is the cleanest of the four. 100% of public methods have direct unit tests, the OCR HTTP path has a fake-server roundtrip test, and the proposed split (OCR vs persistence) lines up with two clearly separate test clusters in the existing file. The `VisionOcrPort` extraction is essentially a refactor where every existing test can be ported as-is and just point at the port instead of HTTP. The persistence half is even cleaner — pure repo mocks, zero HTTP touched.

The one note: the prod class uses `@Value("${openai.api-key}")`-style injection that the existing tests don't gracefully handle when the key is blank. Test 1 above (`parseScorecardImage_noApiKey_throwsExternalServiceException`) confirms the throw path, but the audit's "make OCR optional via flag check" suggestion has no test today — non-blocker for the SOLID refactor itself.

---

### 4. EmailService (1,419 LOC, 42 public methods)

#### A.4 Inventory

**Unit tests:**
- `service/EmailServiceTest.java` — **108 `@Test` methods**, 1,250 LOC. `@InjectMocks` with a `@Mock JavaMailSender`. `@BeforeEach` sets `fromAddress` + `supportAddress` via `ReflectionTestUtils`. Heavy use of `ArgumentCaptor<SimpleMailMessage>` to assert subject/body content.
- 37 unique `emailService.*` invocation targets out of 42 public methods (~88%).
- The pattern is **very disciplined**: every method has at least one happy-path test and a `_mailException_doesNotPropagate` test. This is the most internally-consistent test class in the layer.

**Integration / acceptance:** None directly. The acceptance tests don't assert on email; they hit endpoints that *trigger* sends but the mail goes to a no-op `JavaMailSender`.

**Controller `@WebMvcTest`:** Three slice tests stub `EmailService`: `AdminControllerTest`, `FriendControllerTest`, `SupportControllerTest`. They verify the controller calls `EmailService` with the right args.

**Cypress:** Email content is not asserted in Cypress — only the trigger flows (`auth.cy.ts`, `account.cy.ts`, `tournaments.cy.ts`, `dashboard.cy.ts`, `league-organizer-tools.cy.ts`, `course-setup-scan.cy.ts`). Email-as-blackbox.

#### B.4 Coverage estimate

| Bucket | Methods | Examples |
|---|---|---|
| **Strong** | ~12 | `sendTrialLastChanceEmail` (5 branches: no activity / score posted / friend / both / fail), `sendLeagueTournamentNotification` (6 covering ICS attachment, MIME fallback, plain text, mail exception), `sendSupportContactEmail` (3 — two-email split assertion), `sendOAuthAccountNoticeEmail` (5 — null/blank/capitalize/baseUrl matrix), `sendEmailVerificationEmail` (4), `sendPasswordResetEmail` (3), `sendRoundCalendarInvite/Update/Cancel` (4 each) |
| **Adequate** | ~22 | The vast majority — happy path + `_mailException_doesNotPropagate` per method |
| **Thin** | ~3 | `sendSitePollNotificationEmail` (none in the mock list above — would need direct file check), `sendAdminSignupNotification` blank-recipient branch |
| **Untested** | **~5** | **`sendHostedRegistrationConfirmation` — 0 tests** (newly added, lines 1282-1330). **`sendHostedSponsorConfirmation` — 0 tests** (lines 1338-1383). `sendLeagueBroadcastEmail` blank-sender variant. The private helper `hostedPaymentInstructions` — covered only via callers, so when the callers have 0 tests this is also 0. |

Roughly **37/42 (~88%)** public methods have direct unit test coverage; ~5 are untested at the service layer. The **hosted-tournament email coverage gap is the single most concerning finding** in this audit — those methods shipped in the same session as this audit and are the C2 Adopt customer-facing emails.

#### C.4 Risky surfaces

| Method | Risk | Test today | Test that SHOULD exist |
|---|---|---|---|
| `sendHostedRegistrationConfirmation` | C2 Adopt customer-facing on every paid registration. Includes payment-method instruction switch (5 cases: check, ach, invoice, in_person, card). | **Zero.** | One test per payment method asserting the body block is included; nullable `firstName`/`tournamentDate`/`flight`/`amountCents` branches; `_mailException_doesNotPropagate` parity with the rest of the class. |
| `sendHostedSponsorConfirmation` | C2 Adopt sponsor pipeline. | **Zero.** | Same shape as above; null `tierDisplayName` → "(see organizer)" branch; null `contactName` → "Hi there"; payment-method matrix. |
| `sendLeagueTournamentNotification` MIME fallback | If MIME attachment build throws, falls back to plain text. Bug here = no calendar invite for tournament announcements. | 6 tests including the MIME-fails fallback. | Solid. |
| `sendTrialLastChanceEmail` activity-text composition | Trial cliff is 2026-06-30 — this email needs to land. | 5 tests covering all 4 activity branches. | Solid. |
| Header-injection sanitization (`sanitizeHeader`) | CRLF strip on all subjects/from/to. | Implicitly tested via every `setSubject` capture, but no explicit "input with `\n` is stripped" test. | One direct test asserting `sanitizeHeader("foo\r\nBcc: x@y") → "foo Bcc: x@y"` — defense-in-depth proof. |

#### D.4 Test gaps to fill BEFORE refactoring `EmailService`

| # | Gap | Effort | Priority |
|---|---|---|---|
| 1 | `sendHostedRegistrationConfirmation_check_includesMailCheckInstructions` + `_ach_*` + `_invoice_*` + `_in_person_*` + `_card_*` + `_nullFirstName_usesThereGreeting` + `_mailException_doesNotPropagate` (7 tests) | small | **Blocker** — C2 Adopt revenue path, 0 service-layer coverage today |
| 2 | `sendHostedSponsorConfirmation_*` parity set (~6 tests) | small | **Blocker** — same |
| 3 | `sendSitePollNotificationEmail_*` happy + mail-exception | small | Medium — site polls are admin tooling but currently uncovered |
| 4 | `sanitizeHeader_crlfInjection_strippedFromSubject` direct test | small | Medium — defense in depth |
| 5 | A new `EmailSenderTest` interface contract: when `EmailSender` port is introduced (per audit refactor #1), every existing test in `EmailServiceTest` should be portable to assert against the port boundary not `JavaMailSender`. Plan the test rename/move now so the refactor PR isn't 108 test rewrites. | medium | High — the refactor itself depends on this |
| 6 | One per-domain facade integration test once first facade extracted (e.g. `HostedTournamentEmailsTest` with a mocked `EmailSender`) | medium | Medium — locks in the facade pattern for subsequent extractions |

#### E.4 Verdict: **Needs scaffolding**

EmailService has the layer's most disciplined test class — every public send method follows the same 2-test minimum pattern. But the hosted-tournament emails are completely untested at the service layer, and they're the C2 Adopt customer-facing emails. Gaps 1 and 2 (~13 small tests) are blockers; the rest are nice-to-have. Once those land, this is the easiest of the four to refactor because the per-method test discipline means each can be mechanically ported to the new facade structure.

The 25-call-site refactor risk is in the **callers** of `EmailService` (per audit), not in `EmailService` itself. The callers' tests stub `EmailService` as a Mockito mock (per the call list in `LeagueServiceTest`, `EmailService` is mocked at line 43). Migrating those stubs to `HostedTournamentEmails`/`LeagueEmails`/etc. will be 25 file edits — mechanical, but high count.

---

## Cross-cutting analysis

### 1. Common test patterns observed

- **Dominant style: pure Mockito unit tests** (`@ExtendWith(MockitoExtension.class)` + `@InjectMocks` or manual constructor). All 4 target classes follow this.
- **`@WebMvcTest` slices** with `@MockitoBean` (Spring Boot 3.4+ syntax per CLAUDE.md) for every controller. 33 controller slice test files.
- **`@SpringBootTest` acceptance tests** in `acceptance/` (12 files) extending `AcceptanceTestBase`. These are full-stack against H2 with real `JavaMailSender`-no-op + `MockitoBean DashboardAssistant`.
- **`ReflectionTestUtils.setField` to wire a real `LeaguePermissionService`** in the two largest unit test files. This is a clever pattern that lets tests exercise gating logic without stubbing `doThrow` on every test, but it's also an admission that `LeaguePermissionService` should probably just be one of the things you mock cleanly.
- **`ArgumentCaptor` heavy use** in `EmailServiceTest` and `LeagueServiceTest` for content-shape assertions on save/send.
- **No `@ParameterizedTest`** (or it's vanishingly rare). Lots of near-duplicate tests that could collapse to parameterized cases (e.g. the 5 OAuth provider variants, the 6 `_mailException_doesNotPropagate` blocks).

### 2. Test data / fixtures

**There is no `Fixtures/` package.** Every test file defines its own helpers inline:
- `LeagueServiceTest` lines 96-107 define `user(id, username)` and `league(id, owner)`.
- `LiveScoringServiceTest` lines 61-101 define `user`, `league`, `tournament`, `scorecard`, plus `stubRegistered` (line 110) and `fullyConfiguredHolePars` (line 125) for newer gates.
- `ScorecardServiceTest` lines 47-59 define `user(id)` and `round(id, creator, players)`.
- `EmailServiceTest` constructs `SimpleMailMessage` via `ArgumentCaptor` — no entity helpers because emails don't need them.

This is **the single biggest tax the refactor will pay**. When `LeagueService` becomes 8 services, each new test file will redefine `user(id, username)` and `league(id, owner)` from scratch, or worse, share-by-copy-paste. **Building a `com.golfsync.test.fixtures.LeagueFixtures` (and similar) before starting the refactor would compound returns.** Estimated ~1 day to extract and migrate the 4 target classes' helpers; saves probably 2-3 days across the refactor PRs and prevents fixture drift.

### 3. What's missing layer-wide

- **Zero Testcontainers usage** — all integration tests run on H2 (`spring.jpa.hibernate.ddl-auto=create-drop` per `application-test.properties`). H2 doesn't enforce MySQL-specific behavior: reserved words, generated columns with FK-CASCADE (the InnoDB 1215 lesson from memory `feedback_innodb_generated_column_fk`), JSON column nuances. The refactor must not introduce any MySQL-only construct or test coverage will rot.
- **`spring.liquibase.enabled=false` in test profile** — schema is Hibernate-managed in tests, Liquibase-managed in prod. Drift hazard. Not a refactor blocker, but worth knowing.
- **Acceptance tests don't cover the four target classes well.** Only `CourseTournamentInheritanceAcceptanceTest` (2 tests) and `LeagueIdorAcceptanceTest` (8+ tests, mostly 403-checking) exercise `LeagueService` end-to-end. **There is no `TournamentLifecycleAcceptanceTest` covering create→register→start→score→finalize→leaderboard.** That's the single highest-value integration test missing.
- **Mock-DB rule confirmed honored.** Per memory `feedback_no_mocks_in_integration_tests`: every `@SpringBootTest` I scanned uses real `@Autowired` repos against H2. Only specific external boundaries (`DashboardAssistant`, OpenAI `HttpClient`) are mocked. ✓
- **Cypress on macOS Tahoe blocked locally** (per memory) — covered via GitHub Actions CI. The 9 specs touching league/tournament/scoring exist and run there.
- **No JaCoCo HTML report present** (`golfsync-api/target/site/jacoco/` doesn't exist). `jacoco.exec` exists from a recent run but the report wasn't generated. CI's 80% gate is the only signal; can't read line-coverage per class without `mvn jacoco:report`.

### 4. Top 5 highest-leverage test investments

| # | Investment | Why | Effort | Refactor it unblocks |
|---|---|---|---|---|
| 1 | **Test-fixture builder package** (`com.golfsync.test.fixtures.{User,League,Tournament,Scorecard,Round}Fixtures`) — extract the 8-10 inline helpers across the 4 target test files into shared static builders | Every refactor that splits a class needs new test files; without shared fixtures each new file copy-pastes the same `user(id, username)` setup. Pays back across all 4 refactors and beyond. | 1 day | All 4 |
| 2 | **`createTournament` format-policy coverage** — gaps D.1 #1, #2, #3 (custom Stableford points round-trip, normalizers, start/close lifecycle) | The exact methods that move into `TournamentFormatPolicy` and `LeagueTournamentService` are the ones with thinnest direct coverage. ~7 small tests close the gap. Without them the highest-revenue refactor (Stableford league owner) is dangerous. | 0.5 day | LeagueService split (audit refactor #3) |
| 3 | **`LiveScoringService` math + CSV + audit-log tests** — gaps D.2 #1, #2, #3 (direct `computePerHoleNet`, `buildLeaderboardCsv` happy + injection, `getAuditLog` permission) | Audit refactor #2 extracts `ScorecardScoringMath`, `LeaderboardService`, `LeaderboardCsvExporter`, `ScoreAuditService` — all four of those are the methods with no/thin direct coverage. ~10 small tests. | 0.5 day | LiveScoringService split (audit refactor #2) |
| 4 | **Hosted-tournament email coverage** — gaps D.4 #1, #2 (`sendHostedRegistrationConfirmation` + `sendHostedSponsorConfirmation` × payment methods × edge cases) | C2 Adopt is the first hosted-tournament customer (launching before September). These are the customer-facing emails on every paid registration. Zero service-layer coverage today. ~13 small tests. | 0.5 day | EmailService split (audit refactor #1); also de-risks C2 Adopt independently |
| 5 | **Tournament lifecycle acceptance test** — one `@SpringBootTest`: createTournament → setHolePars → register members → startTournament → submitHoleScore × 18 → finalizeScorecard → getLeaderboard → buildLeaderboardCsv, with a Stableford scoring system | First end-to-end coverage of the Stableford league owner's actual weekly flow. Catches the cross-aggregate writes that unit tests miss, and survives any service split intact since it speaks HTTP. | 1 day | All of LeagueService + LiveScoringService refactor; also de-risks both customer-facing flows independently |

**Total scaffold investment before refactor work begins: ~3.5 days.** Returns dramatically across the 3-4 weeks of audit refactors #1-3 (the highest-ranked) by both preventing regressions and making each new test file cheaper to write.

---

## Refactor-readiness summary

| Service | Verdict | Blockers | Time-to-ready |
|---|---|---|---|
| `LeagueService` | Needs scaffolding | D.1 gaps #1-3 (custom Stableford persistence, format normalizers, tournament lifecycle methods) | ~1 day |
| `LiveScoringService` | Needs scaffolding | D.2 gaps #1-2 (direct `computePerHoleNet`, CSV export coverage) | ~0.5 day |
| `ScorecardService` | **Ready** | None — refactor as-is | 0 |
| `EmailService` | Needs scaffolding | D.4 gaps #1-2 (hosted tournament emails) | ~0.5 day |

**Suggested execution order, matching the audit's "Top 5" rankings:**
1. Build fixture builders (1 day, foundation).
2. Extract `ScorecardService` → `OcrService` + `ScorecardPersistenceService` + `VisionOcrPort` — already ready, no new tests required.
3. Plug the email hosted-tournament gap (0.5 day), then extract `EmailSender` port + `HostedTournamentEmails` facade.
4. Plug the LiveScoring CSV/math/audit gaps (0.5 day), then extract `ScorecardScoringMath` and `LeaderboardCsvExporter` (the two zero-risk extractions).
5. Plug the LeagueService format-policy gaps (1 day), add tournament lifecycle acceptance test (1 day), then begin `LeagueTournamentService` extraction last (it's the highest-revenue and highest-risk).

Total scaffold ahead of the refactor: ~3.5 engineer-days. Without it, the LeagueService/LiveScoringService work risks Stableford league regressions during peak season and silent breakage of C2 Adopt's hosted-tournament emails.

---

**Key file references for navigation:**
- Service prod files: `golfsync-api/src/main/java/com/golfsync/service/{LeagueService,LiveScoringService,ScorecardService,EmailService}.java`
- Service unit tests: `golfsync-api/src/test/java/com/golfsync/service/{LeagueServiceTest,LiveScoringServiceTest,ScorecardServiceTest,ScorecardServiceFirstTimerRobustnessTest,EmailServiceTest}.java`
- Acceptance tests: `golfsync-api/src/test/java/com/golfsync/acceptance/{AcceptanceTestBase,CourseTournamentInheritanceAcceptanceTest,LeagueIdorAcceptanceTest}.java`
- Controller slices: `golfsync-api/src/test/java/com/golfsync/controller/{LeagueControllerTest,LeagueAdminControllerTest,LiveScoringControllerTest,ScorecardControllerTest}.java`
- Test profile: `golfsync-api/src/test/resources/application-test.properties` (H2 + ddl-auto=create-drop, Liquibase disabled)
- Cypress E2E: `golfsync-web/cypress/e2e/{leaderboard,league-organizer-tools,tournaments,scorecard-entry,*-flow}.cy.ts`

---

*Audit produced by Claude (Opus 4.7, 1M context) — static read-only analysis of the GolfSync API service test layer on 2026-05-15. No source code or git state was modified. Companion to `solid-audit-2026-05-15.md`.*
