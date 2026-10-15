# Health Data Restrictions: Consolidated Reference

Every restriction found so far that constrains how Splitbook reads, stores or ships health data, in one place for decision-making. Sourced from the B3 (#3) and B14 (#15) research documents in `docs/research/`.

Two paths are covered. **Health Connect** is the Android on-device API proposed for v1. The **Google Health API** is the cloud REST API originally specced, retained here because its constraints still apply if cloud mode is ever built. Apple is included for the later iOS route.

All pages checked on 2026-10-15. Where a restriction could not be confirmed on an official page, it says so.

## A. Health Connect — reading history

| # | Restriction | Exact rule | Impact |
|---|---|---|---|
| A1 | Default read window | Apps can read data "for up to 30 days prior to when any permission was first granted"; older reads error | Anchored at grant, not rolling: limits only the initial backfill. Data recorded after connecting stays readable however long the user leaves the app unopened |
| A2 | History permission | `READ_HEALTH_DATA_HISTORY` extends reads beyond 30 days | Must be requested for any meaningful backfill |
| A3 | Own-data exemption does not help | Android 14+: no limit reading the app's *own written* data, 30-day limit on other data | Splitbook only reads, never writes, so this exemption never applies to us |
| A4 | Uninstall resets everything | Uninstall revokes all permissions including history; on reinstall "the same default restrictions apply" | New phone or reinstall = history permanently truncated to 30 days |
| A5 | User-controlled auto-delete | "Data older than the selected time period you've set will be automatically deleted each day" | Outside our control; offered periods **could not be confirmed** |
| A6 | Health Connect is a hub, not an archive | "Other apps or devices may have a copy of your data" | Durable copy lives in the source app (Fitbit, Samsung Health, Strava), each with its own retention |

**Consequence:** re-syncing cannot recover deleted local data beyond 30 days, and never across a reinstall. The local database is the system of record.

Sources: [read-data](https://developer.android.com/health-and-fitness/health-connect/read-data), [Get started (help)](https://support.google.com/android/answer/12201227), [Delete data](https://support.google.com/android/answer/12201232), [Manage data](https://support.google.com/android/answer/12990553)

## B. Health Connect — when and how data can be read

| # | Restriction | Exact rule | Impact |
|---|---|---|---|
| B1 | No push | An app "can't get notified of new data" | Sync is polling; no event-driven PR detection |
| B2 | Foreground by default | "Apps can only read data from Health Connect while they are in the foreground" | Default sync happens on app open |
| B3 | Background read is a separate permission | `READ_HEALTH_DATA_IN_BACKGROUND`, gated on `FEATURE_READ_HEALTH_DATA_IN_BACKGROUND` being available | User must grant it; feature availability must be checked at runtime |
| B4 | Background read of other apps' data | **Could not confirm.** Google's rate-limiting page states an app calling from background "will be able to read only its own inserted records"; the read-data page does not say how the background permission interacts with other apps' data | If the restrictive reading is correct, background sync is useless to us — we read only Fitbit/Watch-written records. **Must be tested on a device before relying on background sync** |
| B5 | Change tokens expire | "An unused Changes token expires within 30 days" | Needs a last-read-timestamp fallback and client-ID dedupe. Only triggers when a user leaves the app unopened over 30 days, so it is easy to leave untested |
| B6 | Rate limits exist | Periodic and daily limits on read and changelog calls; "background rate limiting is stricter than foreground" | Numeric values are **not published**; use changelogs over raw reads, batch, and handle failure by resuming |
| B7 | Interruptible | "Access to Health Connect may be interrupted at any point" | Sync must be resumable mid-run |

Sources: [sync-data](https://developer.android.com/health-and-fitness/guides/health-connect/develop/sync-data), [read-data](https://developer.android.com/health-and-fitness/health-connect/read-data), [rate-limiting](https://developer.android.com/health-and-fitness/health-connect/rate-limiting)

## C. Health Connect — GPS routes (most restricted data type)

| # | Restriction | Exact rule | Impact |
|---|---|---|---|
| C1 | Cannot be requested programmatically | `READ_EXERCISE_ROUTES` "can only be granted manually by a user in Health Connect settings or in the route request activity. Attempts to request it by applications will be ignored" | No standard runtime permission prompt; user must grant in settings, or per route |
| C2 | Background reads blocked | Returns `ExerciseRouteResult.ConsentRequired` "even if your app has **Always allow** access" | Routes can never be fetched by a background job |
| C3 | Per-route consent fallback | A dialog grants "a one-time read of a route" | Without Always allow, one tap per workout |
| C4 | Foreground with Always allow | Restriction is documented as background-only, so foreground reads should work without per-route prompts — but Google never states this explicitly. **Could not confirm**; always handle `ConsentRequired` | If it holds, routes sync silently whenever the app is open and only background map-filling is lost. Needs a device test to settle |
| C5 | Write overwrites | Updating a session without a route deletes the existing route | Never write back sessions |

Sources: [exercise-routes](https://developer.android.com/health-and-fitness/health-connect/features/exercise-routes)

## D. Health Connect — platform and distribution

| # | Restriction | Exact rule | Impact |
|---|---|---|---|
| D1 | Android version floor | SDK supports Android 8 (API 26)+, but the Health Connect app requires Android 9 (API 28)+ | Only Android 9+ users can actually use it |
| D2 | Pre-Android 14 setup | Framework module on Android 14 (API 34)+; on Android 13 and lower the user must install the Health Connect app from Play | Must detect availability and prompt to install |
| D3 | No Expo Go | Expo Go "cannot use third-party libraries that require custom native code" | Development build required; changes how users install |
| D4 | Community binding | `react-native-health-connect` is not an official Google or Expo package | **Could not confirm** maturity, plugin options, route/changelog support. Main implementation risk |
| D5 | Play declaration | Health apps declaration form, Data Safety section, published privacy policy; "All access requests for health & fitness and body sensor permissions will be subject to review" | Required for Play distribution; re-reviewed when data types change |
| D6 | Minimum scope rule | "Request the minimum data types needed and provide a valid use case for each" | Denials cite over-requesting; justify each permission |
| D7 | No user cap or fee | Neither is stated anywhere in Health Connect or Play health documentation | Materially better than the Google Health API's hard 100-user cap |
| D8 | Thin session aggregates | `ExerciseSessionRecord` supports only `EXERCISE_DURATION_TOTAL` | All other metrics computed by us from raw records |
| D9 | Split record types | Heart rate and distance are separate records joined by the session's time range | Sync must stitch them together |

Sources: [get-started](https://developer.android.com/health-and-fitness/health-connect/get-started), [publish](https://developer.android.com/health-and-fitness/health-connect/publish), [Play health permissions](https://support.google.com/googleplay/android-developer/answer/12991134), [declaration form](https://support.google.com/googleplay/android-developer/answer/14738291), [Expo libraries](https://docs.expo.dev/workflow/using-libraries/)

## E. Google Health API (cloud path — not used in v1)

Retained because these apply if cloud mode is ever built. Full detail in `docs/research/B3-oauth-testing-mode.md`.

| # | Restriction | Exact rule | Impact |
|---|---|---|---|
| E1 | 7-day refresh token | External user type + Testing status issues "a refresh token expiring in 7 days" | Every user re-authorises weekly |
| E2 | Identity-scope exception only | Applies only to apps requesting solely `openid`/`email`/`profile` | Does not help us; any health scope removes it |
| E3 | 100 test users | Hard cap, each email added manually | Bounds a friends-and-family rollout |
| E4 | 100 lifetime users unverified | "100 new users in total", applying "over the entire lifetime of the project", and it "cannot be reset or changed" | Publishing unverified does not raise the ceiling; slots are spent permanently |
| E5 | Unverified warning screen | Unverified-app / "Danger UI" screen before consent; app name and logo hidden | Exact wording **could not be confirmed** (shown only as an image) |
| E6 | Restricted scopes | "Most scopes for the Google Health API are restricted" | Per-scope classification is **not published**; read it from Cloud Console |
| E7 | Security assessment | Annual third-party CASA assessment, Letter of Validation, required to exceed 100 users | ~$500–$4,500/yr; ~6 weeks (Google's estimate) |
| E8 | Token limits | 100 refresh tokens per account per client ID (oldest silently invalidated); unused tokens lapse after six months | Affects multi-device users |

Sources: [OAuth 2.0](https://developers.google.com/identity/protocols/oauth2), [Manage app audience](https://support.google.com/cloud/answer/15549945), [app state overview](https://developers.google.com/identity/protocols/oauth2/production-readiness/overview), [Health API verification](https://developers.google.com/health/app-verification), [verification FAQ](https://support.google.com/cloud/answer/13463817)

## F. Apple / HealthKit (out of scope)

**Decision (2026-10-15): iOS is out of scope.** The $99/year Apple Developer Program is not justified for a hobby project. This section is kept as costing only, should that ever be revisited.

| # | Restriction | Exact rule | Impact |
|---|---|---|---|
| F1 | Developer Program fee | 99 USD per membership year; required to distribute apps at all | Not a HealthKit fee; the cost of shipping any iOS app |
| F2 | Fee waivers | Available to nonprofits, accredited educational institutions and government entities | May not apply to us |
| F3 | No Google account | HealthKit is on-device with per-type authorisation and `NSHealthShareUsageDescription` strings | Confirms neither platform needs a Google sign-in |

Sources: [Membership](https://developer.apple.com/programs/whats-included/), [Fee waivers](https://developer.apple.com/help/account/membership/fee-waivers/), [Health & fitness](https://developer.apple.com/health-fitness/)

## Open questions needing a device test

These cannot be settled from documentation and change the design if they go the wrong way:

1. **B4** — can a background read with `READ_HEALTH_DATA_IN_BACKGROUND` return records written by other apps? If not, background sync is worthless to us.
2. **C4** — does foreground reading with "Always allow" avoid the per-route consent dialog?
3. **A5** — which auto-delete periods Health Connect offers users.
4. **B6** — the actual numeric rate limits.
5. **D4** — whether `react-native-health-connect` exposes routes, changelogs and the history and background permissions.
6. Whether Health Connect genuinely requires no Google account at all (strongly implied, never stated).
