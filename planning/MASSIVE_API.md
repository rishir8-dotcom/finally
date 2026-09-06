# Massive API (formerly Polygon.io) — Reference for FinAlly

Research notes and code examples for retrieving real-time and end-of-day US
equity prices for **multiple tickers** at once. This document is the input to
`MARKET_INTERFACE.md`, which specifies the abstraction FinAlly actually codes
against.

Everything below was verified against `https://massive.com/docs` and the
official client at `github.com/massive-com/client-python` (September 2026).

---

## 1. Background: the rebrand

Polygon.io rebranded to **Massive** in late 2025 / early 2026.

| Before | After | Notes |
|---|---|---|
| `polygon.io` | `massive.com` | Docs, dashboard, pricing all redirect |
| `api.polygon.io` | `api.massive.com` | Old host still resolves and works |
| `pip install polygon-api-client` | `pip install massive` | Old package still published for back-compat |
| `import polygon` | `import massive` | Same classes, same method names |
| `POLYGON_API_KEY` | `MASSIVE_API_KEY` | Env var read by the client constructor |

Existing API keys keep working unchanged. Every URL path (`/v2/...`, `/v3/...`)
is identical to the Polygon era, which means the large body of Polygon
tutorials and StackOverflow answers is still accurate for request/response
shapes — only the hostname and package name moved.

**For FinAlly:** use the `massive` package and the `MASSIVE_API_KEY` env var, as
specified in `PLAN.md` §5.

---

## 2. Authentication

Two equivalent mechanisms:

```http
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT
Host: api.massive.com
Authorization: Bearer <MASSIVE_API_KEY>
```

or as a query parameter (convenient for `curl`, but leaks the key into logs and
proxy access records — avoid in server code):

```
https://api.massive.com/v2/aggs/ticker/AAPL/prev?apiKey=<MASSIVE_API_KEY>
```

The official Python client sends the `Authorization: Bearer` header. Its
constructor reads `MASSIVE_API_KEY` from the environment when `api_key` is not
passed explicitly, and raises if neither is available.

---

## 3. Rate limits and plan tiers

| Plan | Price | Rate limit | Data recency |
|---|---|---|---|
| Basic (free) | $0 | **5 requests/minute** | 15-minute delayed, 2 years history |
| Starter | ~$29/mo | Unlimited | 15-minute delayed, snapshots + WebSockets |
| Developer | ~$79/mo | Unlimited | 15-min delayed, second aggregates, trades |
| Advanced | ~$199/mo | Unlimited | **Real-time**, quotes, 20+ years history |

Pricing changes; treat the table as an order-of-magnitude guide, not a
contract. The number that drives FinAlly's design is the **free tier's 5
req/min**.

### Consequences for polling design

- 5 req/min ⇒ one request per **12 seconds** at the absolute limit. Poll every
  **15 seconds** to leave headroom for retries and clock skew.
- Therefore the poller must fetch **all tracked tickers in a single request**.
  Ten tickers × one request each would blow the free-tier budget in twelve
  seconds. This single constraint is why §4.1 (the multi-ticker snapshot) is
  the primary endpoint and everything else is a fallback.
- Exceeding the limit returns **HTTP 429**. Treat it as transient: log, skip
  the cycle, retry on the next interval. Never tighten the loop on a 429.

---

## 4. Endpoints that matter for FinAlly

### 4.1 Full Market Snapshot — *the primary endpoint*

```
GET /v2/snapshot/locale/us/markets/stocks/tickers
```

| Param | Type | Notes |
|---|---|---|
| `tickers` | string | Comma-separated symbols, e.g. `AAPL,MSFT,TSLA`. **Omit to get every US ticker** (a very large response — always pass it). |
| `include_otc` | bool | Default `false`. Leave it off. |

This is the only stocks endpoint that returns current prices for an arbitrary
set of tickers in one call, which makes it the right fit for a watchlist.

Response (abridged, one entry per requested ticker):

