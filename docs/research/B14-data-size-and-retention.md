# B14 Research: Local Data Size and Retention

Researched how much space a year of workouts takes on a phone when the device is the only copy, how far back Health Connect lets an app read, and whether deleting old workouts locally can safely be undone by re-syncing.

All pages checked on 2026-10-15. Sources are official Android and Google Play documentation. The Health Connect platform findings live in `B14-health-connect-local-only.md`; this document covers only size and retention.

## 1. How far back can we read? (The constraint everything else follows from)

Health Connect limits how much history an app may read, and the limit resets in a way that matters:

- **Default window:** "By default, all applications can read data from Health Connect for up to 30 days prior to when any permission was first granted." Reading older records without further permission "results in an error."
- **Lifting it:** `PERMISSION_READ_HEALTH_DATA_HISTORY` (`android.permission.health.READ_HEALTH_DATA_HISTORY`) extends reads beyond 30 days.
- **Version split:** On Android 14+, an app has no historical limit reading *its own written* data, but a 30-day limit on other data. On Android 13 and lower, the 30-day limit applies to any data. Splitbook only reads, never writes, so the "own data" exemption does not help us.
- **Uninstall resets it:** "If a user deletes your app, all permissions, including the history permission, are revoked. If the user reinstalls your app and grants permission again, the same default restrictions apply." Google's own example: uninstall on 10 May, reinstall and re-grant on 15 May, and the earliest readable date becomes 15 April.

Two further limits sit on top. The user can set **auto-delete** in Health Connect — "Data older than the selected time period you've set will be automatically deleted each day" — which is outside our control; the exact periods offered are **Could not confirm**, as neither help page lists them. And Health Connect is a hub, not an archive: Google notes that "Other apps or devices may have a copy of your data," so the durable long-term copy lives in whichever app wrote the workout (Fitbit, Samsung Health, Strava), each with its own retention.

**The window is anchored, not rolling.** It is measured from the moment permission was first granted, so it limits only how far *back* the first sync can reach. Everything recorded after the user connects Splitbook stays readable indefinitely: someone who opens the app weekly, or once in six months, loses nothing. What truncates history is a reinstall, which resets the anchor, or the user's own auto-delete setting. The practical sync model that follows is in §5 of the companion document — opening the app is the sync trigger, and not opening it costs no data.

- https://developer.android.com/health-and-fitness/health-connect/read-data (checked 2026-10-15)
- https://support.google.com/android/answer/12201227 (checked 2026-10-15)
- https://support.google.com/android/answer/12201232 (checked 2026-10-15)
- https://support.google.com/android/answer/12990553 (checked 2026-10-15)

## 2. Can we delete locally and re-sync later?

**No — not reliably, and this answers the question directly.** The "delete after 90 days, re-sync on request" design does not work on Health Connect:

- Without `READ_HEALTH_DATA_HISTORY`, a re-sync recovers **30 days at most**. A workout deleted at 90 days is unrecoverable.
- With that permission, recovery depends on data still being in Health Connect — which the user's own auto-delete setting may have removed, and which we do not control.
- After any uninstall and reinstall, the window resets to 30 days regardless of what we held before. A user changing phones, or clearing and reinstalling the app, loses everything older.

So **the local database is the system of record, not a cache.** Anything we delete locally should be treated as gone for good. That inverts the retention question: the design must justify deleting, not justify keeping — and as §3 shows, there is no space pressure forcing a deletion policy at all.

It also makes **export and backup v1 scope rather than a later nicety.** With no cloud copy and no reliable re-sync, a lost or reset phone is total data loss today.

- https://developer.android.com/health-and-fitness/health-connect/read-data (checked 2026-10-15)

## 3. How big does this actually get?

Short answer to the "3–4 GB and it's not feasible" concern: **not close, provided we store normalised rows rather than raw JSON.** A year is tens of megabytes.

**Per-workout estimate**, one hour, from the record structures in the Health Connect reference (`HeartRateRecord` is a list of samples each carrying `time` and `beatsPerMinute`; an `ExerciseRoute.Location` carries `time`, `latitude`, `longitude`, `altitude`, `horizontalAccuracy`, `verticalAccuracy`), stored as typed SQLite columns with realistic row overhead:

| Stream | At 1 sample/sec | At 1 sample/5 sec |
|---|---|---|
| Heart rate | ~86 KB | ~17 KB |
| Distance | ~100 KB | ~20 KB |
| GPS route | ~200 KB | ~40 KB |
| Session row + computed summary | ~2 KB | ~2 KB |
| **Total per hour** | **~390 KB** | **~80 KB** |

**Per-year projection:**

| Profile | 1 sample/sec | 1 sample/5 sec |
|---|---|---|
| Typical: 4 workouts/week, ~50 min | ~69 MB | ~14 MB |
| Heavy: 6 workouts/week, ~75 min | ~156 MB | ~31 MB |

Even the heavy profile at full 1 Hz resolution is ~156 MB/year, so reaching 3 GB would take roughly two decades. The feasibility risk is not volume — it is **format**. Storing raw API responses as JSON blobs inflates this four- to fivefold (a 1 Hz year becomes ~300 MB and keeps compounding), which is how a design like this ends up at multiple gigabytes. Normalised numeric columns avoid that, and GPS routes are the largest single stream, so they are the first thing to make optional if space ever becomes a concern.

