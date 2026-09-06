# Market Simulator — Approach and Code Structure

The default market data source. Runs with zero configuration, no API key, no
network — and has to look convincing on screen for the length of a demo.

Implements `MarketDataSource` (see `MARKET_INTERFACE.md` §5), so it is
interchangeable with the Massive REST poller. Code lives in
`backend/app/market/simulator.py` and `backend/app/market/seed_prices.py`.

---

## 1. What "realistic" has to mean here

The simulator is judged by eye, not by a backtest. The demo requirements set
the bar:

1. **Prices move constantly** — every 500 ms tick should change the last two
   decimal places on most tickers, so the flash animations fire.
2. **Moves are small** — a stock does not jump 3% twice a second. Sub-cent
   drift per tick that accumulates into a plausible intraday path.
3. **Prices stay positive** — a $190 stock never goes to $0 or negative, no
   matter how long the container runs.
4. **Volatile stocks look volatile** — TSLA should visibly bounce more than V.
5. **Related stocks move together** — a green tech tape reads as a market with
   a mood, not ten independent random walks. This is the single detail that
   most makes the terminal look real.
6. **Occasional drama** — a sudden 2–5% move now and then, so the watchlist
   isn't monotone.

Geometric Brownian Motion delivers 1–4 directly, a correlation structure
delivers 5, and a shock injection delivers 6.

---

## 2. The model: Geometric Brownian Motion

The standard model for equity prices, the same one under Black-Scholes. Its
discrete-time exact solution:

```
S(t+dt) = S(t) · exp( (μ − σ²/2)·dt  +  σ·√dt·Z )
```

| Symbol | Meaning | Typical value |
|---|---|---|
| `S(t)` | Current price | 190.00 |
| `μ` | Annualized drift (expected return) | 0.05 (5%/yr) |
| `σ` | Annualized volatility | 0.22 (22%/yr) |
| `dt` | Time step, as a fraction of a trading year | ~8.48e-8 |
| `Z` | Standard normal draw, `N(0,1)` | −0.4 |

Three properties earn GBM its place:

- **Multiplicative, so prices can't go negative.** `exp(...)` is always
  positive, so `S` is always positive. Requirement 3, for free — no clamping,
  no `max(price, 0.01)` hack.
- **Returns scale with price.** A $1 move in NVDA at $800 and a $1 move in JPM
  at $195 are not comparable events, and GBM knows that. Percentage moves are
  what σ controls.
- **The `−σ²/2` term is not decoration.** It's the Itô correction. Without it
  the *median* path drifts upward relative to `μ`, because `E[exp(X)] >
  exp(E[X])` for a random `X`. With it, `μ` means what it says.

### Choosing `dt`

`μ` and `σ` are annualized, so `dt` must be expressed as a fraction of a
trading year — of *market* time, not wall-clock time:

```
TRADING_SECONDS_PER_YEAR = 252 days × 6.5 hours × 3600 s = 5,896,800
DEFAULT_DT               = 0.5 / 5,896,800 ≈ 8.48e-8
```

Sanity check for AAPL at $190 with σ=0.22:

```
σ·√dt = 0.22 × √8.48e-8 ≈ 6.4e-5      ⇒  one standard deviation ≈ 0.0064% of price
190 × 6.4e-5 ≈ $0.012 per tick        ⇒  about one cent
```

Right in the sweet spot: it lands in the second decimal place, so the price
visibly ticks, but it takes many ticks to make a real move. Over an hour
(7,200 ticks) the standard deviation of the accumulated path is
`0.012 × √7200 ≈ $1.03` — about 0.5%, which is what an hour of AAPL actually
looks like.

Note the deliberate **simulated-time = wall-clock-time** choice: the app runs
for a real hour and produces an hour of price action. Compressing time
(1 real second = 1 simulated minute) would produce more dramatic charts but
would make a 20-minute demo cover three trading days, which reads as wrong.

---

## 3. Correlation via Cholesky decomposition

Independent draws per ticker produce a watchlist where half the tickers are
green and half red at all times — statistically fine, visually dead. Real
sectors move together.

The mechanism: draw `n` independent standard normals `Z_ind`, then multiply by
`L`, the lower-triangular Cholesky factor of the desired correlation matrix `C`
(where `C = L·Lᵀ`):

```
Z_corr = L · Z_ind
```

`Z_corr` is still a vector of standard normals, but now with pairwise
correlations exactly `C`. Each ticker's GBM step consumes its own component of
`Z_corr`.

### The correlation structure

Built from sector membership rather than estimated from data — a lookup table,
not a covariance estimate:

| Pair | ρ | Rationale |
|---|---|---|
| Tech ∧ tech (AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX) | 0.6 | Move on the same macro news |
| Finance ∧ finance (JPM, V) | 0.5 | Rate-sensitive together |
| Anything involving TSLA | 0.3 | Nominally tech; trades on its own narrative |
| Cross-sector, or any unknown ticker | 0.3 | Broad market beta |

ρ = 0.6 is high enough that the tech block visibly moves as a bloc, low enough
that individual names still diverge over a few minutes.

### Why the matrix is always decomposable