```json
{
  "status": "OK",
  "count": 1,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 0.98,
      "todaysChangePerc": 0.82,
      "updated": 1605195918306274000,
      "day":     { "o": 119.62, "h": 120.53, "l": 118.81, "c": 120.42, "v": 28727868, "vw": 119.725 },
      "prevDay": { "o": 117.19, "h": 119.63, "l": 116.44, "c": 119.49, "v": 110597265 },
      "min":     { "o": 120.435, "h": 120.468, "l": 120.37, "c": 120.4201, "v": 270796, "t": 1684428600000 },
      "lastTrade":  { "p": 120.47, "s": 236, "t": 1605195918306274000, "x": 10, "i": "4046", "c": [14, 41] },
      "lastQuote":  { "P": 120.47, "S": 4, "p": 120.46, "s": 8, "t": 1605195918507251700 }
    }
  ]
}
```

Field decoder for the compact OHLC objects (`day`, `prevDay`, `min`):

| Key | Meaning |
|---|---|
| `o` `h` `l` `c` | open, high, low, close |
| `v` | volume |
| `vw` | volume-weighted average price |
| `t` | timestamp (see §5) |
| `n` | number of transactions |

`lastTrade`: `p` = price, `s` = size, `t` = SIP timestamp, `x` = exchange id,
`i` = trade id, `c` = condition codes.

**Which field is "the current price"?** In priority order:

1. `lastTrade.p` — the most recent print. Best choice when the market is open.
2. `min.c` — close of the current minute bar. A reasonable smoother.
3. `day.c` — today's running close.
4. `prevDay.c` — yesterday's close. The only meaningful value **outside market
   hours**, and the right fallback when `lastTrade` is absent or stale.

A robust extractor walks that list and takes the first non-null value. See §7.

### 4.2 Single Ticker Snapshot

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}
```

Same payload shape as §4.1 but wrapped in a `ticker` object rather than a
`tickers` array. Useful for validating a symbol the user types into the
watchlist ("does AAPL exist and does it price?") — one ticker, one call.

### 4.3 Unified Snapshot (v3)

```
GET /v3/snapshot?ticker.any_of=AAPL,MSFT,TSLA
```

Newer, cross-asset-class (stocks, options, indices, forex, crypto), and returns
a friendlier decoded shape:

```json
{
  "status": "OK",
  "request_id": "abc123",
  "results": [
    {
      "ticker": "AAPL",
      "type": "stocks",
      "name": "Apple Inc.",
      "market_status": "open",
      "last_quote": { "bid": 20.9, "ask": 21.25, "midpoint": 21.075, "last_updated": 1636573458756383500, "timeframe": "REAL-TIME" },
      "last_trade": { "price": 120.47, "size": 2, "exchange": 316, "sip_timestamp": 1675280958783136800, "timeframe": "REAL-TIME" },
      "session":    { "open": 6.7, "close": 6.65, "high": 7.01, "low": 5.42, "volume": 67 }
    }
  ]
}
```

Other params: `ticker`, `type`, `ticker.gt/gte/lt/lte`, `order`, `sort`, `limit`.

Note `market_status` and the `timeframe` markers ("REAL-TIME" vs "DELAYED") —
genuinely useful for the FinAlly header, which could show "delayed" when the
key is on a non-real-time plan. **However**, the v2 endpoint (§4.1) is the one
the official Python client exposes as a first-class multi-ticker helper
(`get_snapshot_all`), it is battle-tested, and it already carries `prevDay` for
after-hours pricing. FinAlly uses v2 and keeps v3 in reserve.

### 4.4 Previous Day Bar — the end-of-day / fallback endpoint

```
GET /v2/aggs/ticker/{stocksTicker}/prev?adjusted=true
```

```json
{
  "status": "OK",
  "ticker": "AAPL",
  "adjusted": true,
  "queryCount": 1,
  "resultsCount": 1,
  "results": [
    { "T": "AAPL", "o": 115.55, "h": 117.59, "l": 114.13, "c": 115.97, "v": 131704427, "vw": 116.3058, "t": 1605042000000 }
  ]
}
```

**One ticker per call.** For a 10-ticker watchlist this is 10 requests — over
the free-tier budget for a single cycle. Use it only as a degraded fallback
(§8), never as the steady-state poll.

### 4.5 Custom Bars (aggregates) — historical backfill

```
GET /v2/aggs/ticker/{stocksTicker}/range/{multiplier}/{timespan}/{from}/{to}
```

e.g. `/v2/aggs/ticker/AAPL/range/5/minute/2026-09-04/2026-09-05?adjusted=true&sort=asc&limit=5000`

`timespan` ∈ `second | minute | hour | day | week | month | quarter | year`.
`from`/`to` accept `YYYY-MM-DD` or millisecond epochs. Response has the same
`results` array of `{o,h,l,c,v,vw,t,n}` bars, plus `next_url` for cursor
pagination.

Not needed for the FinAlly MVP — `PLAN.md` §10 specifies that sparklines and
the detail chart accumulate on the **frontend** from the SSE stream since page
load. This endpoint is the natural upgrade if pre-populated charts are ever
wanted.

### 4.6 Last Trade

```
GET /v2/last/trade/{stocksTicker}
```

Single ticker, real-time plans only. Superseded for our purposes by the
snapshot, which carries `lastTrade` inline for many tickers at once.

### 4.7 WebSockets — explicitly not used

`wss://socket.massive.com/stocks` streams trades/quotes/aggregates live. It
requires a paid plan, adds a second connection lifecycle to manage, and offers
nothing the 15-second REST poll can't for a simulated portfolio.
`PLAN.md` §6 mandates REST polling. Noted here only so the decision is on the
record.

