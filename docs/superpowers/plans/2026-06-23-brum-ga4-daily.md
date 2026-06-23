# BRUM GA4 Daily Report Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a live GA4 daily report dashboard for BRUM hosted on GitHub Pages, auto-updated every morning by a scheduled cloud agent.

**Architecture:** A static `index.html` fetches `data.json` (GA4 metrics) at load time and renders a branded table. A cloud scheduled task runs daily at 08:00 CET, queries GA4 via MCP, and commits `data.json` to the repo using the GitHub API with a PAT.

**Tech Stack:** Vanilla HTML/CSS/JS (no build step), GitHub Pages, `mcp__scheduled-tasks` cloud scheduler, `mcp__ddf5d80c-af8f-4f70-9046-24b285546d16__ga4_run_report` MCP tool, GitHub REST API for file commits.

## Global Constraints

- GA4 property ID: `423910028`
- GitHub repo: `fra-droid/brum-ga4-daily` (public)
- GA4 metrics: `page_view`, `session_start`, `user_engagement`, `initiate_checkout`
- Branch: `main`
- GitHub Pages root: `/` (main branch root)
- BRUM brand colors: viola `#911ef7`, giallo `#ffee1f`, ink `#0d0d10`, offwhite `#faf9ff`
- No external JS dependencies (no React, no CDN)
- `data.json` must be valid JSON matching the schema defined in Task 2

---

### Task 1: Create GitHub repo, enable Pages, push initial skeleton

**Files:**
- Create: `index.html` (placeholder — full implementation in Task 2)
- Create: `data.json` (placeholder — real data in Task 3)
- Create: `.nojekyll` (prevents GitHub Pages from running Jekyll)

**Interfaces:**
- Produces: live GitHub Pages URL `https://fra-droid.github.io/brum-ga4-daily/`

- [ ] **Step 1: Create the repo via gh CLI**

```bash
cd /Users/francescocucciniello/code/brum-ga4-daily
git init
gh repo create fra-droid/brum-ga4-daily --public --description "BRUM GA4 daily report dashboard" --source=. --remote=origin
```

- [ ] **Step 2: Create `.nojekyll` and placeholder files**

Create `.nojekyll` (empty file):
```bash
touch .nojekyll
```

Create placeholder `data.json`:
```json
{
  "date": "2026-06-22",
  "generated_at": "2026-06-23T08:00:00Z",
  "metrics": [
    { "name": "page_view",         "yesterday": 0, "avg7": 0, "avg15": 0, "avg28": 0 },
    { "name": "session_start",     "yesterday": 0, "avg7": 0, "avg15": 0, "avg28": 0 },
    { "name": "user_engagement",   "yesterday": 0, "avg7": 0, "avg15": 0, "avg28": 0 },
    { "name": "initiate_checkout", "yesterday": 0, "avg7": 0, "avg15": 0, "avg28": 0 }
  ]
}
```

Create placeholder `index.html`:
```html
<!doctype html><html><head><title>BRUM GA4</title></head><body><p>Loading...</p></body></html>
```

- [ ] **Step 3: Commit and push**

```bash
git add .nojekyll data.json index.html docs/
git commit -m "feat: initial repo scaffold"
git push -u origin main
```

- [ ] **Step 4: Enable GitHub Pages**

```bash
gh api repos/fra-droid/brum-ga4-daily/pages \
  --method POST \
  -f source.branch=main \
  -f source.path=/
```

Expected output: JSON with `"html_url": "https://fra-droid.github.io/brum-ga4-daily/"` (may take 1-2 min to go live).

- [ ] **Step 5: Verify Pages is live**

```bash
sleep 30 && curl -s -o /dev/null -w "%{http_code}" https://fra-droid.github.io/brum-ga4-daily/
```

Expected: `200`

---

### Task 2: Build `index.html` dashboard

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `data.json` — fetched via `fetch('./data.json')`
- `data.json` schema:
  ```ts
  {
    date: string,           // "YYYY-MM-DD", date of "yesterday"
    generated_at: string,   // ISO timestamp
    metrics: Array<{
      name: string,         // "page_view" | "session_start" | "user_engagement" | "initiate_checkout"
      yesterday: number,
      avg7: number,
      avg15: number,
      avg28: number
    }>
  }
  ```

