# What Drives Medical Insurance Costs?

**Cleaning a messy insurance dataset and predicting annual medical charges with machine learning.**

Python · pandas · scikit-learn · CatBoost · XGBoost · matplotlib

---

## The question

Given a policyholder's age, sex, BMI, number of children, smoking status and region, can we predict their annual medical insurance charges, and which factors drive cost the most? This is the core question behind pricing a health insurance plan.

## Key findings

- **9,392 raw rows were only 1,334 real policyholders.** 76% of the rows were exact duplicates, 33 charges had a misplaced decimal point (10x too high), and 462 BMI values had been filled in with the column average.
- **Smokers cost 3.6x as much as non-smokers** ($32,050 vs. $8,916 average).
- **Smoking and obesity together are where costs explode.** For smokers, crossing BMI 30 nearly doubles average charges ($21,363 → $41,558). For non-smokers, BMI barely matters ($8,624 → $9,176).
- **The final model (tuned CatBoost) explains 81% of the variation** in log charges on unseen data, with a median prediction error of about **$1,000**.
- **Skipping the cleaning would have produced a misleading score.** On the raw data, 94% of test rows also appear in the training set, which inflates R² to 0.94 by memorization alone.

![Charges vs. BMI](images/bmi_vs_charges.png)

## Data cleaning

The raw file was deliberately messy. Each problem was found, checked and then fixed:

| Problem | How it was found | Fix |
|---|---|---|
| Charges stored as text (`"10,214.64"`) | Column type was text, not numeric | Removed `$` and `,`, converted to numbers |
| 7,160 exact duplicate rows (76%) | `df.duplicated()` | Removed them |
| 33 charges 10x too high (up to $442,607) | Every one had a twin record at exactly 1/10 the amount | Divided by 10 |
| 462 BMIs filled in with the mean (30.261) | One value appeared 25x more often than any other | Treated as missing |
| 489 missing smoker values | `isna()` | Recovered by merging partial duplicates of the same person |
| Partial duplicates (same person, different missing fields) | Same age, sex, children, region and charges | Merged into one row that keeps every known value |

**Result:** 1,334 clean, unique policyholders with only one value (a BMI) left to fill in.

## Exploratory analysis

![Charges vs. age](images/age_vs_charges.png)

Charges rise steadily with age, in **three distinct bands**: non-smokers at the bottom, and two smoker bands above them. The BMI chart at the top shows why: the upper smoker band is smokers with BMI ≥ 30.

## Modeling

- Target: `log(charges)`, because charges are heavily right-skewed. Predictions are converted back to dollars for evaluation.
- Added features: an obesity flag, smoker × obesity and smoker × BMI interactions, and age².
- The data was split 80/20 before any preprocessing. Scaling and encoding live inside scikit-learn `Pipeline`s, so no information leaks from the test set.
- Ten models were compared with 5-fold cross-validation, and the top three tree ensembles were tuned with `GridSearchCV`.

![Model comparison](images/model_comparison.png)

### Final results (tuned CatBoost, held-out test set)

| Metric | Value |
|---|---|
| R² (log charges) | **0.811** |
| R² (dollars) | 0.776 |
| Median absolute error | **$1,017** |
| Mean absolute error | $2,383 |
| Train/test R² gap | −0.006 (no overfitting) |

![Actual vs. predicted](images/actual_vs_predicted.png)

Most predictions fall right on the diagonal. The large errors come from a small group of policyholders, mostly non-smokers, who cost far more than their profile suggests. Those are likely unexpected medical events, which is exactly the kind of risk insurance pools.

## Data leakage: why cleaning mattered

| Same Random Forest model | Test R² |
|---|---|
| Trained on raw data (duplicates kept) | 0.942 (inflated) |
| Trained on cleaned data | 0.799 (honest) |

With the duplicates left in, 94% of test rows were copies of training rows, so the higher score mostly reflects memorization. The cleaned-data score is the realistic estimate for **new** policyholders.

## Next steps

- Model the unexplained high-cost group separately, for example with a classifier that flags likely high-cost policyholders.
- Fit a Gamma or Tweedie GLM, the standard actuarial approach to claim costs, and compare it with the tree models.
- Add prediction intervals so each estimate comes with a range.

## Repository structure

```
├── insurance_charges_analysis.ipynb   # full analysis
├── data/
│   ├── messy_insurance_2.csv          # raw data
│   └── insurance_clean.csv            # cleaned output (created by the notebook)
├── images/                            # charts used in this README
└── requirements.txt
```

## How to run

```bash
pip install -r requirements.txt
jupyter notebook insurance_charges_analysis.ipynb
```
