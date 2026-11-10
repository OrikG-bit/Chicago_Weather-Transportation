# Chicago weather analysis and handoff

Weather-only analysis of the repository's Open-Meteo export. Transportation data and analysis are unchanged. No weather–ridership merge or transit-effect estimation has been performed.

## Findings

The hourly source is structurally clean: **146,448 rows**, no missing measurements, no duplicate UTC timestamps, and no missing hourly timestamps. It covers **2010-01-09 00:00 UTC through 2026-09-23 23:00 UTC**. These checks establish internal consistency, not independent measurement accuracy.

After conversion to `America/Chicago`, the merge-ready dataset contains **6,101 complete local calendar days, 2010-01-09 through 2026-09-22**. The first local date, January 8, 2010, has only six hours; September 23, 2026 has only 19 hours. Both are retained in the coverage audit and excluded from the merge-ready file. The complete days include 17 spring-forward days with 23 hours and 16 fall-back days with 25 hours.

Seasonality is substantial. Using the **15 complete calendar years, 2011–2025**, mean daily temperature is **−1.99°C in winter** and **22.37°C in summer**. January's mean is −3.65°C and July's is 23.47°C. Mean daily precipitation is highest in summer (3.71 mm/day), while winter has the most snowfall (0.54 cm/day). These are descriptive sample averages, not official 30-year climate normals.

Daily thresholds identify the following exposures over all 6,101 complete days. Conditions overlap; their percentages must not be added.

| Exposure | Definition | Days | Share |
|---|---|---:|---:|
| Wet | Precipitation ≥1 mm | 2,141 | 35.09% |
| Heavy precipitation | Precipitation ≥25 mm | 116 | 1.90% |
| Snow | New snowfall ≥0.1 cm | 649 | 10.64% |
| Heavy snow | New snowfall ≥10 cm | 11 | 0.18% |
| Cold | Hourly minimum ≤−10°C | 370 | 6.06% |
| Severe cold | Hourly minimum ≤−20°C | 31 | 0.51% |
| Hot | Hourly maximum ≥30°C | 333 | 5.46% |
| Very hot | Hourly maximum ≥35°C | 20 | 0.33% |
| Windy | Maximum hourly 10 m wind ≥40 km/h | 179 | 2.93% |
| Freeze/thaw exposure | Minimum <0°C and maximum >0°C | 1,046 | 17.14% |

The composite of heavy precipitation, heavy snow, cold, hot, and wind occurs on **964 days (15.80%)**. There are **45 days with two of these conditions**, and no days with more than two in this dataset. The flags are exploratory definitions, not official weather warnings. For rare conditions, especially heavy snow, estimates in the eventual transit analysis may be unstable. `threshold_sensitivity.csv` provides alternative cutoffs rather than implying one definition is uniquely correct.

## Extreme events and the January 2019 case study

| Measure | Local date | Value |
|---|---|---:|
| Lowest sampled hourly temperature | 2019-01-31 | −32.5°C |
| Highest sampled hourly temperature | 2022-06-21 | 37.4°C |
| Largest daily precipitation | 2023-07-02 | 137.8 mm |
| Largest daily new snowfall | 2015-02-01 | 19.74 cm |
| Largest sampled hourly 10 m wind | 2019-11-27 | 57.4 km/h |

These are extremes within this export at its single grid location, not official Chicago records. They were retained because the internal checks did not identify invalid measurements; they have not been independently validated against station records.

The existing transportation notebook focuses on January 27–February 2, 2019. Weather during that window was:

| Local date | Minimum °C | Mean °C | Maximum °C | Snowfall cm |
|---|---:|---:|---:|---:|
| 2019-01-27 | −19.3 | −16.44 | −11.6 | 0.00 |
| 2019-01-28 | −13.4 | −5.80 | 1.0 | 7.49 |
| 2019-01-29 | −22.5 | −17.38 | −14.3 | 0.07 |
| 2019-01-30 | −30.8 | −28.34 | −22.8 | 0.00 |
| 2019-01-31 | −32.5 | −25.31 | −17.0 | 1.12 |
| 2019-02-01 | −17.0 | −12.32 | −9.6 | 0.28 |
| 2019-02-02 | −10.5 | −0.67 | 3.9 | 0.00 |

