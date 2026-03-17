# Documentation — XGBoost Directional Prediction

---

## Objective
- Identify if cross-asset correlation is a more accurate predictor of directional betting than single-asset models
- Identify if cross-asset correlation offsets the risk profile of the trades

---

## Background
- Polymarket provides directional betting for the pairs BTCUSD, ETHUSD, SOLUSD, XRPUSD
- XGBoost is well suited to picking up on relational dependencies

---

## Overview

### Phase 1: Data Manipulation
✅ Data Collection
  * Retrieve candle data for all 4 assets
  * Collate into CSV

✅ Feature Engineering
  * Engineer features for all 4 assets

✅ Data Checking
  * Visual inspection
  * Missingness / error value checks

✅ Data Splitting
  * Decide sample split
  * 80/20 split

### Phase 2: Model Creation
✅ Define Model
  * Specifications
  * Hyperparameters

✅ Build Infrastructure
  * Design mapping
  * Testing pipeline
  * Backtest, walk-forward, permutation

### Phase 3: Validation
✅ Walk-Forward
  * Strategy generalisation & hyperparameter tuning
  * Fold model creation

✅ Permutation
  * Evaluate strategy robustness (statistical significance)
  * Analyse random probability density distributions

[ ] Live Testing
  * Build live infrastructure
  * Evaluate as extended out-of-sample data (2500+)

[ ] Post-Evaluation
  * Backtest strategy against live-tested data
  * Ensure coherence

### Phase 4: Implementation
[ ] Soft Implementation
  * Implement with full trade execution
  * Test over allotted period
  * Capital allocation
  * Monitor results

[ ] Full Implementation
  * Scaling plan
  * Monitor results

---

## Methodology

### Phase 1: Data Manipulation

#### Data Collection
- Collected data from 2022-01-01 to present
- Assets: BTCUSD, ETHUSD, SOLUSD, XRPUSD

#### Feature Engineering
- Added all previous calculations for all 4 assets
- All included in one wide dataset
- 144 features per asset

#### Data Check
```
INFO: len of columns: 576

INFO: unique time intervals found: 2
  - 0 days 01:00:00: 35784 occurrences
  - 0 days 02:00:00: 1 occurrences

WARNING: Inconsistent time intervals detected!

Interval distribution:
time
0 days 01:00:00    35784
0 days 02:00:00        1
Name: count, dtype: int64

Expected interval: 3600000000000 nanoseconds

Found 1 anomalous interval:

Timestamp             | Interval         | Previous Timestamp
----------------------------------------------------------------------
2023-03-24 14:00:00   | 0 days 02:00:00  | 2023-03-24 12:00:00
```

**Notes:**
- One inconsistent value, likely due to a market or Binance error
- Left in as it is a single value and preserves the temporal continuity required for this strategy

#### Data Split
- Training: 2022-01-07 11:00:00 → 2025-04-14 07:00:00
- Testing: 2025-04-14 08:00:00 → 2026-02-06 13:00:00

---

### Phase 2: Model Creation

#### Define Model

**Model Specifications Overview:**
- XGBoost model for relational dependencies
- Using wide dataset for cross-correlational relationships

**Model Hyperparameters Overview:**
- Model recalibration every 2–4 weeks
- Model should utilise the following parameters:
  * colsample_bytree = 0.8
  * n_estimators = 500
  * learning_rate < 0.01
  * max_depth = 4–5
  * early_stopping_rounds = 50

#### Build Infrastructure

**Infrastructure Overview:**
- Model creation intertwined with testing logic
- Separate live model creation script

**Plan:**
- `_5backtest.py` → post-model evaluation
  * Contains production-ready training, test, and evaluation
- `_6walkforward.py` → full backtest
  * Outputs full strategy results for entire periods
  * Inherits backtesting engine class
- `_7permutation.py` → permutation on synthetic future created dataframes
  * Can test entire period with multiple permutations
  * Utilised in production phase
- `test.py` → forward testing component
  * Main trading infrastructure without execution component

**Backtest Overview:**

Inputs:
- df_training_slice → training slice for dataset
- df_testing_slice → testing slice for dataset
- Configuration
- Model parameters

Outputs:
- Equity curve (testing period only)
- Returns curve (testing period only)
- Trade positions curve (testing period only)
- Model → trained model

Classes:
- Trade position
- Backtest portfolio manager
- Backtest results
- Backtest engine

**Walk-Forward Overview:**

Inputs:
- complete_df with start and end date
- Configuration
- Model parameters

Outputs:
- Full outcome results:
  * Equity curve, outcome difference distribution (outcome - prediction)
  * Model stability, consistency, feature importance

