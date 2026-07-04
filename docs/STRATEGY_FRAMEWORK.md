# Market Terminal V3 — Strategy Framework

Version: 1.0 · Date: 2026-07 · Scope: **specification only** — strategy
structures and key aspects are defined here for the backtester; no strategy
code is written at the planning stage.

Extends PRD **E5.6** (strategy DSL) and connects PRD **E5** (backtester) to
**E4** (trading). Architecture context: [`ARCHITECTURE.md`](ARCHITECTURE.md) §7.

Core idea: **a strategy is data, not code.** A strategy is a versioned JSON
config interpreted by `domain::strategy` over the incremental indicator
pipeline (E3.3). It is created and validated in the backtester, packaged, and
loaded by the trading module — first paper, then (optionally) live behind the
risk engine.

---

## 1. Strategy anatomy (common structure)

Every strategy, regardless of class, is composed of the same blocks:

| Block | Contents | Notes |
|---|---|---|
| **Metadata** | id, name, version, class (taxonomy §3), author notes, created/modified | id + version identify a package |
| **Universe** | static instrument list *or* dynamic screener query (E3.1); asset classes; sessions/calendar constraints | dynamic universes re-evaluate on schedule |
| **Data requirements** | timeframe(s), history warm-up depth, required streams (bars/ticks/DOM), auxiliary series (second leg, index, FUTOI…) | backtester refuses to run if the dataset can't satisfy them |
| **Signal block** | named signals = indicator pipeline nodes + logical composition (AND/OR/NOT, crossovers, thresholds, z-scores, time filters) | only E3.3 indicators — same values on chart, screener, backtest, live |
| **Entry rules** | signal → side (long/short/both), entry order type (market / limit at offset / stop), optional confirmation window, max concurrent positions | |
| **Exit rules** | any combination: stop-loss (fixed / ATR-multiple / structure), take-profit (R-multiple / target), trailing (ATR / percent / structure), time exit (bars / session end / calendar), opposite-signal exit | at least one hard stop is **mandatory** (schema validation) |
| **Position sizing** | fixed lots · fixed risk-% per trade (distance-to-stop based) · volatility targeting (ATR / realized σ) · capped Kelly fraction | sizing outputs are clamped by the risk overlay |
| **Risk overlay** | per-trade max risk, per-instrument max position, daily loss stop, max trades/day, correlation/exposure caps | delegated to and enforced by `domain::risk` (E4.3) — strategy may only be stricter, never looser than account limits |
| **Execution policy** | order types allowed, limit-chase rules, max slippage assumption (backtest) / cancel threshold (live), participation cap (max % of volume), no-trade windows (news, auction, roll) | shared semantic simulator ↔ live router |
| **Parameters** | every tunable value declared once: name, type, default, **optimization range + step**, walk-forward defaults | interpreter reads params by name; optimizer/W-F iterate over declared ranges only |

### Design rules

1. **No look-ahead by construction**: signals evaluate on closed bars (or tick
   N uses data ≤ N); interpreter enforces this, not strategy authors.
2. **Determinism**: same config + same dataset ⇒ bit-for-bit identical report
   (PRD invariant §3.5).
3. **One source of truth for math**: any indicator used by a strategy must
   exist in `domain::indicators` — no ad-hoc math inside configs.
4. **Mandatory protective stop** and risk overlay — schema-level validation.

## 2. Strategy config schema (JSON, versioned)

Sketch of schema v1 (full JSON Schema to be authored in T-4.6):

