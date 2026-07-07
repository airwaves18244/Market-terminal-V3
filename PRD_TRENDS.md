# PRD — Cross-Asset Flow & Regime Engine (Market Terminal V3)

Version: 1.0 · Date: 2026-07 · Status: proposed extension to `PRD.md`
Research basis: [`SUMMARY_TRENDS.md`](SUMMARY_TRENDS.md).
Architecture context: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) ·
[`docs/MODULES.md`](docs/MODULES.md) ·
[`docs/STRATEGY_FRAMEWORK.md`](docs/STRATEGY_FRAMEWORK.md) ·
[`docs/DATA_SOURCES.md`](docs/DATA_SOURCES.md) ·
[`docs/LLM_LAYER.md`](docs/LLM_LAYER.md).

This PRD adds a **Cross-Asset Flow & Regime** capability to V3: it detects capital
rotation between Russian asset classes (equities, OFZ, corporate bonds, FX,
cash/money-market), classifies the **market regime** on the rate cycle, and turns
both into **systematic, regime-gated trading rules** across the terminal's
existing 8 strategy classes. It is written to slot into the V3 layering and
milestones with **one new pure-math module (`domain::flow`)** plus a small set of
new data feeds — no architectural rework.

---

## 1. Goal & scope

**Goal.** Give the trader/researcher a live, quantified answer to: *"where is
money flowing between asset classes right now, what regime are we in on the rate
cycle, and which class/strategy does that favor?"* — with backtestable rules.

**In scope (V3 extension):**
- A cross-asset **turnover-rotation** engine (near-zero-sum flow accounting).
- A **regime classifier** {rate level × rate direction} × {flow/risk state}.
- **Flow signals** across the 5 tiers of `SUMMARY_TRENDS.md` §4.
- Screener/alert/LLM-brief integration and **regime-gating** of strategies.
- New data feeds needed for the above (with graceful degradation where absent).

**Out of scope (respect existing non-goals):** macro-forecasting the CBR;
non-MOEX/Finam data beyond the minimal macro reference series (full macro
provider stays **M7 backlog**, `DATA_SOURCES.md` §2); ML regime models (schema
leaves the door open, not built now); any auto-trading (LLM/flow engine never
places orders — `ARCHITECTURE.md` §8).

**Design invariants (inherited, non-negotiable):** all math in `domain` (no
network/UI/DB); mock-first front; deterministic backtest; secrets in keyring;
every LLM feature has a deterministic local fallback.

---

## 2. Users & scenarios (maps to PRD §2 personas)