`numpy.linalg.cholesky` raises `LinAlgError` on a matrix that isn't positive
definite, and an arbitrary table of pairwise correlations is not guaranteed to
be. This one is safe by construction: it's block-structured with a uniform
0.3 floor and intra-block values below 1, which keeps it positive definite for
any ticker set. Anyone editing the constants must respect that — pushing
`INTRA_TECH_CORR` toward 1.0 while adding more groups is the way to break it.
A defensive `try/except LinAlgError` falling back to `self._cholesky = None`
(independent moves) would be cheap insurance and is worth adding.

Cost is `O(n³)` per rebuild, but rebuilds happen only on add/remove ticker, and
`n < 50`. Irrelevant.

---

## 4. Random shock events

Requirement 6. Each ticker, each tick, independently:

```python
if random.random() < event_probability:        # 0.001
    magnitude = random.uniform(0.02, 0.05)     # 2–5%
    sign      = random.choice([-1, 1])
    price    *= 1 + magnitude * sign
```

Expected frequency with 10 tickers at 2 ticks/second:

```
10 × 2 × 0.001 = 0.02 events/second  ⇒  one event roughly every 50 seconds
```

Frequent enough that a demo sees several, rare enough that they read as events
rather than noise. The shock bypasses the correlation structure entirely — it's
idiosyncratic single-name news, so it *should* be uncorrelated.

Because shocks are multiplicative, positivity still holds.

---

## 5. Seed data

`seed_prices.py` holds three tables, kept apart from the simulation logic so
tuning the look of the demo never means touching the math.

```python
SEED_PRICES = {"AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00,
               "TSLA": 250.00, "NVDA": 800.00, "META": 500.00, "JPM": 195.00,
               "V": 280.00, "NFLX": 600.00}

TICKER_PARAMS = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},   # high vol, low drift
    "NVDA": {"sigma": 0.40, "mu": 0.08},   # high vol, strong drift
    "JPM":  {"sigma": 0.18, "mu": 0.04},   # low vol
    "V":    {"sigma": 0.17, "mu": 0.04},   # low vol
    ...
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}

CORRELATION_GROUPS = {"tech": {...}, "finance": {"JPM", "V"}}
INTRA_TECH_CORR, INTRA_FINANCE_CORR, CROSS_GROUP_CORR, TSLA_CORR = 0.6, 0.5, 0.3, 0.3
```

Prices are a plausible snapshot rather than a live quote; nothing in the app
compares them to reality. The **spread** across tickers matters more than the
levels — $175 to $800 exercises the UI's number formatting.

σ values are the differentiator: TSLA at 0.50 moves ~3× as much per tick as V
at 0.17, which is visible in the watchlist within seconds. That contrast is
what sells the simulation.

An unknown ticker (user adds "PYPL") gets a random seed price in
`[50, 300]` and `DEFAULT_PARAMS`, and correlates at 0.3 with everything. The
simulator never rejects a symbol — deliberately, since there is no symbol
universe to validate against.

---

## 6. Code structure

Two classes, split along one line: **pure math** vs **async plumbing**.

### `GBMSimulator` — the model. No asyncio, no cache, no I/O.

```python
class GBMSimulator:
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR

    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001) -> None: ...

    def step(self) -> dict[str, float]:   # advance one tick → {ticker: price}
    def add_ticker(self, ticker: str) -> None
    def remove_ticker(self, ticker: str) -> None
    def get_price(self, ticker: str) -> float | None
    def get_tickers(self) -> list[str]

    # internals
    def _add_ticker_internal(self, ticker: str) -> None   # no Cholesky rebuild
    def _rebuild_cholesky(self) -> None
    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float
```

State: `_tickers` (ordered — the order defines the matrix rows), `_prices`,
`_params`, `_cholesky`.

Because it is pure and synchronous, it is trivially testable: seed the RNG,
step 10,000 times, assert the empirical volatility matches σ within tolerance.
That test would be impossible if the math were tangled with the event loop.

`_add_ticker_internal` exists so `__init__` can add ten tickers and rebuild the
Cholesky factor once instead of ten times.

### `SimulatorDataSource` — the adapter. Implements `MarketDataSource`.

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache,
                 update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None: ...

    async def start(self, tickers) -> None    # build sim, seed cache, launch task
    async def stop(self) -> None              # cancel + await CancelledError
    async def add_ticker(self, t) -> None     # add + seed cache immediately
    async def remove_ticker(self, t) -> None  # remove from sim and cache
    def get_tickers(self) -> list[str]

    async def _run_loop(self) -> None
```

The loop:

```python
while True:
    try:
        prices = self._sim.step()
        for ticker, price in prices.items():
            self._cache.update(ticker=ticker, price=price)
    except Exception:
        logger.exception("Simulator step failed")
    await asyncio.sleep(self._interval)
