# GolfSync API — SOLID Audit (Service Layer)

**Date:** 2026-05-15
**Scope:** 76 `*Service.java` files in `golfsync-api/src/main/java/com/golfsync/service/` plus 5 `*Service.java` files in `com/golfsync/security/`. Static read-only analysis. No code changes.

This is a triage doc. The point is to surface where SOLID friction is buying real pain (test cost, blast radius on small changes, mis-bucketed concerns) and where it's noise that doesn't need fixing. Recommendations assume one engineer doing incremental work between feature shipments.

A persistent macro-finding worth stating once at the top: **no service in the layer has a corresponding interface**. Every service is injected by concrete type (`private final FooService foo`). That's the layer-wide DIP issue. Per-class DIP commentary below assumes this background and only flags it where it bites (e.g. blocks port substitution for an external API).

---

## Tier A — Per-class breakdown (LOC > 700)

### LeagueService.java — 3,201 LOC, 88 methods

The dashboard "kitchen sink" of the league domain. 30+ injected dependencies. Largest single class in the layer by a wide margin.

**Responsibilities counted (12+ distinct concerns):**
1. League CRUD + image management (create, update, delete, set/clear image, default tee).
2. Membership lifecycle (join via invite code, join public, add partner, remove, role changes).
3. Per-member handicap + flight assignment (delegates *some* work to `LeagueHandicapService` and `LeagueFlightService` but keeps the controller-facing wrappers).
4. Event CRUD.
5. Poll CRUD + voting + close + nudge.
6. Message send + list (legacy chat path, dual-writes via `ConversationService`).
7. Announcement CRUD.
8. Tournament CRUD (the giant `createTournament` ~120 lines; update; start; close; delete).
9. Tournament registration (self, on-behalf-of, status changes, unregister).
10. Tournament push notifications (`sendTournamentPush`).
11. Activity feed (`getLeagueActivity`, `markLeagueSeen`).
12. Admin/super-admin operations (`getAllLeaguesAdmin`, `adminDeleteLeague`, `adminDeleteMessage`).
13. DTO building (12 `to*Response` private methods) and JSON parsing helpers (`parseIntJsonArray`, `serializeCustomStablefordPoints`).

**SRP verdict: God class.** At least 12 responsibilities, 30+ injected fields, three distinct aggregate roots touched (`League`, `LeagueTournament`, `LeagueMember`). The class transactionally mutates payments-adjacent state (`registerForTournament`) in one method and serializes Stableford point arrays (`serializeCustomStablefordPoints`) two pages over. The 88 methods cover essentially every league subdomain.

**OCP:** `createTournament` + `updateTournament` + `validateTournamentBody` + `forceShowNetForFormat` + `requireBestKEntitlement` + `normalizeBsgPoolingMode` form an implicit format-driven branching system that grows with every new tournament format. Adding the next format (already pressure incoming for BSG variants per `golfsync-docs/Formats/`) requires editing this class. A `TournamentFormatPolicy` indirection (one interface, one impl per format) would close it for modification.

**ISP:** Controllers don't need everything. `LeagueAdminController` only needs the three `admin*` methods. `TournamentController` only needs the tournament CRUD + registration block. Today every consumer transitively depends on the entire 30-dep blob, which means a Mockito test of any one slice has to deal with the constructor.

**DIP:** `notifyLeagueMembers` calls `pushNotificationService` and `emailService` directly, mixing infra concerns with domain logic. A `LeagueNotifier` port collapsing the email + push fanout would let the domain code stop knowing the channels.

**Concrete refactor sketch:**
- `LeagueService` (kept slim) — league CRUD, image, default tee. ~400 LOC.
- `LeagueMembershipService` — join (code/public/partner), remove, roles, member handicap/flight wrappers. ~400 LOC.
- `LeagueEventService` — event CRUD. ~150 LOC.
- `LeaguePollService` — poll CRUD + vote + close + nudge (extends `AvailabilityPollService` patterns; share). ~250 LOC.
- `LeagueAnnouncementService` — announcement CRUD. ~200 LOC.
- `LeagueTournamentService` — tournament CRUD, lifecycle (start/close/delete), registration. ~600 LOC.
- `LeagueTournamentFormatPolicy` — strategy interface + per-format impls (Stroke / Stableford / BSG / RyderCup) absorbing `forceShowNetForFormat`, `requireBestKEntitlement`, `normalizeBsgPoolingMode`, `serializeCustomStablefordPoints`. ~150 LOC across impls.
- `LeagueAdminService` — admin overrides only. ~80 LOC.
- `LeagueResponseMapper` — the 12 `to*Response` methods. Shared by all the splits.

The seam is natural: each concern already lives behind its own controller endpoint, so the split doesn't change the public contract.

**Risk / blast radius:** 13 files reference `LeagueService` (controllers + tests). Refactor is safe-ish per-extraction because controllers are 1:1 with concerns. Bigger risk is the cross-method shared private helpers (`findUser`, `findLeague`, `requireOwner`, `requireOwnerOrCaptain`, `notifyLeagueMembers`) — those need to either move to `LeaguePermissionService` (already exists, good landing pad) or become a small `LeagueLookups` helper.

