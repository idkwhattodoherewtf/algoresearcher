# RESEARCH_SPEC v2.1 — Multi-Timeframe Edge Discovery, US Equities (Daily · 1h · 15m)

Version 2.1 · Owner: Martin · Executing agent: Claude Code
Binding rules live in CLAUDE.md. This document defines what to build and in what order.

Plan constraints this version is designed around (Polygon Stocks Starter):
5 years historical data (ROLLING — archive on first pull), unlimited API calls,
flat files (S3 bulk downloads), minute aggregates, corporate actions, reference
data. Live data is 15-min delayed — irrelevant for historical research.

## 0. Mission
Generate and test thousands of mechanism-tagged conditional hypotheses of the form

    P( forward return over horizon h clears threshold x | condition set C )

against matched base rates, across three timeframes over a single 5-year window
(2021-07 → present): a DAILY layer on the full point-in-time US universe and an
INTRADAY layer (15m + 1h, derived locally from minute flat files) on the
point-in-time most-liquid ~5,000 names including later-delisted ones. Survivors are
clustered into EDGE FAMILIES (shared mechanism + overlapping events + correlated
P&L); family representatives are promoted into cost-aware walk-forward backtests.
Output: a ranked, validated, mechanism-explained edge atlas plus an honest graveyard.

## 1. Phase map
- Phase 0A — Bulk ingestion: day-aggregate flat files (all tickers, 2021-07 → today)
- Phase 0B — Corporate actions & local adjustment (splits/dividends → factor table)
- Phase 0C — Universe snapshots + minute flat files → resampled 15m/1h intraday layer
- Phase 1  — Consolidated QC & archive sync
- Phase 2  — Feature library (~150–250 features incl. intraday + cross-TF)
- Phase 3  — Base-rate atlas (time-of-day matched for intraday)
- Phase 4  — Hypothesis engine & conditional scans (mechanism-tagged)
- Phase 5  — Edge-family clustering (winner correlation → market logic)
- Phase 6  — Survivor strategy backtests (walk-forward, costs)
- Phase 7  — Robustness & final reporting

Every phase ends with: QC gates pass → `/reports/phaseN_<name>.md` → commit → STATUS.md.

## 2. Phase 0A — Day-aggregate flat files (daily backbone)
Flat files are the primary ingestion path: one compressed file per trading day
containing bars for EVERY US ticker that traded — later-delisted names included,
which is what makes the universe point-in-time.

- Access: S3-compatible endpoint `https://files.polygon.io`, credentials from the
  Polygon dashboard as env vars (POLYGON_S3_ACCESS_KEY / POLYGON_S3_SECRET_KEY),
  client boto3 (or aws-cli). **Discover exact prefixes by listing the bucket** —
  expect day aggregates and minute aggregates under a `us_stocks_sip/`-style prefix;
  do not hardcode paths without verifying against the listing.
- Script `/pipeline/ingest_flatfiles_daily.py`: download all day-aggregate files
  2021-07-01 → today (~1,250 files), resumable (skip existing), failures logged
  with `--retry-failed`.
- Convert to `/data/raw/daily/year=YYYY/date=*.parquet` keeping ticker, o, h, l, c,
  v, vwap/transactions if present. Flat-file bars are UNADJUSTED — adjustment is
  Phase 0B's job.
- Fallback path (plan has unlimited REST calls): grouped-daily endpoint
  `GET /v2/aggs/grouped/locale/us/market/stocks/{date}` per session, if S3 setup
  stalls. Same layout, same downstream.

QC: file count == XNYS sessions; per-date row-count sanity band (~3,000–12,000);
spot-check AAPL, SPY, one high-split name, one delisted name against external charts.

## 3. Phase 0B — Corporate actions & local adjustment
- Pull splits and dividends for all tickers via the reference/corporate-actions
  API (unlimited calls; paginate). Persist `/data/curated/corporate_actions/`.
- Build a per-(ticker, date) cumulative adjustment-factor table (split factors for
  price+volume; optional dividend factors for total-return columns).
- Produce `/data/curated/daily/` (hive-partitioned by year), one row per
  (date, ticker):
      date, ticker, o, h, l, c, v, o_adj, h_adj, l_adj, c_adj, v_adj,
      vwap, n_trades, dollar_vol (unadjusted c*v)
