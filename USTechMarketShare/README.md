# US Tech Market Share — Top / Middle / Bottom 30

Identifies the 30 US Technology companies with the highest, median, and lowest
**revenue-based market share**, using:

- Bloomberg terminal exports (tech-sector TTM + quarterly metrics)
- Nasdaq ticker universe (sector / industry / domicile classification)

## Data sources

| File | Source | Snapshot | Frequency |
| --- | --- | --- | --- |
| `data/raw/bloomberg_tech_sector_ttm.csv` | Bloomberg | YYYY-MM-DD | TTM |
| `data/raw/bloomberg_tech_sector_quarterly.csv` | Bloomberg | YYYY-MM-DD | Quarterly |
| `data/raw/nasdaq_ticker_universe.csv` | Nasdaq (GitHub: zyhe16/top-us-stock-tickers) | YYYY-MM-DD | Snapshot |

## Method

Market share = company `Revenue:Q` ÷ Σ `Revenue:Q` of the filtered US-tech set.
Two definitions are computed:

- **all_us_tech** — share of total US Technology revenue
- **own_industry** — share within the company's `industry` label from the Nasdaq file

## Notebooks

1. `notebooks/01_market_share_selection.ipynb` → produces `data/processed/q1_selected_30_*.csv`
2. `notebooks/02_weekly_tracking.ipynb` → (WIP)
3. `notebooks/03_pattern_analysis.ipynb` → (WIP)

## Reproduce

```bash
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute notebooks/01_market_share_selection.ipynb
