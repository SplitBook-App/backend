# B3 Research: Google OAuth Testing Mode and Unverified App Limits

Researched what Google's OAuth "Testing" publishing status and an unverified "In production" app cost Splitbook in token lifetime, user cap and consent-screen warnings, which Google Health API scopes are needed for exercise sessions, heart rate, distance and GPS, and what verification would involve.

All sources below are official Google pages. All pages checked on 2026-10-15.

## 1. Do refresh tokens expire after about 7 days in Testing mode?

Yes. If the OAuth client belongs to a Google Cloud project whose user type is External and whose publishing status is Testing, Google issues a refresh token that expires in 7 days. Test-user authorizations themselves expire seven days from the time of consent, and the offline-access refresh token expires with them. In practice every Splitbook user would have to re-consent weekly.

Exact condition: External user type + Testing publishing status + at least one scope beyond basic identity.

The only exception Google documents: if the app requests *only* name, email and profile scopes (`userinfo.email`, `userinfo.profile`, `openid`, or their OpenID Connect equivalents), then users do not need to be on the test-user list, see no warning message, and their authorizations do not expire after 7 days. Requesting any other scope removes the exception — so it does not help Splitbook, which needs Health scopes. A separate exception exists for Google Workspace apps marked Trusted by a Workspace admin, which are treated as internal; this does not apply to consumer Google accounts.

Independently of Testing mode, refresh tokens can also stop working if the user revokes access, the token goes unused for six months, or the account exceeds 100 live refresh tokens per OAuth client ID (the oldest is then silently invalidated).

- https://developers.google.com/identity/protocols/oauth2 (checked 2026-10-15)
- https://support.google.com/cloud/answer/15549945 (checked 2026-10-15)

## 2. Maximum number of test users in Testing mode

100. Google describes a hard cap of up to 100 test users listed on the OAuth consent screen; each user's email address must be added manually to the Test users list in the project configuration. Only those users can authorize the app.

- https://support.google.com/cloud/answer/15549945 (checked 2026-10-15)
- https://developers.google.com/identity/protocols/oauth2/production-readiness/overview (checked 2026-10-15)
- https://developers.google.com/health/setup (checked 2026-10-15)

## 3. Publishing "In production" without verification, with sensitive or restricted scopes

**User cap.** Still 100, but a worse 100. An unverified app that presents the unverified-app screen is capped at 100 new users *in total*, and Google states the cap applies over the entire lifetime of the project and cannot be reset or changed. So publishing unverified does not raise the ceiling — it converts a 100-concurrent-test-user list into a one-way 100-user lifetime budget.

**Warning screen.** Yes. Users see an "unverified app" screen before the consent screen, and Google's app-state guidance calls this the "Danger UI" for apps requesting sensitive or restricted scopes. The app's name and logo are not shown on the consent screen while unverified. Google's own help page shows the screen only as an image and does not quote its button wording, so the exact labels are **Could not confirm** on an official page — I checked the Unverified apps help page and the OAuth app state overview; neither transcribes the text.

**Refresh token lifetime.** The 7-day expiry is tied to Testing status, so it goes away. Google's Health API setup page states that after moving to production, refresh tokens generally don't expire unless they are revoked, though a token left unused for a prolonged period (six months, per the OAuth guide) can lapse. Note this is Google's documented behavior for production status generally; neither page I found states a *separate* refresh-token lifetime for the unverified-but-published state specifically.

- https://support.google.com/cloud/answer/15549945 (checked 2026-10-15)
- https://support.google.com/cloud/answer/13463817 (checked 2026-10-15)
- https://support.google.com/cloud/answer/7454865 (checked 2026-10-15)
- https://developers.google.com/identity/protocols/oauth2/production-readiness/overview (checked 2026-10-15)
- https://developers.google.com/health/setup (checked 2026-10-15)

## 4. Google Health API scopes for exercise sessions, heart rate, distance and GPS

| Data Splitbook needs | Scope | Sensitive or restricted? |
|---|---|---|
| Exercise sessions | `https://www.googleapis.com/auth/googlehealth.activity_and_fitness.readonly` | Could not confirm per scope; see below |
| Distance | `https://www.googleapis.com/auth/googlehealth.activity_and_fitness.readonly` (same scope) | Could not confirm per scope; see below |
| Heart rate | `https://www.googleapis.com/auth/googlehealth.health_metrics_and_measurements.readonly` | Could not confirm per scope; see below |
| GPS route | `https://www.googleapis.com/auth/googlehealth.location.readonly` | Could not confirm per scope; see below |

Three read scopes cover all four needs. Google's data-type listing groups both Exercise and Distance under `activity_and_fitness`, and Heart Rate under `health_metrics_and_measurements`. `location.readonly` is described as viewing GPS location recorded during an exercise; route coordinates are downloaded via the `exportExerciseTcx` method, which returns a TCX file and requires that scope — without it, session aggregates in `metricsSummary` remain available but the route export returns an authorization error.

**Classification caveat.** Google does not publish a per-scope sensitive/restricted label for the Google Health API. The scopes reference lists each scope with a description and no classification, and the general OAuth 2.0 scopes list does not include `googlehealth.*` entries at all. What Google does state officially, on the Google Health API app verification page, is: "Most scopes for the Google Health API are restricted." The verification FAQ directs developers to read the approved scopes in their own Cloud Console project rather than to a published list. So: treat all four as restricted for planning purposes — that is the safe reading and matches Google's own summary — but the exact per-scope label is **Could not confirm** from documentation and should be read off the Cloud Console OAuth consent screen for the real project. For context, the legacy Google Fit API classed all of its fitness scopes as restricted.

