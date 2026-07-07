# Market Terminal V3 — Module Catalog

Version: 1.0 · Date: 2026-07
See [`ARCHITECTURE.md`](ARCHITECTURE.md) for layering and runtime model. Each
module lists: purpose, key components, interface (conceptual), dependencies,
and the PRD epic / TODO tasks it implements.

Layer legend: **D** = `domain` (pure math) · **DA** = `data` (adapters) ·
**S** = `storage` · **A** = `app` (Tauri orchestration) · **F** = `front` (Svelte).

---

## 1. Data & Connectivity — PRD E1 (M1)

**Purpose:** ingest live and historical market data from multiple providers
behind one contract.

| Component | Layer | Notes |
|---|---|---|
| `data::http` | DA | shared HTTP client: retry/backoff, rate limiting, tracing, test-fake trait (T-1.2) |
| `data::finam` | DA | Finam Trade API adapter: gRPC/WS streams quotes/trades/bars, history (T-1.4) |
| `data::moex` | DA | MOEX ISS/ALGOPACK adapter (feature `moex`): tradestats/FUTOI/HI2/options board (T-1.3) |
| `MarketDataProvider` trait | DA | capability-declaring provider contract (T-1.11, ARCHITECTURE §4.1) |
| `ProviderRegistry` | A | capability routing, primary/fallback per instrument (T-1.11) |
| Feed supervisor | A | reconnect, stream→polling degradation, watchdog, feed metrics (T-1.4) |
| Instrument master | S/A | canonical `InstrumentId`, provider symbol mappings, contract specs, calendars (T-1.11) |

**Interface:** canonical event stream (`Quote/Trade/Bar/DomSnapshot`), history
pages, instrument resolution. **Depends on:** `domain` types only.
**Consumers:** everything.

## 2. Storage & History — PRD E1.3 (M1)

**Purpose:** production-grade historization and the terminal's system of record.

- DuckDB schema v3 + migrations; dataset catalog (instrument/TF/range/provider/quality-ref) (T-1.5).
- History loader: `missing_ranges` planning → paged backfill under provider rate
  limits, tail top-up, integrity control (duplicates/gaps/monotonic time),
  progress/cancel events to `HistoryTab` (T-1.6).
- Parquet export/import for all dataset tables (quant interop) (T-1.5).
- Hosts: trade journal, strategy registry, backtest reports, settings/workspaces.

**Interface:** dataset catalog API, series readers (bars/ticks by instrument/TF/range).
**Depends on:** `domain` types, provider layer for backfill.

## 3. Data Quality — PRD E1.4 (M1)

**Purpose:** no silent holes — every dataset carries a quality report.

- `domain::quality` (D): gap detection, outlier candles, volume anomalies,
  time-monotonicity checks; pure functions with tests (T-1.7).
- Quality report UI in `DatasetManager` (F) (T-1.8).
- (S) Corporate actions from ISS: splits/dividends, history adjustment (T-1.9).

## 4. Indicator Pipeline — PRD E3.3 (M2)

**Purpose:** one incremental-indicator engine shared by chart, screener, alerts,
and strategies.

- `domain::indicators` (D): trait `IncrementalIndicator` — O(1) per bar/tick,
  serializable state; EMA/SMA/VWAP(+bands)/Bollinger/RSI/MACD/ATR, session
  volume profile; reference-value golden tests (T-2.4).
- Port of existing metrics (MFI, CVD, unusual-volume z-score, …) onto the trait
  without changing results (golden tests) (T-2.5).

**Interface:** `indicator(id, params) → state machine`; composition nodes
(AND/OR/NOT, thresholds, crossovers) reused from `keyactivity`.
**Consumers:** chart (E2.3), screener (E3.1), alert rules (E3.2), strategy
interpreter (STRATEGY_FRAMEWORK §2).

## 5. Charting & Workspaces — PRD E2 (M2)

**Purpose:** the analyst's working surface.

- Docking (dockview): free layout, tabs, float windows; named workspaces (JSON in
  settings) + 3 presets Overview/Scalping/Research (T-2.1, T-2.2).
- Panel linking: color groups A/B/C, instrument+TF selection bus (T-2.3).
- Advanced chart (Lightweight Charts): multi-TF sync, indicator panes, session
  volume profile (T-2.6); drawing tools with per-instrument persistence (T-2.7).
- Trades/positions/orders on chart: entry/exit markers, draggable active-order
  lines → IPC to `TradeSession` (T-2.8).

## 6. Screener — PRD E3.1 (M2)

**Purpose:** find instruments by terminal-wide metrics, live.

- `domain::screener` (D): filter model over the indicator pipeline (turnover,
  %chg, MFI, CVD, unusual-volume z, HI2, FUTOI divergence, distance to extremes,
  spread/liquidity) (T-2.9).
- Saved presets, live updates, result table with sparklines (F).

## 7. Alert Center — PRD E3.2 (M2)

**Purpose:** one event stream instead of three.

- Merge `alerts` + Mega Alerts + Delta robots into a unified prioritized event
  stream: sound, toasts, journal (T-2.10).
- Rule builder UI: AND/OR/NOT composition reusing `keyactivity` nodes.
- (S) Correlation matrix + beta panel (T-2.11); (C) Ctrl+K command line (T-2.13).

## 8. Order Flow — PRD E2.5/E2.6, E4.1 (M3/M5)

**Purpose:** Bookmap/ATAS-class microstructure view.

