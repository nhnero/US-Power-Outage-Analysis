# US-Power-Outage-Analysis
Project in DSC80 at UCSD to analyze a US power outage dataset and find insights through data cleaning, data exploration, statistical tests, and SkLearn pipelines.

Nate Nero (nhnero@gmail.com)

## Introduction

Major power outages are dangerous, costly, and increasingly influenced by extreme weather and aging infrastructure. To better understand these outages, I analyze a comprehensive dataset compiled by Mukherjee, Nateghi, and Hastak (2018), which documents major U.S. power outages from January 2000 to July 2016. According to the Department of Energy, a “major outage” is one that affects at least 50,000 customers or results in a loss of 300 MW of firm load.

The dataset includes information about each event—weather conditions, geographic location, electricity consumption patterns, economic characteristics, and land-use information—making it valuable not just for understanding past outages but also for building predictive tools that can help energy companies, policymakers, and emergency managers plan for future risks.

For this project, I focus on one central question:

What characteristics are associated with higher outage severity, and can we predict the duration of a major power outage?

Outage duration is one of the most important measures of severity, reflecting how long customers remain without power and how much strain an event places on utilities and communities. Understanding what factors contribute to longer restoration efforts can help guide infrastructure investments, climate resilience planning, and emergency response readiness.

Severe outages impose billions of dollars in economic losses, threaten public safety, and are becoming more frequent as climate change causes the global environment to become more extreme. Modeling and predicting outage severity can provide insights into where the grid is most vulnerable.

⸻

Dataset Overview

The dataset contains 1534 rows (fill in with df.shape[0]) and 55 variables, but only a subset is directly relevant to predicting outage severity. Below are the key columns used in this project:

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

To prepare the outage dataset for analysis, I first standardized all date and time information. I converted the outage start and restoration dates into datetime objects and combined them with their corresponding time columns to create precise timestamps. From these timestamps, I extracted the year and month to support time-based analyses.

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