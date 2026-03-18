---
layout: default
title: Economic Inequality and Power Outage Duration
---

<style>
.page-header {
  background-image: url('assets/Powerlines.jpg');
  background-size: cover;
  background-position: center;
  background-color: transparent;
}
</style>

## Introduction

In this project, I examined a dataset of major power outages in the U.S. from January 2000 to July 2016. These major outages are defined by the Department of Energy as events that impacted at least 50,000 customers or caused an unplanned energy demand loss of at least 300 MegaWatts. The dataset was accessed from Purdue University's Laboratory for Advancing Sustainable Critical Infrastructure.

The dataset contains **1,534 outage events** with **51 columns** covering outage details, geographical and climatic characteristics, electricity consumption patterns, and economic characteristics of the affected states.

**Research Question:** Does a state's economic standing predict how long major power outages last? Specifically, do states with lower GDP per capita experience longer outage durations?

This question matters because if economically disadvantaged states consistently face longer outages, it suggests systemic inequities in infrastructure investment or recovery resources that energy companies and policymakers could address.

**Relevant columns for our analysis:**

| Column | Description |
|--------|-------------|
| `OUTAGE.DURATION` | Duration of the outage in minutes (our target variable) |
| `PC.REALGSP.REL` | State GDP per capita relative to the US average (1.0 = national average) |
| `CAUSE.CATEGORY` | The cause of the outage (e.g., severe weather, intentional attack) |
| `CLIMATE.CATEGORY` | Climate classification at the time of outage (warm, cold, normal) |
| `ANOMALY.LEVEL` | Climate anomaly level (El Niño/La Niña index) |
| `POPULATION` | State population |
| `POPPCT_URBAN` | Percentage of urban population in the state |
| `TOTAL.PRICE` | Average electricity price in the state (cents/kWh) |

---

## Data Cleaning and Exploratory Data Analysis

### Cleaning Steps

The raw dataset required several cleaning steps before analysis:

1. **Dropped 29 irrelevant columns**, including redundant economic indicators (`PC.REALGSP.USA`, which is the same for all states in a given year), redundant demographics (`POPPCT_UC`, `POPDEN_UC`, `POPDEN_RURAL`), land/water percentages, electricity sales breakdowns, and broken Excel formula columns (`RES.CUST.PCT`, `COM.CUST.PCT`, `IND.CUST.PCT`). I also dropped post-outage variables (`DEMAND.LOSS.MW`, `CUSTOMERS.AFFECTED`) that would not be known at the time of prediction.

2. **Replaced string `'NA'` with `np.nan`**: The data export process used the string "NA" as a placeholder for missing values. Converting to `np.nan` lets pandas handle them correctly in numeric operations and aggregations.

3. **Combined date and time columns**: Merged `OUTAGE.START.DATE` and `OUTAGE.START.TIME` into a single `OUTAGE.START` datetime column, as recommended in the dataset's special considerations.

After cleaning, the dataset has **1,534 rows and 21 columns**. Here is the head of the cleaned DataFrame:

| POSTAL.CODE | NERC.REGION | CAUSE.CATEGORY | OUTAGE.DURATION | PC.REALGSP.REL | CLIMATE.CATEGORY | OUTAGE.START |
|---|---|---|---|---|---|---|
| MN | MRO | severe weather | 3060 | 1.077 | normal | 2011-07-01 17:00:00 |
| MN | MRO | intentional attack | 1 | 1.090 | normal | 2014-05-11 18:38:00 |
| MN | MRO | severe weather | 3000 | 1.067 | cold | 2010-10-26 20:00:00 |
| MN | MRO | severe weather | 2550 | 1.071 | normal | 2012-06-19 04:30:00 |
| MN | MRO | severe weather | 1740 | 1.092 | warm | 2015-07-18 02:00:00 |

### Univariate Analysis