Classes:
- Backtest engine (inherited)
- Walk-forward portfolio manager
- Walk-forward results
- Walk-forward model evaluation
- Walk-forward engine

**Permutation Overview:**

Inputs:
- Model
- complete_df (for randomisation)
- Configuration
- Model parameters

Outputs:
- Full distribution of equity curves
- Statistical significance

Classes:
- Trade position (inherited)
- Backtest portfolio manager (inherited)
- Walk-forward engine (inherited)
- Permutation
- Permutation results

#### Testing Pipeline and Intention

| Component | Intention | Use Cases | Aims |
|---|---|---|---|
| Backtest | Isolate and evaluate post-model performance | Composite class for other testing methodologies, post-implementation validation | Evaluate past performance of already-implemented strategies |
| Walk-Forward | Evaluate the entire strategy for a given timeframe | Strategy generalisation and hyperparameter tuning | Provide a complete picture of the strategy and its attributes |
| Permutation | Evaluate critical errors (data leakage, overfitting) | Evaluating new strategy or parameter set | Provide statistical robustness |

**Pipeline:**
1. Data manipulation
2. Walk-forward analysis
   * Strategy creation
   * Hyperparameter tuning
3. Permutation analysis
   * Evaluation and robustness testing
4. Live model creation
   * Create live model
   * Utilise recalibration logic
5. Live testing
   * Evaluate as extended out-of-sample testing
   * Compare to backtest post-production model
6. Live production
   * Implement and monitor with risk management metrics

---

### Phase 3: Validation

#### Walk-Forward Analysis

**Hypothesis Testing:**

- Model consistency:
  * H₀ = Walk-forward strategy results fail to maintain profitability against out-of-sample data
  * H₁ = Walk-forward strategy results maintain profitability against out-of-sample data ✅
- Model stability:
  * H₀ = Walk-forward strategy features capture low inter-fold correlation between features ✅
  * H₁ = Walk-forward strategy features capture high inter-fold correlation between features

**Evaluation Metrics:**

- Model consistency:
  * Returns → show consistent profitability across time
  * Win rate → minimal degradation between in/out-of-sample data
  * Drawdowns → relatively unchanged between in/out-of-sample data
- Model stability:
  * Top 10 features remain dominant across folds and between in/out-of-sample data

**Results:**

