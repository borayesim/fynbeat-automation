# Crypto ETF Flow Tracker

`workflows/etf-flow-tracker.json` · 59 nodes · 2 schedule triggers

This workflow publishes **daily** net flows for US spot Bitcoin and Ethereum ETFs and a **weekly** chart summary.

---

## Part 1: Daily flows

### Schedule

The workflow polls hourly from 03:00 to 07:00, then at 08:00, 10:00, 13:00, 16:00, 19:00 and 22:00 (instance time zone: Europe/Istanbul). ETF flow data is published the morning after each US trading day, often with delays, so polling continues until the data is complete.

### Flow

| Stage | Nodes | Purpose |
|---|---|---|
| 1. Detect trading date | `CryptoETF - Daily Flow Summary` → `Select Tracked ETF Assets` → `Validate ETF Flow Data` → `Determine ETF Trading Date` → `Prepare Final ETF Dataset` | Uses the CryptoETF summary endpoint to find the latest trading date that has data for **both** BTC and ETH. |
| 2. Early duplicate check | `Get ETF Publish State Before Final Data` → `Check ETF Date Before Final Data` → `ETF Date Needs Final Data?` | Skips everything if that date has already been published. |
| 3. Time gate | `Final ETF Data Check Time?` | Fetches final data only at the scheduled hours, to save API calls. |
| 4. Primary source | `Prepare SoSoValue ETF Requests` → `Get SoSoValue ETF Summary` → `Build SoSoValue ETF Dataset` | Gets the authoritative `total_net_inflow` for the target date and converts it to USD millions. |
| 5. Fallback | `SoSoValue Fallback Needed?` → `Prepare Farside Fallback Requests` → `Get Farside Fallback Data` → `Attach Farside Fallback Metadata` → `Parse Farside Fallback Data` → `Evaluate Farside Fallback Result` | Runs only for assets SoSoValue could not resolve. Parses the Farside HTML table (see the safety rules below). |
| 6. Merge & verify | `Build Resolved SoSoValue Dataset` → `Resolved Core ETF Data Ready?` → `Build Final ETF Dataset` → `Final ETF Data Ready?` | Each asset must be resolved by exactly one source, on the exact target date, with a finite number. |
| 7. Final duplicate check | `Get Last Published ETF Date` → `Check New ETF Trading Date` → `New ETF Trading Date?` | Checks again right before publishing. |
| 8. History & context | `Build Summary ETF Dataset` → `Preserve ETF Daily Dataset` → `Prepare BTC & ETH History Records` → `Save BTC & ETH Flow History` (upsert) → `Get BTC & ETH Flow History` → `Calculate ETF Historical Context` → `Build Historical Context Object` → `Attach Historical Context to ETF Dataset` → `Prepare Historical ETF Context` | Stores every day's flow, then calculates the current inflow/outflow **streak** (shown when ≥ 3 days) and the **5-day net flow**. |
| 9. Publish | `Prepare Daily ETF Tweet` → `Post Daily ETF Flows to X` → `Update Last Published ETF Date` | Composes the post, switches to a compact format if it is longer than 280 characters, posts it, and saves the published date. |

### Farside fallback safety rules

- The page must contain the expected title ("Bitcoin ETF Flow" / "Ethereum ETF Flow"), which prevents mixing up the BTC and ETH pages.
- An exact date row match is required (`2026-10-02` → `2 Oct 2026`).
- If any fund cell is empty or shows `-`, the row is **not ready**, even when Farside already shows a partial total.
- Accounting negatives such as `(5.0)` are parsed correctly, and `0.0` is a valid value.

---

## Part 2: Weekly report

### Schedule

Every Saturday at 18:00.

### Flow

| Stage | Nodes | Purpose |
|---|---|---|
| 1. Data | `Get BTC Daily Flow History` → `Get ETH Daily Flow History` → `Build Weekly BTC & ETH Dataset` | Filters Monday–Friday of the current week and sums the weekly net flow. |
| 2. Readiness | `Is Weekly ETF Data Ready?` → `Get Last Published Week` → `Is New Weekly Report?` | Requires Friday's data for both assets and checks the week has not been published yet. |
| 3. Graphic | `Prepare Weekly ETF Graphic` → `Build FynBeat Weekly Template` → `Render Weekly ETF Graphic` → `Download Weekly ETF Graphic` | Builds a 1200×675 HTML/CSS chart with an auto-scaling axis and renders it to PNG through the HCTI API. Throws an error if any weekday is missing, rather than drawing a zero. |
| 4. Media upload | `Prepare X Media Upload Metadata` → `Initialize X Weekly Media Upload` → `Merge X Media Upload Date` → `Append X Weekly Media` → `Finalize X Weekly Media Upload` | Uses the X API v2 chunked media upload (INIT / APPEND / FINALIZE) with OAuth 1.0a. |
| 5. Publish | `Prepare Weekly ETF Tweet` → `Post Weekly ETF Report to X` → `Update Last Published Week` | Posts the summary text with the image attached. |

---

## Data Tables

| Table | Columns | Role |
|---|---|---|
| ETF Flow Publish State | `lastPublishedDate` | Last daily post (single row) |
| ETF Flow History | `date`, `symbol`, `netFlow`, `direction` | Rolling history for streak and 5-day calculations |
| ETF Weekly Publish State | `lastPublishedWeek` | Last weekly post (single row) |
