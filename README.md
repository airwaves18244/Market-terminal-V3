# Market Terminal V3

Desktop trading and market-analysis terminal for a private trader/researcher
(futures, FX, stocks, bonds). Tauri + Rust core (`domain`/`data`/`storage`/`app`)
+ Svelte frontend. Data: Finam Trade API + MOEX ISS/ALGOPACK first, multi-source
provider architecture for global markets later. Analysis: multi-LLM copilot
(Anthropic / OpenAI / Google / OpenRouter). Research loop: deterministic
backtester → optimization → walk-forward → Monte Carlo → strategy packages
loaded by the trading module (paper → live behind a risk engine).

## Planning package

| Document | Contents |
|---|---|
| [`PRD.md`](PRD.md) | Product requirements V3 (RU): epics E1–E8, NFR, milestones M1–M6, risks |
| [`TODO.md`](TODO.md) | Task plan (RU): tasks per milestone with model/effort assignment, M7 backlog |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System architecture: layers, runtime/event model, `MarketDataProvider` abstraction, instrument master, LLM layer, strategy artifact pipeline, security |
| [`docs/MODULES.md`](docs/MODULES.md) | Module catalog: purpose, components, interfaces, dependencies, epic mapping |
| [`docs/STRATEGY_FRAMEWORK.md`](docs/STRATEGY_FRAMEWORK.md) | Strategy anatomy, JSON config schema, taxonomy of 8 strategy classes (momentum, swing, breakout, stat-arb, futures arb, carry, volatility, rotation), lifecycle & promotion gates, strategy package format |
| [`docs/DATA_SOURCES.md`](docs/DATA_SOURCES.md) | Provider plan: Finam/MOEX capabilities, adapter duties, global expansion backlog, capability matrix, data-quality requirements |
| [`docs/LLM_LAYER.md`](docs/LLM_LAYER.md) | Multi-LLM layer: provider abstraction, task→model routing, context builders, privacy/redaction, cost control, degradation |
| [`SUMMARY_TRENDS.md`](SUMMARY_TRENDS.md) | Research: cross-asset capital-flow detection on the post-2022 Russian market — closed-system thesis, rate-cycle regime clock, 5-tier flow-signal taxonomy, rotation playbook, system rules |
| [`PRD_TRENDS.md`](PRD_TRENDS.md) | Build plan for the Cross-Asset Flow & Regime engine: `domain::flow` module, regime classifier, data feeds + gaps, screener/alerts/strategy-gating/LLM-brief integration, TR-task plan |

Status: planning approved for decomposition; implementation follows `TODO.md`
milestones M1–M6 (M7 = unscheduled global-data backlog).
