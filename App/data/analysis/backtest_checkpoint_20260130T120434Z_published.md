### Milestone: Production Backtesting Tab published and linked (2026-01-30 12:04:34Z)

- **What’s live**
  - **Backtesting page**: `https://theemeraldnetwork.github.io/Daily_LSTM/backtest_prod_flat.html`
  - **Data paths (served via GitHub Pages)**:
    - `https://theemeraldnetwork.github.io/Daily_LSTM/App/data/backtest/prod_backtest_flat_all_1dp.csv`
    - `https://theemeraldnetwork.github.io/Daily_LSTM/App/data/backtest/prod_backtest_meta.json`
  - **Master dashboard link**: added “Backtesting” link in header of `index.html` → `./backtest_prod_flat.html`

- **Repository updates**
  - Added `.nojekyll` at repo root so CSV/JSON are served directly.
  - Published backtest assets under `App/data/backtest/`.
  - Updated `backtest_prod_flat.html` to read from `App/data/backtest/` and display `updated_at_cet`, `matured_last_date`, `weekly_window`.
  - Updated `index.html` header to include “Backtesting” link.

- **Automation changes**
  - `scripts/elstm_morning_cron.sh`:
    - After certified publish, runs backtest builder, then publishes artifacts to `App/data/backtest/`.
    - Replaced mock URLs in publish/verify steps with `App/data/backtest/...`.
    - Verifies page and data via HTTP (fail-closed if any 4xx/5xx).

- **Verification (HTTP)**
  - `backtest_prod_flat.html`: 200
  - `App/data/backtest/prod_backtest_flat_all_1dp.csv`: 200
  - `App/data/backtest/prod_backtest_meta.json`: 200
  - Master `index.html` renders Backtesting link (visually confirmed).

- **Data status**
  - Ledger written to `Daily_LSTM/backtest_db/prod_backtest_ledger.csv`.
  - QC backfill summary at `Daily_LSTM/backtest_db/qc_backfill_summary.csv`.
  - Aggregates refreshed under `Daily_LSTM/mock/` and published to `App/data/backtest/` with 1-decimal formatting.

- **Pending/Next**
  - `exporter-trade-ledger`: extend exporter to maintain the production trade ledger directly from certified CSVs (idempotent, append-only).
  - `summaries-wk-mo-ytd`: add weekly/monthly/YTD summaries and KPIs (N_trades_closed, %win/%loss, avg/median/mean deltas, max drawdown, Sharpe-like).
  - `qc-guards-backtest`: expand QC (completeness, closed-vs-open validation, publication guards).
  - `cron-integration-backtest`: finalize integrated daily flow once additional summaries/QC are in.

- **Notes**
  - All times default to CET for day labels; timestamp above is UTC (Z).
  - The dashboard remains certified-only for robustness; backtest is decoupled and self-contained.*** End Patch ***!
