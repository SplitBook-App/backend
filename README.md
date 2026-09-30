# Splitbook Backend

The backend for [Splitbook](https://github.com/SplitBook-App/app), a free app that keeps all your fitness personal records in one place.

It receives completed workouts from the Google Health API, stores them, calculates records and best efforts, and sends push notifications when a new record is set.

> **Status:** planning and verification. The backend has not been built yet. The full design is in the app repo's [spec](https://github.com/SplitBook-App/app/blob/main/docs/SPEC.md).

## Tech stack

| Part | Technology |
|---|---|
| Framework | Python + FastAPI |
| Hosting | Google Cloud Run (Docker container) |
| Background work | Google Cloud Tasks |
| Scheduled jobs | Google Cloud Scheduler |
| Database | Neon Postgres, via SQLAlchemy and Alembic |
| Raw workout streams | Google Cloud Storage |
| Data source | Google Health API, via Google OAuth 2.0 |
| Notifications | Expo push notifications |

## How it works

1. A user finishes a workout and it syncs to Google Health.
2. Google Health notifies the backend's webhook. The webhook replies immediately and queues the work in Cloud Tasks.
3. The task fetches the workout's summary, laps, heart rate, distance and GPS.
4. Raw streams go to Cloud Storage; summaries and splits go to Postgres.
5. Records for that workout type are recalculated, and a push notification is sent if a record was broken.

## Planned modules

| Module | Responsibility |
|---|---|
| `webhook` | Receives Google Health notifications and queues processing |
| `google_client` | OAuth token storage and refresh, Google Health API calls |
| `ingestion` | Fetches and stores workouts, mirrors edits and deletes |
| `records` | Pure Python calculation of splits, records, efficiency scores and GPS flags |
| `notifications` | Sends Expo push messages |
| `api` | Endpoints the app reads from |

## Repository layout

```
backend/
├── .env.example         # Template for local environment variables (no real values)
├── CONTRIBUTING.md      # How to work on this repo
└── README.md
```

The FastAPI project will be added in a later issue.

## Getting started

Setup instructions will be added once the FastAPI project is created. You'll need:

- Python 3 and a virtual environment
- Docker, for running the container locally
- Access to the Splitbook Google Cloud project (ask the maintainer)

To prepare your local environment, copy `.env.example` to `.env` and fill in the values you're given. Never commit `.env`.

## Security

- No secrets are stored in this repository. Locally they live in `.env`; in production they live in Google Secret Manager or Cloud Run settings.
- Every webhook notification is verified as coming from Google before it's processed.
- No real health data is committed, including test fixtures.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

To be decided.