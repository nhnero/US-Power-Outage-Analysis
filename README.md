Project in DSC80 at UCSD to analyze a US power outage dataset and find insights through data cleaning, data exploration, statistical tests, and SkLearn pipelines.

Nate Nero (nhnero@gmail.com)

## Introduction

Major power outages are dangerous, costly, and increasingly influenced by extreme weather and aging infrastructure. To better understand these outages, I analyze a  dataset compiled by Mukherjee, Nateghi, and Hastak (2018), which documents major U.S. power outages from January 2000 to July 2016. According to the Department of Energy, a “major outage” is one that affects at least 50,000 customers or results in a loss of 300 MW of firm load.

The dataset includes information about each event—weather conditions, geographic location, electricity consumption patterns, economic characteristics, and land-use information—making it valuable not just for understanding past outages but also for building predictive tools that can help energy companies, policymakers, and emergency managers plan for future risks.

For this project, I focus on one central question:

What characteristics are associated with higher outage severity, and can we predict the duration of a major power outage?

Outage duration is one of the most important measures of severity, reflecting how long customers remain without power and how much strain an event places on utilities and communities. Understanding what factors contribute to longer restoration efforts can help guide infrastructure investments, climate resilience planning, and emergency response readiness.

Severe outages impose billions of dollars in economic losses, threaten public safety, and are becoming more frequent as climate change causes the global environment to become more extreme. Modeling and predicting outage severity can provide insights into where the grid is most vulnerable.

⸻

Dataset Overview

The dataset contains 1534 rows and 55 variables, but only a subset is directly relevant to predicting outage severity. Below are the key columns used in this project:

⸻

Relevant Columns for Understanding and Predicting Severity

1. OUTAGE.DURATION (minutes)

The total length of the outage, from loss of power to full restoration.
Target variable for prediction and main measure of severity.

2. OUTAGE.START.DATE, OUTAGE.START.TIME, OUTAGE.RESTORATION.DATE, OUTAGE,RESTORATION.TIME

Time indicators for when the outage began and ended.
Useful for identifying trends, seasonality, or long-term changes in the grid.

3. U.S._STATE, POSTAL.CODE, NERC.REGION

Geographic identifiers.
Outage severity may depend on regional climate, or infrastructure age.

4. CLIMATE.REGION, ANOMALY.LEVEL, CLIMATE.CATEGORY

Weather and climate variables including El Niño/La Niña conditions.
These help evaluate whether specific climate patterns contribute to more severe outages.

5. CAUSE.CATEGORY, CAUSE.CATEGORY.DETAIL, HURRICANE.NAMES

Descriptions of what caused the outage like storms, equipment failure, hurricanes.
Different causes are associated with  different outage durations.

6. CUSTOMERS.AFFECTED

Number of customers impacted by the event.
Often correlated with restoration difficulty—larger outages may take longer to repair.

7. DEMAND.LOSS.MW

Megawatts of demand lost during the outage.
Provides context about event scale and grid impact.

8. POPULATION, POPPCT_URBAN, POPDEN_RURAL / URBAN / UC

Population and density metrics that may influence outage impacts and restoration logistics.

9. Electricity consumption variables

(RES.SALES, COM.SALES, IND.SALES, TOTAL.SALES, etc.)
These provide insight into the energy demand characteristics of each state.

## Data Cleaning and Exploratory Data Analysis

To prepare the outage dataset for analysis, I first standardized all date and time information. I converted the outage start and restoration dates into datetime objects and combined them with their corresponding time columns to create precise timestamps. From these timestamps, I extracted the year and month.

Next, I addressed data quality issues caused by the data collection process. Several numeric columns (such as outage duration, customers affected, and demand loss) contained placeholder values like 0 or "NA" that do not represent true measurements. These values were replaced with NaN so they would not bias summary statistics or visualizations.

Finally, I standardized state identifiers by mapping full state names to USPS abbreviations, which enabled geographic visualizations such as choropleth maps. These cleaning steps ensured that the analyses reflect meaningful outage behavior rather than artifacts of missing or improperly recorded data.

Here is the first couple of rows and columns from the cleaned DataFrame:

