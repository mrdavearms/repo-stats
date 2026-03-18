# GitHub Repo Stats

A self-hosted dashboard that tracks download counts, page views, unique visitors, referral sources, and clone data for two public GitHub repositories. Hosted on GitHub Pages with no backend, no database, and no build step.

**Live dashboard:** https://mrdavearms.github.io/repo-stats/

## Tracked Repositories

- [bulk-pdf-extractor-and-generator](https://github.com/mrdavearms/bulk-pdf-extractor-and-generator)
- [student-doc-redactor](https://github.com/mrdavearms/student-doc-redactor)

## Architecture

```
repo-stats/
├── .github/workflows/
│   └── collect-stats.yml      # GitHub Action — runs daily via cron
├── data/
│   ├── traffic.json           # Accumulated traffic data (views, clones, referrers, paths)
│   └── releases.json          # Accumulated release download counts
├── index.html                 # Dashboard (single HTML file, loads JSON via fetch)
└── README.md
```

There are only three moving parts:

1. **GitHub Action** (`collect-stats.yml`) — runs daily, calls the GitHub API, merges new data into the JSON files, and commits the result back to this repo.
2. **JSON data files** (`data/`) — the "database". The Action appends to these over time. They live in the repo itself, so git history is the backup.
3. **Dashboard** (`index.html`) — a single HTML file that loads the JSON files client-side using `fetch()` and renders charts and tables. Uses [Chart.js](https://www.chartjs.org/) from CDN. No framework, no bundler, no npm.

## How Data Collection Works

### Schedule

The GitHub Action runs on two triggers:

- **Daily cron** at 06:00 UTC (16:00 AEST)
- **Manual dispatch** — you can trigger it any time from the Actions tab or via CLI:
  ```bash
  gh workflow run collect-stats.yml --repo mrdavearms/repo-stats
  ```

### What It Collects

For each tracked repo, the Action calls these GitHub REST API endpoints:

| Data | Endpoint | Auth Required |
|------|----------|---------------|
| Page views (total + unique per day) | `GET /repos/{owner}/{repo}/traffic/views` | Yes |
| Clones (total + unique per day) | `GET /repos/{owner}/{repo}/traffic/clones` | Yes |
| Top referral sources (snapshot) | `GET /repos/{owner}/{repo}/traffic/popular/referrers` | Yes |
| Most viewed paths (snapshot) | `GET /repos/{owner}/{repo}/traffic/popular/paths` | Yes |
| Release assets + download counts | `GET /repos/{owner}/{repo}/releases` | No (but token used anyway) |

### Deduplication and Merging

GitHub returns the last 14 days of daily traffic data. The Action:

1. Reads the existing `data/traffic.json`
2. Merges new daily entries — if a date already exists, the newer value replaces it (GitHub may update the current day's count on subsequent API calls)
3. Never duplicates a date entry
4. Sorts all arrays by date ascending
5. Commits and pushes only if something changed

This means the JSON files grow over time, accumulating history that GitHub's API would otherwise discard after 14 days.

### Data File Formats

**`data/traffic.json`** — keyed by repo name:

```json
{
  "bulk-pdf-extractor-and-generator": {
    "views": [
      { "date": "2026-03-09", "total": 36, "unique": 2 }
    ],
    "clones": [
      { "date": "2026-03-09", "total": 43, "unique": 24 }
    ],
    "referrers": [
      {
        "date": "2026-03-18",
        "sources": [
          { "referrer": "google.com", "count": 8, "unique": 4 }
        ]
      }
    ],
    "popular_paths": [
      {
        "date": "2026-03-18",
        "paths": [
          { "path": "/mrdavearms/bulk-pdf-extractor-and-generator", "count": 10, "unique": 5 }
        ]
      }
    ]
  }
}
```

**`data/releases.json`** — keyed by repo name:

```json
{
  "bulk-pdf-extractor-and-generator": {
    "releases": [
      {
        "tag": "v2.6",
        "name": "Release Name",
        "published_at": "2026-01-15T00:00:00Z",
        "assets": [
          { "name": "tool.zip", "download_count": 47, "size_bytes": 1234567 }
        ]
      }
    ],
    "history": [
      { "date": "2026-03-18", "total_downloads": 47 }
    ]
  }
}
```

The `history` array records the cumulative download total each day. The dashboard can derive daily deltas from this.

## Dashboard Features

- **Dark mode** UI styled to match GitHub's aesthetic
- **Repo selector** — toggle between individual repos or view combined "All" data
- **Date range filter** — last 7 days, 30 days, 90 days, or all time
- **Summary cards** — total downloads, page views, unique visitors, total clones
- **Charts** (Chart.js):
  - Downloads over time (cumulative, one line per repo)
  - Page views over time (total views + unique visitors)
  - Clones over time (total clones + unique clones)
- **Tables**:
  - Referral sources (most recent snapshot)
  - Popular paths (most recent snapshot)
  - Release assets with download counts and file sizes
- **Mobile responsive**

## Setup (If Starting Fresh)

If you ever need to recreate this from scratch:

### 1. Personal Access Token

The GitHub traffic API requires authentication, even for your own public repos.

1. Go to [GitHub Settings > Developer settings > Personal access tokens > Tokens (classic)](https://github.com/settings/tokens)
2. Generate new token (classic)
3. Name: `repo-stats-tracker`
4. Scope: **`repo`** (full control of private repositories — this broad scope is required by GitHub even though we only read traffic on public repos; there is no narrower scope available)
5. Expiration: 1 year. **Set a calendar reminder to rotate it.**
6. Copy the token immediately (you can't see it again)

### 2. Repository Secret

1. Go to this repo's Settings > Secrets and variables > Actions
2. New repository secret
3. Name: `GH_STATS_TOKEN`
4. Value: paste the token

### 3. GitHub Pages

1. Go to this repo's Settings > Pages
2. Source: Deploy from a branch
3. Branch: `main`, folder: `/ (root)`
4. Save

Or via CLI:
```bash
gh api repos/mrdavearms/repo-stats/pages -X POST -f "build_type=workflow" -f "source[branch]=main" -f "source[path]=/"
```

### 4. Seed Initial Data

```bash
gh workflow run collect-stats.yml --repo mrdavearms/repo-stats
```

Wait ~30 seconds, then check the dashboard.

## Adding a New Repository to Track

1. Open `.github/workflows/collect-stats.yml`
2. Find the `REPOS` array near the top of the shell script:
   ```bash
   REPOS=("mrdavearms/bulk-pdf-extractor-and-generator" "mrdavearms/student-doc-redactor")
   ```
3. Add the new repo to the array:
   ```bash
   REPOS=("mrdavearms/bulk-pdf-extractor-and-generator" "mrdavearms/student-doc-redactor" "mrdavearms/new-repo")
   ```
4. Open `index.html` and update the `REPO_NAMES` array and the repo selector buttons to include the new repo
5. Commit, push, and trigger the Action manually

## Maintenance

### Token Rotation

The PAT expires after 1 year. When it expires:
- The Action will fail silently (API calls return errors, but the script uses `|| echo '...'` fallbacks so it won't crash — it just won't collect real data)
- Generate a new token following the steps above
- Update the `GH_STATS_TOKEN` secret in this repo's settings

### Checking Action Health

```bash
# View recent runs
gh run list --repo mrdavearms/repo-stats --limit 5

# View logs for a specific run
gh run view <run-id> --repo mrdavearms/repo-stats --log
```

### Manual Data Inspection

The JSON files are human-readable. You can inspect them directly:
```bash
cat data/traffic.json | jq '.["bulk-pdf-extractor-and-generator"].views | length'
```

## Limitations and Things to Know

### GitHub's 14-Day Traffic Window
GitHub only retains traffic data (views, clones) for the last 14 days. That's the entire reason this project exists — the daily Action captures data before it disappears. **Any days the Action doesn't run are lost forever.** If the Action is broken for 14+ days, you'll have a gap in your history.

### Clones Include Your Own Activity
GitHub's clone count includes **all** git operations against the repo — your own `git pull`, `git fetch`, `git clone`, as well as anyone else's. There is no way to filter out your own activity via the API. The **"unique" count** is more useful for gauging external interest — it's based on IP address, so your machine counts as 1 unique cloner per day regardless of how many pulls you do.

### Referrer and Path Data Is a Snapshot, Not a Time Series
The referrer and popular paths endpoints return the **top 10** for the trailing 14-day period, not per-day data. The Action stores each day's snapshot, but small referrers may appear and disappear from the top 10 on different days. This means the historical referrer data is approximate, not comprehensive.

### Download Counts Are Cumulative
GitHub's release API returns the **running total** of downloads per asset, not per-day downloads. The Action records this total each day in the `history` array. The dashboard shows the cumulative curve. Daily deltas could be derived but aren't currently displayed.

### Auto-Generated Source Archives Aren't Tracked
Every GitHub release automatically generates "Source code (zip)" and "Source code (tar.gz)" downloads. These are **not counted** by the API — only explicitly uploaded release assets are tracked.

### Unique Visitors vs Unique Referrers
The `unique` field on referrer data sometimes comes back as `null` from the GitHub API. The dashboard displays this as `—`. This is a known GitHub API inconsistency.

### No Distinction Between Signed-In and Anonymous Visitors
GitHub's traffic API does not reveal whether visitors are logged-in GitHub users or anonymous. This data is simply not available.

### GitHub Pages Caching
After the Action pushes new data, the GitHub Pages site may take 1-2 minutes to reflect the update due to CDN caching. Hard-refresh (Cmd+Shift+R) if the dashboard seems stale.

## Download Badges for Repo READMEs

You can add live download count badges to each tracked repo's README. These use shields.io's dynamic JSON badge feature, reading directly from the GitHub Pages data files.

**Bulk PDF Extractor** — add to that repo's README:
```markdown
![Downloads](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fmrdavearms.github.io%2Frepo-stats%2Fdata%2Freleases.json&query=%24%5B%22bulk-pdf-extractor-and-generator%22%5D.history%5B-1%3A%5D.total_downloads&label=downloads&color=blue)
```

**Student Doc Redactor** — add to that repo's README:
```markdown
![Downloads](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fmrdavearms.github.io%2Frepo-stats%2Fdata%2Freleases.json&query=%24%5B%22student-doc-redactor%22%5D.history%5B-1%3A%5D.total_downloads&label=downloads&color=blue)
```

These badges update automatically as the Action collects new data.

## Tech Stack

- **Data collection**: Bash, `curl`, `jq` (all available on GitHub Actions runners by default — no dependencies to install)
- **Dashboard**: Vanilla HTML/CSS/JS, [Chart.js v4](https://www.chartjs.org/) via CDN, [chartjs-adapter-date-fns](https://github.com/chartjs/chartjs-adapter-date-fns) via CDN
- **Hosting**: GitHub Pages (static, free)
- **Storage**: JSON files committed to the repo (git is the database)
