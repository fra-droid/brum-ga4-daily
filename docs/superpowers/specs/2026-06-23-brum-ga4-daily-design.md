# BRUM GA4 Daily Report — Design Spec
_Date: 2026-06-23_

## Overview

A live GA4 daily report dashboard for BRUM, hosted on GitHub Pages at `fra-droid/brum-ga4-daily`. Data is refreshed every morning by a scheduled cloud agent (via `mcp__scheduled-tasks`) that calls the GA4 MCP, computes averages, and commits `data.json` to the repo. The dashboard is a static HTML file that fetches `data.json` at load time.

---

## Architecture

```
┌─────────────────────────────┐
│  Scheduled Cloud Agent      │  runs daily ~08:00 CET
│  - calls ga4_run_report x4  │
│  - computes yesterday/avgs  │
│  - commits data.json to     │
│    GitHub via API            │
└────────────┬────────────────┘
             │ git push (GitHub API)
             ▼
┌─────────────────────────────┐
│  GitHub Repo: fra-droid/    │
│  brum-ga4-daily             │
│  ├── index.html             │
│  └── data.json              │
└────────────┬────────────────┘
             │ GitHub Pages
             ▼
┌─────────────────────────────┐
│  Browser                    │
│  fetch('data.json') → table │
└─────────────────────────────┘
```

---

## Files

### `data.json`
Updated by the scheduled agent every morning. Structure:

```json
{
  "date": "2026-06-22",
  "generated_at": "2026-06-23T08:00:00Z",
  "metrics": [
    {
      "name": "page_view",
      "yesterday": 2703,
      "avg7": 2857.9,
      "avg15": 2958.3,
      "avg28": 3011.7
    },
    {
      "name": "session_start",
      "yesterday": 2027,
      "avg7": 2161.0,
      "avg15": 2234.7,
      "avg28": 2324.9
    },
    {
      "name": "user_engagement",
      "yesterday": 697,
      "avg7": 685.9,
      "avg15": 719.1,
      "avg28": 680.9
    },
    {
      "name": "initiate_checkout",
      "yesterday": 80,
      "avg7": 93.7,
      "avg15": 100.2,
      "avg28": 88.5
    }
  ]
}
```

### `index.html`
Single-file static dashboard. No build step, no dependencies beyond CDN.

- Fetches `data.json` on load and on "Ricarica" click
- Renders a table matching the screenshot exactly
- Computes delta% = `(yesterday - avg7) / avg7 * 100` for the arrow column
- Arrow is green (▲) if delta > 0, red (▼) if delta < 0
- Colors and fonts match BRUM brand: viola `#911ef7`, giallo `#ffee1f`, font Grotesk

---

## Dashboard UI

Matches screenshot exactly:

| Column    | Content |
|-----------|---------|
| ga4       | metric name (bold) |
| yesterday | raw value + colored arrow + delta% vs avg7 |
| avg7gg    | 7-day average |
| avg15     | 15-day average |
| avg28     | 28-day average |

Header: BRUM logo (purple pill) + "Report giornaliero GA4" title + date ("dato di ieri: DD mese YYYY") top-right.

Footer note (small gray text): data source, window info, reload hint.

---

## Scheduled Agent

- **Schedule:** daily at 08:00 CET (07:00 UTC)
- **GA4 property:** `423910028`
- **Queries:** 4 calls to `ga4_run_report` with eventName dimension:
  - yesterday (single day)
  - last 7 days → divide total by 7
  - last 15 days → divide total by 15
  - last 28 days → divide total by 28
- **Commit:** uses GitHub API (HTTPS + PAT stored in agent config) to PUT `data.json` on `main` branch
- **GitHub PAT needed:** `contents:write` scope on `fra-droid/brum-ga4-daily`

---

## Deployment

1. Create repo `fra-droid/brum-ga4-daily` (public)
2. Enable GitHub Pages on `main` branch root
3. Push `index.html` + initial `data.json`
4. Create GitHub PAT with `contents:write`
5. Create scheduled cloud task with the PAT and schedule

---

## Out of Scope

- Authentication / access control (public dashboard)
- Historical charts
- Other metrics beyond the 4 shown