---

## 5. Timestamps — the sharpest edge in this API

Massive uses **three different time units** across the surface, and mixing them
up produces dates in 1970 or in the year 54,000.

| Where | Unit | Example |
|---|---|---|
| `lastTrade.t`, `lastQuote.t`, snapshot `updated` (SIP timestamps) | **nanoseconds** | `1605195918306274000` (19 digits) |
| Aggregate bar `t` (`day`, `prevDay`, `min`, `/aggs` results) | **milliseconds** | `1605042000000` (13 digits) |
| Python `time.time()`, FinAlly's `PriceUpdate.timestamp` | **seconds** (float) | `1605042000.0` (10 digits) |

Convert defensively rather than trusting a hardcoded divisor:

```python
def to_unix_seconds(raw: int | float | None) -> float | None:
    """Normalize a Massive timestamp (ns, µs, or ms) to Unix seconds."""
    if not raw:
        return None
    magnitude = len(str(int(raw)))
    if magnitude >= 19:      # nanoseconds
        return raw / 1_000_000_000
    if magnitude >= 16:      # microseconds
        return raw / 1_000_000
    if magnitude >= 13:      # milliseconds
        return raw / 1_000
    return float(raw)        # already seconds
```

> ⚠️ **Known defect in the current code.** `backend/app/market/massive_client.py`
> reads `snap.last_trade.timestamp / 1000.0`. Two problems: the client's
> `LastTrade` model has no `timestamp` attribute (the field is `sip_timestamp`),
> and even if it did, SIP timestamps are nanoseconds, not milliseconds. The
> `except (AttributeError, TypeError)` around that block silently swallows the
> first error, so **every ticker is skipped on every poll and the cache stays
> empty** when a real API key is configured. `MARKET_INTERFACE.md` §6 specifies
> the corrected extractor. This path is not exercised by the test suite —
> `test_massive.py` mocks the snapshot objects, so the mocks share the bug.

---

## 6. The official Python client

