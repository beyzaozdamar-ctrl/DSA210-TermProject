# DSA210-TermProject
from pathlib import Path

import matplotlib.pyplot as plt
import pandas as pd
import seaborn as sns
from scipy import stats
import statsmodels.formula.api as smf


BASE_DIR = Path(__file__).resolve().parent
PRODUCTION_FILES = [
    BASE_DIR / "CSV ler kopyası" / "data.csv",
    BASE_DIR / "CSV ler kopyası" / "Bant Ana Veri.csv",
]
WEATHER_FILE = BASE_DIR / "Kayseri 2024-10-01 to 2026-02-03.csv"
OUTPUT_DIR = BASE_DIR / "analysis_outputs"


def load_production_data() -> pd.DataFrame:
    frames = []

    for file_path in PRODUCTION_FILES:
        df = pd.read_csv(file_path)
        df.columns = [column.strip() for column in df.columns]
        df = df.rename(
            columns={
                "Çevrim Süresi": "Çevrim Süresi (sn)",
                "Plansız Duruşlar(Dk)": "Plansız Duruşlar (Dk)",
                "Saatlik Planlanan                    Üretim Adeti": "Saatlik Planlanan Üretim Adeti",
            }
        )
        df["source_file"] = file_path.name
        frames.append(df)

    hourly = pd.concat(frames, ignore_index=True, sort=False)

    # Keep only real hourly observations. The raw exports also contain blank footer rows.
    hourly = hourly[hourly["Tarih"].notna() & hourly["Saat"].notna()].copy()
    hourly["Tarih"] = pd.to_datetime(hourly["Tarih"], errors="coerce")

    numeric_columns = [
        "Üretim Adeti",
        "Arıza Adeti",
        "Plansız Duruşlar (Dk)",
        "Saatlik Planlanan Üretim Adeti",
        "OLE",
        "Kişi Sayısı",
    ]
    for column in numeric_columns:
        if column in hourly.columns:
            hourly[column] = pd.to_numeric(hourly[column], errors="coerce")

    return hourly


def build_daily_dataset(hourly: pd.DataFrame) -> pd.DataFrame:
    daily = (
        hourly.groupby(hourly["Tarih"].dt.date)
        .agg(
            avg_ole=("OLE", "mean"),
            median_ole=("OLE", "median"),
            total_production=("Üretim Adeti", "sum"),
            total_failures=("Arıza Adeti", "sum"),
            total_downtime=("Plansız Duruşlar (Dk)", "sum"),
            avg_workers=("Kişi Sayısı", "mean"),
            planned_production=("Saatlik Planlanan Üretim Adeti", "sum"),
            hourly_obs=("OLE", "count"),
        )
        .reset_index()
        .rename(columns={"Tarih": "date"})
    )

    daily["failure_rate"] = daily["total_failures"] / daily["total_production"]
    daily["downtime_per_hour"] = daily["total_downtime"] / daily["hourly_obs"]

    weather = pd.read_csv(WEATHER_FILE)
    weather["date"] = pd.to_datetime(weather["datetime"]).dt.date
    weather["temp_c"] = (weather["temp"] - 32) * 5 / 9

    merged = daily.merge(weather, on="date", how="inner")
    merged["is_rainy"] = merged["precip"].fillna(0) > 0
    merged["is_overcast"] = merged["cloudcover"].fillna(0) >= 75
    merged["season"] = pd.Categorical(
        pd.to_datetime(merged["date"]).map(
            lambda value: {
                12: "Winter",
                1: "Winter",
                2: "Winter",
                3: "Spring",
                4: "Spring",
                5: "Spring",
                6: "Summer",
                7: "Summer",
                8: "Summer",
                9: "Autumn",
                10: "Autumn",
                11: "Autumn",
            }[value.month]
        ),
        categories=["Winter", "Spring", "Summer", "Autumn"],
        ordered=True,
    )
    return merged


def compute_hypothesis_tests(daily: pd.DataFrame) -> pd.DataFrame:
    results = []

    group_tests = [
        ("H1", "Rainy vs non-rainy", "is_rainy", "avg_ole"),
        ("H2", "Rainy vs non-rainy", "is_rainy", "failure_rate"),
        ("H3", "Overcast vs clearer days", "is_overcast", "avg_ole"),
        ("H4", "Overcast vs clearer days", "is_overcast", "downtime_per_hour"),
    ]

    for hypothesis_id, comparison, group_column, metric in group_tests:
        sample_a = daily.loc[daily[group_column], metric].dropna()
        sample_b = daily.loc[~daily[group_column], metric].dropna()
        _, ttest_p = stats.ttest_ind(sample_a, sample_b, equal_var=False)
        _, mw_p = stats.mannwhitneyu(sample_a, sample_b, alternative="two-sided")
        results.append(
            {
                "hypothesis_id": hypothesis_id,
                "test_type": comparison,
                "metric": metric,
                "group_a_n": len(sample_a),
                "group_b_n": len(sample_b),
                "group_a_mean": sample_a.mean(),
                "group_b_mean": sample_b.mean(),
                "welch_t_pvalue": ttest_p,
                "mann_whitney_pvalue": mw_p,
            }
        )

    for metric in ["avg_ole", "failure_rate", "downtime_per_hour"]:
        groups = [group[metric].dropna().values for _, group in daily.groupby("season", observed=False)]
        _, anova_p = stats.f_oneway(*groups)
        _, kruskal_p = stats.kruskal(*groups)
        season_means = daily.groupby("season", observed=False)[metric].mean().to_dict()
        results.append(
            {
                "hypothesis_id": f"Seasonal-{metric}",
                "test_type": "Seasonal differences",
                "metric": metric,
                "group_a_n": None,
                "group_b_n": None,
                "group_a_mean": None,
                "group_b_mean": None,
                "welch_t_pvalue": anova_p,
                "mann_whitney_pvalue": kruskal_p,
                "season_means": season_means,
            }
        )

    return pd.DataFrame(results)


