# Market Terminal V3 — Data Sources Plan

Version: 1.0 · Date: 2026-07
Adapter contract and instrument master: [`ARCHITECTURE.md`](ARCHITECTURE.md) §4.

Scope decision: **Finam Trade API and MOEX ISS/ALGOPACK are the first-class
providers of V3** (per PRD E1). Global providers for US/European futures,
stocks, FX, and bonds/macro are a **planned expansion (M7 backlog)** — the
adapter architecture is built for them now, the integrations are scheduled
later.

---

## 1. First-class providers (V3, scheduled)

### 1.1 Finam Trade API — primary broker feed

| Aspect | Plan |
|---|---|
| Transport | gRPC + WebSocket streams; shared retry/backoff policy |
| Capabilities | `ReferenceData`, `BarsHistory`, `TicksHistory`, `QuotesStream`, `TradesStream`, `DomStream`, `BarsStream`, `OrderRouting` (M3, feature `live-trading`) |
| Asset classes | MOEX stocks, FORTS futures/options, currency section (FX-linked), bonds |
| Rate limits | ~200 req/min — enforced by shared `RateLimiter`; backfill scheduler pages within budget, night windows for deep history |
| Resilience | supervisor: auto-reconnect with jittered backoff, degradation to polling, gap detection, feed metrics (lag/reconnects) — PRD E1.1, TODO T-1.4 |
| Contract fixation | live fixtures before parser code (`(verify)` tasks); `OrderService`/`AccountsService` fixtures at start of M3 (T-3.5) — statuses and partial fills are an open question (PRD §9.1) |
| Open questions | historical tick depth (PRD §9.2) — affects tick backtests (E5.1); verify early in M1 |

### 1.2 MOEX ISS / ALGOPACK — exchange analytics feed

| Aspect | Plan |
|---|---|
| Transport | HTTP JSON (`data::http`: retry/backoff/limits/tracing — T-1.2), feature `moex` |
| Capabilities | `ReferenceData` (securities directory, calendars), `BarsHistory` (candles), `CorporateActions` (splits/dividends — E1.5), `OptionsBoard` (E6.1); ALGOPACK: tradestats, FUTOI, HI2, Super Candles |
| Role | analytics enrichment (flow/positioning data unavailable from broker feed) + fallback history/reference source + options board |
| Contract fixation | live JSON fixtures for tradestats/futoi/hi2/options board **before** parsers (T-1.1) — top of the critical path |

### 1.3 Cross-provider duties (both, and every future adapter)

- Map provider symbols into the **instrument master** (canonical `InstrumentId`,
  contract specs, sessions/calendar) — T-1.11.
- Historization through the same loader: `missing_ranges` planning, paged
  backfill, integrity checks (duplicates/gaps/time monotonicity), quality
  report per dataset (E1.3/E1.4). **Zero silent holes** (NFR).
- Typed error surface (retryable/fatal/auth) for the supervisor; health panel
  visibility (E8.3).

## 2. Planned global expansion — M7 backlog (architecture-ready, unscheduled)

Selection criteria for future adapters: official API with clear licensing for
individual use, historical depth sufficient for backtests, cost sane for a
private trader, and coverage of one of the user's asset classes.

| Slot | Candidate profile (examples) | Capabilities to implement | Notes |
|---|---|---|---|
| US stocks + ETFs | Polygon.io / Tiingo / Alpaca Data-class API | `ReferenceData`, `BarsHistory`, `TicksHistory`, `QuotesStream`, `TradesStream`, `CorporateActions` | corporate actions and delisted-symbol history matter for honest backtests |
| Global futures | Databento-class API (CME/ICE/Eurex) | `ReferenceData` (contract chains!), `BarsHistory`, `TicksHistory`, `DomStream` (MBP) | contract specs + roll calendars must land in the instrument master; tick data is the expensive part — plan storage budget |
| FX | broker API of the user's FX broker (OANDA-class REST/stream) | `ReferenceData`, `BarsHistory`, `QuotesStream`, `OrderRouting` (much later) | FX has no consolidated tape — mark FX datasets as broker-specific in quality reports |
| Bonds / rates / macro | FRED-class macro API + treasury/curve source | `ReferenceData`, `BarsHistory` (daily) | feeds the carry/curve strategy class (STRATEGY_FRAMEWORK §3.6) and macro context for LLM briefs |
| News/calendar (optional) | economic-calendar API | n/a (separate `EventProvider` mini-trait) | powers no-trade windows in execution policy and morning brief context |

M7 also includes the cross-cutting work global data forces:
- multi-currency P&L and account base-currency conversion;
- per-exchange trading calendars/sessions in the instrument master (schema is
  reserved in V3, populated per provider);
- timezone-correct session alignment for multi-leg strategies across venues.

## 3. Provider capability matrix (target state)

| Capability | Finam | MOEX ISS/ALGOPACK | US stocks (M7) | Futures (M7) | FX (M7) | Macro (M7) |
|---|---|---|---|---|---|---|
| ReferenceData | ● | ● | ● | ● | ● | ● |
| BarsHistory | ● | ● | ● | ● | ● | ● (daily) |
| TicksHistory | ● (depth TBD) | — | ● | ● | ○ | — |
| QuotesStream | ● | — | ● | ● | ● | — |
| TradesStream | ● | — | ● | ● | — | — |
| DomStream | ● | — | ○ | ● | — | — |
| CorporateActions | — | ● | ● | — | — | — |
| OptionsBoard | ○ | ● | ○ | ○ | — | — |
| OrderRouting | ● (M3) | — | — | — | ○ | — |

● planned/required · ○ optional/if available · — not applicable.

## 4. Data-quality requirements (provider-independent)

Every dataset, from any provider, passes `domain::quality` (T-1.7):

1. **Gap detection** against the instrument's trading calendar (not naive
   clock-time) — gaps are recorded, visible, and re-fetchable, never silent.
2. **Outlier candles** (price spikes beyond k·ATR with no volume support) —
   flagged, not auto-deleted; corporate-action-aware once E1.5 lands.
3. **Volume anomalies** — zero-volume bars in liquid sessions, volume
   inconsistent between TFs of the same range.
4. **Time monotonicity + duplicates** — hard errors at ingest.
5. **Cross-provider reconciliation** (when two providers cover one instrument)
   — daily close/volume diff report; discrepancies beyond tolerance flag the
   dataset.
6. **Multi-leg alignment** (stat-arb/spreads): synchronized timestamps across
   legs verified before a backtest is allowed to run (STRATEGY_FRAMEWORK §3.4).

Quality report is stored with the dataset and shown in `DatasetManager`
(T-1.8); backtests record the dataset fingerprint they ran on
(STRATEGY_FRAMEWORK §5).