```jsonc
{
  "schema": "strategy/v1",
  "meta": { "id": "mom-breakout-ri", "name": "RI momentum breakout",
            "class": "momentum", "version": 3 },
  "universe": { "static": ["FUT:RTS@FORTS"],
                "sessions": ["main"], "calendar": "MOEX" },
  "data": { "tf": "5m", "warmup_bars": 400, "streams": ["bars"],
            "aux": [] },
  "signals": {
    "trend_up":  { "op": "gt", "a": {"ind": "ema", "params": {"len": "p.fast"}},
                              "b": {"ind": "ema", "params": {"len": "p.slow"}} },
    "breakout":  { "op": "cross_above", "a": {"src": "close"},
                              "b": {"ind": "donchian_high", "params": {"len": "p.chan"}} },
    "vol_ok":    { "op": "gt", "a": {"ind": "atr", "params": {"len": 14}},
                              "b": {"const": "p.min_atr"} }
  },
  "entry": { "long":  { "when": {"all": ["trend_up", "breakout", "vol_ok"]},
                        "order": {"type": "stop", "offset_ticks": 2} },
             "short": null, "max_positions": 1 },
  "exit":  { "stop":  {"type": "atr_multiple", "mult": "p.stop_atr"},
             "take":  {"type": "r_multiple",  "r": "p.take_r"},
             "trail": {"type": "atr", "mult": "p.trail_atr", "after_r": 1.0},
             "time":  {"type": "session_end"} },
  "sizing": { "type": "risk_percent", "risk_pct": 0.5 },
  "risk":   { "max_trades_per_day": 6, "daily_loss_stop_r": 3.0 },
  "execution": { "max_slippage_ticks": 2, "no_trade": ["auction", "expiry_day"] },
  "params": {
    "fast":     {"default": 20, "range": [10, 40],  "step": 5},
    "slow":     {"default": 80, "range": [40, 160], "step": 20},
    "chan":     {"default": 55, "range": [20, 100], "step": 5},
    "stop_atr": {"default": 2.0, "range": [1.0, 4.0], "step": 0.5},
    "take_r":   {"default": 3.0, "range": [1.5, 5.0], "step": 0.5},
    "trail_atr":{"default": 2.5, "range": [1.5, 4.0], "step": 0.5},
    "min_atr":  {"default": 50, "range": [0, 200], "step": 25}
  }
}
```

Multi-leg strategies (stat-arb, spreads) add a `legs` array with per-leg
instruments/ratios and signals computed on **synthetic series** (spread,
ratio, z-score) declared in the `signals` block — see §3.4–3.5.

## 3. Strategy class taxonomy — structures and key aspects

For each class: concept, typical signal structure, holding period, suitable
markets, data needs, key parameters, principal risks, and what "good" looks
like in evaluation. These are the templates the backtester's strategy editor
ships with (as config skeletons, not code) — T-4.11.

### 3.1 Momentum / trend-following

| Aspect | Definition |
|---|---|
| Concept | ride persistent directional moves; enter with strength, exit on weakening |
| Signals | MA/EMA crossovers or slope; Donchian/channel breakout with trend filter; time-series momentum (return over N periods > threshold); ADX regime filter |
| Holding | hours → weeks (intraday variant: session-bounded) |
| Markets | futures (index/commodity/rates), FX, liquid stocks — anything that trends |
| Data | bars (1m–1d), ATR for stops/sizing; long warm-up (≥ slow window ×3) |
| Key parameters | fast/slow windows, channel length, ATR stop multiple, trend-filter threshold |
| Sizing | volatility targeting (ATR-normalized) — canonical for this class |
| Principal risks | whipsaw/chop regimes (many small losses), gap risk over sessions, regime shift |
| Evaluation | expect win rate 30–45% with payoff > 2; equity smoothness across *regimes* matters more than total return; must survive W-F on trending *and* choppy windows |

### 3.2 Swing / mean-reversion

| Aspect | Definition |
|---|---|
| Concept | fade short-term extremes back toward a mean; multi-day swing holds |
| Signals | RSI(2–14) extremes with higher-TF trend alignment; Bollinger %B reversion; distance-from-MA z-score; gap-fade with volume confirmation |
| Holding | 1–10 days (swing); minutes–hours (intraday reversion) |
| Markets | stocks and stock indices (strongest reversion), FX ranges; **careful in futures with hard trends** |
| Data | daily + intraday bars; breadth/regime context (E3 metrics) as filters |
| Key parameters | lookback of the mean, entry z-score/RSI threshold, exit-at-mean vs fixed R, max holding days |
| Sizing | fixed risk-% per trade; scale-in (max 2 steps) allowed but capped by risk overlay |
| Principal risks | "falling knife" — mean keeps moving; correlated entries across similar tickers; short side carries squeeze risk |
| Evaluation | expect win rate 55–70% with payoff < 1; tail control is the metric to watch (MAE distribution, worst-trade share of P&L); correlation cap across open positions |

### 3.3 Breakout / volatility expansion

| Aspect | Definition |
|---|---|
| Concept | enter when price leaves a compression range as volatility expands |
| Signals | opening-range breakout; NR7/inside-bar compression → range break; Bollinger squeeze release; unusual-volume z-score confirmation (E3.1 metric) |
| Holding | intraday → few days |
| Markets | futures (index/commodities), volatile stocks; session-structure-aware |
| Data | intraday bars, session calendar (auction/open times), volume; T&S helpful for confirmation |
| Key parameters | range definition window, breakout offset (ticks), volume confirmation threshold, session time windows |
| Principal risks | false breakouts (majority of signals), slippage at exactly the busiest moments, news spikes |
| Evaluation | slippage sensitivity analysis is mandatory (backtest at 1×/2×/3× assumed slippage); dependence on first-hour liquidity must be visible in the report by time-of-day breakdown |

