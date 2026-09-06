# Market Data Interface — Unified Python API

The single abstraction every part of FinAlly uses to get a stock price. One
interface, two implementations: the **Massive REST poller** when
`MASSIVE_API_KEY` is set, the **GBM simulator** otherwise. Nothing downstream —
SSE, portfolio valuation, trade execution, the LLM's context builder — knows or
cares which is running.

Companion documents: `MASSIVE_API.md` (the upstream API this wraps),
`MARKET_SIMULATOR.md` (how the simulator generates prices).

Implementation lives in `backend/app/market/`.

---

## 1. The shape of it

```
                 create_market_data_source(cache)
                              │
              MASSIVE_API_KEY set?  ──yes──►  MassiveDataSource
                              │                (REST poll, 15s)
                              no
                              │
                              ▼
                      SimulatorDataSource
                       (GBM tick, 500ms)
                              │
                              │  both write only here
                              ▼
                    ┌───────────────────┐
                    │    PriceCache     │  thread-safe, in-memory
                    │  ticker → Price   │  + monotonic version counter
                    │       Update      │
                    └─────────┬─────────┘
                              │  everyone reads from here
              ┌───────────────┼───────────────┬──────────────┐
              ▼               ▼               ▼              ▼
        SSE endpoint    Portfolio      Trade execution   LLM context
      /api/stream/prices  valuation     (fill price)      builder
```

The central rule: **producers write to the cache, consumers read from the
cache, and the two never talk to each other.** Consumers never `await` a data
source. A price read is a synchronous dictionary lookup that cannot fail, block,
or raise — which is what makes valuing a portfolio on every request cheap and
what makes swapping the data source a one-line change.

---

## 2. Public API

```python
from app.market import (
    PriceUpdate,                # immutable price record
    PriceCache,                 # thread-safe store
    MarketDataSource,           # the ABC
    create_market_data_source,  # factory
    create_stream_router,       # FastAPI SSE router
)
```

That is the entire surface. `SimulatorDataSource`, `MassiveDataSource` and
`GBMSimulator` are deliberately *not* exported from the package root: app code
should never name a concrete implementation.

---

## 3. `PriceUpdate` — the currency of the system

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)   # Unix seconds

    @property
    def change(self) -> float: ...           # price - previous_price, 4dp
    @property
    def change_percent(self) -> float: ...   # % vs previous_price, 4dp, 0.0 if prev == 0
    @property
    def direction(self) -> str: ...          # "up" | "down" | "flat"

    def to_dict(self) -> dict: ...           # all of the above, JSON-ready
```

Design notes:

- **Frozen.** A price is a fact about a moment; it is never mutated. Cache
  updates replace the object rather than editing it, which is what lets readers
  hold a reference without a lock.
- **`slots=True`.** Tens of thousands of these are created per hour; slots keep
  each one small.
- **`previous_price` is the previous *tick*, not the previous *day*.** This is
  what drives the green/red flash animation in `PLAN.md` §2. It is emphatically
  not the day's change percentage shown in the watchlist column — see §8.
- **`timestamp` is always Unix seconds (float).** Massive's nanosecond and
  millisecond timestamps are normalized at the boundary (§6). No other unit
  ever enters the system.

---

## 4. `PriceCache` — the single point of truth

```python
class PriceCache:
    def update(self, ticker: str, price: float,
               timestamp: float | None = None) -> PriceUpdate: ...
    def get(self, ticker: str) -> PriceUpdate | None: ...
    def get_price(self, ticker: str) -> float | None: ...
    def get_all(self) -> dict[str, PriceUpdate]: ...
    def remove(self, ticker: str) -> None: ...

    @property
    def version(self) -> int: ...            # bumped on every update()

    def __len__(self) -> int: ...
    def __contains__(self, ticker: str) -> bool: ...
```

**Concurrency.** A `threading.Lock` guards every access. The simulator runs on
the event loop; the Massive poller's HTTP call runs in a worker thread via
`asyncio.to_thread`. A `threading.Lock` (not an `asyncio.Lock`) is the type
that correctly covers both. Critical sections are a handful of dict operations,
so contention is not a concern — but `get_all()` returns a shallow copy so that
callers can iterate without holding anything.

**`update()` computes the delta for you.** It looks up the prior entry, carries
its price into `previous_price`, and rounds both to 2dp. On the first update for
a ticker, `previous_price == price`, so `direction` is `"flat"` — no spurious
flash when a ticker first appears.

**The version counter** is the SSE change-detection mechanism. It increments on
every write. The stream generator compares `cache.version` to the value it last
sent and skips the emit entirely if nothing moved. That keeps an idle stream
silent instead of pushing an identical 2 KB payload twice a second.

**`remove()` is how a ticker leaves.** Called by `MarketDataSource.remove_ticker`.
Note the interaction with `PLAN.md` §6: the tracked set is
`watchlist ∪ open positions`, so removing a ticker from the *watchlist* must
only call `remove_ticker` if no position remains open in it. That check belongs
to the watchlist route, not to this layer.

---

## 5. `MarketDataSource` — the abstract contract

```python
class MarketDataSource(ABC):
    @abstractmethod
    async def start(self, tickers: list[str]) -> None: ...
    @abstractmethod
    async def stop(self) -> None: ...
    @abstractmethod
    async def add_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    def get_tickers(self) -> list[str]: ...
