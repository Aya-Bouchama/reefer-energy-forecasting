# Reefer Energy Consumption Forecasting â€” ARIMAX vs XGBoost

Forecasting the electrical consumption of **refrigerated containers (reefers)** on board a container ship, using
real operational data from a CMA CGM vessel. Research internship at the
**LIS lab (Laboratoire d'Informatique et des SystÃ¨mes, CNRS / Aix-Marseille UniversitÃ©)**, as part of **TNTM**, a
research partnership between CMA CGM and LIS on the digital transformation of maritime transport.

> **TL;DR** â€” On a 25-day Singapore â†’ Valencia voyage (12 power sources, 159 reefers, 2 h time step),
> a pooled **XGBoost** model beats a per-source **ARIMAX** baseline: **11.4 % vs 16.3 %** mean relative RMSE at
> equivalent memory, and **7.8â€“7.9 %** once trained on 7 voyages and enriched with contextual features.

![Relative RMSE per source â€” ARIMAX vs XGBoost](figures/rmse_arimax_vs_xgboost.png)

---

## Context

Reefers keep perishable goods at a fixed **setpoint** (from âˆ’30 Â°C to +20 Â°C) for weeks, which makes them one of
the largest and most variable electrical loads on a container ship. Anticipating this load helps plan on-board power
generation, reduce fuel consumption and emissions.

Consumption is not measured per container but at **power sources** (reefer panels), each feeding a group of reefers
located in the same area of the ship (Bay / Tier / Row). The target is therefore an **aggregated signal** per source.

## Data

| | |
|---|---|
| Vessel | CMA CGM Grace Bay |
| Reference voyage | **Leg 30**, Singapore â†’ Valencia, 11 Jan â€“ 4 Feb 2025 (24.7 days) |
| Pooled training | 7 legs (21, 22, 23, 24, 30, 34, 44) |
| Target | Electrical consumption per power source (kW), resampled to **2 h** |
| Exogenous variables | Ambient air temperature (2 m), reefer setpoints, equipment age |
| Scope (Leg 30) | 12 deck power sources, 159 reefers |

Filters: deck positions only (exposed to sun and weather), at least 3 reefers per source, and sources consuming less
than 1 kW for â‰¥ 95 % of the voyage are discarded.

> **The data is confidential (CMA CGM) and is not included in this repository.** See [`data/README.md`](data/README.md)
> for the expected files and structure.

## Method

**Evaluation protocol** â€” chronological split **80 % train / 10 % validation / 10 % test** for every source.
All choices (ARIMAX order, XGBoost hyperparameters, window size, memory order) are made on the **validation** set;
the test set is only used once for the final score (no optimistic bias).
Metric: **RMSE** and **relative RMSE** = RMSE / mean real consumption on the test period, so that sources of very
different sizes (â‰ˆ 8 kW to 110 kW on average) can be compared.

### 1. ARIMAX (statistical baseline)

$$\hat{y}(t) = c + \sum_{i=1}^{p} \phi_i\, y(t-i) + \sum_{j=1}^{q} \theta_j\, \varepsilon(t-j) + \beta\, T_{amb}(t)$$

- One model per source, order $(p, q) \in \{0,1,2\}^2$ selected by grid search (ambient temperature is almost
  linearly correlated with consumption, Ï â‰ˆ 0.99).
- **V0 â€” one-shot**: coefficients frozen, the whole test period predicted at once.
- **Multi-horizon study**: V0 error at 2 h, 6 h, 12 h, 24 h and 48 h.
- **V2 â€” rolling update**: every 2 h the real observed value is fed back to the model (no re-estimation), compared with a
  naive persistence baseline.
- **Coefficient analysis**: explains why the rolling update has no effect on some sources (AR/MA root cancellation).

### 2. XGBoost (machine learning)

- A single **pooled** model for all sources, hyperparameters (`learning_rate`, `max_depth`, `n_estimators`)
  tuned on validation with early stopping.
- **Equivalent-memory comparison** with ARIMAX: temperature + consumption at $t-1$ ($p=1$) or $t-1, t-2$ ($p=2$),
  $p$ selected per source on validation.
- **Feature enrichment**: rolling consumption statistics (window 4â€“12 h tuned on validation),
  **setpoint** composition (5 classes, thermal lift, setpoint entropy) and **equipment age** (4 classes, mean,
  dispersion), trained on Leg 30 only or pooled over 7 legs.

## Results (Leg 30, test set)

| Model | Training data | Mean relative RMSE |
|---|---|---|
| ARIMAX V0 â€” one-shot | Leg 30 | 16.3 % |
| ARIMAX V2 â€” rolling update | Leg 30 | 9.9 % |
| XGBoost â€” equivalent memory ($p$ = 1 or 2) | Leg 30 | 11.4 % |
| XGBoost â€” temperature + consumption history | 7 legs | 8.09 % |
| XGBoost â€” + setpoint features | 7 legs | **7.77 %** |
| XGBoost â€” + equipment age features | 7 legs | 7.90 % |

**Key findings**

- ARIMAX performance is driven by the **volatility** of each source: the coefficient of variation explains the ranking
  of sources much better than the number of reefers (correlation 0.84 vs âˆ’0.65).
- The one-shot ARIMAX **drifts** with the horizon: on volatile sources the error almost doubles between 2 h and 48 h.
- The rolling update (V2) reduces the error by 35â€“54 % on 9 of 12 sources, but has no effect when AR and MA roots
  nearly cancel out.
- At **equal information**, XGBoost is about **30 % better** than ARIMAX (better on 8 of 12 sources); ARIMAX stays
  better on the smoothest, most regular sources.
- Pooling several voyages helps more than adding contextual features: setpoint and age only bring a marginal,
  source-dependent gain.

| One-shot (V0) vs rolling (V2) ARIMAX | XGBoost vs ARIMAX V0 |
|---|---|
| ![V0 vs V2](figures/arimax_v0_vs_v2_source146.png) | ![XGBoost vs ARIMAX](figures/xgboost_vs_arimax_source103.png) |

![Multi-horizon evaluation](figures/arimax_horizons_source127.png)

## Repository structure

```
.
â”œâ”€â”€ notebooks/
â”‚   â””â”€â”€ reefer_consumption_forecasting.ipynb   # full pipeline: data â†’ ARIMAX â†’ XGBoost (sections follow the report)
â”œâ”€â”€ data/
â”‚   â””â”€â”€ README.md                              # expected (confidential) input files
â”œâ”€â”€ figures/                                   # figures used in this README
â”œâ”€â”€ report/
â”‚   â””â”€â”€ internship_report_FR.pdf               # full internship report (in French)
â”œâ”€â”€ requirements.txt
â””â”€â”€ README.md
```

## How to run

```bash
git clone https://github.com/Aya-Bouchama/reefer-energy-forecasting.git
cd reefer-energy-forecasting
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
# put the data files in data/ (see data/README.md), then:
jupyter notebook notebooks/reefer_consumption_forecasting.ipynb
```

The whole notebook runs in about 10 minutes on a laptop CPU.

## Tech stack

Python Â· pandas Â· NumPy Â· statsmodels (SARIMAX) Â· XGBoost Â· scikit-learn Â· Plotly

## Author

**Aya Bouchama** â€” engineering student at Centrale MÃ©diterranÃ©e (IAM).
