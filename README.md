# ev-adoptability-analysis

**Causal and predictive analysis of electric vehicle integration across Africa, China, and the EU.**

Quantitative codebase supporting the paper:

> Nsan, N. (2025). *Bridging Policy, Infrastructure, and Innovation: A Causal and Predictive Analysis of Electric Vehicle Integration Across Africa, China, and the EU.*

---

## Overview

This repository contains the full analytical pipeline for a multi-method study of electric vehicle (EV) adoption feasibility across three distinct regulatory and infrastructure environments: Sub-Saharan Africa, China, and the European Union.

The study combines **machine learning prediction**, **econometric causal inference**, and **time series analysis** to answer two core questions:

1. **What drives EV adoption feasibility?** — Which policy, infrastructure, and energy mix variables most strongly predict EV market share?
2. **Does causality hold?** — Are those relationships causal (panel fixed effects) or merely associative?

---

## Methodology

The analysis proceeds in five stages:

### 1. Feature Engineering & Composite Feasibility Score

Raw panel data is loaded for each entity (country/region) across multiple years. Two derived features are constructed:

**Charger Station Density (CSD):**
```
CSD = Charging Infrastructures / Population Density (per km²) × 100,000
```
This normalises charger counts for population and geographic spread — a more meaningful metric than raw infrastructure counts.

**Subsidy encoding:**
The categorical `Subsidies and Tax Exemption` field is binary-encoded (Yes = 1, No = 0) for compatibility with regression models.

**PCA-based Feasibility Score:**
A composite EV readiness index is constructed from three indicators:
- `EVAR (%)` — EV adoption rate
- `CSD` — charger density (derived above)
- `RES (%)` — renewable energy share in the electricity mix

These are standardised and compressed to a single principal component via PCA, then min-max scaled to a 0–100 index. This composite score forms both an exploratory metric and a predictive target.

---

### 2. Exploratory Data Analysis

- **Correlation heatmap** — Pearson correlations across all input features with publication-quality formatting
- **EVAR distribution by bin** — adoption rates stratified into five brackets (0–20%, 20–40%, 40–60%, 60–80%, >80%) to reveal regional clustering
- **Variable distributions by entity** — overlaid KDE/histogram plots for EVAR, RES, and CSD across Africa, China, and EU

---

### 3. Predictive Modelling — Four Algorithms

All models predict the EV Feasibility Score from the same feature set. A 70/30 stratified train-test split (stratified by entity) ensures geographic balance across both sets.

Features are standardised via `StandardScaler` before model fitting.

#### 3a. Linear Regression
Baseline model providing interpretable coefficients and a performance lower bound.

#### 3b. Random Forest (GridSearchCV-tuned)
Ensemble of 300 decision trees with hyperparameter tuning:
```python
RandomForestRegressor(n_estimators=300, max_depth=5)
```
Tuned via `GridSearchCV` across max_depth, min_samples_split, and n_estimators.

#### 3c. XGBoost
Gradient-boosted trees with learning rate scheduling:
```python
XGBRegressor(n_estimators=300, learning_rate=0.1, max_depth=5)
```

#### 3d. Neural Network (Keras / TensorFlow)
Compact feedforward network with regularisation:
```python
Dense(4, activation='relu', kernel_regularizer=l2(0.01))
Dense(1)
```
Trained with Adam optimiser, early stopping (patience=50 on validation loss), and L2 regularisation to prevent overfitting on the limited panel dataset.

**Evaluation metrics** reported for all models on both training and test sets:

| Metric | Description |
|---|---|
| R² | Proportion of variance explained |
| MSE | Mean squared error |
| MAE | Mean absolute error |

---

### 4. SHAP Explainability

SHAP (SHapley Additive exPlanations) values are computed for all four models to quantify feature contributions at the individual prediction level — moving beyond global feature importance to per-instance attribution.

| Model | SHAP method |
|---|---|
| Linear Regression | `shap.LinearExplainer` (interventional perturbation) |
| Random Forest | `shap.TreeExplainer` |
| XGBoost | `shap.TreeExplainer` |
| Neural Network | `shap.KernelExplainer` (background = 50 training samples) |

Summary beeswarm plots are generated for all four models and exported at 600 DPI for publication.

A **cross-model feature importance comparison table** aggregates normalised SHAP-derived importance scores across all four algorithms, enabling identification of variables that are robustly important regardless of model choice.

---

### 5. Causal Inference — Panel Fixed Effects

To move from association to causation, a **Panel OLS model with entity fixed effects** is estimated using the `linearmodels` package:

```python
from linearmodels import PanelOLS
```

