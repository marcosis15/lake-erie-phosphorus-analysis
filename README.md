# **Lake Erie Phosphorus & Orthophosphate Analysis**

## **Project Overview**
This project analyzes phosphorus and orthophosphate concentrations across Lake Erie shoreline monitoring station using EPA and USGS water quality data.The goal is to understand the drivers of nutrient pollution levels, a kew factor behind Lake Erie's recurring algal blooms, and to test whether a predictive model can estimate nutrient concentrations from location and seasonal data
Phosphorus and Orthophosphate were analyzes as two related but chemically distinct datasets: Orthophosphate is the immediate bioavailable, dissolved form of phosphorus, while phosphorus . . . They were kept sepearate throughout the analysis rather than merged, and compared at the end.


## **Data Source**
Data was pulled from the Water Quality Portal (a joint USGS/EPA database), filtered to:
Site Type: Lake, Resovoir, Impoundement 
Sample media: Water
Characteristic group: Nutrient

## **Data Cleaning**
Cleaning presented a real unit-standardization challenge: phosphorus measurements in this dataset came reported in multiple unit conventions — mg/L, µg/L, "as P" (elemental phosphorus basis), and "as PO4" (phosphate ion basis). These aren't interchangeable; converting between them requires a molar mass ratio (94.97/30.97 ≈ 3.066).

Where units were ambiguous (e.g. unlabeled "mg/L"), the assumption was made to treat them as "as P," per EPA's documented default convention for phosphorus reporting. Each dataset (Phosphorus, Orthophosphate) was standardized to a single internal unit basis (µg/L as P and µg/L as PO4, respectively) before analysis.

Station location data (latitude/longitude) was joined in via MonitoringLocationIdentifier, after first deduplicating the site metadata table to avoid row-multiplication during the merge.

## **SQL Exploration**

Both datasets were loaded into a SQLite database for exploratory querying, including:

Station-level averages, ranked to identify pollution hotspots
Yearly and monthly aggregate trends
A JOIN identifying 25 monitoring stations with both Phosphorus and Orthophosphate measurements — used later for the cross-dataset comparison

## **Machine Learning — Phosphorus**

Baseline (Linear Regression): Using Year and Month alone, R² = 0.22 — weak, suggesting time alone is a poor predictor.

Adding latitude/longitude to the linear model did not help (R² = 0.18) — a signal that the relationship between location and phosphorus levels isn't linear.

Random Forest, using Year, Month, Latitude, and Longitude, performed substantially better (R² = 0.74 on an initial single train/test split), confirming the relationship is non-linear and spatially driven.

Cross-validation: An initial cross-validation attempt using default (unshuffled) folds produced a misleading, highly unstable result (mean R² = -0.74 across folds, with one fold as low as -4.86). This happened because the data's inherent ordering (by date/station) meant folds weren't representative random samples. Switching to shuffled K-Fold cross-validation resolved this, producing a much more stable and trustworthy estimate: mean R² = 0.657 across 5 folds.

Hyperparameter tuning via GridSearchCV (shuffled folds) identified max_depth=10, n_estimators=300 as optimal, improving the cross-validated score slightly to R² = 0.676.

Feature importance revealed Longitude as the dominant predictor (0.59), far ahead of Month (0.16), Latitude (0.13), and Year (0.12) — suggesting phosphorus pollution is driven primarily by where along the shoreline a station sits (east-west position) rather than by season or long-term trend. This aligns with the known concentration of agricultural runoff sources (e.g., the Maumee River) in Lake Erie's western basin.

## **Machine Learning — Orthophosphate**

Given Orthophosphate's smaller sample size (217 rows vs. 438 for Phosphorus), a single train/test split proved unreliable (R² = -0.07). Rather than force a full modeling workup on an underpowered sample, Orthophosphate was evaluated using the same shuffled cross-validation and tuning pipeline as Phosphorus, primarily as a comparison/validation check against the Phosphorus findings, not as a standalone predictive model.

Shuffled CV Mean R²: 0.326
Tuned CV R²: 0.331
Dominant feature: Longitude (0.33), consistent with Phosphorus, though with a flatter importance distribution across all four features (Latitude 0.30, Month 0.21, Year 0.16)

## **Comparitive Conclusion**

Both nutrients, modeled independently, identified Longitude as the strongest predictor, despite being different chemical measurements analyzed with separate pipelines, they point to the same spatial pattern.

At the 25 monitoring stations where both nutrients were measured, station-level average Phosphorus and average Orthophosphate values showed a moderate positive correlation (r = 0.52) — further evidence that both nutrients tend to be elevated at the same locations, consistent with a shared pollution source (e.g., agricultural runoff) rather than independent, unrelated processes.

## **Limitations**

Orthophosphate's smaller sample size limits confidence in its model performance relative to Phosphorus.
Unlabeled unit conventions required a documented assumption ("as P" as default) rather than certainty from the source data.
Neither dataset includes environmental variables likely to matter (rainfall, upstream land use, discharge events), which may explain why even the best-performing model explains roughly two-thirds of variance rather than more.
The station correlation (r = 0.52), while meaningful, is based on only 25 shared stations — a modest sample for a correlation estimate.

## **Tools**
Python, pandas, SQLite (via sqlite3), scikit-learn (Linear Regression, Random Forest, cross-validation, GridSearchCV), matplotlib
