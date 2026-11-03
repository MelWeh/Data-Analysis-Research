# US Tech Market Share — Top / Middle / Bottom 30

Identifies the **30 US Technology companies** — the **top 10**, **middle 10**, and
**bottom 10** by **market share** — using the Nasdaq ticker universe for
sector/industry classification and the Bloomberg terminal exports for metrics.
The 30 companies are then tracked weekly (price, volume, implied share) and
their quarterly fundamentals (revenue, EBITDA, net income, EPS, P/E) are
recorded for pattern analysis.

---

## Questions answered

| # | Question | Notebook | Status |
| --- | --- | --- | --- |
| 1 | Top / middle / bottom 10 US tech companies by market share | `01_market_share_selection.ipynb` | Done |
| 2 | Weekly and quarterly tracking for the selected 30 | `02_weekly_tracking.ipynb` | Done |
| 3 | Patterns and trends discovered in the tracking data | `03_pattern_analysis.ipynb` | Done |

## Repository layout

    .
    ├── README.md
    ├── requirements.txt
    ├── data/
    │ ├── raw/
    │ ├── bloomberg_tech_sector_ttm.csv
    │ │ ├── bloomberg_tech_sector_quarterly.csv
    │ │ ├── nasdaq_ticker_universe.csv
    │ │ ├── Price_daily.csv
    │ │ ├── Shares_Outstanding_History.csv
    │ │ └── Quarterly_Fundamentals_BDH.csv
    │ └── processed/
    │ ├── q1_us_tech_with_shares.csv
    │ ├── q1_selected_30_all_tech.csv
    │ ├── weekly_tracking.csv
    │ ├── quarterly_tracking.csv
    │ └── quarterly_tracking_changes.csv
    ├── notebooks/
    │ ├── 01_market_share_selection.ipynb
    │ ├── 02_weekly_tracking.ipynb
    │ └── 03_pattern_analysis.ipynb
    └── outputs/
    ├── figures/
    │ ├── f1_concentration.png
    │ └── f2_share_trend.png
    └── tables/
    ├── f0_summary.csv
    ├── f1_concentration.csv
    ├── f2_share_trend.csv
    ├── f3_pe_summary.csv
    └── f5_exclusions.csv   

## Data sources

| File | Source | Frequency | Used by |
| --- | --- | --- | --- |
| `bloomberg_tech_sector_ttm.csv` | Bloomberg terminal | TTM | `01` (cross-check only) |
| `bloomberg_tech_sector_quarterly.csv` | Bloomberg terminal | Quarterly snapshot | `01` |
| `nasdaq_ticker_universe.csv` | Nasdaq (GitHub: `zyhe16/top-us-stock-tickers`) | Snapshot | `01` |
| `Price_daily.csv` | Bloomberg terminal | Daily | `02` |
| `Shares_Outstanding_History.csv` | Bloomberg terminal | Quarterly | `02` |
| `Quarterly_Fundamentals_BDH.csv` | Bloomberg terminal (BDH function) | Quarterly | `02` |

The three snapshot files were exported on **different dates** and disagree on
market cap (e.g. NVDA appears as $5.12T / $4.99T / $5.21T across the three).
Every analysis uses one snapshot consistently; document which snapshot each
result is based on.

## Method — Question 1 (market share)

Market share is not a field in any input file. It is computed as:

    share_i = metric_i / sum(metric_j)   over the filtered US-tech universe

Two candidate metrics were tested: revenue and market cap.

### Data-quality finding — why market-cap share, not revenue share

The Bloomberg revenue fields (`Revenue:Q` and `Revenue T12M`) contain
per-company figures **2.5× to 4.3× larger than the companies' reported TTM
revenue**, with the ratio **varying by company**:

| Ticker | `Revenue T12M` in export | Reported TTM (approx.) | Ratio |
| --- | --- | --- | --- |
| AAPL | $466.82B | ~$390B | 1.20× |
| MSFT | $346.45B | ~$270B | 1.28× |
| NVDA | $302.97B | ~$150B | 2.02× |
| MU | $133.19B | ~$30–35B | ~4× |
| DELL | $151.20B | ~$90–95B | ~1.6× |

`Revenue:Q` showed the same inconsistency (median ratio to true quarterly
revenue = 2.01×, spread 1.15×–6.03×). Neither field is a consistent
cross-sectional measure, so **shares computed from revenue would compare
different periods across companies**. The verification cell (Cell 4 in `01`)
is retained as an audit trail.

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

---

## Method — Question 2 (weekly & quarterly tracking)

`02_weekly_tracking.ipynb` reads `q1_selected_30_all_tech.csv` and produces
three files:

