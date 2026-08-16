# Economics & metrics — why the signal doesn't trade (fee floor, hit-rate, and the metric glossary)

*Consolidated 2026-08-16 from the run.011 bias audit, the four independent
subagent verifications, and a new fee-exceedance measurement on the local
w5 stream. Companion to `run011.analysis.md` and `ALL_RUNS_ANALYSIS.md`; this
file holds the *economic* and *metric* explanation those files reference.*

## TL;DR

The predictions do **not** fail from a bug or leakage. They fail because the
ranking signal is real but **~4–7× too weak to clear the fee floor, and maker
execution inverts the selection it does have.** The router (run.011) failed
because its premise — a causally-predictable fill-timing signal — collapsed on
honest re-estimate (0.610 → 0.518 chance). All layers are structural, not code.

The one number that matters: the model concentrates a 20bp/90s move **17.9×**
above random; fees require **~114–136×**.

## 1. Why predictions failed — root cause (five layers)

1. **The signal is real but tiny in absolute terms.** Ranking IC +0.0976 (30s)
   / +0.0496 (90s), daily t +7.75 / +4.66, IC positive in all 4 folds. But an
   IC ≈ 0.10 ranks the *continuous* return well while only partially
   transferring to the *binary* 20bp-cross — 17.9× lift is what that IC buys.

2. **The economic wall (quantified).** A 20bp TP nets +13bp (taker) after fees;
   a miss costs ≈ −18bp → breakeven hit ≈ 58% (taker) / ≈ 49% (maker). The
   model lands at 7.7% hit → 17.9× concentration where 114–136× is required.
   Gap ≈ 4–7× (anchored 6.4× maker).

3. **Adverse selection at maker entry is a structural identity, not a tunable.**
   A resting buy limit fills exactly when price dips to it — i.e. when the
   up-move already failed. `hit_f 4.4% ≪ hit_m 18.0%`. No wait/δ/delay grid,
   hybrid, or entry-delay moves it (runs 008→010a). run.009 proved it is not a
   label artifact.

4. **The router premise collapsed.** run.010a's up18 FAVOURABLE OOF AUC 0.610
   was a fold-averaging artifact over degenerate day-folds; the pooled
   re-estimate → 0.518 (chance). What survives (dn18 0.613) is carried by
   regime/time proxies (`vol_ratio_1h_24h`, `lsr_z`, `minute_cos`, …) that die
   under walk-forward. The causal router then found max +0.00 bps/trig on val
   → τ=1.01 → routed 0/2,091.

5. **Statistical power / starvation.** 33 days → 4 folds × 4.7–4.8d test,
   ~148 (up) / 555 (dn) routable val triggers, ~71 favourable positives. Too
   thin to select a τ — the "vacuous fail" was pre-ordained.

## 2. Independent verification (four read-only subagent audits, 2026-08-16)

All four pillars confirmed with high confidence:

| Hypothesis | Verdict | Confidence |
|---|---|---|
| No look-ahead leakage (targets, labels, features, folds, triggers) | **Confirmed** — signal not inflated | high |
| `sim_maker` adverse selection is real, not a sim bug | **Confirmed** | high |
| Router causal & honest (τ=1.01 is a trustworthy "no val signal") | **Confirmed** | high |
| Economics arithmetic chain holds | **Confirmed** (breakeven 58.1%/48.4% literal vs 58.5%/49% stated) | high |

Surfaced by the audits (minor, non-fatal):