|   OUTAGE.DURATION | STATE_ABBREV   |   CUSTOMERS.AFFECTED | OUTAGE.START        | OUTAGE.RESTORATION   |
|------------------:|:---------------|---------------------:|:--------------------|:---------------------|
|              3060 | MN             |                70000 | 2011-07-01 17:00:00 | 2011-07-03 20:00:00  |
|                 1 | MN             |                  nan | 2014-05-11 18:38:00 | 2014-05-11 18:39:00  |
|              3000 | MN             |                70000 | 2010-10-26 20:00:00 | 2010-10-28 22:00:00  |
|              2550 | MN             |                68200 | 2012-06-19 04:30:00 | 2012-06-20 23:00:00  |
|              1740 | MN             |               250000 | 2015-07-18 02:00:00 | 2015-07-19 07:00:00  |

### Univariate Analysis

This plot shows the average outage length (minutes) in each state. This plot shows that states in the Midwest and Northeast tend to have longer outages.
 <iframe
  src="assets/outage_map.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

### Bivariate Analysis

This plot shows the relationship between the number of customers affected and outage duration. There seems to be no clear correlation between the two. 
This could mean that there could be outages that affect many customers but are quickly fixed and that there are outages that affect many customers and 
take a long time to fix.
 <iframe
  src="assets/bi_scatter.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

### Interesting Aggregate

This aggregate shows the average duration by year and cause.

|   YEAR |   equipment failure |   fuel supply emergency |   intentional attack |   islanding |   public appeal |   severe weather |   system operability disruption |
|-------:|--------------------:|------------------------:|---------------------:|------------:|----------------:|-----------------:|--------------------------------:|
|   2000 |             nan     |                  nan    |              nan     |     nan     |         nan     |          3783.33 |                        970      |
|   2001 |             494     |                  nan    |              nan     |     nan     |         256.333 |         10140    |                        711.778  |
|   2002 |             nan     |                  nan    |              nan     |     nan     |         nan     |          5991    |                        204.333  |
|   2003 |             520.167 |                  nan    |             1329.5   |     nan     |        1548     |          6299.47 |                       2528.57   |
|   2004 |             268.4   |                  nan    |              nan     |     nan     |        3559.29  |          5064.96 |                         96.3333 |
|   2005 |             145     |                  nan    |              nan     |     nan     |         nan     |          6047.94 |                        228.75   |
|   2006 |             159     |                  nan    |              nan     |     nan     |         357     |          4015.09 |                        229      |
|   2007 |             317.833 |                 1676    |              nan     |      47     |        2430.5   |          2896.3  |                        459.25   |
|   2008 |             360.222 |                17578.3  |              nan     |     308.5   |        6057.5   |          5065.12 |                        382.143  |
|   2009 |            8624.8   |                 6570    |              nan     |     380.25  |         384     |          3905.22 |                        229.333  |
|   2010 |             330.6   |                 9660    |              nan     |     111     |        1351.57  |          3763.82 |                       2288.92   |
|   2011 |              18     |                 8265    |              840.403 |     346.667 |        1648.62  |          3229.91 |                        579      |
|   2012 |             105     |                 6487.5  |              356.157 |      58.5   |        1020.67  |          4127.62 |                        385.5    |
|   2013 |             292.5   |                 7624.33 |              363.853 |     239.857 |         nan     |          2678.92 |                        160.333  |
|   2014 |             nan     |                22617.7  |              778.429 |      56     |         570.4   |          1426.37 |                         52      |
|   2015 |             nan     |                  255    |              213.293 |     146.75  |         336.75  |          2123.08 |                        375      |
|   2016 |             nan     |                34072    |              490.12  |     223     |         nan     |          1640.4  |                        947.125  |

## Assessing Missingness

### NMAR Analysis

I believe the column DEMAND.LOSS.MW could be NMAR (Not Missing At Random). Utilities might omit reporting megawatt demand loss when the true value is either very small, or difficult to estimate meaning the probability that the value is missing depends on the unobserved value itself. If the missingness is instead MAR, additional info would be needed like info about which utilities had advanced metering, how reporting requirements changed over time, or whether certain utilities systematically used customers affected as an easier substitute metric.

### Missingness Dependency

I performed a permutation test using the Total Variation Distance (TVD) to test whether missingness in DEMAND.LOSS.MW depends on CAUSE.CATEGORY. The observed TVD was 1.8022, and after 5,000 permutations, the resulting p-value was 0, indicating that none of the shuffled datasets produced a statistic as extreme as the observed one. This provides strong evidence against the null hypothesis and shows that missingness in DEMAND.LOSS.MW depends on outage cause, meaning the data are not missing completely at random.

Null Hypothesis: Missingness is independent of outage cause

Alt Hypothesis: Missingness varies by cause category

<iframe
  src="assets/missing_dependence_perm.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

## Hypothesis Testing