- Hygiene: drop non-positive prices / negative volume; enforce h ≥ max(o,c),
  l ≤ min(o,c) (log + drop violations); de-dup (date,ticker) keep last; exclude
  obvious non-common-stock instruments by documented regex (tickers with '.', '-',
  identifiable unit/warrant/right suffixes, test tickers) — conservative, since the
  universe filter does most exclusion.

**Adjustment cross-check QC (mandatory before Phase 2):** for 25 tickers spanning
several splits (plus 5 no-split controls), fetch REST daily aggregates with
`adjusted=true` and assert the locally adjusted series matches within tolerance
over the full window. Failures block the pipeline.

## 4. Phase 0C — Universe snapshots + intraday layer from minute flat files
**Universe list.** From curated daily data, snapshots every June 30 and December 31
starting 2021-12-31: top 3,000 tickers by trailing 60d median dollar volume, price
≥ $2, listed ≥ 60 sessions — all as of the snapshot date. Union of snapshots =
intraday ticker list (~4,500–6,000 names incl. delisted) with per-ticker active
windows. Persist the list.

**Minute flat files → 15m/1h.** Stream one day at a time (~1,250 files, tens of MB
compressed each):
1. Download the day's minute-aggregate flat file.
2. Filter to the universe list (respecting active windows).
3. Convert timestamps to America/New_York; tag `rth` (09:30–16:00).
4. Resample to 15m and 1h bars: o=first, h=max, l=min, c=last, v=sum,
   n_trades=sum, vwap=volume-weighted; bars aligned to :00/:15/:30/:45 (15m) and
   :00 (1h) in NY time; no bar emitted where no trades occurred.
5. Apply adjustment factors → adjusted columns alongside unadjusted.
6. Append to `/data/curated/intraday/tf={15m,1h}/year=YYYY/` and DELETE the raw
   minute file. Disk high-water mark stays low; only resampled output is kept.
- Resumable per date; a manifest records processed dates.
- 1-minute bars are NOT retained in v2.1 (disk); revisit only if a validated
  finding needs sub-15m resolution, and then only for a small ticker subset.

QC: per-day resampled bar counts vs sessions for 50 liquid names (~26 RTH 15m bars);
timestamp monotonicity; DST-transition days verified (March/November); cross-layer
check on a 50-ticker sample — sum of RTH 15m volume ≈ daily volume, RTH 15m
high/low envelope matches daily high/low within tolerance.

## 5. Phase 1 — Consolidated QC & archive
Run the full QC suite across layers; produce the coverage report (tickers per date,
bars per ticker, null maps). Then execute the CLAUDE.md stewardship rule: rsync
`/data/curated/` to the Hetzner VPS (Martin provides destination), record sync date
in STATUS.md. The vendor window is rolling; this archive is the permanent copy.

## 6. Phase 2 — Feature library
All right-aligned, per ticker, unit-tested incl. anti-lookahead invariance. Daily
families (computed on adjusted series unless noted):

A. **Trend / RS** — ret over {5,10,21,63,126,252}; RS vs SPY (SPY is in the data);
cross-sectional RS percentile per date; distance from MA(10/20/50/200) %; MA slopes;
MA-stack flags; days since 252d high; % below 252d high.
B. **Compression / tight bars** — NR7; NR4/NR7 count in 10d; range % z-score;
ATR(14)/c percentile vs own trailing 252d; BB(20,2) width percentile; consecutive
inside days; max close-to-close move 5/10d; coil score = 63d range / 252d range.
C. **Volume character** — RVOL = v/mean(v,50); dry-up = mean(v,5)/mean(v,50); 21d
up/down-volume ratio; 20d median dollar volume + liquidity bucket; OBV slope.
D. **Candle / position** — closing range; gap %; |gap|>2% count in 21d; wick ratios.
E. **Structure events** — undercut-and-reclaim; inside week; higher-low count;
new 21/63/252d closing-high flags; distance to pivot (% below 21d high).
F. **Vol regime** — realized vol 21d percentile; vol-of-vol; ATR trend.
G. **Context** — SPY regime tags (above/below 200MA; realized-vol tercile).
H. **Calendar** — DOW, month, month-end (last 3 sessions), quarter-end, January.

