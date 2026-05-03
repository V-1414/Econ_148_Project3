# Econ 148 Project 3 — Intergenerational Mobility Across U.S. Counties

Track A (Open Data Project). The economic question: *what county-level features predict upward mobility for children born to low-income parents, and does a non-linear model uncover interactions the standard linear specification misses?*

The analysis pairs an OLS baseline (the variables Chetty et al. emphasise) with a random-forest comparison and SHAP-based interpretation. The interesting finding is in the **interactions** the random forest surfaces, not the headline R² gap. See [CLAUDE.md](CLAUDE.md) for the full set of locked-in modelling decisions.

## Repository layout

```
Econ_148_Project3/
├── Analysis/
│   ├── draft1.ipynb          # 9-predictor baseline (preserved for reference)
│   ├── draft2.ipynb          # active notebook: 11-predictor specification
│   └── data/                 # gitignored; populate per "Data" section below
├── CLAUDE.md                 # working notes / decision log
├── appendix.md               # AI-tool disclosure (course requirement)
├── Project 3 Checkpoint 1.pdf
└── README.md
```

The folder name is **lowercase `data/`** because that's what the notebook reads (`pyreadstat.read_dta("data/...")`). On macOS this is case-insensitive so `Data/` happens to work locally; on Linux it does not. Use lowercase when populating a fresh clone.

The active notebook is **[Analysis/draft2.ipynb](Analysis/draft2.ipynb)**. Update this README whenever the active notebook changes.

## Reproducibility

The notebook is designed to produce identical numbers and figures across runs. Three things make that work:

1. **Single seed**. All stochastic operations (random forest, K-fold splits, NumPy and Python `random`) read from the `SEED` constant defined near the top of the notebook (currently `42`).
2. **Complete-case analytic sample**. Rows with any missing predictor are dropped before any model is fit; sample size is always reported as `len(analytic)`.
3. **Relative paths only**. The notebook reads from `data/…` relative to its own directory. There are no machine-specific paths, so a fresh clone with the data placed under `Analysis/data/` runs without edits.

### How to reproduce

1. **Clone the repo.**
   ```bash
   git clone https://github.com/V-1414/Econ_148_Project3.git
   cd Econ_148_Project3
   ```

2. **Install dependencies.** Python 3.10+. The notebook auto-installs `pyreadstat` and `shap` on first run via `pip`; if you prefer to install explicitly:
   ```bash
   pip install numpy pandas matplotlib seaborn pyreadstat statsmodels scikit-learn shap
   ```

3. **Download the data files** (see table below) and place them as shown. None of the files ship with the repo — they are gitignored because two of them exceed GitHub's 100MB limit, and the rest are freely re-downloadable from the original sources.

4. **Run the notebook end-to-end.**
   ```bash
   jupyter notebook Analysis/draft2.ipynb
   ```
   Run cells top-to-bottom. With the seed set, the OLS coefficients, RF CV R², SHAP rankings, and PDP curves should match exactly.

### Data sources and file placement

All files go under `Analysis/data/`.

| File | Source |
|---|---|
| `county_outcomes_dta.dta` | Opportunity Atlas — https://opportunityinsights.org/data/ ("The Opportunity Atlas: Mapping the Childhood Roots of Social Mobility") |
| `cty_covariates.dta` | Opportunity Insights "Neighborhood Characteristics by County" — https://opportunityinsights.org/data/ |
| `Table_8_county_covariates.dta` | Chetty, Hendren, Kline, Saez (2014), Table 8 county covariates — https://opportunityinsights.org/paper/land-of-opportunity/ |
| `onlinedata3 (2).dta` | Chetty, Hendren, Kline, Saez (2014), Online Data Table 3 — https://opportunityinsights.org/paper/land-of-opportunity/ |
| `health_ineq_online_table_12.dta` | Chetty et al. (2016), Health Inequality Project, Online Table 12 — https://healthinequality.org/data/ |
| `social_capital_county.csv` | Chetty et al. (2022) social capital data — https://opportunityinsights.org/data/ ("Social Capital and Economic Mobility") |
| `NCHSurb-rural-codes.csv` | NCHS Urban–Rural Classification Scheme — https://www.cdc.gov/nchs/data_access/urban_rural.htm |

`health_ineq_online_table_12.dta` is **latin-1 encoded** and `NCHSurb-rural-codes.csv` is also read with `encoding="latin1"`; the notebook handles both.

## What's modelled

- **Outcome**: `kfr_pooled_pooled_p25` — mean income rank of children whose parents were at the 25th percentile of the national income distribution.
- **11 predictors** (standardised before fitting): two segregation measures, two school-quality measures, family structure, inequality, commuting, social capital, poverty share, median household income, population density. Construction and fallbacks are documented in [CLAUDE.md](CLAUDE.md).
- **Models**: baseline OLS with HC3 SEs, OLS with four theory-motivated interactions, OLS by NCHS urban-rural class (heterogeneity check), random forest with SHAP global / SHAP interaction / partial-dependence interpretation.

## Notes for graders / readers

- AI-tool usage is disclosed in [appendix.md](appendix.md).
- Code follows the course rules: no function exceeds 10 lines, no emojis, every code cell preceded by a markdown cell explaining the *what* and *why*.
- Working notes — design decisions, locked-in conventions, status / open questions — live in [CLAUDE.md](CLAUDE.md).
