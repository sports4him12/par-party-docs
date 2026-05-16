# Voice Command System for League Tournament Scoring — Design

**Date:** 2026-05-16
**Status:** Draft (pre-coding); open questions / risks in §K
**Owner:** Ryan
**Scope:** League tournaments only (casual rounds handled in a separate doc)

## Goal

Let a player score and query their league tournament via voice on Siri, Google Assistant, and Alexa. Two commands in v1:
- **Score submission** — "Hey Siri, I got a par on hole 3"
- **Leaderboard query** — "Hey Siri, where do I stand?"

## Locked decisions

| Decision | Choice |
|---|---|
| Platforms v1 | Siri (iOS App Intents) + Google Assistant (App Actions) + Alexa skill — all three |
| Commands v1 | Both score submission AND leaderboard queries |
| Active round resolution | Server heuristic: `status==SCORING_OPEN` + user enrolled + dated today; if 0 or >1, voice asks for disambiguation |
| Hole inference | Default to next unscored hole + spoken confirmation before commit |
| Scope | League tournaments only |

## A. Architecture overview

Three independent voice surfaces converge on one new backend slice (`/api/voice/*`).

**Critical insight:** Siri and Google Assistant App Actions run **inside the user's mobile app process** — they inherit the existing httpOnly auth cookies + `X-User-Id` header. Alexa runs **in AWS Lambda** with no access to the device's cookie jar — it must speak HTTPS to the API with an `Authorization: Bearer <jwt>` header obtained via OAuth2 account linking.

```
                                            ┌────────────────────────────┐
  iPhone ── "Hey Siri, par on 4" ────▶ App Intent (Swift, in-app)        │
                                            │ shares cookie jar w/ app   │
                                            │ shares SecureStore via     │
                                            │ App Group                  │
                                            └──────────┬─────────────────┘
                                                       │ HTTPS + cookies
  Android ─ "Hey Google, ..."  ───────▶ App Action (Kotlin, in-app) ─┐
                                                                     │ HTTPS + cookies
                                                                     ▼
  Echo ── "Alexa, ask Golf Sync ..."                          api.golfsync.io
       │                                                       /api/voice/*
       ▼                                                            ▲
   Alexa cloud ─▶ Lambda (Node 20)  ───── HTTPS + Bearer JWT ───────┘
       │                            │
       │   OAuth2 account linking   │
       └─▶ /oauth2/authorize ──▶ web login (Alexa app web view)
           /oauth2/token   ──▶ JWT bearer
```

Three handler paths, three languages:
1. **iOS App Intents** — Swift, in an `AppIntentsExtension` target + a thin Intents framework linked into the main app
2. **Android App Actions** — Kotlin, declared via `actions.xml`, handled by an Activity/Service
3. **Alexa skill** — Node.js 20 Lambda using `ask-sdk-core`, deployed via CDK

## B. Expo workflow implications

Current SDK: **Expo SDK 54**, RN 0.81.5, `newArchEnabled: true`, no `ios/`/`android/` on disk (managed).

Three options:

1. **`expo prebuild`** — generates native dirs; commit them OR regen in EAS. App Intents + App Actions both need native code. Prebuild supports config plugins; an AppIntents extension target requires a custom plugin that injects it into the Xcode project on each prebuild. Costly first time, automatable thereafter.
2. **Expo Modules / config plugin only** — works for code running *inside the app*; insufficient for iOS App Intents because the OS introspects the extension target before app launch.
3. **Stay managed, ship Alexa only** — lowest friction. Lambda has zero mobile dependency.

**Recommendation:** phased — start with option 3 (Alexa), then commit to a one-time `expo prebuild` migration (option 1) + custom AppIntents config plugin. The team is already partway down the native-deps path (`expo-dev-client`, Sentry RN, Google Sign-In, Apple Sign-In); prebuild is the direction Expo nudges large apps anyway.

**Open question:** team's stance on committing `ios/`/`android/` to the repo (current managed workflow implies a deliberate "no native code" position). Treat the prebuild migration as a real product decision.

## C. Auth model

**Existing reality:** `JwtAuthenticationFilter` already accepts both `auth_token` cookie AND `Authorization: Bearer <jwt>` header. The bearer-token path is functional but unused. We don't need a new token format — we need a flow that hands an existing-shape JWT to the Alexa skill.

