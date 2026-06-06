# When the Lights Go Out: Predicting Power Outage Duration in the U.S.

**By Tigran Zhamakochyan**

---

## Introduction

This project investigates major power outages in the continental U.S. from January 2000 to July 2016. I am particularly interested in this dataset as I come from a country where power outages happen often, and I believe that understanding what makes outages last longer has real consequences on people's daily lives.

**Main Question: What factors predict how long a power outage will last?**

This question matters because utility companies and emergency responders need to anticipate outage duration to allocate resources effectively. A model that predicts outage duration can help minimize harm to affected communities.

The dataset contains **1,534 rows**. The relevant columns are:

| Column | Description |
|--------|-------------|
| `OUTAGE.DURATION` | Duration of the outage in minutes (target variable) |
| `CAUSE.CATEGORY` | General cause of the outage (severe weather, intentional attack, etc.) |
| `CLIMATE.REGION` | U.S. climate region where the outage occurred |
| `ANOMALY.LEVEL` | Oceanic El Niño/La Niña index indicating climate conditions |
| `CUSTOMERS.AFFECTED` | Number of customers affected by the outage |
| `MONTH` | Month the outage started (captures seasonality) |
| `POPULATION` | State population at time of outage |
| `NERC.REGION` | North American Electric Reliability Corporation region |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The following cleaning steps were performed:

1. **Combined date and time columns** into single `OUTAGE.START` and `OUTAGE.RESTORATION` timestamp columns, then dropped the original four separate columns.
2. **Replaced 0 values with NaN** in `OUTAGE.DURATION`, `CUSTOMERS.AFFECTED`, and `DEMAND.LOSS.MW` since a duration or customer count of 0 is not physically meaningful.
3. **Converted `YEAR` to int and `MONTH` to Int64** to fix float dtype issues from the Excel import.
4. **Kept only relevant columns** — reduced from 56 columns to 22 most useful ones.

Here is the head of the cleaned DataFrame:

| YEAR | MONTH | U.S._STATE | NERC.REGION | CLIMATE.REGION | CAUSE.CATEGORY | OUTAGE.DURATION |
|------|-------|-----------|-------------|----------------|----------------|-----------------|
| 2011 | 7 | Minnesota | MRO | East North Central | severe weather | 3060.0 |
| 2014 | 5 | Minnesota | MRO | East North Central | intentional attack | 1.0 |
| 2010 | 10 | Minnesota | MRO | East North Central | severe weather | 3000.0 |
| 2012 | 6 | Minnesota | MRO | East North Central | severe weather | 2550.0 |
| 2015 | 7 | Minnesota | MRO | East North Central | severe weather | 1740.0 |

### Univariate Analysis

The plot below shows the distribution of major power outages by cause category. Severe weather is by far the most common cause (763 outages), followed by intentional attacks (418 outages). This shows that natural and human-caused factors dominate.

<iframe src="assets/cause_bar.html" width="800" height="500" frameborder="0"></iframe>

The histogram below shows outage duration in hours (excluding extreme outliers above 500 hours). The distribution is heavily right-skewed — most outages resolve within 50 hours, but a small number last hundreds of hours. This motivated using a log transformation in the final model.

<iframe src="assets/duration_hist.html" width="800" height="500" frameborder="0"></iframe>

### Bivariate Analysis

The box plot below shows outage duration by cause category. Fuel supply emergencies have the highest median duration (~66 hours), while intentional attacks are resolved the quickest (~2 hours median). This strongly supports using cause category as a predictor.

<iframe src="assets/box_cause.html" width="800" height="500" frameborder="0"></iframe>

The bar chart below shows median outage duration by month. December and September have the longest median durations (~26 hours). Interestingly, summer months do not have the longest durations, which influenced my feature engineering decisions.

<iframe src="assets/monthly_duration.html" width="800" height="500" frameborder="0"></iframe>

### Interesting Aggregates

The table below summarizes key statistics grouped by cause category, sorted by median duration. Fuel supply emergencies last longest on average (225 hours mean, 66 hours median), while islanding outages are resolved the fastest. Note the large gap between mean and median for fuel supply emergencies, indicating extreme outliers.

| CAUSE.CATEGORY | Count | Mean Duration (hours) | Median Duration (hours) | Mean Customers Affected |
|:-------------------------------|------:|----------------------:|------------------------:|------------------------:|
| fuel supply emergency | 38 | 225 | 66 | 1 |
| severe weather | 741 | 65 | 41 | 190972 |
| public appeal | 69 | 24 | 8 | 15999 |
| equipment failure | 54 | 31 | 4 | 105451 |
| system operability disruption | 120 | 12 | 4 | 211066 |
| intentional attack | 332 | 9 | 2 | 18753 |
| islanding | 44 | 3 | 1 | 7233 |

---

## Assessment of Missingness

### NMAR Analysis

I believe the `DEMAND.LOSS.MW` column is likely **NMAR**. This column has 901 missing values — more than any other key column. The missingness is likely related to the value itself: large demand losses are hard to measure accurately, so companies may not spend resources estimating them. Additionally, smaller outages may not have had demand loss recorded at all because it wasn't considered significant enough. To make this MAR, we would need additional data about which utility companies reported each outage and their individual reporting practices.

### Missingness Dependency

I analyzed whether the missingness of `OUTAGE.DURATION` depends on other columns using permutation tests with TVD as the test statistic.