- **P2 swing/positional analyst (primary):** opens the **Flow & Regime** panel →
  sees the current regime chip, the cross-asset turnover-share chart, the yield
  gap, and money-fund flows → gets the rotation read ("early easing: duration
  over equities, accumulate div-payers") → builds positions accordingly.
- **P3 quant researcher:** uses flow/regime series as **strategy signals and
  gates**; backtests a rotation strategy that is only active in declared regimes;
  validates it survives walk-forward across tightening *and* easing windows.
- **P1 active trader:** consumes regime as **context/guardrail** — flow alerts
  ("unusual OFZ-vs-equity turnover rotation"), no-trade guards on CBR days.

---

## 3. Functional requirements

Priorities: **M** must · **S** should · **C** could. IDs `TR*` (Trend/Rotation)
to avoid collision with PRD `E*`.

### TR1 — Cross-asset flow data (M) → depends on E1, `data::moex`
- **TR1.1 (M)** Per-board **turnover aggregation**: daily Σ value/volume traded
  per asset class — equities (TQBR), OFZ (TQOB), corporate bonds (TQCB), FX
  (CETS), derivatives (FORTS) — from ISS `securities` marketdata (`VALTODAY`)
  and `history/.../totals`. Emit a canonical `ClassTurnover{class, val, vol,
  trades, date}` series.
- **TR1.2 (M)** **Positioning feed**: ALGOPACK **FUTOI** (`FIZ`/`YUR` long/short,
  holders count) for Si/CNY/RI/MIX; **tradestats** aggressor imbalance per class;
  reuse the T-1.1/T-1.3 MOEX contract work (already planned) — this is *new
  consumers*, not new transport.
- **TR1.3 (M)** **Cross-asset price/yield references**: RGBI, RGBI-Y, RUCBITR,
  IMOEX, MOEXREPO/RUONIA-implied via ISS index history; CNY/RUB & USD/RUB (CETS)
  and Si/CNY futures (FORTS) for FX & basis.
- **TR1.4 (S)** **Supply feeds** (leading regime tells): OFZ auction take-up /
  bid-to-cover and issuance mix. **Not in ISS** → a small **MinFin scraper**
  (`data::minfin`, feature-flagged) producing `OfzAuction{date, offered, placed,
  cover, wa_yield, series_type}`; degrade to "unavailable" cleanly.
- **TR1.5 (S)** **Money-market fund AUM/flows** ("cash on the sidelines"): source
  from MOEX fund statistics / provider pages; `FundFlow{date, aum, net_flow}`.
  Manual-import fallback if no clean feed.
- **TR1.6 (S)** **Macro reference series**: key rate history + **CBR meeting
  calendar** + RUONIA. Minimal `data::macro_ref` (manual/CSV + optional CBR
  scrape) until the M7 macro provider; **the key rate is the master input and is
  not in the MOEX API**, so this cannot be skipped.
- **TR1.7 (M)** **Corporate-actions/dividends** for reinvestment-flow modeling:
  ISS `/iss/securities/<sec>/dividends` (reuse E1.5); build the dividend calendar
  and expected reinvestment flow.

Data-availability detail and confirmed gaps: **§5**.

### TR2 — `domain::flow` pure-math engine (M) → new module, layer **D**
All computation is pure functions/incremental indicators on the E3.3 trait, ≥90%
test coverage, deterministic (golden tests against fixtures).

- **TR2.1 (M)** **Turnover-share & rotation**: per-class share of total turnover;
  **rolling z-score** vs a **seasonally-adjusted baseline** (day-of-week, holiday,
  expiry, month-of-year controls); a `RotationVector` = coordinated share shifts
  (one class up, another down) with a rotation strength/score.
- **TR2.2 (M)** **Cross-asset yield gap** ("rotation clock"): money-fund rate
  (≈ key rate) vs OFZ YTM (RGBI-Y) vs equity dividend/earnings yield; gap series
  + threshold-cross events (equity-vs-OFZ gap closing = equity-leg trigger).
- **TR2.3 (M)** **Regime classifier**: state = {rate level bucket, rate direction}
  × {flow/risk state from TR2.1 + money-fund flow + breadth}. Deterministic,
  hysteresis-guarded (no flip-flop), emits `Regime{monetary, flow, label,
  confidence, since}`. Labels align to `SUMMARY_TRENDS.md` §3/§5
  (Tightening / Peak / EarlyEasing / LateEasing / Neutral × RiskOff / Rotation /
  RiskOn).
- **TR2.4 (M)** **Positioning signals**: FUTOI `FIZ`/`YUR` net-position and
  divergence (FX-futures vs index-futures); **retail-crowding extreme** (contrarian)
  detector; HI2 concentration and OBStats liquidity-withdrawal flags.
- **TR2.5 (S)** **Supply signals**: OFZ auction bid-to-cover trend; **floater↔fixed
  issuance-mix** ratio and its regime interpretation; substituted/CNY bond
  flow; corporate primary-issuance drain.
- **TR2.6 (S)** **FX/basis**: annualized Si/CNY futures basis vs key rate (implied
  carry); tax-window ruble-support and dividend-conversion demand markers.
- **TR2.7 (M)** **Seasonal/event calendar model**: CBR meeting days, dividend
  season & expected reinvestment pulse, tax windows (~25th–28th), FORTS expiries
  (3rd Fri quarterly), index rebalances, OFZ auction Wednesdays → typed
  `FlowEvent`s with expected direction and no-trade flags.
- **TR2.8 (C)** **Type-C overhang scenario** input: a parameterized
  "de-escalation release" scenario generator for stress testing (not a base
  signal).

### TR3 — Screener, alerts, panel (M/S) → E3 integration
- **TR3.1 (M)** New **screener metrics** (`domain::screener`, E3.1): turnover-share
  z-score, rotation strength, yield-gap, FUTOI-`FIZ` net & divergence, auction
  cover, issuance-mix, money-fund net flow — filterable/sortable like existing
  metrics.
- **TR3.2 (M)** **Flow alerts** in the unified Alert Center (E3.2): "unusual
  cross-asset rotation", "regime change", "yield-gap trigger", "retail crowding
  extreme", "weak OFZ auction", "CBR-day/expiry no-trade window". Composable via
  the existing AND/OR/NOT rule builder.
- **TR3.3 (S)** **Flow & Regime panel** (front, mock-first): regime chip (like
  the breadth-regime pill), stacked cross-asset turnover-share chart, yield-gap
  chart, money-fund flow, FUTOI positioning, upcoming `FlowEvent` calendar.
  Links into the panel bus (A/B/C groups, E2.2).
- **TR3.4 (S)** **Regime overlay**: expose the current `Regime` terminal-wide (a
  first-class shared state, alongside breadth) so any panel/strategy can read it.

### TR4 — Strategy-framework integration (M/S) → STRATEGY_FRAMEWORK
- **TR4.1 (M)** **Regime gate in strategy config**: add an optional
  `regime_gate` block — a strategy declares the regimes it may trade in; the
  runner (T-3.9) **suppresses off-regime strategies** (a stricter overlay,
  consistent with the risk-overlay "strategy may only be stricter" rule).
  Signals from `domain::flow` are exposed to the interpreter as E3.3-style nodes
  (so they are usable in `signals`/`entry`/`exit` like any indicator).
- **TR4.2 (S)** **Rotation strategy templates** (config skeletons, T-4.11 style —
  data, not code): (a) **Duration-at-the-turn** (buy RGBI/long-OFZ on yield-gap +
  early-easing regime); (b) **Cash→Equity rotation** (deploy from money-fund into
  dividend payers → cyclicals as regime advances + money-fund outflow confirms);
  (c) **Gov↔Corp spread** (RUCBITR-vs-RGBI stat-arb); (d) **FX carry/basis**
  (Si/CNY basis vs key rate); (e) **Sector rotation, regime-gated** (RRG, only in
  late-easing/easy + risk-on). Each is a template over the existing classes
  (Carry, Stat-arb, Futures-arb, Sector-rotation), not a new engine.
- **TR4.3 (M)** **Regime-aware validation gate**: extend the walk-forward gate
  (G3) so a rotation/regime strategy must be evaluated across **both tightening
  and easing** windows (regime coverage report); flags strategies fit to a single
  regime. Feeds the existing gate report (T-4.10).

### TR5 — LLM flow brief (S) → E7 / LLM_LAYER
- **TR5.1 (S)** New LLM task **`flow.regime_brief`** (routing per `LLM_LAYER.md`
  §2, **strong** tier, fallback → mid → **deterministic local template**): a
  daily narrative of the regime read, the dominant rotation, the yield-gap clock,
  supply tells, and the week's `FlowEvent`s. Context builder is deterministic
  (structured JSON from `domain::flow`); the **local fallback renders the same
  context** as a template so the feature works offline. Extends the morning brief
  (T-6.3). Redaction rules unchanged (market-only task → unrestricted).

---

## 4. Architecture placement (no rework)

| Concern | Placement | Notes |
|---|---|---|
| Flow math, regime classifier, signals | **`domain::flow`** (new, layer D) | pure, incremental (E3.3 trait), deterministic, ≥90% coverage |
| Turnover/positioning/index feeds | `data::moex` (existing) + new **consumers** | reuses T-1.1/T-1.3 contracts; adds board-totals & index-history parsing |
| OFZ auctions | **`data::minfin`** (new adapter, feature `minfin`) | HTML scrape → typed `OfzAuction`; behind `MarketDataProvider`-style capability |
| Money-fund AUM, macro ref (rate/calendar/RUONIA) | **`data::macro_ref`** (new, minimal) | manual/CSV + optional scrape; **M7 macro provider supersedes later** |
| Regime as shared state | `app` orchestration | published terminal-wide like breadth regime; consumed by panels/screener/runner |
| Screener/alerts/panel/brief | existing E3.1/E3.2/E7 modules | new metrics/rules/task, not new subsystems |
| Strategy gating | `domain::strategy` + strategy runner (E4/E5) | `regime_gate` block + flow signal nodes |

Everything routes through the existing `MarketDataProvider`/registry, instrument
master, historization (`missing_ranges`, integrity, quality report), and the
indicator pipeline. New adapters declare capabilities and ship `(verify)`
fixtures before parser code, exactly as `DATA_SOURCES.md` §1.3 requires.

---

## 5. Data sources & availability (with confirmed gaps)

Grounded against MOEX ISS / ALGOPACK / MinFin (see `SUMMARY_TRENDS.md` §4, App. A).
Paths are the integration targets; **all must be smoke-tested live before parser
code** (contract-fixture rule). ✅ = available via API · ⚠️ = derived/partial ·
❌ = no clean API (scrape/manual).

| Signal | Source | Path / product | Status |
|---|---|---|---|
| Per-board turnover (equities/OFZ/corp/FX) | ISS | `/iss/engines/<eng>/markets/<mkt>/boards/<board>/securities` (`VALTODAY`); `/iss/history/engines/stock/totals/...` | ✅ |
| FUTOI (FIZ/YUR positioning) | ALGOPACK | `fo/futoi` (`pos_long/short`, `clgroup`, 5-min) | ✅ |
| Tradestats (aggressor imbalance) | ALGOPACK | `eq|fo|fx/tradestats` (`val`,`vol`,`trades`,`pr_vwap`,imbalance) | ✅ |
| HI2 concentration / OBStats depth | ALGOPACK | `hi2`, `obstats` (`spread_bbo`,`imbalance_vol_bbo`,`levels_*`) | ✅ (OBStats: token auth) |
| RGBI / RGBI-Y / RUCBITR / IMOEX | ISS | index pages + `/iss/.../markets/index/...` history | ✅ (verify exact RGBI path) |
| MOEXREPO / RUONIA | ISS/CBR | MOEXREPO index; RUONIA only as derived bond field `IRICPICLOSE` | ⚠️ standalone RUONIA via CBR/manual |
| CETS FX spot (CNY/USD) & Si/CNY futures | ISS | `/iss/engines/currency/markets/selt/boards/CETS/...`; `/iss/engines/futures/markets/forts/boards/RFUD/...` | ✅ (real-time = subscription) |
| Dividends | ISS | `/iss/securities/<sec>/dividends` (`RegistryCloseDate`,`Value`) | ✅ |
| OFZ auction bid-to-cover & issuance mix | MinFin | per-auction HTML pages | ❌ scrape (`data::minfin`) |
| Corporate issuance volume/mix (floater↔fixed↔substituted) | mixed | primary market; no clean demand API | ❌ scrape/manual/3rd-party |
| Money-market fund AUM/flows | MOEX/providers | fund statistics pages | ⚠️ scrape/manual import |
| Key rate + CBR meeting calendar | CBR | cbr.ru | ❌ manual/CSV + optional scrape |

**Degradation rule (M):** each `domain::flow` signal declares its required feeds;
if a tier is unavailable the signal is marked `degraded`/omitted and the regime
classifier down-weights it — the engine never blocks (mirrors `LLM_LAYER.md` §6
and the "no silent holes" quality rule).

---

## 6. Non-functional requirements (extend PRD §5)

| Metric | Target |
|---|---|
| Flow/regime recompute latency | ≤ 1 s after end-of-day/aggregate update; intraday tiers ≤ frame budget |
| `domain::flow` test coverage | ≥ 90% lines; golden tests on fixtures; deterministic |
| Regime stability | hysteresis prevents >1 label flip per N sessions absent real change |
| Backtest determinism | flow signals bit-for-bit reproducible (PRD invariant §3.5) |
| Data degradation | any single feed absent → engine runs with that signal down-weighted, visibly flagged |
| Provenance | every derived number traces to source series + as-of date (no hard-coded levels) |

---

## 7. Milestone mapping & task plan (model/effort per TODO conventions)

Placed to **piggyback existing milestones** (M1 data, M2 pipeline/screener/alerts,
M4 strategy/backtest, M6 LLM). Model roles per `TODO.md`: **Fable 5** =
architect/reviewer, **Opus 4.8** = lead multi-layer, **Sonnet 5** = executor,
**Haiku 4.5** = quick/boilerplate. DoD = green CI (fmt, clippy -D warnings, tests,
svelte-check/vitest).

| ID | Task | Milestone | Model | Effort |
|---|---|---|---|---|
| TR-1 | `(verify)` Fix live contracts for board-totals, FUTOI, tradestats, index-history (RGBI/RUCBITR/IMOEX), dividends, CETS/FORTS FX — fixtures + edge cases | M1 (with T-1.1) | Opus 4.8 | high |
| TR-2 | `data::minfin` (feature `minfin`) OFZ auction scraper → typed `OfzAuction`; `data::macro_ref` (rate/calendar/RUONIA, CSV+scrape); money-fund AUM importer; typed errors + degradation | M1 | Sonnet 5 | medium |
| TR-3 | **`domain::flow` core**: turnover-share + seasonally-adjusted z-score, `RotationVector`, cross-asset yield gap — pure math, golden tests | M2 (with T-2.4) | Fable 5 | high |
| TR-4 | **Regime classifier** (rate×direction × flow/risk), hysteresis, `Regime` state; deterministic tests across historical windows | M2 | Opus 4.8 | high |
| TR-5 | Positioning/supply/FX signals: FUTOI divergence & retail-crowding, HI2/OBStats flags, auction cover, issuance-mix, Si/CNY basis | M2 | Sonnet 5 | medium |
| TR-6 | Seasonal/event calendar model → typed `FlowEvent`s (CBR/dividends/tax/expiry/rebalance/auction) + no-trade flags | M2 | Sonnet 5 | medium |
| TR-7 | Screener metrics (TR3.1) + Alert Center rules (TR3.2) wiring | M2 | Sonnet 5 | medium |
| TR-8 | **Flow & Regime panel** (front, mock-first): regime chip, turnover-share chart, yield-gap, money-fund, FUTOI, event calendar; panel-bus linking | M2 | Sonnet 5 | medium |
| TR-9 | Regime shared-state publish in `app` + terminal-wide overlay | M2 | Sonnet 5 | low |
| TR-10 | Strategy `regime_gate` block + flow signal nodes for the interpreter; runner suppression of off-regime strategies | M4 (with T-4.6/T-3.9) | Opus 4.8 | high |
| TR-11 | Rotation strategy **templates** (config skeletons, TR4.2 a–e) — data, not code | M4 (with T-4.11) | Sonnet 5 | medium |
| TR-12 | Regime-coverage walk-forward gate extension (both tightening & easing) → gate report | M4 | Fable 5 | medium |
| TR-13 | LLM task `flow.regime_brief` (context builder + prompt template + **local fallback**), routing entry, extend morning brief | M6 (with T-6.3) | Sonnet 5 | medium |
| TR-14 | Docs: fold results into README/SUMMARY; update MODULES with `domain::flow` | M2/M6 | Haiku 4.5 | low |
| TR-R | Review: correctness of regime classifier & flow accounting identity; look-ahead audit of flow signals | M2/M4 | Fable 5 | high |

---

## 8. Acceptance criteria (success = these hold)

1. The terminal displays a **live regime label** on the rate cycle and the
   **cross-asset turnover-share** chart, recomputed from live data, with full
   provenance (no hard-coded levels).
2. A **rotation and a regime-change alert** fire correctly on historical fixtures
   spanning the 2024 peak → 2025-26 easing (e.g. detects the bonds-lead-equities
   sequence and the money-fund plateau).
3. The **yield-gap "rotation clock"** is exposed as a metric and as a trigger
   usable in the screener and in strategy configs.
4. At least the **five rotation templates** (TR4.2) load in the strategy editor
   and pass G1 backtest deterministically; **regime-gated** strategies are
   provably suppressed off-regime by the runner.
5. The **walk-forward regime-coverage gate** rejects a strategy fit to a single
   regime.
6. The **`flow.regime_brief`** renders online *and* offline (local fallback) with
   identical context.
7. Every flow signal **degrades gracefully** when its data tier is missing, and
   says so in the UI.

---

## 9. Risks & mitigations (extend PRD §8)

| Risk | Mitigation |
|---|---|
| Key data not in a clean API (OFZ auctions, RUONIA, fund AUM) | isolate in `data::minfin`/`data::macro_ref`; degrade-and-flag; manual import path; supersede with M7 macro provider |
| Short post-2022 history → overfit thresholds | adaptive/rolling normalization; mandatory walk-forward across tightening *and* easing (TR-12); MC drawdown gate |
| Retail-flow **reflexivity** (signals self-fulfilling then reverting) | size by conviction × regime; crowding as **contrarian**; no all-in on one signal |
| Regime **flip-flop** on noise | hysteresis + confidence + minimum-dwell in the classifier (TR-4) |
| Look-ahead in flow/seasonal signals | closed-bar evaluation enforced by the interpreter (STRATEGY_FRAMEWORK §1); explicit look-ahead audit (TR-R) |
| Geopolitical **type-C release** tail | scenario generator (TR2.8), not a base signal; stress in MC; event-risk no-trade windows |
| FX fragmentation / thin substituted-bond liquidity | CNY-centric FX gauge; liquidity filters on basis signals; mark FX by venue/instrument |
| Scope creep into macro-forecasting | hard non-goal (§1); engine *reads* the rate, never predicts it |

---

## 10. Open questions

1. Cleanest reliable source for **money-market fund AUM/flows** (MOEX statistics
   vs provider pages) — pick during TR-2.
2. Exact ISS paths for **RGBI/RGBI-Y/RUCBITR** index history (Sonnet research
   inferred but could not confirm live) — resolve in the TR-1 `(verify)` step.
3. **Regime bucket boundaries** (rate levels, z-score thresholds, the yield-gap
   equity-trigger near ~10–12%) — initial defaults from `SUMMARY_TRENDS.md`,
   then calibrated/walk-forwarded; must stay config, not code.
4. Whether **OBStats/real-time FUTOI** (subscription tiers) are in budget, or the
   engine runs on delayed/EOD tiers first.
