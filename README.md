# Term Project 2: Volatility-Volume-based Order Management Utilizing Statistical and Rule-based Techniques

## Overview
This repository implements order slicing optimization on slippage cost for agent orders across 7 futures markets. Given 1-min OHLC data, through rolling state classification with interval volatility, volume and price change parameters, as well as emprical probabiltiy density function (EPDF) construction on price movements, the algorithm sets target price to place limit order, subject to fill rate and profit trade-off. The project compare performance of three strategies: the original agent orders as baseline, agent order with resubmission, and the EPDF strategy with optimized hyper-parameters on the target initial filled probabiltiy (p_initial), order live period (tau_fill), maximum allowed resubmission (max_resubmits) and half-life for EWMA and EWMV computation (m_peroids)

## Data Requirements
Market OHLC and agent order data should be organized in the following structure with exact naming format, the program maps market folder name to corresponding agent order file's market identifier as the market variable.
The notebook should be placed under the same root directory as the order management folder.
### Data Directory Structure
order management/
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

## Structure and Application of the notebook