- [ ] **Step 1: Write `index.html`**

Replace `index.html` with the full implementation:

```html
<!doctype html>
<html lang="it">
<head>
<meta charset="utf-8">
<title>BRUM · Report GA4 Giornaliero</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --viola:     #911ef7;
  --giallo:    #ffee1f;
  --ink:       #0d0d10;
  --ink3:      #5b5b66;
  --ink4:      #9a9aa8;
  --offwhite:  #faf9ff;
  --white:     #ffffff;
  --line:      #e7e5f1;
  --lineSoft:  #f1eff9;
  --success:   #1fb87a;
  --danger:    #ff3b3b;
  --header-bg: #f0eefb;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif;
  background: var(--offwhite);
  color: var(--ink);
  min-height: 100vh;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding: 32px 16px;
}

.card {
  background: var(--white);
  border-radius: 16px;
  box-shadow: 0 2px 12px rgba(145,30,247,.08);
  overflow: hidden;
  width: 100%;
  max-width: 720px;
}

/* ── Header ── */
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 24px 16px;
  border-bottom: 1px solid var(--line);
}

.header-left { display: flex; align-items: center; gap: 14px; }

.brum-badge {
  background: var(--viola);
  color: var(--giallo);
  font-weight: 800;
  font-size: 13px;
  letter-spacing: 1px;
  padding: 6px 12px;
  border-radius: 8px;
  text-transform: uppercase;
}

.header-title {
  font-size: 18px;
  font-weight: 700;
  color: var(--ink);
}

.header-date {
  font-size: 13px;
  color: var(--ink4);
}

/* ── Table ── */
table {
  width: 100%;
  border-collapse: collapse;
}

thead tr {
  background: var(--header-bg);
}

thead th {
  font-size: 12px;
  font-weight: 700;
  color: var(--viola);
  text-align: right;
  padding: 10px 20px;
  letter-spacing: .3px;
}

thead th:first-child { text-align: left; }

tbody tr {
  border-bottom: 1px solid var(--lineSoft);
}

tbody tr:last-child { border-bottom: none; }

tbody td {
  padding: 14px 20px;
  font-size: 15px;
  text-align: right;
  color: var(--ink);
}

tbody td:first-child {
  text-align: left;
  font-weight: 700;
  font-size: 14px;
}

.delta {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 13px;
  font-weight: 600;
  margin-left: 6px;
}

.delta.up   { color: var(--success); }
.delta.down { color: var(--danger); }
.delta.flat { color: var(--ink4); }

.avg-val { color: var(--ink3); font-size: 14px; }

/* ── Footer ── */
.footer {
  padding: 12px 24px 16px;
  border-top: 1px solid var(--lineSoft);
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.footer-note {
  font-size: 11px;
  color: var(--ink4);
  line-height: 1.5;
}

.reload-btn {
  flex-shrink: 0;
  background: var(--viola);
  color: var(--white);
  border: none;
  border-radius: 8px;
  padding: 7px 16px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: opacity .15s;
}

.reload-btn:hover { opacity: .85; }
.reload-btn:disabled { opacity: .5; cursor: default; }

.error-msg {
  padding: 32px 24px;
  text-align: center;
  color: var(--danger);
  font-size: 14px;
}
</style>
</head>
<body>
<div class="card" id="card">
  <div class="header">
    <div class="header-left">
      <span class="brum-badge">BRUM</span>
      <span class="header-title">Report giornaliero GA4</span>
    </div>
    <span class="header-date" id="header-date">caricamento…</span>
  </div>
  <div id="body-slot"></div>
  <div class="footer">
    <span class="footer-note" id="footer-note">
      Fonte: GA4 property 423910028 · finestra: ultimi 28 giorni ·
      la freccia accanto a "yesterday" confronta il dato di ieri con la media a 7 giorni ·
      usa Ricarica per aggiornare
    </span>
    <button class="reload-btn" id="reload-btn" onclick="loadData()">Ricarica</button>
  </div>
</div>

<script>
const METRIC_LABELS = {
  page_view:         'page_view',
  session_start:     'session_start',
  user_engagement:   'user_engagement',
  initiate_checkout: 'initiate_checkout',
};

function fmt(n) {
  if (n == null || isNaN(n)) return '—';
  return Number(n).toLocaleString('it-IT', { maximumFractionDigits: 1 });
}

function delta(yesterday, avg7) {
  if (!avg7) return { pct: null, cls: 'flat', arrow: '·' };
  const pct = ((yesterday - avg7) / avg7) * 100;
  if (pct > 0)  return { pct, cls: 'up',   arrow: '▲' };
  if (pct < 0)  return { pct, cls: 'down', arrow: '▼' };
  return { pct: 0, cls: 'flat', arrow: '·' };
}

function formatDate(dateStr) {
  if (!dateStr) return '';
  const [y, m, d] = dateStr.split('-').map(Number);
  const months = ['gennaio','febbraio','marzo','aprile','maggio','giugno',
                  'luglio','agosto','settembre','ottobre','novembre','dicembre'];
  return `dato di ieri: ${d} ${months[m-1]} ${y}`;
}

function renderTable(data) {
  const rows = data.metrics.map(m => {
    const d = delta(m.yesterday, m.avg7);
    const pctStr = d.pct != null ? (d.pct > 0 ? '+' : '') + d.pct.toFixed(0) + '%' : '';
    return `
      <tr>
        <td>${METRIC_LABELS[m.name] || m.name}</td>
        <td>
          ${fmt(m.yesterday)}
          ${d.pct != null ? `<span class="delta ${d.cls}">${d.arrow} ${pctStr}</span>` : ''}
        </td>
        <td class="avg-val">${fmt(m.avg7)}</td>
        <td class="avg-val">${fmt(m.avg15)}</td>
        <td class="avg-val">${fmt(m.avg28)}</td>
      </tr>`;
  }).join('');

  return `
    <table>
      <thead>
        <tr>
          <th>ga4</th>
          <th>yesterday</th>
          <th>avg7gg</th>
          <th>avg15</th>
          <th>avg28</th>
        </tr>
      </thead>
      <tbody>${rows}</tbody>
    </table>`;
}

async function loadData() {
  const btn = document.getElementById('reload-btn');
  const slot = document.getElementById('body-slot');
  btn.disabled = true;
  btn.textContent = '…';

  try {
    const res = await fetch('./data.json?t=' + Date.now());
    if (!res.ok) throw new Error('HTTP ' + res.status);
    const data = await res.json();

    document.getElementById('header-date').textContent = formatDate(data.date);
    slot.innerHTML = renderTable(data);
  } catch (e) {
    slot.innerHTML = `<div class="error-msg">Errore nel caricamento dei dati: ${e.message}</div>`;
  } finally {
    btn.disabled = false;
    btn.textContent = 'Ricarica';
  }
}

loadData();
</script>
</body>
</html>
```