```bash
uv add massive          # FinAlly already has `massive>=1.0.0` in backend/pyproject.toml
```

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()                       # reads MASSIVE_API_KEY from env
client = RESTClient(api_key="...")          # or pass explicitly
```

Useful constructor options: `trace=True` / `verbose=True` (log request URLs and
headers), `raw=True` (return raw `urllib3` responses instead of parsed models),
`custom_json=orjson` (swap the JSON parser).

### Conventions

- `get_*` returns one record; `list_*` returns an iterator that auto-paginates.
- `limit` is the **page size**, not a total cap. Pass `pagination=False` for a
  single page.
- **The client is synchronous.** It is built on `urllib3`, has no async
  variant, and will block the event loop if called directly from a coroutine.
  Always wrap calls in `asyncio.to_thread(...)` inside FastAPI.

### Calls relevant to FinAlly

```python
# Multi-ticker snapshot — the primary poll (one HTTP request)
snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT"],
)   # -> list[TickerSnapshot]

# Single ticker (symbol validation)
snap = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)   # -> TickerSnapshot

# Previous close (fallback / EOD)
prev = client.get_previous_close(ticker="AAPL")

# Historical bars
bars = list(client.list_aggs(
    ticker="AAPL", multiplier=5, timespan="minute",
    from_="2026-09-04", to="2026-09-05",
))
```

Note `get_snapshot_all` and `get_snapshot_ticker` both take `market_type` as
their **first positional argument** — a common source of `TypeError` when it is
omitted.

### Parsed model shapes

`TickerSnapshot` (from `massive.rest.models.snapshot`):

| Attribute | Type | From JSON |
|---|---|---|
| `ticker` | `str \| None` | `ticker` |
| `day` | `Agg \| None` | `day` |
| `prev_day` | `Agg \| None` | `prevDay` |
| `min` | `MinuteSnapshot \| None` | `min` |
| `last_trade` | `LastTrade \| None` | `lastTrade` |
| `last_quote` | `LastQuote \| None` | `lastQuote` |
| `todays_change` | `float \| None` | `todaysChange` |
| `todays_change_percent` | `float \| None` | `todaysChangePerc` |
| `updated` | `int \| None` (ns) | `updated` |
| `fair_market_value` | `float \| None` | `fmv` |

`LastTrade` — note the field names, which differ from the JSON keys:

| Attribute | From JSON key |
|---|---|
| `price` | `p` |
| `size` | `s` |
| `sip_timestamp` | `t` ← **nanoseconds; there is no `timestamp` attribute** |
| `participant_timestamp` | `y` |
| `trf_timestamp` | `f` |
| `exchange` | `x` |
| `conditions` | `c` |
| `id` | `i` |

`Agg` (used for `day` / `prev_day`): `open`, `high`, `low`, `close`, `volume`,
`vwap`, `timestamp` (ms), `transactions`.

`MinuteSnapshot` (used for `min`): `open`, `high`, `low`, `close`, `volume`,
`vwap`, `timestamp` (ms), `accumulated_volume`.

Every field is `Optional`. A snapshot for a ticker that has not traded today —
or a bad symbol, or a symbol outside your plan's entitlement — comes back with
`last_trade=None`. Parsing code must handle that without raising.

---

## 7. Worked example: polling many tickers safely

```python
import asyncio
import logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

logger = logging.getLogger(__name__)

PRICE_SOURCES = ("last_trade", "min", "day", "prev_day")


def to_unix_seconds(raw):
    if not raw:
        return None
    d = len(str(int(raw)))
    return raw / (1e9 if d >= 19 else 1e6 if d >= 16 else 1e3 if d >= 13 else 1)


def extract_price(snap) -> tuple[float, float] | None:
    """Return (price, unix_seconds) from a TickerSnapshot, or None."""
    if (lt := getattr(snap, "last_trade", None)) and lt.price:
        return lt.price, to_unix_seconds(lt.sip_timestamp)
    for attr in ("min", "day", "prev_day"):
        bar = getattr(snap, attr, None)
        if bar and bar.close:
            return bar.close, to_unix_seconds(getattr(bar, "timestamp", None))
    return None


def fetch(client: RESTClient, tickers: list[str]) -> dict[str, tuple[float, float]]:
    """One HTTP request for all tickers. Synchronous — call via to_thread."""
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
    out = {}
    for snap in snapshots:
        if snap.ticker and (parsed := extract_price(snap)):
            out[snap.ticker] = parsed
    return out