### 3.4 Statistical arbitrage (pairs / spread mean-reversion)

| Aspect | Definition |
|---|---|
| Concept | trade the *spread* between related instruments, not direction |
| Structure | 2+ legs: spread = A − β·B (or log-ratio); z-score of spread over lookback; enter at ±z_entry, exit at z_exit/0, hard stop at ±z_stop |
| Signals | cointegration screen (research-stage, offline), rolling hedge ratio β (OLS/Kalman-style rolling estimate), spread z-score bands |
| Holding | hours → weeks |
| Markets | same-sector stocks; futures vs its proxy basket; related commodity contracts; FX crosses vs synthetic cross |
| Data | synchronized bars for all legs (alignment by exchange time — data-quality critical), borrow/short constraints for stocks |
| Key parameters | lookback for β and z, z_entry/z_exit/z_stop, β re-estimation frequency, max half-life accepted |
| Sizing | per-leg sized to equal dollar (or β-weighted) exposure; spread-level risk-% on z_stop distance |
| Principal risks | **cointegration break** (structural: the single biggest risk), legging risk on execution, short leg availability, correlated spreads blowing up together |
| Evaluation | report must show spread half-life stability across W-F windows; P&L per leg (detect hidden directional bet); stress: what if one leg gaps 5% against |

### 3.5 Futures arbitrage (calendar / basis / index)

| Aspect | Definition |
|---|---|
| Concept | relative-value between contracts of the same underlying: calendar spreads (near vs far month), futures-vs-spot basis, index future vs component basket |
| Structure | legs with exchange-defined ratios; signal on spread deviation from fair value (carry/dividend/rate model) or from its own rolling band |
| Holding | days → until expiry convergence |
| Markets | MOEX FORTS calendars and basis (Si/RI vs spot), later global futures |
| Data | full contract chain (specs, expiries, roll calendar from instrument master), spot/underlying series, FUTOI as flow context |
| Key parameters | fair-value model inputs (rate, dividends), entry/exit deviation bands, days-to-expiry entry window, roll rules |
| Principal risks | convergence later than expiry of your patience (margin/carry cost), fair-value model error, liquidity in far months, forced roll |
| Evaluation | P&L must come from convergence, not leg direction (report decomposition); margin usage over time; worst-case daily margin call scenario |

### 3.6 Carry (FX carry / bond curve roll-down)

| Aspect | Definition |
|---|---|
| Concept | earn the yield differential (FX) or roll-down the curve (bonds) while managing the price risk of the position |
| Signals | FX: rate differential ranking with volatility filter and crash-regime cutoff (risk-off indicator); bonds: curve steepness, roll-down per unit duration |
| Holding | weeks → months (slowest class in the terminal) |
| Markets | FX pairs/futures, government bond futures / OFZ curve |
| Data | daily bars, rate/curve reference data (macro provider — M7 backlog; until then manual reference inputs), calendar of central-bank meetings |
| Key parameters | ranking lookback, vol filter threshold, risk-off cutoff rule, rebalance frequency |
| Principal risks | carry crashes ("up the stairs, down the elevator") — negative skew is structural; central-bank surprises |
| Evaluation | skew/tail metrics dominate: Sortino over Sharpe, drawdown profile, performance in risk-off windows isolated in the report; Monte-Carlo CI mandatory before paper |

### 3.7 Volatility & options strategies

| Aspect | Definition |
|---|---|
| Concept | trade implied-vs-realized volatility level or IV-rank extremes with defined-risk option structures |
| Structure | entry condition on IV metrics (IV Rank/Percentile from E6.3, realized-vs-implied spread) → structure template (vertical, iron condor, calendar, straddle) from the options module (E6.2) |
| Holding | days → weeks (theta-driven) |
| Markets | MOEX index/stock options (E6.1 board) |
| Data | live options board (chains, IV, greeks, OI), underlying series, IV history |
| Key parameters | IV-rank entry threshold, structure selection rules, delta bands for adjustment, profit-take % of max credit, DTE entry/exit windows |
| Principal risks | gap through short strikes, IV regime shift, liquidity/wide markets in strikes, pin risk at expiry |
| Evaluation | scenario grid (spot ±X% × IV ±Y pp × time) from E6.2 is part of the gate report; max structural loss must fit per-trade risk budget by construction |

### 3.8 Cross-sectional momentum / sector rotation

