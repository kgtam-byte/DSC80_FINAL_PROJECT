# ⚡ What Makes a Power Outage Severe?
Project for DSC 80 at UCSD
by: Kaitlyn Tam  
---

## Introduction

This project uses the **Major Power Outage Risks in the U.S.** dataset compiled by the Laboratory for Advancing Sustainable Critical Infrastructure (LASCI) at Purdue University. The dataset documents major power outage events across the continental U.S. from January 2000 to July 2016 — specifically, outages that affected at least 50,000 customers or caused an unplanned energy loss of 300+ megawatts.

Beyond the outage events themselves (when, where, how long), the dataset also includes regional characteristics such as climate patterns, land-use features, electricity consumption, and economic indicators that may contribute to outage risk.

**Central Question:** *What are the characteristics of major power outages with higher severity?*

Outage severity is measured using `OUTAGE.DURATION` (in minutes), since longer outages generally indicate greater disruption to infrastructure and consumers. Understanding which risk factors are associated with more severe outages could help energy companies prioritize preventive measures and predict where and when high-severity events are likely to occur.

The dataset contains **1,534 rows** and 57 columns. The most relevant columns for this project are:

| Column | Type | Description |
|---|---|---|
| `YEAR` | Quantitative | Year the outage occurred |
| `MONTH` | Ordinal | Month the outage occurred |
| `U.S._STATE` | Nominal | State where the outage occurred |
| `NERC.REGION` | Nominal | NERC reliability region involved |
| `CLIMATE.REGION` | Nominal | U.S. climate region (9 regions, defined by NCEI) |
| `CLIMATE.CATEGORY` | Nominal | Climate episode: Warm, Cold, or Normal (based on ONI index) |
| `CAUSE.CATEGORY` | Nominal | Category of the event cause |
| `OUTAGE.DURATION` | Quantitative | Duration of the outage in minutes **(target variable)** |
| `DEMAND.LOSS.MW` | Quantitative | Peak demand lost during the outage (megawatts) |
| `CUSTOMERS.AFFECTED` | Quantitative | Number of customers affected |
| `TOTAL.SALES` | Quantitative | Total electricity consumption in the state (MWh) |
| `POPULATION` | Quantitative | State population in the year of the outage |
| `POPPCT_URBAN` | Quantitative | Percentage of state population living in urban areas |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The raw Excel file contained metadata rows at the top, so I skipped the first 5 rows using `header=5` and removed the first data row (which contained units, not values) using `.loc[1:]`. I then selected the 23 columns most relevant to analyzing outage severity.

**Handling missing values:**
- **Categorical columns** (`U.S._STATE`, `NERC.REGION`, `CLIMATE.REGION`, `CLIMATE.CATEGORY`, `CAUSE.CATEGORY`): Missing values were filled with `'Unknown'` so they could be treated as their own category in downstream analysis, rather than being silently dropped.
- **Numeric columns** (e.g., `ANOMALY.LEVEL`, `TOTAL.SALES`, `POPULATION`, `POPPCT_URBAN`): Missing values were imputed with the **column median**. Outage-related variables tend to be right-skewed due to rare extreme events, so the median is more robust than the mean.

Note that `OUTAGE.DURATION`, `DEMAND.LOSS.MW`, and `CUSTOMERS.AFFECTED` were intentionally **not** imputed globally, since imputing the target variable before modeling would introduce bias. The `nan` values visible in `DEMAND.LOSS.MW` below reflect the non-trivial missingness in this column, which is explored further in the Assessment of Missingness section.

**Cleaned DataFrame (first 5 rows):**

|   YEAR |   MONTH | U.S._STATE   | CLIMATE.REGION     | CAUSE.CATEGORY     |   OUTAGE.DURATION |   DEMAND.LOSS.MW |   CUSTOMERS.AFFECTED |
|-------:|--------:|:-------------|:-------------------|:-------------------|------------------:|-----------------:|---------------------:|
|   2011 |       7 | Minnesota    | East North Central | severe weather     |              3060 |              nan |                70000 |
|   2014 |       5 | Minnesota    | East North Central | intentional attack |                 1 |              nan |                  nan |
|   2010 |      10 | Minnesota    | East North Central | severe weather     |              3000 |              nan |                70000 |
|   2012 |       6 | Minnesota    | East North Central | severe weather     |              2550 |              nan |                68200 |
|   2015 |       7 | Minnesota    | East North Central | severe weather     |              1740 |              250 |               250000 |

### Univariate Analysis

The choropleth below shows median outage duration by state. States in the **Northeast and Upper Midwest** (e.g., Michigan, New York, West Virginia) tend to have the longest median durations, suggesting these regions face structurally harder-to-resolve outages — possibly due to aging infrastructure or more frequent severe weather events.