- Trading DOM: one-click ladder trading, bracket in one action, quick sizes,
  centering, cumulative depth, position average price on ladder (T-3.3).
- Liquidity heatmap (S): DOM-snapshot ring buffer in Rust, decimation,
  Canvas/WebGL render, trade bubbles — performance spike first, go/no-go
  (T-5.1, T-5.2).
- Footprint chart (S): bid×ask clusters per price, delta, diagonal-imbalance
  highlighting ≥ N×, POC/VAH/VAL — extends `domain::delta` (T-5.3).

## 9. Trading — PRD E4 (M3)

**Purpose:** paper-parity execution now, live behind safeguards.

| Component | Layer | Notes |
|---|---|---|
| Execution simulator | D | Market/Limit/Stop/StopLimit, Day/GTC, OCO/bracket, partial fills, time-priority queue; deterministic tests (T-3.1) |
| `domain::risk` | D | limits (max position/instrument, daily loss, orders/min), auto-flatten on daily stop, kill switch; pre-trade check in the **single** order path, paper+live (T-3.2) |
| `FinamOrderRouter` (S) | DA | live placement/cancel/status, position & account reconcile on reconnect, feature `live-trading` (T-3.5, T-3.6) |
| Trade journal | S/F | executions with metric-context snapshot, P&L calendar, stats by instrument/time/setup, CSV export (T-3.4) |
| LIVE safeguards | F/A | PAPER/LIVE mode, first-order confirmation, red LIVE indication, kill-switch integration (T-3.7) |
| **Strategy runner** | A | loads validated strategy package from registry, executes intents through risk engine — paper first (T-3.9, STRATEGY_FRAMEWORK §5) |

## 10. Backtester & Research — PRD E5 (M4)

**Purpose:** honest, deterministic research loop.

- Tick-level engine: trade-tape driven, queue-position limit fills, book-based
  slippage; bit-for-bit determinism (T-4.1).
- Parameter optimization: grid + random search, rayon-parallel, result heatmap,
  neighbor-stability metric (T-4.2).
- Walk-forward: rolling IS/OOS windows, aggregated report, overfitting flag (T-4.3).
- Metrics: Sharpe/Sortino/Calmar, MAE/MFE, P&L distribution, drawdown profile,
  exposure; side-by-side run comparison (T-4.4).
- (S) Monte Carlo trade resampling → drawdown confidence intervals (T-4.5).
- CI benchmarks (criterion) with NFR thresholds (T-4.8).

## 11. Strategy Framework — PRD E5.6 extended (M4, spec: [`STRATEGY_FRAMEWORK.md`](STRATEGY_FRAMEWORK.md))

**Purpose:** strategies as data — defined in the backtester, loaded by trading.

- `domain::strategy` (D): versioned JSON config schema (universe, signals,
  entry/exit, sizing, risk overlay, execution policy, parameters) + validation
  + interpreter over the indicator pipeline (T-4.6).
- Strategy registry (S): packages, versions, lifecycle draft→validated→paper→
  live→retired (T-4.9).
- Promotion gates: optimization stability, walk-forward OOS efficiency,
  Monte-Carlo drawdown CI vs risk budget — gate report generator (T-4.10).
- UI: strategy editor (T-4.6) with 8 class templates (T-4.11), dataset picker,
  gate report, registry browser (T-4.7).

## 12. Options Analytics — PRD E6 (M5)

- Live ISS options board: chains, IV/greeks/OI per strike, smile calibration on
  live quotes (models exist in `domain::options`) (T-5.4).
- Position analysis: aggregated portfolio greeks, payoff now/at-expiry,
  scenarios spot ±X% / IV ±Y pp / time −N days (T-5.5).
- (C) 3D IV surface, IV Rank/Percentile history (T-5.6).

## 13. LLM Copilot — PRD E7 extended (M6, spec: [`LLM_LAYER.md`](LLM_LAYER.md))

- `data::llm` (DA): `LlmProvider` trait; Anthropic/OpenAI/Google/OpenRouter
  implementations over `data::http`; streaming, tool use, token/cost accounting
  (T-6.1).
- Task router: per-task model routing config, fallback chains, cost caps,
  response cache; redaction pass for private data (T-6.1, T-6.11).
- Features: Key Activity summary (T-6.2), morning brief (T-6.3), (C) chat panel
  with read-only tool use over IPC (T-6.4).
- Deterministic local fallback for every LLM feature.

## 14. Platform & Quality — PRD E8 (M6)

- Packaging: MSI/NSIS via GitHub Actions Windows runner, (S) Tauri updater (T-6.5).
- Health panel: feed statuses, lag, memory, DB size; rotating file logs,
  diagnostics export (T-6.6).
- Performance pass: table virtualization everywhere, Rust→UI event batching
  ≤ 60 Hz (T-6.7).
- (S) RU/EN localization (T-6.8); (S/C) light theme, hotkey editor (T-6.9).

---

## Dependency graph (module level)

```
Instrument master ──┬─▶ Data & Connectivity ─▶ Storage & History ─▶ Backtester
                    │           │                      │                │
                    │           ▼                      ▼                ▼
                    │   Indicator Pipeline ─▶ Screener/Alerts   Strategy Framework
                    │           │                                      │
                    │           ▼                                      ▼
                    └─▶ Charting/Workspaces      Trading (simulator+risk+runner)
                                │                                      │
                                └────────── Order Flow ◀───────────────┘
LLM Copilot: reads Storage + domain summaries (no write/trade path)
Platform/Quality: cross-cutting
```
