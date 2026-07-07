# Cross-Asset Capital Flows on the Russian Market — Research Summary

Version: 1.0 · Date: 2026-07 · Author: quant research (for Market Terminal V3)
Scope: how to **detect and trade capital rotation** between Russian asset classes
(equities, government (OFZ) and corporate bonds, FX spot & futures, cash/money
market) in the **post-2022 closed-capital regime**.

Companion PRD for implementation: [`PRD_TRENDS.md`](PRD_TRENDS.md). This document
holds the **findings and reasoning**; the PRD holds the **build plan**.

> **Data caveat.** Regime facts below are grounded to mid-2026 public sources
> (Bank of Russia, MOEX, MinFin, market commentary). Numbers are directional and
> dated; the terminal must recompute everything from live data, never hard-code a
> level. Endpoint-level data availability is detailed in `PRD_TRENDS.md` §5.

---

## 0. The one-sentence thesis

> Since 2022 the Russian market is a **near-closed capital system** whose master
> variable is the **Bank of Russia key rate**; because foreign flows no longer
> dominate, money rotating out of one domestic asset class must land in another,
> so **cross-asset flow is close to zero-sum and therefore unusually
> measurable** — and the rotation sequence across the rate cycle is stable enough
> to encode as **systematic, regime-gated trading rules**.

Everything else in this document unpacks that sentence.

---

## 1. Why 2022 is the structural break (and why it *helps* flow analysis)

Before 2022 the marginal price-setter on MOEX was **global capital**: foreigners
held roughly half of the free float in blue chips and a large share of OFZ, and
prices tracked EM risk sentiment, the carry trade, and Euroclear-settled flows.
Flow analysis was noisy because the biggest flows crossed the border and were
invisible on-exchange.

After February 2022 that channel was cut, in both directions:

| Change (2022→) | Mechanism | Consequence for flow analysis |
|---|---|---|
| Foreign capital frozen | Non-resident assets from "unfriendly" jurisdictions immobilized; **type-C accounts** trap dividends/coupons (114-FZ delisting, 319-FZ conversion) | The former dominant flow is now **inert**; on-exchange flow ≈ the *whole* flow |
| Depositary receipts forcibly converted | Receipts (расписки) bought offshore converted to local shares; CBR throttled sales to **0.2%/day** of converted stock to prevent a crash | Created a multi-year **selling overhang** that slowly decayed; a known, dateable supply drag |
| Cross-border investing restricted | Residents can't freely buy foreign assets; capital is **penned inside the ruble contour** | The investable ruble pool is largely **conserved** → rotation is near-zero-sum |
| Retail became the market | 40.1M brokerage accounts by end-2025 (35.1M end-2024); retail ≈ **~80% of equity turnover** | The marginal buyer is a **domestic individual** → sentiment, dividends, and tax seasonality dominate; herding/momentum stronger |
| FX market fragmented | June-2024 sanctions on NCC pushed **USD/EUR off-exchange**; **CNY/RUB** became the liquid on-exchange pair | On-exchange FX signal = **CNY-centric**; USD/RUB read via fixing/futures/OTC |
| Eurobond access lost → **substituted bonds** (замещающие) | FX-denominated bonds settled *in rubles* domestically | A new **"FX-exposure-without-leaving" asset class** — a distinct rotation node |
| Rates went structurally high | Key rate 7.5% → **21% (Oct-2024 peak)** → easing through 2025-26 | **Cash became a real competing asset class**; money-market funds are now a first-class rotation node, not a footnote |

**Net effect.** Removing foreign flow removed the loudest source of noise. In a
closed system, a fall in one class's turnover/holdings shows up as a rise in
another's — the plumbing is visible on MOEX boards. This is the core reason a
flow-and-rotation engine is worth building for *this* market specifically.

---

## 2. The players and the plumbing of the closed system

Who actually moves money between classes, and what each one responds to:

