# iOS App Store — Publishing Guide

This document walks through every step required to get the Golf Sync mobile app onto the App Store, from enrolling in Apple's developer program through a live App Store listing.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Apple Developer account | $99/yr — enroll at developer.apple.com/enroll |
| Mac with Xcode 15+ | Required for signing and submission |
| Expo CLI | Already in the project (`npx expo`) |
| EAS CLI | `npm install -g eas-cli` |

---

## 1. Enroll in the Apple Developer Program

1. Go to [developer.apple.com/enroll](https://developer.apple.com/enroll) and sign in with your Apple ID.
2. Choose **Individual / Sole Proprietor** (or Entity if you have a business entity).
3. Pay the $99/yr fee. Approval typically takes 24–48 hours.
4. Once approved, your Apple Developer account will have a **Team ID** (10-character string like `ABC123DEFG`) — you'll need this later.

---

## 2. Configure `app.json` for iOS

The `golfsync-mobile/app.json` already has the core fields. Before building you must verify/update:

```json
{
  "expo": {
    "name": "Golf Sync",
    "version": "1.0.0",
    "ios": {
      "bundleIdentifier": "com.golfsync.mobile",
      "buildNumber": "1",
      "supportsTablet": false
    }
  }
}
```

- **`bundleIdentifier`** — must match exactly what you register in App Store Connect. `com.golfsync.mobile` is already set.
- **`buildNumber`** — increment this integer every time you upload a new build. Start at `"1"`.
- **`version`** — the user-visible version (e.g. `"1.0.0"`). Only needs to change when you release a new version to the store.

---

## 3. Create the App in App Store Connect

1. Go to [appstoreconnect.apple.com](https://appstoreconnect.apple.com) and sign in.
2. Click **My Apps → + → New App**.
3. Fill in:
   - **Platform**: iOS
   - **Name**: Golf Sync
   - **Primary Language**: English (U.S.)
   - **Bundle ID**: `com.golfsync.mobile` (register it if prompted — select "Xcode iOS Wildcard" or enter manually)
   - **SKU**: any unique internal identifier, e.g. `golfsync-ios-001`
4. Click **Create**. You now have an App Store listing shell.

---

## 4. Set Up EAS Build (Expo Application Services)

EAS Build handles compilation and code signing in the cloud — no local Xcode build required.

### 4a. Log in to EAS

```bash
eas login
# Enter your Expo account credentials (create one free at expo.dev if needed)
```

### 4b. Initialize EAS in the project

```bash
cd golfsync-mobile
eas build:configure
```

This creates `eas.json`. Accept defaults — it will look like:

```json
{
  "cli": { "version": ">= 10.0.0" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {}
  }
}
```

### 4c. Link your Apple Developer account

```bash
eas credentials
# Select iOS → choose your Apple account → EAS will handle certificate + provisioning profile
```

EAS can auto-create and manage your **Distribution Certificate** and **App Store Provisioning Profile** — let it do this.

---

## 5. Add Required App Store Assets

Before submitting, prepare these in App Store Connect under your app's listing:

### Screenshots (required)
Apple requires screenshots for at least one of these sizes:
- **6.9" Display** (iPhone 16 Pro Max): 1320 × 2868 px
- **6.5" Display** (iPhone 14 Plus): 1242 × 2688 px

Take screenshots using the iOS Simulator in Xcode, or use a tool like [AppMockUp](https://app-mockup.com).

Minimum: 1 screenshot per required device size. Maximum: 10 per size.

### App Description (required)
Write a plain-English description of Golf Sync for the store listing. Example:

> Golf Sync keeps your golf group connected. Schedule rounds, invite friends, vote on dates with polls, track scores, and message your playing partners — all in one place.

### Keywords (optional but important for discovery)
Up to 100 characters. Example: `golf,rounds,scorecard,schedule,friends,handicap,tee time`

### Privacy Policy URL (required)
Apple requires a privacy policy URL. Host one publicly — e.g. on your existing web app or GitHub Pages — before submitting.

### Support URL (required)
A URL where users can get help. Can be your website's contact page.

### Age Rating
Run the questionnaire in App Store Connect. Golf Sync has no objectionable content — select **4+**.

---

## 6. Build for Production

From inside `golfsync-mobile`:

```bash
eas build --platform ios --profile production
```

- EAS builds in the cloud. Takes ~15–25 minutes.
- You'll get a link to monitor the build at [expo.dev/builds](https://expo.dev/builds).
- When complete, the `.ipa` artifact is stored by EAS.

---

## 7. Submit to App Store Connect

Once the production build succeeds:

```bash
eas submit --platform ios --profile production
```

EAS will:
1. Upload the `.ipa` to App Store Connect via the Apple Transporter API.
2. The build appears under **TestFlight** in App Store Connect within ~30 minutes.

---

## 8. TestFlight — Internal Testing (Optional but Recommended)

Before submitting for App Store review, test with TestFlight:

1. In App Store Connect → **TestFlight** → add yourself (and up to 100 internal testers) under **Internal Testing**.
2. Internal testers don't require Apple review — available immediately after processing (~30 min).
3. Install the TestFlight app on your iPhone and accept the invite.

---

## 9. Submit for App Store Review

1. In App Store Connect, go to your app → **App Store** tab.
2. Select the build you uploaded from the **Build** section.
3. Fill in all required metadata (screenshots, description, privacy policy URL, support URL, age rating).
4. Click **Submit for Review**.

Review typically takes **24–48 hours** for first-time submissions. Apple may ask follow-up questions via App Store Connect messages.

---

## 10. Release

After Apple approves:
- **Manual release**: Go to App Store Connect → click **Release This Version**. Goes live within ~1 hour.
- **Automatic release**: Set this in the submission options so it goes live immediately after approval.

---

## Subsequent Releases

For updates after the initial launch:

1. Increment `buildNumber` in `app.json` (EAS can do this automatically with `"autoIncrement": true`).
2. If it's a new version visible to users, also bump `version` (e.g. `1.0.0` → `1.1.0`).
3. Run `eas build --platform ios --profile production`.
4. Run `eas submit --platform ios`.
5. In App Store Connect, select the new build and submit for review.

Minor updates (bug fixes, UI tweaks) typically receive **expedited review** (same day).

---

## Common Pitfalls

| Issue | Fix |
|---|---|
| Bundle ID mismatch | Make sure `app.json` `bundleIdentifier` matches App Store Connect exactly |
| Missing push notification entitlement | Golf Sync doesn't use push yet — leave this unchecked |
| App crashes on launch | Test on a real device via TestFlight before submitting |
| Missing privacy policy | Host a simple policy page — required even for apps that collect no data |
| Screenshots wrong size | Use the Xcode Simulator set to the correct device model |
| `eas build` fails on credentials | Run `eas credentials` and let EAS regenerate certificates |

---

## Cost Summary

| Item | Cost |
|---|---|
| Apple Developer Program | $99/yr |
| EAS Build (free tier) | 30 builds/month free, then ~$0.06/build |
| EAS Submit | Free |
| Hosting (existing backend) | No additional cost |