<iframe
  src="assets/outage_duration_by_state.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

The bar chart below shows the distribution of demand loss severity across all major outages. The vast majority fall in the **Very Small (0–100 MW)** and **Small (100–500 MW)** categories — truly catastrophic losses above 3,000 MW are rare but represent the most severe events in this dataset.

<iframe
  src="assets/demand_loss_dist.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

### Bivariate Analysis

The box plot below compares demand loss (MW) across U.S. climate regions on a log scale. The **Southeast** and **West** show the highest median demand losses, while the **Northwest** shows notably lower losses — suggesting geography and regional grid infrastructure influence how severely outages affect peak demand.

<iframe
  src="assets/demand_loss_by_region.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

### Interesting Aggregates

The table below groups outages by climate region and shows mean and median demand loss and customers affected. The **West** and **Southeast** consistently show the highest average impact per outage, while the **Northwest** and **West North Central** regions show the lowest — reinforcing the geographic patterns seen in the bivariate plot above.

| CLIMATE.REGION     |   Mean CUSTOMERS.AFFECTED |   Mean DEMAND.LOSS.MW |   Median CUSTOMERS.AFFECTED |   Median DEMAND.LOSS.MW |
|:-------------------|--------------------------:|----------------------:|----------------------------:|------------------------:|
| Central            |                  126810   |               477.5   |                     76000   |                   200   |
| East North Central |                  138389   |               560.4   |                    111196   |                   240   |
| Northeast          |                  121960   |               537.4   |                     64971.5 |                    50.5 |
| Northwest          |                   81420   |               177.9   |                      8500   |                    10.5 |
| South              |                  183501   |               399.1   |                     79000   |                   215.5 |
| Southeast          |                  180540   |               761.5   |                     81000   |                   281.5 |
| Southwest          |                   39028.9 |               424.6   |                         0   |                     0   |
| Unknown            |                  125076   |               452.5   |                     60943   |                   170   |
| West               |                  194580   |               651.5   |                     59729   |                   180   |
| West North Central |                   47316   |               326     |                     34500   |                   128   |

---

## Assessment of Missingness

### MNAR Analysis

I believe `DEMAND.LOSS.MW` is likely **MNAR** (Missing Not at Random). The reasoning is based on the data generating process: demand loss measurements may be missing precisely *because* of the nature of the outage itself. Very small outages may not warrant precise demand loss reporting, while extremely large or chaotic outages (e.g., widespread storm damage) may have disrupted the measurement and reporting infrastructure entirely. In both cases, the probability that the value is missing depends on the unobserved value itself — which is the definition of MNAR.

To make this missingness MAR, we would want additional data such as the reporting utility's data collection protocols, whether the outage affected the utility's own monitoring systems, or independent third-party estimates of demand loss that could explain why certain events lack records.

### Missingness Dependency

I analyzed the missingness of `DEMAND.LOSS.MW` against two columns:

**Depends on `CAUSE.CATEGORY` (p < 0.05):** A permutation test using the max-min difference in missingness rates across cause categories yielded a p-value below 0.05. We reject the null hypothesis — the missingness of `DEMAND.LOSS.MW` does depend on cause category. Certain outage types (e.g., intentional attacks) have much lower missingness rates than others (e.g., fuel supply emergencies), suggesting that some outage types are harder to quantify or occur in situations where measurements are unavailable.

**Does NOT depend on `MONTH` (p > 0.05):** A permutation test using the absolute difference in mean month between missing and non-missing groups yielded a p-value above 0.05. We fail to reject the null — there is insufficient evidence that missingness in `DEMAND.LOSS.MW` depends on the month of the outage.

<iframe
  src="assets/missingness_permutation.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

---

## Hypothesis Testing

**Null Hypothesis:** The distribution of outage duration categories is the same for warm and normal climate episodes.

**Alternative Hypothesis:** The distribution of outage duration categories differs between warm and normal climate episodes.

**Test Statistic:** Total Variation Distance (TVD) — appropriate because both variables (`OUTAGE_DURATION_bins` and `CLIMATE.CATEGORY`) are categorical. TVD measures the total difference between two categorical distributions, capturing whether warm and normal climate episodes produce meaningfully different outage duration profiles.

**Significance Level:** α = 0.05 — a standard threshold that balances the risk of false positives against false negatives. Given that this is an exploratory analysis rather than a high-stakes decision, 0.05 is appropriate.