| Actor | Size / role | Rotates between | Primary driver | On-screen footprint |
|---|---|---|---|---|
| **Retail** (физлица) | ~40M accounts, ~80% of equity volume | cash funds ↔ bonds ↔ equities ↔ FX | rate level, dividends, sentiment, headlines | FUTOI `FIZ`; tradestats aggressor imbalance; equity turnover |
| **Banks / treasury** | dominant OFZ & repo bid | repo ↔ short OFZ ↔ floaters | key rate, liquidity (RUONIA/repo) | OFZ auction take-up; MOEXREPO; floater share |
| **Corporates** | bond issuers, FX users | issue bonds; hold/sell FX | funding cost vs rate view; tax/dividend calendar | primary issuance mix; CETS/futures FX flow |
| **Institutions** (НПФ/insurers) | slow, mandated | long OFZ ↔ blue-chip equity | mandates, curve level | steady auction/blue-chip bid |
| **MinFin** | largest single bond issuer | supplies OFZ (fixed 26-/floater 29-series) | budget need, rate expectations | **weekly OFZ auctions**; issuance mix |
| **Bank of Russia** | sets the key rate | — (the valve, not a player) | inflation, ruble, expectations | **8 rate decisions/yr** |
| **Trapped non-residents** | latent overhang | frozen (type-C) | geopolitics | inert until a de-escalation scenario |

**Key implication:** with foreigners inert, the rotation is essentially a contest
between **retail sentiment** and the **risk-free cash yield set by the CBR**. That
is a two-body problem, which is why it is tractable.

---

## 3. The master cycle — a rate-driven "regime clock"

The key rate is the gravitational constant. It sets (a) the **risk-free cash
yield** (money-market funds ≈ key rate), (b) the **equity discount rate**,
(c) **bond prices** inversely, and (d) the **ruble carry**. The cycle of capital
across classes is therefore a function of the **rate level and its direction**.

### 3.1 The canonical rotation sequence at a rate-cycle turn

Empirically (and confirmed by 2024-26 flow data), the sequence around a peak is:

```
  TIGHTENING            PEAK / PIVOT           EARLY EASING           LATE EASING / EASY
  rate ↑ or high-flat   rate tops (21%, '24)   rate ↓, still tight    rate ↓ toward neutral
──────────────────────┼──────────────────────┼──────────────────────┼──────────────────────
 money → CASH & floaters │ CASH hoard peaks     │ ① LONG OFZ rally      │ ② EQUITIES lead
 short OFZ, замещающие   │ money-fund AUM peaks  │   (RGBI front-runs cuts)│   (div payers→cyclicals→growth)
 EQUITIES de-rate       │ EQUITIES bottom       │   fixed-coupon issuance │ money-funds DRAIN
 RGBI falls (yield ↑)   │ RGBI bottoms (max yld)│   returns; carry good  │ small-cap/beta, sector rotation
 ruble firm (carry)     │                       │   equities still lag    │ ruble softer (carry gone)
```

**The critical, tradable regularities:**

1. **Bonds lead equities at the turn.** Long OFZ (duration, RGBI) rallies *first*
   as the market front-runs cuts — before equities. The first great trade of an
   easing cycle is **buying duration at the yield peak**, not buying stocks.
2. **Equities lag until cash stops competing.** As long as the risk-free cash
   yield (~key rate) sits far above the equity dividend/earnings yield, money
   stays in bonds and funds. Equities take real leadership only when the rate
   falls toward a threshold where cash no longer pays enough — commentary places
   that trigger near **~10-12%** for Russia.
3. **The rotation is money-fund-gated.** The wall of money-market fund AUM is the
   fuel. Sustained **net outflow** from money funds is the confirmation that the
   rotation into risk has actually started; rising AUM says it hasn't.

### 3.2 Where the clock reads in mid-2026 (worked example)

- Key rate **21% (Oct-24) → 16% (Dec-25) → 14.25% (Jun-26)**; CBR baseline
  average **13–15% for 2026**. → **Early-easing** regime, still restrictive.
- Money-market fund AUM kept **rising** through the cuts (>1.6 trn ₽, ~2.5M
  investors by 2026) — the great rotation has **not** fired.
- Jan-2026 retail allocation: of ~139 bn ₽ added, **93.5 bn → bonds, 44.8 bn →
  funds, only 0.7 bn → equities.** → Phase ① (bonds) is playing out; Phase ②
  (equities) has not.
- **Reading:** favor **duration (OFZ) and quality carry** over equities; treat
  equities as *accumulate, not chase*; watch money-fund flows and the rate's
  approach toward ~12% as the switch for the equity leg. A sharp geopolitical
  de-escalation is the discontinuous accelerator of the same rotation.