def compute_correlations(daily: pd.DataFrame) -> pd.DataFrame:
    rows = []
    weather_columns = ["temp_c", "humidity", "precip", "cloudcover", "windspeed", "solarradiation"]
    target_columns = ["avg_ole", "failure_rate", "downtime_per_hour"]

    for weather_column in weather_columns:
        for target_column in target_columns:
            sample = daily[[weather_column, target_column]].dropna()
            rho, pvalue = stats.spearmanr(sample[weather_column], sample[target_column])
            rows.append(
                {
                    "weather_variable": weather_column,
                    "target_variable": target_column,
                    "n": len(sample),
                    "spearman_rho": rho,
                    "pvalue": pvalue,
                }
            )

    return pd.DataFrame(rows).sort_values("spearman_rho", key=lambda s: s.abs(), ascending=False)


def run_regression(daily: pd.DataFrame) -> pd.DataFrame:
    ole_model = smf.ols(
        "avg_ole ~ temp_c + precip + cloudcover + humidity + avg_workers + C(season)",
        data=daily,
    ).fit(cov_type="HC3")

    failure_model = smf.ols(
        "failure_rate ~ temp_c + precip + cloudcover + humidity + avg_workers + C(season)",
        data=daily.dropna(subset=["failure_rate"]),
    ).fit(cov_type="HC3")

    records = []
    for model_name, model in [("avg_ole_model", ole_model), ("failure_rate_model", failure_model)]:
        for parameter, coefficient in model.params.items():
            records.append(
                {
                    "model": model_name,
                    "term": parameter,
                    "coef": coefficient,
                    "pvalue": model.pvalues[parameter],
                    "r_squared": model.rsquared,
                }
            )

    return pd.DataFrame(records)


def create_plots(daily: pd.DataFrame) -> None:
    sns.set_theme(style="whitegrid")

    plt.figure(figsize=(12, 5))
    sns.lineplot(data=daily, x=pd.to_datetime(daily["date"]), y="avg_ole", linewidth=1.8)
    plt.title("Daily Average OLE Over Time")
    plt.xlabel("Date")
    plt.ylabel("Average OLE")
    plt.tight_layout()
    plt.savefig(OUTPUT_DIR / "ole_time_series.png", dpi=180)
    plt.close()

    plt.figure(figsize=(8, 5))
    sns.boxplot(data=daily, x="season", y="avg_ole", hue="season", palette="Set2", legend=False)
    plt.title("Average OLE by Season")
    plt.xlabel("Season")
    plt.ylabel("Average OLE")
    plt.tight_layout()
    plt.savefig(OUTPUT_DIR / "ole_by_season.png", dpi=180)
    plt.close()

    plt.figure(figsize=(8, 5))
    sns.boxplot(
        data=daily,
        x="is_overcast",
        y="avg_ole",
        hue="is_overcast",
        palette="Set1",
        legend=False,
    )
    plt.title("Average OLE on Overcast vs Clearer Days")
    plt.xlabel("Overcast day")
    plt.ylabel("Average OLE")
    plt.xticks([0, 1], ["No", "Yes"])
    plt.tight_layout()
    plt.savefig(OUTPUT_DIR / "ole_overcast_comparison.png", dpi=180)
    plt.close()

    plt.figure(figsize=(8, 5))
    sns.scatterplot(data=daily, x="humidity", y="avg_ole", hue="season", alpha=0.75)
    sns.regplot(data=daily, x="humidity", y="avg_ole", scatter=False, color="black")
    plt.title("Humidity vs Average OLE")
    plt.xlabel("Humidity")
    plt.ylabel("Average OLE")
    plt.tight_layout()
    plt.savefig(OUTPUT_DIR / "humidity_vs_ole.png", dpi=180)
    plt.close()


def main() -> None:
    OUTPUT_DIR.mkdir(exist_ok=True)

    hourly = load_production_data()
    daily = build_daily_dataset(hourly)

    daily.to_csv(OUTPUT_DIR / "daily_merged_dataset.csv", index=False)
    compute_hypothesis_tests(daily).to_csv(OUTPUT_DIR / "hypothesis_test_results.csv", index=False)
    compute_correlations(daily).to_csv(OUTPUT_DIR / "correlation_results.csv", index=False)
    run_regression(daily).to_csv(OUTPUT_DIR / "regression_results.csv", index=False)
    create_plots(daily)

    summary = {
        "hourly_rows_after_cleaning": len(hourly),
        "daily_rows_after_merge": len(daily),
        "date_min": str(min(daily["date"])),
        "date_max": str(max(daily["date"])),
        "avg_ole_mean": round(daily["avg_ole"].mean(), 4),
        "failure_rate_mean": round(daily["failure_rate"].mean(), 4),
        "downtime_per_hour_mean": round(daily["downtime_per_hour"].mean(), 4),
    }
    pd.Series(summary).to_csv(OUTPUT_DIR / "analysis_summary.csv", header=False)


if __name__ == "__main__":
    main()