January 29–31 forms a three-day severe-cold spell. January 30 remained below −20°C at every sampled hour. Snowfall on January 28 preceded the deepest cold, supporting the inclusion of previous-day and preceding-three-day snowfall features. These features measure prior snowfall, not remaining snow on the ground. This is a weather-based reason to study the episode; it does not establish the size or cause of a ridership change.

## Source, units, and transformations

The raw file is `datasets/open-meteo-41.86N87.65W179m.csv`. Its metadata identifies latitude **41.862915°N**, longitude **87.648770°W**, elevation 179 m, timezone GMT, UTC offset zero. The repository's `Notes` links to the Open-Meteo historical-weather API. The export does not record the exact model, download timestamp, or API request. Treat it as a gridded historical weather series rather than a verified station observation. The original file is unchanged, with its SHA-256 stored in `quality_report.json`.

1. Read the hourly table after the three metadata/padding rows. Validate schema, timestamp continuity, missingness, numeric finiteness, nonnegative amounts/speeds, broad temperature plausibility, and rain ≤ total precipitation.
2. Interpret original timestamps as UTC, then convert to `America/Chicago`. Construct `service_date` from the local date. Removing timezone information before conversion would assign some evening weather to the wrong day.
3. Calculate local-day temperature mean/min/max, precipitation/rain/snowfall sums, sustained-wind mean/max, maximum hourly precipitation, and exposure-hour counts. Means use the actual sampled hours, including DST days; minima/maxima are sampled hourly extrema, not continuous station extrema.
4. Compare observed hours with the true length of each local date. Export partial days separately in the audit; do not fill or scale their totals. No imputation, winsorization, or removal of valid extremes is applied.
5. Add calendar fields, transparent condition flags, overlapping-condition counts, peak-hour summaries, lagged exposures, and a descriptive monthly temperature reference.

**Precipitation and snowfall cannot be added directly.** Precipitation is total liquid-water equivalent in mm; rain is a component of precipitation. Snowfall is new-snow depth in cm, not snow depth or a separate additive amount of liquid precipitation. No fixed snow-to-water conversion is assumed. Total precipitation and rain are strongly correlated (**Pearson r = 0.980**); avoid treating them as independent predictors without considering collinearity. `weather_correlations.csv` and its chart show the remaining relationships; pooled correlations also contain seasonality.

Peak windows use records labeled **07:00, 08:00, 09:00** and **16:00, 17:00, 18:00 Chicago time**, Monday–Friday, including holidays. Weekend peak fields are intentionally missing. These are chosen clock-hour exposure windows, not measured CTA peak definitions. Accumulation windows follow the source's hourly timestamp labels; do not infer minute-level storm timing from them.

Lag-1 fields refer to the previous complete local calendar day. Prior-three-day totals exclude the current day. Consequently the first lag-1 row and first three prior-three-day rows are intentionally missing. The 2017–2019 subset is selected after these features are calculated, so January 1, 2017 retains its preceding exposures.

The monthly temperature anomaly subtracts the corresponding 2011–2025 monthly mean from a day's mean temperature. It is a broad seasonal adjustment, not a day-of-year normal. It includes future years relative to earlier rows: for prediction or chronological validation, recompute reference values using training data only. Same-day weather summaries also describe exposures that are not all known at the beginning of that day.

## Three-file handoff

Place these files at the following paths in your existing repository:

1. `notebooks/01_weather_exploration.ipynb`
2. `reports/weather/WEATHER_ANALYSIS.md`
3. `datasets/processed/weather/daily_weather_chicago.csv`

The notebook is self-contained: the complete transformation code is in its loading and aggregation
sections and independent validation is in its last cell. It requires no separate
helper scripts. Run all cells to recreate supporting tables, charts, the feature
dictionary, the hourly export, and the 2017–2019 subset from the existing raw CSV.
The daily CSV is already prepared for your later merge; rerunning is optional.

## Analysis outputs

