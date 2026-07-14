# Repo Stats — Project Instructions

## What This Is

A self-hosted GitHub Pages dashboard tracking traffic (views, clones, referrers, paths), release downloads, and star counts for three public repos (`bulk-pdf-extractor-and-generator`, `student-doc-redactor`, `naplan-cohort-tracker`). No backend, no database, no build step. The PAT still needs `repo` scope — GitHub's traffic API requires it even for public repos. (Historical note: `naplan-cohort-tracker` used to be a private, email-only repo handled by a dedicated block that fetched fresh at send time and never persisted; it went public and was promoted to a normal tracked repo on 2026-07-14, so it now persists history and appears on the dashboard like the others.)

**Live:** https://mrdavearms.github.io/repo-stats/
**Repo:** https://github.com/mrdavearms/repo-stats

## Architecture

- **Single workflow** (`.github/workflows/collect-stats.yml`) — pure bash/curl/jq, no external Actions beyond `actions/checkout@v5`. Runs daily at 06:00 UTC via cron + manual `workflow_dispatch`.
- **Two JSON data files** (`data/traffic.json`, `data/releases.json`) — the "database". The Action reads, merges, deduplicates, and commits them back. Git history is the backup.
- **Single HTML dashboard** (`index.html`) — vanilla HTML/CSS/JS + Chart.js from CDN. No npm, no framework, no build.
- **HTML email** — sent after each collection run via Gmail SMTP (curl). Dark themed with card layout.

## Key Design Decisions

- **No external dependencies**: The workflow uses only curl and jq (pre-installed on GitHub runners). The dashboard uses only Chart.js from CDN. This is intentional — keep it zero-maintenance.
- **Deduplication by date**: Traffic data merges by replacing existing entries for the same date (GitHub updates current-day counts on subsequent calls). Never duplicate a date.
- **Referrer `unique` field can be null**: GitHub's API sometimes returns null for referrer unique counts. The dashboard displays `—` and the workflow coalesces to 0.
- **YAML-safe shell**: Heredocs with colons break YAML parsing. All email/HTML content is built via `printf` and shell variables, never raw heredocs with colons at line start.

## Repo Secrets (3 required)

| Secret | Purpose |
|--------|---------|
| `GH_STATS_TOKEN` | PAT (classic) with `repo` scope — needed for traffic API even on public repos |
| `GMAIL_ADDRESS` | Gmail address for daily email reports |
| `GMAIL_APP_PASSWORD` | Gmail App Password (not regular password) for SMTP |

The workflow checks token expiry via the `github-authentication-token-expiration` response header and warns in the email 30 days before expiry.

## Tracked Repos

**Public repos** (persisted to `data/*.json` + shown on dashboard + emailed) are defined in two places that must stay in sync:
1. **Workflow**: `REPOS` array (line ~25) and email loop repo list
2. **Dashboard**: `REPO_NAMES` array and the repo selector buttons in HTML

When adding a new public repo, update both files.

**Private / email-only repos:** there are currently none. This project previously supported an email-only mode (used for `naplan-cohort-tracker` while it was private) via a dedicated block in the "Send daily email summary" step that fetched stats fresh at send time and persisted nothing, keeping the repo off the public dashboard and out of git. That block was removed on 2026-07-14 when NAPLAN went public and was promoted to a normal tracked repo. If a genuinely private repo needs email-only tracking again, restore that pattern (fetch fresh at send time, build the row with `build_repo_row`, persist nothing) rather than adding it to the `REPOS` array — see git history for the original implementation.

## Data Shapes

### traffic.json
Each repo has: `views[]`, `clones[]`, `referrers[]`, `popular_paths[]`, `stars[]`, `forks[]`, `watchers[]`, `star_events[]`
- views/clones: `{date, total, unique}` — daily entries, sorted by date
- referrers: `{date, sources: [{referrer, count, unique}]}` — daily snapshots of top 10. NOTE: GitHub's referrers API returns the unique field as `uniques` (plural); the collector reads `.uniques` (a long-standing bug read `.unique`, recording 0 for every referrer — fixed; historical points stay 0).
- popular_paths: `{date, paths: [{path, count, unique}]}` — daily snapshots of top 10
- stars / forks / watchers: `{date, count}` — daily count. Watchers = the repo's `subscribers_count` (the real "Watch" count), NOT the API's legacy `watchers_count` (which aliases stars).
- star_events: `[{date, login}]` — one row per stargazer (when they starred + who), deduped by login. Built from the `stargazers` endpoint with the `application/vnd.github.star+json` media type, which adds `starred_at`. This is true historical star timing (not just a daily count). Stargazers of a public repo are already public, so the login is not a leak.

### releases.json
Each repo has: `releases[]` (current snapshot of all releases/assets), `history[]` (`{date, total_downloads}` — cumulative daily), `asset_history[]` (`{date, assets: [{name, download_count}]}` — daily per-installer-asset snapshot, filtered by `INSTALLER_REGEX`, so you can see which installer/platform people choose over time)

