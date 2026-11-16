<p align="center">
  <img src="assets/fynbeat-logo.jpg" width="110" alt="FynBeat logo">
</p>

<h1 align="center">FynBeat – Automated Market Intelligence Bot</h1>

<p align="center">
  Production n8n workflows that turn raw market data into verified, ready-to-read posts on X (Twitter) — every trading day, with no manual work.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/n8n-workflow%20automation-EA4B71?logo=n8n&logoColor=white" alt="n8n">
  <img src="https://img.shields.io/badge/JavaScript-data%20processing-F7DF1E?logo=javascript&logoColor=black" alt="JavaScript">
  <img src="https://img.shields.io/badge/X%20API-v2-000000?logo=x&logoColor=white" alt="X API v2">
  <img src="https://img.shields.io/badge/status-live%20in%20production-2ea44f" alt="Live">
</p>

<p align="center">
  🔴 <b>Live on X:</b> <a href="https://x.com/FynBeatHq">@FynBeatHq</a>
</p>

---

## Overview

FynBeat is a set of always-on n8n workflows that monitor financial data sources and publish posts on X automatically. It is made of two independent pipelines:

| Pipeline | What it does | Cadence |
|---|---|---|
| **[Crypto ETF Flow Tracker](docs/etf-flow-tracker.md)** | Publishes daily US spot Bitcoin & Ethereum ETF net flows, with streak and 5-day context, plus a branded weekly chart. | Daily + weekly (Saturday) |
| **[Earnings Tracker](docs/earnings-tracker.md)** | Watches 17 large-cap US companies and posts EPS & revenue vs. estimates the moment results are released. | Every 5 min during US pre-market & after-hours windows on earnings days |

Both pipelines follow the same design principle: **fail closed**. If the data is missing, incomplete, inconsistent or already published, nothing gets posted.

---

## Live posts

Real, unedited posts published automatically by these workflows on [@FynBeatHq](https://x.com/FynBeatHq).

**Daily ETF flows with streak and 5-day context**

<img src="assets/tweet-daily-etf.png" width="600" alt="Daily crypto ETF flows post">

**Weekly ETF report: chart rendered from an HTML/CSS template**

<img src="assets/tweet-weekly-etf.png" width="600" alt="Weekly crypto ETF flows post with chart">

**Earnings results, posted minutes after release**

<img src="assets/tweet-earnings-aapl.png" width="600" alt="AAPL earnings post">

<img src="assets/tweet-earnings-amd.png" width="600" alt="AMD earnings post">

---

## Architecture

### 1. Crypto ETF Flow Tracker

```mermaid
flowchart LR
    A[⏰ Polling schedule<br/>several times a day] --> B[CryptoETF API<br/>detect latest trading date]
    B --> C{New date &<br/>not yet published?}
    C -- no --> X1[Stop]
    C -- yes --> D[SoSoValue API<br/>primary source]
    D --> E{BTC & ETH<br/>both ready?}
    E -- no --> F[Farside fallback<br/>HTML scraping + completeness checks]
    F --> G
    E -- yes --> G[Build final dataset]
    G --> H[(Flow history<br/>Data Table)]
    H --> I[Streak & 5-day<br/>net flow context]
    I --> J[Compose tweet<br/>≤ 280 chars]
    J --> K[𝕏 Post]
    K --> L[(Save last<br/>published date)]
```

```mermaid
flowchart LR
    W[⏰ Saturday 18:00] --> W1[Fetch BTC & ETH<br/>daily history]
    W1 --> W2{Mon–Fri<br/>complete?}
    W2 -- yes --> W3[Build HTML/CSS<br/>chart template]
    W3 --> W4[Render image<br/>HCTI API]
    W4 --> W5[X media upload<br/>init → append → finalize]
    W5 --> W6[𝕏 Post with image]
```

### 2. Earnings Tracker

```mermaid
flowchart LR
    subgraph Daily prep
    P1[⏰ 10:00 daily] --> P2[Earnings calendar API]
    P2 --> P3[Split by timing<br/>BMO / AMC]
    P3 --> P4{In 17-company<br/>watchlist?}
    P4 -- yes --> P5[(Daily watchlist<br/>Data Table)]
    end
    subgraph Live polling
    Q1[⏰ Every 5 min] --> Q2{Inside NY window?<br/>06–10 BMO / 15–19 AMC}
    Q2 -- yes --> Q3[FMP earnings API]
    Q3 --> Q4[Match with watchlist<br/>symbol + date]
    Q4 --> Q5{Actual EPS &<br/>revenue released?}
    Q5 -- yes --> Q6[Calculate surprise %]
    Q6 --> Q7{Already posted?}
    Q7 -- no --> Q8[𝕏 Post]
    Q8 --> Q9[(Record posted<br/>earnings)]
    end
    P5 -.-> Q4
```

---

## Engineering highlights

- **Multi-source data with fallback.** ETF flows come from SoSoValue; if an asset is not ready, the workflow falls back to parsing Farside's HTML table and checks that every fund column is filled in, so a partial total is never accepted.
- **Duplicate protection.** n8n Data Tables store the last published trading date, the last published week and every earnings post (symbol + date), so the same data is never posted twice.
- **Fail-closed validation at every stage.** Dates, symbols, numeric types and asset completeness are checked before anything is built. A real `0` flow is treated as valid, and missing data is never treated as `0`.
- **Time-zone aware scheduling.** Earnings polling runs only inside US pre-market and after-hours windows, using `America/New_York` so daylight saving time is handled automatically.
- **Stateful context.** A rolling flow history powers phrases like *"3 consecutive days of inflows"* and the 5-day net flow, with a compact fallback format when the tweet would exceed 280 characters.
- **Programmatic graphics.** The weekly report is a dynamic HTML/CSS template with an auto-scaling Y-axis, rendered to an image and posted through the X API v2 chunked media upload.
- **Publish verification.** The earnings post is only recorded as done after the X API returns a valid tweet ID.

---

## Tech stack

| Area | Tools |
|---|---|
| Orchestration | n8n (Schedule Trigger, Code, IF, Merge, HTTP Request, Data Tables, X node) |
| Logic | JavaScript (Code nodes), Luxon date handling |
| Data sources | CryptoETF API, SoSoValue OpenAPI, Farside (fallback), Financial Modeling Prep, EarningsAPI |
| Rendering | HTML/CSS → HCTI (htmlcsstoimage) API |
| Publishing | X API v2 (OAuth 2.0 for posts, OAuth 1.0a for media upload) |
| State | n8n Data Tables |

---

## Repository structure

```text
.
├── workflows/
│   ├── etf-flow-tracker.json     # daily + weekly crypto ETF pipeline (59 nodes)
│   └── earnings-tracker.json     # 17-company earnings pipeline (23 nodes)
├── docs/
│   ├── etf-flow-tracker.md       # node-by-node walkthrough
│   ├── earnings-tracker.md       # node-by-node walkthrough
│   └── setup.md                  # how to import and run
└── assets/                       # logo and sample report image
```

---

## Getting started

1. Import a file from `workflows/` into n8n (**Workflows → Import from File**).
2. Create the required credentials and Data Tables.
3. Replace the `YOUR_...` placeholders with your own API keys.

The full guide is in **[docs/setup.md](docs/setup.md)**.

> All API keys, credential IDs and instance identifiers have been removed from the exported workflows.

---

## Disclaimer

This project is for informational purposes only and is not investment advice. Data accuracy depends on third-party providers.