Intraday families:
I. **Time-of-day** — opening-range (5/15/30-min) levels + position/breakout state;
first-hour volume share vs 20d same-window norm; lunch drift; last-hour return and
volume share; bar-of-day index.
J. **Gap dynamics** — gap % vs prior close; fill progress by 10:00/11:00/close;
gap out-of-compression vs in-extension (joined daily coil score).
K. **VWAP** — distance to session VWAP; crosses; reclaim/reject events; anchored
VWAP from most recent gap day, distance and hold count.
L. **Time-matched RVOL** — cumulative session volume vs same-time-of-day 20d
average (naive full-day divisor prohibited); per-bar RVOL.
M. **Cross-TF context (the differentiator)** — PRIOR-day daily features joined onto
every intraday bar: coil score, RS percentile, distance-to-pivot, dry-up, MA-stack,
days-since-252d-high, liquidity bucket, regime tag.
N. **Bar microstructure proxies** — 15m range percentile vs trailing same-slot;
consecutive same-direction bars; wick asymmetry; n_trades per bar;
volume-without-progress flag.

All indicators (SMA/EMA/RSI/MACD and beyond) are computed locally here — never via
Polygon's indicator endpoints (4-indicator ceiling, per-call overhead, untestable
for lookahead).

Warm-up: 252-session features are null until ~mid-2022; scans handle nulls
explicitly.

## 7. Cost model
Per-side cost by 20d median dollar-volume bucket, applied at next-bar-open fills:
L1 ≥ $50M → 5 bps · L2 $10–50M → 12 bps · L3 $2–10M → 30 bps · L4 < $2M → 70 bps.
Intraday: costs ×1.5 for fills in the first 15 minutes; no fills on zero-volume
bars; median hold < 4 bars in L3/L4 ⇒ COST-FRAGILE flag requiring a
spread-sensitivity table (0.5×, 1×, 2× assumed costs) before promotion.
Commissions configurable, default 0.

## 8. Phase 3 — Base-rate atlas
Forward return: signal on bar t (close) → entry at open of bar t+1:
    fwd_ret(h) = c(t+h) / o(t+1) − 1
Horizons: daily h ∈ {1,3,5,10,21}; 1h h ∈ {1,2,7,14} bars; 15m h ∈ {1,2,4,8,26}
bars, plus "to same-day close" and "to next-day open" (overnight isolated as its
own horizon). Unconditional distributions sliced by year × liquidity bucket ×
regime × (intraday) time-of-day bucket {09:30–10:30, 10:30–12:00, 12:00–14:00,
14:00–15:30, 15:30–16:00}. **All lift reported vs the MATCHED slice.**

## 9. Phase 4 — Hypothesis engine
- Atoms: each feature × threshold set (percentile cuts ≤10/≤25/≥75/≥90 or flags);
  expect 600–1,200 atoms.
- Hypotheses: cross-family pairs first; a dedicated CROSS-TIMEFRAME campaign
  (daily-context atom × intraday atom) — the least-mined shelf; triples only from
  top-500 pairs. 5k–50k per campaign, exact count logged. Long and short scored.
- Mechanism tag + 1–2 sentence rationale at generation time (taxonomy: MOM-UNDER,
  REV-OVER, LIQ, EXEC, CONSTR, VOL-REG, INFO). Orphans face FDR 1%.
- Evaluation on DISCOVERY only: N events, distinct tickers, per-year counts,
  mean/median fwd_ret per horizon, hit rate vs matched base rate, lift, cluster
  bootstrap CIs by date.
- Ranking: FDR-adjusted lift → economic size net of §7 costs → breadth.
- Outputs: `/reports/campaign_<name>.md` + parquet of ALL results incl. failures;
  falsified folklore → DEADENDS.md.