```

### Contract, in detail

| Method | Obligations on the implementation |
|---|---|
| `start(tickers)` | Populate the cache with an initial price for every ticker **before returning**, then launch a background task. Called exactly once. Callers may assume the cache is non-empty afterwards, so the first SSE frame is never blank. |
| `stop()` | Cancel the background task and await its `CancelledError`. Idempotent — safe to call twice, or without a prior `start()`. Must not write to the cache afterwards. |
| `add_ticker(t)` | Idempotent. Normalize to upper-case and strip whitespace. Seed a price if the implementation can do so cheaply. |
| `remove_ticker(t)` | Idempotent. Must also call `cache.remove(t)` so a stale price doesn't linger. |
| `get_tickers()` | Synchronous (it's a plain read). Returns a copy. |

`add_ticker` / `remove_ticker` are `async` even though neither implementation
currently awaits anything. That is intentional: a future source that must
subscribe or unsubscribe over a socket needs the async signature, and changing
it later would ripple through every call site.

### Latency asymmetry, stated plainly

`add_ticker` on the simulator produces a price immediately. On Massive it
produces one **up to 15 seconds later**, on the next poll. Any UI or API that
adds a ticker must tolerate a brief `None` price. `GET /api/watchlist` should
return `price: null` rather than `0.0` for a ticker with no cached price —
`0.0` renders as a real price and looks like a crash.

---

## 6. `MassiveDataSource` — the real-data implementation

```python
MassiveDataSource(
    api_key: str,
    price_cache: PriceCache,
    poll_interval: float = 15.0,
)
```

**The loop.** `start()` constructs a `RESTClient`, performs one immediate poll
so the cache is warm, then schedules `_poll_loop()`, which sleeps `interval` and
polls, forever.

**One request per cycle, always.** `client.get_snapshot_all(SnapshotMarketType.STOCKS,
tickers=self._tickers)` fetches every tracked ticker in a single HTTP call.
This is not an optimization; it is the design constraint. The free tier allows
5 requests/minute, so per-ticker calls are impossible. Ten tickers and fifty
tickers cost exactly the same: one request.

**Never block the event loop.** The `massive` client is synchronous urllib3.
The call goes through `asyncio.to_thread`. A 2-second API stall would otherwise
freeze every SSE stream and every HTTP request in the process.

**Price extraction** walks a fallback chain, because `last_trade` is `None`
whenever the market is closed:

```python
last_trade.price  →  min.close  →  day.close  →  prev_day.close
```

**Timestamp normalization** converts by magnitude — SIP timestamps are
nanoseconds, aggregate bar timestamps are milliseconds (`MASSIVE_API.md` §5):

```python
def _to_unix_seconds(raw) -> float | None:
    if not raw:
        return None
    d = len(str(int(raw)))
    return raw / (1e9 if d >= 19 else 1e6 if d >= 16 else 1e3 if d >= 13 else 1)
```

**Errors never propagate.** `_poll_once` wraps everything in `try/except
Exception`, logs, and returns. The loop retries on the next interval. A 401, a
429, a DNS failure and a malformed payload all degrade to "prices stop updating
for a bit" rather than a crashed background task and a dead terminal.

> **Required fixes before this path is trusted.** The current
> `massive_client.py` reads `snap.last_trade.price` and
> `snap.last_trade.timestamp / 1000.0` with no fallback chain. `LastTrade` has
> no `timestamp` attribute (it is `sip_timestamp`, in nanoseconds), and the
> surrounding `except (AttributeError, TypeError): continue` swallows the
> resulting `AttributeError` per ticker — so with a real API key the cache never
> populates. Three changes are needed:
> 1. Use `sip_timestamp` and normalize by magnitude, not `/1000.0`.
> 2. Add the `min`/`day`/`prev_day` fallback chain so closed-market hours work.
> 3. Make `tests/market/test_massive.py` mocks mirror the real `TickerSnapshot`
>    and `LastTrade` field names, so this class of bug fails a test instead of
>    passing one.

**Poll interval should be configurable.** `MASSIVE_POLL_INTERVAL` (seconds,
default 15) lets a paid-tier user drop to 2–5s without a code change. The
factory reads it.

---

## 7. `SimulatorDataSource` — the default implementation

```python
SimulatorDataSource(
    price_cache: PriceCache,
    update_interval: float = 0.5,
    event_probability: float = 0.001,
)
```

Wraps `GBMSimulator` (see `MARKET_SIMULATOR.md`) in the same lifecycle. `start()`
builds the simulator, seeds the cache with each ticker's opening price, and
launches a loop that calls `step()` every 500 ms and writes the results.

Two properties worth naming because downstream code depends on them:

- **Every tick moves every ticker.** The simulator has no concept of a closed
  market or an untraded symbol, so the cache is always fully populated.
- **`add_ticker` is instantaneous.** A newly added symbol has a price before the
  method returns, unlike the Massive path.

The exception handler in the run loop is inside the `while`, so a single bad
step logs and continues rather than killing the loop.

---

## 8. What this interface deliberately does not provide

Naming these keeps the boundary honest — each is someone else's job:

| Not provided | Who owns it |
|---|---|
| Historical bars / price history | Frontend, accumulated from SSE since page load (`PLAN.md` §10). Backfill would use `/v2/aggs` (`MASSIVE_API.md` §4.5). |
| Daily change % (vs. previous close) | Not the same as `PriceUpdate.change_percent`, which is tick-over-tick. If the watchlist needs a true daily figure, the source must expose `prev_day.close` as a separate `session_open` field. **Open item — decide before building the watchlist column.** |
| Symbol validation | Watchlist route. Massive: `get_snapshot_ticker`. Simulator: accepts anything and invents a seed price. |
| Persistence | Nothing here is durable. The cache is in-memory and dies with the process; SQLite holds positions, trades and snapshots. |
| Multi-user isolation | One cache, one source, process-wide. Correct for FinAlly's single-user design and unchanged by future auth — prices aren't per-user. |

---

## 9. Wiring it into FastAPI

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.market import PriceCache, create_market_data_source, create_stream_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    cache = PriceCache()
    source = create_market_data_source(cache)

    # tracked set = watchlist ∪ tickers with an open position  (PLAN.md §6)
    tickers = sorted(set(db.get_watchlist_tickers()) | set(db.get_position_tickers()))
    await source.start(tickers)

    app.state.price_cache = cache
    app.state.market_source = source
    try:
        yield
    finally:
        await source.stop()


app = FastAPI(lifespan=lifespan)
app.include_router(create_stream_router(app.state.price_cache))
```