**Result:** The observed TVD was ~0.099. The permutation test yielded a p-value of **0.04**, which is less than our significance level. We reject the null hypothesis. The data are consistent with the claim that outage duration distributions differ across climate categories — suggesting that climate episode type may be associated with outage severity, though we cannot establish causation from this test alone.

<iframe
  src="assets/hypothesis_tvd.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

---

## Framing a Prediction Problem

**Prediction Task:** Predict the duration of a major power outage in minutes (`OUTAGE.DURATION`).

**Type:** Regression — the response variable is continuous.

**Response Variable:** `OUTAGE.DURATION`, chosen because longer outages generally indicate more severe events and greater disruption to infrastructure and consumers.

**Evaluation Metric:** Root Mean Squared Error (RMSE), because it measures prediction accuracy in minutes and penalizes large errors more heavily — important for a use case where severely underestimating a long outage has real consequences. RMSE is preferred over MAE here because large prediction errors (e.g., predicting a 1-hour outage when it lasts 5 days) are disproportionately costly.

**Time of Prediction Justification:** All features used (`CAUSE.CATEGORY`, `CLIMATE.REGION`, `CLIMATE.CATEGORY`, `SEASON`, `POPULATION`, `TOTAL.SALES`, `POPPCT_URBAN`) represent information that would be known at or before the onset of an outage — location, climate context, and cause classification are all available before the outage resolves.

---

## Baseline Model

**Model:** Linear Regression in a `sklearn` Pipeline with `ColumnTransformer` preprocessing.

**Features:**
- `CAUSE.CATEGORY` — **Nominal**. Encoded with `OneHotEncoder` since there is no natural ordering between cause categories (e.g., "severe weather" is not inherently greater than "intentional attack").
- `MONTH` — **Ordinal**, but treated as nominal via `OneHotEncoder` to capture non-linear seasonal patterns (both summer and winter can be high-severity seasons, which a linear encoding would miss).

**Performance:**
- Training RMSE: **4,754 minutes**
- Test RMSE: **7,114 minutes**

The baseline model is not particularly strong — a test RMSE of ~7,114 minutes (~4.9 days) is large relative to the dataset's median outage duration of roughly 1,500 minutes. This is expected given we are using only two features and a linear model, but it establishes a useful benchmark for improvement in the final model.

---

## Final Model

**Model:** Random Forest Regressor in a `sklearn` Pipeline, selected for its ability to capture non-linear interactions between features — important here because outage severity likely depends on complex combinations of location, climate, and cause that a linear model cannot capture.

**New Features Engineered:**
- `LOG_POPULATION` — log-transform of state population. Population is right-skewed; log-transforming compresses extreme values and makes the relationship with outage duration more linear. Larger populated states likely have more complex grid infrastructure, making outages harder to resolve quickly.
- `SEASON` — derived from `MONTH` by binning into Winter/Spring/Summer/Fall. Captures broad seasonal demand patterns more cleanly than raw month numbers, since both summer (heat waves) and winter (storms) are high-severity seasons.

**Additional Features:** `CLIMATE.REGION`, `CLIMATE.CATEGORY`, `TOTAL.SALES`, `POPPCT_URBAN` — all providing richer context about the regional grid and climate environment that influences how long outages last.

**Hyperparameter Tuning:** `GridSearchCV` with 5-fold cross-validation over `max_depth` in {5, 10, 20} and `n_estimators` in {100, 200}. Best parameters: `max_depth=5`, `n_estimators=100`.

**Performance:**
- Training RMSE: **3,644 minutes**
- Test RMSE: **6,947 minutes**

The final model improved test RMSE by ~167 minutes over the baseline (~2.4% improvement). The much lower training RMSE (3,644 vs 4,754) indicates the richer feature set and non-linear model better captures the underlying patterns in the data.

---

## Fairness Analysis

**Groups:**
- Group A: **Winter outages** (months 12, 1, 2, 3)
- Group B: **Summer outages** (months 6, 7, 8, 9)

**Evaluation Metric:** RMSE — consistent with the regression metric used throughout.

**Null Hypothesis:** The model is fair with respect to season. Its RMSE for Winter and Summer outages are roughly the same, and any observed difference is due to random chance.

**Alternative Hypothesis:** The model is unfair. Its RMSE for Winter outages differs from its RMSE for Summer outages.

**Test Statistic:** Absolute difference in RMSE between the two groups.

**Significance Level:** α = 0.05

**Result:** Observed RMSE difference = 7,476 minutes (Winter: 12,626 min, Summer: 5,149 min). Permutation test p-value = **0.2952**. We fail to reject the null hypothesis — there is insufficient evidence that the model performs significantly differently for Winter vs. Summer outages. Any observed difference is consistent with random variation.