async def poll_forever(tickers: list[str], interval: float = 15.0):
    client = RESTClient()  # MASSIVE_API_KEY from env
    while True:
        try:
            prices = await asyncio.to_thread(fetch, client, tickers)
            logger.info("polled %d/%d tickers", len(prices), len(tickers))
        except Exception:
            logger.exception("poll failed; retrying next cycle")
        await asyncio.sleep(interval)
```

Equivalent raw HTTP, no client library:

```python
import httpx, os

async def fetch_raw(tickers: list[str]) -> dict:
    async with httpx.AsyncClient(timeout=10.0) as http:
        r = await http.get(
            "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers",
            params={"tickers": ",".join(tickers)},
            headers={"Authorization": f"Bearer {os.environ['MASSIVE_API_KEY']}"},
        )
        r.raise_for_status()
        return {
            t["ticker"]: t["lastTrade"]["p"]
            for t in r.json().get("tickers", [])
            if t.get("lastTrade")
        }
```

---

## 8. Error handling

| Status / condition | Meaning | Response |
|---|---|---|
| `401` | Missing or invalid API key | Log once, loudly. Do not retry in a tight loop — the key won't fix itself. Consider falling back to the simulator. |
| `403` `NOT_AUTHORIZED` | Key valid, endpoint not in plan | Fall back to `/v2/aggs/ticker/{t}/prev` per ticker (accepting the request cost), or to the simulator. |
| `429` | Rate limit exceeded | Skip the cycle; the next interval retries. Optionally back off to 30s for a few cycles. |
| `5xx` / timeout / connection reset | Upstream trouble | Log, skip, retry next interval. |
| `status: "DELAYED"` in body | 200 OK, but 15-min delayed data | Fine for FinAlly. Surface as a "delayed" badge if desired. |
| Empty `tickers` array | Unknown symbol, or no data yet | Log at debug. Keep the last cached price; don't zero it out. |
| `last_trade` is `None` | Market closed, or symbol hasn't traded | Fall through to `min` → `day` → `prev_day` (§7). |

The governing rule for FinAlly: **a market data failure must never take down the
app.** The poller catches broadly, logs, and lets the previous cached prices
stand. Stale prices beat a blank terminal.

---

## 9. Summary of decisions for FinAlly

| Decision | Choice | Why |
|---|---|---|
| Package | `massive` (not `polygon-api-client`) | Current name; `PLAN.md` env var already matches |
| Transport | REST polling | `PLAN.md` §6; WebSockets need a paid plan |
| Primary endpoint | `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=…` | Only multi-ticker price endpoint; one request per cycle |
| Client call | `get_snapshot_all(SnapshotMarketType.STOCKS, tickers)` | Direct wrapper for the above |
| Poll interval | 15s default, env-overridable | Fits the free tier's 5 req/min with headroom |
| Blocking | `asyncio.to_thread` | The client is synchronous |
| Price field | `last_trade.price` → `min.close` → `day.close` → `prev_day.close` | Works during and outside market hours |
| Timestamps | Magnitude-based normalization to Unix seconds | Three different units in one payload |
| Failure mode | Log and keep last cached prices | Never break the UI over a data hiccup |

---

## Sources

- [Massive API Docs](https://massive.com/docs)
- [Stocks REST API Overview](https://massive.com/docs/rest/stocks/overview)
- [Full Market Snapshot](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Single Ticker Snapshot](https://massive.com/docs/rest/stocks/snapshots/single-ticker-snapshot)
- [Unified Snapshot (v3)](https://massive.com/docs/rest/stocks/snapshots/unified-snapshot)
- [Previous Day Bar](https://massive.com/docs/rest/stocks/aggregates/previous-day-bar)
- [Custom Bars](https://massive.com/docs/rest/stocks/aggregates/custom-bars)
- [massive-com/client-python](https://github.com/massive-com/client-python)
- [Client getting-started (DeepWiki)](https://deepwiki.com/massive-com/client-python/2-getting-started)
- [Polygon.io / Massive pricing overview](https://apicostcalc.com/polygon.html)