In-sample:
```
BACKTEST RESULTS
----------------
TOTAL TRADES:         88503
TOTAL WINS:           47519
WIN RATE:             53.69%
TOTAL RETURNS:        6535.00%
MAX CONSEC. WINS:     22
MAX CONSEC. LOSSES:   29
MAX DRAWDOWN:         43.41%
AVG CONSEC. WINS:     2.63
AVG CONSEC. LOSSES:   2.27
AVG DRAWDOWN:         1.39%
FULL KELLY FRACTION:  7.38%
1/4 KELLY FRACTION:   1.85%
----------------
MODEL EVALUATION: BTCUSD
----------------------------
TOTAL MODELS EVALUATED: 34
TOP 20 FEATURES (BY AVG GAIN):
BTCUSD_vwap_distance_period_12             13.325464
BTCUSD_price_zscore_period_12              13.106827
BTCUSD_avg_taker_sell_price_diff           12.806227
BTCUSD_close_to_vwap                       12.397202
BTCUSD_avg_taker_buy_price_diff            11.993665
ETHUSD_vwap_distance_period_12             11.725910
BTCUSD_buy_efficiency                      11.673234
ETHUSD_avg_taker_buy_price_diff            11.556929
ETHUSD_avg_taker_sell_price_diff           11.369646
ETHUSD_close_to_vwap                       11.351045
BTCUSD_distance_from_ema_period_12         11.216954
BTCUSD_sell_efficiency                     11.206537
BTCUSD_volume_imbalance                    11.193518
ETHUSD_sell_efficiency                     10.932463
BTCUSD_quote_volume_efficiency             10.924618
SOLUSD_avg_taker_buy_price_diff            10.872423
BTCUSD_net_taker_volume                    10.696952
ETHUSD_price_zscore_period_12              10.621894
BTCUSD_buyer_pressure_change_period_168    10.598793
BTCUSD_avg_trade_size_change_period_48     10.568162
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0634

MODEL EVALUATION: ETHUSD
----------------------------
TOTAL MODELS EVALUATED: 34
TOP 20 FEATURES (BY AVG GAIN):
ETHUSD_price_zscore_period_12         12.645857
BTCUSD_price_zscore_period_12         12.530995
BTCUSD_avg_taker_buy_price_diff       12.526382
BTCUSD_close_to_vwap                  12.387353
BTCUSD_avg_taker_sell_price_diff      12.113890
BTCUSD_vwap_distance_period_12        12.024712
ETHUSD_vwap_distance_period_12        11.949704
ETHUSD_avg_taker_sell_price_diff      11.892491
ETHUSD_quote_volume_efficiency        11.866060
ETHUSD_close_to_vwap                  11.552969
ETHUSD_buy_efficiency                 11.399676
ETHUSD_kyle_lambda_period_12          11.292932
BTCUSD_quote_volume_efficiency        11.277681
ETHUSD_sell_efficiency                11.265483
BTCUSD_sell_efficiency                11.254340
ETHUSD_distance_from_ema_period_12    11.128010
ETHUSD_avg_taker_buy_price_diff       10.904837
ETHUSD_price_zscore_period_24         10.902135
BTCUSD_buy_efficiency                 10.886974
ETHUSD_volume_imbalance               10.820421
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0516

MODEL EVALUATION: SOLUSD
----------------------------
TOTAL MODELS EVALUATED: 34
TOP 20 FEATURES (BY AVG GAIN):
BTCUSD_avg_taker_sell_price_diff           11.984945
BTCUSD_sell_efficiency                     11.459774
ETHUSD_quote_volume_efficiency             11.244408
SOLUSD_vwap_distance_period_12             11.143407
SOLUSD_avg_taker_buy_price_diff            11.039710
ETHUSD_price_zscore_period_12             10.978004
SOLUSD_buy_efficiency                      10.933869
SOLUSD_avg_taker_sell_price_diff           10.921482
BTCUSD_buy_efficiency                      10.864264
BTCUSD_close_to_vwap                       10.854300
BTCUSD_buyer_pressure_change_period_168    10.853134
ETHUSD_avg_taker_buy_price_diff            10.794321
BTCUSD_buy_price_impact                    10.788043
SOLUSD_close_to_vwap                       10.774807
ETHUSD_kyle_lambda_period_12               10.743116
ETHUSD_amihud_period_12                    10.677068
ETHUSD_avg_taker_sell_price_diff           10.632784
SOLUSD_price_zscore_period_12              10.554263
BTCUSD_price_zscore_period_12              10.539696
SOLUSD_volume_imbalance                    10.535263
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0443

MODEL EVALUATION: XRPUSD
----------------------------
TOTAL MODELS EVALUATED: 34
TOP 20 FEATURES (BY AVG GAIN):
XRPUSD_distance_from_ema_period_12              13.596631
XRPUSD_price_zscore_period_12                   12.848462
XRPUSD_vwap_distance_period_12                  12.414846
XRPUSD_avg_taker_sell_price_diff                12.011814
XRPUSD_distance_from_ema_period_24              11.803943
XRPUSD_close_to_vwap                            11.767947
XRPUSD_avg_taker_buy_price_diff                 11.507461
XRPUSD_vwap_distance_period_24                  11.435864
BTCUSD_price_zscore_period_12                   11.430959
ETHUSD_kyle_lambda_period_96                    11.309346
BTCUSD_close_to_vwap                            11.179173
XRPUSD_price_zscore_period_24                   10.995550
XRPUSD_cumulative_volume_imbalance_period_12    10.858816
XRPUSD_close_change_period_12                   10.535476
BTCUSD_buyer_pressure_change_period_168         10.524793
XRPUSD_vwap_proxy                               10.474216
BTCUSD_avg_taker_sell_price_diff                10.457894
BTCUSD_vwap_distance_period_12                  10.452774
BTCUSD_sell_price_impact                        10.449262
BTCUSD_buy_efficiency                           10.390201
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0658
```

