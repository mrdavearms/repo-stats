# Repo Stats — Project Instructions

## What This Is

A self-hosted GitHub Pages dashboard tracking traffic (views, clones, referrers, paths), release downloads, and star counts for four public repos (`bulk-pdf-extractor-and-generator`, `student-doc-redactor`, `naplan-cohort-tracker`, `naplan-cohort-tracker-releases`). No backend, no database, no build step. The PAT still needs `repo` scope — GitHub's traffic API requires it even for public repos. (Historical note: `naplan-cohort-tracker` used to be a private, email-only repo handled by a dedicated block that fetched fresh at send time and never persisted; it went public and was promoted to a normal tracked repo on 2026-07-14, so it now persists history and appears on the dashboard like the others.)

**NAPLAN is two repos, and that is deliberate.** `naplan-cohort-tracker` is the source repo: its `release.yml` builds the installers and creates every release as a **draft** (`releaseDraft: true`) so they can be reviewed, then mirrors the public downloads — plus a Pages install page — to `naplan-cohort-tracker-releases`. So the source repo has **zero public releases by design** and the mirror holds all the real download numbers. Both are tracked: source for traffic/stars, mirror for downloads. Do not "fix" the drafts by publishing them — that creates a duplicate public download point outside the install page and outside the mirror's stats. (This was misdiagnosed as a broken release pipeline on 2026-07-20; it is not.)

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
Each repo has: `releases[]` (current snapshot of all releases/assets; every asset carries `installer`, `updater` and `platform` flags written by the collector), `history[]` (`{date, total_downloads, update_checks}` — cumulative daily), `asset_history[]` (`{date, assets: [{tag, name, platform, download_count}]}` — daily per-installer-asset snapshot, filtered by `INSTALLER_REGEX`). Entries written before 2026-09-22 lack the flags, `update_checks` and the `tag`/`platform` fields; the dashboard falls back to name-based classification for those. NOTE: `bulk-pdf-extractor-and-generator` reuses the same asset file name in every release, so pre-2026-09-22 `asset_history` entries for it cannot be attributed to a version.

**Asset classification lives in one place: the collector.** `INSTALLER_REGEX` and `UPDATER_REGEX` (job-level env) stamp each asset with `installer` / `updater` / `platform` (windows | mac | linux | other) in `releases[]`. The dashboard and the email read the flags. The dashboard's `isInstaller()` / `platformOf()` helpers carry a copy of the rules only as a fallback for old data — if you change the regexes, change the helpers too.

**Update checks.** `update_checks` is the summed `download_count` of auto-updater manifests (`latest.yml`, `latest-mac.yml`, `latest.json`). An installed copy fetches one of these every time it checks for a new version (typically each launch), so it is the best available proxy for the *active installed base*. It is a count of checks, not of machines.

**Bulk PDF's update checks are NOT measurable, and its 0 is not a usage figure.** Its releases carry no updater manifest; the app's `check_for_update()` in `pdf_generator.py` sends a HEAD request to `github.com/.../releases/latest` (a redirect), deliberately avoiding api.github.com because a school shares one IP and would hit the 60/hour unauthenticated limit. GitHub counts that request nowhere: not in asset downloads, not in traffic views/paths (verified 2026-09-22: `/releases/latest` has never appeared in its popular paths). The email and dashboard therefore say "not measurable" for any app with no updater asset (`has_updater` / `hasUpdater()`), never 0. If a measurable figure is ever wanted for Bulk PDF, the app would have to fetch a small manifest file from its release instead.

**Download counting (important).** `total_downloads` (and the NAPLAN email figure) counts only real **installer/package** assets — those matching `INSTALLER_REGEX` (job-level env in the workflow: `.dmg/.exe/.msi/.pkg/.appimage/.deb/.rpm/.zip`, case-insensitive). Auto-updater manifests (`latest.json`, `latest*.yml`), detached signatures (`.sig`), electron diff blockmaps (`.blockmap`) and Tauri/electron update bundles (`.tar.gz`) are **excluded** — they are fetched automatically by installed apps/CI and would otherwise inflate the count several-fold (e.g. a Tauri app with 8 assets reported 8× its real installs). `releases[]` still stores **all** assets (the dashboard's Release Assets table shows everything); only the aggregate total is filtered. The filter is defined once in `INSTALLER_REGEX` (job-level env) and applied during collection for every tracked repo; the separate NAPLAN email block that also used it was removed on 2026-07-14. Draft releases are filtered out before this stage — see "Things to Watch Out For". Caveat: `.zip` is assumed to be an app package, so a sample-data `.zip` attached to a release would be miscounted as an install. This filter applies **going forward only** — historical `history[]` points collected before the fix remain inflated and are not recomputed (per-asset history was never stored, so they can't be reconstructed without fabricating data). Expected one-time artefact: on the first run after the fix, the corrected (lower) total replaces the inflated one, so the dashboard's "since yesterday" download trend shows a transient negative delta for ~1–2 days (e.g. student-doc-redactor 159 → 134). This is the metric correction, not lost downloads.

## Dashboard Features

- Repo selector (All / individual) and date range filter (7d / 30d / 90d / All)
- 8 summary cards with trend indicators (installer downloads with a burst-adjusted "≈ N excluding bursts" note, update checks, views, unique visitors (daily sum), clones, stars, forks, watchers)
- 4 charts (new downloads per day as stacked bars with suspected bursts in red, cumulative downloads, views, clones)
- 6 tables (downloads by version with Windows/Mac split + update checks, suspected automated bursts, referrers, popular paths, release assets with a type column, recent stars)
- A plain-English "What these numbers can and cannot tell you" section at the foot of the page
- CSV export button (respects current filters; includes forks/watchers columns)
- Dark mode, mobile responsive

