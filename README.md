# Rockfall Hazard Model

Model branch of **[AI Rockfall Detection & Prevention](../../tree/main)** — a binary classifier
that scores slope-failure risk from hydro-geotechnical conditions.

> The operator dashboard, system architecture and full project documentation live on the
> [`main`](../../tree/main) branch.

---

## Contents

| File | Purpose |
|---|---|
| `rockfall_ml_model.py` | Training notebook (Colab export) — the run that produced the artifact |
| `rockfall_ml_pipeline.py` | Generalised pipeline variant of the same workflow |
| `best_rockfall_model_logistic_regression.pkl` | Serialised model package — estimator, scaler, feature names, metrics |
| `training_dataset.csv` | Full labelled pool — 9,864 rows |
| `test_dataset.csv` | Held-out test split — 1,973 rows |
| `validation_dataset.csv` | Validation split — 1,973 rows |

---

## Data

9,864 labelled observations, near-balanced at **50.3 % positive**, over six features:

| Feature | Unit | Range |
|---|---|---|
| `Rainfall_mm` | mm | 0 – 300 |
| `Rainfall_3Day` | mm | 0 – 600 |
| `Rainfall_7Day` | mm | 0 – 1000 |
| `Temperature_C` | °C | 5 – 40 |
| `Soil_Strain` | — | 0 – 0.01 |
| `Pore_Water_Pressure_kPa` | kPa | 20 – 200 |

`Label` is 1 where conditions indicate imminent failure.

---

## Model

Six candidates were benchmarked under 5-fold cross-validated F1 — Logistic Regression, Random
Forest, Gradient Boosting, XGBoost, LightGBM and an MLP. **Logistic Regression won** and is the
serialised artifact; on this data the extra capacity of the ensembles bought nothing.

The `.pkl` is a package, not a bare estimator — it carries the fitted `StandardScaler`, the
feature order and the recorded metrics alongside the model, so inference cannot drift from the
preprocessing it was trained with.

### Measured performance

Held-out test split, n = 1,973:

| Metric | Test | Validation |
|---|---:|---:|
| Accuracy | **0.9782** | 0.9696 |
| Precision | 0.9817 | 0.9660 |
| Recall | 0.9748 | 0.9738 |
| F1 | **0.9782** | 0.9699 |
| ROC-AUC | **0.9758** | 0.9714 |

Confusion matrix on test: **TN 963 · FP 18 · FN 25 · TP 967**.

A false negative — a failure scored as safe — is the costly error in this domain. 25 of 992
positive cases are missed, so **recall is the metric to optimise**, and the operating threshold
should be moved off the 0.5 default before any field deployment.

---

## Limitations

> [!WARNING]
> These caveats matter more than the headline score. Read them before reusing this model.

- **The signal is one feature.** `Rainfall_mm` correlates with the label at **0.897**; every
  other feature is below **0.01**. Standardised coefficients agree — Rainfall_mm at **+3.86**,
  all others under **0.10**.
- **A trivial baseline nearly matches it.** The rule `Rainfall_mm > 50` alone reaches **0.9713**
  accuracy against the model's 0.9782.
- **The data appears synthetic**, with labels derived from a rainfall rule. Antecedent rainfall,
  strain and pore pressure are established drivers of real slope failure, yet contribute nothing
  here — so this does not reflect field behaviour.
- **The splits overlap.** `test_dataset.csv` and `validation_dataset.csv` are exact subsets of
  `training_dataset.csv`. The metrics above are valid because the training run held its own
  split out, but **re-running the notebook with a different seed will leak** train into test.
  Split from the pool yourself instead.

The pipeline is sound and the numbers are honest; the dataset is simply too easy to demonstrate
real predictive skill. Validation against measured slope-failure telemetry is the work that
would make this model meaningful.

---

## Usage

```bash
pip install scikit-learn pandas numpy joblib xgboost lightgbm matplotlib seaborn
python rockfall_ml_model.py          # retrain end to end
```

Loading the trained artifact:

```python
import joblib, pandas as pd

pkg = joblib.load("best_rockfall_model_logistic_regression.pkl")
model, scaler = pkg["model"], pkg["scaler"]

X = pd.read_csv("test_dataset.csv").drop("Label", axis=1)[pkg["feature_names"]]
risk = model.predict_proba(scaler.transform(X))[:, 1]   # P(failure)
```

`pkg` also exposes `model_name`, `feature_names`, `performance` and `training_info`.
