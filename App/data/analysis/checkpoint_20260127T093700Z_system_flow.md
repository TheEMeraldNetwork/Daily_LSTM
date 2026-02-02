# Checkpoint — Production Flow and Engineering
Timestamp: 2026-01-27T09:37:00Z

## Scope
- Document the end-to-end system: math, data flow, engineering, scheduling, and publishing.
- Capture the exact production behavior as of this checkpoint (certified-only dashboard in production).

## Core Model Logic (R4a-candidate)
- Quality: `F10/C1 ≤ 1.03` (strictly below; production enforces `≤ 1.03` in UI and exporter).
- Direction: `F10/MA20 ≥ 1.03` (MA20 at T; computed from momentum cache or price fallback).
- Momentum: `BB %b ≥ 0.60` (at T; computed from momentum cache or price fallback).
- Persistence: 2 of last 3 days where both Direction and Momentum hold.
- Market flag: SPY 10-day return at T−1, binning: `<0`, `0-1`, `1-2`, `>2`.
- Stop-loss indication (default): lower Bollinger band at T−1 as percentage from entry (approx. `A1_(T−1)`): `stop_pct = (LB / entry) − 1`, where `LB = MA20 − 2·SD20`. Optional policy `vol_q` uses lower quantile of last-`N` daily returns.

## Data Inputs
- Predictions (per day): `data/processed/predictions/predictions_YYYYMMDD.json`
  - Keys: `symbol`, `forecast10_path_T_to_T10` (or `forecast10_T`), `forecast1_price_T`, `consensus1_price_T`, fallbacks from `estimate1` and `P0` when necessary.
- Prices cache (AlphaVantage DAILY_ADJUSTED JSON): `data/raw/prices/{SYMBOL}_daily.json`
  - Used to compute prev-day actuals, MA20, BB %b, and SPY 10d trend.
- Sentiment (Tigro dashboard): scraped “Last Week” column; loaded on-demand in exporter when `--sentiment-source tigro`.
- Momentum cache (optional): `App/data/analysis/momentum_kpis_latest.csv` (per-symbol rows with date, `bb_percent_b`, `bb_middle` (MA20), `pct_above_sma200`).

## Morning Pipeline (authoritative for production)
- Entrypoint: `scripts/elstm_morning_cron.sh` (LaunchAgent, 08:00 CET; business days only).
- Steps:
  1) Mirror iCloud project into local workspace (`rsync`) to avoid TCC/iCloud race issues under launchd.
  2) Compute previous business day (handles Monday → Friday).
  3) Preflight email (start info, prev bday, host).
  4) Attempt inference for T (`run_inference_and_backtest.py`); proceed if predictions already exist.
  5) Momentum KPI compute for symbol universe (best-effort).
  6) Export certified-only CSV for T+1 using prev-day actuals with flags:
     - `scripts/export_dashboard_for_date.py YYYY-MM-DD --only-certified --include-sentiment --sentiment-source tigro --include-momentum --append-r4a-flags`
     - Output: `App/data/certified_candidates_YYYYMMDD.csv` with columns: `symbol`, `f10_over_c1_x`, `f10_over_prev_a1_x`, `f10_T`, `a1_prev`, optional sentiment, and appended `spy_trend_bin`, `rule_4a`, `stop_loss_pct`.
  7) QC pre-publish (see QC memo).
  8) Publish: rsync `App/data/` into local clone of `Daily_LSTM` repo and push `main`.
  9) QC post-publish (see QC memo).
  10) Landing email with QC summaries and dashboard link.

## Nightly Pipeline (decoupled from production UI)
- Nightly no longer affects the production dashboard rendering.
- Nightly artifacts (comparison table, bins) can be generated and inspected offline/back-end as needed.
- Production dashboard is now certified-only to guarantee availability even if nightly artifacts are missing.

## Dashboard Rendering (Production)
- Repo: `Daily_LSTM` (GitHub Pages).
- Page: `publisher/Daily_LSTM/index.html` (now certified-only).
- Data source: `App/data/certified_candidates_YYYYMMDD.csv` (latest present).
- Columns displayed:
  - `Symbol`, `F10/C1 (%)`, `F10_T`, `A1_(T-1)`, `F10 / A1(T-1) (%)`, `Sent7d`, `Sent>0.2`, `BB %b`, `Pct>200d`, `Rule` (legacy), `Market`, `R4a`, `Stop`.
- The UI enforces certification filter (`F10/C1 ≤ 1.03`) and shows QC tallies (Market non-empty, R4a Yes/No/NA, Stops non-empty).

## File Schema Guarantees
- Certified-only headers in morning mode:
  - Base: `symbol`, `f10_over_c1_x`, `f10_over_prev_a1_x`, `f10_T`, `a1_prev`
  - Sentiment optional: `sentiment_7d`, `sentiment7d_gt_0_2`
  - Appended R4a flags: `spy_trend_bin`, `rule_4a`, `stop_loss_pct`
- Nightly mode (decoupled): may include `a1_T`, `f10_over_a1_T_x` and momentum diagnostics, but not used by prod UI.

## Scheduling
- LaunchAgent: `~/Library/LaunchAgents/com.elstm.morning.plist`
  - `StartCalendarInterval`: Weekday includes Monday (1), Time 08:00 CET.
  - Logs: `~/Library/Logs/elstm_morning_{timestamp}.log`

## Engineering Practices
- Python `venv` for inference and scripts (`/Users/davideconsiglio/ELSTM_venv312/bin/python3`).
- Keychain-backed credential retrieval (AlphaVantage, SMTP).
- Strict error handling and guardrails (see QC memo).
- Schema append-only for new summary fields to avoid breaking joins.
- Decoupled UI from nightly, making the system resilient to partial data.

## Commit and Publish Flow
- Morning job stages and commits `App/data/` into local clone of `Daily_LSTM`, pushes to `main`.
- GitHub Pages serves from `main` (no `gh-pages` branch).
- Post-publish QC confirms CSV availability on Pages before sending landing email.

## Backfill Operations
- Re-run exporter for specific dates with `--only-certified --append-r4a-flags` to populate missing certified files, then publish via `Daily_LSTM` repo.

## Notes / Next
- If re-enabling nightly analytics front-end, deploy on a separate page to keep prod certified-only.
- Consider adding a small “last file served” sticker driven by the latest certified filename.