The histogram below shows the distribution of outage duration. It is heavily right-skewed — the vast majority of outages are resolved within a few thousand minutes, but a small number of extreme events drag the distribution far to the right.

<iframe src="assets/outage-duration-hist.html" width="800" height="600" frameborder="0"></iframe>

### Bivariate Analysis

The scatter plot below shows the relationship between state GDP per capita (relative to the US average) and outage duration, colored by cause category. Most long-duration outages are concentrated among states at or below the national average GDP, particularly for severe weather and fuel supply emergencies. High-GDP states rarely experience extremely long outages.

<iframe src="assets/gdp-vs-duration-by-cause.html" width="800" height="600" frameborder="0"></iframe>

### Interesting Stats

Grouping outages by economic tier reveals a clear pattern: median outage duration drops steadily as state wealth increases. Below-average GDP states have a median duration of 1,440 minutes, while high-GDP states have a median of just 270 minutes.

| Economic Tier | Mean (min) | Median (min) | Count |
|---|---|---|---|
| Low (<0.9) | 2,673.5 | 806.5 | 400 |
| Below Avg (0.9-1.0) | 3,850.4 | 1,440.0 | 303 |
| Above Avg (1.0-1.1) | 2,065.6 | 1,014.0 | 336 |
| High (>1.1) | 2,162.4 | 270.0 | 437 |

---

## Assessment of Missingness

### MNAR Analysis

I believe `OUTAGE.DURATION` is likely **MNAR** (Missing Not At Random) because the most extreme outages — particularly fuel supply emergencies lasting weeks — may have missing durations precisely because restoration was so prolonged or chaotic that exact times were never recorded. The missingness is likely related to the value of the duration itself. If I had access to utility company incident logs with independent restoration timestamps, we could verify whether the missing durations correspond to the most severe events, which would make the missingness MAR instead.

### Missingness Dependency

I performed permutation tests to analyze whether the missingness of `OUTAGE.DURATION` depends on other columns.

**Test 1 — Missingness depends on CAUSE.CATEGORY:** Fuel supply emergencies make up 22% of missing-duration outages but only 2.6% of non-missing ones. Using Total Variation Distance (TVD) as the test statistic, the permutation test yielded a very low p-value (≈ 0.0), confirming that the distribution of cause categories is significantly different for missing vs. non-missing duration rows.

<iframe src="assets/missingness-cause-category.html" width="800" height="600" frameborder="0"></iframe>

**Test 2 — Missingness does NOT depend on PC.REALGSP.REL:** The mean relative GDP is nearly identical for missing vs. non-missing rows (1.025 vs. 1.027). Using absolute difference in means as the test statistic, the permutation test yielded a high p-value, confirming that missingness of duration does not depend on state economic standing.

<iframe src="assets/missingness-gdp.html" width="800" height="600" frameborder="0"></iframe>

---

## Hypothesis Testing

**Null Hypothesis:** The median outage duration for states with below-average GDP (PC.REALGSP.REL < 1.0) is the same as for states with above-average GDP (PC.REALGSP.REL ≥ 1.0). Any observed difference is due to random chance.

**Alternative Hypothesis:** The median outage duration for below-average GDP states is greater than for above-average GDP states.

**Test Statistic:** Difference in medians (below-average minus above-average). I use median rather than mean because outage duration is heavily right-skewed, making the median a more robust measure.

**Significance Level:** 0.05

### Justification

I chose a permutation test because the extreme right skew in outage duration violates the normality assumptions of parametric tests like a t-test. The permutation test makes no distributional assumptions and directly simulates the null hypothesis by shuffling group labels. Splitting states at a relative GDP of 1.0 is a natural threshold since it represents the national average, creating two meaningful groups that directly address our research question.

<iframe src="assets/hypothesis-test.html" width="800" height="600" frameborder="0"></iframe>

### Results

