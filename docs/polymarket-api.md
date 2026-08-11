# Polymarket API Reference

> Complete reference for all three Polymarket APIs: Gamma, Data, and CLOB.

> ⚠️ **CLOB V2 since 2026-04-28 — no V1 compatibility.** This page was written
> in Feb 2026 and re-verified against the official docs on **2026-08-06**:
> endpoint tables, fee section, Data API parameters, order payload, error codes
> and all Python snippets.
>
> **`py-clob-client` is archived and non-functional** — its repository states it
> "should not be used for new or existing integrations". Every snippet on this
> page now uses the current official SDK; see [SDKs](#sdks--client-libraries).

## Table of Contents

- [Three APIs Overview](#three-apis-overview)
- [Base URLs](#base-urls)
- [Authentication](#authentication)
- [Gamma API (Markets & Events)](#gamma-api-markets--events)
- [Data API (Positions & Trades)](#data-api-positions--trades)
- [CLOB API (Orderbook & Trading)](#clob-api-orderbook--trading)
  - [Market Data Endpoints](#market-data-endpoints)
  - [Trading Endpoints](#trading-endpoints)
- [Bridge API](#bridge-api)
- [Order Types](#order-types)
- [Signature Types](#signature-types)
- [Tick Sizes & Neg Risk](#tick-sizes--neg-risk)
- [Fee Rates](#fee-rates)
- [Rate Limits](#rate-limits)
- [Error Codes](#error-codes)
- [SDKs & Client Libraries](#sdks--client-libraries)

---

## Three APIs Overview

Polymarket is served by **three separate APIs**, each handling a different domain:

| API | Base URL | Auth Required | Purpose |
|-----|----------|---------------|---------|
| **Gamma API** | `https://gamma-api.polymarket.com` | No | Markets, events, tags, series, comments, sports, search, profiles |
| **Data API** | `https://data-api.polymarket.com` | No | User positions, trades, activity, holders, open interest, leaderboards |
| **CLOB API** | `https://clob.polymarket.com` | Partial (trading only) | Orderbook, pricing, order placement/cancellation, heartbeat |
| **Bridge API** | `https://bridge.polymarket.com` | No | Deposits & withdrawals (proxy for fun.xyz) |

---

## Base URLs

```
Gamma:  https://gamma-api.polymarket.com
Data:   https://data-api.polymarket.com
CLOB:   https://clob.polymarket.com
Bridge: https://bridge.polymarket.com
```

---

## Authentication

### Public vs Authenticated

- **Gamma API** and **Data API**: Fully public — no auth required
- **CLOB read endpoints** (orderbook, prices, spreads): No auth required
- **CLOB trading endpoints** (orders, cancellations, heartbeat): Require L2 auth headers

### Two-Level Auth Model

| Level | Method | Used For |
|-------|--------|----------|
| **L1** | EIP-712 signed message with private key | Creating/deriving API credentials, signing orders |
| **L2** | HMAC-SHA256 with API credentials | Authenticating trading API requests |

### L1 Auth Headers (Key Management)

```
POLY_ADDRESS:   0xYOUR_WALLET
POLY_SIGNATURE: <EIP-712 signature>
POLY_TIMESTAMP: <unix timestamp>
POLY_NONCE:     <nonce, default 0>
```

### L2 Auth Headers (Trading)

```
POLY_ADDRESS:    0xYOUR_WALLET
POLY_SIGNATURE:  <HMAC-SHA256 signature>
POLY_TIMESTAMP:  <unix timestamp>
POLY_API_KEY:    <your apiKey>
POLY_PASSPHRASE: <your passphrase>
```

The HMAC signature is computed over `timestamp + method + path + body` using your API `secret`.

### Creating API Credentials

With the unified SDK, credential derivation happens inside `create()` — you
never handle the L2 creds yourself:

```python
import os
from polymarket import SecureClient

with SecureClient.create(
    private_key=os.environ["POLYMARKET_PRIVATE_KEY"],
    wallet=os.environ["POLYMARKET_WALLET_ADDRESS"],
) as client:
    ...
```

With the CLOB-only client the two steps are explicit — note it is
`create_or_derive_api_key()`, singular:

```python
import os
from py_clob_client_v2 import ClobClient

# L1: derive credentials from the wallet
client = ClobClient(host=host, chain_id=chain_id, key=os.environ["PK"])
creds = client.create_or_derive_api_key()

# L1 + L2: fully authenticated client
client = ClobClient(host=host, chain_id=chain_id, key=os.environ["PK"], creds=creds)
```

### REST API Key Endpoints

```
POST https://clob.polymarket.com/auth/api-key     # Create new credentials
GET  https://clob.polymarket.com/auth/derive-api-key  # Derive existing credentials
```

---

## Gamma API (Markets & Events)

Base: `https://gamma-api.polymarket.com`

### Markets

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/markets` | GET | List markets (paginated, filterable) |
| `/markets/{id}` | GET | Get market by condition ID |
| `/markets?slug={slug}` | GET | Get market by slug |
| `/markets/{id}/tags` | GET | Get tags for a market |
| `/markets/sampling` | GET | Get sampling markets (random subset) |
| `/markets/simplified` | GET | Get simplified market objects |
| `/markets/sampling/simplified` | GET | Sampling + simplified |

### Events

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/events` | GET | List events (paginated) |
| `/events/{id}` | GET | Get event by ID |
| `/events?slug={slug}` | GET | Get event by slug |
| `/events/{id}/tags` | GET | Get event tags |

### Tags, Series, Search

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/tags` | GET | List all tags |
| `/tags/{id}` | GET | Get tag by ID |
| `/tags?slug={slug}` | GET | Get tag by slug |
| `/tags/{id}/related` | GET | Get related tags |
| `/series` | GET | List series |
| `/series/{id}` | GET | Get series by ID |
| `/public-search?q=...` | GET | Search markets, events, profiles — the param is **`q`**, see below |

### Comments & Profiles

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/comments` | GET | List comments (by market, user, or ID) |
| `/comments/{id}` | GET | Get comment by ID |
| `/profiles/{address}` | GET | Get public profile by wallet address |

### Sports

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/sports/metadata` | GET | Sports metadata |
| `/sports/market-types` | GET | Valid sports market types |
| `/sports/teams` | GET | List teams |

### ⚠️ Gamma traps verified 2026-08-11

Six behaviours that differ from what the endpoint tables imply. Each was hit in
a real study and each cost time or corrupted a sample before it was found.

**`/public-search` takes `q`, not `query`.** Sending `query=` returns
HTTP 422 `{"type":"validation error","error":"query argument \"q\": empty"}` —
the message names the parameter it wanted, which is easy to misread as a
complaint about the value you sent.

**`closed_time_min` / `closed_time_max` are recognised but return HTTP 500.**
There is no way to filter by resolution time. To enumerate markets that
*resolved* in a window: page `end_date_min`/`end_date_max` and post-filter on
`closedTime` client-side, widening the endDate scan on both sides — measured
`|closedTime − endDate| > 2 days` for ~33% of markets, extremes 217 days early
and 57 days late. A widened scan is not optional padding: at full scale 11.5% of
one 180-day frame came from the widening bands, and those are exactly the
early- and late-resolvers.

**`closedTime` is not ISO-8601.** Format is `YYYY-MM-DD HH:MM:SS+00` (space, no
`T`, two-digit offset) while `endDate`, `startDate` and `updatedAt` are ISO-Z.
Normalise before comparing or the comparison silently fails.

**Pagination caps are silent.** `limit` is capped at 100 without an error
(`limit=500` returns 100 rows). `offset` works to 2000 and returns HTTP 422 at
~2125 pointing at `/markets/keyset`. A binary search on `offset` measures the
cap, not the row count.

**There is no `category` field.** Not on markets, not on embedded events —
verified across 800 sampled markets. Tags are the only category signal: either
`GET /markets/{id}/tags` per market, or enumerate `/events`, which embeds both
`tags[]` and `markets[]` and has a higher rate limit (500 vs 300 per 10s).

**Market text mutates after resolution.** `updatedAt > closedTime` on 800/800
sampled resolved markets. Treat `description` as untrusted for any
point-in-time analysis; `question` and event `title` are safer but not verified
immutable — snapshot the text you use and record `updatedAt` alongside it.

### Prices History

```
GET https://clob.polymarket.com/prices-history?market={token_id}&startTs={s}&endTs={e}&fidelity=1
```

**`interval=max` is a trap.** It silently ignores `fidelity` below 10 (serving
600-second bars when you asked for 60), and for markets that closed more than
~31–33 days ago it returns `{"history":[]}` — which reads as "data was purged"
when the data is in fact still there and reachable through explicit timestamps.

Verified behaviour with `startTs`/`endTs` (measured 2026-08-11 on 8 resolved
markets, ages 13 days to 6 months, volumes $5k–$99k):

- **1-minute history persists at least 6 months after resolution**, including
  $5k long-tail markets. 1 minute is the floor — `fidelity=0` still returns a
  60-second grid.
- The window has a **maximum span between 15 and 21 days**: 15 days succeeds,
  21 and 30 return HTTP 400
  `{"error":"invalid filters: 'startTs' and 'endTs' interval is too long"}`.
  The limit is fidelity-independent, so page in ≤15-day chunks.
- No points-per-response cap was found — single responses of 20,155 and 21,594
  points returned complete.
- The series is **carry-forward**: a point exists every 60 seconds even in
  minutes with no trade, so "the price at t" can be a stale print from hours
  earlier. It is trade-price only — there is no bid/ask, spread or depth
  history anywhere. For anything execution-related use Data API `/trades`
  prints instead, which are retained for the same horizon and carry real
  timestamps and sizes.
- Active markets expose `bestBid`/`bestAsk` directly on the Gamma market object,
  so a live quote needs no CLOB call at all.

---

## Data API (Positions & Trades)

Base: `https://data-api.polymarket.com`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/positions?user={address}` | GET | Current positions for a user |
| `/closed-positions?user={address}` | GET | Closed positions |
| `/positions?market={condition_id}` | GET | Positions for a market |
| `/top-holders?market={condition_id}` | GET | Top holders for a market |
| `/portfolio-value?user={address}` | GET | Total value of user's positions |
| `/trades?user={address}` | GET | Trades for a user — **see the `takerOnly` trap below** |
| `/trades?market={condition_id}` | GET | Trades for a market |
| `/activity?user={address}` | GET | User activity (incl. SPLIT / MERGE / REDEEM / CONVERSION) |
| `/v1/leaderboard` | GET | Trader leaderboard rankings — **see `timePeriod` below** |
| `/open-interest?market={condition_id}` | GET | Open interest for a market |
| `/volume?event_id={id}` | GET | Live volume for an event |
| `/total-markets-traded?user={address}` | GET | Total markets a user has traded |
| `/accounting-snapshot?user={address}` | GET | Download ZIP of CSV accounting data |

### ⚠️ Two defaults that silently corrupt research samples

Both verified 2026-08-06. Both have already produced months of wrong data.

**`GET /trades` defaults to `takerOnly=true`.** Omit the parameter and you get
only the fills where the wallet *crossed the spread* — for a liquidity provider
that is not a sample of their trading, it is a sample of one side of it. Always
send it explicitly:

```
GET /trades?user=0x…&takerOnly=false&limit=500&offset=0&start=…&end=…
```

| Param | Default | Limit |
|-------|---------|-------|
| `takerOnly` | **`true`** | — |
| `limit` | 100 | max 10,000 |
| `offset` | 0 | **max 10,000 — beyond returns 400, not an empty page** |
| `start` / `end` | ~3-year window | epoch **seconds** |
| `side`, `market`, `eventId`, `filterType`+`filterAmount` | — | `market` and `eventId` are mutually exclusive |

The response has **no maker/taker role field** — once both are included you
cannot tell which a given fill was. `side` is BUY/SELL, not role. Fields:
`proxyWallet`, `side`, `asset`, `conditionId`, `size`, `price`, `timestamp`,
`title`, `slug`, `eventSlug`, `outcome`, `outcomeIndex`, `transactionHash`.

**`GET /v1/leaderboard` has no `window` parameter.** The parameter is
`timePeriod`, uppercase, and it defaults to `DAY`. Sending `window=month` is
silently ignored and you get the *daily* board — which looks like a working
query and ranks on the noisiest surface the API offers.

| Param | Values | Default |
|-------|--------|---------|
| `timePeriod` | `DAY` \| `WEEK` \| `MONTH` \| `ALL` | **`DAY`** |
| `category` | `OVERALL`, `POLITICS`, `SPORTS`, `ESPORTS`, `CRYPTO`, `CULTURE`, `MENTIONS`, `WEATHER`, `ECONOMICS`, `TECH`, `FINANCE` | `OVERALL` |
| `orderBy` | `PNL` \| `VOL` | `PNL` |
| `limit` | 1–50 | 25 |
| `offset` | 0–**1000** | 0 |

Response rows: `rank`, `proxyWallet`, `userName`, `vol`, `pnl`, `profileImage`,
`xUsername`, `verifiedBadge`. With `limit ≤ 50` and `offset ≤ 1000`, a single
period tops out around 1,050 rows.

**General rule both cases share:** an unknown query parameter is *ignored*, not
rejected. The server answers with its own default and the response looks
perfectly healthy. If a filter matters, assert that it was applied.

### Builder Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/builders/leaderboard` | GET | Aggregated builder leaderboard |
| `/builders/volume-timeseries` | GET | Daily builder volume time-series |

---

## CLOB API (Orderbook & Trading)

Base: `https://clob.polymarket.com`

### Market Data Endpoints (Public — No Auth)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/book?token_id={id}` | GET | Order book for a token |
| `/books` | POST | Batch order books (body: token IDs) |
| `/price?token_id={id}&side={BUY\|SELL}` | GET | Best market price |
| `/prices` | GET/POST | Batch market prices |
| `/midpoint?token_id={id}` | GET | Midpoint price (avg best bid/ask) |
| `/midpoints` | GET/POST | Batch midpoints (max 500 tokens) |
| `/spread?token_id={id}` | GET | Spread (best ask - best bid) |
| `/spreads` | GET/POST | Batch spreads |
| `/last-trade-price?token_id={id}` | GET | Last trade price and side |
| `/last-trade-prices` | GET/POST | Batch last trade prices (max 500) |
| `/tick-size?token_id={id}` | GET | Minimum tick size |
| `/tick-size/{token_id}` | GET | Tick size (path parameter) |
| `/clob-markets/{condition_id}` | GET | **All CLOB params for a market in one call** — tokens, tick size, `mbf`/`tbf` base fees, `fd` fee curve, rewards, RFQ |
| `/fee-rate?token_id={id}` | GET | Fee rate for a token |
| `/fee-rate/{token_id}` | GET | Fee rate (path parameter) |
| `/prices-history` | GET | Historical price data |
| `/markets?next_cursor={cursor}` | GET | CLOB market list (paginated) |
| `/time` | GET | Server time (Unix timestamp) |

### Trading Endpoints (Require L2 Auth)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/order` | POST | Place single order |
| `/orders` | POST | Place batch orders (max 15) |
| `/order/{id}` | DELETE | Cancel single order |
| `/orders` | DELETE | Cancel multiple orders (max 3000) |
| `/cancel-all` | DELETE | Cancel all open orders |
| `/cancel-market-orders` | DELETE | Cancel all orders for a market+asset |
| `/order/{id}` | GET | Get single order by ID |
| `/orders` | GET | Get user's open orders (paginated) |
| `/trades` | GET | Get user's trades (paginated) |
| `/heartbeat` | POST | Send heartbeat (prevents auto-cancel) |
| `/order-scoring/{id}` | GET | Check if order is scoring for rewards |

#### POST /order — Place Single Order

> ⚠️ **The payload below is the CLOB V1 order struct and no longer works.** V2
> (live 2026-04-28) **removed** `nonce`, `feeRateBps` and `taker` from the signed
> order and **added** `timestamp` (ms, provides uniqueness), `metadata` and
> `builder`. EIP-712 domain version went `1` → `2` with a new `verifyingContract`.
> A V1-signed order is rejected — typically as `invalid fee rate (0)`.
> See [migration_to_v2.md](migration_to_v2.md).

```json
{
  "order": {
    "salt": 12345678,
    "maker": "0xYOUR_WALLET",
    "signer": "0xYOUR_WALLET",
    "tokenId": "12345...",
    "makerAmount": "50000000",
    "takerAmount": "100000000",
    "expiration": "0",
    "timestamp": "1777374000000",
    "metadata": "0x00...",
    "builder": "0x00...",
    "side": "BUY",
    "signatureType": 0,
    "signature": "0x..."
  },
  "orderType": "GTC",
  "tickSize": "0.01",
  "negRisk": false
}
```

**Response:**
```json
{
  "orderID": "0xabc123...",
  "status": "LIVE",
  "transactionsHashes": []
}
```

**Statuses**: `live` (resting), `matched` (immediately filled), `delayed` (sports 3s delay), `unmatched` (delayed order placed on book).

#### POST /orders — Batch Orders (Max 15 per request)

Orders processed in parallel. Each gets individual status.

#### Heartbeat

If heartbeats are enabled and not sent regularly, **all open orders are auto-cancelled**. Useful for automated systems that need a dead-man's switch.

```
POST /heartbeat
```

---

## Bridge API

Base: `https://bridge.polymarket.com` (proxies fun.xyz)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/deposit-addresses` | POST | Create deposit addresses |
| `/withdrawal-addresses` | POST | Create withdrawal addresses |
| `/quote` | GET | Get fee/output quote for bridge |
| `/supported-assets` | GET | List supported chains and tokens |
| `/transaction-status/{id}` | GET | Track bridge transaction status |

---

## Order Types

| Type | Code | Description |
|------|------|-------------|
| **GTC** | `"GTC"` | Good Till Cancel — rests on book until filled or cancelled |
| **GTD** | `"GTD"` | Good Till Date — auto-expires at specified time |
| **FOK** | `"FOK"` | Fill Or Kill — fill entirely or reject immediately |
| **FAK** | `"FAK"` | Fill And Kill — fill what's available, cancel the rest |

**Post-Only**: Orders that would cross the spread are rejected (guarantees maker status).

## Signature Types

| Type | Value | Description |
|------|-------|-------------|
| **EOA** | `0` | Standard Ethereum wallet. Funder = EOA address, needs POL for gas. |
| **POLY_PROXY** | `1` | Magic Link proxy wallet. Requires exported PK from Polymarket.com. |
| **GNOSIS_SAFE** | `2` | Gnosis Safe multisig proxy (most common for new users). |

The wallet address on Polymarket.com is the **proxy wallet** (funder). Use it as the `funder` parameter.

## Tick Sizes & Neg Risk

Each market has specific tick size and neg_risk parameters that **must** be included in order requests.

```python
resp = client.get_market(condition_id="0xCONDITION_ID")
tick_size = resp["minimum_tick_size"]  # "0.01" or "0.001"
neg_risk = resp["neg_risk"]            # True or False
```

**Tick size changes dynamically**: When price > 0.96 or < 0.04, tick size changes from 0.01 to 0.001.

## Fee Rates

> ⚠️ Rewritten **2026-08-06** against <https://docs.polymarket.com/trading/fees>.
> The previous version of this section described the Jan–Feb 2026 rollout, when
> fees applied only to a handful of market types. That has not been true since
> the **2026-03-30 fee structure V2**: most categories now charge takers.

### Which markets charge fees

**Most of them.** Fees are per *category*, and only **Geopolitics** is fee-free.
Do not assume a market is free — read its parameters (below).

| Category | Taker rate | Maker rebate |
|----------|-----------:|-------------:|
| Crypto | 0.07 | 20% |
| Sports | 0.05 | 15% |
| Economics, Culture, Weather, Other | 0.05 | 25% |
| Finance, Politics, Mentions, Tech | 0.04 | 25% |
| **Geopolitics** | **0** | — |

Sports moved from 0.03 → 0.05 (rebate 25% → 15%) on **2026-07-10**. Rates change;
treat this table as a fallback, not as the source of truth.

### Fee formula

```
fee = C × feeRate × p × (1 - p)        # C = shares traded, p = price, in USDC
```

- **Makers are never charged.** Only takers pay. Maker rebates are paid daily
  under program terms, *not* per fill — do not credit them to a simulated fill.
- The fee is charged **in USDC for both buys and sells**.
- Rounded to 5 decimal places; the smallest fee charged is `0.00001 USDC`.
- Symmetric about p=0.5, so 30¢ and 70¢ cost the same in dollars.

**As a share of money staked** the fee is `feeRate × (1 − p)` — monotonically
worse the cheaper the contract, which is the part that surprises people:

| Entry price | Cost at rate 0.05 |
|-------------|------------------:|
| 0.15 | **425 bps of stake** |
| 0.50 | **250 bps** |
| 0.85 | **75 bps** |

For anything priced in basis points of capital, that is a first-order term, not
a rounding detail.

### Reading the real parameters per market

```
GET https://clob.polymarket.com/clob-markets/{condition_id}
```

Abbreviated keys: `mbf` maker base fee (bps), `tbf` taker base fee (bps), and
`fd` = { `r` rate, `e` curve exponent, `to` taker-only }. The published formula
above has no exponent term (i.e. `e = 1`), but `fd.e` is authoritative per
market and has carried other values — read it, don't assume it.

```json
{ "mos": 5, "mts": 0.01, "mbf": 0, "tbf": 500,
  "fd": { "r": 0.05, "e": 1, "to": true } }
```

`mbf`/`tbf` can read `0` while `fd.r` is live, so **do not derive "fee-free"
from a single field** — check both.

### ⚠️ `feeRateBps` is a V1 field and means nothing now

CLOB V2 removed `feeRateBps` from the signed order struct entirely; the protocol
sets the fee at match time. A `fee_rate_bps: "0"` seen on a WebSocket event, or a
uniform `1000` on an old market object, is a vestige of the retired mechanism —
**neither is evidence that a market is fee-free**. Reading a zero there as "fees
are zero" is a mistake that has already been made in production.

## Rate Limits

All limits enforced via Cloudflare throttling (sliding windows, requests are delayed not rejected).

### General
| Endpoint | Limit |
|----------|-------|
| General | 15,000 req / 10s |
| Health `/ok` | 100 req / 10s |

### Gamma API
| Endpoint | Limit |
|----------|-------|
| General | 4,000 req / 10s |
| `/events` | 500 req / 10s |
| `/markets` | 300 req / 10s |
| `/markets` + `/events` listing | 900 req / 10s |
| `/comments` | 200 req / 10s |
| `/tags` | 200 req / 10s |
| `/public-search` | 350 req / 10s |

### Data API
| Endpoint | Limit |
|----------|-------|
| General | 1,000 req / 10s |
| `/trades` | 200 req / 10s |
| `/positions` | 150 req / 10s |
| `/closed-positions` | 150 req / 10s |

### CLOB API
| Category | Endpoint | Limit |
|----------|----------|-------|
| General | — | 9,000 req / 10s |
| Book | `/book` | 1,500 req / 10s |
| Book batch | `/books` | 500 req / 10s |
| Price | `/price` | 1,500 req / 10s |
| Price batch | `/prices` | 500 req / 10s |
| Midpoint | `/midpoint` | 1,500 req / 10s |
| Midpoint batch | `/midpoints` | 500 req / 10s |
| Price history | `/prices-history` | 1,000 req / 10s |
| Tick size | — | 200 req / 10s |
| Ledger | `/trades`, `/orders`, `/order` | 900 req / 10s |
| Auth | API key endpoints | 100 req / 10s |

### CLOB Trading (Burst + Sustained)
| Endpoint | Burst (10s) | Sustained (10min) |
|----------|-------------|-------------------|
| `POST /order` | 3,500 | 36,000 |
| `DELETE /order` | 3,000 | 30,000 |
| `POST /orders` | 1,000 | 15,000 |
| `DELETE /orders` | 1,000 | 15,000 |
| `DELETE /cancel-all` | 250 | 6,000 |
| `DELETE /cancel-market-orders` | 1,000 | 1,500 |

### Other
| Endpoint | Limit |
|----------|-------|
| Relayer `/submit` | 25 req / 1 min |
| User PNL API | 200 req / 10s |

## Error Codes

All errors return `{"error": "<message>"}`.

### Global Errors
| Code | Error | Description |
|------|-------|-------------|
| 401 | `Unauthorized/Invalid api key` | Missing/invalid API key |
| 401 | `Invalid L1 Request headers` | HMAC signature mismatch |
| 429 | `Too Many Requests` | Rate limited — exponential backoff |
| 503 | `Trading is currently disabled` | Exchange paused entirely |
| 503 | `Trading is currently cancel-only` | Can cancel but not place orders |

### Order Errors
| Code | Error | Description |
|------|-------|-------------|
| 400 | `invalid post-only order: order crosses book` | Post-only order would match immediately |
| 400 | `Price breaks minimum tick size rule` | Price doesn't align with tick size |
| 400 | `Size lower than the minimum` | Order too small |
| 400 | `not enough balance / allowance` | Insufficient **pUSD** or token allowance (holding unwrapped USDC counts as zero) |
| 400 | `invalid fee rate (0)` | V1 `feeRateBps` sent on a V2 order — see [migration_to_v2.md](migration_to_v2.md) |
| 400 | `invalid nonce` | Nonce already used |
| 400 | `FOK orders are fully filled or killed` | FOK couldn't be completely filled |
| 400 | `no orders found to match with FAK order` | FAK found zero matches |
| 425 | (Too Early) | Matching engine restarting — retry with backoff |

---

## SDKs & Client Libraries

### Official SDKs

Verified against PyPI and the official docs on **2026-08-06**. There are two
current Python packages and they are not interchangeable.

| Package | What it is | Version | Python | Use when |
|---------|-----------|---------|--------|----------|
| **`polymarket-client`** | Official **unified** SDK ([Polymarket/py-sdk](https://github.com/Polymarket/py-sdk)) — public data, auth, trading, builder attribution, wallet/gasless flows | 0.3.0 (2026-08-04) | ≥3.11 | default; this is what the docs' own quickstart uses |
| **`py-clob-client-v2`** | CLOB **V2 only** client, by Polymarket Engineering | 1.1.0 (2026-07-17) | ≥3.9.10 | you only need the order book and order placement, or you are on Python 3.9/3.10 |
| ~~`py-clob-client`~~ | **ARCHIVED — non-functional** | — | — | never |
| TypeScript | `@polymarket/client` | — | — | TS equivalent of the unified SDK |

> `polymarket-client` is pre-1.0 and its own README warns that **minor releases
> on the 0.x line may include breaking changes**. Pin the version.

The archived repo's own notice: it "has been archived and is no longer
maintained. The client is no longer functional and should not be used for new or
existing integrations." If a snippet anywhere imports `py_clob_client` (no
suffix), it is dead code.

### Builder SDKs (for Builder Program apps)

Note: V2 replaced the `POLY_BUILDER_*` headers with a `builder` field in the
order payload, and the unified SDK exposes it as a `builder_code` argument on
order calls — so a separate signing SDK may no longer be needed.

| Language | Package |
|----------|---------|
| TypeScript | `@polymarket/builder-signing-sdk` |
| Python | `py_builder_signing_sdk` |

### Relayer SDKs (for gasless transactions)

| Language | Package |
|----------|---------|
| TypeScript | `@polymarket/builder-relayer-client` |
| Python | `py-builder-relayer-client` |

### Python Quick Start

**Unified SDK — `pip install polymarket-client`**

Read-only needs no credentials at all:

```python
from polymarket import Market, PublicClient

with PublicClient() as client:
    market: Market = client.get_market(url="https://polymarket.com/event/example-market")
```

Trading. Note `price` and `size` are **strings**, and the client is a context
manager — `AsyncSecureClient` mirrors this with `async with` / `await`:

```python
import os
from polymarket import SecureClient

with SecureClient.create(
    private_key=os.environ["POLYMARKET_PRIVATE_KEY"],
    wallet=os.environ["POLYMARKET_WALLET_ADDRESS"],
) as client:
    response = client.place_limit_order(
        token_id=yes_token_id,
        side="BUY",
        price="0.52",
        size="10",
    )
    if response.ok:
        print(response.order_id, response.status)   # live | matched | delayed | unmatched
    else:
        print(response.code, response.message)
```

**CLOB-only SDK — `pip install py-clob-client-v2`**

Still takes `host` / `chain_id` / `key` positionally-named, unlike the TypeScript
V2 client which moved to an object config:

```python
import os
from py_clob_client_v2 import (
    ClobClient, OrderArgs, OrderType, PartialCreateOrderOptions, Side,
)

client = ClobClient(host=host, chain_id=chain_id, key=os.environ["PK"])
creds = client.create_or_derive_api_key()
client = ClobClient(host=host, chain_id=chain_id, key=os.environ["PK"], creds=creds)

resp = client.create_and_post_order(
    order_args=OrderArgs(
        token_id="",
        price=0.4,
        side=Side.BUY,
        size=100,
    ),
    options=PartialCreateOrderOptions(tick_size="0.01"),
    order_type=OrderType.GTC,
)
```

---

*See also: [WebSocket Channels](polymarket-websocket.md) · [Order Management](polymarket-orders.md) · [Market Structure](polymarket-markets.md)*