## Email Report

**The email is the primary product; the dashboard is secondary.** It is built in the "Send daily email summary" step from the committed `data/*.json` files plus `/tmp/collect_warnings.txt` (written by the collect step's `warn()` on every fetch failure).

It shows **one card per app**, not per repo: the `APPS` array maps a label to a traffic repo and a downloads repo (`label|traffic_repo|downloads_repo`). NAPLAN is one card fed by `naplan-cohort-tracker` (views/clones/stars/referrers) and `naplan-cohort-tracker-releases` (downloads/update checks). When adding a repo, add it to `REPOS` in the collect step, to `APPS` here, and to the dashboard.

Layout, top to bottom:
- **Warning banner** (orange) if any fetch failed today or an app's newest page-view day is 4+ days old. The subject gets "- data incomplete".
- **At a glance**: yesterday's installer downloads across all apps with a Windows/Mac split, the 7-day figure, 7-day update checks, and a count of flagged jumps. The subject line carries the yesterday figure.
- **Per app**: stars/forks/watching; three cards **Views (date)**, **Clones (date)**, **Downloads (all-time)** with the change since the previous snapshot; then **New downloads** (yesterday, per version and platform, e.g. "v1.9.2 Windows ×2, v1.9.2 Mac ×1", plus update checks), **Last 7 days** (downloads with Windows/Mac split and top versions; update checks; views/unique/clones; views trend vs last week), **All-time** (downloads with platform split, update checks, views), **Where visitors came from (14d)** (top 5 referrers with unique counts), **Release pages (14d)** (views/unique of `/releases*` paths = people who went looking for the download) and **Top pages**.
- **Legend** explaining what downloads, update checks, views/clones and referrers do and do not mean.

**Rules the email follows (keep them):**
- **Views/clones show the latest COMPLETE day GitHub has published, labelled with its date** (`latest_day()`), never "yesterday". GitHub's traffic feed lags 1–2 days; the old email read `.date == YESTERDAY` and silently showed 0 on lag days.
- **Per-version download deltas** (`asset_window()`) compare the latest `asset_history` snapshot with the latest snapshot on/before the cutoff, keyed by `tag|name`. Snapshots written before 2026-09-22 have no `tag`; they are matched by name only, and if names repeat (bulk-pdf) the function returns no baseline and the email falls back to the plain history total with "per-version split not available yet". This clears itself as tagged history accrues (yesterday from 2026-09-23, 7-day window from 2026-09-29).
- **Update checks** deltas (`checks_window()`) return empty, shown as "not enough history yet", when either side predates the field. Never 0.
- **Negative daily delta** is reported as "GitHub revised the total down by N", not as a red loss.
- **A one-day jump of 15+** is orange "unusually large, likely a crawler" and adds "- check for crawler" to the subject.
- **"Yesterday" wording** is only used when the previous snapshot really is yesterday's; otherwise "since <date>" / "Since last report".

**Offline test** (no mail, no API): extract the step, stub `curl`, run it in a copy of the repo. See "Common Tasks". To exercise the per-version deltas, clone the last `asset_history` entry as yesterday with a few counts subtracted.

## Things to Watch Out For

- **Download counts are fetches, not people, and crawlers sweep the installers.** GitHub has no unique-downloader figure. The count includes a person downloading twice, a Windows copy auto-updating (both electron-updater and the Tauri updater fetch the full installer), and security scanners / crawlers. Confirmed crawler sweeps: 2026-05-27/28 (+92 on one 600 MB Doc Redactor .dmg and +42 on one Bulk PDF .dmg, same two days) and 2026-07-10 (every Bulk PDF Mac .dmg back to v2.7 gained exactly +1 in one day). The dashboard's burst rule (day gains ≥ 15 AND ≥ 8× the repo's median active day, `detectBursts()` in `index.html`) flags these and the Downloads card shows a total with each burst day replaced by a typical day. The rule is a heuristic: a genuine launch day would be flagged too, which is why it says "suspected". GitHub also revises counts downwards occasionally (e.g. 155 → 148 on 2026-05-28), which shows as a negative day.
- **Mac download counts are the least trustworthy.** Both crawler sweeps hit `.dmg` files only. As of 2026-09-22 Mac counts were 182 of 301 for Doc Redactor and 103 of 160 for Bulk PDF; Windows counts (119 and 57) and NAPLAN's (75 Windows / 9 Mac) look like people.
- **Absence of data is never zero.** Two separate bugs came from the same mistake and both are now guarded — keep the rule in mind before adding any new metric:
  1. **Draft releases are excluded from download counts** (`select(.draft | not)` when building the release snapshot). Draft assets are reachable only with push access, so their `download_count` is the maintainer/CI, never the public. Counting them reported phantom installs for `naplan-cohort-tracker`.
  2. **A failed fetch skips its write rather than recording 0.** All API calls go through `gh_fetch()` (5 attempts, linear backoff, non-zero return on total failure); each write is guarded by a `*_OK` flag. A GitHub partial outage on 2026-07-20 previously wrote `0 downloads` over the mirror's true 72 and zeroed star/fork/watcher counts. `views`/`clones`/`star_events` merge additively so an empty body is already a no-op there and they need no guard.
  To verify the guards still hold, stub `curl() { return 22; }` over the extracted collection step against a copy of `data/` and assert both JSON files come back byte-identical.
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