**Test 1: Missingness of `OUTAGE.DURATION` vs `CAUSE.CATEGORY`**
- Observed TVD: 0.469
- P-value: 0.0
- At significance level 0.05, we reject the null hypothesis. The missingness of `OUTAGE.DURATION` **does depend** on `CAUSE.CATEGORY` — intentional attacks are much more likely to have missing duration than severe weather outages.

<iframe src="assets/tvd_cause.html" width="900" height="500" frameborder="0"></iframe>

**Test 2: Missingness of `OUTAGE.DURATION` vs `MONTH`**
- Observed TVD: 0.143
- P-value: 0.184
- At significance level 0.05, we fail to reject the null hypothesis. The missingness of `OUTAGE.DURATION` **does not depend** on `MONTH`.

<iframe src="assets/tvd_month.html" width="900" height="500" frameborder="0"></iframe>

---

## Hypothesis Testing

**Null Hypothesis:** Severe weather outages have the same mean duration as non-weather outages. Any observed difference is due to random chance.

**Alternative Hypothesis:** Severe weather outages last longer on average than non-weather outages.

**Test Statistic:** Difference in group means (severe weather mean − non-weather mean). This is appropriate because we are comparing a quantitative variable across two groups and have a directional alternative hypothesis.

**Significance Level:** 0.05

**Results:**
- Mean duration severe weather: 3899.7 minutes
- Mean duration non-weather: 1499.9 minutes
- Observed difference: 2399.9 minutes
- P-value: 0.0 (0 out of 10,000 permutations exceeded the observed difference)

We reject the null hypothesis. The data is consistent with the alternative hypothesis that severe weather outages last significantly longer than non-weather outages. Note that this does not prove causation — only that the association is unlikely due to chance.

<iframe src="assets/hyp_test.html" width="900" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

**Prediction Problem:** Predict the duration of a major power outage (`OUTAGE.DURATION`) in minutes.

**Type:** Regression — duration is a continuous numerical variable.

**Response Variable:** `OUTAGE.DURATION`. This directly answers our central question about what factors predict how long an outage will last.

**Metric:** RMSE (Root Mean Squared Error). RMSE penalizes large errors more heavily than MAE, which is important here since severely underestimating a long outage has real consequences for resource allocation.

**Time of prediction justification:** At the moment an outage begins, we know the location, cause, climate conditions, and time of day/year. We do NOT know the restoration time or duration. All features used in the model are available at the start of the outage.

---

## Baseline Model

The baseline model uses two features with Linear Regression:

- `CAUSE.CATEGORY` — **nominal**, encoded with OneHotEncoder (drop='first') since it has no natural order
- `MONTH` — **ordinal**, left as-is since it is already numerical

All steps are implemented in a single sklearn Pipeline.

**Performance:** Baseline RMSE = **5289.73 minutes** (~3.7 days). This is a poor result, which is expected given only 2 simple features. The model essentially learns average duration per cause category and month, ignoring many important factors like location, climate, and time of day.

---

## Final Model

The final model uses RandomForestRegressor with the following features:

**Nominal (OneHotEncoded):** `CAUSE.CATEGORY`, `CLIMATE.REGION`, `CLIMATE.CATEGORY` — different regions and climate types have different infrastructure and repair capabilities.

**Quantitative (StandardScaled):** `ANOMALY.LEVEL`, `CUSTOMERS.AFFECTED`, `POPPCT_URBAN` — climate anomaly intensity, outage scale, and urbanization all affect repair speed.

**Engineered features:**
- `LOG_POPULATION` — log transform of population, which is heavily right-skewed
- `MONTH_SIN` + `MONTH_COS` — cyclical encoding of month so December and January are treated as close together
- `HOUR_SIN` + `HOUR_COS` — cyclical encoding of outage start hour, capturing day/night crew availability
- `IS_WEEKEND` — weekend outages may last longer due to fewer available crews
- `IS_BUSINESS_HOURS` — outages starting during business hours have full crews available immediately

**Hyperparameter tuning:** GridSearchCV with 5-fold cross-validation over n_estimators, max_depth, min_samples_leaf, and criterion.

**Best hyperparameters:** criterion=squared_error, max_depth=None, min_samples_leaf=1, n_estimators=300

**Performance:**
- Final Model RMSE: **4751.60 minutes**
- Baseline RMSE: **5289.73 minutes**
- Improvement: **538.13 minutes** (~10% better)

The improvement comes from using more informative features, Random Forest's ability to capture non-linear relationships, and proper cyclical and log encodings.

---

## Fairness Analysis

**Group X:** High population states (POPULATION above median)
**Group Y:** Low population states (POPULATION below median)

**Evaluation metric:** RMSE

**Null Hypothesis:** Our model is fair. Its RMSE for high and low population states are roughly the same, and any differences are due to random chance.

**Alternative Hypothesis:** Our model is unfair. Its RMSE differs between high and low population states.

**Test statistic:** Absolute difference in RMSE between the two groups.
**Significance level:** 0.05

**Results:**
- RMSE High Population: 5941.34 minutes
- RMSE Low Population: 3039.46 minutes
- Observed difference: 2901.88 minutes
- P-value: 0.203

Since the p-value (0.203) is greater than 0.05, we fail to reject the null hypothesis. The difference in RMSE between high and low population states is not statistically significant — it could occur by random chance. Our model does not appear to systematically perform worse for either group.

<iframe src="assets/fairness.html" width="900" height="500" frameborder="0"></iframe>
