# Term Project 2: Volatility-Volume-based Order Management Utilizing Statistical and Rule-based Techniques

This repository implements order slicing optimization on slippage cost for agent orders across 7 futures markets. Given 1-min OHLC data, through rolling state classification with interval volatility, volume and price change parameters, as well as emprical probabiltiy density function (EPDF) construction on price movements, the algorithm sets target price to place limit order, subject to fill rate and profit trade-off. The project compare performance of three strategies: the original agent orders as baseline, agent order with resubmission, and the EPDF strategy with optimized hyper-parameters on the target initial filled probabiltiy (p_initial), order live period (tau_fill), maximum allowed resubmission (max_resubmits) and half-life for EWMA and EWMV computation (m_peroids)

## Data Requirements

### Data Directory Structure
order_management/
├── data/
│ ├── EuroStoxx/
│ │ ├── AIAgent_EuroStoxx.csv # agent orders
│ │ ├── VGH22.csv # contract data
│ │ └── VGM22.csv # next contract
│ ├── GBP-British Pound
│ │ ├── AIAgent_GBPUSD.csv
│ │ ├── BPM20.csv
│ │ └── BPU20.csv
│ ├── German Bunds - German Government Bonds
│ │ ├── AIAgent_Bunds.csv
│ │ ├── RXM25.csv
│ │ ├── RXU25.csv
│ │ └── RXZ25.csv
│ └── ... (Gold, HeatingOil, JPY, Nasdaq)