This is exactly the kind of read the terminal should produce **automatically**
from live data.

---

## 4. Flow-signal taxonomy — what to actually measure

Signals are organized in five tiers, from direct flow measurement to predictable
calendar pulses. Each maps to a concrete data source in `PRD_TRENDS.md` §5.

### Tier 1 — Turnover rotation (direct, the user's "unusual volume")
- **Cross-asset turnover share.** Daily Σ value-traded per class — equities
  (TQBR), OFZ (TQOB), corporate bonds (TQCB), FX (CETS), derivatives (FORTS) —
  expressed as **each class's share of total**, then **z-scored vs a
  seasonally-adjusted trailing baseline**. Rising share of one + falling of
  another = rotation. This is the primary "atypically large turnover" detector.
- **Why z-score, not raw:** turnover has trend, day-of-week, holiday, and
  expiry seasonality; only the *anomaly vs its own regime* is signal.

### Tier 2 — Positioning & microstructure (who is moving)
- **FUTOI (ALGOPACK):** individuals (`FIZ`) vs legal entities (`YUR`) net
  long/short in **Si/CNY** (FX) vs **RI/MIX** (equity index) futures. Divergence
  = rotation between FX-hedging and equity exposure; **retail crowding extremes
  = contrarian**.
- **Aggressor imbalance (tradestats):** net buy/sell pressure per class per bar.
- **HI2 concentration:** rising concentration = institutional ("smart-money")
  dominance in a class.
- **OBStats depth/spread:** liquidity withdrawal ahead of a rotation (book
  thinning in FX before a stock→FX move).

### Tier 3 — Cross-asset relative value (price-based confirmation)
- **RGBI / RGBI-Y vs IMOEX:** the stock-vs-bond rotation; bonds usually lead.
- **RUCBITR (corp) vs RGBI (sovereign) spread:** credit-driven flow gov ↔ corp.
- **The cross-asset yield gap:** money-fund rate (≈ key rate) vs OFZ YTM vs
  equity dividend/earnings yield. When the equity yield closes the gap to OFZ,
  the equity leg becomes likely. **This gap is the single best "rotation clock."**
- **FX basis:** annualized Si/CNY futures basis vs key rate = implied carry;
  deviations reveal directional FX demand.

### Tier 4 — Supply / plumbing (the pipes that drain the pool)
- **OFZ auction bid-to-cover** (weekly, MinFin): institutional duration appetite;
  weak auctions = duration rejected = rate fear / risk-off.
- **OFZ issuance mix** — **floater (29-series) vs fixed (26-series)**: heavy
  floaters = "high-for-longer" expectation; a shift to fixed = confidence the
  rate has peaked → **bullish duration** (issuers/MinFin locking rates).
- **Corporate issuance volume & mix** (fixed vs floater vs substituted/FX):
  primary market **drains ruble liquidity** from secondary; a surge in
  fixed-coupon corporate supply is a **late-easing** tell (issuers locking cheap
  funding). In 2024 floaters were **~53% of all bond issuance** — a textbook
  high-rate fingerprint.
- **Substituted / CNY bond issuance & redemptions:** FX-substitute demand;
  redemptions release FX-seeking money into the system.
- **Money-market fund net flows (AUM):** the "cash on the sidelines" gauge and
  the rotation **fuel/confirmation** signal (see §3).

### Tier 5 — Calendar catalysts (predictable flow pulses)
- **CBR meetings (8/yr):** the master catalyst — trade the *guidance* repricing,
  not just the decision.
- **Dividend season (May–Aug; ~60% of payouts in July):** dividend gap + a
  **reinvestment bid** that returns to equities weeks after record dates.
  Caveat: **type-C (foreign) dividends leak out of the loop** and don't reinvest.
- **Tax period (monthly ~25th–28th, quarterly peaks):** exporter **FX sales →
  ruble bid** → USD/CNYRUB dips; predictable ruble-support window.
- **FORTS quarterly expiry/roll** (3rd Fri, Mar/Jun/Sep/Dec): OI migration,
  basis dislocations, pin risk.
- **Index rebalance (quarterly):** mechanical add/delete equity flows.
- **OFZ auctions (weekly, Wed):** recurring duration supply.