### Siri + Google Assistant (on-device)
No backend changes. Intent handler issues `URLSession`/`OkHttpClient` requests; the platform cookie store carries the existing auth cookie. `X-User-Id` header convention must be replicated in native handlers — read from the App Group's shared SecureStore.

**iOS wrinkle:** `expo-secure-store` stores items in the per-app keychain, not an App Group. Adding the AppIntents extension requires migrating the session blob to a keychain access group (`group.com.golfsync.mobile`) and updating `golfsync-mobile/lib/auth.ts` to write to it via `keychainService`. The cookie jar (`URLSession.shared.configuration.httpCookieStorage`) is already App-Group-aware when configured.

### Alexa (cloud) — OAuth2 account linking
Required by Alexa: "Auth Code Grant" linking. New backend surface:

- `GET /oauth2/authorize` — consent page; user logs in, approves, server 302s to Alexa's `redirect_uri` with a one-time `code`. PKCE required (`code_challenge` + `code_challenge_method=S256`).
- `POST /oauth2/token` (grant_type=authorization_code) — code → JWT exchange; verifies PKCE `code_verifier`.
- `POST /oauth2/token` (grant_type=refresh_token) — refresh path per RFC 6749.

**Implementation:** **Spring Authorization Server** (`spring-boot-starter-oauth2-authorization-server`). Hand-rolling OAuth2 is a security tarpit. Library handles PKCE/code/refresh correctly and integrates with existing `UserDetailsServiceImpl`.

**New tables** (Liquibase changeset `153-oauth2-clients-and-tokens.sql`):
- `oauth2_clients(id, client_id, client_secret_hash, redirect_uris, allowed_scopes, created_at)` — pre-seeded with Alexa skill row
- `oauth2_authorization_codes(code_hash, client_id, user_id, redirect_uri, scope, code_challenge, code_challenge_method, expires_at)` — 60s TTL, one-time
- `oauth2_refresh_tokens(token_hash, client_id, user_id, scope, expires_at, revoked_at)` — 90-day rolling

Access tokens remain stateless JWTs (existing format), but carry a **`scope: "voice"`** claim so the same token can't post-edit league settings. Token lifetime: **24h access / 90d refresh** for Alexa (vs. 3h/7d for cookie flow). A longer voice token is a wider exfiltration window — mitigated by `scope=voice` keeping blast radius tight.

### Scope/permissions
Voice-scoped tokens can call **only** `/api/voice/**` + `GET /api/leagues/tournaments/*/scoring/leaderboard`. Enforce via `@PreAuthorize("hasAuthority('SCOPE_voice')")` on the new controllers. Cookie/full-app JWT carries no scope claim → implicitly all-permission → stays compatible with everything else.

```
POST /oauth2/token (authorization_code)
  validate client_id + client_secret_hash
  load oauth2_authorization_codes by hash(code)
  if missing or expired or consumed: 400
  verify SHA-256(code_verifier) == code_challenge
  delete code row
  access  = jwtUtil.generate(userId, role, scope="voice", ttl=24h)
  refresh = random128bit
  insert oauth2_refresh_tokens(hash(refresh), client_id, user.id, "voice", now+90d)
  return {access_token, refresh_token, token_type:"Bearer", expires_in:86400}
```

## D. Backend additions

New `VoiceController` at `golfsync-api/src/main/java/com/golfsync/controller/VoiceController.java`, base path `/api/voice`. Backed by `VoiceService` at `.../service/VoiceService.java`.

### `GET /api/voice/active-tournament`
Heuristic: `LeagueTournamentRepository` for `status==SCORING_OPEN` + user enrolled (join via `LeagueTournamentRegistrationRepository.findByUserIdWithTournamentAndLeague`) + `tournamentDate == today` (or `today` within `[startDate, endDate]` for multi-course).

Response:
```json
{
  "resolution": "ONE" | "ZERO" | "MULTIPLE",
  "tournamentId": 123,
  "tournamentName": "Tuesday Night League — Week 7",
  "scorecardId": 9876,
  "courseName": "Pine Hill",
  "holes": [{"holeNumber":1,"par":4,"strokes":null}, ...],
  "nextUnscoredHole": 4,
  "candidates": [...]
}
```

`nextUnscoredHole` is computed server-side from `LeagueTournamentHoleScoreRepository`. Living on the server means all three platforms agree — handler code that recomputes the next hole is the kind of drift that produces "Siri said 4, Alexa said 5" tickets.

