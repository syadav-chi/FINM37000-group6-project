# Pricing the Fed with SOFR Futures

SOFR futures recover the market-implied path of the federal funds rate, sharpest at the next meeting and usually already in the book when the statement is read.

On monthly SR1 and SR3 settlements the full model's next-meeting error averages 6.15 bp (bias -0.09 bp; 44 dates, 30 meetings). A simplified curve that ignores meeting dates averages 14.34 bp on those dates (bias -7.06 bp, RMSE 21.76 bp, against the full model's 8.20 bp). On 1,166 daily SR1 dates mean absolute error is 11.34 bp and 13.33 bp. Changes, 572 dates: 9.78 bp versus 25.05 bp. Holds, 594 dates: 12.83 bp versus 2.05 bp. Daily RMSE is 20.77 bp and 20.57 bp. In each pair the full model is first.

Error grows with the horizon. Monthly mean absolute error goes from 6.15 bp one meeting ahead to 54.15 bp at six, and bias from -0.09 bp to -43.36 bp. Next-meeting direction is right on 43 of 44 dates, using a 12.5 bp hike, cut, or hold line.

Of 33 statements from 26 January 2022 through 28 January 2026, 25 moved the front meeting by less than 2 bp from 13:55 to 16:00 Eastern. The largest identified moves were 18 September 2024 (-11.12 bp), 29 October 2025 (+4.49 bp), and 15 June 2022 (+3.77 bp). On 18 September the level went from -37.19 bp at 13:55 to -48.31 bp at 16:00. Databento marks that day degraded (11 contracts, pricing RMSE 0.33 bp).

## The question

How much of the Fed's path is already in SOFR futures, and does a calendar-aware model beat a smooth curve?

## Data

The futures are Databento CME Globex `GLBX.MDP3`: SR1 and SR3 settlements from the statistics schema, plus one-minute `ohlcv-1m` bars from 13:00 to 16:00 Eastern on announcement days. Overnight SOFR is the New York Fed fixing. The target range is FRED `DFEDTARL` and `DFEDTARU`. Meetings are the Federal Reserve's scheduled calendar. The settlement window is January 2022 through September 2026. The key is read from `~/.databento_api_key`, then `DATABENTO_API_KEY`. See `DATA.md` and `FED_RATE_DATA.md`.

## Method

An SR1 settles to average SOFR in a calendar month, and an SR3 to compounded SOFR between IMM dates. The price is 100 minus that rate. We fix the day's target-range midpoint, set the SOFR-minus-policy basis from up to 20 published fixings, and choose meeting moves by regularized least squares so model prices match the strip. Error is the implied cumulative policy change minus the realized change, in basis points.

The simplified curve bootstraps SR1 with no meeting step. Monthly results use `sofr_pricing.fit_many` on the first futures date of each month (basis on 56 dates, fit on 44, six horizons). The daily table is a separate next-meeting SR1 bootstrap. See `docs/METHODOLOGY.md`, `FORECAST_EVAL.md`, `CURVE_COMPARE.md`, `final_visualizations/VISUALIZATIONS.md`, and `INTRADAY.md`.

## Figures

On 1 April 2022, the first monthly date that fitted, the next meeting was +45.13 bp implied against +50 bp realized. Six meetings out it was +214.82 bp against +400 bp.

![Path on 1 April 2022](docs/figures/01_fed_paths.png)

![Error by meetings ahead](docs/figures/02_forecast_errors_by_horizon.png)

The third figure is the monthly comparison, 6.15 bp against 14.34 bp. Its right panel splits those 44 dates into changes and holds. The wide hold gap is the daily table, a separate sample.

![Monthly full model and simplified curve](docs/figures/03_model_comparison.png)

| Horizon | N | Meetings | Bias (bp) | MAE (bp) | RMSE (bp) |
| --- | ---: | ---: | ---: | ---: | ---: |
| 1 | 44 | 30 | -0.09 | 6.15 | 8.20 |
| 2 | 44 | 30 | -5.57 | 15.12 | 19.75 |
| 3 | 44 | 30 | -14.27 | 28.24 | 36.09 |
| 4 | 44 | 30 | -25.27 | 38.51 | 50.59 |
| 5 | 44 | 30 | -34.74 | 46.68 | 63.31 |
| 6 | 44 | 30 | -43.36 | 54.15 | 73.11 |

| Next meeting | N | Full-model MAE (bp) | Simplified-curve MAE (bp) |
| --- | ---: | ---: | ---: |
| Target changed | 572 | 9.78 | 25.05 |
| Target held | 594 | 12.83 | 2.05 |