## 10. Phase 5 — Edge-family clustering (winners → market logic)
1. Event-overlap matrix (Jaccard on (date, ticker[, bar]) event sets).
2. P&L correlation matrix (daily-aggregated event returns per survivor).
3. Hierarchical clustering over combined distance → EDGE FAMILIES.
4. Per family, a card in FAMILIES.md: dominant mechanism, members, single best
   representative, regime map, liquidity range surviving costs, and a
   plain-language paragraph of the market logic ("who is on the other side and why
   do they keep paying?").
5. Independence score: family-vs-family P&L correlations, and each family vs SPY
   and a simple momentum factor. The portfolio draws one representative per family.
No-family, no-mechanism winners are parked as probable mining artifacts unless they
re-survive validation untouched.

## 11. Phase 6 — Survivor strategies
Family representatives (~15–40) become event-driven bar-level backtests: entry next
bar open; exit menu = time stop, OR-low/structure stop, 2×ATR stop, VWAP-loss stop,
EOD-flat vs overnight-hold (both, reported separately); equal weight, capped
concurrency, per-name risk cap; §7 costs. Walk-forward: parameters frozen on
discovery, evaluated once on validation. Metrics: CAGR, maxDD, Sharpe, profit
factor, exposure, turnover, per-year table, cost-sensitivity row.
Parameter-neighborhood: ±1 step on every threshold; knife-edges rejected.

## 12. Phase 7 — Robustness & reporting
Regime slices within the window: 2021 H2 topping phase, 2022 bear, 2023–24 bull,
2025, 2026 H1. Liquidity buckets, time-of-day, long vs short, decay trend
2021→2025. Explicitly test the classic folklore (unconditional ORB, RSI<30 bounce,
gap-fill, VWAP mean-revert) and document outcomes in DEADENDS.md.
**Stated limitation in every summary:** the window contains one full bear-recovery
cycle but no 2018 Q4 or 2020-crash regime; treat regime-sensitive findings
accordingly. Deliverables: FINDINGS.md, DEADENDS.md, FAMILIES.md, frozen strategy
set awaiting Martin's explicit "unlock holdout".

## 13. Seed intraday hypothesis families (expand creatively beyond these)
1. ORB conditioned on daily coil: 30-min opening-range break where PRIOR daily coil
   score is bottom-decile vs ORB in extended names. (MOM-UNDER)
2. Gap-and-go vs gap-and-fade: same |gap|, opposite daily context — out of
   compression vs into extension. (INFO / REV-OVER)
3. VWAP reclaim after ≥ 8 bars below, top-decile daily-RS names only. (EXEC)
4. Time-matched RVOL surge with flat price (absorption) → next 4–8 bars. (LIQ/EXEC)
5. First 15m NR bar of the day after ≥ 3 daily inside/tight days. (MOM-UNDER)
6. Premarket-high break in the first hour, conditioned on daily distance-to-pivot.
   (MOM-UNDER)
7. 10:00–10:30 reversal: fade of the first-30-min move conditional on gap size and
   daily trend alignment. (REV-OVER)
8. Last-hour ignition: 14:30+ new session high on time-matched RVOL ≥ 2 in daily
   leaders → close / next open. (EXEC/CONSTR)
9. Anchored-VWAP-from-gap-day defended ≥ 3 touches, then break — both directions.
   (EXEC)
10. Overnight-hold edge: EP-like daily signature (63d closing high, RVOL ≥ 2,
    closing range ≥ 0.8) → buy last 15m, sell next open vs next close. (INFO)
11. OPEX-Friday behavior of L1 names near round-number strikes vs normal Fridays.
    (CONSTR)
12. Month-end last-2-days drift in top-RS vs bottom-RS deciles, and its intraday
    timing. (CONSTR)
13. ≥ 5 consecutive same-direction 15m bars inside bottom-decile daily coil —
    continuation or exhaustion, per liquidity bucket. (MOM vs REV)
14. Down-gap absorbed by 11:00 (recross of open, morning closing range ≥ 0.7) in
    names above the daily 200MA. (REV-OVER)
15. Volatility-regime gate on all of the above: SPY 200MA side × realized-vol
    tercile — an edge alive in one regime only is a finding, not a failure. (VOL-REG)

## 14. Compute plan
Stream-process minute flat files one day at a time (download → filter → resample →
delete); keep only 15m/1h curated output (~5–15 GB Parquet incl. daily layer).
Vectorized DuckDB scans partitioned by month for Phase 4; event-driven backtests
only for Phase 6 survivors. Recommended Codespace: 8-core / 32GB RAM / 64GB disk;
monitor `df -h` during Phase 0C.

## 15. Definition of done (v2.1)
Phases 0–7 complete; curated archive synced to the VPS; ≥ 10,000 mechanism-tagged
hypotheses tested and logged across both layers; edge families documented with
market-logic cards; representatives walk-forward validated with cost sensitivity;
every report reproducible; holdout untouched and awaiting explicit unlock.