### `POST /api/voice/score`
Voice-friendly wrapper around `LiveScoringService.submitHoleScore`. Body:
```json
{
  "tournamentId": 123,          // optional; server resolves via heuristic if absent
  "holeNumber": 4,              // optional; defaults to nextUnscoredHole
  "strokes": 4,                 // either strokes OR scoreLabel must be present
  "scoreLabel": "PAR",          // ALBATROSS|EAGLE|BIRDIE|PAR|BOGEY|DOUBLE_BOGEY|TRIPLE_BOGEY
  "clientRequestId": "uuid-v4", // idempotency key
  "confirm": false,
  "confirmationToken": null
}
```

Server logic:
1. Resolve tournament (heuristic if null)
2. Resolve hole (`nextUnscoredHole` if null)
3. If `scoreLabel` set, look up hole's par from `LeagueTournamentHoleParRepository`; convert via offsets (PAR=0, BIRDIE=-1, EAGLE=-2, ALBATROSS=-3, BOGEY=+1, …)
4. Idempotency: hash `clientRequestId` into Redis (existing ElastiCache in CDK) key `voice:idempotency:<uuid>` with the response. Replay within 60 min.
5. If `confirm == false` → return confirmation token (no write)
6. If `confirm == true` → validate `confirmationToken`, then commit via `liveScoringService.submitHoleScore(...)`

Commit response:
```json
{
  "committed": true,
  "holeNumber": 4,
  "strokes": 4,
  "spokenResponse": "Got it — a par on hole 4. You're now 2 over through 4.",
  "scorecard": { ...LiveScoringScorecardResponse... }
}
```

**Why a wrapper, not direct submit-hole?** (1) All three platforms need the same next-unscored-hole logic, (2) `scoreLabel → strokes` conversion needs the hole's par, (3) server-generated `spokenResponse` keeps phrasing uniform across Siri/Google/Alexa.

### `GET /api/voice/leaderboard-summary?tournamentId=...`
Reuses `liveScoringService.getLeaderboard`. Distills to one sentence:
```json
{
  "spokenResponse": "You're tied for 3rd at 4 over, two strokes behind the leader Sam Diaz at 2 over.",
  "rank": 3,
  "totalStrokes": 76,
  "relativeToPar": 4,
  "thru": 14
}
```

