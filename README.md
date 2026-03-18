# GitHub Repo Stats

A lightweight dashboard that tracks download counts, page views, unique visitors, referral sources, and clone data for public GitHub repositories.

## How It Works

- **GitHub Action** runs daily at 06:00 UTC (16:00 AEST), collecting traffic and release data via the GitHub API
- Data is committed to `data/traffic.json` and `data/releases.json`, preserving history beyond GitHub's 14-day retention window
- **Dashboard** (`index.html`) loads the JSON files client-side — no build step, no backend

## Tracked Repositories

- [bulk-pdf-extractor-and-generator](https://github.com/mrdavearms/bulk-pdf-extractor-and-generator)
- [student-doc-redactor](https://github.com/mrdavearms/student-doc-redactor)

## Setup

1. Create a [Personal Access Token (classic)](https://github.com/settings/tokens) with `repo` scope
2. Add it as a repository secret named `GH_STATS_TOKEN` (Settings > Secrets and variables > Actions)
3. Enable GitHub Pages (Settings > Pages > Source: main branch, root)
4. Trigger the Action manually once to seed initial data

## Dashboard

Available at: https://mrdavearms.github.io/repo-stats/
