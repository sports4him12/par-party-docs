# Smartwatch support — design memo

**Date:** 2026-05-17
**Status:** Pre-implementation; backlog item until paying-customer signal
**Owner:** Ryan
**Decision context:** Asked end-of-day 2026-05-17 while wrapping the voice-commands work. Not in this session's scope to build — captured here so the answer "what do I need to do special?" doesn't have to be rediscovered later.

## Summary

For the next ~6 months, the answer is: **nothing.** Push notifications already reach paired Apple Watch and Wear OS watches today, with zero code on our side. A first-class Watch app is a real product investment best done after Golf Sync has paying users specifically asking for it.

When the time comes, the work is:
- iOS / Apple Watch: ~2-3 weeks of focused engineering for a useful v1
- Android / Wear OS: another ~2 weeks but with a smaller user base, so lower ROI
- No new dev-account fees (covered by the existing Apple Developer Program + Google Play console)

Recommended sequence: ship Watch as v3 after the IAP launch + first paying-customer requests, not pre-emptively.

## Today (no work required)

### Apple Watch — companion notifications
Any push notification the iPhone receives appears on a paired Watch automatically. The Watch's "iPhone Notifications" mode (default for most users) is sufficient.

This covers:
- "Your tournament starts in 30 min"
- "Mike posted a score"
- "Your friend joined your round"
- Any future push we add for the casual scorecard or voice features

**Zero engineering effort** — we already send notifications via APNs, the OS handles Watch propagation.

### Wear OS — companion notifications
Same story. Android's notification system mirrors to paired Wear OS watches automatically. Tested implicitly by every Android user who has a Pixel Watch / Galaxy Watch in pairing mode.

## A first-class Apple Watch app (v3+)

### What "first-class" means
A native app that runs ON the Watch — not just a notification mirror. The user can:
- Tap a hole on the wrist to enter strokes without unlocking their phone
- See current rank in their active tournament
- Get haptic feedback on score-posted or tournament-start

Skip on a watch (stays on iPhone): tournament setup, league management, anything text-heavy, scorecard scan upload, payment settlement.

### Required engineering work
- **New Xcode target in `ios/`** — a watchOS app target, paired with the existing iPhone target. The Expo prebuild does NOT manage watchOS targets; this would either be hand-rolled outside prebuild OR a custom config plugin extending what `plugins/withVoiceCommands.js` already demonstrates.
- **SwiftUI UI code** — there is no React Native for watchOS. Apple supports SwiftUI on watchOS 7+; everything is custom Swift.
- **`WatchConnectivity` framework (`WCSession`)** — bidirectional state sync between phone and watch:
  - Phone → Watch: active tournament, current hole pars, pace timer
  - Watch → Phone: stroke entries, hole transitions
- **Independent App Store review** — Apple bundles the Watch app with the iPhone binary but reviews each side independently. Watch apps are typically reviewed faster than iPhone ones (smaller surface area).
- **No new fees** — the Apple Developer Program ($99/yr already paid) covers watchOS.

### Realistic v1 scope
1. Active-tournament rank readout (read-only mirror of the LeaderboardIntent voice response)
2. Per-hole stroke entry with tap-stepper UI
3. Haptic on score-posted by a teammate (uses the same push payload the phone receives)

**Estimate:** 2-3 weeks focused engineering. The hard part isn't UI — it's the state sync model.

### Risks
- watchOS API churn (Apple deprecates watchOS APIs aggressively; what works on watchOS 10 may need rework on 12)
- Battery — anything with continuous-foreground UI on a Watch face is a battery problem
- Pairing edge cases (watch not paired, watch out of range, etc.) need graceful degradation

## A first-class Wear OS app (lower priority)

### Required engineering work
- **New Android Studio module in `android/`** — Wear OS target with its own AndroidManifest. Same caveat as iOS: Expo prebuild doesn't manage it.
- **Jetpack Compose for Wear OS** — different from regular Compose (different component library, different layout primitives)
- **`Wearable.MessageClient` / `Wearable.DataClient`** for phone↔watch sync
- **Independent Google Play track** for the Wear OS app
- **No new fees** — covered by the Google Play developer registration ($25 one-time, already paid)

### Why lower priority
- Wear OS holds ~15% smartwatch market share vs Apple Watch's ~30% (varies by region but holds globally as of 2026)
- Wear OS user demographic skews more "Android power user" — overlaps less with the Golf Sync demographic (mostly iPhone today per existing membership telemetry, though that may shift)

### Realistic v1 scope
Same three features as Apple Watch v1. Build it AFTER the iOS Watch app proves the design pattern.

**Estimate:** 2 weeks if we copy the iOS state-sync model. More if we have to redesign for Wear OS's different paradigm.

## Prerequisite ordering (what must ship first)

1. **IAP membership** is live — without paid users, there's no signal on what watch features users would pay for. Per the `project_ios_launch_strategy.md` memory, IAP is v2 (post-2026-06-30 trial cliff).
2. **At least 3 paying users have asked for Watch support unprompted.** Don't pre-build this — pre-built Watch apps for "future users" are how startups burn months on features nobody uses.
3. **Casual scorecard mobile UI is shipped.** Per the existing design doc + the 2026-05-17 user decision, the casual scorecard ships bundled with voice on mobile. Watch reuses the same per-hole stroke entry interaction model; the phone version must exist first to be the source of truth.

## When NOT to build this

- "Apple released a new Watch and we should support it on launch day." Apple Watch launches don't drive Golf Sync user acquisition. Wait for paid-user demand.
- "A competitor (Arccos, 18Birdies) has a Watch app." Their Watch apps are typically pace-of-play + GPS-rangefinder features (requires partnerships with course-mapping data providers). Our differentiation is tournament management, which translates poorly to a watch.
- "A potential investor asks about smartwatch strategy." Answer: "Already supported via notifications; native app on the roadmap pending paying-user demand." Don't build to placate investors.

## Out of scope (long tail, document for completeness)

- **GPS rangefinder on the watch** — requires a course-mapping dataset (Garmin, GolfLogix, Hole19's data). Six-figure annual licensing or a build-our-own data pipeline. Not a Golf Sync v1 feature, period.
- **WatchOS-only standalone use** (i.e. golfing without your phone) — Apple Watch GPS + cellular variants can theoretically run apps without the phone present. Edge case; defer indefinitely.
- **HealthKit integration** (heart rate while playing, calories burned) — interesting but not the value proposition. Defer.