Reading a price anywhere else:

```python
price = request.app.state.price_cache.get_price("AAPL")
if price is None:
    raise HTTPException(503, "No price available for AAPL yet")
```

Keeping the tracked set in sync:

```python
# POST /api/watchlist
await request.app.state.market_source.add_ticker(ticker)

# DELETE /api/watchlist/{ticker}  — only if no position remains
if not db.has_position(ticker):
    await request.app.state.market_source.remove_ticker(ticker)

# after a buy that opens a new position in an unwatched ticker
await request.app.state.market_source.add_ticker(ticker)
```

---

## 10. `create_stream_router` — SSE

`GET /api/stream/prices`, `text/event-stream`. The generator:

1. Emits `retry: 1000` so `EventSource` reconnects after 1 s.
2. Every 500 ms, compares `cache.version` against the last value sent; emits
   only if it changed.
3. Sends **all** tracked prices in one JSON object keyed by ticker, rather than
   one event per ticker — fewer frames, and the client applies a consistent
   snapshot.
4. Breaks on `request.is_disconnected()`, and handles `CancelledError` at
   shutdown.

```
retry: 1000

data: {"AAPL": {"ticker":"AAPL","price":190.5,"previous_price":190.4,
                "timestamp":1757116800.5,"change":0.1,"change_percent":0.0526,
                "direction":"up"}, "GOOGL": {...}}
```

Headers set: `Cache-Control: no-cache`, `Connection: keep-alive`,
`X-Accel-Buffering: no` (defeats nginx response buffering, which otherwise holds
SSE frames until a buffer fills).

**Cadence mismatch with Massive is expected and fine.** The stream ticks at
500 ms regardless of source; with a 15-second poll the version simply doesn't
change for ~30 of those checks, so nothing is sent. The client sees smooth
updates from the simulator and stepped updates from Massive, with no special
handling on either side.

---

## 11. Testing the abstraction

**Conformance.** Both implementations should be exercised against the same
parametrized suite proving the §5 contract: `start` populates the cache;
`stop` is idempotent and halts writes; `add`/`remove` are idempotent;
`remove_ticker` clears the cache entry; `get_tickers` returns a copy.

**Massive without the network.** Mock `RESTClient.get_snapshot_all`. The mocks
must use the real model field names (`last_trade.price`, `last_trade.sip_timestamp`,
`prev_day.close`) — mocks that mirror the implementation's mistakes test
nothing, which is exactly how the §6 defect survived a 73-test suite. Cover:
nanosecond conversion, the `last_trade=None` fallback path, 429 handling
(cache unchanged, loop alive), and a malformed snapshot (skipped, others still
processed).

**Determinism.** Seed `numpy.random` and `random` for simulator tests.

**No real API key in CI.** The factory selects the simulator when
`MASSIVE_API_KEY` is unset, which is also what makes `LLM_MOCK=true` E2E runs
hermetic.

---

## 12. Adding a third source

The exercise the abstraction exists for. To add, say, an Alpaca or Finnhub
source:

1. Implement `MarketDataSource` in `app/market/<name>_client.py`.
2. Add a branch to `create_market_data_source()`.
3. Add it to the conformance suite from §11.

No other file changes. If a new source ever requires a change outside those
three places, the interface has leaked and should be fixed rather than worked
around.