- [ ] **Step 2: Verify locally**

Open `index.html` in browser (double-click or `open index.html`). Confirm:
- BRUM badge (purple) + title visible
- Table renders 4 rows with placeholder zeros
- "Ricarica" button works without console errors

- [ ] **Step 3: Commit and push**

```bash
git add index.html
git commit -m "feat: add GA4 daily report dashboard"
git push
```

- [ ] **Step 4: Verify on GitHub Pages**

Open `https://fra-droid.github.io/brum-ga4-daily/` — confirm table loads correctly.

---

### Task 3: Fetch real GA4 data and populate `data.json`

**Files:**
- Modify: `data.json`

**Interfaces:**
- Consumes: `mcp__ddf5d80c-af8f-4f70-9046-24b285546d16__ga4_run_report`
- Produces: `data.json` with real values matching the schema from Task 2

**Note:** This task is executed manually in a Claude Code session using the GA4 MCP tool. The scheduled agent (Task 4) will repeat this automatically every morning.

- [ ] **Step 1: Call GA4 MCP for yesterday's event counts**

Call `ga4_run_report` with:
- `property_id`: `"423910028"`
- `dimensions`: `["eventName"]`
- `metrics`: `["eventCount"]`
- `start_date`: `"yesterday"`
- `end_date`: `"yesterday"`

Extract values for: `page_view`, `session_start`, `user_engagement`, `initiate_checkout`.

- [ ] **Step 2: Call GA4 MCP for last 7 days**

Same call with `start_date: "7daysAgo"`, `end_date: "yesterday"`. Divide each eventCount by 7.

