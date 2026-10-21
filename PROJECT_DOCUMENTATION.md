# IntoAEC Marketing Dashboard — Complete Project Documentation

**Package:** `intoacec-dashboard` v1.0.0  
**Description:** IntoAEC Social Media Analytics Dashboard  
**Last documented:** September 2026

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Directory Structure](#3-directory-structure)
4. [System Architecture](#4-system-architecture)
5. [Getting Started](#5-getting-started)
6. [Environment Configuration](#6-environment-configuration)
7. [Application Workflow (End-to-End)](#7-application-workflow-end-to-end)
8. [Code Workflow by Module](#8-code-workflow-by-module)
9. [API Reference](#9-api-reference)
10. [Frontend Structure & Client Flow](#10-frontend-structure--client-flow)
11. [External Integrations](#11-external-integrations)
12. [Caching & Storage](#12-caching--storage)
13. [Utility & Diagnostic Scripts](#13-utility--diagnostic-scripts)
14. [Data Payload Shapes](#14-data-payload-shapes)
15. [Known Gaps & Notes](#15-known-gaps--notes)

---

## 1. Project Overview

The IntoAEC Marketing Dashboard is a unified internal analytics application that pulls live performance data from multiple platforms and displays it in one browser UI:

| Platform | What is shown |
|----------|----------------|
| **Google Analytics 4** | Users, sessions, views, bounce rate, traffic channels, top pages, countries, demographic filters |
| **Facebook** | Page fans/followers, post engagement, insights |
| **Instagram** | Followers, media, engagement, audience demographics |
| **YouTube** | Subscribers, views, videos, analytics, top videos |
| **Google Search Console** | Clicks, impressions, CTR, position, queries, pages, countries, devices, URL inspection, Core Web Vitals |
| **PageSpeed Insights** | Lighthouse scores + field/lab web vitals for any URL |
| **Weekly Comparison** | Client-side week-over-week deltas from already-loaded data |

There is **no database**. Secrets stay on the server. The browser only receives aggregated JSON.

---

## 2. Tech Stack

| Layer | Technology |
|-------|------------|
| Runtime | Node.js (ES modules, `"type": "module"`) |
| Backend | Express 4, CORS, dotenv, axios, googleapis |
| Frontend | Single-file vanilla HTML/CSS/JS (`index.html` ~6300 lines) |
| Charts | Chart.js 4.4 (CDN) |
| Auth | Meta access token; Google OAuth2 refresh token; API keys for CrUX / PageSpeed |
| Storage | In-memory caches + `gsc_url_inspections.json` for GSC inspection records |
| Unused dep | `exceljs` (listed in package.json, not imported) |

**Scripts**

```bash
npm start          # node server.js  — start dashboard
npm run fetch      # node analysis.js — CLI analytics run
```

---

## 3. Directory Structure

The repo is intentionally flat (no `src/`, `routes/`, or `public/` split):

```
marketing_dashboard/
├── server.js                 # Express app — all HTTP routes
├── analysis.js               # FB / IG / GA4 / YouTube pipeline
├── gsc.js                    # Search Console, indexing, CrUX
├── pagespeed.js              # PageSpeed Insights analysis
├── index.html                # Full UI + all client JavaScript
├── benchmarks_config.js      # AEC SaaS benchmark profiles
├── get-token.js              # One-time Google OAuth refresh-token helper
├── ga4-diagnostic.js         # CLI GA4 connectivity check
├── youtube-diagnostic.js     # CLI YouTube connectivity check
├── package.json
├── .env                      # Secrets (gitignored)
├── gsc_url_inspections.json  # Persistent URL inspection store (gitignored)
├── PROJECT_DOCUMENTATION.md  # This file
└── *.md                      # Feature specs / prompts / older docs
```

---

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Browser                                                     │
│  index.html + Chart.js + benchmarks_config.js                │
│  fetch('/api/...')                                           │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTP
┌────────────────────────────▼────────────────────────────────┐
│  Express (server.js)                                         │
│  Static files + JSON APIs                                    │
└───────┬──────────────────┬──────────────────┬───────────────┘
        │                  │                  │
        ▼                  ▼                  ▼
  analysis.js           gsc.js           pagespeed.js
  Meta Graph            Search Console   PageSpeed Insights
  GA4 Data API          URL Inspection
  YouTube APIs          Chrome UX (CrUX)
        │                  │                  │
        └──────────┬───────┴──────────────────┘
                   ▼
              .env credentials
```

**Design principles**

1. All third-party credentials live only in `.env` on the server.
2. Each major data domain has its own module (`analysis.js`, `gsc.js`, `pagespeed.js`).
3. `server.js` is a thin HTTP layer: validate input → call module → return JSON.
4. Failures in one platform become `warnings[]`; other platforms still return data.

---

## 5. Getting Started

### Prerequisites
- Node.js 18+ recommended
- Meta (Facebook) Page + Instagram Business account with a long-lived access token
- Google Cloud project with OAuth client + APIs enabled:
  - Google Analytics Data API
  - YouTube Data API v3
  - YouTube Analytics API
  - Search Console API
  - Chrome UX Report API (API key)
  - PageSpeed Insights API (API key)

### Install & run

```bash
npm install
# Configure .env (see section 6)
node get-token.js    # first time: obtain GOOGLE_REFRESH_TOKEN with all scopes
npm start            # opens http://localhost:3000 (or next free port up to 3010)
```

Open the printed URL. Click **Refresh Live Data** to pull social + GA4 + YouTube. Search Console and PageSpeed load from their own tabs/actions.

---

## 6. Environment Configuration

| Variable | Purpose |
|----------|---------|
| `PORT` | Listen port (default `3000`) |
| `PAGE_ID` | Facebook Page ID |
| `ACCESS_TOKEN` | Meta Graph user/page token |
| `IG_USER_ID` | Instagram business user ID |
| `YOUTUBE_CHANNEL_HANDLE` | Channel handle (e.g. `@IntoAEC`) |
| `YOUTUBE_API_KEY` | Present in env; primary YouTube path uses OAuth, not this key |
| `GA_PROPERTY_ID` | GA4 property ID |
| `GA_MEASUREMENT_ID` | Present in env; not used by server fetchers |
| `GOOGLE_CLIENT_ID` | OAuth client ID |
| `GOOGLE_CLIENT_SECRET` | OAuth client secret |
| `GOOGLE_REFRESH_TOKEN` | Offline refresh token |
| `GSC_SITE_URL` | Default Search Console property (`sc-domain:example.com` or URL-prefix) |
| `CRUX_API_KEY` | Chrome UX Report |
| `PAGESPEED_API_KEY` | PageSpeed Insights (falls back to other keys if unset) |
| `GSC_INDEXING_CACHE_TTL_MINUTES` | Inspection freshness (default ~30–60) |
| `GSC_INDEXING_CONCURRENCY` | Parallel URL inspections (default `5`) |
| `GSC_INDEXING_MAX_INSPECTIONS_PER_SCAN` | Cap per background scan (default `2000`) |

**OAuth scopes** (issued by `get-token.js`):

- `analytics.readonly`
- `yt-analytics.readonly`
- `youtube.readonly`
- `webmasters.readonly`

---

## 7. Application Workflow (End-to-End)

### 7.1 Server boot

```
npm start
  → node server.js
  → loads analysis.js / gsc.js / pagespeed.js (each loads .env via dotenv)
  → express.static(__dirname)
  → startServer(PORT || 3000)
  → on EADDRINUSE, try next port until 3010
```

### 7.2 First browser load

```
User opens http://localhost:PORT/
  → GET / → index.html
  → Browser loads Chart.js CDN + benchmarks_config.js
  → Inline script runs loadCachedData()
  → GET /api/data → runAnalytics() → JSON payload
  → applyDataset(payload) → renderAll()
  → Overview / FB / IG / GA / YT tabs populate
  → GSC may auto-load when that tab is active
```

### 7.3 Refresh Live Data

```
User clicks #refreshBtn
  → POST /api/refresh
  → runAnalytics() (same pipeline as /api/data)
  → applyDataset() → renderAll()
  → toast shows success / warnings
```

### 7.4 Cross-platform analytics pipeline (`runAnalytics`)

```
runAnalytics()
  │
  ├─ parallel: Facebook (if PAGE_ID + ACCESS_TOKEN)
  │     getPageToken() → page info + insights → all posts → processFbPosts()
  │
  ├─ parallel: Instagram (if IG_USER_ID + ACCESS_TOKEN)
  │     account info + insights + audience + all media → processIgMedia()
  │
  ├─ sequential block: GA4 (if property + OAuth)
  │     parallel: metrics (90d) + traffic sources (90d) + top pages (365d)
  │
  ├─ LinkedIn: intentionally skipped (“LinkedIn data removed”)
  │
  ├─ YouTube (if handle + OAuth)
  │     resolve channel → public stats → uploads playlist → videos
  │     parallel: YT Analytics + top videos (90d)
  │
  └─ shapeDashboardPayload({ fb, ig, ga, yt, warnings })
        → { fetchedAt, warnings, summary, GA4_*, FB_POSTS, IG_POSTS, YT_VIDEOS, ... }
```

Platform errors are caught individually and pushed into `warnings`; the rest of the payload still returns.

### 7.5 Search Console workflow (independent of `/api/data`)

```
User opens Search Console tab / changes date or site
  → GET /api/gsc/sites (optional site picker)
  → GET /api/gsc?siteUrl&startDate&endDate[&refresh=true]
  → server checks 5-minute in-memory cache
  → miss: runGSC() → Search Analytics queries (daily, queries, pages, countries, devices)
  → normalize { success, data } → cache → UI renderGSC() / pages / countries / queries

URL Inspection sub-tab:
  → User enters URL → triggerUrlInspection()
  → POST /api/gsc/url-inspection { url, siteUrl }
  → validate URL + property membership
  → inspectSingleUrlWithCache() → URL Inspection API + memory/file cache
  → renderUrlInspectionResult()

Core Web Vitals sub-tab:
  → GET /api/core-web-vitals?url&device
  → CrUX records:queryRecord (1h cache)
  → renderCoreWebVitals()
```

### 7.6 Full-site indexing engine (backend; UI largely unused)

```
GET /api/gsc/indexing?siteUrl&page&pageSize&status&search&refresh
  → getIndexingData()
  → sync sitemaps (parse XML, recurse sitemap indexes)
  → merge URLs into gsc_url_inspections.json
  → optionally startIndexingScan() in background
  → concurrent inspectUrl() with GSC_INDEXING_CONCURRENCY
  → paginated JSON response

POST /api/gsc/indexing/inspect { siteUrl, url }
  → inspectSingleUrl() — demand inspection into the JSON store
```

Current `index.html` focuses on **single URL inspection** (`/api/gsc/url-inspection`), not the full indexing table UI.

### 7.7 Page Performance workflow

```
User enters URL + Mobile/Desktop on Page Performance tab
  → POST /api/pagespeed/analyze { url, strategy, refresh }
  → analyzePageSpeed() → PageSpeed Insights v5
  → 1-hour in-memory cache
  → renderPageSpeedResults() (+ overview charts when available)
```

### 7.8 GA4 filters & countries

```
Filter UI:
  → GET /api/analytics/dimension-values?dimension=...
  → user selects values → POST /api/analytics/filter { filters, startDate, endDate }
  → getFilteredGA4Data() → filtered metrics/charts/pages/countries
  → UI updates GA tab only

Countries:
  → GET /api/analytics/countries → runGA4Countries() → top countries by views
```

### 7.9 Weekly Comparison

Purely client-side: `renderWeeklyComparison()` splits already-loaded daily series into this week vs last week and renders deltas/charts. **No dedicated API.**

---

## 8. Code Workflow by Module

### 8.1 `server.js` — HTTP gateway

| Responsibility | Detail |
|----------------|--------|
| Static hosting | Entire project root via `express.static` |
| Body parsing | `express.json()` |
| CORS | Enabled for local/dev use |
| Caching | In-memory `gscCache` Map, 5-minute TTL for `/api/gsc` |
| Port failover | 3000 → 3010 on `EADDRINUSE` |

Imports:

- `runAnalytics`, `runGA4Countries`, `getGA4DimensionValues`, `getFilteredGA4Data` from `analysis.js`
- GSC/CrUX helpers from `gsc.js`
- `analyzePageSpeed` from `pagespeed.js`

### 8.2 `analysis.js` — Social + GA4 + YouTube

| Function group | Role |
|----------------|------|
| `initGoogle()` | Build OAuth2 client + Analytics Data client |
| `getPageToken()` / `fetchFb*` / `processFbPosts` | Facebook Graph v20 |
| `fetchIg*` / `processIgMedia` | Instagram Graph |
| `getGA4Metrics` / `Traffic` / `TopPages` / `TopCountries` | GA4 reports |
| `getGA4DimensionValues` / `getFilteredGA4Data` | Filter support |
| `getYouTube*` | Channel resolve, Data API, Analytics API |
| `parallelMap` | Concurrency limiter for post/media fan-out |
| `shapeDashboardPayload` | Normalize into UI-friendly keys |
| `runAnalytics` | Orchestrator (exported) |

**LinkedIn helpers** (`getLinkedIn*`) still exist and map into `LI_*` payload fields, but `runAnalytics()` does not call them.

### 8.3 `gsc.js` — Search Console + indexing + CrUX

| Function group | Role |
|----------------|------|
| `buildOAuthClient` / `queryGSC` | Auth + Search Analytics wrapper |
| `fetchDailyPerformance`, `TopQueries`, `TopPages`, `Device`, `Country` | Dimensioned queries |
| `runGSC` | Aggregate performance report |
| `listSites` | List SC properties for the OAuth account |
| `inspectUrl` / `runIndexingStatus` | URL Inspection API |
| `initDb` / `readDb` / `writeDb` | `gsc_url_inspections.json` persistence |
| `parseSitemapXml` / `getSitemapUrlsForProperty` | Sitemap discovery |
| `getIndexingData` / `startIndexingScan` / `runScanInBackground` | Full-site scan engine |
| `inspectSingleUrl` / `inspectSingleUrlWithCache` | Manual inspection |
| `isUrlInProperty` / `mapUrlInspectionError` | Validation & error mapping |
| `fetchCoreWebVitals` | CrUX API |

### 8.4 `pagespeed.js` — PageSpeed Insights

| Function | Role |
|----------|------|
| `analyzePageSpeed(url, strategy, forceRefresh)` | Call PSI v5, score categories, format LCP/INP/CLS, cache 1h |
| Status helpers | Map metric values to good / needs-improvement / poor |

### 8.5 `index.html` — UI + client orchestration

Major client functions:

| Function | Purpose |
|----------|---------|
| `loadCachedData` | Initial `GET /api/data` |
| `applyDataset` | Store payload in globals / state |
| `renderAll` | Re-render overview + platform tabs |
| `fetchGSC` / `loadGscSites` / `renderGSC` | GSC performance UI |
| `renderGscPages` / `fetchGscPageIndexing` | Pages panel |
| `renderGscQueries` | Queries panel with client pagination |
| `triggerUrlInspection` / `renderUrlInspectionResult` | Single URL inspection |
| `fetchCoreWebVitals` / `renderCoreWebVitals` | CrUX panel |
| `fetchPageSpeedAnalysis` / `renderPageSpeedResults` | PSI tab |
| `applyGAFiltersToServer` / filter UI helpers | GA4 demographic filters |
| `fetchGA4Countries` | Country report |
| `renderWeeklyComparison` | Week-over-week |
| `renderLinkedIn` | Legacy; data not fetched |

Nav tabs (`data-tab`): `overview`, `ga`, `fb`, `ig`, `yt`, `gsc`, `page-performance`, `weekly`.

GSC sub-panels: `performance`, `pages`, `countries`, `queries`, `url-inspection`, `vitals`.

### 8.6 `benchmarks_config.js`

Defines `BENCHMARK_PROFILES` (AEC SaaS, Construction SaaS, Architecture SaaS, Engineering SaaS, General SaaS) and a small `BenchmarkService` used by the GA/overview UI to compare metrics against target ranges.

---

## 9. API Reference

### Main analytics
| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/` | — | Serve `index.html` |
| `GET` | `/benchmarks_config.js` | — | Serve benchmarks |
| `GET` | `/api/data` | `runAnalytics()` | Fresh social + GA4 + YouTube payload |
| `POST` | `/api/refresh` | `runAnalytics()` | Same as above (explicit refresh) |

### Google Analytics

| Method | Path | Body / Query | Description |
|--------|------|--------------|-------------|
| `GET` | `/api/analytics/countries` | — | Top countries by page views |
| `GET` | `/api/analytics/dimension-values` | `dimension`, `startDate`, `endDate` | Distinct dimension values |
| `POST` | `/api/analytics/filter` | `{ filters, startDate, endDate }` | Filtered GA4 dashboard slice |

### Search Console

| Method | Path | Body / Query | Description |
|--------|------|--------------|-------------|
| `GET` | `/api/gsc` | `siteUrl`, `startDate`, `endDate`, `refresh` | Performance overview (5 min cache) |
| `GET` | `/api/gsc/sites` | — | List accessible SC properties |
| `GET` | `/api/gsc/page-indexing` | `siteUrl`, dates, `refresh` | Top pages list (from GSC data) |
| `GET` | `/api/gsc/indexing` | `siteUrl`, `page`, `pageSize`, `status`, `search`, `refresh` | Full sitemap indexing module |
| `POST` | `/api/gsc/indexing/inspect` | `{ siteUrl, url }` | Inspect one URL into indexing store |
| `POST` | `/api/gsc/url-inspection` | `{ url, siteUrl }` + optional `?refresh=true` | Cached single-URL inspection for UI |

### Web vitals / PageSpeed

| Method | Path | Body / Query | Description |
|--------|------|--------------|-------------|
| `GET` | `/api/core-web-vitals` | `url`, `device`/`formFactor`, `refresh` | CrUX LCP / INP / CLS |
| `POST` | `/api/pagespeed/analyze` | `{ url, strategy, refresh }` | PSI analysis |
| `GET` | `/api/pagespeed/analyze` | `url`, `strategy`/`device`, `refresh` | Same via query string |

---

## 10. Frontend Structure & Client Flow

```
┌─ Header ──────────────────────────────────────────────┐
│  Brand · Refresh Live Data · Date range · Source pill │
└───────────────────────────────────────────────────────┘
┌─ Nav tabs ────────────────────────────────────────────┐
│ Overview │ GA │ FB │ IG │ YT │ GSC │ Page Perf │ Weekly│
└───────────────────────────────────────────────────────┘
┌─ Active panel ────────────────────────────────────────┐
│  KPI cards · Chart.js canvases · Tables · Forms       │
└───────────────────────────────────────────────────────┘
```

**Typical client data flow**

1. Global state holds the last analytics payload + GSC/PSI results.
2. Tab clicks only show/hide panels; they do not always refetch.
3. Explicit actions (Refresh, Analyze, Inspect, Apply filters) trigger `fetch`.
4. Chart.js instances are destroyed/recreated on re-render to avoid leaks.
5. Date-range controls can re-request GSC (and related panels) with new dates.

---

## 11. External Integrations

| Service | Auth | Module |
|---------|------|--------|
| Meta Graph API v20.0 | `ACCESS_TOKEN` (+ resolved page token) | `analysis.js` |
| GA4 Data API `v1beta` | OAuth refresh token | `analysis.js` |
| YouTube Data API v3 | OAuth | `analysis.js` |
| YouTube Analytics API v2 | OAuth | `analysis.js` |
| Search Console (Webmasters) | OAuth `webmasters.readonly` | `gsc.js` |
| URL Inspection API | Same OAuth | `gsc.js` |
| Chrome UX Report | `CRUX_API_KEY` | `gsc.js` |
| PageSpeed Insights v5 | `PAGESPEED_API_KEY` (+ fallbacks) | `pagespeed.js` |

---

## 12. Caching & Storage

| Store | Location | TTL / behavior |
|-------|----------|----------------|
| `gscCache` | `server.js` memory | 5 minutes |
| URL inspection cache | `gsc.js` memory | Per-inspection freshness via env TTL |
| `vitalsCache` | `gsc.js` memory | 1 hour |
| `pagespeedCache` | `pagespeed.js` memory | 1 hour |
| `activeScans` / `lastSitemapSync` | `gsc.js` memory | Scan job state |
| `gsc_url_inspections.json` | Disk | Persistent indexing/inspection records `{ pages: { ... } }` |

No SQL, Redis, or session store.

---

## 13. Utility & Diagnostic Scripts

| Script | Usage | Purpose |
|--------|-------|---------|
| `get-token.js` | `node get-token.js` | OAuth consent on `localhost:3005`; prints new `GOOGLE_REFRESH_TOKEN` |
| `ga4-diagnostic.js` | `node ga4-diagnostic.js` | Verify GA4 property access |
| `youtube-diagnostic.js` | `node youtube-diagnostic.js` | Verify YouTube channel + analytics access |
| `npm run fetch` | `node analysis.js` | Run analytics pipeline from CLI |

---

## 14. Data Payload Shapes

### `/api/data` & `/api/refresh` (simplified)

```json
{
  "fetchedAt": "ISO-8601",
  "warnings": ["optional platform skip messages"],
  "summary": {
    "fbFans": 0,
    "fbFollowers": 0,
    "igFollowers": 0,
    "igFollowing": 0,
    "igMediaCount": 0,
    "ytSubscribers": 0,
    "ytTotalViews": 0,
    "ytTotalVideos": 0,
    "ytPublishedAt": "YYYY-MM-DD"
  },
  "GA4_DAILY": [{ "date", "users", "newUsers", "sessions", "views", "bounceRate", "dur", "conversions", "demoViews" }],
  "GA4_CHANNELS": [{ "channel", "sessions" }],
  "GA4_PAGES": [{ "path", "title", "views", "duration" }],
  "FB_POSTS": [],
  "IG_POSTS": [],
  "YT_VIDEOS": [],
  "LI_DAILY": [],
  "LI_PAGES": [],
  "LI_SOCIAL_CHANNELS": [],
  "LI_DEVICES": []
}
```

### `/api/gsc` normalized response

```json
{
  "success": true,
  "data": {
    "siteUrl": "...",
    "dateRange": { "startDate", "endDate" },
    "clicks": 0,
    "impressions": 0,
    "ctr": 0,
    "position": 0,
    "dailyPerformance": [],
    "topQueries": [],
    "topPages": [],
    "countries": [],
    "devices": []
  }
}
```

---

## 15. Known Gaps & Notes

1. **Full indexing UI** — Backend routes `/api/gsc/indexing` exist; current UI uses single-URL inspection instead. Spec: `Implement Separate Google Search Console Indexing Module for All Pages.md`.
2. **LinkedIn** — Helper functions and `renderLinkedIn()` remain, but `runAnalytics()` does not fetch LinkedIn data.
3. **`exceljs`** — Dependency unused.
4. **`GA_MEASUREMENT_ID` / `YOUTUBE_API_KEY`** — Largely unused by primary server fetch paths (YouTube uses OAuth).
5. **Monolith UI** — All client logic lives in one HTML file; fine for an internal tool, harder to scale as a multi-app codebase.
6. **Older markdown** — Many `*.md` files are implementation prompts or partial docs (`ga_documentation.md`, `gsc_documentation.md`, `walkthrough.md`, etc.). Prefer this file as the unified reference.

---

## Quick Reference: User Action → API

| User action | Endpoint |
|-------------|----------|
| Open dashboard / load data | `GET /api/data` |
| Refresh Live Data | `POST /api/refresh` |
| GA dimension filter | `GET /api/analytics/dimension-values` then `POST /api/analytics/filter` |
| GA countries | `GET /api/analytics/countries` |
| Load / refresh GSC | `GET /api/gsc` |
| List GSC sites | `GET /api/gsc/sites` |
| Top pages helper | `GET /api/gsc/page-indexing` |
| Inspect one URL | `POST /api/gsc/url-inspection` |
| Core Web Vitals | `GET /api/core-web-vitals` |
| PageSpeed Analyze | `POST /api/pagespeed/analyze` |
| Weekly Comparison | _(no API — client-side)_ |

---

*End of documentation.*
