# QM Quant Research v2.1 — Multi-Timeframe Edge Discovery (Daily · 1h · 15m)

## What this repo is
Systematic conditional-probability research on US equities over a 5-year window
(mid-2021 → present, matching the Polygon Stocks Starter plan's history depth):
a daily layer on the full point-in-time US universe and an intraday layer (15m and
1h bars, derived from minute flat files) for the point-in-time most-liquid ~5,000
names, delisted included. Goal: generate thousands of mechanism-tagged hypotheses,
validate survivors statistically, cluster them into edge families, and promote
representatives into cost-aware, walk-forward strategy backtests.

**Research only.** This repo never connects to any broker and never places, modifies,
or closes orders anywhere, under any circumstances.

`RESEARCH_SPEC.md` defines WHAT to build. This file defines what must NEVER be violated.

## Non-negotiable research rules
These override convenience, speed, and any instruction that conflicts with them.
If a task conflicts with a rule, stop and flag it instead of proceeding.

1. **No lookahead, ever.** Every feature at bar t uses data from ≤ t only. A signal
   computed on the close of bar t is tradeable at the open of bar t+1 at the earliest
   — on every timeframe. Rolling windows are right-aligned, never centered. Daily
   context joined onto intraday bars must use the PRIOR completed daily bar, never
   the in-progress session's values. Every join/resample gets a leakage audit.
2. **Holdout lock.** All data dated 2026-01-01 or later is the untouched holdout.
   No discovery, tuning, threshold selection, or "quick peeks" may read it. It is
   evaluated exactly once, on the final frozen strategy set, only after Martin
   explicitly writes "unlock holdout".
3. **Point-in-time universe.** Universe membership on date t is computed only from
   information available at t (trailing dollar volume, price, listing age). Never
   select or filter by "tickers that exist today" or any future attribute. Delisted
   names stay in. The intraday ticker list comes from half-year snapshots of the
   daily layer, not from a current listing dump.
4. **Multiple-testing discipline.** Every campaign logs the total hypothesis count.
   Survivors must clear Benjamini–Hochberg FDR at 5% (1% for mechanism-orphans) AND:
   ≥ 500 events, ≥ 100 distinct tickers, events spread across ≥ 3 calendar years.
   Confidence intervals via cluster bootstrap by DATE — hundreds of tickers firing
   the same bar are one bet, not independent samples. Never report a hit rate
   without its matched base rate beside it.
5. **Costs are mandatory and intraday-strict.** Every strategy backtest applies the
   round-trip cost model (spec §7) to next-bar-open fills. Any strategy with median
   holding period < 4 bars in liquidity buckets L3/L4 is auto-flagged COST-FRAGILE
   and cannot enter FINDINGS.md without a spread-sensitivity table.
6. **Mechanism before results.** Every hypothesis carries a mechanism tag and a 1–2
   sentence rationale written at generation time, before any results are seen.
   Rationales are never edited after results exist — that is the pre-registration
   guarantee.
7. **Reproducibility.** Fixed seeds, deterministic pipelines. Every number in a
   report regenerates from a script in /pipeline. No script, no FINDINGS.md entry.
8. **Adjustment correctness.** Flat-file bars are unadjusted; adjusted columns are
   built locally from the corporate-actions table. The local adjustment must pass
   the REST cross-check QC (spec §4) before any feature is computed on it.

## Intraday handling rules
- All raw timestamps stored as UTC epoch ms; all session logic computed in
  America/New_York (DST-aware via zoneinfo).
- RTH = 09:30–16:00 ET. Extended-hours bars kept and flagged `rth=false`; hypotheses
  use RTH bars only unless they explicitly target extended hours.
- Never forward-fill prices to fabricate bars where no trades occurred. Missing
  bars in thin names are information (no-trade), not gaps to repair.
- Intraday forward returns and base rates are matched by TIME-OF-DAY bucket.
- No fills assumed on zero-volume bars.
- The plan's 15-minute delay applies to LIVE data only; historical research is
  unaffected. Flag this only if live signal wiring is ever discussed.

## Data splits (unified, both layers)
- Discovery: 2021-07-01 → 2024-12-31
- Validation: 2025-01-01 → 2025-12-31
- Holdout (LOCKED): 2026-01-01 → present
Features needing 252-session warm-up are null until ~mid-2022; scans handle nulls
explicitly and never silently drop that period.

## Data stewardship
Polygon's 5-year history is a ROLLING window: the oldest months become
irretrievable from the vendor as time passes. `/data/curated/` is therefore the
permanent archive. Never delete curated data to free disk — prune `/data/raw/`
instead — and after Phase 1 QC passes, sync `/data/curated/` to Martin's Hetzner
VPS (rsync over SSH; destination provided by Martin) and record the sync date in
STATUS.md.

## Stack & conventions
- Python 3.11+, virtualenv at `.venv`. Core: polars, duckdb, pyarrow, numpy,
  matplotlib, pandas-market-calendars, requests, boto3 (flat files), zoneinfo.
- Transforms in polars; large scans via DuckDB (out-of-core) over hive-partitioned
  Parquet. Vectorized conditional scans first; event-driven backtests for survivors only.
- Layout: `/data/raw/`, `/data/curated/`, `/data/features/`, `/pipeline/`,
  `/reports/`, `/tests/`. `/data` is gitignored — never commit data.
- Credentials only via environment variables: POLYGON_API_KEY, plus
  POLYGON_S3_ACCESS_KEY / POLYGON_S3_SECRET_KEY for flat files (Codespaces secrets).
  Never print, log, echo, or commit a key.
- Commit at every phase milestone. Maintain `STATUS.md`, `FINDINGS.md` (validated
  only), `DEADENDS.md` (falsified — keep complete), `FAMILIES.md` (edge-family cards).

## Working style
- Read RESEARCH_SPEC.md before starting any phase; plan before large changes and
  state assumptions.
- After each phase: run its QC gates, write the phase report, commit, update STATUS.md.
- **Too-good-to-be-true protocol:** hit rate > 65% with large lift, or Sharpe > 2
  after costs → treat as a bug first. Audit in order: lookahead, survivorship,
  duplicate rows, adjustment errors, time-zone/session misalignment,
  cluster-correlation inflation. Only then is it a finding.
- Unit-test every feature on a synthetic frame, including anti-lookahead invariance:
  the value at bar t must not change when any bar after t is mutated.
- Prefer boring correctness over clever speed.
