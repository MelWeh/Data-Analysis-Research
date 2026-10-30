# US Tech Market Share — Top / Middle / Bottom 30

Identifies the 30 US Technology companies — the top 10, middle 10, and bottom 10
by market share — using the Nasdaq ticker universe for sector/industry
classification and the Bloomberg terminal exports for metrics. The 30 companies
are then tracked weekly (price, volume, implied share) and their quarterly
fundamentals (revenue, EBITDA, net income, EPS, P/E) are recorded for pattern
analysis.

## Repository layout

    .
    |-- README.md
    |-- requirements.txt
    |-- data/
    |   |-- raw/
    |   |   |-- bloomberg_tech_sector_ttm.csv
    |   |   |-- bloomberg_tech_sector_quarterly.csv
    |   |   `-- nasdaq_ticker_universe.csv
    |   `-- processed/
    |       |-- q1_us_tech_with_shares.csv
    |       `-- q1_selected_30_all_tech.csv
    |-- notebooks/
    |   |-- 01_market_share_selection.ipynb
    |   |-- 02_weekly_tracking.ipynb
    |   `-- 03_pattern_analysis.ipynb
    `-- outputs/
        |-- figures/
        `-- tables/

## Data sources

File                                 Source                                              Frequency                       Snapshot date
bloomberg_tech_sector_ttm.csv        Bloomberg terminal export                           Trailing twelve months          TODO
bloomberg_tech_sector_quarterly.csv  Bloomberg terminal export                           Per source (mixed — see below)  TODO
nasdaq_ticker_universe.csv           Nasdaq (GitHub: zyhe16/top-us-stock-tickers)        Snapshot                        TODO

The three files are different snapshots and disagree on market cap (e.g. NVDA =
5.12T / 4.99T / 5.21T across the three). State which snapshot each result is
based on in any report.

## Method — Question 1 (market share)

Market share is not a field in any input file. It is computed as:

    share_i = metric_i / sum(metric_j)   over the filtered US-tech universe

Two candidate metrics were tested: revenue and market cap.

### Data-quality finding (why market-cap share is used)

The Bloomberg export's revenue fields (Revenue:Q and Revenue T12M) contain
per-company figures 2.5x to 4.3x larger than the companies' reported TTM
revenue, with the ratio varying by company:

    Ticker   Revenue T12M in export   Reported TTM (approx.)   Ratio
    AAPL     $466.82B                 ~$390B                   1.20x
    MSFT     $346.45B                 ~$270B                   1.28x
    NVDA     $302.97B                 ~$150B                   2.02x
    MU       $133.19B                 ~$30-35B                 ~4x
    DELL     $151.20B                 ~$90-95B                 ~1.6x

Revenue:Q showed the same inconsistency (median ratio to true quarterly revenue
= 2.01x, spread 1.15x to 6.03x). Neither revenue field is a consistent
cross-sectional measure.

The verification cell (Cell 4) is retained in the notebook for auditability. It
prints the ratio table and a verdict line so a reader can see the evidence.

### Market-cap share (primary metric)

    share_of_all_tech_pct  = Market Cap / sum(Market Cap US tech) * 100
    share_of_industry_pct  = Market Cap / sum(Market Cap industry) * 100

Market cap is a clean, single-period, consistently reported figure across all
1,000+ US tech companies in the universe. The top-10 membership is stable across
revenue-based, TTM-based, and market-cap-based measures; only the ordering and
the exact percentages change.

### Universe filters

    - is_us_domiciled == True from the Nasdaq file (drops TSM, ASML, SAP, ARM, etc.)
    - sector == "Technology" from the Nasdaq file
    - Market Cap:Q > 0
    - Drop SIFY (corrupt row in the Bloomberg export) and GOOG (dedupe vs GOOGL)
    - Drop preferreds / warrants / units (tickers containing ^ or ending W)

### Bucket definition

Sorted descending by share_of_all_tech_pct:

    top 10     = ranks 1-10
    middle 10  = 10 rows centered on the median rank
    bottom 10  = ranks N-9 to N

## Caveats

1. industry labels are a coarse Nasdaq taxonomy. "Semiconductors" lumps fabless,
   foundry, memory, and equipment companies together; "Computer Manufacturing"
   lumps Apple and Dell with HP. share_of_industry_pct is a taxonomy-constrained
   proxy, not an antitrust market definition.
2. Single-company industries produce 100% shares by construction. TRNR shows
   100% of "Consumer Electronics/Appliances" because it is the only US-tech
   company in that bucket. Not a dominance signal.
3. KLA's classification under "Electronic Components" (not "Semiconductors")
   inflates its industry share to 41.3%. Reflects the taxonomy, not a data
   error.
4. Bottom-10 shares are all <0.0001% — read as "effectively zero", not as a
   precise ranking. The ranking within the bottom ten is noise.
5. Shares are relative to the filtered universe, not "all US tech companies on
   earth". The denominator is ~1,000 US-domiciled technology companies with
   positive market cap in the Bloomberg export.
6. Snapshot ambiguity: the three raw files disagree on market cap and were
   exported on different dates. Document which snapshot each result is based on.

## Reproduce

    pip install -r requirements.txt
    jupyter nbconvert --to notebook --execute notebooks/01_market_share_selection.ipynb

The notebook writes:

    data/processed/q1_us_tech_with_shares.csv    (full universe)
    data/processed/q1_selected_30_all_tech.csv   (30 selected companies)

## Notebooks

    01_market_share_selection.ipynb   Top/middle/bottom 30 US tech by market share   Done
    02_weekly_tracking.ipynb          Weekly price, volume, share + quarterly fund.  WIP
    03_pattern_analysis.ipynb         Trends / patterns discovered in Q2 data         WIP
    