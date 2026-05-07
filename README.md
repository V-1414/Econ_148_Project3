# Econ 148 Project 3 — Intergenerational Mobility Across U.S. Counties

Track A (Open Data Project), Topic 4. The economic question: *what county-level features predict upward mobility for children born to low-income parents, and does a non-linear model uncover interactions the standard linear specification misses?*

The analysis pairs an OLS baseline (the variables Chetty et al. emphasise) with a random-forest comparison and SHAP-based interpretation. The interesting finding is in the **interactions** the random forest surfaces, not the headline R² gap. 

## Contents of this submission

```
Econ_148_Project3/
├── Analysis/
│   ├── analysis.ipynb       # the notebook to run
│   └── data/                # all datasets needed for replication, bundled
├── README.md                # this file
├── CLAUDE.md                # working notes / locked-in modelling decisions
└── appendix.md              # AI-tool usage disclosure 
```

The data folder is bundled inside `Analysis/` so the notebook's relative paths (`pyreadstat.read_dta("data/...")`) resolve without edits. Folder name is **lowercase `data/`** because that is what the notebook reads — do not rename it to `Data/` on a case-sensitive filesystem (e.g. Linux / DataHub).

## How to replicate

Two paths are supported. Both use the same notebook and the same bundled data; the only difference is where Python runs.

### Option A: Local (VS Code + Jupyter)

1. **Unzip** the submission. From here on, `Econ_148_Project3/` is the working directory.
2. **Python 3.10+** required. From the `Econ_148_Project3/` directory, install dependencies:
   ```
   python3 -m pip install numpy pandas matplotlib seaborn pyreadstat statsmodels scikit-learn shap
   ```
   (The notebook also auto-installs `pyreadstat` and `shap` on first run via `!{sys.executable} -m pip install ...`, so this step is optional.)
3. **Open the notebook** in VS Code (or any Jupyter front-end):
   ```
   jupyter notebook Analysis/analysis.ipynb
   ```
4. **Run all cells top-to-bottom.** Use *Kernel → Restart & Run All* (Jupyter) or *Run All* (VS Code). Total runtime is roughly 5–10 minutes; the slowest cells are the random-forest hyperparameter sweep and the SHAP interaction-value computation.

### Option B: UC Berkeley DataHub

1. Log in to DataHub (`datahub.berkeley.edu`) with your Berkeley account.
2. Click *Upload* and upload the `Analysis/` folder (or, if DataHub blocks folder uploads, upload `analysis.ipynb` and the `data/` folder separately so they end up at `Analysis/analysis.ipynb` and `Analysis/data/...`).
3. Open `analysis.ipynb` in DataHub's Jupyter interface.
4. **Run all cells top-to-bottom** with *Kernel → Restart & Run All*. The first cell auto-installs `pyreadstat` and `shap` into the DataHub kernel; everything else is pre-installed.

In either path the notebook should reproduce every figure, table, and reported number end-to-end with no manual intervention.

## Reproducibility guarantees

The notebook is designed to produce identical numbers and figures on every run. Three things make that work:

1. **Single seed**. All stochastic operations (random forest, K-fold splits, NumPy and Python `random`) read from the `SEED` constant defined in the imports/setup section (currently `SEED = 42`).
2. **Complete-case analytic sample**. Rows missing the outcome or any predictor are dropped before any model is fit; the sample size (currently 1,382 counties) is printed at the end of the feature-engineering section.
3. **Relative paths only**. The notebook reads from `data/...` relative to its own directory. As long as the unzipped folder structure is preserved, no path edits are needed.

If you need to re-download the underlying data from primary sources (e.g. to verify the bundled files), the seven datasets come from:

| File | Primary source |
|---|---|
| `county_outcomes_dta.dta` | Opportunity Atlas — https://opportunityinsights.org/data/ |
| `cty_covariates.dta` | Opportunity Insights "Neighborhood Characteristics by County" — https://opportunityinsights.org/data/ |
| `Table_8_county_covariates.dta` | Chetty, Hendren, Kline, Saez (2014), Table 8 — https://opportunityinsights.org/paper/land-of-opportunity/ |
| `onlinedata3 (2).dta` | Chetty, Hendren, Kline, Saez (2014), Online Data Table 3 — https://opportunityinsights.org/paper/land-of-opportunity/ |
| `health_ineq_online_table_12.dta` | Chetty Health Inequality Project, Online Table 12 — https://healthinequality.org/data/ |
| `social_capital_county.csv` | Chetty et al. (2022), Social Capital Atlas — https://opportunityinsights.org/data/ |
| `NCHSurb-rural-codes.csv` | NCHS Urban–Rural Classification Scheme — https://www.cdc.gov/nchs/data_access/urban_rural.htm |

`health_ineq_online_table_12.dta` is **latin-1 encoded** and `NCHSurb-rural-codes.csv` is also read with `encoding="latin1"`; the notebook handles both.

## What's modelled

- **Outcome**: `kfr_pooled_pooled_p25` — mean adult income rank of children whose parents were at the 25th percentile of the national income distribution.
- **11 predictors** (standardised before fitting): two segregation measures (racial Theil and income Theil), two school-quality measures (output and input), family structure, inequality, commuting, social capital, poverty share, median household income, population density. Constructions and fallbacks are documented in [CLAUDE.md](CLAUDE.md) and in the `Feature Engineering` section of the notebook.
- **Models**: baseline OLS with HC3 standard errors, OLS with four theory-motivated interaction terms, OLS by NCHS urban-rural class (heterogeneity check), per-class interaction OLS, segregation-only specifications (race-only / income-only / both), and a random forest with SHAP global / SHAP interaction / partial-dependence interpretation.

## Notes for graders

- **AI-tool usage** is disclosed in [appendix.md](appendix.md).