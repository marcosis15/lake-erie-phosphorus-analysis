# **Lake Erie Phosphorus & Orthophosphate Analysis**

## **Project Overview**
This project analyzes phosphorus and orthophosphate concentrations across Lake Erie shoreline monitoring station using EPA and USGS water quality data.The goal is to understand the drivers of nutrient pollution levels, a kew factor behind Lake Erie's recurring algal blooms, and to test whether a predictive model can estimate nutrient concentrations from location and seasonal data
Phosphorus and Orthophosphate were analyzes as two related but chemically distinct datasets: Orthophosphate is the immediate bioavailable, dissolved form of phosphorus, while phosphorus . . . They were kept sepearate throughout the analysis rather than merged, and compared at the end.


## **Data Source**
Data was pulled from the Water Quality Portal (a joint USGS/EPA database), filtered to:
Site Type: Lake, Resovoir, Impoundement 
Sample media: Water
Characteristic group: Nutrient
Note on geographic scope: the initial query was not restricted to specific counties, and a later coordinate check revealed a subset of returned stations (31 stations, 254 of 438 Phosphorus rows) fell well outside the Lake Erie basin, including sites near Columbus, OH, roughly 150 miles south of the lake. These were identified by checking station latitude/longitude against Lake Erie's known bounds and removed (filtered to latitude ≥ 41.0°N) before analysis. All results below reflect the corrected, geographically-validated dataset.

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

Baseline (Linear Regression): Using Year and Month alone, R² = 0.03 — essentially no predictive power, confirming time alone doesn't explain phosphorus levels.

Random Forest, using Year, Month, Latitude, and Longitude, performed considerably better on an initial single train/test split (R² = 0.53), indicating a non-linear, feature-driven relationship linear regression couldn't capture.

Cross-validation: An initial cross-validation attempt using default (unshuffled) folds produced a misleading, unstable result, since the data's inherent ordering (by date/station) meant folds weren't representative random samples. Switching to shuffled K-Fold cross-validation produced a stable, trustworthy estimate: mean R² = 0.422 across 5 folds.

Hyperparameter tuning via GridSearchCV (shuffled folds) identified max_depth=10, n_estimators=100 as optimal — matching the cross-validated baseline almost exactly (R² = 0.422), suggesting the default model was already close to its practical ceiling given the available features and sample size.

Feature importance revealed Latitude as the dominant predictor (0.47), ahead of Year (0.18), Month (0.17), and Longitude (0.17) — meaning phosphorus levels vary more strongly north-south than east-west across these stations. (Note: prior to the geographic correction described above, Longitude appeared dominant instead — a result skewed by the inclusion of inland, out-of-region stations. This is itself informative: it shows how much a geographic scoping error can distort a spatial feature-importance finding.)

# **Machine Learning — Orthophosphate**

Orthophosphate (188 rows) was evaluated using the same shuffled cross-validation and tuning pipeline as Phosphorus.

Shuffled CV Mean R²: 0.508
Tuned CV R²: 0.520
Dominant feature: Longitude (0.53), followed by Year (0.24) and Month (0.20) — Latitude had minimal influence (0.03)

Notably, Orthophosphate's model performed better than Phosphorus's on the corrected data (R² 0.52 vs 0.42), despite the smaller sample size — and identified a different dominant spatial axis (Longitude) than Phosphorus did (Latitude). This divergence is a genuine finding, not a contradiction: the two nutrients may be governed by somewhat different transport/loading mechanisms even at shared sites, which the station-level correlation below explores further.

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
