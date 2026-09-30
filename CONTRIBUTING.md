# Contributing to Splitbook (backend)

Thanks for working on Splitbook. This guide covers how work moves from an issue to `main`.

## Where work comes from

- All work is tracked as issues on the **Splitbook v1** project board.
- Pick up issues assigned to you, and move the card as you go: **Todo → In progress → In review**.
- **Done** is set by the reviewer, not by the person doing the work.
- If something isn't covered by an issue, ask for one to be created before starting.

## Workflow

1. Make sure your local `main` is up to date: `git checkout main && git pull`.
2. Create a branch for the issue (see naming below).
3. Commit your work in small, clear steps.
4. Push the branch and open a pull request into `main`.
   - If you don't have write access to this repo, fork it, push the branch to your fork, and open the PR from there.
5. Link the issue in the PR description with `Closes #<issue-number>`.
6. Move the card to **In review** and wait for approval.
7. Address review comments with new commits on the same branch. A new push dismisses the previous approval, so the reviewer will approve again.
8. Once approved and all conversations are resolved, the PR is **squash-merged** into `main`.

Direct pushes, force-pushes and self-approval on `main` are not allowed.

## Branch naming

| Type | Pattern | Example |
|---|---|---|
| New feature | `feature/<short-name>` | `feature/webhook-receiver` |
| Bug fix | `fix/<short-name>` | `fix/pause-adjusted-split` |
| Documentation | `docs/<short-name>` | `docs/local-setup` |
| Setup or tooling | `chore/<short-name>` | `chore/ruff` |

Use lowercase and hyphens, and keep names short.

## Commits and pull requests

- Write commit messages in the imperative, describing what the commit does: "Add split calculation", not "Added stuff".
- One issue per PR. If a PR grows beyond its issue, split it.
- Fill in every section of the PR template.
- If a change alters behavior described in the spec (`docs/SPEC.md` in the app repo), open a matching PR there to update it, and link the two PRs.

## Tests

- Every change to the `records` module (splits, records, efficiency scores, pause handling, GPS flags) must include unit tests.
- `records` stays pure Python: no web, database or cloud code inside it, so it can be tested with sample data alone.
- Test data is made-up sample data, never real workouts.

Test tooling will be set up in a later issue.

## Research issues

Research issues are answered with a comment on the issue, not a PR. Use this format for each checklist item:

```
**Item:** <checklist item>
**Answer:** <what the official docs say, in your own words>
**Source:** <link to the official page>
**Checked on:** <date>
**Impact on spec:** None / Needs change — <what should change>
```

Only official documentation counts as a source. If something can't be confirmed, write "Could not confirm" and explain what you found.

## Secrets and data

- **Never commit secrets:** OAuth client secrets, refresh tokens, database URLs with passwords, service account keys or API keys.
- Local values go in `.env`, which is ignored by git. `.env.example` lists variable names only, with no real values. When you add a new variable, add its name to `.env.example` in the same PR.
- In production, secrets live in Google Secret Manager or Cloud Run settings, never in the repo.
- **Never commit real health data**, including your own workouts as test files or logs.
- Don't log tokens or raw health data.
- If you ever commit a secret, **tell the maintainer immediately.** Deleting the file isn't enough, because git history keeps it; the secret must be rotated.

## Code style

Linting and formatting tools will be set up in a later issue. Until then, keep code consistent with the surrounding files and use type hints on all functions.

## Questions

Ask in the issue you're working on, so the answer stays with the work.