Null: The distribution of outage durations is the same across all cause categories, and any differences in group means are due to random chance.

Alt: At least one cause category has a mean outage duration that differs significantly from what would be expected under random chance.

I used a permutation test with the Total Variation Distance (TVD) as the test statistic. TVD is a good choice here because it captures aggregate differences across all cause categories rather than focusing on a single pairwise comparison, making it well-suited for detecting overall dependence between outage duration and cause.

I used a significance level of 0.05.

The observed TVD was 20,910. After generating a permutation distribution with 5,000 random shuffles of outage durations across cause categories, the resulting p-value was 0, meaning none of the permuted test statistics were as large as the observed TVD.

Since the p-value is far below the significance level, there is strong evidence against the null hypothesis. This suggests that outage duration is associated with cause category, and that differences in mean outage duration across causes are unlikely to be due to random chance alone. This could also suggest that the cause
categories could help with predicing outage duration and severity.


## Framing a Prediction Problem

My model will try to predict outage serverity as a multiclass classification problem. The goal is to predict whether a power outage will be short, medium, or long, using DURATION_BIN, which is created by splitting OUTAGE.DURATION into three quantile-based categories. Duration is a natural measure of severity and is especially important for planning emergency response and resource allocation.

I evaluate models using macro F1 score instead of accuracy. Macro F1 balances precision and recall and treats all severity classes equally. This makes macro F1 a better metric for comparing both the baseline and final models.

All features used in the model (e.g., outage cause, region, climate indicators, anomaly levels, and population density) are information that would be available at or near the start of an outage.

## Baseline Model

For the baseline model, I trained a multiclass classification Decision Tree to predict outage duration category (short, medium, or long).

The model uses four features that would be known at the time of prediction:

Three nominal categorical features:
    CAUSE.CATEGORY, NERC.REGION, and CLIMATE.REGION

One ordinal feature:
    YEAR

All categorical variables were one-hot encoded using OneHotEncoder, while the numerical feature was passed through without transformation.

Model hyperparameters (max_depth and min_samples_leaf) were selected using 5-fold cross-validation with macro F1 score as the evaluation metric. The final baseline model achieved a macro F1 score of 0.588 on the held-out test set.

Overall, this baseline model performs moderately well but is not super strong, particularly because it struggles to accurately classify the medium-duration outages and relies on limited feature information.

## Final Model

For my final model I added the following features: CAUSE.CATEGORY, NERC.REGION, CLIMATE.REGION, CLIMATE.CATEGORY, YEAR, ANOMALY_Z, DEMAND.LOSS.MW, CUSTOMERS.AFFECTED, POPDEN_URBAN, and POPDEN_RURAL, along with missingness indicators for demand loss and customers affected. I used a RandomForestClassifier and got a macro F1 score of 0.638.

I added ANOMALY_Z to capture unusually extreme climate conditions, which are more likely to cause longer outages. DEMAND.LOSS.MW and CUSTOMERS.AFFECTED reflect the scale of an outage, while their missingness indicators help the model learn patterns related to incomplete reporting. Population density features capture differences in infrastructure and restoration speed between urban and rural areas. Climate and regional features account for geographic and environmental differences in outage behavior.

The best parameters GridSearchCV found were
criterion: gini
max_depth: 10
min_samples_leaf: 1
n_estimators: 100

Since the macro F1 score improved from the baseline value of 0.588 to 0.638, this shows that the final model provides better and more balanced predictive performance.

## Fairness Analysis

For the fairness analysis, I compared model performance between two geographic groups based on climate region.
Group X (Warm regions): South, Southeast, Southwest
Group Y (Cold regions): Northeast, Northern Rockies & Plains, Upper Midwest

The evaluation metric was macro F1 score.

Null Hypothesis: The model is fair with respect to climate region; the macro F1 score for warm regions and cold regions is the same, and any observed difference is due to random chance.

Alt Hypothesis: The model is unfair; the macro F1 score differs between warm and cold regions.

I used the difference in macro F1 scores between the two groups (warm − cold) as the test statistic. This statistic directly measures whether the model performs systematically better for one group than the other.

I ran a permutation test with 2,000 permutations, randomly shuffling group labels while keeping predictions fixed. I used a significance level of 0.05.
Observed test statistic: 0.025
p-value: 0.3275

Since the p-value (0.3275) is much larger than the significance level of 0.05, it fails to reject the null hypothesis. This suggests that there is no statistically significant evidence that the model performs differently for warm versus cold climate regions. Any observed difference in macro F1 score is consistent with random variation rather than the model being unfair.