Daily bias is +0.13 bp for the full model and -6.58 bp for the simplified curve. Mean simplified-minus-full error is -6.71 bp.

On 15 June 2022 the front meeting was already +70.37 bp at 13:55, before a 75 bp hike. On 29 October 2025 horizons 2 and 3 ended +9.77 and +10.44 bp versus 13:55.

![Three largest announcement afternoons](reports/intraday/real/repricing_paths.png)

## Limits

Futures prices include risk premia, so an implied move averages over outcomes. A wrong basis falls on the nearest meeting. The fixing calendar is weekdays only. Four afternoons have 7 meetings and rank 6, so the path chart omits them. 17 September 2025 is also degraded and moved +1.45 bp. Five 2026 statements fall after 16 September 2026, the last published meeting.

## Who built it

Maisha Miah built the Databento settlement pipeline (issue 1). Ivy Chan built the FOMC calendar, realized rates, and futures alignment (issues 2 and 14) and the final figures (issue 7). Theo Li built the SR1 and SR3 pricing framework (issue 3) and the Fed-path estimator (issue 4). Sankalp Yadav built the forecast scorecard (issue 5), the full-versus-simplified comparison (issue 6), and the intraday announcement study (issue 8). The original plan is in `PLANNING.md`.

## What is left

Score the daily bootstrap past the next meeting, put its step on the pricing code's next business day, add a holiday calendar, and extend the meeting list past 16 September 2026.

## How to run

Use Python 3.12 or newer. Run the commands from the repository root, after installing the dependencies.

```bash
python -m pip install -r requirements.txt
```

The Databento key is a single line in `~/.databento_api_key`, mode 600. `sofr_data/config.py` reads that file first and then the `DATABENTO_API_KEY` environment variable. Do not commit the key. Raw and processed market files under `data/` are gitignored.

Price every Databento pull before downloading, and stop if the quote is above 5 dollars. Cached chunks are not requested again. On this sample, with the caches already present, both quotes printed $0.0000.

```bash
python scripts/fetch_sofr_futures.py --dry-run
python scripts/fetch_sofr_futures.py --yes
python -m scripts.fetch_rate_data
python scripts/fetch_intraday_sofr.py --dry-run
python scripts/fetch_intraday_sofr.py
```

`fetch_sofr_futures.py` writes `data/processed/sofr_futures_settlements.parquet`. `fetch_rate_data` writes `data/processed/sofr_daily.csv`, `data/processed/effr_daily.csv`, and `data/processed/fomc_meetings.csv`. `fetch_intraday_sofr.py` writes `data/processed/sofr_intraday_ohlcv_1m.parquet`.

The monthly horizon table, the monthly comparison, and the three figures in `docs/figures/` come from the commands below. `figures/` is gitignored, so the plot command writes `figures/historical/`. The README embeds the copies committed at `docs/figures/01_fed_paths.png`, `docs/figures/02_forecast_errors_by_horizon.png`, and `docs/figures/03_model_comparison.png`.

```bash
python -m scripts.prepare_historical_forecasts \
  --futures data/processed/sofr_futures_settlements.parquet \
  --sofr data/processed/sofr_daily.csv \
  --decisions data/processed/fomc_meetings.csv \
  --output-dir data/processed/visualizations
python -m scripts.prepare_visualization_inputs \
  --implied data/processed/visualizations/implied_forecasts.csv \
  --decisions data/processed/fomc_meetings.csv \
  --output-dir data/processed/visualizations
python -m scripts.prepare_historical_comparison
python -m scripts.generate_final_figures \
  --errors data/processed/visualizations/forecast_errors.csv \
  --decisions data/processed/fomc_meetings.csv \
  --comparison data/processed/visualizations/model_comparison.csv \
  --obs-date 2022-04-01 \
  --output-dir figures/historical
```

The horizon table is `figures/historical/forecast_accuracy_by_horizon_formatted.csv`. The monthly full-versus-simplified table is `figures/historical/model_comparison_summary_formatted.csv`.

The daily change-and-hold table is the printout of:

```bash
python -m scripts.run_curve_compare_real
```

The announcement figure is already at the path the study writes. Regenerate it with:

```bash
python scripts/run_intraday_demo.py --real --out reports/intraday/real
```

That rewrites `reports/intraday/real/repricing_paths.png` and the tables beside it, including `reports/intraday/real/snapshot_changes.csv`.

These four commands need no Databento key and no download. The synthetic intraday charts are a planted jump. Their titles say so, and they are not the historical numbers above.

```bash
python -m scripts.run_forecast_eval_demo
python -m scripts.run_curve_compare_demo
python scripts/run_intraday_demo.py --out reports/intraday
python -m pytest -q
```