Fixed effects control for all time-invariant unobserved heterogeneity at the entity level (geography, culture, historical infrastructure) — isolating the within-entity effect of policy and infrastructure changes on EV adoption over time.

The panel dataset is indexed by (Entity, Year) and the model is specified as:

```
EVAR(%) ~ CSD + RES(%) + GDP_per_capita + Sub_Tax + EntityFE + ε
```

This answers: *holding everything constant about a country, does increasing charger density cause higher EV adoption?*

---

### 6. Time Series Stationarity

Before interpreting trends in EVAR(%) over time, stationarity is assessed:

- **Augmented Dickey-Fuller (ADF) test** — tests the null hypothesis of a unit root (non-stationarity)
- **ARMA order selection** — `arma_order_select_ic` with AIC criterion determines optimal AR and MA lag structure for EVAR(%) time series by entity

This ensures that predictive models trained on panel data are not spuriously regressing on non-stationary trends.

---

## Key Variables

| Variable | Description | Source |
|---|---|---|
| `EVAR (%)` | EV adoption rate (% of new vehicle sales) | IEA / national statistics |
| `RES (%)` | Renewable energy share of electricity mix | Our World in Data / IEA |
| `CSD` | Charger Station Density (chargers per 100k people per km²) | Derived |
| `Sub & Tax` | Binary: subsidies or tax exemption in place (1/0) | Policy databases |
| `GDP per capita` | GDP per capita (USD) | World Bank |
| `PD` | Population density (people per km²) | World Bank |
| `lam` | Brooks-Corey pore size distribution index | — |
| `Feasibility_Score` | Composite PCA index (0–100) | Derived |

**Entities covered:** African nations (Ghana, Nigeria, South Africa, Kenya, Egypt and others), China, and major EU member states.

---

## Repository Structure

```
ev-adoptability-analysis/
├── EV_adoptability_workbook_nsan.ipynb   ← full analysis notebook
├── README.md
├── requirements.txt
└── data/
    └── Panel_Data_For_Analysis.xlsx      ← panel dataset (not included — see Data section)
```

---

## Reproducing the Analysis

```bash
git clone https://github.com/nhoyidi-nsan/ev-adoptability-analysis
cd ev-adoptability-analysis
pip install -r requirements.txt

# Place Panel_Data_For_Analysis.xlsx in data/
# Update the file path in Cell 2 of the notebook, then:
jupyter notebook EV_adoptability_workbook_nsan.ipynb
```

---

## Requirements

```
pandas>=1.5
numpy>=1.24
matplotlib>=3.7
seaborn>=0.12
scikit-learn>=1.2
xgboost>=1.7
tensorflow>=2.11
shap>=0.42
statsmodels>=0.13
linearmodels>=5.3
openpyxl>=3.1
```

Install all:
```bash
pip install -r requirements.txt
```

---

## Data Availability

The panel dataset (`Panel_Data_For_Analysis.xlsx`) is not included in this repository. It was compiled from the following public sources:

- **IEA Global EV Outlook** — EV adoption rates and charging infrastructure
- **Our World in Data** — Renewable energy share
- **World Bank Open Data** — GDP per capita, population density
- **National policy databases** — Subsidy and tax exemption status

A data dictionary is provided in the notebook (Cell 8, `X_.describe()`).

---

## Methodology Notes

**Why four models?**
Each model makes different assumptions. Linear regression is interpretable but assumes linearity. Tree-based methods (RF, XGBoost) capture non-linear interactions. The neural network provides a flexible functional form with regularisation. Agreement across models on feature importance strengthens the conclusions — disagreement reveals where non-linearity matters.

**Why SHAP over standard feature importance?**
Standard permutation importance conflates correlated features. SHAP values are grounded in cooperative game theory and give each feature its fair marginal contribution to each individual prediction — additive, consistent, and model-agnostic.

**Why fixed effects over pooled OLS?**
Africa, China, and the EU differ enormously in ways we cannot fully measure (political systems, culture, historical infrastructure). Pooled OLS would attribute these omitted differences to the observed variables, biasing coefficients. Entity fixed effects absorb all time-invariant unobservables, giving cleaner estimates of within-entity effects.

**Why ADF testing?**
EV adoption rates are trending upward in most entities. Regressing non-stationary series on each other produces spuriously high R² values. The ADF test confirms whether first-differencing is needed before time series modelling.

---

## Citation

If you use this code or methodology, please cite:

```
Nsan, N. (2025). Bridging Policy, Infrastructure, and Innovation: A Causal and
Predictive Analysis of Electric Vehicle Integration Across Africa, China, and the EU.
```