**We reject the null hypothesis:** The observed difference in medians was 594 minutes (1,042 vs. 448 minutes) with a p-value of 0.0001, well below our significance level of 0.05. The data suggest that states with below-average GDP tend to experience longer outage durations than states with above-average GDP. However, since this is an observational study and not a randomized controlled trial, we cannot conclude that lower economic standing directly causes longer outages — confounding factors such as cause category or climate may also play a role.

---

## Framing a Prediction Problem

**Prediction Problem:** Predict the duration of a major power outage (in minutes).

**Type:** Regression, since outage duration is a continuous numeric variable.

**Response Variable:** OUTAGE.DURATION — this directly measures outage severity from the public's perspective: how long they go without power.

**Evaluation Metric:** RMSE (Root Mean Squared Error). I chose RMSE over Mean Absolute Error because RMSE penalizes large errors more heavily, which is appropriate here since underestimating a very long outage is more costly than being slightly off on a short one. RMSE is also in the same units as our target (minutes), making it easy to interpret.

**Time of Prediction Justification:** At the time a power outage begins, we would already know the state's economic characteristics (GDP per capita, utility sector investment), the cause of the outage, climate conditions, population and urbanization level, and electricity pricing. I deliberately exclude post-outage information like customers affected and demand loss, since those would not be available when trying to predict how long the outage will last.

---

## Baseline Model

### Methodology

My baseline model is a **Linear Regression** using two features:
- **PC.REALGSP.REL** (quantitative): state GDP per capita relative to US average — passed through without transformation
- **CAUSE.CATEGORY** (nominal): cause of the outage — one-hot encoded using `OneHotEncoder` with the first category dropped to avoid multicollinearity

These were chosen because my EDA and hypothesis test both showed they are associated with outage duration. All preprocessing and model training are implemented in a single sklearn `Pipeline`.

### Performance:

| Metric | Training | Test |
|---|---|---|
| RMSE | 4,915 min | 7,185 min |
| R² | 0.1590 | 0.1587 |

The model explains only about 16% of the variance in outage duration. The similar R² values on training and test data mean the model is **not overfitting**, but it is clearly **underfitting** — with only two features and a linear model, it lacks the capacity to capture the complexity of outage duration. The test RMSE of 7,185 minutes is very large relative to the mean duration of 2,625 minutes, confirming that this baseline is **not a good model** and motivating the improvements in the final model.

---

## Final Model

### Enhancements

I improved the baseline by adding five new features, applying appropriate transformations, switching to a nonlinear model, and tuning hyperparameters.

**New features:**

- **CLIMATE.CATEGORY** (nominal, one-hot encoded) was added because climate conditions directly affect the type and severity of weather-related outages. Cold climates face ice storms while warm climates face hurricanes, and each has a different recovery profile. One-hot encoding converts the three categories into binary columns so the model can learn separate effects for each.
- **ANOMALY.LEVEL** (quantitative, QuantileTransformer) captures El Niño and La Niña conditions, which increase the likelihood of extreme weather events that tend to cause longer outages. The QuantileTransformer maps the unevenly distributed raw values to a normal distribution, helping the model make better splits on this feature.
- **POPULATION** (quantitative, StandardScaler) was included because larger states have more complex grids and more customers to restore. StandardScaler centers the values around zero with unit variance, which matters because population is in the millions while other features are single digits.
- **POPPCT_URBAN** (quantitative, QuantileTransformer) reflects that urban areas have denser infrastructure that can be repaired more efficiently, while rural areas require longer travel times for repair crews. The QuantileTransformer normalizes the distribution since most states cluster at the high end of urbanization.
- **TOTAL.PRICE** (quantitative, QuantileTransformer) was added because electricity pricing reflects infrastructure investment levels — states with higher prices may have better-maintained grids. The QuantileTransformer spreads out tightly clustered price values so the model can distinguish between small but meaningful differences.

I also **log-transformed the target variable** to handle the extreme right skew (skew ≈ 8.0) in outage duration, which helps the model learn from both short and long outages rather than being dominated by extreme values.

