# Earnings Tracker

`workflows/earnings-tracker.json` · 23 nodes · 2 schedule triggers

This workflow publishes quarterly earnings results (EPS and revenue vs. analyst estimates) for a curated watchlist of 17 large-cap US companies, minutes after the results are released.

## Watchlist

`NVDA` `MSFT` `AAPL` `GOOGL` `AMZN` `META` `TSLA` `AMD` `AVGO` `TSM` `NFLX` `PLTR` `ORCL` `MU` `ASML` `JPM` `WMT`

The list is defined in the `Filter Premium Earnings Companies` and `Filter Active Premium Companies` nodes.

---

## Part 1: Daily preparation (10:00)

| Stage | Nodes | Purpose |
|---|---|---|
| 1. Calendar | `Earnings API Timing` → `Validate Daily Earnings Calendar` | Fetches today's earnings calendar and checks the response shape (`pre`, `after`, `notSupplied` arrays). |
| 2. Reset | `Clear Daily Earnings Watchlist` | Clears yesterday's watchlist. |
| 3. Classify | `Split Earnings By Timing` → `Keep BMO & AMC Only` | Labels each company **BMO** (before market open) or **AMC** (after market close) and drops companies with no timing information. |
| 4. Filter & save | `Filter Premium Earnings Companies` → `Save Daily Earnings Watchlist` | Keeps only the 17 tracked symbols and saves them for the polling stage. |

## Part 2: Live polling (every 5 minutes)

| Stage | Nodes | Purpose |
|---|---|---|
| 1. Window | `Determine Active Earnings Window` | Uses `America/New_York` time. It returns **BMO** between 06:00 and 10:00, **AMC** between 15:00 and 19:00, and stops the run at any other time. |
| 2. Fetch | `FMP Poll Current Earnings` | Gets today's earnings records from Financial Modeling Prep. |
| 3. Match | `Get Active Earnings Watchlist` → `Filter Active Premium Companies` → `Match FMP Earnings With Watchlist` | Matches on **symbol and exact date**, and takes key metadata from the watchlist rather than the API. |
| 4. Readiness | `Actual Earning Available` | Continues only when actual and estimated EPS and revenue are all finite numbers. |
| 5. Compute | `Calculate FMP Earnings Surprise` → `Prepare FMP Tweet Data` | Calculates the surprise %, guarding against division by zero and negative estimates (uses `Math.abs(estimated)`). |
| 6. Compose | `Build Earnings Tweet` → `Validate Tweet Length` | Formats EPS (`-$0.12`) and revenue (`$57.01B`), labels the result Beat, Miss or In Line, and enforces a 275-character safety limit. |
| 7. Dedupe | `If row does not exist1` | Skips the post if this symbol and date has already been published. |
| 8. Publish & verify | `Create Tweet` → `Validate X Earnings Publish Success` → `Record Posted Earnings` | Posts the tweet, requires a numeric tweet ID in the response, and only then records the post. |

---

## Data Tables

| Table | Columns | Role |
|---|---|---|
| Daily Earnings Watchlist | `symbol`, `companyName`, `earningsDate`, `earningsTiming` | Today's tracked reporters |
| Posted Earnings | `symbol`, `earningsDate` | Duplicate protection |

## Design notes

- **Two-stage design.** The calendar is fetched once a day, while the expensive polling only runs in narrow time windows on matching symbols. This keeps API usage low and lets the bot react within minutes.
- **Fail closed.** Missing numbers stop the item instead of producing a misleading post.