---

## 5. The cross-asset rotation playbook (regime → what to favor)

Combine the **monetary regime** (rate level + direction, §3) with a **flow/risk
state** (Tier-1 turnover rotation + money-fund flows + breadth). The resulting
grid is the trading map. It maps directly onto the terminal's existing **8
strategy classes** (STRATEGY_FRAMEWORK §3).

| Regime | Favor (long) | Avoid / fund | Terminal strategy class engaged |
|---|---|---|---|
| **Tightening / peak** | floaters (29-series, corp floaters), money-market funds, short OFZ, substituted/CNY bonds (FX hedge) | long-duration OFZ, high-beta & growth equities | Carry (floater), — (defensive) |
| **Early easing + rotation** (mid-2026) | **long OFZ / RGBI (duration)**, quality corporates, *accumulate* dividend equities | cash funds (start reducing), floaters (roll to fixed) | **Carry (duration roll-down)**, Stat-arb (gov↔corp), begin Sector-rotation watch |
| **Late easing** | **equities lead** — dividend payers → cyclicals → growth/small-cap; reduce duration | money funds (drain), new floater supply | **Momentum**, **Sector rotation (RRG)**, Breakout |
| **Neutral / easy + risk-on** | growth/small-cap, high-beta; carry/short-vol; FX shorts (ruble softer) | long bonds (roll risk), defensives | Momentum, Volatility/options, Futures-arb (basis) |
| **Range / uncertain** | mean-reversion within bands; pairs; collect carry | directional bets | Swing/mean-reversion, Stat-arb |

**FX overlay (all regimes):** ruble is supported in tightening (carry) and in the
tax window (exporter sales), pressured in easing (carry erodes) and around large
import/dividend-conversion demand. Trade **CNY/RUB** as the liquid on-exchange
gauge; read USD/RUB via fixing + Si futures. Substituted/CNY bonds are the
domestic vehicle for a "ruble-down" view without leaving the contour.

---

## 6. What is genuinely different here vs a normal market (design consequences)

1. **Zero-sum rotation** → build the turnover-share engine as a *closed
   accounting identity across classes*, not isolated per-class indicators.
2. **Retail is the marginal buyer** → dividend/tax seasonality and FUTOI-`FIZ`
   positioning are **first-class signals**, and retail crowding is a **contrarian**
   input.
3. **RTS is derivative now.** RTS ≈ IMOEX × USD/RUB; it is *not* a foreign-flow
   proxy anymore. Use **IMOEX (ruble)** as the equity truth; keep RTS only as an
   FX-adjusted view.
4. **Cash is an asset class.** Money-market fund AUM/flows must be a **node in the
   rotation map**, with the cross-asset yield gap as the rotation clock.
5. **FX is CNY-centric and fragmented** → don't assume a consolidated USD tape;
   mark FX signals by venue/instrument, prefer CNY & Si futures.
6. **Substituted/CNY bonds are a distinct node** for FX exposure inside the
   ruble contour.
7. **Type-C overhang is a tail scenario** → model a "de-escalation release" both
   ways (foreign selling of equities vs. return bid) as a scenario, not a base case.
8. **The key rate is the master input** and is **not** cleanly in the MOEX API →
   the terminal needs a small, reliable **macro reference series** (rate, meeting
   calendar, RUONIA) as a manual/scraped input until the M7 macro provider lands.

---

## 7. Key risks, caveats, and failure modes

- **Reflexivity of retail flow.** Because retail *is* the market, flow signals can
  be self-fulfilling then violently mean-revert; size rotation trades by
  conviction × regime, never all-in on a single crowded signal.
- **Data gaps (see PRD §5).** OFZ auction bid-to-cover (MinFin HTML, no clean
  API), a standalone RUONIA series, corporate-issuance calendar, and money-fund
  AUM are **not** cleanly in ISS/ALGOPACK — they require scraping/manual feeds.
  The engine must degrade gracefully when a tier is missing.
- **Regime fragility / geopolitics.** A single headline can reprice every class at
  once; regime classification must be robust to gaps and must flag "event risk"
  windows (CBR day, major expiries) as no-trade or reduced-size.