**Model:** Random Forest Regressor, chosen because it can capture nonlinear relationships between features and duration that linear regression misses.

**Hyperparameter tuning** was performed using GridSearchCV with 5-fold cross-validation over:
- `n_estimators`: [100, 200] — more trees give more stable predictions
- `max_depth`: [5, 10, 20, None] — controls the overfitting/underfitting tradeoff
- `min_samples_split`: [2, 5, 10] — prevents the model from fitting to noise in small groups

**Best hyperparameters:** 200 trees, unlimited depth, minimum 10 samples per split.

The residual plot below shows the distribution of prediction errors across actual outage durations. Points near the red dashed line indicate accurate predictions, while points far from it represent larger errors. The model tends to underpredict for very long outages, which is expected given the extreme skew in the data.

<iframe src="assets/residual-plot.html" width="800" height="600" frameborder="0"></iframe>

### Performance Comparison

| Metric | Baseline (Test) | Final Model (Test) |
|---|---|---|
| RMSE | 7,185 min | ~3,667 min |
| R² | 0.1587 | ~0.20 |

<iframe src="assets/model-comparison.html" width="800" height="600" frameborder="0"></iframe>

The final model nearly halved the test RMSE compared to the baseline. While R² of 0.20 means the model still only explains about 20% of the variance, this is a meaningful improvement and suggests that outage duration is inherently difficult to predict from pre-outage information alone.

---

## Fairness Analysis

### The Test 

**Question:** Does our model perform worse for economically disadvantaged states?

**Group X:** States with below-average GDP (PC.REALGSP.REL < 1.0)

**Group Y:** States with above-average GDP (PC.REALGSP.REL ≥ 1.0)

**Evaluation Metric:** RMSE

**Null Hypothesis:** The model is fair — its RMSE is roughly the same for both groups, and any observed difference is due to random chance.

**Alternative Hypothesis:** The model is unfair — its RMSE is higher for below-average GDP states than for above-average GDP states.

**Test Statistic:** RMSE(below) - RMSE(above)

**Significance Level:** 0.05

<iframe src="assets/fairness-test.html" width="800" height="600" frameborder="0"></iframe>

### Results

The observed RMSE was 4,320 minutes for below-average GDP states and 2,954 minutes for above-average GDP states — a difference of 1,366 minutes. The permutation test yielded a p-value of 0.035. At the 0.05 significance level, we reject the null hypothesis. The model appears to perform unfairly, with higher prediction errors for economically disadvantaged states. This ties back to the core theme of our project: economic inequality is associated not only with longer outages, but also with greater difficulty in predicting outage duration.

---

## Conclusion

**Economic inequality is associated with longer power outages.** Throughout this project, every stage of analysis pointed to the same conclusion: states with lower GDP per capita tend to experience longer outage durations than their wealthier counterparts.

**Key findings:**

- States with below-average GDP have a median outage duration of 1,042 minutes, more than double the 448 minutes for above-average GDP states
- Our permutation test confirmed this difference is statistically significant (p = 0.0001), not just random chance
- The final Random Forest model cut prediction error nearly in half compared to the baseline, with test RMSE dropping from 7,185 to ~3,667 minutes
- The fairness analysis revealed that the model itself performs worse for economically disadvantaged states (p = 0.035), suggesting that outage patterns in poorer states are harder to predict

**What this means:** These findings suggest that economic disadvantage compounds during power outages — not only do poorer states experience longer disruptions, but the factors driving those durations are more complex and harder to model. This could reflect differences in infrastructure investment, repair crew availability, or the types of outages that disproportionately affect lower-income regions.

**Limitations:** This is an observational study, so we cannot claim that low GDP *causes* longer outages. Confounding factors like climate, geography, and grid age likely play a role. Additionally, our best model still only explains about 20% of the variance in outage duration, meaning much of what determines recovery time lies outside the features available in this dataset.

*By Gilad Segal*
