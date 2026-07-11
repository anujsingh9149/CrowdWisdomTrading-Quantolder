# CrowdWisdomTrading-Quantolder
# CrowdWisdomTrading Quant: Strategy Permutation Predictor

A modular, production-style, and reproducible machine learning pipeline designed to predict the optimal trading strategy permutation for future periods. The system integrates historical trading execution logs with macroeconomic events, cleans and aggregates transaction data, engineers advanced temporal and market context features, and runs chronological Walk-Forward Validation using XGBoost.

---

## Folder Structure

```
crowdwisdom_quant/
│
├── data/
│   ├── raw/                  # Raw historical execution logs (CSV)
│   ├── processed/            # Feature-engineered dataset (CSV)
│   └── external/             # Cached macroeconomic raw scrapes
│
├── database/
│   ├── trading.db            # SQLite database containing macro and trading logs
│   └── schema.sql            # SQLite database table definitions and indices
│
├── scraper/
│   ├── macro_scraper.py      # Cleans, parses, and loads macro events into database
│   └── apify_client.py       # Wrapper for the Apify SDK economic calendar actor
│
├── preprocessing/
│   ├── generate_mock_data.py # High-fidelity mock logs/events generator (for immediate runs)
│   ├── clean_logs.py         # Standardizes logs, filters accounts, groups fills < 500ms
│   ├── feature_engineering.py# Generates time, market, and rolling trading features
│   └── merge_data.py         # Chronological merge_asof for trades and macro events
│
├── models/
│   ├── train.py              # Model selection and final retraining script
│   ├── walk_forward_validation.py # Chronological sliding window validation loop
│   ├── predict.py            # Global inference prediction runner
│   └── metrics.py            # Financial and statistical performance metrics
│
├── visualization/
│   ├── heatmap.png           # Heatmap of best predicted strategies by time period
│   ├── equity_curve.png      # Cumulative PnL curve (Model vs. Static Baselines)
│   ├── heatmap.py            # Generates best strategy allocation heatmap
│   └── equity_curve.py       # Generates cumulative PnL comparison chart
│
├── reports/
│   ├── evaluation.md         # Detailed markdown performance report
│   └── performance.pdf       # Polished executive PDF summary sheet
│
├── config.py                 # Centralized configuration loader (env variables, paths, params)
├── requirements.txt          # Python dependencies
├── main.py                   # Main pipeline orchestrator script
├── README.md                 # Project documentation
└── .env                      # Environment configurations (credentials and paths)
```

---

## Installation & Setup

1. **Prerequisites**
   * Python 3.11 or higher
   * `pip` package manager

2. **Clone the Repository**
   Ensure the directory structure matches the folder layout.

3. **Install Dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Environment Configuration**
   Copy or edit the `.env` file in the root directory:
   ```env
   # Leave empty to automatically fall back to simulated macro events
   APIFY_TOKEN=
   
   ACTOR_ID=pintostudio/economic-calendar
   DATABASE_PATH=database/trading.db
   MODEL_PATH=models/model.pkl
   ```

---

## Running the Pipeline

To execute the entire end-to-end pipeline automatically, run:

```bash
python main.py
```

This single command triggers the following sequence:
1. **Scrape macroeconomic events** for the past 180 days (or generate high-fidelity synthetic events if `APIFY_TOKEN` is blank).
2. **Update the SQLite database** (`macro_events` table).
3. **Load historical trading logs** (or generate realistic mock logs if missing).
4. **Clean the trading logs** (standardize to UTC, remove duplicates, filter invalid accounts, and group consecutive transactions within 500ms).
5. **Update the SQLite database** (`trading_logs` table).
6. **Execute Feature Engineering** (compute temporal, macroeconomic surprise/distance, and rolling PnL/Win Rate trading features).
7. **Train machine learning models** (XGBoost vs. Random Forest) and save the best model payload.
8. **Execute Chronological Walk-Forward Validation** (folds of 30-day train / 7-day test) to generate out-of-sample metrics.
9. **Generate prediction tables** (`prediction_table.csv`).
10. **Draw visualizations** (heatmap and equity curve).
11. **Compile reports** (`evaluation.md` and `performance.pdf`).

---

## Code Quality & Implementation Details

* **Lookahead Bias Prevention**: Standard scaling and categorical encoding are fit exclusively on the training slice of each walk-forward fold, then applied to the test slice. Rolling features (e.g. `rolling_pnl` and `rolling_win_rate`) are shifted by 1 trade to ensure the model does not train on future information.
* **500ms Grouping**: Handled via vectorized shifts to identify fills occurring within 500ms of each other for the same account, strategy, simulation, and direction. Prices are aggregated using volume/quantity weighting (VWAP).
* **High-Fidelity Mock Data**: Generated logs contain systematic patterns:
  - `Breakout_14` excels during high-importance macro releases with large forecast surprises.
  - `Mean_Reversion_20_2` and `Grid_Trading_5` lose money during high-impact macro releases but are highly profitable in quiet, range-bound regimes.
  - This ensures that the ML model has actual signal to learn from, resulting in a model-selected portfolio that significantly outperforms static benchmarks.

---

## Final Deliverables

The execution generates:
* **SQLite database**: `database/trading.db`
* **Processed dataset**: `data/processed/processed_dataset.csv`
* **Walk-Forward Validation metrics**: `walk_forward_results.csv`
* **Prediction table**: `prediction_table.csv`
* **Trained ML model**: `models/model.pkl`
* **Heatmap**: `visualization/heatmap.png`
* **Equity Curve**: `visualization/equity_curve.png`
* **Evaluation report**: `reports/evaluation.md`
* **Executive PDF dashboard**: `reports/performance.pdf`

---

## Future Improvements

1. **Hyperparameter Tuning**: Integrate Optuna to optimize XGBoost hyperparameters dynamically across walk-forward folds.
2. **Alternative Models**: Add LightGBM and Recurrent Neural Networks (LSTM/GRU) for temporal sequence modeling.
3. **Advanced Market Features**: Scrape volatility index (VIX) and order book imbalance metrics to complement macroeconomic features.
4. **Execution Latency Simulation**: Model transaction slippage and trading fees in the portfolio equity curve.