Out-of-sample:
```
BACKTEST RESULTS
----------------
TOTAL TRADES:         12867
TOTAL WINS:           6742
WIN RATE:             52.40%
TOTAL RETURNS:        617.00%
MAX CONSEC. WINS:     17
MAX CONSEC. LOSSES:   25
MAX DRAWDOWN:         37.50%
AVG CONSEC. WINS:     2.62
AVG CONSEC. LOSSES:   2.38
AVG DRAWDOWN:         3.60%
FULL KELLY FRACTION:  4.80%
1/4 KELLY FRACTION:   1.20%
----------------
MODEL EVALUATION: BTCUSD
----------------------------
TOTAL MODELS EVALUATED: 5
TOP 20 FEATURES (BY AVG GAIN):
XRPUSD_open_change_period_48             12.795036
BTCUSD_sell_price_impact                 12.767150
BTCUSD_close_change_period_24            12.736389
BTCUSD_avg_taker_sell_price_diff         12.220894
XRPUSD_vol_norm_volatility_period_24     12.040105
ETHUSD_avg_taker_sell_price_diff         12.008086
SOLUSD_vol_norm_volatility_period_24     11.875666
ETHUSD_close_to_vwap                     11.870477
BTCUSD_vwap_distance_period_12           11.862360
SOLUSD_sell_price_impact                 11.795766
BTCUSD_net_taker_volume                  11.771426
XRPUSD_trade_intensity                   11.763614
ETHUSD_price_zscore_period_12            11.748013
SOLUSD_avg_taker_sell_price_diff         11.744688
ETHUSD_avg_taker_buy_price_diff          11.730094
BTCUSD_volume_concentration_period_96    11.679077
BTCUSD_volume_change_period_168          11.662331
BTCUSD_volatility_change_period_96       11.577769
SOLUSD_volume_change_period_168          11.560187
ETHUSD_vwap_distance_period_12           11.550617
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0326

MODEL EVALUATION: ETHUSD
----------------------------
TOTAL MODELS EVALUATED: 5
TOP 20 FEATURES (BY AVG GAIN):
BTCUSD_price_zscore_period_12                   12.683624
BTCUSD_cumulative_volume_imbalance_period_24    12.503434
ETHUSD_vwap_distance_period_12                  12.501689
SOLUSD_sell_price_impact                        12.439655
SOLUSD_buy_efficiency                           12.308952
XRPUSD_kyle_lambda_period_24                    12.232067
XRPUSD_open_change_period_24                    12.151707
SOLUSD_distance_from_ema_period_24              11.909179
ETHUSD_amihud_period_24                         11.891432
XRPUSD_sell_price_impact                        11.862430
ETHUSD_buy_efficiency                           11.798830
SOLUSD_volume_change_period_96                  11.727270
SOLUSD_price_zscore_period_12                   11.664739
XRPUSD_volume_price_corr_period_12              11.660619
ETHUSD_low_change_period_12                     11.653326
SOLUSD_trade_size_volatility_period_24          11.629051
ETHUSD_vwap_distance_period_24                  11.567915
ETHUSD_avg_taker_buy_price_diff                 11.474655
ETHUSD_avg_taker_sell_price_diff                11.464073
XRPUSD_avg_trade_size_change_period_168         11.429064
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0073

MODEL EVALUATION: SOLUSD
----------------------------
TOTAL MODELS EVALUATED: 5
TOP 20 FEATURES (BY AVG GAIN):
XRPUSD_quote_volume_momentum_period_96          12.695464
ETHUSD_taker_buy_qa_volume                      12.475522
BTCUSD_cumulative_volume_imbalance_period_48    12.115732
ETHUSD_buy_efficiency                           12.086115
SOLUSD_taker_buy_qa_volume                      12.072828
SOLUSD_volume_price_corr_period_168             12.047272
SOLUSD_open_change_period_96                    11.970747
SOLUSD_vwap_distance_period_12                  11.633631
SOLUSD_close_change_period_48                   11.629688
SOLUSD_num_trades                               11.626886
ETHUSD_kyle_lambda_period_48                    11.527662
ETHUSD_vpin_period_96                           11.439997
SOLUSD_quote_volume_momentum_period_48          11.418437
ETHUSD_price_zscore_period_168                  11.414257
XRPUSD_price_zscore_period_12                   11.401563
BTCUSD_volatility_change_period_96              11.395747
SOLUSD_taker_buy_ba_volume                      11.370580
BTCUSD_volume_concentration_period_12           11.346526
SOLUSD_volume_change_period_96                  11.345526
ETHUSD_taker_sell_ba_volume                     11.308160
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0394

MODEL EVALUATION: XRPUSD
----------------------------
TOTAL MODELS EVALUATED: 5
TOP 20 FEATURES (BY AVG GAIN):
ETHUSD_volume_imbalance                   12.479079
SOLUSD_volatility_change_period_96        12.162711
ETHUSD_close_to_vwap                      12.144378
ETHUSD_net_taker_volume                   12.100254
SOLUSD_volume_change_period_96            12.086314
SOLUSD_buy_efficiency                     12.029059
SOLUSD_taker_sell_ba_volume               11.948258
SOLUSD_buyer_pressure_change_period_12    11.771544
BTCUSD_volume_change_period_96            11.652031
ETHUSD_volume_price_corr_period_96        11.630504
XRPUSD_volatility_change_period_168       11.612005
ETHUSD_num_trades                         11.545988
ETHUSD_kyle_lambda_period_12              11.544908
XRPUSD_volume_change_period_12            11.543983
BTCUSD_vol_norm_volatility_period_168     11.499258
SOLUSD_vwap_proxy                         11.498658
SOLUSD_close_change_period_24             11.498581
BTCUSD_open_change_period_48              11.491183
SOLUSD_volume_concentration_period_96     11.449979
XRPUSD_volume_concentration_period_48     11.447361
MODEL DRIFT (AVG INTER-FOLD CORRELATION): 0.0332
```