```

The `try` is **inside** the `while`, so a single failed step logs and the loop
survives. The reverse arrangement would silently kill price updates for the
lifetime of the process — the exact failure that is hardest to notice in a
demo.

`start()` seeds the cache before returning, so the first SSE frame after
startup already carries all ten tickers.

### `step()` in full

```python
def step(self) -> dict[str, float]:
    n = len(self._tickers)
    if n == 0:
        return {}

    z = np.random.standard_normal(n)
    if self._cholesky is not None:
        z = self._cholesky @ z                  # correlate

    result = {}
    for i, ticker in enumerate(self._tickers):
        p = self._params[ticker]
        mu, sigma = p["mu"], p["sigma"]

        drift     = (mu - 0.5 * sigma**2) * self._dt
        diffusion = sigma * math.sqrt(self._dt) * z[i]
        self._prices[ticker] *= math.exp(drift + diffusion)

        if random.random() < self._event_prob:
            self._prices[ticker] *= 1 + random.uniform(0.02, 0.05) * random.choice([-1, 1])

        result[ticker] = round(self._prices[ticker], 2)
    return result
```

Note that `_prices` holds **full precision** and only the returned value is
rounded to 2dp. Rounding the state itself would quantize every step to whole
cents and, at these tick sizes, destroy the drift entirely.

---

## 7. Performance

Per tick, with n tickers: one `numpy` normal draw (vectorized), one `n×n`
matrix-vector product, and a Python loop of n `exp` calls. At n=10 that is
tens of microseconds against a 500 ms budget — four orders of magnitude of
headroom. There is no reason to vectorize the inner loop; the explicit version
is far easier to read and to unit-test, and it stays cheap well past n=100.

The `PriceCache` writes are n lock acquisitions per tick. Uncontended locks
cost ~50 ns. Also negligible.

---

## 8. Testing

| Property | Test |
|---|---|
| Positivity | 100k steps, `assert all(p > 0)` |
| No NaN/inf | `assert math.isfinite(p)` throughout |
| Volatility calibration | Seeded RNG, N steps, empirical σ of log-returns ≈ configured σ within tolerance |
| Drift correction | Long run; median log-price grows at ≈ `(μ − σ²/2)·t` |
| Correlation | Two tech tickers over many steps: sample correlation of log-returns ≈ 0.6 ± tolerance |
| Cholesky validity | `_rebuild_cholesky` succeeds for 1, 2, 10, 50 tickers and every sector mix |
| Single ticker | n=1 ⇒ `_cholesky is None`, uncorrelated path, no crash |
| Empty | n=0 ⇒ `step()` returns `{}` |
| Unknown ticker | Gets a seed price in [50,300] and `DEFAULT_PARAMS` |
| Add/remove | Matrix rebuilds; removed ticker absent from `step()` output and cache |
| Shock frequency | With `event_probability=1.0`, every tick moves 2–5% |
| Lifecycle | `start` seeds cache; `stop` is idempotent and halts writes |

Seed both `np.random` and `random` — the simulator uses both.

Statistical assertions need loose tolerances and fixed seeds. A ±10% band on an
estimate from 10k samples is honest; anything tighter is a flaky test waiting
for CI.

---

## 9. Tuning guide

Everything worth adjusting, and what it does:

| Knob | Location | Effect |
|---|---|---|
| `update_interval` | `SimulatorDataSource` | Tick rate. 0.5s matches the SSE cadence; lowering it also lowers per-tick move size via `dt`, so total volatility is unchanged. |
| `dt` | `GBMSimulator` | Simulated time per tick. **Raise this to compress time** — the one lever that makes charts more dramatic without touching σ. |
| `sigma` | `TICKER_PARAMS` | Per-ticker choppiness. The main visual dial. |
| `mu` | `TICKER_PARAMS` | Long-run direction. Barely visible in a demo-length session. |
| `event_probability` | `SimulatorDataSource` | Shock frequency. 0.001 ⇒ ~1 per 50s across 10 tickers. |
| Correlation constants | `seed_prices.py` | How bloc-like the tape looks. |
| `SEED_PRICES` | `seed_prices.py` | Starting levels. |

**If the demo looks too flat**, raise `dt` (compress time) before raising σ.
Raising σ makes prices jitter unrealistically tick-to-tick; raising `dt` makes
the *path* move faster while each individual tick still looks right.

---

## 10. Deliberate omissions

Simplifications made on purpose. Each is a real feature of markets that FinAlly
does not need:

| Not modeled | Why |
|---|---|
| Market hours / weekends | Prices move 24/7. A demo run at 9pm must still show a live tape. |
| Bid/ask spread | Market orders fill instantly at the single quoted price (`PLAN.md` §3). No spread to cross. |
| Volume | Nothing in the UI displays it. |
| Volatility clustering (GARCH) | Real volatility is autocorrelated; σ here is constant. Adds complexity invisible over a demo. |
| Mean reversion (Ornstein-Uhlenbeck) | GBM's unbounded random walk is fine over demo timescales. |
| Jump-diffusion (Merton) | The shock injection is a cruder version of the same idea, with one tunable parameter instead of three. |
| Market impact | The portfolio is $10k and doesn't move markets. |
| Splits, dividends, halts | No corporate-action machinery anywhere in the app. |

If any of these are ever wanted, the boundary is clean: they all live inside
`GBMSimulator.step()`, and `SimulatorDataSource` never has to change.
