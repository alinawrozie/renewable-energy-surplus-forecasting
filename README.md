# Forecasting Energy Surplus for Renewable Energy Services

## 1. Project Overview
This repository contains a production-grade machine learning project designed to predict renewable energy surplus (wind and solar) at least **24 hours in advance** for Brighton and Colchester. 

The primary business goal is to enable an energy provider to offer a **free electricity tariff** to local customers during periods of clean energy surplus. However, offering free electricity when there is no surplus incurs significant financial losses (since the provider must purchase deficit power from the grid at high spot-market prices). Therefore, the system is designed to **minimize false positives** and optimize operational margins using a cost-driven decision framework.

### Project Scope:
- **Dual Energy Resources:** Models both **Wind Energy** and **Solar Energy** generation using physics-based formulations.
- **Dual Locations:** Pipelines are trained, tested, and optimized separately for **Brighton** and **Colchester**.
- **Dual Forecasting Formulations:** Compares contemporaneous weather regression against true time-series sequence forecasting (24h ahead).
- **Cost-Optimized Decisions:** Integrates predictions with consumer load demand and optimizes decision thresholds under a financial penalty cost model.

---

## 2. Core Implementation Decisions

1. **Target Leakage Prevention:**
   - Designed a true **24-hour ahead forecasting model** where the feature matrix at time $t$ uses only weather and target observations lagged by 24 hours (i.e. features ending at $t-24$ are used to predict energy at $t$). This ensures the model is physically runnable in a real forecasting scenario.

2. **Temporal LSTM Sequence Dimensioning:**
   - Structured the data into proper 3D temporal tensors: `(n_samples, lookback_window, n_features)` where `lookback_window = 24` hours of historical weather/energy features ending at $t-24$ are processed by the LSTM to forecast the output at $t$. This correctly utilizes the recurrent network's memory capability.

3. **Preprocessing & Interpolation:**
   - Reindexed raw records to a full hourly grid leaving missing hours as `NaN`, then applied linear interpolation (`.interpolate(method='linear')`) to preserve weather feature continuity. Solar radiation features are constrained to absolute 0 during night hours (8 PM to 6 AM) for physical consistency.

4. **Chronological Time-Series Validation:**
   - Implemented a robust chronological train/test split (80% train, 20% test) before sequence building or scaling to guarantee no lookahead target leakage.

---

## 3. Physical Modeling & Assumptions

- **Air Density ($ho$):** Assumed standard value of $1.225 	ext{ kg/m}^3$.
- **Wind Energy ($E_w$):** Modeled for a wind farm of **20 Horizontal Axis Wind Turbines (HAWT)**. Each turbine has a blade length of $L = 59.8	ext{ m}$, yielding a sweep area of $A = \pi \cdot L^2 pprox 11,234.4	ext{ m}^2$.
  $$E_w = rac{1}{2} ho A (v_{	ext{m/s}})^3 \cdot 20 \cdot rac{1}{1000} \quad (	ext{kW})$$
  *(Wind speed is converted from kph to m/s by multiplying by $0.27778$.)*
- **Solar Energy ($E_s$):** Modeled for a local solar park with active panel surface area $A_{	ext{solar}} = 5,000 	ext{ m}^2$ and average efficiency $\eta = 18\%$.
  $$E_s = A_{	ext{solar}} \cdot \eta \cdot I_{	ext{radiation}} \cdot rac{1}{1000} \quad (	ext{kW})$$

---

## 4. Modeling Results (24h Ahead Forecasting)

Model performance is evaluated on the holdout test set (final 20% of the timeline) for both locations.

### Brighton (24h Ahead Forecasting)
| Model | Test MSE (kW^2) | Test MAE (kW) | Description |
| :--- | :---: | :---: | :--- |
| **Persistence Baseline** | 2,742,638,803 | 25,338.6 | Naive prediction: $y_t = y_{t-24}$ |
| **XGBoost Lagged** | 1,730,199,405 | 22,864.4 | Boosted trees on lagged features |
| **LSTM Sequence** | **1,641,647,572** | **22,845.3** | Recurrent sequence model (24h lookback) |

### Colchester (24h Ahead Forecasting)
| Model | Test MSE (kW^2) | Test MAE (kW) | Description |
| :--- | :---: | :---: | :--- |
| **Persistence Baseline** | 1,628,132,875 | 17,281.6 | Naive prediction: $y_t = y_{t-24}$ |
| **XGBoost Lagged** | 1,054,207,658 | 16,635.5 | Boosted trees on lagged features |
| **LSTM Sequence** | **985,035,180** | **16,105.1** | Recurrent sequence model (24h lookback) |

*The LSTM sequence model successfully out-performs both the naive persistence baseline and XGBoost at both physical sites.*

---

## 5. Cost-Optimized Decision-Making System

The typical daily consumer energy requirement is loaded from `requirement.csv`. We evaluate surplus as:
$$	ext{Surplus}_t = 	ext{Generation}_t - 	ext{Requirement}_t$$

### Operational Cost Model:
- **False Positive (Deficit) Cost ($C_{FP}$):** Offering free energy during a deficit. The company must buy spot-market power to cover customer demand. **Cost = £0.20 per kWh**.
- **False Negative (Opportunity) Cost ($C_{FN}$):** Underestimating surplus, resulting in wasted/curtailed green power. **Cost = £0.02 per kWh**.

### Threshold Optimization Results:
A naive threshold ($	ext{Surplus} > 0.0	ext{ kW}$) suffers high costs due to the 10x penalty of False Positives. We swept the predicted surplus threshold $	heta$ to minimize total cost:
- **Brighton:** The optimal surplus threshold is **15.0 MW (15,000 kW)**, reducing operational cost from £10.45M to £6.27M (**40.0% financial savings**, saving **£4.18M**).
- **Colchester:** The optimal surplus threshold is **14.8 MW (14,798 kW)**, reducing operational cost from £7.55M to £4.19M (**44.5% financial savings**, saving **£3.36M**).

---

## 6. Directory Structure & Execution
- `data/` - Folder containing raw weather CSV files (not uploaded to GitHub).
- `01_Data_Exploration.ipynb` - Processes raw weather files, models wind + solar energy, and outputs cleaned data.
- `02_Modelling.ipynb` - Fits models (XGBoost, LSTM), evaluates performance, and sweeps business decision thresholds.
- `Energy-Surplus-Forecasting-Presentation.pptx` - PowerPoint presentation showing methodology, comparison tables, and cost curves.
- `update_presentation.py` - Script used to generate the presentation deck.
- `requirements.txt` - Python package dependencies.