- [ ] **Step 3: Call GA4 MCP for last 15 days**

Same call with `start_date: "15daysAgo"`, `end_date: "yesterday"`. Divide each eventCount by 15.

- [ ] **Step 4: Call GA4 MCP for last 28 days**

Same call with `start_date: "28daysAgo"`, `end_date: "yesterday"`. Divide each eventCount by 28.

- [ ] **Step 5: Write `data.json` with real values**

Using the values from steps 1-4, write `data.json` in the format:
```json
{
  "date": "<yesterday as YYYY-MM-DD>",
  "generated_at": "<now as ISO timestamp>",
  "metrics": [
    { "name": "page_view",         "yesterday": <val>, "avg7": <val>, "avg15": <val>, "avg28": <val> },
    { "name": "session_start",     "yesterday": <val>, "avg7": <val>, "avg15": <val>, "avg28": <val> },
    { "name": "user_engagement",   "yesterday": <val>, "avg7": <val>, "avg15": <val>, "avg28": <val> },
    { "name": "initiate_checkout", "yesterday": <val>, "avg7": <val>, "avg15": <val>, "avg28": <val> }
  ]
}
```

Round avg values to 1 decimal place.

- [ ] **Step 6: Commit and push**

```bash
git add data.json
git commit -m "data: populate with real GA4 data"
git push
```

- [ ] **Step 7: Verify dashboard shows real data**

Open `https://fra-droid.github.io/brum-ga4-daily/` (or reload). Confirm:
- Numbers match raw GA4 data
- Delta arrows are colored correctly (green ▲ if yesterday > avg7, red ▼ if lower)
- Date in header shows yesterday's date

---

### Task 4: Create GitHub PAT and set up scheduled cloud agent

**Files:** none (configuration only)

**Interfaces:**
- Consumes: GitHub PAT with `contents:write` on `fra-droid/brum-ga4-daily`
- Produces: cloud scheduled task that runs daily at 07:00 UTC (08:00 CET) and commits fresh `data.json`

- [ ] **Step 1: Create GitHub PAT**

Go to `https://github.com/settings/tokens/new` and create a **fine-grained PAT**:
- Name: `brum-ga4-daily-writer`
- Expiration: 1 year
- Repository access: `fra-droid/brum-ga4-daily` only
- Permissions → Contents: **Read and write**

Copy the token (shown only once).

- [ ] **Step 2: Create the scheduled cloud task**

Use `mcp__scheduled-tasks__create_scheduled_task` with a prompt that instructs the agent to:

1. Call `ga4_run_report` 4 times (yesterday, 7daysAgo, 15daysAgo, 28daysAgo) with `property_id=423910028`, `dimensions=["eventName"]`, `metrics=["eventCount"]`
2. Extract counts for `page_view`, `session_start`, `user_engagement`, `initiate_checkout` from each response
3. Compute averages (divide totals by 7, 15, 28 respectively), rounded to 1 decimal
4. Build `data.json` per the schema
5. Fetch the current `data.json` SHA from GitHub API: `GET https://api.github.com/repos/fra-droid/brum-ga4-daily/contents/data.json` with `Authorization: token <PAT>`
6. PUT the new content to GitHub API: `PUT https://api.github.com/repos/fra-droid/brum-ga4-daily/contents/data.json` with body `{ message, content (base64), sha }`

Schedule: `0 7 * * *` (07:00 UTC daily)

- [ ] **Step 3: Verify agent runs correctly**

Trigger the scheduled task manually once. Confirm:
- `data.json` on GitHub shows updated `generated_at` timestamp
- Dashboard at `https://fra-droid.github.io/brum-ga4-daily/` shows fresh data after reload

---

## Self-Review Notes

- Spec coverage: ✓ all 4 GA4 metrics, ✓ yesterday/avg7/avg15/avg28, ✓ delta arrows, ✓ BRUM branding, ✓ GitHub Pages, ✓ scheduled agent, ✓ PAT setup
- No placeholders in code blocks
- Type consistency: `data.json` schema defined in Task 2 Interfaces, consumed identically in Task 3 Step 5 and Task 4 Step 2
- `fmt()` function handles null/NaN gracefully
- `?t=` cache-bust on fetch ensures Ricarica always gets fresh data from Pages CDN
