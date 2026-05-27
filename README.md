# Term Project 2: Volatility-Volume-based Order Management Utilizing Statistical and Rule-based Techniques

## Overview
This repository implements order slicing optimization on slippage cost for agent orders across 7 futures markets. Given 1-min OHLC data, through rolling state classification with interval volume, volatility and price change parameters, as well as emprical probabiltiy density function (EPDF) construction on price movements, the algorithm sets target price to place limit order, subject to fill rate and profit trade-off. 

The project compare performance of three strategies: the original agent orders as baseline, agent order with resubmission, and the EPDF strategy with optimized hyper-parameters on the target initial filled probabiltiy `p_initial`, order live period `tau_fill`, maximum allowed resubmission `max_resubmits` and half-life for `EWMA` and `EWMV` computation `m_peroids`.

Other parameters including the holding period `τ`, the number of states for `volume`, `volatility` and `price change` classifications `(M,N,K)` prompt user input.

## Data Requirements
Market OHLC and agent order data should be organized in the following structure with exact naming format, the program maps market folder name to corresponding agent order file's market identifier as the market variable.
The notebook should be placed under the same root directory as the order management folder.
### Data Directory Structure
```text
order_management/
├── data/
│   ├── EuroStoxx/
│   │   ├── AIAgent_EuroStoxx.csv      # agent orders
│   │   ├── VGH22.csv                  # contract data
│   │   └── VGM22.csv                  # next contract
│   ├── GBP-British Pound/
│   │   ├── AIAgent_GBPUSD.csv
│   │   ├── BPM20.csv
│   │   └── BPU20.csv
│   ├── German Bunds - German Government Bonds/
│   │   ├── AIAgent_Bunds.csv
│   │   ├── RXM25.csv
│   │   ├── RXU25.csv
│   │   └── RXZ25.csv
│   ├── Gold/
│   │   ├── ...
│   ├── HeatingOil/
│   │   ├── ...
│   ├── JPY/
│   │   ├── ...
│   └── Nasdaq/
│   │   ├── ...
```
### Contract files
Each contract file contains **1‑minute OHLCV** data with the following columns:

| Column  | Description                          |
|---------|--------------------------------------|
| `datetime` | Timestamp (localized, see note)   |
| `open`     | Open price                         |
| `high`     | High price                         |
| `low`      | Low price                          |
| `close`    | Close price                        |
| `volume`   | Trading volume                     |

Markets such as JPY, HeatingOil, EuroStoxx,Bunds and GBP uses Excel time in the datetime section, "load_lm_auto" funtion handles the conversion.

### Agent Order files
| Column         | Description                                 |
|----------------|---------------------------------------------|
| `timestamp`    | Order submission time (will be aligned to next tradable bar) |
| `direction`    | `1` for long (buy), `-1` for short (sell)   |
| `quantity`     | Number of contracts                         |
| `limit_price`  | Original agent quoted price                 |

Agent Orders are used as Baseline strategy for benchmarking (Strtegy 1), its quantity and price are preserved as limit order submission to evaluate on fill rate, slippage and pnl performance, which is benchmarked against EPDF strategy.

## Methodology Outline
The algorithm consists of four main stages: **Data Cleaning**,**Rolling EPDF Constructions**, **Order Execution Simulation**, and **Hyper-parameters Tuning**.