**Interpretation:**
- Model consistency: Model holds true out-of-sample with a win rate of ~53%, showing minor degradation out-of-sample
- Model stability: Stability is weaker than anticipated — cross-correlation features appear to expand the shift in feature importance across market regimes

**Notes:**
```python
config = {
    'filepath': 'analysis/data/crypto_1h_testing.csv',
    'start': '2025-04-01',
    'end': '2026-01-31',
    'symbols': ['BTCUSD', 'ETHUSD', 'SOLUSD', 'XRPUSD'],
    'starting_equity': 100,
    'training_periods': 4,
    'testing_periods': 1,
    'walk_forward_shift_periods': 1,
    'upper_threshold': 0.505,
    'lower_threshold': 0.495
}

model_parameters = {
    'n_estimators': 1000,
    'early_stopping_rounds': 100,
    'learning_rate': 0.01,
    'max_depth': 4,
    'min_child_weight': 20,
    'gamma': 0.1,
    'subsample': 0.8,
    'colsample_bytree': 0.8,
    'random_state': 42
}
```

Observations:
- Model performance shows the highest stability and consistency seen to date
- Diversifying across multiple assets increases trade volume, theoretically increasing the total returns that can be generated

---

#### Permutation Analysis

**Hypothesis Testing:**

- Model robustness:
  * H₀ = Permutation analysis proves no feature-target relationship within the baseline model
  * H₁ = Permutation analysis proves a feature-target relationship is structured within the baseline model ✅

**Evaluation Metrics:**

- Model robustness:
  * Baseline outperforms permutations
  * Statistically significant p-value < 0.05

**Results:**

In-sample:
```
PERMUTATION ANALYSIS SUMMARY
-----------------------------
Base Strategy Return:       6627
Average Permutated Return:  970.41
Max Permutated Return:      1916
p-value:                    0.00990
```

Out-of-sample:
```
PERMUTATION ANALYSIS SUMMARY
-----------------------------
Base Strategy Return:       756
Average Permutated Return:  159.73
Max Permutated Return:      408
p-value:                    0.00990
```

**Interpretation:**
- Both in/out-of-sample groups vastly outperform randomised permutations
- Both groups show statistically significant results with p-values < 0.05

**Notes:**
- Both groups show substantially better results than random, allowing confident rejection of the null hypothesis
- Model is ready for live testing once infrastructure is complete

---

#### Forward Testing

**Hypothesis Testing:**

- Model validation:
  * H₀ = Strategy model and assumptions do not hold true in forward testing
  * H₁ = Strategy model and assumptions are validated in forward testing

**Evaluation Metrics:**
- Evaluate over approximately 2,500+ trades
- Does not exceed fat-tailed event thresholds: max consecutive losses, max drawdown period
- Win rate above 50% — if marginally above 50%, observe more values; if above 51%, model can be live traded
- Forward test metrics match or are similar to backtest metrics

**Implementation:**

Scripts:
```
├── clients/
│   ├── collection_agent.py
│   ├── trade_entry_agent.py
│   └── trade_exit_agent.py
├── production/
│   └── live_model.pkl
├── setup/
│   ├── claim.py
│   └── set_up_wallet.py
├── storage/
│   ├── data_attributes.py
│   ├── memory.py
│   └── trade_log.csv
├── strategy/
│   └── engine.py
├── config.py
├── main.py
└── test.py
```

**Results:** *Not viable*

**Interpretation:** *Not viable*

**Notes:**
- Forward testing guides the range of plausible values expected under a normal distribution
- With a sample size of ~1,300 trades, the range of plausible values narrows — expected win rate range of 52 ± 1.32% at a 95% confidence interval
- Only ~15 days of data were collected, sufficient to identify the directional trend and generate ~1,300 trade records
- Win rate expected to drift from 50% → 52% as sample size increases

---

#### Post-Evaluation

**Hypothesis Testing:**

- Post-results evaluation:
  * H₀ = Strategy does not conform to post-production model evaluation
  * H₁ = Strategy conforms to post-production model evaluation

**Evaluation Metrics:**
- Backtest results align with forward-testing results
- Same or similar equity curve

**Results:** *Not viable*

**Interpretation:** *Not viable*

**Notes:** *Not viable*

---

> **Notice:** Live implementation has halted indefinitely — refer to README.md for project status.
