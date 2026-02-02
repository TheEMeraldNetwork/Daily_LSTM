# QC and Guardrails — Calculations to Publishing
Timestamp: 2026-01-27T09:37:00Z

This memo lists all quality controls across the pipeline, their intent, and failure modes.

## Calculations (Exporter) — `scripts/export_dashboard_for_date.py`
- Inputs:
  - Predictions JSON for T must exist; otherwise `exit 2` with `ERROR: missing predictions ...`.
  - Previous-day actuals map is assembled from:
    - `App/data/comparison_table_{prev}.csv` or `App/data/prev_actuals_{prev}.csv` when available.
    - Fallback from prices cache `data/raw/prices/{SYMBOL}_daily.json`; will refresh AlphaVantage if missing.
  - Sentiment (optional): Tigro scraping; errors are logged as `[DQ] sentiment: tigro_error=...` and exporter continues.
- Momentum (Direction/Momentum KPIs):
  - Prefer `App/data/analysis/momentum_kpis_latest.csv` (per date).
  - Fallback: compute MA20 and BB %b from price history for the exact date needed.
- R4a persistence:
  - Build `F10_T`, `MA20_T`, `%b_T` for T, T−1, T−2.
  - Count hits where both Direction and Momentum hold; require `hits ≥ 2`.
- Stop-loss:
  - Default `boll_lower` at T−1; returns empty if insufficient history (less than 20 closes) or invalid entry.
  - `vol_q` alternative returns lower-quantile daily move; empty if window insufficient.
- Outputs:
  - Always writes `App/data/prev_actuals_{prev}.csv` for traceability.
  - Morning mode (`--only-certified --append-r4a-flags`) writes certified-only CSV with appended `spy_trend_bin`, `rule_4a`, `stop_loss_pct`. Schema is append-only.
- DQ prints:
  - `[DQ] comparison: ...` (nightly path only).
  - `[DQ] prev_actuals: from_file_initial=... newly_filled=... nonempty=... of N`.
  - `[R4a] total=... rule_4a_true=... spy_bins={...} stops_nonempty=...`.

## Morning Cron — `scripts/elstm_morning_cron.sh`
- Business-day gating:
  - Skips Sat/Sun unless `ELSTM_FORCE_RUN=1`.
  - Computes `PREV_BDAY` by walking back days; Monday resolves to Friday.
- Preflight checks:
  - Email with start time, weekday, prev bday, host.
  - AlphaVantage GLOBAL_QUOTE probe (best-effort warning).
- Predictions presence:
  - Attempt inference; if `predictions_T.json` still missing, retry once after 60s.
  - On failure to find predictions: `exit 2`.
- Momentum compute:
  - Best-effort; does not fail the run.
- Exporter call (morning flags):
  - Writes `App/data/certified_candidates_{T}.csv`.
- Fail-closed QC before publish:
  - File must exist and be non-empty bytes; otherwise `exit 3`.
  - File must have at least 1 data row (header excluded); otherwise `exit 3`.
  - Quick scan: count rows, `spy_bin_nonempty`, `rule4_true`, `rule2_ok`. If `spy_bin_nonempty=0`, `exit 4`.
- Publish step:
  - Syncs `App/data` to local clone of `Daily_LSTM` and pushes `main`.
- Post-publish QC:
  - Re-scan published CSV: count rows, `nonempty_sma200`, `nonempty_bbpct`, `rule_ok` (legacy).
  - Verify GitHub Pages serves the newly published CSV (line count ≥ 2). If not, `exit 5`.
- Landing email:
  - Includes QC summaries and dashboard URL.

## Frontend (Production Dashboard) — `publisher/Daily_LSTM/index.html`
- Certified-only rendering:
  - Discovers latest certified dates from `App/data/certified_candidates_YYYYMMDD.csv`.
  - Renders table even if comparison tables are missing (decoupled from nightly).
- Input parsing robustness:
  - Numeric parsing uses `toNum()` to avoid treating empty strings as 0.
  - UI enforces `F10/C1 ≤ 1.03` certification filter.
- On-screen QC counters:
  - Market non-empty count, `R4a` Yes/No/NA tallies, number of non-empty stops.
- Stable schema:
  - New summary columns (`spy_trend_bin`, `rule_4a`, `stop_loss_pct`) are appended at far right to avoid breaking joins.

## Failure Modes and Exits
- `exit 2`: predictions missing after retry (morning), or exporter reports missing predictions.
- `exit 3`: certified file missing or has zero rows.
- `exit 4`: `spy_trend_bin` empty for all certified rows (signals KPI generation issue).
- `exit 5`: GitHub Pages did not yet serve the certified CSV (publishing visibility problem).
- All exits produce clear log lines; emails are sent for preflight and, if reached, landing with QC details.

## Operational Notes
- Emails use Keychain-backed credentials (or env) and include minimal logs so operators see progress.
- Publishing uses a local clone of `Daily_LSTM` to avoid iCloud git issues; pushes to `main`.
- Certified-only UI ensures users never see an empty page due to missing nightly artifacts.