| Output | Contents |
| --- | --- |
| `weekly_tracking.csv` | Per (ticker, week): `price_avg`, `volume_avg`, `shares_out`, `market_cap_w`, `share_of_selected_30_pct_w`, `share_of_static_universe_pct_w`, `pe_w_from_eps`, `fund_asof_date` |
| `quarterly_tracking.csv` | Per (ticker, date): `revenue_q`, `ebitda_q`, `rev_less_ebitda_q`, `ni_q`, `eps_q` (panel, one row per BDH observation) |
| `quarterly_tracking_changes.csv` | Collapsed version of `quarterly_tracking.csv` — one row per report-date change |

Key design choices:

- **Fundamentals are a panel, not a snapshot.** Built from
  `Quarterly_Fundamentals_BDH.csv` and joined to weeks with
  `merge_asof(direction="backward")` on the effective date — no week sees a
  quarter not yet reported.
- **`pe_w_from_eps = price_avg / (eps_q × 4)`**, **NaN** when `eps_q <= 0`.
  The 4 assumes `IS_EPS` is a quarterly diluted figure.
- **Weekly volume is de-duplicated.** Days where volume equals the previous
  day's value for the same ticker are excluded from the weekly mean
  (flagged in `volume_duplicated`).
- **Shares are quarterly**, forward-filled to weeks. Intra-quarter buybacks
  and dilutions are invisible.
- **Two share denominators are used:**
  `share_of_selected_30_pct_w` (within the 30 selected companies) and
  `share_of_static_universe_pct_w` (within the full US tech snapshot). They
  are **not interchangeable**.

---

## Method — Question 3 (pattern analysis)

`03_pattern_analysis.ipynb` reads `02`'s outputs and produces **five findings**,
each with one table and (for findings 1–2) one chart:

| # | Finding | Evidence |
| --- | --- | --- |
| 1 | Three regimes, not a continuum | `f1_concentration.png`, `f1_concentration.csv` |
| 2 | Concentration is rising over the window | `f2_share_trend.png`, `f2_share_trend.csv` |
| 3 | Valuation tight at the top, wide in the middle | `f3_pe_summary.csv` |
| 4 | Volume tracks returns, not market share | printed Spearman ρ |
| 5 | 7 tickers excluded from share analyses | `f5_exclusions.csv` |

See `outputs/tables/f0_summary.csv` for all five headlines in one table.

---

## Data-quality exclusions

Seven of the 30 tickers are `NaN` in all share columns throughout the
tracking window:

| Ticker | Bucket | Reason | Effect |
| --- | --- | --- | --- |
| AVGO | top10 | `split_basis_suspect` | Shares vs prices on different split basis |
| KLAC | top10 | `split_basis_suspect` | Same |
| BTLN | bottom10 | `split_basis_suspect` | Same |
| YYAI | bottom10 | `split_basis_suspect` | Same |
| WOLF | middle10 | `stale_shares` | Shares file missing 2024-06 and 2024-09 |
| DAIC | bottom10 | `stale_shares` | Same |
| HCTI | bottom10 | `stale_shares` | Same |

These companies **remain** in the price and volume analyses — only their
weekly market cap and share columns are `NaN`. Disclosing them prevents a
reader from interpreting `NaN` as "zero share".

---

## Caveats

1. **`industry` labels are a coarse Nasdaq taxonomy.** "Semiconductors" lumps
   fabless, foundry, memory, and equipment companies together. `share_of_industry_pct`
   is a taxonomy-constrained proxy, not an antitrust market definition.
2. **Single-company industries produce 100% shares by construction.**
   `TRNR` shows 100% of "Consumer Electronics/Appliances" because it is the
   only US tech company in that bucket. Not a dominance signal.
3. **KLA's classification under "Electronic Components"** (not "Semiconductors")
   inflates its industry share to 41.3%. Reflects the taxonomy, not a data
   error.
4. **Bottom-10 shares are all < 0.0001%** — read as "effectively zero", not
   as a precise ranking. The ranking within the bottom ten is noise.
5. **Shares are relative to the filtered universe**, not "all US tech
   companies on earth". The denominator is 328 US-domiciled technology
   companies with positive market cap in the quarterly snapshot.
6. **Snapshot ambiguity**: the three raw snapshot files disagree on market cap
   and were exported on different dates.
7. **Market cap is a proxy for market share.** A company with high market cap
   but low revenue (or vice versa) is misweighted under this definition. The
   primary reason for the choice is the revenue-data inconsistency documented
   above, not a belief that market cap is a superior metric in general.
8. **The sample is 82 weeks.** The concentration trend (Finding 2) is
   descriptive, not a long-run forecast.
9. **`pe_w_from_eps` is `NaN` when `eps_q <= 0`.** Finding 3 is "among
   EPS-positive names". The middle bucket's true valuation dispersion is
   wider than shown.

## Reproduce

    pip install -r requirements.txt

    jupyter nbconvert --to notebook --execute notebooks/01_market_share_selection.ipynb
    jupyter nbconvert --to notebook --execute notebooks/02_weekly_tracking.ipynb
    jupyter nbconvert --to notebook --execute notebooks/03_pattern_analysis.ipynb