### Concurrency / idempotency
`clientRequestId` is the contract. Each platform handler generates a UUID v4 once per intent invocation (NOT per network retry — the platform's retry must reuse the id). Server deduplicates on a 60-min Redis window. `LiveScoringController.submitHoleScore` is unchanged; idempotency happens in the wrapper.

## E. Mobile (Expo) — what changes

### iOS App Intents
- New extension target `GolfSyncAppIntents` at `golfsync-mobile/ios/GolfSyncAppIntents/` (post-prebuild)
- Files: `RecordScoreIntent.swift`, `LeaderboardIntent.swift`, `GolfSyncShortcuts.swift` (the `AppShortcutsProvider`)
- Config plugin `golfsync-mobile/plugins/with-app-intents.js` injects the extension target on each prebuild
- Update `golfsync-mobile/lib/auth.ts` to use `keychainService` (App Group identifier) so the extension reads the session
- Extension calls API directly from extension process (NOT deep-linking into the app — snappier UX). Uses a `URLSession` configured with the App Group's shared `HTTPCookieStorage`.

Intent shapes:
- `RecordScoreIntent`: params `holeNumber: Int?`, `score: ScoreLabelEnum`. Returns `IntentResult & ProvidesDialog`. Calls `/api/voice/score?confirm=false`, speaks the prompt, awaits `.requestConfirmation()`, calls `/api/voice/score?confirm=true` with the returned `confirmationToken`.
- `LeaderboardIntent`: no params. Calls `/api/voice/leaderboard-summary`, speaks the response.

### Android App Actions
- `actions.xml` at `golfsync-mobile/android/app/src/main/res/xml/actions.xml` (post-prebuild)
- Custom intent (BIIs like `actions.intent.CREATE_RECORD` are poor semantic fits)
- Handler `GolfSyncVoiceActivity` at `golfsync-mobile/android/app/src/main/java/com/golfsync/mobile/voice/GolfSyncVoiceActivity.kt`. Reuses the app's `OkHttpClient` + cookie jar.
- **Risk flag:** Google is migrating App Actions into Gemini's app integration framework. App Actions works on shipping Android today; Gemini is the future. Build for the next 18 months, plan a port.

### App-side UI work
- New screen `golfsync-mobile/app/voice-settings.tsx` — three toggles (Siri, Google, Alexa) + "Connect Alexa" button that universal-links into the Alexa companion app's account-link flow
- Onboarding card on the active-tournament screen when `GET /api/voice/active-tournament` returns `resolution: "ONE"`: "Try saying: Hey Siri, I got a par on hole 3." Track first-use in SecureStore.

## F. Alexa skill — what changes

### Repo location
**New top-level directory** `golfsync-alexa-skill/` (in the monorepo). Cost: Lambda is free under 1M invocations/mo; CloudWatch ~$0.50/GB ingested.

### Files
- `golfsync-alexa-skill/skill-package/skill.json` — manifest with account-linking pointing to `https://golfsync.io/oauth2/authorize` and `/oauth2/token`
- `golfsync-alexa-skill/skill-package/interactionModels/custom/en-US.json` — `RecordScoreIntent` (slots: `holeNumber: AMAZON.NUMBER`, `score: GolfScoreType` custom slot — PAR/BIRDIE/EAGLE/BOGEY/etc.) and `LeaderboardIntent`
- `golfsync-alexa-skill/lambda/index.js` — Node 20 handler using `ask-sdk-core`. Reads access token from `handlerInput.requestEnvelope.context.System.user.accessToken`, calls `https://api.golfsync.io/api/voice/score` with `Authorization: Bearer ${token}`.
- `golfsync-alexa-skill/package.json` — `ask-sdk-core` + Node 20 native `fetch`

### CDK changes
Add a `lambda.Function` in `golfsync-cdk/lib/golfsync-cdk-stack.ts` analogous to the existing `pushover-relay` Lambda. **Manual step (not CDK):** register the skill in Amazon Developer Console, point at the Lambda ARN, fill account-linking config (no CloudFormation provider for ASK).

## G. Confirmation flow (cross-platform)

Server-driven, uniform phrasing, stateless via signed token.

`POST /api/voice/score` with `confirm: false`:
1. Resolve tournament, hole, strokes
2. Build spoken text: "Recording a par on hole 4. Correct?"
3. Build JWT (HS256, same secret) with `{tournamentId, userId, holeNumber, strokes, clientRequestId, iat, exp=now+90s, purpose:"voice-confirm"}`
4. Return `{ committed: false, confirmationPrompt, confirmationToken, expiresInSeconds: 90 }`

Handler speaks the prompt. On "Yes" → `POST /api/voice/score` with `confirm: true` + same `clientRequestId` + `confirmationToken`. Server:
1. Verifies signature, expiry, `purpose=="voice-confirm"`
2. Checks claims match the (re-resolved) request — same tournament, hole, strokes
3. Writes via `liveScoringService.submitHoleScore`
4. Returns committed result

Stateless — no server-side pending-confirmation table. Drift (concurrent web edit between preview and commit) returns 409 + re-prompts.

```
POST /api/voice/score {confirm:false, holeNumber:null, scoreLabel:"PAR", clientRequestId:"uuid-1"}
  → resolve tournament=123, hole=4, par=4, strokes=4
  → token = jwtSign({tournamentId:123, userId:u, holeNumber:4, strokes:4,
                     clientRequestId:"uuid-1", purpose:"voice-confirm", exp: now+90s})
  → return {committed:false, confirmationPrompt:"Recording a par on hole 4. Correct?",
            confirmationToken: token}

POST /api/voice/score {confirm:true, confirmationToken:<jwt>, clientRequestId:"uuid-1"}
  → jwtVerify; reject if expired/invalid
  → if claims.userId != caller.userId: 403
  → idempotencyCheck("voice:idempotency:uuid-1") → replay if exists
  → liveScoringService.submitHoleScore(u, 123, 4, 4, null)
  → cache response, 60 min TTL
  → return {committed:true, ..., spokenResponse:"Got it — a par on hole 4..."}
```

Per-platform: Siri → `.requestConfirmation()`; Google → confirm-slot; Alexa → `Dialog.ConfirmIntent` directive. All three speak the server-provided `confirmationPrompt` verbatim.

## H. Watch / wearable note

Apple Watch could host `RecordScoreIntent` natively (App Intents are watchOS-aware in iOS 16+/watchOS 9+) but Expo prebuild doesn't generate watchOS targets and `react-native` has no watchOS support — the watch app would be pure Swift, sharing only the App Intent code via a shared framework. Wear OS has similar pain. **Defer to phase 2.** Phone-based voice covers ~95% of "hands dirty, want to log a score" use cases.

## I. Rollout / phasing

**Revised 2026-05-16:** Alexa moved to the last phase. **Siri + Google ship together, both done, before Alexa starts.** Both run on-device inside the app and reuse the existing cookie auth — no OAuth2 needed for either. This shrinks Phase 1 considerably (just the `/api/voice/*` endpoints, no Spring Authorization Server).

1. **Phase 1 — Backend voice endpoints (no OAuth2).** Ship `/api/voice/active-tournament`, `/api/voice/score`, `/api/voice/leaderboard-summary`. Authenticated by the existing cookie/JWT. No mobile changes. Test via curl. Deployable as normal Spring Boot release. **OAuth2 is NOT in Phase 1** — deferred to the Alexa phase.
2. **Phase 2 — `expo prebuild` + Siri (iOS App Intents) + Google (Android App Actions), shipped together.** Both platforms covered before moving on. Largest user-value lever; biggest mobile workflow change. The `expo prebuild` migration is a one-time cost that benefits both. iOS 16 minimum required. Android App Actions risk (Gemini deprecation) re-evaluated at start of this phase — may pivot to Gemini app integration.
3. **Phase 3 — Alexa skill + OAuth2.** Spring Authorization Server with voice-scoped tokens, Liquibase migration for the OAuth2 client/token tables, Lambda + skill manifest. Smallest audience, biggest infra lift. Deferred to last so the user-facing phone surfaces ship first.

Each phase ships independently. Phase 1 is reusable infrastructure for all three voice surfaces.

## J. Out of scope / follow-ups

- Voice for casual rounds (separate doc — casual rounds lack SCORING_OPEN guarantee, harder active-round resolution)
- Voice owner commands (start tournament, finalize a player, override scores) — player's own card only in v1
- Designated-scorer voice flow ("Mark got a birdie on 4") — server already supports `targetUserId`, but voice resolution (which Mark? which tournament?) is real product surface
- Voice locales other than en-US/en-GB
- Watch/Wear apps (§H)

## K. Risks

- **App Actions deprecation.** Google migrating to Gemini app integrations. Phase 2 may pivot the Android half mid-flight.
- **iOS deployment target.** App Intents requires iOS 16+. Expo SDK 54 default is iOS 15.1. Phase 2 forces a bump to iOS 16; loses any iOS 15 users.
- **OAuth2 attack surface.** Account-linking flows have long history of token-leak bugs. Use Spring Authorization Server, enable PKCE, strict redirect URI allow-lists, log token issuance.
- **Alexa account-linking UX.** Clunky WebView in the Alexa app. Expect substantial first-time drop-off. Mitigation: "Link Alexa" deep link from voice-settings screen via Alexa companion universal link.
- **Voice misrecognition.** "Five"/"nine" near-homophones; "par"/"bogey" frequently confused. Confirmation flow mitigates but doesn't eliminate. Consider 10-minute undo from voice-settings.
- **Cost.** Lambda + CloudWatch for Alexa: ~$5-20/mo at modest adoption. Spring Authorization Server adds zero infra cost (in-JVM). Real cost is engineering time — Phase 2 (prebuild + AppIntents + App Actions together) is multi-sprint, not a week.
- **Concurrent edits.** Voice commit could race with web edit. Confirmation-token claims include strokes; if scorecard moved between preview and commit, 409 + re-prompt.

## L. Critical files for implementation
- `golfsync-api/src/main/java/com/golfsync/controller/LiveScoringController.java`
- `golfsync-api/src/main/java/com/golfsync/config/SecurityConfig.java`
- `golfsync-api/src/main/java/com/golfsync/security/JwtAuthenticationFilter.java`
- `golfsync-mobile/app.json` (deployment target bump for Phase 2)
- `golfsync-cdk/lib/golfsync-cdk-stack.ts` (Alexa Lambda)
- new: `golfsync-api/src/main/java/com/golfsync/controller/VoiceController.java`
- new: `golfsync-api/src/main/java/com/golfsync/service/VoiceService.java`
- new: `golfsync-api/src/main/resources/db/changelog/changes/153-oauth2-clients-and-tokens.sql`
- new: `golfsync-alexa-skill/` directory
- new: `golfsync-mobile/plugins/with-app-intents.js` (Phase 2)
