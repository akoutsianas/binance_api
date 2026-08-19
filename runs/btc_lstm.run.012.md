# run.012 — build notes (v5 event-level features)

Companion to `runs/btc_lstm.run.012.ipynb`. Written 2026-08-19, before any run.012 execution.

## What it is

run.011 (5s bars, causal maker/skip router, gated) + the v5 collector's raw
event stream as a new feature block. One variable per run: **the feature set**.
Bar grid, horizons, targets, folds, model, training loop, calibration, trigger
rules, sims, router, and the gate are byte-identical to run.011.

## Why

Runs 008–011 closed the bar-level programme: pooled OOF AUC ~0.51–0.52 on every
observable bar-level feature; the run.011 router found max +0.00 bps/trig. The
v5 collector (`binance_live_orderbook_v5.py`) docstring named the remaining
lever: **event-level order flow** — sub-bar timing and per-message granularity
that 5s bars average away. v5 writes verbatim `@depth@100ms` + `@trade` to
`events.<sd|st|fd|ft>.w5.<YYMMDD_HH>.jsonl.gz` (hourly UTC rotation, append-mode).
run.012 is the first run that reads it.

## Data plumbing

- Bars: unchanged (`35day_btc_data.tar.gz` → `/root/btc_data`, w5 files only).
- Events: new tar `btc_events_w5.tar.gz` → `/root/btc_events`. Build on the
  server: `cd events && tar czf ../btc_events_w5.tar.gz . && cd ..`
  (the loader globs `**/events.<tag>.w5.*.jsonl.gz` recursively, so either tar
  layout works).
- Config additions (cell 3): `EVENTS_DIR='/root/btc_events'`,
  `EVENTS_ERA_ONLY=True`, `MIN_EVENTS_DAYS=6`.
- `orjson` added to the cell-1 pip install (5–10× parse speed; falls back to
  stdlib `json`).

## Loader (cell 4: `load_event_features`)

- Streams parsed: `fd` (future depth diffs + resync snapshots), `ft` (future
  trades), `st` (spot trades). `sd` (spot depth) skipped on purpose: marginal
  value over `fd`, and skipping halves parse time.
- Bar key = `floor(r / W) * W` (event receive epoch) — the same machine clock
  as the CSV `spot_timestamp`, so the merge in `prepare()` is tz-free. Never
  parse datetime strings for alignment (v5 docstring rule).
- Replay dedupe per stream: skip depth msg if `u <= last_u`, trade if
  `t <= last_t`. Sequence gaps on fd (`U > last_u + 1`) count into
  `ev_fd_missed`. Snapshot records (`k="snapshot"`) count into `ev_fd_resync`
  and reset the dedupe chain.
- Unparseable lines (truncated tail after a crash) are skipped and counted.
- Per-bar accumulators are built in one pass; per-bar numpy post-processing
  (gaps, weighted times, sweep two-pointer) runs after.

## The 20 ev_* features (per 5s bar, strictly intra-bar → causal)