- `opt_est_funding_rate_sample` (Binance's `estimatedSettlePrice`) and
  `opt_remaining_time_sample` are collected but **never referenced by any
  feature** — the only forward-flavored collector field is inert (see §5).
- `favourable = filled & (net>0)` is slightly broader than "hit TP" (a
  time-stop closing +7…+20bp also counts) — the fill-timing AUCs were measured
  against an *easier* target than "reached TP", so the 0.518/0.613 are, if
  anything, optimistic.
- Router τ is tuned on *pooled* val across overlapping folds → mild optimism
  (inflation) bias, not a leak; moot since it failed anyway.
- Sim is optimistic on fill *probability* (ignores queue position, assumes fill
  on close-trade-through) yet still −9.6/−11 bps → a realistic queue model is
  strictly worse. Adverse selection is reinforced.

## 3. Fee-exceedance by horizon (new measurement)

Question: *at what horizon does a fee-sized move become common?* Measured on
the local w5 stream (13 files, 35,603 bars, 2.06 days, 2026-07-13…15,
BTC ~$62–65k). Direction-agnostic.

Diagram: `../fee_exceedance_by_horizon.png` (solid = |move| at horizon end,
dashed = fee touched at any point; green band = model has edge 30s–2min, red
band = IC≈0).

| horizon | maker 4bp (end / touch) | TP 7bp (end / touch) | taker 10bp (end / touch) |
|---|---|---|---|
| 30s | 16.6% / 21.8% | 5.0% / 6.5% | 1.8% / 2.4% |
| 60s | 29.1% / 41.7% | 11.2% / 16.1% | 4.7% / 6.8% |
| 90s | 37.9% / 56.3% | 16.9% / 25.5% | 7.9% / 12.0% |
| 2min | 44.2% / 66.2% | 22.0% / 34.3% | 10.7% / 17.0% |
| 5min | 63.7% / 91.6% | 42.0% / 67.2% | 26.6% / 44.4% |
| 10min | 74.3% / 98.5% | 56.3% / 87.3% | 41.6% / 70.0% |
| 15min | 79.1% / 99.8% | 64.1% / 95.1% | 50.0% / 82.5% |
| 1h | 89.9% / 100% | 81.9% / 100% | 74.0% / 100% |

Crossing 50% (taker 10bp): **touch ~6min, endpoint ~20min.** But at the
horizon where the model has signal (30–90s), only **2.4–12%** of intervals
even touch a 10bp round-trip.

### The reframe — this curve is NOT profitability

The exceedance curve is the **base rate** (opportunity rate): direction-agnostic,
model-free. Profitability additionally requires predicting *direction*, and that
requirement pulls in the opposite direction:

- Move-frequency **rises** with horizon (√-scaling) — 50% at ~6min.
- Predictive edge **dies** with horizon — IC t: 30s +12.9 → 2min +1.6 → 10min ≈0.

**Short horizon = signal but fee-move rare (~2% at 30s); long horizon = fee-move
common but zero signal.** There is no horizon where both hold — that absence *is*
the wall the project has been hitting; it is not a horizon not yet tried.

Why the project's numbers look far rarer: the *directional, first-touch*
base rate at 90s/20bp is **0.43%**, ~30× rarer than the direction-agnostic
"10bp touch" (12%). Three compounding factors: 20bp vs 10bp, first-touch
(stricter), and directional (≈half the two-sided rate).

## 4. Directional opportunity counts (prediction-capable horizon)

Directional first-touch (the notebook's own `lup/ldn` label logic), per-day
opportunities. Prediction-capable horizon = 30s–2min (IC significant, peaks
30s, dead by 2min).

| horizon | θ=10bp (taker fee) | θ=20bp (project target) |
|---|---|---|
| 30s | ~416/day | ~52/day |
| 90s | ~2,072/day | ~326/day |
| 2min | ~2,940/day | ~536/day |

**Volatility caveat — this 2-day slice overstates the long-run rate.** The
sample gives lup_18 @20bp = **1.16%**, but the 35-day run's sanity anchor is
**0.43%** (this July 13–15 window moved BTC ~4.7% in 2 days ≈ 2.4× average
vol). The documented 33-day rate ⇒ **~74 up + ~62 dn ≈ 136/day** at 90s/20bp.

**Samples are not the bottleneck.** At θ=10bp fee-moves are abundant even at
the signal horizon (~400/day at 30s, ~2,000/day at 90s → ~13k–66k over 33
days). At θ=20bp it is ~4,500 directional events over 33 days — enough to
train the heads, marginal for the *router* (~15% favourable ⇒ ~675 examples).
The failure is edge-strength (17.9× vs 114×) and adverse selection, not sample
scarcity. More data sharpens the IC estimate; it does not raise its ceiling.

## 5. The two collected-but-unused features

`opt_est_funding_rate_sample` + `opt_remaining_time_sample` (from
`poll_premium_index`). Assessed 2026-08-16: **low value, wrong direction.**

- **`est_funding_rate_sample` is misnamed — a price, not a rate.**
  `binance_live_orderbook_v4.py:556`: `est_funding_rate = float(data["estimatedSettlePrice"])`.
  It holds Binance's estimated settle price ≈ the futures **mark** price. It is
  ~redundant with `mark_price_sample` (also collected) and with the
  spot↔futures `basis_z_4h` the model already uses. Adding it = a second copy
  of mark price, a slow regime feature of exactly the class that dies under
  walk-forward.
- **`remaining_time_sample` is a funding-cycle sawtooth = a time feature.**
  `:559`: `remaining_time = next_funding_time/1000 − now`. Already encoded by
  `near_funding` (binary <15min) + `minute_sin/cos`/`hour_sin/cos`. Adds only
  finer resolution to an existing time encoding — and time-of-day/funding
  features are precisely what carried the *fake* dn18 0.613 that the causal
  router failed on.

Neither touches the three failure causes (concentration shortfall, adverse
selection, fill-timing). Adding them risks re-inflating the regime-proxy
artifact run.011 just retired.

## 6. Sub-second order-flow — no special equipment

Sub-second capture is a **software + storage** change, not hardware. The
collector already ingests the finest public Binance feed (`@depth@100ms` +
`@trade`) and already computes per-message OFI / add-cancel flow in
`apply_depth_update` — then aggregates it into 5s bars and discards the event
sequence. Sub-second capture = persist the per-event stream (depth-diff: price,
qty, side, U/u, local ts; trade: price, qty, aggressor side, t).

Considerations, all commodity: **location** (Greece → ~50–150ms; irrelevant
for *capturing* since U/u IDs give canonical order, matters only for *live*
trading — move to AWS Tokyo `ap-northeast-1` near USDT-M futures);
**storage** (GBs/day); **CPU** (same async design, keep the writer off the
receive path). There is no public sub-100ms/tick feed — the finest public
granularity is the 100ms depth diff + per-trade stream already in use.

## 7. Metric glossary (plain words)

### 7.1 IC — Information Coefficient

Rank correlation (Spearman) between the model's score and the actual outcome.
Measures **ordering**, not magnitude: does a higher score mean a bigger move?
+1 perfect, 0 random, −1 backwards. This project: +0.0976 (30s) / +0.0496 (90s)
— weak but real. A strong *ranking* of the continuous return only partially
transfers to the *binary* θ-cross, which is why IC ≈ 0.10 buys 17.9× lift, not
114×.

### 7.2 Daily t-stat

The t-statistic of the **mean daily IC** — a significance test of "is the IC
reliably positive day after day, or luck?". Computed by: IC per test day →
`t = mean(daily IC) / (std(daily IC) / √n_days)`. Per-day (not pooled) because
bars within a day are correlated; each day is one independent observation.
|t| > 2 = significant. Project: h6 **+7.75**, h18 **+4.66** over 20 days.
t answers *"does the signal exist?"* (yes, strongly); it says nothing about
*"is it big enough to clear fees?"* — a signal can be significant (t=7.75)
and economically worthless (17.9× vs 114×).

### 7.3 Breakeven hit rate

The fraction of trades that must win for average P&L = 0. Forced by payoff +
fees, independent of the model.

```
EV = hit·(win) − (1−hit)·(loss)  →  hit = loss/(win+loss)
```

| | win | loss | breakeven |
|---|---|---|---|
| taker (10bp RT) | +13bp (20−7) | −18bp | **58.1%** (doc: 58.5%) |
| maker (4bp RT) | +16bp (20−4) | −15bp | **48.4%** (doc: 49%) |

Above 50% because the **win is capped** (TP limit) while the **loss is not**
(gap-through stop), and fees tax both branches. A 50% coin flip loses −2.5bp.

### 7.4 Concentration / lift — why 115×

`lift = hit_rate / base_rate`. Base rate of a 20bp directional 90s move is
0.43%. Required concentration = breakeven-hit / base-rate = **49% / 0.43% ≈
114×** (maker) or 58.5% / 0.43% ≈ 136× (taker). Achieved 17.9× (→ 7.7% hit),
31× unanimity (→ 13.2%). Gap ≈ 4–7×, anchored 6.4×.

Worked example, 10,000 bars (base 0.43% ⇒ ~43 winners):

| | random | model today | fees require |
|---|---|---|---|
| bars picked (top 0.1%) | — | 10 | 10 |
| winners among picks | 0.04 | 0.77 (7.7%) | 4.9 (49%) |
| lift | 1× | 17.9× | 114× |

### 7.5 Win/loss +13/−18 — why "any move" still collapses to two numbers

"+13/−18" are **outcomes of the exit rules**, not a claim about move size.
Every path maps to one of three exits: **TP** (touch +20 → capped +13, always),
**SL** (touch −20 → taker at breaching close, avg −30, can gap worse),
**time-stop** (90s elapsed → taker at close, avg −10). Non-hits blend
`0.4×(−30) + 0.6×(−10) = −18`.

The asymmetry is the point: win **capped** at +13 (TP throws away anything past
+20); loss **uncapped** (gap-through). "Any move can happen" is exactly why the
loss is a spread while the win is a ceiling.

Why not let winners run? (a) winners don't run (~90% of touches survive 90s but
~10% survive 10min — mean reversion); (b) edge dead by 2min; (c) endpoint
variant scores −9.4bp ≈ TP/SL −10bp — the big-move tail is symmetric and
unpredictable. run008's ceiling: even an omniscient exit ≈ 0bp. Exits
redistribute path-capture; they cannot add edge.