| Aspect | Definition |
|---|---|
| Concept | periodically rank a universe and hold the leaders (long) / laggards (short-if-available); the portfolio version of momentum |
| Signals | relative strength vs index over N weeks; RRG quadrant transitions (existing panel math); breadth-regime gate (only deploy when market breadth is healthy) |
| Holding | weeks; scheduled rebalance (weekly/monthly) |
| Markets | stock universe (sector leaders), sector futures where available |
| Data | daily bars for full universe, index series, breadth metrics (existing E3 panels) |
| Key parameters | ranking lookback(s), number of positions, rebalance period, breadth-gate threshold |
| Sizing | equal-weight or inverse-volatility across selected names |
| Principal risks | momentum crashes at regime turns, concentration in one hot sector, turnover costs |
| Evaluation | turnover and cost-drag explicitly in report; sector-concentration exposure metric; compare against buy-and-hold index baseline in the same window |

### Taxonomy → terminal features used

| Class | Indicator pipeline | Screener | Options module | Multi-leg engine | DOM/tick data |
|---|---|---|---|---|---|
| Momentum | ● | ○ | | | |
| Swing/mean-rev | ● | ● | | | |
| Breakout | ● | ● | | | ● (confirmation) |
| Stat-arb | ● | ● (pair screen) | | ● | |
| Futures arb | ● | | | ● | ○ |
| Carry | ● | | | | |
| Volatility | ● | | ● | ● | |
| Sector rotation | ● | ● | | | |

## 4. Lifecycle and promotion gates

```
draft ──▶ backtested ──▶ optimized ──▶ WF-validated ──▶ MC-checked ──▶ paper ──▶ live ──▶ retired
  │            │              │              │               │           │         │
  └────────────┴──────────────┴──── any failed gate returns to draft ────┴─────────┘
```

| Gate | Check | Pass criteria (defaults, per-strategy overridable) |
|---|---|---|
| **G1 Backtest** | deterministic run on full dataset, costs & slippage on | positive expectancy after costs; ≥ 100 trades (or ≥ 30 for slow classes §3.6–3.8); no single trade > 15% of net P&L |
| **G2 Optimization stability** | grid/random search heatmap (T-4.2) | chosen params sit on a **plateau**: median of neighbor cells ≥ 70% of chosen cell's objective; no isolated-peak selection |
| **G3 Walk-forward** | rolling IS/OOS (T-4.3) | W-F efficiency (OOS/IS objective ratio) ≥ 0.5; ≥ 60% of OOS windows profitable; no parameter drift beyond declared ranges |
| **G4 Monte Carlo** | trade-order resampling (T-4.5) | 95th-percentile max drawdown ≤ strategy's declared risk budget; P(ruin at daily-stop cascade) below threshold |
| **G5 Paper** | live paper trading via the real runner + risk engine | ≥ 20 trades or ≥ 4 weeks; realized slippage within 1.5× backtest assumption; behavior matches simulator (paper parity report) |
| **G6 Live** | manual owner decision | G1–G5 green + security review of live path (T-3.R) done; starts at minimum size |

Gate results are produced by the backtester as a **gate report** (T-4.10)
stored with the package; the trading module refuses to load a package into
paper without G1–G4, and into live without G1–G5 (enforced in code, not
convention).

## 5. Strategy package — what the trading module loads

```
StrategyPackage {
  config:        strategy JSON (schema §2), immutable snapshot
  params:        frozen parameter set chosen by validation (no ranges — values)
  risk_block:    resolved risk overlay (merged: strategy ∩ account limits, strictest wins)
  reports:       { backtest_ref, optimization_ref, walkforward_ref, montecarlo_ref }
  hashes:        sha256 of config+params; dataset fingerprint used for validation
  status:        draft | validated | paper | live | retired
  audit:         created_at, promoted_at per state, promoted_by (owner action log)
}
```

- Stored in the **strategy registry** (`storage`, T-4.9); versions are
  append-only — promoting a modified config creates a new version starting at
  `draft`.
- The **strategy runner** (`app`, T-3.9) loads a package, instantiates the
  interpreter with frozen params, subscribes to required data, and emits order
  *intents*; every intent passes `domain::risk` pre-trade checks (E4.3) before
  becoming an order — identical path for paper and live.
- Kill switch flattens and suspends all running strategies instantly (E4.3).
- (C, later) Semi-automation mode (E4.6): runner proposes, trader confirms
  each order.

## 6. What is explicitly out of scope at this stage

- Writing any concrete strategy configs beyond editor skeleton templates.
- ML-based signal generation (feature store, model training) — possible V4
  direction; the config schema reserves a `signal` node type for external
  series to keep the door open.
- Portfolio-level optimization across strategies (beyond correlation/exposure
  caps in the risk overlay).