- https://developers.google.com/health/scopes (checked 2026-10-15)
- https://developers.google.com/health/data-types (checked 2026-10-15)
- https://developers.google.com/health/data-types/workouts (checked 2026-10-15)
- https://developers.google.com/health/app-verification (checked 2026-10-15)
- https://support.google.com/cloud/answer/13463817 (checked 2026-10-15)

## 5. What verification would involve

Three parts, for restricted scopes:

1. **OAuth app verification.** Google's Trust and Safety team reviews the app's identity, each requested scope and its justification. Vague or duplicated justifications cause delays. Brand verification (app name, logo, domain ownership) is a prerequisite and typically takes 2-3 business days.
2. **In-app disclosure.** The app must tell users, inside normal app use, what health and fitness data it accesses, how it is used and how it is shared. A privacy policy alone does not satisfy this.
3. **Third-party security assessment.** Required annually for apps using restricted scopes that store or transmit restricted-scope data on servers — which Splitbook does, since the backend holds the data. It is run by a Google-empanelled assessor against the App Defense Alliance CASA framework (based on OWASP ASVS). Passing produces a Letter of Validation / Letter of Assessment, submitted back to Trust and Safety. Existing HITRUST, SOC 2 or ISO/IEC 27001 certification may shorten it but does not replace the assessor. Reverification and reassessment are required at least every 12 months.

Published figures: Google estimates restricted scope verification at about 6 weeks and says the process can take several weeks overall; the Health API page estimates roughly 2-3 weeks for Tier-2 apps and 4-6 weeks for Tier-3, with assessor fees of roughly $500 to $4,500 USD depending on complexity. Google notes these estimates are not guaranteed. The Google Health API setup page states plainly that supporting more than 100 users requires completing a third-party security review.

- https://developers.google.com/health/app-verification (checked 2026-10-15)
- https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification (checked 2026-10-15)
- https://support.google.com/cloud/answer/13463817 (checked 2026-10-15)
- https://developers.google.com/health/setup (checked 2026-10-15)

## Recommendation

**Stay in Testing mode and accept weekly re-sign-in.**

The decision looks like a trade between a weekly annoyance and no weekly annoyance, but that is not what is on offer. Both options are capped at 100 users, so publishing unverified buys no extra reach. What it does buy is the removal of the 7-day token expiry — and it pays for that with a cap that is permanent and non-resettable: every one of those 100 slots is consumed for the lifetime of the project, with no way to free one when a friend stops using the app or signs in from a second account. Testing mode's 100 is a list you edit; unverified production's 100 is a budget you spend. For a friends-and-family app that will iterate, re-create accounts, and test with throwaway logins, burning the lifetime budget during the prototype phase is the more expensive mistake.

Publishing unverified also puts the "unverified app" / Danger UI screen in front of every user, with the app's name and logo stripped from the consent screen — a worse first impression than the testing notice, and an actively bad one when the next screen asks for heart rate and GPS data.

Verification is the real fix, and it is not a near-term option: a third-party CASA assessment at roughly $500-$4,500 and several weeks of review is disproportionate to a friends-and-family build. It is the right move only if Splitbook goes public.

**What each choice costs the user:**

- **Testing mode (recommended):** every user re-authorizes roughly weekly. The app must handle `invalid_grant` on refresh as an expected state, not an error — prompt a clean re-consent rather than failing a sync. Backfill after re-consent must be able to pick up data missed during any gap, and the user must be on the 100-entry test-user list.
- **Unverified production:** no weekly re-sign-in, but every user passes an alarming unverified-app warning before granting health access, sees no app name or logo on the consent screen, and permanently consumes one of 100 lifetime slots that can never be reclaimed.

Revisit this if Splitbook heads for public launch; at that point verification becomes mandatory anyway, and the 7-day question disappears with it.

## Spec impact

Proposed changes to `docs/SPEC.md` in the SplitBook-App/app repo. Listed only — not edited here, since that file lives in the other repo.

- **Finding 1 (7-day refresh token expiry) — Change §17 "Google OAuth and Google Health API".** State that the app runs in Testing publishing status and that refresh tokens expire 7 days after consent, so users re-authorize weekly. Add that the backend must treat `invalid_grant` on token refresh as an expected condition that triggers a re-consent prompt, not a hard failure, and that sync must backfill data covering the unauthorized gap.
- **Finding 2 (100 test users) — Change §17.** Record the hard cap of 100 test users and that each address is added manually to the Test users list; note this bounds the friends-and-family rollout.
- **Finding 3 (unverified production limits) — Change §17.** Record the decision to stay in Testing, with the reason: unverified production carries a non-resettable 100-user lifetime cap and shows an unverified-app warning. Note that moving to production removes the 7-day expiry but is not planned.
- **Finding 4 (scopes) — Change §2 and §17.** List the exact scopes the app requests: `googlehealth.activity_and_fitness.readonly` (exercise sessions and distance), `googlehealth.health_metrics_and_measurements.readonly` (heart rate), `googlehealth.location.readonly` (GPS routes, required for `exportExerciseTcx`). State that these are treated as restricted scopes. §2 should reflect that GPS routes arrive as TCX via a separate export call rather than as plain data points.
- **Finding 5 (verification) — None.** No spec change now. Verification is out of scope for the friends-and-family build; revisit only on a public-launch decision.
