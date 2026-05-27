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
  - `m_period` – half‑life for `EWMA/EWMV` (depends on τ)

  To improve the efficiency of the algorithm without introducing look-ahead bias, the parameter selection is based on order execution results on the validation set (0.6/0.2/0.2 train-validation-test split of the full dataset). EPDF constructed on the train set is applied, and with the rolling precomputed state table, state `(m,n,k)` for the tradable interval could be retrieved.

  For each candidate parameter, the **validation score** is computed comparing the improvement of EPDF strategy from agent with resubmission strategy (benchmark)

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
|:---------:|:-----------:|---------|:------------------------------------:|
| `build_all_markets_daily` | `etract_dir`=str,<br>`coverage_cutoff`=0.9,<br>`expected_method`='p95'  | Processes all markets by assigning trading dates, localizing time, and filtering only active trading days with sufficient coverage ratio after stable starts |Dataframes: `all_daily_raw`, `all_daily`, `all_daily_clean`, `stable_starts`, `minute_store` |
| `infer_tick_sizes_from_minute_store`| `minute_store`=pd.DataFrame,<br>`max_decimals`=6,<br>`min_count`=5, `coverage_threshold`=0.95 |  Infer tick size for every market by detecting the the differences between sorted unique price levels, and identify the smallest price increment that explains the vast majority of observed price changes| `tick_size_table`=pd.DataFrame|
| `build_adjacent_contract_pairs` | `summary_by_contract`<br>=pd.DataFrame | Determine the rollover dates for adjacent contract pairs, and aggregated across markets into a table| `roll_table`=pd.DataFrame|
| `Merge_Contracts`|`market`=str,<br>`minute_store`=pd.DataFrame,<br>`roll_table`=pd.DataFrame, `stable_starts`=pd.DataFrame| Merge contracts into continuous chain for a specific market| `merged`=pd.DataFrame|

