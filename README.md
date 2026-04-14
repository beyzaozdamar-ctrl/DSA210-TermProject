# DSA210-TermProject
# EDA and Hypothesis Testing

## Exploratory Data Analysis

The production data was collected from two raw hourly exports and then cleaned before analysis. During cleaning, blank footer rows and non-observation rows in the raw files were removed by keeping only records with both a valid date (`Tarih`) and hour (`Saat`). After cleaning, the hourly production data was aggregated to the daily level and merged with the daily weather data for Kayseri.

The final merged dataset contains 361 daily observations between October 1, 2024 and February 3, 2026. For each day, the following production indicators were calculated:

- `avg_ole`: daily average Overall Line Efficiency
- `total_production`: total units produced in that day
- `total_failures`: total reported failures
- `failure_rate`: total failures divided by total production
- `total_downtime`: total unplanned downtime in minutes
- `downtime_per_hour`: total downtime divided by the number of hourly observations
- `avg_workers`: average number of workers observed during the day

The weather variables used in the EDA were daily mean temperature, humidity, precipitation, cloud cover, wind speed, solar radiation, and weather condition labels.

### Main EDA Findings

- The average daily OLE in the merged dataset is approximately `0.689`.
- The mean daily failure rate is approximately `0.018`, which corresponds to about 1.8 failures per 100 produced units.
- Average downtime per observed production hour is about `8.28` minutes, although this metric is highly dispersed.
- Seasonal differences are visible in the production metrics. Average OLE is higher in winter and spring, and lower in summer and autumn.
- Rain alone does not appear to separate performance strongly, but high cloud cover and seasonal structure produce clearer differences.

### Seasonal Pattern in OLE

Average OLE by season:

- Winter: `0.721`
- Spring: `0.734`
- Summer: `0.659`
- Autumn: `0.651`

This indicates that the line operates more efficiently in winter and spring than in summer and autumn. Since production takes place indoors, this pattern may reflect indirect channels such as worker mood, commuting burden, routine changes, or seasonal production planning.

## Hypothesis Testing

The statistical testing stage was designed to evaluate whether weather conditions are associated with measurable changes in production performance. Because several variables are not normally distributed, both parametric and non-parametric tests were considered. Group comparisons were checked with Welch's t-test and Mann-Whitney U tests, while season-level comparisons were checked with one-way ANOVA and Kruskal-Wallis tests.

### Hypothesis 1

**H0:** Rainy days do not differ from non-rainy days in daily average OLE.  
**H1:** Rainy days have a different daily average OLE than non-rainy days.

Results:

- Rainy-day mean OLE: `0.6978`
- Non-rainy mean OLE: `0.6849`
- Welch's t-test p-value: `0.5592`
- Mann-Whitney p-value: `0.6373`

Conclusion:

This hypothesis is **not supported**. Rainfall by itself does not produce a statistically significant shift in daily OLE.

### Hypothesis 2

**H0:** Rainy days do not differ from non-rainy days in failure rate.  
**H1:** Rainy days have a different failure rate than non-rainy days.

Results:

- Rainy-day mean failure rate: `0.0173`
- Non-rainy mean failure rate: `0.0185`
- Welch's t-test p-value: `0.4698`
- Mann-Whitney p-value: `0.7424`

Conclusion:

This hypothesis is **not supported**. There is no statistically significant evidence that rainfall changes the daily failure rate.

### Hypothesis 3

**H0:** Overcast days do not differ from clearer days in daily average OLE.  
**H1:** Overcast days have a different daily average OLE than clearer days.

Results:

- Overcast-day mean OLE: `0.7301`
- Clearer-day mean OLE: `0.6733`
- Welch's t-test p-value: `0.0065`
- Mann-Whitney p-value: `0.0394`

Conclusion:

This hypothesis is **supported**. Cloudier days are associated with significantly different OLE values. Interestingly, the effect is opposite to the initial intuition: OLE is higher on overcast days in this dataset.

### Hypothesis 4

**H0:** OLE, failure rate, and downtime do not vary across seasons.  
**H1:** At least one season differs from the others.

Results:

- OLE seasonal ANOVA p-value: `0.0072`
- OLE seasonal Kruskal-Wallis p-value: `0.0027`
- Failure-rate seasonal ANOVA p-value: `0.0103`
- Failure-rate seasonal Kruskal-Wallis p-value: `0.0334`
- Downtime seasonal ANOVA p-value: `0.1119`
- Downtime seasonal Kruskal-Wallis p-value: `0.2310`

Conclusion:

This hypothesis is **partially supported**. Seasonal differences are statistically significant for OLE and failure rate, but not for downtime per hour.

## Correlation Analysis

Spearman correlation analysis was used because the production and weather variables are not all normally distributed. The strongest monotonic relationships were still relatively weak:

- Humidity vs OLE: `rho = 0.142`, `p = 0.007`
- Cloud cover vs OLE: `rho = 0.117`, `p = 0.026`

These coefficients are statistically significant but small in magnitude, which suggests that weather may contribute to performance variation, yet it explains only a limited share of daily changes on its own.

## Regression Check

To check whether the cloud-cover result was simply a seasonal artifact, an OLS model with robust standard errors was estimated:

`avg_ole ~ temperature + precipitation + cloud cover + humidity + average workers + season`

Key findings:

- Humidity remains positively associated with OLE (`p = 0.013`).
- Average worker count is strongly positively associated with OLE (`p < 0.001`).
- Cloud cover is no longer significant after controlling for season and worker count (`p = 0.651`).
- The model explains about `27.5%` of the variation in daily OLE.

This means the simple overcast-day difference observed in the EDA is likely mixed with broader seasonal or operational effects rather than reflecting an isolated cloud-cover effect.

## Overall Interpretation

The analysis does not provide strong evidence that rain directly lowers production performance. However, it does show that production efficiency and failure behavior vary significantly across seasons, and that some weather-related indicators such as humidity and cloudiness are associated with daily efficiency in the raw comparisons.

Because the factory is climate controlled, these findings should be interpreted cautiously. The weather signal may be acting through indirect channels such as mood, routine, commute difficulty, or staffing variation, but it may also be capturing seasonal production planning and workforce allocation. For that reason, the safest conclusion is that **weather-related context appears to be associated with production performance, but the effect is modest and partly confounded by operational factors**.