**Download counting (important).** `total_downloads` (and the NAPLAN email figure) counts only real **installer/package** assets — those matching `INSTALLER_REGEX` (job-level env in the workflow: `.dmg/.exe/.msi/.pkg/.appimage/.deb/.rpm/.zip`, case-insensitive). Auto-updater manifests (`latest.json`, `latest*.yml`), detached signatures (`.sig`), electron diff blockmaps (`.blockmap`) and Tauri/electron update bundles (`.tar.gz`) are **excluded** — they are fetched automatically by installed apps/CI and would otherwise inflate the count several-fold (e.g. a Tauri app with 8 assets reported 8× its real installs). `releases[]` still stores **all** assets (the dashboard's Release Assets table shows everything); only the aggregate total is filtered. The filter is defined once in `INSTALLER_REGEX` and used by both the public-repo collection and the NAPLAN email block. Caveat: `.zip` is assumed to be an app package, so a sample-data `.zip` attached to a release would be miscounted as an install. This filter applies **going forward only** — historical `history[]` points collected before the fix remain inflated and are not recomputed (per-asset history was never stored, so they can't be reconstructed without fabricating data). Expected one-time artefact: on the first run after the fix, the corrected (lower) total replaces the inflated one, so the dashboard's "since yesterday" download trend shows a transient negative delta for ~1–2 days (e.g. student-doc-redactor 159 → 134). This is the metric correction, not lost downloads.

## Dashboard Features

- Repo selector (All / individual) and date range filter (7d / 30d / 90d / All)
- 7 summary cards with trend indicators (views, unique visitors, clones, downloads, stars, forks, watchers)
- 3 charts (downloads, views, clones over time)
- 4 tables (referrers, popular paths, release assets, recent stars)
- CSV export button (respects current filters; includes forks/watchers columns)
- Dark mode, mobile responsive

## Email Report

The daily email (`build_repo_row` helper) shows per repo:
- Meta line: stars · forks · watching
- 3 cards: **Views (yesterday)**, **Clones (yesterday)**, **Downloads** (all-time, with a coloured "▲ +N today" delta from the last two `history` points)
- Summary line: **Last 7d** views/unique/clones + **All-time**
- Engagement line: **Top sources** + **Top pages** (top 3 each, from the latest referrer/path snapshot)

**Why "yesterday", not "today":** GitHub buckets traffic by UTC midnight and lags by hours, so the current-UTC-day ("today") count reads ~0 and is misleading. Yesterday is the most recent *complete* day. The 7-day window uses `date >= WEEK_AGO`.

## Things to Watch Out For

- **GitHub retains traffic for only 14 days.** If the Action is broken for 14+ days, data is permanently lost. Check Action health if you haven't looked in a while.
- **Clone counts include your own git pulls/fetches.** The "unique" count is more meaningful for external interest.
- **Don't put colons at the start of lines in workflow `run:` blocks** — YAML interprets them as mapping keys. Use `printf` instead.
- **The workflow commits to `main` directly** (via the bot). If you're working on changes locally, pull before pushing to avoid conflicts.
- **A stale local clone makes the data look "frozen" even when nothing is broken.** The bot pushes a new data commit every day, so a local checkout goes out of date fast. Before concluding the pipeline is dead because the latest date looks old, `git fetch` and compare `HEAD` vs `origin/main` — the live data lives on the remote, not your working copy. (This is the usual cause of a "my stats stopped updating / look wrong" scare.)
- **Views/clones lag GitHub's API by ~1–2 days**, so the newest `views`/`clones` date is normally a couple of days behind today's `stars`/`forks`/`watchers` date (those are written with `$TODAY` every run). This is GitHub's traffic pipeline, not a collection bug.
- **`actions/checkout` is pinned to `@v5`** (bumped from v4 on 2026-06-13, ahead of GitHub forcing Node 20 actions off after 2026-06-16). It's the only external Action; keep it current.
- **GitHub Pages CDN caches** for 1-2 minutes after a push. Hard-refresh if the dashboard seems stale.

## Common Tasks

```bash
# Trigger manual data collection
gh workflow run collect-stats.yml --repo mrdavearms/repo-stats

# Check recent Action runs
gh run list --repo mrdavearms/repo-stats --limit 5

# View logs for a run
gh run view <run-id> --repo mrdavearms/repo-stats --log

# Count collected data points
jq '.["bulk-pdf-extractor-and-generator"].views | length' data/traffic.json

# Validate workflow YAML (no yamllint/pyyaml on this machine; ruby is present)
ruby -ryaml -e "YAML.load_file('.github/workflows/collect-stats.yml'); puts 'OK'"

# Watch a run to completion (non-zero exit if it fails)
gh run watch <run-id> --repo mrdavearms/repo-stats --exit-status

# Offline-test the email step (no real mail/API): extract its run: script, then
# source it with a stubbed curl() — sample JSON for github URLs, return 0 for smtps://.
ruby -ryaml -e 'puts YAML.load_file(".github/workflows/collect-stats.yml")["jobs"]["collect"]["steps"].find{|s| s["name"]=="Send daily email summary"}["run"]' > /tmp/email_step.sh
bash -n /tmp/email_step.sh
```
