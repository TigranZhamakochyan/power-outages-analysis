# Lights out : Predicting Power Outage Duration in the U.S

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
| `CLIMATE.CATEGORY` | Climate classification (warm/cold/normal) at time of outage |
| `OUTAGE.START` | Timestamp of when the outage began (used to extract hour and day of week) |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning


1. I loaded the Excel file with skiprows=5 to skip the metadata rows at the top of the raw file, then dropped the extra header row and the `variables` column that came with the original dataset format.

2. I combined date and time columns into single. I used `OUTAGE.START` and `OUTAGE.RESTORATION` timestamp columns by concatenating the date and time strings and parsing with `pd.to_datetime`. Then I dropped the original four separate date/time columns.

3. I replaced all the 0 values with NaN in `OUTAGE.DURATION`, `CUSTOMERS.AFFECTED`, and `DEMAND.LOSS.MW` columns. A value of 0 in these columns is not physically meaningful, because an outage cannot last 0 minutes, and 0 customers affected or 0 MW lost indicates missing data rather than a true zero.

4. I also converted `YEAR` to int and `MONTH` to Int64 to fix float dtype issues caused by the Excel import. `MONTH` uses nullable integer type to preserve NaN values.

5. I kept only the 21 of the most relevant columns. I reduced it from the original 56 columns to those useful for analysis and modeling, including cause, climate, geographic, economic, and time-related features.

These cleaning steps ensure that missing data is properly represented as NaN, that timestamps are usable for feature extraction, and that the DataFrame is focused on relevant information.

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

The histogram below shows outage duration in hours (excluding extreme outliers above 500 hours). The distribution is heavily right-skewed as most outages resolve within 50 hours, but a small number last hundreds of hours. This motivated using a log transformation in the final model.

<iframe src="assets/duration_hist.html" width="800" height="500" frameborder="0"></iframe>

### Bivariate Analysis

The box plot below shows outage duration by cause category. Fuel supply emergencies have the highest median duration, while intentional attacks are resolved the quickest. This strongly supports using cause category as a predictor.

<iframe src="assets/box_cause.html" width="800" height="500" frameborder="0"></iframe>

The bar chart below shows median outage duration by month. December and September have the longest median durations (~26 hours). Interestingly, after creating this graph I looked at my model and deleted an engineered feature IS_SUMMER which provided bianary information whether the month is Summer. Before this graph I used to think that in Summers power outages happen more often.

<iframe src="assets/monthly_duration.html" width="800" height="500" frameborder="0"></iframe>

### Interesting Aggregates

The table below summarizes key statistics grouped by cause category, sorted by median duration. Fuel supply emergencies last longest on average (225 hours mean, 66 hours median), while islanding outages are resolved the fastest. Additionally, the large gap between mean and median for fuel supply emergencies, indicates that there are extreme outliers.

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

I believe the `DEMAND.LOSS.MW` column is likely NMAR. This column has 901 missing values, which is more than any other key column. The missingness is likely related to the value itself, as large demand losses are hard to measure accurately, so companies may not spend resources estimating them. Additionally, smaller outages may not have had demand loss recorded at all because it wasn't considered significant enough. To make this an MAR, we would need additional data about which utility companies reported each outage and their individual reporting practices.

### Missingness Dependency

I analyzed whether the missingness of `OUTAGE.DURATION` depends on other columns using permutation tests with TVD as the test statistic.

**Test 1: Missingness of `OUTAGE.DURATION` vs `CAUSE.CATEGORY`**

Null Hypothesis: The missingness of `OUTAGE.DURATION` is independent of `CAUSE.CATEGORY`. Any observed difference is due to random chance.

Alternative Hypothesis: The missingness of `OUTAGE.DURATION` depends on `CAUSE.CATEGORY`.

Test Statistic: TVD between the distribution of `CAUSE.CATEGORY` when duration is missing vs not missing.

Significance Level: 0.05

Observed TVD: 0.469\
P-value: 0.0\
I reject the null hypothesis. The missingness of `OUTAGE.DURATION` depends on `CAUSE.CATEGORY`, which means intentional attacks are much more likely to have missing duration than severe weather outages.


<iframe src="assets/tvd_cause.html" width="900" height="500" frameborder="0"></iframe>

**Test 2: Missingness of `OUTAGE.DURATION` vs `MONTH`**

Null Hypothesis: The missingness of `OUTAGE.DURATION` is independent of `MONTH`. Any observed difference is due to random chance.

Alternative Hypothesis: The missingness of `OUTAGE.DURATION` depends on `MONTH`.

Test Statistic: TVD between the distribution of `MONTH` when duration is missing vs not missing.

Significance Level: 0.05

Observed TVD: 0.143\
P-value: 0.182\
I fail to reject the null hypothesis. The missingness of `OUTAGE.DURATION` does not depend on `MONTH`.

<iframe src="assets/tvd_month.html" width="900" height="500" frameborder="0"></iframe>

---

## Hypothesis Testing

Null Hypothesis: Severe weather outages have the same mean duration as non-weather outages. Any observed difference is due to random chance.

