# Setup

## Requirements

- n8n (self-hosted or cloud) with **Data Tables** support
- X developer account (OAuth 2.0 for posting; OAuth 1.0a for media upload in the weekly report)
- API keys for the data providers below

## 1. Import

In n8n, go to **Workflows → Import from File** and select a file from `workflows/`. The imported workflows are **inactive** by default.

## 2. API keys

Search each workflow for `YOUR_` and replace the placeholders. For better security, move them into n8n credentials (Header Auth / Query Auth) instead of plain parameters.

| Placeholder | Provider | Used in |
|---|---|---|
| `YOUR_CRYPTOETF_API_KEY` | cryptoetf.today | ETF: daily summary, weekly BTC/ETH history |
| `YOUR_SOSOVALUE_API_KEY` | SoSoValue OpenAPI | ETF: primary daily flow source |
| `YOUR_FMP_API_KEY` | Financial Modeling Prep | Earnings: live polling |
| `YOUR_EARNINGSAPI_KEY` | earningsapi.com | Earnings: daily calendar with BMO/AMC timing |

## 3. Credentials

| n8n credential type | Purpose |
|---|---|
| X OAuth2 API | Posting tweets |
| X OAuth1 API | Media upload (weekly chart) |
| HTTP Basic Auth | HCTI (htmlcsstoimage.com) image rendering |

## 4. Data Tables

Create the tables below, then select them in the nodes that show a `YOUR_..._TABLE_ID` placeholder.

| Table | Columns | Initial row |
|---|---|---|
| ETF Flow Publish State | `lastPublishedDate` (string) | one row with a past date, e.g. `2026-01-01` |
| ETF Flow History | `date` (string), `symbol` (string), `direction` (string), `netFlow` (number) | — |
| ETF Weekly Publish State | `lastPublishedWeek` (string) | one row with a past date |
| Daily Earnings Watchlist | `symbol`, `companyName`, `earningsDate`, `earningsTiming` (string) | — |
| Posted Earnings | `symbol`, `earningsDate` (string) | — |

## 5. Time zone

The ETF workflow's schedule and time gate assume the instance time zone is **Europe/Istanbul**. Earnings polling windows use `America/New_York` regardless of the instance setting.

## 6. Test, then activate

Run each workflow manually once and check the outputs before activating it.