- `notebooks/01_weather_exploration.ipynb`: populated, executed weather-only notebook with analysis, tables, charts, and handoff guidance.
- Complete transformation code and independent source/export validation are included inside the notebook. The original development validation also verified that transportation files were unchanged.
- `datasets/processed/weather/daily_weather_chicago.csv`: **51 columns**, one row per complete local date; primary merge file.
- `datasets/processed/weather/daily_weather_2017_2019.csv`: **1,095 rows**, matching the pre-COVID study period already used in the transportation notebook.
- `datasets/processed/weather/hourly_weather_chicago.csv`: source variables with explicit UTC/local timestamps, local date/hour, and weekday.
- `datasets/processed/weather/daily_coverage_audit.csv`: all 6,103 observed local dates, including both partial days.
- `datasets/processed/weather/monthly_weather.csv` and `annual_weather.csv`: summaries with completeness flags. Use only complete periods for direct comparisons of totals.
- `reports/weather/data_dictionary.csv`: every daily column, units, dtype, definition, and intentional null semantics.
- `reports/weather/`: quality audit, monthly reference, seasonal summaries, frequencies, thresholds, correlations, ranked extremes, severe-cold spells, and January 2019 extract.
- `figures/weather/`: 19 tutorial charts covering seasonality, distributions, exposure frequencies, correlations, and the 2019 cold spell.

## Merge handoff

Use the complete local-day CSV and join on `service_date`, parsed as a timezone-naive date in both tables. This corresponds to a Chicago **calendar day**. Confirm whether the CTA source assigns overnight boardings to a preceding operational service day; if so, its cutoff must be established before claiming exact temporal alignment. Daily CTA data cannot support claims about rush-hour ridership effects even though peak-hour weather features are available.

```python
weather = pd.read_csv(
    "datasets/processed/weather/daily_weather_chicago.csv",
    parse_dates=["service_date"],
)
# Future step only: use your existing transportation dataframe without changing raw data.
transport = df.copy()
transport["service_date"] = pd.to_datetime(transport["service_date"]).dt.normalize()
assert weather["service_date"].is_unique
assert transport["service_date"].is_unique  # Expected for this daily-total CTA source.
combined = transport.merge(
    weather, on="service_date", how="left", validate="one_to_one", indicator=True
)
print(combined["_merge"].value_counts())
# Inspect unmatched dates; never interpret missing weather as dry/benign weather.
```

This is an example for the later merge, not an operation performed in this analysis. CTA dates before January 9, 2010 or after September 22, 2026 will not match complete weather coverage. If working from a station/route-level table instead, its date key will repeat and the appropriate validation is `many_to_one`.

For the final analysis, begin with continuous temperature, precipitation and snowfall features plus a small number of interpretable event flags. Adjust for weekday, holidays, season, and long-term changes, and treat the COVID period separately or explicitly model its disruption. Consider nonlinearity, weather interactions, persistence, and serial correlation. Do not include every derived column in a single model: many are deterministic or highly correlated alternatives. Compare extreme days against calendar-matched controls, rather than attributing all raw ridership differences to weather. Service reliability requires actual service/delay information; daily boardings alone measure ridership.

## Limitations

One grid point cannot represent neighborhood-level storms or all CTA routes. Exact weather model/provenance and independent station accuracy remain unverified. No humidity, visibility, gusts, freezing rain, road/surface temperature, snow depth, or official warning data are available here. Freeze/thaw exposure is not observed icing, low temperature is not wind chill, and sustained wind is not gust speed. Partial 2010 and 2026 coverage must not be compared as full years. Weather-only analysis cannot rank which conditions affect transportation most strongly or establish causal effects.

## Reproduce and validate

Keep the original weather CSV at `datasets/open-meteo-41.86N87.65W179m.csv`.
Install the analysis dependencies in the environment used by your Jupyter kernel:

```bash
python -m pip install "pandas>=2.2,<3" "numpy>=1.26,<3" "matplotlib>=3.8,<4" "seaborn>=0.13,<0.14" scipy ipython
```