Alternative Hypothesis: Severe weather outages last longer on average than non-weather outages.

Test Statistic: Difference in group means (severe weather mean − non-weather mean). This is appropriate because we are comparing a quantitative variable across two groups and have a directional alternative hypothesis.

Significance Level: 0.05

**Results:**
Mean duration severe weather: 3899.71 minutes
Mean duration non-weather: 1499.852 minutes
Observed difference: 2399.857 minutes
P-value: 0.0

I reject the null hypothesis. The data is consistent with the alternative hypothesis that severe weather outages last significantly longer than non-weather outages. Note that this does not prove causation — only that the association is unlikely due to chance.

<iframe src="assets/hyp_test.html" width="900" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

Prediction Problem: Predict the duration of a major power outage (`OUTAGE.DURATION`) in minutes.

Type: Regression, since outage duration is a continuous numerical variable with no natural categories.

Response Variable: `OUTAGE.DURATION`. I chose this because it directly answers what factors determine how long a power outage will last. Duration is also the most actionable metric for utility companies and emergency responders who need to plan resource allocation.

Metric: I chose RMSE over MAE because outage duration is heavily right-skewed with extreme outliers, which means some outages last hundreds of hours. RMSE penalizes these large errors more heavily, which reflects the real-world cost of severely underestimating how long an outage will last. Predicting a 10-hour outage as 2 hours is far more harmful than predicting a 3-hour outage as 2 hours, and RMSE captures this asymmetry better than MAE.

Time of Prediction Justification: At the moment an outage begins, a utility company would know the cause of the outage, the geographic location and climate region, the current climate conditions, and the time of day and day of week. They would for sure not know the restoration time, the total duration, or how many customers will ultimately be affected. Therefore, I only use features that are known at the start of the outage. I excluded `OUTAGE.RESTORATION` and `OUTAGE.DURATION` itself from the features, and I extracted only the start time components (`OUTAGE.HOUR`, `IS_WEEKEND`, `IS_BUSINESS_HOURS`) from `OUTAGE.START`.

---

## Baseline Model

The baseline model uses two features with Linear Regression:

`CAUSE.CATEGORY` is nominal, thus I encoded it with OneHotEncoder and used (drop='first') since it has no natural order
`MONTH` is ordinal, thus I left it as-is since it is already numerical

Performance: Baseline RMSE is 5289.73 minutes (~3.7 days). This is a bad result, which is a little dissapointing. The model essentially learns average duration per cause category and month, ignoring many important factors like location, climate, and time of day, which will be later added for final model.

---

## Final Model

The final model uses RandomForestRegressor with the following features:

Nominal (OneHotEncoded): `CAUSE.CATEGORY`, `CLIMATE.REGION`, `CLIMATE.CATEGORY` as different regions and climate types have different infrastructure and repair capabilities.

Quantitative (StandardScaled): `ANOMALY.LEVEL`, `CUSTOMERS.AFFECTED`, `POPPCT_URBAN` as the climate anomaly intensity, outage scale, and urbanization all affect repair speed.

Engineered features:
`LOG_POPULATION` is a log transform of population, which as the graphs showed is heavily right-skewed
`MONTH_SIN` + `MONTH_COS` I did cyclical encoding of month so December and January are treated as close together
`HOUR_SIN` + `HOUR_COS` are also cyclical encoded of outage start hour, which helps us to capturing day/night crew availability
`IS_WEEKEND` is a bianary feature that helps us to use the fact that weekend outages may last longer due to fewer available crews
`IS_BUSINESS_HOURS` is also a bianary feature and is used as outages starting during business hours have full crews available immediately

Hyperparameter tuning: GridSearchCV with 5-fold cross-validation over n_estimators, max_depth, min_samples_leaf, and criterion.

Best hyperparameters were identified criterion=squared_error, max_depth=None, min_samples_leaf=1, n_estimators=300

Performance:
Final Model RMSE: 4751.60 minutes
Baseline RMSE: 5289.73 minutes
Improvement: 538.13 minutes (about 10% better)

The improvement comes from using more informative features, Random Forest's ability to capture non-linear relationships, and proper cyclical and log encodings.

---

## Fairness Analysis

Group X: High population states (POPULATION above median)
Group Y: Low population states (POPULATION below median)

Evaluation metric: RMSE

Null Hypothesis: The model is fair. Its RMSE for high and low population states are roughly the same, and any differences are due to random chance.

Alternative Hypothesis: The model is unfair. Its RMSE differs between high and low population states.

Test statistic: Absolute difference in RMSE between the two groups.
Significance level: 0.05

Results:
RMSE High Population states: 5298.49 minutes
**RMSE Low Population states: 3929.65 minutes
Observed difference: 1368.84 minutes
P-value: 0.212

Since the p-value (0.212) is greater than 0.05, I fail to reject the null hypothesis. The difference in RMSE between high and low population states is not statistically significant and it could occur by random chance. My model does not appear to systematically perform worse for either group.

<iframe src="assets/fairness.html" width="900" height="500" frameborder="0"></iframe>
