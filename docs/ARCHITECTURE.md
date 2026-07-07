# Market Terminal V3 — System Architecture

Version: 1.0 · Date: 2026-07 · Status: approved for decomposition
Companion documents: [`MODULES.md`](MODULES.md) · [`STRATEGY_FRAMEWORK.md`](STRATEGY_FRAMEWORK.md) · [`DATA_SOURCES.md`](DATA_SOURCES.md) · [`LLM_LAYER.md`](LLM_LAYER.md) · [`../PRD.md`](../PRD.md) (requirements, Russian) · [`../TODO.md`](../TODO.md) (task plan)

This document defines the target architecture for Market Terminal V3 — a desktop
trading and market-analysis terminal for a private trader/researcher covering
futures, FX, stocks, and bonds. It extends the PRD (§3 "Principles and
architectural invariants") with three architectural additions:

1. a **multi-source market-data provider layer** (Finam + MOEX first, global providers later without rework),
2. a **multi-LLM analysis layer** (Anthropic, OpenAI, Google, OpenRouter behind one abstraction),
3. a **strategy artifact pipeline** connecting the backtester to the trading module.

---

## 1. Technology stack

| Layer | Technology | Rationale |
|---|---|---|
| Shell / packaging | Tauri 2 (Rust) | small footprint, native performance, single binary per OS |
| Core | Rust workspace (crates: `domain`, `data`, `storage`, `app`) | deterministic math, high-throughput tick processing (NFR: ≥50k ticks/s) |
| Persistence | DuckDB (analytical store) + Parquet (export/import/interchange) | columnar analytics on history, zero-server, quant-friendly interchange |
| Frontend | Svelte + TypeScript, Lightweight Charts, dockview, Canvas/WebGL for order flow | mock-first panels runnable in a plain browser (PRD invariant §3.3) |
| IPC | Tauri events/commands, batched + coalesced ≤ 60 Hz | NFR: feed-event→pixel p50 ≤ 100 ms |
| Secrets | OS keyring (+ `.env` for dev, outside repo) | PRD invariant §3.4 |

## 2. Layering (invariant)

```
┌───────────────────────────────────────────────────────────────────┐
│ front (Svelte)   panels · workspaces · charts · DOM · dialogs     │
│                  mock-first: every panel runs in browser on mocks │
├───────────────────────────────────────────────────────────────────┤
│ app (Tauri)      orchestration · supervisors · IPC · schedulers   │
│                  event coalescing → UI · session state            │
├──────────────────────────┬────────────────────────────────────────┤
│ data (adapters)          │ storage                                │
│  provider registry       │  DuckDB catalog (datasets, journal,    │
│  data::finam (gRPC/WS)   │  strategy registry, settings)          │
│  data::moex  (HTTP ISS)  │  Parquet import/export                 │
│  data::http  (shared)    │  migrations (schema v3)                │
│  data::llm   (LLM APIs)  │                                        │
├──────────────────────────┴────────────────────────────────────────┤
│ domain (pure)    indicators · quality · risk · strategy · backtest│
│                  screener · delta/footprint · options · alerts    │
│                  NO network, NO UI, NO database — math only       │
└───────────────────────────────────────────────────────────────────┘
```

**Dependency rule:** `domain` depends on nothing above it. `data`/`storage` are
thin adapters that map external contracts into `domain` types. `app` wires
everything; `front` sees only IPC DTOs. Violations fail code review.

**Testing rule (invariant):** network contracts are frozen as fixtures before
implementation (`(verify)` tasks); orchestration is tested against fakes;
`domain` targets ≥ 90% line coverage; backtests are bit-for-bit deterministic.

## 3. Runtime & event model

### 3.1 Ingest pipeline

```
provider stream ──▶ ingest task (per provider/instrument group)
      │                │  normalize → canonical events (Quote/Trade/Bar/DomSnapshot)
      │                ▼
      │           domain pipelines (incremental indicators, alert rules,
      │           delta/footprint accumulation, screener metrics)
      │                ▼
      │           state stores (ring buffers for DOM/T&S, series caches)
      │                ▼
      └──watchdog  UI batcher: coalesce per panel topic, flush ≤ 60 Hz frames
                        ▼
                  Tauri events → Svelte stores → panels
```

- **One async ingest task per stream**, supervised: auto-reconnect with jittered
  backoff; degradation to REST polling when a stream is down; watchdog emits feed
  health metrics (lag, gap count, reconnect count) — PRD E1.1.
- **Backpressure:** order-flow data (DOM snapshots, T&S) lands in fixed-size
  ring buffers in Rust; the UI batcher decimates to frame rate. Slow consumers
  never stall ingest.
- **Canonical event types** live in `domain` and are provider-agnostic; adapters
  translate provider payloads at the boundary (see §4).
- **Clock discipline:** all events carry exchange timestamp + local receive
  timestamp; downstream math uses exchange time, feed-health uses the delta.

### 3.2 Threading model

| Concern | Model |
|---|---|
| Network I/O (streams, HTTP) | tokio async tasks, one supervisor per provider |
| Domain pipelines | synchronous, driven by ingest tasks; hot paths O(1)/event |
| Backtest / optimization | rayon thread pool (CPU-bound, isolated from live runtime) |
| Storage writes | dedicated writer task, batched inserts; readers use DuckDB snapshots |
| UI delivery | single batcher task, frame-coalesced |

Live trading and research workloads are isolated: a running optimization must
not add latency to the live feed path (separate pools, bounded queues).

## 4. Market-data provider layer (extension → PRD E1)

Goal: **Finam + MOEX ISS/ALGOPACK now, global providers (US stocks/futures, FX,
bonds/macro) later — added as new adapters, with zero rework of consumers.**
Details and provider matrix: [`DATA_SOURCES.md`](DATA_SOURCES.md).

### 4.1 `MarketDataProvider` contract (trait, `data`)

Each provider adapter implements a single trait and **declares its
capabilities**; consumers query the registry by capability, never by provider
name.

Capabilities (bitset / enum set):

| Capability | Meaning |
|---|---|
| `ReferenceData` | instrument lookup/search, contract specs, trading calendar |
| `BarsHistory` | historical OHLCV with paging, per-TF depth limits |
| `TicksHistory` | historical trades (tick) with paging |
| `QuotesStream` | live L1 quotes |
| `TradesStream` | live time & sales |
| `DomStream` | live order book (levels count declared) |
| `BarsStream` | live bar updates |
| `CorporateActions` | splits/dividends (E1.5) |
| `OptionsBoard` | option chains, IV/OI (E6.1) |
| `OrderRouting` | order placement/cancel/status (E4.5) — trading-capable providers |

Trait surface (conceptual):

```
trait MarketDataProvider {
    fn id(&self) -> ProviderId;
    fn capabilities(&self) -> CapabilitySet;
    fn rate_limits(&self) -> RateLimitPolicy;          // enforced by shared limiter
    async fn resolve(&self, query: SymbolQuery) -> Vec<InstrumentRef>;
    async fn history(&self, req: HistoryRequest) -> Page<CanonicalBars|CanonicalTicks>;
    async fn subscribe(&self, req: StreamRequest) -> CanonicalEventStream;
    // corporate actions / options board behind capability checks
}
```

Adapter obligations:
- translate provider symbols/payloads → canonical types **at the boundary**;
- honor the shared `RateLimiter` (per-provider budgets, e.g. Finam 200 req/min);
- surface errors as typed conditions (retryable / fatal / auth) for the supervisor;
- ship contract fixtures (`(verify)`) before parser code.

### 4.2 Instrument master & symbol normalization

A terminal-wide **instrument master** (in `storage`) gives every tradable thing
one canonical identity:

```
InstrumentId (canonical, stable)
 ├─ asset class: Future | FX | Stock | Bond | Option | Index
 ├─ contract spec: tick size, tick value, lot, currency, expiry/roll (futures),
 │                 coupon/maturity (bonds), base/quote (FX)
 ├─ trading sessions & calendar (per exchange)
 └─ provider mappings: { finam: "SBER@TQBR", moex: "SBER", <future providers>: … }
```

- Consumers (charts, screener, backtester, risk) speak `InstrumentId` only.
- Symbol resolution/search goes through the master; adapters register mappings.
- Multi-currency: every instrument declares its quote currency; P&L conversion
  is a deferred M7 concern but the schema reserves it now.

### 4.3 Provider registry, routing, failover

- `ProviderRegistry` (in `app`): registers adapters enabled by feature flags,
  routes each `(InstrumentId, capability)` request to the configured primary
  provider with optional fallback (e.g. Finam stream ↓ → MOEX ISS polling for
  the same instrument, where mappings exist).
- Health-driven degradation: supervisor demotes a failing provider stream to
  polling, promotes back on recovery; all transitions visible in the health
  panel (PRD E8.3).
- History loading honors `missing_ranges` planning + integrity checks
  (duplicates/gaps/time monotonicity) regardless of provider (PRD E1.3, E1.4).

## 5. Storage architecture

DuckDB single file (per profile) + Parquet interchange:

| Area | Contents |
|---|---|
| `datasets` catalog | bar/tick datasets: instrument, TF, range, provider, quality report ref |
| market history | bars/ticks tables partition-pruned by instrument/TF/date |
| instrument master | canonical instruments, provider mappings, contract specs, calendars |
| trade journal | executions with metric-context snapshots, P&L calendar (E4.4) |
| **strategy registry** | strategy packages, versions, lifecycle states, backtest report refs (§7) |
| backtest reports | run configs, metrics, equity curves, optimization grids, WF windows |
| settings/workspaces | named layouts (JSON), drawings per instrument, alert rules |

Rules: migrations versioned (schema v3); Parquet export/import for every
dataset table (quant interop: Python/polars reads the same files); no silent
gaps — quality report is part of the dataset record.

## 6. LLM analysis layer (extension → PRD E7)

Multi-provider from day one. Full spec: [`LLM_LAYER.md`](LLM_LAYER.md).

- **`LlmProvider` trait** (`data::llm`): `chat(request) -> stream`, tool-use
  support, token accounting; implementations: Anthropic, OpenAI, Google,
  OpenRouter (aggregator). Built on shared `data::http` (retry/backoff).
- **Task router:** each analysis task (morning brief, key-activity summary,
  screener explanation, strategy critique, journal review, chat) maps to a
  configured `(provider, model, effort)` with a fallback chain. Cheap models
  for periodic summaries, strong models for reasoning-heavy critique.
- **Privacy & safety:** redaction pass strips account identifiers and absolute
  position sizes before any external call; per-task opt-out; keys in keyring;
  responses cached; per-day cost budget with hard stop.
- **Degradation:** every LLM feature has a deterministic local fallback
  (template summary from the same context data) — terminal never blocks on LLM
  availability (PRD E7.1).

## 7. Strategy artifact pipeline (backtester → trading)

The backtester is where strategies are **defined, parameterized, and
validated**; the trading module **loads** the resulting artifact. Strategy
anatomy, taxonomy (momentum, swing/mean-reversion, breakout, stat-arb, futures
arbitrage, carry, volatility, sector rotation) and the config schema:
[`STRATEGY_FRAMEWORK.md`](STRATEGY_FRAMEWORK.md).

```
research (dataset picker → strategy config JSON → deterministic backtest)
   ▼
validation gates (optimization stability → walk-forward OOS → Monte-Carlo DD CI)
   ▼
strategy package  = config + frozen params + risk block + report hash + status
   ▼
strategy registry (storage; states: draft → validated → paper → live → retired)
   ▼
trading module loads package → paper execution (always first)
   ▼
live execution — feature-gated, risk engine pre-checks every order, kill switch
```

Key invariant: **simulator and live router share one execution semantic**
(order types, OCO/bracket, partial fills, time-priority queue) so that paper
parity is meaningful (PRD E4.2). The strategy interpreter emits *intents*;
`domain::risk` is the mandatory gate between intent and order for both paper
and live (PRD E4.3).

## 8. Security model

- Secrets (broker tokens, LLM keys) only in OS keyring; `.env` for dev, never
  committed; logs scrubbed of secrets.
- Live trading: compile-time feature `live-trading` + explicit PAPER/LIVE mode
  switch + first-order-of-session confirmation + red LIVE indication + kill
  switch (PRD E4.5); mandatory security review (TODO T-3.R) before default-on.
- Risk engine checks are in the *only* code path to order submission — there is
  no bypass API.
- LLM calls: redaction (§6), no order-placement tools exposed to LLM tool-use
  (chat can *read* terminal state, never trade).

## 9. Observability & quality

- Health panel: per-feed status/lag/reconnects, memory, DB size (E8.3).
- Structured file logs with rotation; diagnostics export bundle.
- CI: `fmt`, `clippy -D warnings`, `cargo test --workspace`, svelte-check,
  vitest; criterion benchmarks guard NFR §5 (tick throughput, backtest speed);
  perf regressions fail CI.

## 10. Extension points (deliberate, reserved now)

| Future need | Reserved mechanism |
|---|---|
| Global data providers (M7 backlog) | `MarketDataProvider` trait + instrument master mappings (§4) |
| New LLM providers/models | `LlmProvider` trait + routing config (§6) |
| New strategy classes | strategy config schema blocks + indicator pipeline (E3.3) |
| New panels | docking workspace JSON + panel linking bus (E2.1/E2.2) |
| Additional brokers for routing | `OrderRouting` capability on provider trait |

## 11. Epic mapping

| PRD epic | Architecture section |
|---|---|
| E1 Data & connectivity | §3.1, §4, §5 |
| E2 Charting & workspaces | §3.1 (UI batcher), §10 |
| E3 Screener/alerts/indicators | §3.1 (domain pipelines) |
| E4 Trading | §7, §8 |
| E5 Backtest & optimization | §3.2, §7 |
| E6 Options | §4.1 (`OptionsBoard`), §5 |
| E7 LLM copilot | §6 |
| E8 Platform & quality | §9 |
| *Extension:* multi-source data | §4 (new) |
| *Extension:* multi-LLM | §6 (new) |
| *Extension:* strategy pipeline | §7 (new) |
