# B14 Research: Health Connect On-Device, Account-Free Local Mode (Android v1)

Researched whether Splitbook can read workouts on Android through Health Connect with no Google sign-in, which permissions are needed, what Google Play requires to ship it, how it fits Expo, how new workouts arrive without a webhook, and what the later Apple route would cost.

All pages checked on 2026-10-15. Sources are official Android, Google Play, Expo and Apple documentation. Data size and retention are covered separately in `B14-data-size-and-retention.md`.

## 1. Does Health Connect need a Google account, Cloud project or OAuth client?

No sign-in is needed, as far as Google's documentation goes — and this is the finding the whole v1 scope rests on.

Health Connect is an Android platform component, not a web API. Access is granted through ordinary Android permissions: the app declares `android.permission.health.*` entries in its manifest and requests them at runtime through `PermissionController.createRequestPermissionResultContract()`, which shows the Health Connect permission screen. The user grants or denies per data type. Google's user-facing help page states plainly that "Your data is stored locally, on your device."

Nowhere in the setup path is there a Google Cloud project, an OAuth client ID, a consent screen, a publishing status, a test-user list or a refresh token. Every constraint found in B3 (#3) — the 7-day refresh token expiry, the 100-user cap, restricted-scope verification — attaches to the OAuth client of the *Google Health API*, a different product. None of it applies here.

**Honest limit on this answer:** no official page states "Health Connect does not require a Google account." I am inferring from the absence of any account step across the setup guide, the platform architecture page and the user help page, none of which mention sign-in at all. The positive evidence (manifest permissions, runtime permission contract, local storage) is strong and consistent, but the explicit negative statement is **Could not confirm**. Worth a five-minute check on a real device before the architecture is locked.

- https://developer.android.com/health-and-fitness/health-connect/get-started (checked 2026-10-15)
- https://developer.android.com/health-and-fitness/guides/health-connect/plan/developer-functionality (checked 2026-10-15)
- https://support.google.com/android/answer/12201227 (checked 2026-10-15)

## 2. Permissions for exercise sessions, heart rate, distance and GPS route

| Data | Permission | Notes |
|---|---|---|
| Exercise sessions | `android.permission.health.READ_EXERCISE` (`ExerciseSessionRecord`) | Standard runtime request |
| Heart rate | `android.permission.health.READ_HEART_RATE` (`HeartRateRecord`) | Standard runtime request |
| Distance | `android.permission.health.READ_DISTANCE` (`DistanceRecord`) | Standard runtime request |
| GPS route | `android.permission.health.READ_EXERCISE_ROUTES` | **Behaves differently — see below** |

Heart rate and distance are not nested inside the session. They are separate record types read by querying the session's start and end times, then filtering `HeartRateRecord` and `DistanceRecord` to that interval. The sync code has to stitch them together itself.

**Routes are the awkward one.** `READ_EXERCISE_ROUTES` (plural; the write permission is singular) cannot be granted the way the others are. Google's reference states that it "can only be granted manually by a user in Health Connect settings or in the route request activity. Attempts to request it by applications will be ignored." Routes written by *other* apps additionally require a per-route consent dialog granting a one-time read, and if the app is in the background, Health Connect returns `ExerciseRouteResult.ConsentRequired` even when the app holds "Always allow" route access. Google's guidance is to request routes only on deliberate user interaction while the user is engaged with the UI.

Practical consequence: exercise sessions, heart rate and distance sync cleanly and unattended. **GPS routes cannot be bulk-imported in the background.** Either the user taps through a consent dialog per workout, or routes become an opt-in, on-demand feature ("show me the map for this run") rather than something synced automatically. This should be decided before the UI is designed.

A route is a list of points with `time`, `latitude`, `longitude`, `altitude`, `horizontalAccuracy` and `verticalAccuracy`. Note also that `ExerciseSessionRecord` supports only one aggregate, `EXERCISE_DURATION_TOTAL`, so every other summary metric must be computed by us from the raw records.

- https://developer.android.com/reference/android/health/connect/HealthPermissions (checked 2026-10-15)
- https://developer.android.com/health-and-fitness/health-connect/features/exercise-routes (checked 2026-10-15)
- https://developer.android.com/health-and-fitness/health-connect/experiences/workouts (checked 2026-10-15)

## 3. What Google Play requires

There is a gatekeeper, but it is a Play review rather than an OAuth verification, and nothing like the CASA security assessment B3 found.

To publish an app using Health Connect data types you must, in Play Console: review the Play health policies, complete the Data Safety section, and complete the **Health apps declaration form** on the App content page. Google requires the minimum data types needed with a valid, detailed use case for each, and a privacy policy posted on the Play listing. The declaration is re-reviewed when you submit a version that uses a different set of data types. All health and fitness permission requests are subject to review; denials cite unclear justification, misalignment with approved use cases, or requesting more permissions than necessary, and can be revised and resubmitted.

**No user cap and no fee are stated anywhere** in the Play or Health Connect publishing documentation — a material difference from the Google Health API's hard 100-user ceiling. Google does not publish a review timeline for the declaration.

Note this only bites at Play Store distribution. For a friends-and-family build installed directly (APK or internal testing track), the declaration is not the blocker it would be for a public listing — though internal/closed testing tracks have their own tester limits, which I did not research.

- https://developer.android.com/health-and-fitness/health-connect/publish (checked 2026-10-15)
- https://support.google.com/googleplay/android-developer/answer/12991134 (checked 2026-10-15)
- https://support.google.com/googleplay/android-developer/answer/14738291 (checked 2026-10-15)

## 4. Expo support

**Expo Go will not work.** Health Connect needs native Android code, and Expo's documentation is explicit that Expo Go cannot use third-party libraries requiring custom native code. We need a **development build** (local, or EAS Build), with the library wired in through a config plugin. This is a normal, well-trodden Expo path, but it does mean the project leaves the Expo Go workflow permanently — worth knowing up front since it changes how a non-technical friend installs the app.

**Android version floor.** The Health Connect SDK supports Android 8 (API 26) and higher, but the Health Connect app itself requires Android 9 (API 28) or higher — so in practice only Android 9+ users can actually use it. On **Android 14 (API 34) and higher** Health Connect is part of the Android framework with no setup required; on **Android 13 and lower** the user must install the Health Connect app from the Play Store first. The app must handle the "Health Connect not available" case gracefully.

**Library choice is Could not confirm.** The standard community binding is `react-native-health-connect`, but it is not an official Google or Expo package, and I did not find it documented on docs.expo.dev. Its maturity, config-plugin options and whether it exposes routes and change tokens need to be checked against its own repository before committing. This is the main implementation risk in the Android plan.

- https://developer.android.com/health-and-fitness/health-connect/get-started (checked 2026-10-15)
- https://docs.expo.dev/workflow/using-libraries/ (checked 2026-10-15)
- https://docs.expo.dev/workflow/customizing/ (checked 2026-10-15)

## 5. Getting new workouts with no webhook

There is no push. Google states plainly that an app "can't get notified of new data," and recommends checking at two points: when the app comes to the foreground (via lifecycle events), and periodically while it remains in the foreground.

The efficient mechanism is **change tokens**. Call `getChangesToken` with the data types you care about, then `getChanges(token)` returns `UpsertionChange` and `DeletionChange` entries plus a next token; loop until exhausted and store the next token for the following sync. Google recommends a separate token per data type, so one revoked permission doesn't break the whole sync, and filtering upsertions whose `dataOrigin.packageName` is our own app to avoid re-importing.

Two constraints matter for our design:

- **An unused change token expires within 30 days.** Google requires a fallback strategy; the one it calls most ideal is to store the timestamp of the last successful read and, on expiry, re-read from that timestamp (or the last 30 days) and deduplicate using client IDs.
- **Foreground only by default.** "Apps can only read data from Health Connect while they are in the foreground." A separate `READ_HEALTH_DATA_IN_BACKGROUND` permission exists and can be granted by the user. Access can be interrupted mid-sync at any time, so sync must be resumable.

**Impact on "new PR" notifications.** Without background read, a new personal record cannot be detected until the user opens Splitbook — so the notification is "you set a PR on your run this morning," delivered whenever they next open the app, not moments after the workout ends. With `READ_HEALTH_DATA_IN_BACKGROUND` granted, a periodic background job could narrow that, but it is still polling, not push, so timing depends on the job's cadence and Android's power management. Note also that GPS routes stay unavailable in the background regardless (§2), so a background PR check can use session, heart rate and distance only. This is a real, user-visible downgrade from the cloud webhook design and should be written into the spec rather than discovered later.

- https://developer.android.com/health-and-fitness/guides/health-connect/develop/sync-data (checked 2026-10-15)
- https://developer.android.com/health-and-fitness/health-connect/read-data (checked 2026-10-15)

## 6. Cost of the later Apple route

**Apple / HealthKit.** The Apple Developer Program is **99 USD per membership year**, and enrolment is required to distribute apps at all — it is not a HealthKit surcharge, it is the cost of shipping any iOS app. Fee waivers exist for nonprofits, accredited educational institutions and government entities. HealthKit itself carries no separate fee. Like Health Connect, it is an on-device API using per-type authorisation with `NSHealthShareUsageDescription` strings, so **no Google account is involved in the Apple route either**.

**Google OAuth.** Whether Google OAuth itself is free is **Could not confirm**. No official page states that OAuth 2.0 or Sign in with Google is free of charge or caps free users. The only explicit per-user pricing I found is Google Cloud **Identity Platform**, a different managed product (first 50,000 monthly active users at $0.00, then $0.0055/MAU) — it is not what a plain OAuth client uses, and quoting its numbers for our case would be wrong. The costs I *can* source for the Google Health API path are from B3 (#3): Google's own verification is free, but restricted scopes require an annual third-party CASA security assessment at roughly **$500–$4,500 USD per year**, and unverified apps are capped at 100 lifetime users.

**So the answer to "do we need a Google account if we go Apple too" is: no.** The natural pairing is Health Connect on Android and HealthKit on iOS, both on-device, both account-free, total known cost $99/year for the Apple Developer Program plus whatever Play distribution requires. Routing iOS through the Google Health API instead would reintroduce every OAuth constraint from B3 *and* the CASA bill, for strictly worse privacy, and should be rejected unless some capability turns out to be unavailable on HealthKit.

- https://developer.apple.com/programs/whats-included/ (checked 2026-10-15)
- https://developer.apple.com/help/account/membership/fee-waivers/ (checked 2026-10-15)
- https://developer.apple.com/health-fitness/ (checked 2026-10-15)
- https://cloud.google.com/identity-platform/pricing (checked 2026-10-15)
- `docs/research/B3-oauth-testing-mode.md` in this repo

## Recommendation

**Build v1 on Health Connect, Android only, with no accounts.** The evidence supports it clearly, and it dissolves the problem B3 was written about rather than managing it.

B3's conclusion was to stay in Testing mode and accept that every user re-authorises weekly, because the alternatives were worse. This option removes the question entirely: no OAuth client means no publishing status, no 7-day refresh token, no 100-user cap, no CASA assessment, and health data that never leaves the device. For a friends-and-family app whose v1 has no leaderboards, that is a strictly better position on privacy, cost and user friction simultaneously — which is rare enough to be worth taking.

The costs are real but bounded, and three are worth accepting consciously:

1. **GPS routes cannot sync unattended.** `READ_EXERCISE_ROUTES` can't be requested programmatically and other apps' routes need per-route consent that fails in the background. Treat maps as an on-demand feature, not part of automatic sync.
2. **PR notifications become "next time you open the app."** No push exists; background read is a user-granted permission and still polling. The spec's notification timing has to change.
3. **We leave Expo Go for development builds**, and the React Native binding is community-maintained rather than official — the main implementation risk, and the first thing to spike.

Android-only is the right call for v1, and iOS later means HealthKit, not Google: $99/year and the same on-device, account-free shape. Do not route iOS through the Google Health API.

One thing this does *not* solve, covered in the companion document: with no cloud copy, the on-device database becomes the only durable record, and Health Connect's own read window is limited. Export and backup therefore move from "nice later" to v1 scope.

## Spec impact

Proposed changes to `docs/SPEC.md` in the SplitBook-App/app repo. Listed only — that file is in the other repo and is updated separately.

- **Finding 1 (no accounts) — Change §2 and §17.** State that Android v1 reads health data through Health Connect using Android runtime permissions, with no Google account, no OAuth client and no Google Cloud project. Supersede the §17 "Google OAuth and Google Health API" design for v1 and mark it as the cloud path retained for a possible later cloud mode.
- **Finding 2 (permissions and routes) — Change §2 and §5.** List the four permissions. Record that heart rate and distance are separate records joined by the session's time range, that `ExerciseSessionRecord` aggregates only total duration so all other metrics are computed by us, and that GPS routes require per-route user consent and are unavailable in the background — so routes are an on-demand feature, not part of automatic sync.
- **Finding 3 (Play) — Change §16.** Record the Play Health apps declaration, Data Safety section and published privacy policy as release prerequisites, and note there is no documented user cap or fee.
- **Finding 4 (Expo) — Change §3.** Record that the app requires an Expo development build (not Expo Go), targets Android 9+ with framework-level Health Connect on Android 14+, must install-prompt on Android 13 and lower, and must degrade gracefully where Health Connect is unavailable.
- **Finding 5 (sync and notifications) — Change §6 and §11.** Replace webhook-driven delivery with foreground sync using per-data-type change tokens, a last-read timestamp fallback for 30-day token expiry, client-ID deduplication and resumable sync. Restate "new PR" notification timing as on next app open, or best-effort via an optional background-read permission.
- **Finding 6 (cost and iOS) — Change §16.** Record that iOS will use HealthKit, not the Google Health API, at $99/year for the Apple Developer Program, and that no Google account is required on either platform.
