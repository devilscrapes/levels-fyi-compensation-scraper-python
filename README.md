<p align="center"><a href="https://apify.com/DevilScrapes/levels-fyi-compensation-scraper"><img src="assets/banner.svg" alt="Levels.fyi Compensation Scraper — Python example" width="100%"></a></p>

# Levels.fyi Compensation Scraper in Python: per-level tech pay as a pandas-ready table

<p align="center">
<a href="https://apify.com/DevilScrapes/levels-fyi-compensation-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/levels-fyi-compensation-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/levels-fyi-compensation-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository is a short, tested Python client for the **[Levels.fyi Compensation Scraper](https://apify.com/DevilScrapes/levels-fyi-compensation-scraper)**, an Apify Actor published by Devil Scrapes. You name companies by their Levels.fyi slug (`google`, `meta`, `stripe`) and, optionally, a job family such as `software-engineer`; the Actor reads the matching Levels.fyi pages in the cloud and hands back tidy rows of reported compensation. Each row says which level the offer was at, where it was based, how many years of experience the person had, and how the package splits into salary, stock and bonus. Output lands in `results.json` and `results.csv`, ready for a notebook or a pay-band spreadsheet.

> **Actor:** [Levels.fyi Compensation Scraper](https://apify.com/DevilScrapes/levels-fyi-compensation-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [levels-fyi-compensation-scraper-javascript](https://github.com/devilscrapes/levels-fyi-compensation-scraper-javascript)

## 🔥 What you get

- **Pay components stay apart**: in the default mode each row is a single self-reported offer, so salary, annual equity and annual bonus arrive as three numbers next to the total rather than one blended figure.
- **Two page types, one row shape**: a company's salaries page and its per-job-family pages both map onto the same 21 columns, and anything a page type does not publish is left null instead of estimated.
- **Every company and job family combined**: list three companies and two job families and the Actor visits all six pages in turn.
- **A run-wide row budget**: `maxResults` counts rows across all pages, and the Actor stops opening new pages once the budget is spent.
- **Sample size travels with the data**: each row repeats its level bucket's submission counts, so you can see how many reports sit behind the handful of rows you received.
- **Bad slugs do not sink a batch**: a company or job family that yields nothing is logged and skipped, and the run only fails when every page comes back empty.
- **Transient errors are retried** with growing pauses before a page is given up on.

## 🎯 Use cases

### Draw a pay ladder for one employer

Fetch a single company and job family, group by `level` and take medians. A few lines of pandas show how salary, equity and bonus climb from one level to the next.

### Measure how equity-heavy a company pays

Divide `stock_annual_value` by `total_compensation` per row and compare the ratio across companies. Two employers with similar totals can carry very different equity risk, and this column makes that visible without manual reading.

### Keep a dated archive for trend analysis

Run `python main.py` weekly from cron, write each CSV into a folder named by date and concatenate them later. With `offer_date` on every individual row you can bucket offers by quarter and watch a level drift over time.

### Feed a comp model or BI dashboard

Append each CSV to a warehouse table and join on company and level; column names stay fixed between runs, so dashboards built once keep working.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/levels-fyi-compensation-scraper-python.git
cd levels-fyi-compensation-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Levels.fyi Compensation Scraper](https://apify.com/DevilScrapes/levels-fyi-compensation-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Strip away argument parsing and CSV writing and this is what remains. `call()` launches the Actor and waits for it; `iterate_items()` walks the resulting dataset one compensation row at a time.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/levels-fyi-compensation-scraper").call(run_input={'mode': 'company_job_family', 'companies': ['google'], 'jobFamilies': ['software-engineer'], 'maxResults': 5, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` reads `input.json` and forwards it to the Actor as is. The bundled input asks for Google's software engineer page and caps the run at 5 rows, enough to inspect the columns for a few cents. Slugs are the path segments of a Levels.fyi URL: `levels.fyi/companies/google/salaries/software-engineer` gives `google` and `software-engineer`. Switch `mode` to `company_overview` and drop `jobFamilies` to get one summary row per title instead.

```json
{
  "mode": "company_job_family",
  "companies": [
    "google"
  ],
  "jobFamilies": [
    "software-engineer"
  ],
  "maxResults": 5,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `mode` | string | no | `"company_job_family"` | `company_job_family`, `company_overview` |
| `companies` | array | yes | `["google"]` | — |
| `jobFamilies` | array | no | `[]` | — |
| `maxResults` | integer | no | `200` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

A row's `granularity` tells you which kind it is. `individual_submission` rows come from job-family pages and carry the full split; `aggregate_title` rows come from the overview page, where `total_compensation` is a median and the salary, level, location and offer date columns are empty. Amounts are plain numbers with a `currency_code` beside them, so they load straight into numeric columns.

| Field | Type | Always present |
|---|---|---|
| `granularity` | string | yes |
| `company` | string | yes |
| `company_slug` | string | yes |
| `job_family` | string or null | no |
| `job_family_slug` | string or null | no |
| `title` | string or null | no |
| `level` | string or null | no |
| `track` | string or null | no |
| `location` | string or null | no |
| `years_of_experience` | number or null | no |
| `years_at_company` | number or null | no |
| `base_salary` | number or null | no |
| `stock_annual_value` | number or null | no |
| `bonus_annual_value` | number or null | no |
| `total_compensation` | number or null | no |
| `currency_code` | string | yes |
| `offer_date` | string or null | no |
| `level_submission_count` | integer or null | no |
| `level_submission_count_last_12_months` | integer or null | no |
| `source_url` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result row | $0.0018 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/levels-fyi-compensation-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

The snippet below turns individual reports into a per-level summary with pandas (`pip install pandas`). It keeps only submission rows, takes medians for every pay component, adds the median equity share and writes `pay_ladder.csv`. The `sample_rows` column next to `bucket_total` reminds you how small the visible sample is.

```python
import json

import pandas as pd

rows = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
subs = rows[rows["granularity"] == "individual_submission"].copy()
money = ["base_salary", "stock_annual_value", "bonus_annual_value", "total_compensation"]
subs[money] = subs[money].apply(pd.to_numeric, errors="coerce")
subs["stock_share"] = subs["stock_annual_value"] / subs["total_compensation"]

ladder = (
    subs.groupby(["company", "job_family", "level"])
    .agg(
        sample_rows=("total_compensation", "size"),
        bucket_total=("level_submission_count", "max"),
        median_base=("base_salary", "median"),
        median_stock=("stock_annual_value", "median"),
        median_bonus=("bonus_annual_value", "median"),
        median_total=("total_compensation", "median"),
        stock_share=("stock_share", "median"),
    )
    .sort_values("median_total")
    .round(2)
)
ladder.to_csv("pay_ladder.csv")
print(ladder.to_string())
```

## ❓ FAQ

### Do I need a Levels.fyi login or API key?

No. Your Apify token covers everything here. Levels.fyi shows these salary pages to anonymous visitors, so nothing about your own Levels.fyi profile is touched.

### Why do I get fewer rows than a level's submission count?

A job-family page only embeds a limited preview of reports for each level, not every report on file. The two count columns describe the whole bucket, so compare them with your row count before treating a median as representative.

### How do I find the right slugs?

Open the company on Levels.fyi and copy the path segments from the address bar. The Actor lowercases your entries and turns spaces into hyphens, but a display name like `Google Inc` will not match a real page.

### Does it cover salaries outside the US?

This version reads the default, US-locale pages. Rows carry `currency_code`, yet non-US locales and other currencies have not been confirmed, so plan your analysis around USD for now.

### Can I call it from a notebook instead of the command line?

Yes. Import `main` and run `main.main(["input.json", "--out", "data"])` inside a cell, then read `data/results.csv` with pandas. The function returns 0 on success, which also suits Airflow or a CI job.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/levels-fyi-compensation-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Levels.fyi Compensation Scraper on Apify Store](https://apify.com/DevilScrapes/levels-fyi-compensation-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/levels-fyi-compensation-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [BuiltIn Jobs Scraper](https://apify.com/DevilScrapes/builtin-jobs-scraper) — job listings with posted salary bands to set against reported pay.
- [Dice Tech Jobs Scraper](https://apify.com/DevilScrapes/dice-tech-jobs-scraper) — open tech roles on Dice, searchable by skill and location.
- [Wellfound Startup Jobs Scraper](https://apify.com/DevilScrapes/wellfound-jobs-scraper) — startup openings with their advertised compensation and funding stage.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/levels-fyi-compensation-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Levels.fyi.