Open `notebooks/01_weather_exploration.ipynb` and run all cells. The notebook works
from the repository root or its `notebooks/` directory. It rebuilds weather outputs
from the raw source and runs independent validation in the last cell. All 25 code cells were successfully executed before this handoff. Runtime versions are recorded
in the generated `quality_report.json`. No transportation data is read or changed.

## Plotting lessons and expanded comparison analysis

The self-contained notebook now includes **19 chart lessons** and **25 executed code cells**. Every chart explains its analytical question, construction pattern, interpretation, and limitations. Matplotlib controls individual plot elements and layouts; pandas provides concise plotting of aggregated tables; Seaborn supplies statistical plots and facets. Rendered charts are embedded in the notebook, and rerunning saves PNGs under `figures/weather/tutorial/`.

| Lesson | Plot | Library |
|---:|---|---|
| 1 | A time-series chart with pandas | pandas plotting → Matplotlib axes |
| 2 | Histograms in a Matplotlib subplot grid | Matplotlib |
| 3 | Compare distributions fairly with Seaborn | Seaborn histplot |
| 4 | Monthly box plots and extreme observations | Seaborn boxplot |
| 5 | Violin plots for positive rainfall | Seaborn violinplot |
| 6 | Grouped bars for rain-versus-snow frequency | pandas bar plotting |
| 7 | Monthly amount bars with uncertainty intervals | Matplotlib + NumPy bootstrap |
| 8 | A month-by-year heatmap | Seaborn heatmap + pandas pivot |
| 9 | A density plot that preserves zero precipitation | Matplotlib hexbin |
| 10 | Small multiples: compare the same relationship by season | Seaborn relplot / FacetGrid |
| 11 | Exceedance curves for extreme-event frequency | Seaborn ecdfplot |
| 12 | A correlation heatmap and redundant predictors | Seaborn heatmap |
| 13 | A 100% stacked bar chart | pandas stacked bar plotting |
| 14 | Measure overlapping adverse-weather exposures | Seaborn heatmap + matrix operations |
| 15 | An annotated multi-panel event study | Matplotlib shared axes, fill_between, annotations |
| 16 | A calendar heatmap from daily observations | pandas pivot + Seaborn heatmap |
| 17 | An hourly seasonal heatmap | Seaborn heatmap on hourly data |
| 18 | Paired AM–PM comparison and a difference histogram | Seaborn scatterplot + pandas histogram |
| 19 | Consecutive-event durations | pandas run grouping + bar plotting |

The complete-year 2011–2025 reference has 5,479 days. Rain ≥1 mm occurs on 1,656
days (30.22%); new snowfall ≥0.1 cm occurs on 588 days (10.73%). There are 132
mixed days (2.41%), 1,524 rain-only days, 456 snow-only days, and 3,367 days meeting
neither threshold. Mixed means both occurred within a day, not necessarily together.
The neither category includes trace precipitation. Event flags and categories must
not be interchanged without considering their thresholds and denominators.

Monthly amount charts first aggregate within complete years; rain and snowfall
remain on separate panels in their original units. Error bars are 95% percentile
intervals from 4,000 year-block bootstrap resamples, not forecasts or the spread of
individual event amounts. This assumes exchangeability of years and does not model
climate trends. Additional tables contain the intervals and reference correlations.

The longest observed wet and snow runs within the reference are both 10 days.
Runs touching the reference edges are boundary-censored; snow-day runs measure new
snowfall rather than persistent snow cover. Persistence motivates examining lagged
exposures and accounting for serial dependence in later transit analysis.

The final notebook section gives a nonexecuted merge example for the existing CTA
CSV: parse `service_date`, preserve comma-separated boarding counts numerically,
left-join a selected weather subset with `validate='one_to_one'`, inspect unmatched
dates, and then reshape `bus` and `rail_boardings` into date–mode rows. Start with
2017–2019 to match the existing pre-COVID study. Confirm the service-day cutoff,
adjust for calendar patterns and long-term changes, and avoid interpreting raw
bus-versus-rail boarding differences as mode-specific weather sensitivity.
No transportation merge or transport-effect estimation was performed.