- **Non-stationarity.** Post-2022 history is short; thresholds (z-scores,
  yield-gap triggers) must be **adaptive/rolling**, and any strategy must pass
  walk-forward on *both* tightening and easing windows (the framework's G3 gate).
- **Substituted-bond & FX liquidity** can vanish; basis signals need liquidity
  filters.
- **Survivorship & corporate actions:** dividend-driven equity signals must be
  corporate-action-adjusted (E1.5) or they misread gaps as flow.

---

## 8. Conclusions → the system rules (hand-off to the PRD)

The research resolves into a small set of **hard rules** the terminal should
encode. These become the backbone of `PRD_TRENDS.md`.

1. **Measure rotation as an identity.** Compute daily cross-asset turnover shares
   (equities/OFZ/corp/FX/derivatives + money-fund AUM) and z-score each vs a
   seasonally-adjusted baseline. Rotation = coordinated share shift.
2. **Anchor everything to the rate cycle.** Maintain a **regime classifier** with
   axes {rate level, rate direction} × {flow/risk state}; label the current cell
   and expose it terminal-wide as a first-class state (like breadth regime).
3. **Watch the yield gap as the rotation clock.** Track money-fund rate vs OFZ YTM
   vs equity dividend/earnings yield; the closing of the equity-vs-OFZ gap is the
   equity-leg trigger.
4. **Bonds before stocks at the turn.** Encode the sequence: duration first,
   equities second (gated by money-fund outflows + yield gap), FX/carry overlay.
5. **Read supply, not just price.** Ingest OFZ auction take-up, OFZ/corp issuance
   mix (floater↔fixed), and substituted-bond flows as leading regime tells.
6. **Respect the calendar.** Treat CBR days, dividend season, tax windows, and
   FORTS expiries as scheduled flow pulses with defined behaviors and no-trade
   guards.
7. **Gate the 8 strategy classes by regime.** Each strategy declares the regimes
   it is allowed to trade in; the runner suppresses off-regime strategies.
8. **Treat cash and the type-C overhang explicitly.** Money funds are a node;
   the overhang is a scenario. Never assume an open-market foreign bid.
9. **Adaptive thresholds + graceful degradation.** Rolling/normalized triggers;
   every signal has a fallback when its data tier is unavailable.

These rules are deliberately implementable within the existing V3 architecture —
the indicator pipeline (E3.3), screener (E3.1), alert center (E3.2), strategy
framework, and LLM brief — with one new pure-math module (`domain::flow`) and a
handful of new data feeds. The build plan is in [`PRD_TRENDS.md`](PRD_TRENDS.md).

---

## Appendix A — Grounding sources (mid-2026)

- Bank of Russia — key-rate decisions & calendar: <https://www.cbr.ru/eng/dkp/cal_mp/>, <https://www.cbr.ru/eng/press/keypr/>
- MOEX — 2024/2025 results, retail accounts, money-market growth: <https://www.moex.com/n78254>, <https://www.moex.com/n98155>
- MOEX ALGOPACK (tradestats/FUTOI/HI2/OBStats): <https://moexalgo.github.io/>
- MOEX indices (RGBI/RUCBITR/MOEXREPO): <https://www.moex.com/en/index/RGBI>, <https://www.moex.com/en/index/MOEXREPO>
- MinFin — OFZ auction results & issuance plans: <https://minfin.gov.ru/en/document?id_4=314997-domestic_government_bonds_ofz_auctions_schedule_1q_2026>
- Substituted bonds & floater share of issuance (SberCIB / Vedomosti): <https://sbercib.ru/floaters>, <https://www.vedomosti.ru/kapital/investments/articles/2024/10/29/1071306-korporatsii-investiruyut-obligatsii>
- Money-market funds / rotation (Alfa, RBC, commentary): <https://alfabank.ru/alfa-investor/t/kak-snizhenie-klyuchevoy-stavki-povliyalo-na-fondy-denezhnogo-rynka/>, <https://moex.rbc.ru/fondy-denezhnogo-rynka/>
- 2022 structural change (114-FZ/319-FZ, type-C, DR conversion, sale throttle): <https://www.morganlewis.com/pubs/2023/03/update-deadline-for-mandatory-conversion-of-russian-issuers-depositary-receipts-has-passed-whats-next>

*All figures are dated and directional; the terminal recomputes from live data.*