### 0.Definition of key inputs
  - `market`: Select from dropdown, the futures market to trade
  - `M`:Number of states for `volume` metric, choose between 2 to 4
  - `N`:Number of states for `volatility` metric, choose between 2 to 4
  - `K`:Number of states for ``ΔPrice` metric, choose between 2 to 4
  - `τ`:Select from dropdown between [5,10,15,30,60], Holding period


### 1.Data Cleaning

- **Trading date alignment**
  
  CME-traded futures (Nasdaq, JPY, GBP, Gold, Heating Oil): trading day runs from 18:00 previous day to 17:00 current day (NY time). Timestamps are shifted back by 18 hours before date extraction.  
  Eurex-traded futures (EuroStoxx, Bunds): active session 01:00–22:00 CET; timestamps shifted back by 1 hour.

- **Stable start date detection**
  
  To identify a stable start of the contract after which it is considered active and liquid, a contract-specific full-session benchmark is computed as the 95th percentile of daily observed trading minutes over the sample period. 
  A contract is considered to have entered a stable trading period once its daily observed minutes reach at least 80% of this benchmark for 10 consecutive trading days.

- **Coverage filtering**
  
  Define exepected_minutes of a trading day as he 95th percentile of observed daily minutes. Days with coverage ratio='observed_minutes/expected_minutes'>0.9 are retained, rest are considered as missing records.
  
- **Roll date identification**
  
  Adjacent contracts are compared over their overlapping period. The roll date is the earliest day where the next contract's volume exceeds the current contract's volume for 2 consecutive trading days to ensure persistent liquidity shift.
  
- **Tick size**
  
  For each contract, we aggregate all observed open, high, low, and close prices, compute the differences between sorted unique price levels, and identify the smallest price increment that explains the vast majority of observed price changes.

### 2.EPDF Construction

The model builds conditional distributions of future price movements using only past information upto the latest full interval to aviod look-ahead bias. State classification, EPDF construction and order processing are carried out in parallel, on rolling basis. 

The algorithm move through time-sorted agent order record, and process all **compelete** interval states and EPDF updates up to the order time, if order is not filled during its lifetime and resubmission is required, it will proceed to the next effective bar time without updating new EPDF and assuming same market state. This approximation is not expected to have much impact on the result, given the large size of EPDF data and short order lifetime within an hour.

- **Interval slicing**

  Contracts are merged starting from its stable start date and rolled over timely. The continuous contract for the specific market is split into fixed-length intervals of length `τ` . On 'Open' at time tj, the processing of past interval [tj-τ,tj) is carried out. Three metrics are computed for each interval:
  - `Volume`: Sum of volume
  - `Volatility`: Represented by range movement, High-Low over the interval
  - `ΔPrice`: Next interval's Open - current interval's first Open
  
- **State classification**
  For each metric, exponential weighted moving average and variance (`EWMA` and `EWMV`) are updated per interval with half‑life `m_period` (converted to decay factor `λ`).
  Each metric is discretied into 2-4 states based on actual interval value's deviation from `EWMA`, scaled by `EWMV`:
  - state number=2: cutoff being 0, state=0 if metric <= `EWMA`, state=1 otherwise.
  - state number=3: cutoff being [-0.5,0.5]. state=0 if metric ≤ EWMA−0.5EWMV, state=1 if between `EWMA` − 0.5`EWMV` and `EWMA` + 0.5`EWMV` , state=2 otherwise.
  - state number=4: cutoff being [-0.7,0,0.7].
  The combined state classification is indexed and stored in a tuple `(m,n,k)`.
  
- **Conditional probability arrays**

  For each state `(m,n,k)`, we store empirical histograms of:
  - `Range` (total price movement = high – low over next τ minutes)  
  - `RangeUp` (upward movement = max price – open)  
  - `RangeDown` (downward movement = open – min price, stored as absolute value)
The counts of `Range`=l are divided by total counts in the state to constructe the EPDF.

### 3. Order Execution Simulation

- **Setting target price for limit order**
  
  Given a new order from the agent record at time t:
  - Determine the current state `(m,n,k)` using only historical data up to `t`.
  - Reference price `O_t` = current market open.
  - For a **buy** order, we seek the smallest price offset `d` such that  
    `P(RangeDown ≥ d | state) ≥ p_initial`.  The limit price = `O_t – d`.  
    For a **sell** order, use `RangeUp` and limit price = `O_t + d`.
  - `p_initial` ∈ (0,1) controls aggressiveness, larger value indicates closer to market price thus larger probability of filled.
    
- **Resubmission policy**
  
  For strategy 2 (EPDF strategy) and strategy 3 (Agent order with resubmission), if order is not filled within `τ_fill` minutes, it is cancelled and resubmitted with a new limit price. This repeats up to 'max_resubmits' times, which is a market specific hyper-parameter.
  
- **Evaluation Framework**

  Three strategies are compared to isolate the effect of repricing and resubission:
  
| Strategy | Description |
|----------|-------------|
| **1. Agent baseline** | Original agent price, no resubmission (single check on submission bar). |
| **2. EPDF repricing + resubmission** | Our method: replace price with EPDF target, allow resubmissions. |
| **3. Agent with resubmission** | Keep original agent price, but apply the same resubmission policy as Strategy 2. |

Evaluation compares the key metrics:

| Metric | Description |
|--------|-------------|
| **Final net position** | Terminal inventory after all filled trades. |
| **Final portfolio value** | Terminal cash + inventory position, representing cumulative PnL. |
| **Order fill rate** | Proportion of original orders eventually executed (measured at order level, not submission‑attempt level for resubmission strategies). |
| **Average fill time** | Average time from effective market entry to execution (filled orders only). |
| **Submitted & filled quantity** | Submitted vs. executed quantity on buy/sell sides, plus unfilled quantity. |
| **Buy‑side / sell‑side fill rates** | Quantity‑based fill rates separately for buys and sells (identifies asymmetries). |
| **Average slippage** | Negative for better execution: for buys, slippage below effective open; for sells, above effective open. |

### 4. Hyperparameter Tuning

A grid search is performed on the training/validation split using the **mod_score** metric that balance PnL improvement against a fill-rate penalty.
Optimal hyper-parameters are market specific.

- **Grid Search**
  
  Four hyper parameters are tuned in the following order:
  - `p_initial` – target fill probability (grid 0.50–0.95, step 0.05)
  - `τ_fill` – order lifetime in minutes (grid 5–55)
  - `max_resubmits` – maximum attempts (1–6)
  - `m_period` – half‑life for EWMA/EWMV (depends on τ)

To improve the efficiency of the algorithm without introducing look-ahead bias, the parameter selection is based on order execution results on the validation set (0.6/0.2/0.2 train-validation-test split of the full dataset). EPDF constructed on the train set is applied, and with the rolling precomputed state table, state `(m,n,k)` for the tradable interval could be retrieved.

  For each candidate parameter, the **validation score** is computed as

  - PnL= `Portfolio value` (with `initial_cash`=0)
  - ΔPnL = PnL_EPDF – PnL_benchmark
  - `FillShortfall` = max(0, `α` × `FillRate_benchmark` – `FillRate_EPDF`)
  - `mod_score` = ΔPnL – `λ_penalty` × `FillShortfall`
  where `α = 0.8` (EPDF must achieve at least 80% of the benchmark fill rate), and `λ_penalty = 1` (penalty weight for fill‑rate shortfall)

The parameter with teh highest validation score is selected
  
- **Out of Sample Performance**
Once all hyper-parameters are optimized:
  - Rebuild EPDF on the **combined training+validation** data
  - Evaluate order execution performance on the **test set** with the optimal parameters set.

- **Notes on implementation**
  The optimal parameters values are hard coded for retrieval, the codes are marked down for reference.

## Structure and Application of the notebook

### Block 1. Helper functions for Market data preparation
This cell reads and cleans raw 1-minute OHLC contract file data for all 7 markets, assigns trading days, detects stable start dates and roll dates of contracts, and merges consecutive contracts into continuous chain.

-**Key functions**
| Function| Inputs    | Description | Outputs    |
|---------|-----------|---------|------------------------------------|
| `build_all_markets_daily` | `etract_dir`=str,`coverage_cutoff`=0.9,`expected_method`='p95'  | Processes all markets by assigning trading dates, localizing time, and filtering only active trading days with sufficient coverage ratio after stable starts |Dataframes: `all_daily_raw`, `all_daily`, `all_daily_clean`, `stable_starts`, `minute_store` |
| `infer_tick_sizes_from_minute_store`| `minute_store`=pd.DataFrame, `max_decimals`=6, `min_count`=5, `coverage_threshold`=0.95 |  Infer tick size for every market by detecting the the differences between sorted unique price levels, and identify the
smallest price increment that explains the vast majority of observed price changes|             |`tick_size_table`=pd.DataFrame
| `build_adjacent_contract_pairs`     | `summary_by_contract` =pd.DataFrame | Determine the rollover dates for adjacent contract pairs, and aggregated across markets into a table| `roll_table`=pd.DataFrame
| `Merge_Contracts`    |`market`=str, `minute_store`=pd.DataFrame, `roll_table`=pd.DataFrame, `stable_starts`=pd.DataFrame| Merge contracts into continuous chain for a specific market| `m`erged`=pd.DataFrame

