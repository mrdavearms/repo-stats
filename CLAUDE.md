# Repo Stats — Project Instructions

## What This Is

A self-hosted GitHub Pages dashboard tracking traffic (views, clones, referrers, paths), release downloads, and star counts for two public repos. No backend, no database, no build step.

**Live:** https://mrdavearms.github.io/repo-stats/
**Repo:** https://github.com/mrdavearms/repo-stats

## Architecture

- **Single workflow** (`.github/workflows/collect-stats.yml`) — pure bash/curl/jq, no external Actions beyond `actions/checkout@v4`. Runs daily at 06:00 UTC via cron + manual `workflow_dispatch`.
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

Defined in two places that must stay in sync:
1. **Workflow**: `REPOS` array (line ~25) and email loop repo list
2. **Dashboard**: `REPO_NAMES` array and the repo selector buttons in HTML

When adding a new repo, update both files.

## Data Shapes

### traffic.json
Each repo has: `views[]`, `clones[]`, `referrers[]`, `popular_paths[]`, `stars[]`
- views/clones: `{date, total, unique}` — daily entries, sorted by date
- referrers: `{date, sources: [{referrer, count, unique}]}` — daily snapshots of top 10
- popular_paths: `{date, paths: [{path, count, unique}]}` — daily snapshots of top 10
- stars: `{date, count}` — daily star count

### releases.json
Each repo has: `releases[]` (current snapshot of all releases/assets), `history[]` (`{date, total_downloads}` — cumulative daily)

## Dashboard Features

- Repo selector (All / individual) and date range filter (7d / 30d / 90d / All)
- 5 summary cards with trend indicators (views, unique visitors, clones, downloads, stars)
- 3 charts (downloads, views, clones over time)
- 3 tables (referrers, popular paths, release assets)
- CSV export button (respects current filters)
- Dark mode, mobile responsive

## Things to Watch Out For

- **GitHub retains traffic for only 14 days.** If the Action is broken for 14+ days, data is permanently lost. Check Action health if you haven't looked in a while.
- **Clone counts include your own git pulls/fetches.** The "unique" count is more meaningful for external interest.
- **Don't put colons at the start of lines in workflow `run:` blocks** — YAML interprets them as mapping keys. Use `printf` instead.
- **The workflow commits to `main` directly** (via the bot). If you're working on changes locally, pull before pushing to avoid conflicts.
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
```