**Priority: HIGH.** This is the class most likely to cause bugs from cross-concern coupling (e.g. the May 10 organizer-add-partner relaxation left a `friendService` field that nothing now uses; that's a symptom). Proceed by extracting one concern at a time, each as its own PR. **Tournament code is paid-customer load-bearing — extract `LeagueTournamentService` last and with extra test scaffolding** (the Stableford league owner is the first paid customer).

---

### LiveScoringService.java — 2,048 LOC, 38 methods

Tournament play loop: start scorecard, submit holes, leaderboard, finalize, owner overrides.

**Responsibilities counted (7):**
1. Scorecard lifecycle (`startScorecard` ×3 overloads, `finalizeScorecard`).
2. Per-hole score writes (`submitHoleScore` ×2 overloads, including foursome-scorer permission).
3. Course/tee par grid resolution (`readParsFromTournament`, `readParsFromCourse`, `readParsFromSnapshot`, `parsForScorecard`, `overlayPerTee`, `overlayLeagueTee`, `parseIntJsonArray`).
4. Net score computation (`computePerHoleNet`, `teeAwareCourseHandicap`, `recalculateTotals`, `lowestTeamCourseHandicap`).
5. Leaderboard build (`getLeaderboard` ×2, `tiebreakKey`, `loadFlightAssignmentsForLeague`).
6. CSV export (`buildLeaderboardCsv` ×2, `csvField`).
7. Owner override + audit log (`ownerOverrideScore`, `ownerSetPlayerStatus`, `ownerOverrideTee`, `getAuditLog` ×2, `setMarker`, `confirmAsMarker`).

**SRP verdict:** Two cohesive domains tangled. The "compute totals + leaderboard" half is conceptually a pure calculation with read-only inputs. The "scorecard lifecycle + permission gate + audit" half is an aggregate write coordinator. They're tangled because the lifecycle methods return shapes that need the rendering math, but a clean Reader/Writer split would surface that.

**OCP:** Pars resolution has four ordered branches (snapshot → league tee → per-tee → course-default). A new par source (planned per-league override per memory `project_per_league_course_overrides`) appends a fifth branch. A `ParGridSource` interface with `Optional<ParsAndSi> tryResolve(scorecard)` and a chain-of-responsibility list would close it cleanly.

**ISP:** `RyderCupService.getTeamLeaderboard` injects `LiveScoringService` only to call `computePerHoleNet`. That's a one-method dependency carrying 30+ unused entry points. Justifies extracting at minimum `ScorecardScoringMath` as its own bean.

**Concrete refactor sketch:**
- `ScorecardWriteService` — `startScorecard`, `submitHoleScore`, `finalizeScorecard`, `setMarker`, `confirmAsMarker`. Owner override stays here. ~700 LOC.
- `ScorecardScoringMath` — pure compute: `computePerHoleNet`, `recalculateTotals`, `parsForScorecard`, `overlayPerTee`, `overlayLeagueTee`, `readParsFrom*`, `teeAwareCourseHandicap`, `tiebreakKey`. No `@Transactional`, no repos beyond read-only par tables. ~500 LOC.
- `LeaderboardService` — `getLeaderboard` ×2, `loadFlightAssignmentsForLeague`, leaderboard entry build. ~300 LOC.
- `LeaderboardCsvExporter` — `buildLeaderboardCsv` ×2, `csvField`. ~80 LOC. (Or fold into `LeaderboardService`; small enough.)
- `ScoreAuditService` — `getAuditLog` ×2, `sanitizeAuditReason`, the audit-log writes that are inline in override methods today. ~150 LOC.
- `ParGridResolver` — chain-of-responsibility for the 4 par sources. ~120 LOC.

**Risk / blast radius:** 9 files reference `LiveScoringService`. RyderCup, controllers, payout. Extracting `ScorecardScoringMath` first is essentially zero-risk because the computations are stateless. Then the leaderboard. The lifecycle slice is the riskiest because of the owner-override audit-write paths.

**Priority: HIGH.** This is paid-customer-active code (Stableford league plays here every week) AND C2 Adopt will hit it for hosted tournaments. The size makes regressions easy. Extract `ScorecardScoringMath` and `LeaderboardService` first — both are well-bounded and would unblock RyderCup using its real one-method dependency without dragging the lifecycle in.

---

### ScorecardService.java — 1,584 LOC, 30 methods

Two completely separate concerns named alike: OCR scorecard parsing (image → JSON) and casual-round scorecard persistence (DB writes).

**Responsibilities counted (4):**
1. OpenAI Vision OCR for played scorecards (`parseScorecardImage` ×2, `parseScorecardImages` ×2, `parseOcrJson`, `buildOcrPrompt`, `lookupStroke`, `weakestConfidence`, `mostInformativeMode`, `firstNonBlank`, all the JSON helpers).
2. OpenAI Vision OCR for blank course-setup cards (`parseCourseSetupImage`, `parseCourseSetupImages`, `parseCourseSetupOcrJson`).
3. Direct HTTP call to OpenAI with embedded API key + model name (uses `java.net.http.HttpClient`, base64-encodes images, builds the prompt, reads the response).
4. Casual-round scorecard persistence: `saveScorecard`, `loadScorecard`, `loadScorecardFor`, `loadSharedScorecard`, `resetScorecard`, `listUserScorecards`, the `persistScorecardForUser` helper, share-link `generateShortCode`.

**SRP verdict:** Two services in a trench coat. The OCR half has zero overlap with the persistence half — different repos, different external systems, different error modes. The class is held together only by the word "scorecard" in the name.

**OCP/DIP:** OpenAI is reached via raw `HttpClient` with `@Value`-injected key — no port. Swapping vendors (Anthropic, Google Vision) requires line edits inside the parser methods. A `VisionOcrPort` interface with one OpenAI implementation behind it would also let unit tests stub the HTTP call instead of standing up a fake server.

**Concrete refactor sketch:**
- `ScorecardOcrService` — all `parse*` methods + the OCR prompt + the JSON readers. Calls a new `VisionOcrPort` instead of `HttpClient` directly. ~700 LOC.
- `VisionOcrPort` (interface) + `OpenAiVisionAdapter` (impl) — encapsulates the Vision call, base64 encoding, model + key. ~200 LOC.
- `ScorecardPersistenceService` — `saveScorecard`, `loadScorecard*`, `resetScorecard`, `listUserScorecards`, share-link logic. ~600 LOC.

**Risk / blast radius:** Only 2 files reference `ScorecardService`, so the rename + split is mechanical. Low-risk extraction.

**Priority: MEDIUM.** Real SRP win, real testability win (OCR can finally be unit-tested without mocking `HttpClient`), but no users are bleeding from the current arrangement. Do it during a quiet week between feature work. The `VisionOcrPort` extraction is also a prereq if `ContentModerationService` (also calls OpenAI directly) wants to share an HTTP client.

---

### EmailService.java — 1,419 LOC, 47 methods

47 `send*Email` methods. Single responsibility on the surface ("send an email") but each method hardcodes its own subject, body, and template logic inline.

**Responsibilities counted (1 functional + many implicit templates):**
1. Send transactional and marketing email via `JavaMailSender`.

The "many implicit templates" is the issue: each method is its own template. There's no separation between (a) the channel (SMTP via `JavaMailSender`), (b) the templates (subject + body), and (c) the policy (which template to send for which event). The class also embeds business rules — `sendTrialLastChanceEmail` knows about trial cliffs, `sendPaymentFailedDay7Email` knows about dunning sequencing.

**SRP verdict:** Pass on the channel responsibility. **Fail** on the template/policy mixing — every new email type forces a new method on this class. New domains (hosted tournaments shipping in this same session: `sendHostedRegistrationConfirmation`, `sendHostedSponsorConfirmation`) didn't get to grow into their own service; they slid into the existing 1,400-line file.

**OCP:** Adding a new email = adding a method. Period. There's no extension point. Given the cadence (every feature ships at least one new template), this is the layer's most-frequently-modified file. Closed for modification = no.

**DIP:** Single `JavaMailSender` dependency is fine. The bigger DIP issue is that every consumer of email (`AuthService`, `LeagueService`, `RoundService`, `MembershipService`, etc.) injects `EmailService` directly. 25 call sites. Any restructure to support a queue/Lambda outbox is hard because every caller depends on the concrete class.

**ISP:** The 47 methods on one bean are a textbook fat interface. `RoundInviteService` needs `sendRoundInviteEmail`. `MembershipService` needs the dunning trio. They share a constructor dep but not a single sensible cohesion.

**Concrete refactor sketch:**
- `EmailSender` (port) — one `send(EmailMessage msg)` method. Adapter on top of `JavaMailSender`. ~80 LOC.
- `EmailTemplateRegistry` — `EmailMessage render(EmailTemplate template, Map<String, Object> vars)`. Templates can be Java records or moved to filesystem (Thymeleaf/Freemarker; Spring has support out-of-the-box). ~200 LOC.
- Per-domain notifier facades that other services depend on:
  - `AuthEmails` (verify, reset, OAuth notice, account deletion).
  - `MembershipEmails` (welcome, payment-failed ×3, renewal reminder, cancelled).
  - `TrialEmails` (welcome, friend nudge, round nudge, ending soon, last chance).
  - `LeagueEmails` (invitation, tournament notif, poll, announcement, broadcast, owner request approve/deny, etc.).
  - `RoundEmails` (calendar invite/update/cancel, round invite, friend request).
  - `HostedTournamentEmails` (registration + sponsor confirmations).

**Risk / blast radius:** **HIGHEST in the layer (25 call sites).** Big-bang extraction would touch 25 files. Realistic plan: introduce `EmailSender` port + first one facade (e.g. `HostedTournamentEmails` since that's the newest code), migrate call-sites for that one domain, repeat. Old `EmailService` shrinks one extraction at a time and eventually goes away.

**Priority: HIGH** for the port + per-domain facade pattern. **MEDIUM** for fully draining the existing class. Even just adding the port (without splitting templates yet) would let SES/SMTP swap out without touching domain code, and would un-block the deferred SES verification monitor (memory `project_ses_verification_monitor`) cleanly.

---

### RyderCupService.java — 1,348 LOC, 19+ methods

Team tournament setup + leaderboard for Ryder-Cup-style matches.

**Responsibilities counted (4):**
1. Team CRUD + roster (`createTeam`, `deleteTeam`, `assignPlayerToTeam`, `removePlayerFromTeam`, `validatePlayersOnTeam`).
2. Match CRUD (`createMatch`, `deleteMatch`, `setMatchResult`, `persistMatchPlayers`, `nextMatchNumber`).
3. Session CRUD (`createSession`, `updateSession`, `deleteSession`, `listSessions`, `nextSessionDisplayOrder`).
4. Three different leaderboards (`buildStrokeTotalLeaderboard`, `buildBestKByHoleRangeLeaderboard`, `buildMatchPlayLeaderboard`) plus the `getMyMatches` view.

**SRP verdict:** Borderline-violation. Three CRUD subdomains are reasonable as one team-tournament service (they're tightly coupled aggregates). The leaderboard math is the part that doesn't belong — it's three implementations of the same conceptual operation, branching internally on tournament format. That's the exact OCP smell that the existing `ScoringStrategy` pattern already solves elsewhere.

**OCP:** `getTeamLeaderboard` does an internal switch on `teamScoringMode` to dispatch to one of three private builders. Adding a fourth team format edits this class.

**DIP:** Calls `liveScoringService.computePerHoleNet(sc)` (line ~734) — only uses one method off `LiveScoringService`. Tight coupling to a 2,048-line class for one read.

**Concrete refactor sketch:**
- Keep `RyderCupService` for the CRUD/setup methods (~600 LOC).
- Extract `TeamLeaderboardStrategyFactory` mirroring `ScoringStrategyFactory`, with `StrokeTotalTeamLeaderboard`, `BestKByHoleRangeTeamLeaderboard`, `MatchPlayTeamLeaderboard` as Spring beans. (~700 LOC distributed.)
- Each strategy depends on `ScorecardScoringMath` (the proposed extraction from `LiveScoringService`) instead of the full `LiveScoringService`.

**Risk / blast radius:** Only 2 files reference `RyderCupService`. Self-contained refactor. The match-play scoring path is the most sensitive — it's still being filled in per the inline TODOs (the file says "MVP slice 2026-05-11").

**Priority: MEDIUM.** Currently small blast radius but actively growing (Ryder Cup is a roadmap area). Doing the extraction now while the strategies are still small (~250 LOC each) is much cheaper than later. Pair with the `ScorecardScoringMath` extraction from `LiveScoringService` so both teams of changes land coherently.

---

### RoundService.java — 1,251 LOC, 31 methods

Casual-round (non-tournament) lifecycle. Distinct from `LiveScoringService` (which is tournament-only) and from `LeagueService` (which is league-only).

**Responsibilities counted (6):**
1. Round CRUD (`createRound`, `updateRound`, `findById*`, `findRoundsForUser`, `findAll`, `cancelRound`, `completeRound`, `adminDeleteRound`).
2. Round invite + booking attest (`attestBooking`, `respondToInvite`, `addPlayerToRound`, `nudgePlayer`).
3. Round state machine (`updateStatus`, `bindWorkflowInstance`).
4. Score submission (`submitScore`, `submitTeamScore`, `setRoundTeams`).
5. Friend feeds + leaderboards (`getFriendScoreFeed`, `getFriendLeaderboard`, `getCourseLeaderboard`).
6. Per-player payment tracking (`setPlayerDues`, `markPlayerPayment`).
7. ICS calendar generation (`buildRoundIcs`).
8. Format/scoring parsing (`parseScoringMode`, `parseScoringSystem`, `parseMatchPlayVariant`, `parseRoundFormat`, `persistRoundGroups`).

**SRP verdict: Mixed concerns (6 distinct concerns).** The friend feed/leaderboard methods and the per-player dues methods are particularly out of place — they don't share state with the round-lifecycle methods and could be elsewhere. The ICS builder is a natural separate concern.

**OCP:** `parseRoundFormat` + `persistRoundGroups` switch on `RoundFormat` — same OCP pressure as `LeagueService`'s tournament-format branching. As round formats expand, this grows.

**Concrete refactor sketch:**
- `RoundService` (kept) — round CRUD, lifecycle, invite, attest, state. ~600 LOC.
- `RoundScoringService` — score submission, format/scoring parsing, group persistence. ~250 LOC.
- `RoundPaymentService` — `setPlayerDues`, `markPlayerPayment`. ~80 LOC.
- `FriendActivityService` — friend feed + leaderboards. ~150 LOC. Better home than the round bean.
- `RoundIcsService` (or move into `EmailService`'s ICS path which already exists). ~80 LOC.

**Risk / blast radius:** 10 call sites. Medium-risk because the round controller is dual-consumer (web + mobile). Per memory `project_owner_parity_rule` mobile parity matters; any controller signature change requires a mobile rebuild.

**Priority: MEDIUM.** Less load-bearing than `LeagueService` or `LiveScoringService` for revenue, but the file is still big enough to be a slow-read for new contributors. Extract the leaderboard methods first (lowest blast radius).

---

### ConversationService.java — 971 LOC, 26 methods

Unified chat: DMs, league rooms, tournament rooms. Replaces the legacy `MessageService` + `LeagueService.sendMessage` paths via dual-write during cutover.

**Responsibilities counted (5):**
1. Conversation creation/lookup (`findOrCreateDmConversation` ×2, `findOrCreateLeagueGeneral`, `findOrCreateLeagueTournament`, `createDmConversation`, `createLeagueConversation`).
2. Message CRUD (`postMessage`, `loadMessages`, `loadThread`, `softDeleteMessage`).
3. Reactions + mute (`toggleReaction`, `muteConversation`, `unmuteConversation`).
4. Member status / presence (`setStatus`, `clearStatus`, `getActiveStatuses`, `renderStatuses`) — feels separate from chat, but the rooms-list UI uses statuses inline, so they're one product feature.
5. Legacy mirror + backlog hooks (`mirrorLegacyDm`, `mirrorLegacyLeagueMessage`, `enrollUserInLeagueChat`, `removeUserFromLeagueChat`).
6. DTO rendering (`toConversationResponse`, `renderMessages`).

**SRP verdict:** Reasonable cohesion for the chat domain. The status-management methods (4) are the soft spot — they read/write a different aggregate (`LeagueMemberStatus`) and don't share entity state with conversations. Worth carving out as `MemberPresenceService` once the chat redesign cutover stabilizes (memory `project_chat_redesign_in_flight`).

**OCP/DIP:** Internal switching on `ConversationType` in DTO rendering and inbox listing. Not severe — chat types are a known closed set (DM, LEAGUE_GENERAL, LEAGUE_TOURNAMENT) and aren't growing rapidly.

**ISP:** `ChatPushService` only needs `loadById` + `participantsOf`. `LeagueService.createLeague` only needs `findOrCreateLeagueGeneral`. Splitting reads from writes (`ConversationReader` vs `ConversationWriter`) would make those consumers' tests trivial.

**Concrete refactor sketch:**
- `ConversationService` (kept) — conversation lookup/create + message CRUD + reactions + mute. ~700 LOC.
- `MemberPresenceService` — `setStatus`, `clearStatus`, `getActiveStatuses`, `renderStatuses`. ~150 LOC.
- `ChatLegacyMirror` — `mirrorLegacyDm`, `mirrorLegacyLeagueMessage`, the backlog enroll/strip helpers. Already conceptually transient code (will go away once cutover finishes). Naming the seam makes the eventual deletion easy. ~150 LOC.

**Risk / blast radius:** 7 files reference `ConversationService`. Caution: the chat redesign is mid-flight per memory; refactor only after cutover lands and the legacy mirror goes away.

**Priority: LOW (right now), MEDIUM (post-cutover).** The class is reasonably cohesive for its size. Defer until the legacy mirror drops, then do the presence extraction.

---

### AuthService.java — 855 LOC, 19+ methods

Authentication (password + OAuth), registration, password lifecycle, refresh tokens, email verification, the per-email amplification rate-limiter.

**Responsibilities counted (6):**
1. Password auth: `register`, `login`, `changePassword`.
2. Refresh token lifecycle: `refresh`, `revokeRefreshTokensForUser`, `issueRefreshToken`, `hashToken`.
3. Email verification: `issueEmailVerification`, `resendEmailVerification`, `verifyEmail`.
4. Password reset: `requestPasswordReset`, `completePasswordReset`.
5. OAuth: `findOrCreateOAuthUser`, `exchangeGoogleIdToken`, `exchangeAppleIdToken`, `handleAppleNotification`, `createSessionForUser`, `deriveUniqueUsername`, `emailLocalPart`.
6. Per-email amp rate limiter (`exceededEmailAmpCap`, `EmailSendWindow` record, `emailAmpWindows` ConcurrentHashMap).

**SRP verdict: Borderline.** The six concerns are all genuinely "auth", and the per-email rate-limiter being colocated with the methods that send the amplifying emails is defensible (the abstraction overhead for an in-process counter isn't worth a separate class). The OAuth path is the cleanest extraction candidate because it has its own collaborators (`GoogleIdTokenVerifierService`, `AppleIdTokenVerifierService`, `AppleTokenExchangeService`).

**DIP:** `AuthService` already injects three security/* verifier services as ports. Good pattern — keep it.

**Concrete refactor sketch:**
- `AuthService` (kept) — register, login, change password, refresh tokens, the rate limiter. ~500 LOC.
- `EmailVerificationService` — issue, resend, verify. ~150 LOC.
- `PasswordResetService` — request + complete. ~100 LOC.
- `OAuthSignInService` — Google + Apple flows + `findOrCreateOAuthUser` + `deriveUniqueUsername` + Apple notification handler. ~250 LOC.

**Risk / blast radius:** Only 6 references. Self-contained.

**Priority: MEDIUM.** Less urgent than the giant classes above but the file is approaching "needs split" size. The OAuth extraction in particular would be valuable because the Apple IAP / Apple Sign-In code shares failure modes the OAuth methods need to handle, and they should be one bag.

---

## Tier B — Focused breakdown (LOC 300–700)

### RoundLeaderboardService.java (595 LOC)
Cohesive within "leaderboard for a single Round". Multiple format-specific builders (`buildIndividualEntries`, `buildMatchPlayEntries`, `buildTeamAwareEntries`, `buildTeamEntries`) with internal branching on `ScoringMode` / `RoundFormat`. **Other-principle violation:** OCP — adding a new round format edits this class instead of plugging in a strategy. **Priority: LOW.** Existing `ScoringStrategy` pattern already covers per-system math; the per-format dispatch could rhyme but the gain is small for casual rounds (low stakes vs tournament).

### CuratedTournamentRefreshService.java (575 LOC)
Three concerns mashed: HTTP search via Serper, HTML scraping (`scrapePage`, `extractLocation`, `extractCity`), and geocoding (`geocodeCity`). All wrapped in a per-state refresh loop. **Other-principle violation:** DIP — Serper, scraper, and geocoder are inline; no ports, can't unit-test the orchestration without standing up HTTP. **Priority: MEDIUM.** Background job; bugs surface as bad data days later. A `TournamentScraper`/`Geocoder` port pair would make scraper drift much cheaper to debug. But it only runs from a scheduled job; defer until that job becomes flaky.

### CourseService.java (543 LOC)
Course CRUD + tee CRUD + scan-driven course-setup save, plus a `propagateCourseSetupToTournaments` that mutates downstream tournaments. The propagation method is the cohesion smell — it crosses into the tournament aggregate. **Priority: MEDIUM.** Per memory `project_per_league_course_overrides`, the per-league override work will touch this; refactor when that work begins so the new boundary is defined while you're already in the file.

### PairingsService.java (455 LOC)
Already uses a `PairingStrategy` interface (good — `pickStrategy(String key)` dispatches to `HandicapBalancedPairingStrategy` / `RandomPairingStrategy`). The class itself orchestrates: load tournament, pick strategy, persist results, build response. **Verdict: Cohesive.** The string-keyed strategy lookup could be enum-keyed for type safety, but that's polish. **Priority: NONE.**

### MembershipService.java (439 LOC)
Stripe webhook entry points + entitlement gates. Two responsibilities: (a) handle Stripe events, (b) `requireOrganizer` / `requireFullAccess` / `requirePaidMembership` style gates. The latter is on the read-side and is what controllers actually call. **Other-principle violation:** SRP soft-violation — split into `StripeMembershipWebhookHandler` and `MembershipEntitlementService`. Also the `parseTier` / `inferTierFromSubscription` is implicit OCP pressure as plans grow. **Priority: MEDIUM.** Stripe-touching code is sensitive (memory `feedback_stripe_silence_alarm_muted`); split when the next billing-related feature lands.

### AvailabilityPollService.java (430 LOC)
Poll CRUD + voting + nudge. **Verdict: Cohesive** — single feature surface. Nearly identical to the poll subset embedded in `LeagueService`; opportunity to consolidate. **Priority: LOW.** Worth merging with `LeagueService`'s poll methods (Tier A item) when that extraction happens.

### RoundGameService.java (427 LOC)
Side games (Nassau, presses, CTP) inside a round. Has internal logic for Nassau presses (`runNassauMatch`, `runAutoPress`). **Verdict: Cohesive** for the game scope. **Other-principle violation:** OCP — game type branching is implicit; adding "Skins" or "Wolf" (which the docs repo's Formats/ folder mentions) would patch this class instead of registering a new strategy. **Priority: LOW** until a new game type is actually being added; then take the opportunity to introduce a strategy.

### HostedRegistrationService.java (411 LOC) — *new this session*
Public registration intake for hosted tournaments. Eight responsibilities listed in its own header comment: validate, resolve flight, upsert user, generate confirmation, capacity guard, optional Stripe intent, league auto-enroll. **Verdict: Borderline.** All concerns are genuinely "intake of one registration", so the cohesion holds. **Other-principle violation:** DIP — calls `HostedStripePaymentService` directly; calls `EmailService` directly via the controller layer; user upsert is inline rather than via `UserService`. The user upsert is the worst smell — there are now two places that create users (`AuthService.register` and here) with subtly different defaults. **Priority: MEDIUM.** Newly written; baseline isn't bad, but the user-upsert duplication is a real footgun. Extract a shared `UserSignupService.upsertFromExternal(email, name)` before this code grows.

### GroupService.java (406 LOC)
Group CRUD + invite/respond + chat enrollment + notify. Cohesive for the group feature. **Other-principle violation:** Mild ISP — calls `EmailService` directly for both notification methods; same pattern as elsewhere. **Priority: LOW.**

### LeagueCourseService.java (395 LOC)
Per-league course override CRUD (tees + per-tee per-hole grids). Single feature surface, recently added (migration 131). **Verdict: Cohesive.** **Priority: NONE.**

### AppleIapService.java (383 LOC)
Apple StoreKit transaction verification + state apply. Direct dependency on `app-store-server-library` SDK (`SignedDataVerifier`). **Verdict: Cohesive** for the IAP scope. **Other-principle violation:** Ties together verification (could be a port) and entitlement apply (domain). **Priority: LOW** — IAP is parked per memory `project_iap_branch_bookmark`; don't touch unless reactivating.

### HostedPaymentReconciliationService.java (338 LOC) — *new this session*
Payment-state transitions for hosted registrations and sponsors. The doubled methods (`markRegistrationPaid` + `markSponsorPaid`, `refundRegistration` + `refundSponsor`, etc.) suggest a missing common type. **Verdict: Borderline.** The repetition is real but the common-type extraction (a `Payable` interface implemented by both entities) carries tradeoffs across the JPA model. **Priority: LOW** unless adding a third payable. The bulk-paid path (the actual C2 Adopt killer feature per the comment) is well-isolated.

### ReferralService.java (324 LOC)
Referral codes + Stripe balance crediting. **Verdict: Cohesive.** Direct Stripe call (`adjustStripeBalance`) is the one infra coupling — stays with the referral domain because it's only used here. **Priority: NONE.**

### UserService.java (302 LOC)
User profile + search + delete. The `deleteSelf` method is interesting — it calls `revokeAppleRefreshTokenIfPresent` and `cancelStripeSubscription` inline, mixing user-ops with two third-party calls. **Other-principle violation:** SRP — account deletion is its own workflow (it spans Stripe, Apple, Email, DB). **Priority: MEDIUM.** Extract `AccountDeletionService` — it's the only orchestration over multiple external systems and currently has no clear test seam. 44 call sites for `UserService` overall, so don't move *everything*; just the deletion path.

### TournamentDiscoveryService.java (300 LOC)
Search for nearby tournaments via Serper (HTTP + cache). **Verdict: Cohesive** with one external dep. **Other-principle violation:** DIP — Serper is direct; would benefit from a `TournamentSearchProvider` port shared with `CuratedTournamentRefreshService` (which also uses Serper). **Priority: LOW.**

---

## Tier C — Scan results (LOC < 300)

| Class | LOC | Verdict | Flag? |
|---|---|---|---|
| LeagueInvitationService | 275 | Cohesive | No |
| PayoutService | 269 | Borderline | No |
| GolferRatingService | 266 | Cohesive | No |
| AdminCourseService | 264 | Cohesive | No |
| ChatPushService | 257 | Cohesive | No |
| HostedTournamentService | 256 | Cohesive | No |
| HostedSponsorService | 256 | Cohesive | No |
| GooglePlacesService | 255 | Cohesive | No |
| BestKOfNPerHoleTeamScoring | 252 | Cohesive | No |
| ContentModerationService | 249 | **Mixed concerns** | **Yes** |
| LeagueTournamentScoring | 241 | Cohesive | No |
| PaymentService | 226 | Cohesive | No |
| AppleIdTokenVerifierService (security/) | 227 | Cohesive | No |
| CourseSearchService | 210 | Cohesive | No |
| PromoCodeService | 207 | Cohesive | No |
| LeagueBroadcastService | 196 | Cohesive | No |
| SitePollService | 194 | Cohesive | No |
| AppleNotificationVerifierService (security/) | 191 | Cohesive | No |
| UserUploadsService | 191 | Cohesive | No |
| LeagueFlightService | 186 | Cohesive | No |
| TournamentOrganizerService | 182 | Cohesive | No |
| FeaturedService | 165 | Cohesive | No |
| PushNotificationService | 163 | Cohesive | No |
| AppleTokenExchangeService (security/) | 163 | Cohesive | No |
| TournamentRegistrationService | 159 | Cohesive | No |
| AvailabilityPollSweepService | 152 | Cohesive | No |
| RoundInviteService | 150 | Cohesive | No |
| FriendService | 146 | Cohesive | No |
| LeagueHandicapService | 139 | Cohesive | No |
| WorkflowService | 137 | Cohesive | No |
| RegistrationPriceService | 137 | Cohesive | No |
| LeagueOwnerRequestService | 137 | Cohesive | No |
| DunningService | 133 | Cohesive | No |
| HostedStripePaymentService | 130 | Cohesive | No |
| AppleClientSecretService (security/) | 130 | Cohesive | No |
| AnalyticsService | 128 | Cohesive | No |
| HostedStripeWebhookService | 117 | Cohesive | No |
| TrialDripService | 116 | Cohesive | No |
| LeaguePermissionService | 115 | Cohesive | No |
| ChatSyncBacklogReplayJob | 110 | Cohesive | No |
| HandicapBalancedPairingStrategy | 106 | Cohesive | No |
| ScoringCommon | 105 | Cohesive | No |
| FeatureEntitlementService | 98 | Cohesive | No |
| TournamentReportService | 97 | Cohesive | No |
| UserEventService | 95 | Cohesive | No |
| TournamentReminderService | 93 | Cohesive | No |
| StablefordScoringStrategy | 84 | Cohesive | No |
| ChicagoScoringStrategy | 83 | Cohesive | No |
| ModifiedStablefordScoringStrategy | 79 | Cohesive | No |
| ConfirmationNumberGenerator | 78 | Cohesive | No |
| RandomPairingStrategy | 74 | Cohesive | No |
| GoogleIdTokenVerifierService (security/) | 70 | Cohesive | No |
| UserReportService | 68 | Cohesive | No |
| TournamentStateRequestService | 68 | Cohesive | No |
| StripeEventDedupeService | 68 | Cohesive | No |
| StripeEventDedupeCleanupService | 67 | Cohesive | No |
| ProductEventsService | 65 | Cohesive | No |
| SuggestedFriendService | 60 | Cohesive | No |
| FavoriteCourseService | 59 | Cohesive | No |
| StrokeScoringStrategy | 57 | Cohesive | No |
| ScoringStrategy (interface) | 52 | Cohesive | No |
| FeedbackService | 52 | Cohesive | No |
| PairingStrategy (interface) | 48 | Cohesive | No |
| SiteSettingsService | 46 | Cohesive | No |
| ScoringStrategyFactory | 39 | Cohesive | No |
| AuditLogService | 39 | Cohesive | No |
| TokenCleanupService | 30 | Cohesive | No |

**Flagged class:**

- **ContentModerationService (249 LOC)** — three independent moderation channels (`checkUrls`, `checkWithOpenAi`, `checkKeywords`) plus a separate `moderateImage` path. The text channels are chained internally with implicit ordering. Adding a fourth moderation source patches this class; the cleaner shape is a `ModerationCheck` interface with each implementation as a Spring bean and the service as a chain runner. Low priority — works today and the call sites just want a verdict — but if a real moderation policy gets layered on (community reports, etc.) this is the natural extraction point.

---

## Cross-cutting analysis

### 1. Repeated infra coupling (DIP)

The layer has **no service-side ports** for external systems. Every service that touches an external system reaches through the SDK or `HttpClient` directly. The repeat offenders:

| External system | Services calling directly | Port would help most because |
|---|---|---|
| **Email (JavaMailSender)** | `EmailService` only — but 25 callers depend on the concrete `EmailService` | Largest fan-in in the layer; introducing `EmailSender` port enables SES/SMTP swap, queue/outbox pattern, and per-domain notifier facades (the real refactor lever) |
| **Stripe SDK** | `MembershipService`, `ReferralService`, `PaymentService`, `HostedStripePaymentService`, `HostedStripeWebhookService`, `StripeEventDedupeService`, `UserService` (`cancelStripeSubscription`), `HostedSponsorService`, `HostedPaymentReconciliationService`, `HostedRegistrationService` | Stripe is the most-spread infra dep. A `BillingGateway` port for the read/write operations actually used (intent create, balance adjust, customer cancel, refund) would unblock testing without `Stripe.apiKey =` setup |
| **OpenAI HTTP** | `ScorecardService`, `ContentModerationService` | Both classes have `HttpClient` + API key inline. `VisionOcrPort` + `ChatModerationPort` (separate because the calls differ) would unify on one HTTP adapter and become trivially mockable |
| **Push (Firebase/Expo)** | `PushNotificationService`, `LiveScoringService`, `LeagueService`, `AuthService`, `RegistrationPriceService` | `PushNotificationService` *is* the port shape conceptually but other services skip it. Audit and route everything through it |
| **S3** | `LeagueService`, `UserUploadsService`, `ContentModerationService` | `UserUploadsService` is closer to a port; `LeagueService` and `ContentModerationService` should depend on it not on `S3Client` |
| **Apple StoreKit + Apple Sign-In** | `AppleIapService`, `AppleIdTokenVerifierService`, `AppleNotificationVerifierService`, `AppleTokenExchangeService`, `AppleClientSecretService` | Already split into 5 verifier-style services in `security/`. Good baseline. The IAP service is the only one tangling verification with domain apply — split if reactivating IAP |
| **Serper / Google Places (HTTP)** | `CuratedTournamentRefreshService`, `TournamentDiscoveryService`, `GooglePlacesService` | Both Serper callers re-implement the HTTP. One `SerperClient` adapter would dedupe |

**Highest-leverage port to introduce first: `EmailSender`.** Largest fan-in, least domain risk, and unblocks a per-domain refactor of templates that's the actual goal.

### 2. Transactional boundaries

`@Transactional` is used in 57 of the 81 classes inspected, applied at the method level (no class-level `@Transactional` defaults). Quick observations:

- **Consistent on writes.** Every CRUD method I sampled in `LeagueService`, `LiveScoringService`, `RoundService` has `@Transactional`. Good.
- **Reads are usually unannotated** — relying on Spring's default per-call session. Acceptable in MVC + Hibernate but means lazy fetches outside a tx will throw. Several `to*Response` methods walk associations; if a controller invokes them outside a tx they'd break. Spot-check passed (the `to*Response` calls are inside `@Transactional` methods today) but this is the kind of latent hazard a `@Transactional(readOnly = true)` on read-side methods would close.
- **Cross-aggregate writes to watch:**
  - `LeagueService.createTournament` writes `LeagueTournament` + `LeagueTournamentRegistration` + `LeagueTournamentCourse` + `LeagueTournamentHolePar` (transitively via the seed-from-prior-tournament path) inside one `@Transactional`. Correct.
  - `LeagueService.createLeague` writes `League` + `LeagueMember` + materializes a `Conversation` via `conversationService.findOrCreateLeagueGeneral` inside one tx. The conversation creation in turn writes `ConversationParticipant` rows. This is one tx today (good) but if `ConversationService` ever becomes async, the boundary breaks silently.
  - `HostedRegistrationService.register` writes `User` (upsert) + `LeagueTournamentRegistration` + `LeagueMember` (auto-enroll) + Stripe PaymentIntent (external). The Stripe call is inside the `@Transactional` — if Stripe throws, the user + registration roll back, but if Stripe *succeeds* and the DB commit then fails, you have an orphan PaymentIntent. Standard outbox-pattern hazard; fine for the C2 Adopt traffic profile but worth noting.
- **No obvious method that mutates across aggregates without a tx boundary.** Most methods either have `@Transactional` or are pure reads.

### 3. DTO / mapping responsibility

**No mapper package exists.** Every service builds its own response DTOs inline. The pattern is consistent: `private FooResponse toFooResponse(Foo entity)` near the end of the class.

Counted in `LeagueService`: 12 `to*Response` methods (~15-30 lines each). Same shape in `RoundService`, `LiveScoringService`, `ConversationService`, `CourseService`, `GroupService`, `AdminCourseService`, `RoundLeaderboardService`, `PayoutService`, `LeagueLeagueResponseMapper` etc.

Two costs:
- **Cohesion noise.** Every service file is ~10-25% mapper code, padding the LOC and pulling attention away from business logic.
- **Test surface.** Mapper logic can only be exercised by also exercising the parent service method, even though it's pure mapping with no DB.

**Candidates for dedicated mappers:**
- `LeagueResponseMapper` — extracted from `LeagueService` first (12 mappers). Best ROI given the size.
- `RoundResponseMapper` — `RoundService.toResponse` plus the leaderboard entry builder.
- `ScorecardResponseMapper` + `LeaderboardResponseMapper` — from the proposed `LiveScoringService` split.
- `ConversationResponseMapper` — from `ConversationService.toConversationResponse` + `renderMessages`.

MapStruct would do this in one annotation per mapper but isn't currently in the project; manual mapper classes are the lower-ceremony choice.

### 4. Top 5 highest-leverage refactors

| # | Refactor | Why | Rough effort | Blast radius |
|---|---|---|---|---|
| 1 | **Introduce `EmailSender` port + first per-domain notifier facade** (start with `HostedTournamentEmails` since hosted code is newest and lowest-risk to migrate) | Highest-fan-in dep in the layer (25 callers). Port unlocks SES outbox, queue, swap, and ends inline-template growth. Per-domain facades shrink `EmailService` one extraction at a time | 2-3 days for port + first facade + 5 call-site migrations. Then ~½ day per subsequent facade | Low if done per-domain. Each migration touches only that domain's services |
| 2 | **Split `LiveScoringService` into `ScorecardScoringMath` + `LeaderboardService` + `ScorecardWriteService`** | Tournament code is paid-customer-active AND C2 Adopt's hosted tournaments hit it. Math layer is stateless, zero-risk to extract first. RyderCup gets a one-method dep instead of carrying the whole thing | 2 days for `ScorecardScoringMath` extraction (lowest risk). Another 2-3 days for `LeaderboardService`. Lifecycle split is a bigger lift, save for later | Math: zero. Leaderboard: 4 callers. Lifecycle: 9 callers |
| 3 | **Extract `LeagueTournamentService` from `LeagueService` (along with a `TournamentFormatPolicy` strategy interface)** | The single biggest cohesion win. `LeagueService` becomes legible. Tournament-format growth (next BSG variants) stops requiring edits to a 3,200-line file. Format policy closes the OCP gap that's been opening with every new tournament type | 4-5 days incl. tests. Do *after* item 2 because both services are heavily test-coupled | High visibility — the Stableford league owner runs through this code every week. Stage with a feature flag and integration tests covering create + update + start + close + Stableford-specific paths |
| 4 | **Extract `LeagueResponseMapper` + apply the pattern to the 4 next-largest services** | Removes ~400 LOC of mapper code from `LeagueService` alone, and sets a precedent that future services follow. Net reduction in service size across the layer. Mapper unit tests get cheap | 1 day for `LeagueService`. ~½ day each for the next four | Internal — no public contract changes. Tests need to know about the new helper bean |
| 5 | **Extract `AccountDeletionService` from `UserService`** | The only place in the layer that orchestrates Stripe + Apple + Email + DB. Currently no clean test seam. Single self-contained extraction with high SOLID payoff (clear SRP, clear DIP via the orchestrated ports) | 1 day | Very low — only `UserController.deleteSelf` calls it |

The first three pull the most weight. Items 4 and 5 are the cheap-but-tidy follow-ons that compound when the bigger extractions are underway.

---

## Bugs spotted in passing

- **`LeagueService` line ~52-59 (vestigial `friendService` field):** the field is `@SuppressWarnings("unused")` and the class comment notes it's vestigial after the 2026-05-10 friend-gate relaxation. Not a bug, but a known dead constructor dep that's still inflating every test that wires `LeagueService` manually. Safe to drop in a follow-up.
- **`LeagueService.createLeague` warning re. dual-write path:** the comment says "after the cutover follow-up drops the legacy paths, this is the only..." (line 119). Worth checking that the cutover for chat-redesign actually does drop the legacy paths to avoid silent dual-write drift.
- **`HostedRegistrationService`-side user upsert duplicates `AuthService.register`-side defaults.** Two paths now create users with subtly different field initialization. Not a live bug yet, but the next "User has X but not Y" surprise will trace back here.
- **`LiveScoringService.getLeaderboard` (line ~970)** uses a comparator cascade that ties back to `PayoutService.computeWinnersByPosition`. The two are documented to "always agree" — but they live in different files with no shared comparator. A future change to the cascade in one and not the other would silently break payout / leaderboard alignment. Candidate for a `TournamentTiebreakComparator` extraction.
- **`ScorecardService` calls OpenAI Vision with a `@Value`-injected key** but doesn't seem to handle a missing key gracefully (it'd 500 on the OCR call). Worth a feature-flag check on `openAiApiKey.isBlank()` if the OCR feature is meant to be optional.

---

*Audit produced by Claude (Opus 4.7, 1M context) — static read-only analysis of the GolfSync API service layer on 2026-05-15. No source code or git state was modified.*