-**Other functions** (called inside key functions)
| Function| Inputs    | Description | Outputs    |
|---------|-----------|---------|------------------------------------|
| `assign_trading_day` | `df`,`market`=str  | Conducts timezone localization & Shifts timestamps (CME: back 18h; Eurex:Back 1h). Adds `dt_local` and `trading_day` columns | Modified DataFrame|
| `add_expected_and_coverage`| `daily`=pd.DataFrame, `expected_method`='p95' |Computes expected minutes and coverage ratio | `daily`=pd.DataFrame (with  `expected_minutes, coverage`|
| `to_daily_observed`| `df_1m`, `contract`=str|Aggregates 1‑min data to daily observed minutes, volume| `daily`=pd.DataFrame|
| `find_stable_start`     |`daily` =pd.DataFrame, set of parameters  |Finds first date where observed minutes ≥80% of 95th percentile for 10 consecutive days|`stable_start_date`=pd.DataFrame

-**Plotting functions**
| Function| Description |
|---------|-----------|
| `plot_contract_hourly_heatmap` | Plot hourly activity heatmap by contract with average records per day  |
| `plot_roll_pair_from_store`| Plots volume around roll date for visual validation.|

-**Key data structures**
| Variable | Type | Description |
|----------|------|-------------|
| `minute_store` | `dict` | Key: `(market, contract)` → DataFrame with `plot_time, open, high, low, close, volume, trading_day` |
| `all_daily_raw` | `pd.DataFrame` | Daily observed minutes for all contracts (unfiltered). |
| `all_daily_clean` | `pd.DataFrame` | Daily data after coverage filtering (≥0.9). |
| `stable_starts` | `pd.DataFrame` | Columns: `market, contract, stable_start_date` |
| `roll_table` | `pd.DataFrame` | Columns: `market, contract_from, contract_to, roll_date` |
| `summary_by_contract` | `pd.DataFrame` | Aggregated per‑contract stats: `n_days, expected_minutes, coverage_mean, total_volume, first_date, last_date` |
| `tick_size_table` | `pd.DataFrame` | Tick size per contract with diagnostics. |
| `tick_size_map` | `dict` | Market → tick size (consistent within market). |

### 2. Order records preparation
### 3. (Marked down) Hyper-parameter tuning
### 4. Run_analysis function
### 5. Outputs