**Confidence.** These are modelled estimates from documented record structures, not measurements. The sampling rate of real Fitbit and Pixel Watch data is the main unknown — continuous heart rate is typically coarser than 1 Hz, which would push real numbers toward the right-hand column. #9 ("Fetch a real workout and measure it's size") owns the measured figures and should replace these once it runs. Note that #9 is currently blocked on the Google Cloud project and OAuth client; on the Health Connect path those measurements can be taken on-device with no cloud setup at all, which unblocks it.

- https://developer.android.com/reference/kotlin/androidx/health/connect/client/records/HeartRateRecord (checked 2026-10-15)
- https://developer.android.com/health-and-fitness/health-connect/features/exercise-routes (checked 2026-10-15)
- https://developer.android.com/health-and-fitness/health-connect/data-types (checked 2026-10-15)

## 4. Keeping it out of gigabytes: storage strategy

Size is a design choice here, not a given. Four rules, in order of how much they save:

**Rule 1 — Normalised typed columns, never JSON blobs.** Store each sample as integer columns in SQLite (`session_id`, `t_offset_ms`, `value`), not as a serialised API response. This alone is the 4–5x difference between ~69 MB and ~300 MB for a typical year, and it is the single decision that keeps a decade inside a few hundred megabytes.

**Rule 2 — Store time as an offset, not a timestamp.** Each sample needs only milliseconds since its session start, which fits in a 4-byte integer, rather than a full 8-byte epoch timestamp or an ISO-8601 string. An ISO string costs ~24 bytes per sample against 4 — on a 1 Hz year that is the difference between ~69 MB and well over 200 MB.

**Rule 3 — Downsample on ingest, keep the summary forever.** Nothing in the product needs per-second heart rate from two years ago; it needs the PR, the averages and the trend. So store two tiers:

| Tier | Contents | Size | Retention |
|---|---|---|---|
| Summary | Session row, computed metrics, PR records, heart-rate zone totals | ~2 KB per workout | Forever, never auto-deleted |
| Raw streams | Per-sample heart rate, distance, route points | ~80–390 KB per hour | Configurable; full resolution recent, downsampled or dropped when older |

At ~2 KB per workout, the summary tier is about **0.4 MB per year** — a lifetime of training history costs less than a single photo. The entire size question is therefore only ever about the raw tier.

**Rule 4 — Routes are the first thing to shed.** GPS is the largest single stream (~200 KB/hour at 1 Hz, over half the total). If space ever becomes a concern, dropping or simplifying old route geometry recovers more than anything else, and route lines survive heavy simplification with no visible change on a phone-sized map.

Applying rules 1 and 2 alone, with everything kept at full resolution, a heavy user's decade lands near 1.5 GB; with rule 3 trimming raw streams beyond a couple of years it stays in the low hundreds of megabytes indefinitely. Gigabyte territory is only reached by storing raw JSON and never trimming — which rules 1 and 3 exist to prevent.

## Recommendation

**Keep everything by default, and ship an export in v1.**

The retention policy follows from §2 rather than from space. Since deletion is irreversible and a year costs tens of megabytes, a 90-day cull would trade something valuable — the user's entire training history, and with it the trend and PR comparisons Splitbook exists to show — for savings that do not matter on a modern phone. A deletion policy is the right answer when storage is scarce or the data is recoverable; here neither holds.

Concretely:

1. **Default retention: keep all workouts indefinitely.** Never auto-delete session summaries, computed metrics or PR records — they are ~2 KB each and are the product.
2. **Request `READ_HEALTH_DATA_HISTORY`** so first install backfills more than 30 days. Treat that backfill as a one-off opportunity: whatever it captures may be the only copy we ever get.
3. **Ship export and import in v1.** A single file the user can save off-device. Without it, a lost phone is total loss, and it doubles as the migration path if cloud mode arrives later.
4. **If a retention control is offered, scope it to raw streams only** — the user may discard per-second heart rate, distance and route points beyond a period they choose, while summaries and PRs stay forever. Routes first, as the largest stream. Warn plainly at the point of choosing that discarded raw data cannot be re-synced.
5. **Store normalised typed columns, never raw JSON blobs**, per the four rules in §4. This is the single decision that keeps a decade of history inside a few hundred megabytes.

## Spec impact

Proposed changes to `docs/SPEC.md` in the SplitBook-App/app repo. Listed only — that file is in the other repo and is updated separately.

- **Finding 1 (read window) — Change §2 and §17.** Record the 30-day default read window, that `READ_HEALTH_DATA_HISTORY` is requested to extend it, that uninstall or reinstall resets it to 30 days, and that the user's own Health Connect auto-delete setting is outside our control.
- **Finding 2 (no reliable re-sync) — Change §3 and §16.** State that the on-device database is the system of record, not a cache, and that locally deleted data is unrecoverable. Add export and import of all user data as v1 scope, and state that a lost or reset device means total data loss without it.
- **Finding 3 (sizing) — Change §17.** Replace the Cloud Storage sizing assumptions for local mode with on-device figures: roughly 80–390 KB per workout hour depending on sample rate, roughly 14–156 MB per year. Add the design rule that raw streams are stored as normalised typed columns, never as JSON blobs, and cite #9 as the source of measured figures once available.
- **Finding 4 (retention policy) — Change §3.** Record the default of indefinite retention, that summaries and PR records are never auto-deleted, and that any user-configurable retention applies to raw streams only with an explicit warning that the data cannot be recovered.
- **Finding 5 (storage strategy) — Change §17.** Record the four storage rules: normalised typed columns rather than JSON blobs; sample times stored as integer offsets from session start; a two-tier split where the ~2 KB summary tier is kept forever and only raw streams are ever trimmed; and GPS route geometry as the first stream to downsample or drop under space pressure.