Depth (`fd`): `ev_fd_nmsg_z` (msg count, log-z vs 4h), `ev_fd_levels_z`
(levels touched/msg), `ev_fd_maxq_z` (max single-level qty), `ev_fd_pull_share`
(qty=0 removals / levels), `ev_fd_gmin_z` (min inter-msg gap ms),
`ev_fd_last1s` (share of msgs in the bar's last 1s), `ev_fd_time_ask_imb`
((mean t of ask-touching − bid-touching msgs)/W), `ev_fd_missed` (log1p,
clipped), `ev_fd_resync` (raw count).

Trades (`ft`): `ev_ft_ntr_z`, `ev_ft_gmin_z`, `ev_ft_time_imb` ((qty-weighted
mean t of SELLS − BUYS)/W; >0 = sells later), `ev_ft_last1s`, `ev_ft_first1s`,
`ev_ft_maxpos` (time position of the largest trade, 0..1), `ev_ft_sweep1s`
(max same-side qty in any 1s WALL-CLOCK window / bar qty — the bar schema's
run-length proxy is trade-count-based), `ev_ft_maxrel_z` (max/median trade
size, log-z).

Trades (`st`): `ev_st_ntr_z`, `ev_st_time_imb`, `ev_st_last1s` (spot lead-lag
at sub-bar resolution).

All engineered in `prepare()` using the run.006 conventions (4h time-based
rolling z, clip ±8; bounded shares raw).

## Era + missing-data rules

- `ev_coverage` marker (1 where any fd event covers the bar) is computed BEFORE
  sentinel fills; the cell-5 events-era slice keys on it. It never enters
  `ALL_FEATURES`.
- Sentinel fills (whole frame; pre-era rows are sliced away): counts/shares → 0,
  min-gap → bar length in ms ("as quiet as measurable"), maxrel → 1 (log → 0).
  The only NaNs left are pre-era rows.
- Hole diagnostic: bars where the bar CSV's `future_depth_msg_count` > 0 but no
  event file covers the bar are counted and printed before slicing.
- `EVENTS_ERA_ONLY` aborts with a clear message when there are no event files,
  no time overlap with the bars tar, or < `MIN_EVENTS_DAYS` of coverage.
  `EVENTS_ERA_ONLY=False` reproduces run.011 without ev_* features.
- Feature computation order is unchanged: features/targets on the FULL frame
  first (24h rollers warm at the boundary), then schema-3 slice, then events
  slice.

## Gate + pre-registered reads

Gate identical to run.011: A∧B∧C on either side (routed maker sim > 0 with
day-bootstrap CI excluding 0; routed lift ≥ 3× pooled and ≥ 2× in ≥ 3/4 folds;
routed fill ≥ FILL_MIN and per-fold sim > 0 in ≥ 3/4 folds). D info-only.

Pre-registered diagnostics (the point of the run):

- cell 10D FAVOURABLE pooled OOF AUC **with** ev_* features vs run.011's
  bar-level benchmark (0.610 on the old window).
- cell 10D univariate top-6: does any `ev_*` feature appear at all?

Read-out: PASS → carry the ev_* block into the long-window run. AUC < 0.58
with events → adverse selection is not observable in order flow at any captured
resolution; remaining levers are queue-position modelling / execution mechanics,
not prediction. Abort at the era guard → rerun when ≥ `MIN_EVENTS_DAYS` exist.
Below ~2 weeks of captures the run is a pipeline check, not a verdict (the fold
printer warns on <7-day test folds).

## Window comparability caveat

run.011's reference numbers (up18 maker −9.62 / dn18 −10.96 bps) are on the OLD
35-day bar-only window — NOT comparable to run.012's events-era window. Cell
10R's trailing print was patched to say so.

## Verification (2026-08-19, pre-execution)

- All 18 code cells `ast.parse` clean; notebook outputs cleared; no stale
  `run011` references.
- Loader: 29 hand-computed checks on synthetic v5-format files pass — counts,
  levels, maxq, pull share, min gaps, first/last-1s shares, max-trade position,
  sweep share, max/median ratio, buy/sell timing asymmetry, missed-sequence
  counting, resync counting, replay dedupe (depth `u` and trade `t`), quiet-bar
  absence, single-msg gap sentinel.
- `prepare()` merge: epoch-keyed merge aligns the right bars; `ev_coverage`
  1/0 correct on covered/quiet bars; sentinel fills correct; raw ev columns
  fully populated.
- Z-score path: 1000-bar synthetic run → 100% finite ev_*_z after the 720-bar
  warmup, 100% merge coverage.
- One build-time bug found and fixed by the tests (loader summary print iterated
  an int); one test-expectation error (ev_fd_time_ask_imb hand-calc forgot msg1
  touches bids) fixed in the test, not the code.

## Build tooling

`run.012` was generated programmatically from run.011 (surgical cell edits,
outputs cleared) — builder and tests live in `/tmp/opencode/build_run012.py`,
`/tmp/opencode/test_run012_events.py` (ephemeral; re-derive from git if needed).