-**Other functions** (called inside key functions)
| Function| Inputs    | Description | Outputs    |
|:---------:|:-----------:|--------|:------------------------------------:|
| `assign_trading_day` | `df`,`market`=str  | Conducts timezone localization & Shifts timestamps (CME: back 18h; Eurex:Back 1h). Adds `dt_local` and `trading_day` columns | Modified DataFrame|
| `add_expected_and_coverage`| `daily`=pd.DataFrame, `expected_method`='p95' |Computes expected minutes and coverage ratio | `daily`=pd.DataFrame (with  `expected_minutes`,`coverage`|
| `to_daily_observed`| `df_1m`, `contract`=str|Aggregates 1‑min data to daily observed minutes, volume| `daily`=pd.DataFrame|
| `find_stable_start`     |`daily` =pd.DataFrame, <br>set of parameters  |Finds first date where observed minutes ≥80% of 95th percentile for 10 consecutive days|`stable_start_date`=pd.DataFrame|

-**Plotting functions**
| Function| Description |
|:---------:|-----------|
| `plot_contract_hourly_heatmap` | Plot hourly activity heatmap by contract with average records per day  |
| `plot_roll_pair_from_store`| Plots volume around roll date for visual validation.|

-**Key data structures**
| Variable | Type | Description |
|:----------:|:------:|-------------|
| `minute_store` | `dict` | Key: `(market, contract)` → DataFrame with columns:`plot_time`, `open`, `high`, `low`, `close`, `volume`, `trading_day` |
| `all_daily_raw` | `pd.DataFrame` | Daily observed minutes for all contracts (unfiltered). |
| `all_daily_clean` | `pd.DataFrame` | Daily data after coverage filtering (≥0.9). |
| `stable_starts` | `pd.DataFrame` | Columns: `market`, `contract`, `stable_start_date` |
| `roll_table` | `pd.DataFrame` | Columns: `market`, `contract_from`, `contract_to`, `roll_date` |
| `summary_by_contract` | `pd.DataFrame` | Aggregated per‑contract stats: `n_days`, `expected_minutes`, `coverage_mean`, `total_volume`, `first_date`, `last_date` |
| `tick_size_table` | `pd.DataFrame` | Tick size per contract with diagnostics. |
| `tick_size_map` | `dict` | `Market` → `tick size` (consistent within market). |
| `merged` | `pd.DataFrame` | Columns include: `plot_time`(localized time), `open`, `high`, `low`, `close`,`volume`,`trading_day`(shifted),`contract` |


### Block 2. EPDF & state classification helper functions
Core functions that compute interval metrics, update EWMA/EWMV, classify market states, and build/update the empirical probability density functions (EPDF) for price movements. These are used both in the rolling backtest and in hyperparameter tuning.

-**Key functions**
| Function| Inputs    | Description | Outputs    |
|:---------:|:-----------:|---------|:------------------------------------:|
| `past_interval_para` | `tau`=int, `merged`=pd.DataFrame, `j`=int (counter), `t_begin_lst`=list, `t_true_end`=pd.Timestamp | for each τ‑minute interval, extract data, computes `volume`,`volatility`,`Δprice`,record the true end of data for the interval  |`interval_data`=pd.DataFrame, `param_lst`=list, `t_begin_lst`=list, `t_true_end`=pd.Timestamp`|
| `EWMA_EWMV`| `j`=int, `param`= float, `sum_W`=float, `sum_WX`= float, `sum_WSS`=float, `m_period`= int  |Updates exponentially weighted moving average and variance for each metric in each interval |`sum_W`=float, `sum_WX`= float, `ewma`=float, `sum_WSS`=float,`ewmv`=float|
| `class_state`| `df_ewma`= pd.DataFrame, `df_ewmv`=pd.DataFrame, `param_lst`= list, `state_threshold`=pd.DataFrame|Classifies each metric into a state benchmarking against `state_threshold` cutoffs|integers： `m`,`n`,`k`|
| `update_count`     | `m`,`n`,`k`=int, `Range_count`, `RangeUp_count`, `RangeDown_count`= np.ndarray, `interval_data`= pd.DataFrame, `tick_size`=float| Updates increments the corresponding EPDF count arrays for the given interval. |Updated count arrays|
| `EPDF`     |`daily` =pd.DataFrame, set of parameters  |Finds first date where observed minutes ≥80% of 95th percentile for 10 consecutive days|`stable_start_date`=pd.DataFrame|
| `find_order_price`     |`Range_count`, `RangeUp_count`, `RangeDown_count`= np.ndarray,<br>`l`=int,`direction`=int, `m`,`n`,`k`=int|Returns the probability of a specific price movement (in ticks) conditional on the state. |`density`=float|

-**Other functions** (called inside the key functions)
| Function| Inputs    | Description | Outputs    |
|:---------:|:-----------:|---------|:------------------------------------:|
| `threshold_of_state` | `state_count_lst`: list[int]  | Returns cutoff thresholds relative to EWMA for different number of states|`state_threshold`=pd.DataFrame|
| `find_L`|  `merged`=pd.DataFrame, `tick_size`= float |To determin the size of the 4th dimension (storing ticksize movement) for count arrays, regroup the data by 30mins and record the maximum movement with some buffer to be the size (especially useful for `HeatingOil` which have wild movements | `L`=int|

-**Plotting functions**
| Function| Description |
|:---------:|-----------|
| `plot_states` |Plots colored bar to visualize interval state classification  |
| `plot_EPDF` |Plots conditioanl EPDF distribution for each state  |
| `plot_param`|  Plots three metrics' movement across intervals with EMWA trend and 1x EWMV band around||

-**Key data structures**
| Variable | Type | Description |
|:----------:|:------:|-------------|
| `Range_count`, `RangeUp_count`, `RangeDown_count` | `np.ndarray` | 4-dim array indexed (m,n,k,l) to store counts of specific ticksize movement value for a particular state |
| `states` | `list[list]` | record states`(m,n,k)` index for each interval|


### Block 3. Order records preparation and execution helper functions
This block includes core functions to prepare agent order record files: load order files and align timestamps to market trading hours. It also includes functions to execute orders for three strategies with resubmission logic, and process the filled orders record for pnl computation.

-**Key functions**
| Function| Inputs    | Description | Outputs    |
|:---------:|:-----------:|---------|:------------------------------------:|
| `prepare_order_effective_from_path` | `path`=str, `merged`=pd.DataFrame| Loads and cleans agent order file with `Agent_Order_Cleaning` then aligns each order to the first market bar with `plot_time >= order.dt_local`  |`df_orders`=pd.DataFrame (with "effective_bar_time" column)|
| `split_market`| `df_order`=pd.DataFrame, <br>`train_ratio`=0.6, <br>`valid_ratio`= 0.2  |For grid search for hyperparemeters values part, split the orders chronologically into train/validation/ test sets|`train_orders`,`val_orders`, `test_orders`|
| `inventory_backtest_from_trades`     | `df_trade`=pd.DataFrame (filled trades),<br> `market_sorted`= pd.DataFrame,<br>`initial_cash`= 0.0| For each strategy, from the trade_table obtained from `to_trade_table` function, it conducts bar-level backtest by updating cash,position, and portfolio value.  |`df_bar`=pd.DataFrame|
| `summarize_our_execution`| `df_exec`=pd.DataFrame (execution record with all order submission)|Computes execution statistics for 3 strategies (fill rates, quote distance, slippage, attempts per order) |`summary`= pd.DataFrame


-**Other functions** (called inside the key functions)
| Function| Inputs    | Description | Outputs    |
|:---------:|:-----------:|---------|:------------------------------------:|
| `is_order_file` | `path`=str  | Checks if filename contains `"AIAgent"` and extracts market name|`market`=str or `None`, `df_orig_order`=pd.DataFrame or `None`|
| `Agent_Order_Cleaning`|`path`=str |Reads raw order CSV, converts Excel datetime to proper timestamp, localises timezone, returns cleaned DataFrame | `df_order`=pd.DataFrame (adding columns "dt_local", "price", "order")|
| `to_trade_table` | `df_exec`= pd.DataFrame (execution record with all order submission) | Filters filled attempts, renames columns to "dt_local", "price", "action"| `df_trade`=pd.DataFrame (filled trades)|
| `build_strategy_data` | pd.Dataframes: `df_bar`, `df_trades` (for each one of three strategies)| composing a dictionary labeld by strategy name as input for plotting function| `strategy_data`=dict|

-**Plotting functions**
| Function| Description |
|:---------:|-----------|
| `plot_three_strategies` |plots portfolio value changes for three strategies |
| `plot_strategies_details` |finds the day with maximum filled order volume, plots EPDF order execution details along market price movements for a 3h interval 9am to 12pm |

-**Key data structures**
| Variable | Type | Description |
|:----------:|:------:|-------------|
| `df_orders` | `pd.DataFrame` | Cleaned orders with added columns: `dt_local`, `price`, `order` (signed), `effective_bar_time`. |
| `df_exec` | `pd.DataFrame` | Per‑attempt execution records. Columns include: `dt_local`, `order`, `qty`, `attempt_no`, `submission_time`(start time of a particular attempt), `effective_bar_time`(the timestamp of the first market bar after `submission_time`), <br>`effective_open`(`open`orice of the bar), `state_time`, `m`,`n`,`k`, `price`,<br> `live_until`(deadline of current attempt before resubmission), `filled`('True' or 'False'), <br>`fill_dt`(Timestamp of the bar where the fill occurred), `fill_price`, <br>`fill_time_min`(minutes taken to fill the order), `attempt_status`(categorical status), `filled_action`(signed quantity filled)|
| `df_trades` | `pd.DataFrame` |Filled trades only. Columns include: `dt_local`, `price`, `action` |
| `df_bar` | `pd.DataFrame` |Bar‑level portfolio backtest.Columns include: `plot_time`, `open`, `high`, `low`, `close`, `position`,` holding_pnl`, `trade_cashflow`, `cash`,`market_value`, `portfolio_value` |
| `summary`<br>(agent_basline/<br>agent_w/_resub/<br>EPDF) | `pd.DataFrame` |One‑row summary of performance with metrics: `original fill rate`, `attempt fill rate`, `buy/sell fill rates`, `avg fill time`, `avg quote distance`, `avg slippage`, `avg attempts per order` |

-**Notes for computation**

 For every filled order, updates:
 - `trade_cashflow`-=`action`*`fill_price`
 - `position`+=`action`
 - `cash`+=`trade_cashflow`
 - `market_value`=`position`*`current_close`
 - `portfolio_value`=`cash`+`market_value`

 
### 4. Run_analysis function (Main)
The block first constructs `agent_file_map` which is a dictionary using `market` (folder name) as key, and values containing 'agent_code' and `path`. It then constructs `tick_size_map` from `tick_size_table`, outputing a dictionary with `market` as key and values being respective `tick_size`. With these dictionaries, corresponding datasets could be retrieved following user input value for `market`

The block contains main UI promting user input, and calls the `run_analysis` function. The core logic is executed via `rolling_backtest_with_optimal_params` function, taking hard-coded values of optimal hyperparamters found through Block 5 below. The algorithm outputs EPDF and states related plots, pnl comparison and execution performance metrics across three strategies. 

-**`process_order`**

  - Inputs

    `order_row`=np.array, `current_state`=tuple(int), `current_counts`=tuple(np.ndarray), `price_mode`=str, `max_tries`=int (=1 for 'agent_baseline'       strategy)
  
  - Logic

     As the inner function called in `rolling_backtest_with_optimal_params` loops, it simluates one order with resubmission logic. given the current       state classification and the most updated `RangeUp_count` and `Rangedown_count` from historical data (all full intervals before 'effective_bar_time' of     the  current order. Depending on the `price_mode` being 'agent' or 'EPDF', it set limit order at prices same as agent order records or with                   `find_order_price` function. Within the allowed `max_tries`, it proceeds through bars, each submission active for `tau_fill` minutes, and resubmit with       updated limit order price until filled. For each attempt, it updates a row of record in `df_exec` dataframe.
  
  - Output

    `records`=pd.DataFrame (`df_exec` for a particular order row in agent original record, may containing resubmission)
  
-**`rolling_backtest_with_optimal_params`**

  - Inputs
    
    `df_orders`=pd.DataFrame, `merged`=pd.DataFrame, `tau`=int, `state_threshold`=list, `state_count_lst`=list,`tick_size`=float, `best_params`=list,     `initial_cash`=0.0
  
  - Logic
    
    It loops through order rows in agent order record, for each order, it process all full intervals before its 'effective_bar_time', handling state classificaiton and EPDF updates, then call the `process_order` function to simulate order execution. No updates of states and EPDF occurs between       resubmission of the same order. After processing all orders, it calls  `inventory_backtest_from_trades` for each of the three strategies to generate PnL     comparison from `df_exec`
  
  - Output
    
    `results`=dictionary {'key'='strategy name', 'value'=[df_bar,df_trades,df_exec]}

-**`compare_strategy_pnl`**

 - Inputs:`results`=dict, `initial_cash`=0.0
 - Logic: Extracts final portfolio value, cumulative PnL, net position, filled quantity for each strategy.
 - Output: pd.DataFrame (one row per strategy)


### 5. (Marked down) Hyperparameter tuning analysis

In this part, we first tune the 4 hyperparameters in one market in the aforementioned order, and check the robustness of this approach by plotting `mod_score` of different value combinations.

Then the grid search process is looped through all markets to find market specific optimal sets of parameters. Note that instead of carrying out rolling EPDF and states updates, we split the data and construct EPDF on train data, find optimal parameters according to validation sets results, and output out-of-sample results using test set, with EPDF constructed on train+validation sets data. Therefore, three extra functions of order and market data processing are introduced.

This analysis is marked down with results as hard-coded dictionary so that it will not be rerun with the main algorithm

-**Key functions**
| Function| Inputs    | Description | Outputs    |
|:---------:|:-----------:|---------|:------------------------------------:|
| `precompute_state_table` | `merged`=pd.DataFrame,`tau`=int,<br>`state_threshold`=list,`m_period`=float  | Precomputing state table, so that when processing a specific order, we could look up its state from the last full interval's state classification |pd.DataFrame|
| `build_execution_orders_from_agent`| `df_order`=pd.DataFrame,`merged`=pd.DataFrame,<br>`df_state`,`RangeUp_count`,<br>`RangeDown_count`,`p_initial`,<br>`tick_size`,`tau_fill`,<br>`max_resubmits`,`price_mode`=str|Similar to `process_order` function, processing orders with resubmission logic| `df_exec`=pd.DataFrame|
|`build_epdf_model_from_merged`|`merged_sub`,`tau`,<br>`state_threshold`,<br>`m_period`,`state_count_lst`,<br>`tick_size` | build epdf from partial merged market data| `df_state_sub`, three `Range_counts` |
| `evaluate_one_parameter_setting` |`df_order`,`merged`,`df_state`,<br>`RangeUp_count`,`RangeDown_count`,<br>`tick_size`,`tau_fill`,<br>`max_resubmits`,`p_initial`|for a specific hyperparameter, loop over its possible values and run order execution process for agent resubmission and EPDF strategy, comparing performance of `fill_rate`,`avg_slippage_vs_open` and `final_port_value`  | pd.DataFrame|

-**Key pipelines**

  - `run_one_split_pipeline`

  Execute the complete trading strategy pipeline (state classification, order simulation, execution, backtesting) on a single data split (e.g., training, validation, or test set) for a given set of orders and market data. The goal is to produce performance metrics for two order‑placing strategies.

  - `run_one_market_with_splits`

  Calls `run_one_split_pipeline` and run the complete pipeline for a single market, but across three data splits: train, validation, and test. Directly applied to obtain out-of-sample result with the optimal parameter value on test set.

- `tune_one_market_parameters`
  
  Tune the hyperparameters for a single market using only the training and validation sets following the aforementioned order.

- `run_all_market_with_market_specific_params`

  For each market, perform market‑specific tuning with `tune_one_market_parameters`, then evaluate the best found parameters on the test set only with `run_one_split_pipeline`

-**Results** (`τ`=30, (`M`,`N`,`K`)=(2,4,3))

| Market                                      | p_initial | tau_fill | max_resubmits | m_period |
|:---------------------------------------------:|:-----------:|:----------:|:---------------:|:----------:|
| EuroStoxx                                   | 0.50      | 5        | 1             | 10       |
| GBP - British Pound                         | 0.75      | 25       | 5             | 2        |
| German Bunds - German Government Bonds      | 0.50      | 15       | 5             | 8        |
| Gold                                        | 0.65      | 15       | 5             | 6        |
| HeatingOil                                  | 0.50      | 5        | 6             | 6        |
| JPY - Japanese Yen                          | 0.90      | 10       | 4             | 2        |
| Nasdaq                                      | 0.95      | 5        | 4             | 2        |

## Limitations and futher improvements

### Hyperparameters are dependent on multiple factors
The optimal hyperparameter values are market‑specific, which is reasonable given that differences in volatility and liquidity across products affect fill rates and slippage. However, tuning was performed with fixed input values for `τ`, `M`, `N`, and `K`. Robustness tests show that many hyperparameter combinations yield very similar mod_score values, particularly for max_resubmits. This may explain why the optimal max_resubmits is large in certain markets.
For the decision‑related hyperparameters `p_initial`, `tau_fill`, and `max_resubmits`, a synergistic effect is expected: in less liquid markets, larger `tau_fill` and more `max_resubmits` should be required to achieve lower slippage and higher fill rates. However, products such as Heating Oil achieve the best validation performance with low `p_initial` and `tau_fill`, yet with many resubmissions. This can be explained by the observed high volatility in several periods (e.g., 30‑minute range movements of over 3,000 ticks). Instead of sacrificing PnL by setting conservative limit order prices (high `p_initial`) to increase fill rates, it proves more effective to give each order a short lifetime and resubmit it multiple times with timely updated limit prices.
Given the complex dynamics of different markets, it is difficult to provide concrete reasoning for why hyperparameters take specific optimal values, and they remain sensitive to user inputs. Our research on this specific set of user‑input values demonstrates that applying the optimal hyperparameters with the EPDF strategy improves slippage and PnL on test sets for six out of seven markets.
  
### Challenge of balancing fill rate and pnl when the agent incurs net losses
Agents trading EuroStoxx, German Bunds, and Heating Oil futures are making losses. Therefore, it is difficult to balance losses against fill rates, because increasing the fill rate would likely result in even greater losses. We navigated this by introducing the `mod_score` metric for hyperparameter selection, which depends on both the difference in PnL and the difference in fill rate between the agent with resubmission and the EPDF strategies. Nevertheless, the results remain sensitive to the fact that losses are incurred; the optimal `p_initial` values for these three markets are all low (0.5), and the EPDF strategy achieves a lower first‑attempt fill rate than the original agent.



    
