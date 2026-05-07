# CLAUDE.md — Econ 148 Project 3

Working notes for Claude. Update this file as decisions are made so future sessions stay consistent. Don't restate what's obvious from the code; capture decisions, framings, and conventions that should persist.

## Project framing

- **Course / track**: Econ 148 Project 3, Track A (Open Data Project).
- **Topic**: Intergenerational mobility across US counties (Topic 4).
- **Required deliverables**:
  1. A clear economic question.
  2. **Econometric baseline** (OLS with the variables Chetty et al. emphasize).
  3. **ML comparison** (random forest with feature importance).
  4. Discussion of what worked, what didn't.
- **The interesting question** (per the spec, and what the writeup should center on): not just predictive accuracy — *whether ML uncovers interactions the linear model misses, and what those interactions imply for policy*. Treat OLS-vs-RF gap, SHAP interaction values, and PDPs as the core evidence, not headline R².

## Data

Raw data files live in `Analysis/Data/` (`.dta` and `.csv`) and `Other data/` (`.xls`). They are gitignored — re-download from Opportunity Insights and the Chetty Health Inequality Project. The active working notebook is [Analysis/draft2.ipynb](Analysis/draft2.ipynb); [Analysis/draft1.ipynb](Analysis/draft1.ipynb) is preserved as the 9-predictor baseline.

| Source | File | Used for |
|---|---|---|
| Opportunity Atlas (2018) | `county_outcomes_slim.csv` (canonical bundled file; provenance/construction script in README) | Outcome: `kfr_pooled_pooled_p25`; race × sex p25 mobility for gap analysis |
| Geography of Mobility (2014) | `onlinedata3 (2).dta` / `online_data_tables-2.xls` | `gini`, `inc_share_1perc`, `s_rank_8082` |
| Changing Opportunity (2022) | `cty_covariates.dta` / `cty_covariates.csv` | `mean_commutetime2000`, `gsmn_math_g3_2013`, `singleparent_share2010`, `poor_share2010`, etc. |
| Health Inequality (Chetty Table 12) | `health_ineq_online_table_12.dta` | `cs_race_theil_2000` (racial segregation, Theil), `cs00_seg_inc` (income segregation, Theil), `score_r` (income-adjusted test-score percentile), `ccd_pup_tch_ratio` (pupil-teacher ratio) — added in draft2 |
| Social capital | `Analysis/Data/social_capital_county.csv` | `ec_county` (economic connectedness) |
| Urban–rural | `Analysis/Data/NCHSurb-rural-codes.csv` | NCHS 6-level urban-rural classification |
| Chetty Table 8 | `Table_8_county_covariates.csv` / `.dta` | Additional covariates |

**Merge key**: 5-digit FIPS = `state * 1000 + county` (integer). Always cast `state` and `county` to `int` before constructing FIPS — some sources load them as float.

## Locked-in modeling decisions

These are settled; don't re-derive in new cells. If a change is needed, update this section and the notebook together.

**Outcome**: `mobility = kfr_pooled_pooled_p25` (mean income rank of children whose parents were at the 25th percentile).

**Secondary outcomes** (for gap/heterogeneity analysis, not main models): `racial_gap_wb = kfr_white_male_p25 − kfr_black_male_p25`, `gender_gap_black = kfr_black_female_p25 − kfr_black_male_p25`.

**The 11 predictors** (standardized, mean 0, SD 1 before fitting):

```python
PREDICTORS = ["seg_race", "seg_income",
              "school_quality", "student_teacher_ratio",
              "single_parent", "inequality", "commute", "social_capital",
              "poor_share2010", "med_hhinc2016", "popdensity2010"]
```

The original specification used 9 predictors. As of the [Analysis/draft2.ipynb](Analysis/draft2.ipynb) revision: (a) the segregation construct is captured by **two** complementary Theil-index measures (racial segregation *and* income segregation, both Census 2000 from the Health Inequality Project), and (b) the school-quality construct is captured by **two** complementary measures (an income-adjusted output measure *and* a resource-input measure). All four new variables come from `health_ineq_online_table_12.dta` (Chetty Health Inequality Project, Table 12), which is latin-1 encoded — load with `pyreadstat.read_dta(..., encoding="latin1")`. The merge key is `cty` (5-digit FIPS, integer). The analytic sample is 1,382 counties (down from 1,578 in the original 9-predictor draft, driven mostly by `ccd_pup_tch_ratio`'s ~8% missingness).

The earlier `seg_score = poor_share_black2010` variable was dropped after diagnostic checks showed it was essentially uncorrelated (r ≈ 0.05) with proper Theil-index segregation measures and was more accurately a *Black poverty rate* than a segregation measure. `poor_share_black2010` is still loaded but is no longer used as a predictor; the `seg_hi` regime stratifier now uses `seg_race` instead.

**Constructions** (with fallbacks to handle missing in newer release):

- `seg_race`              = `cs_race_theil_2000` (Chetty Health Inequality Table 12) — Theil-index racial segregation, Census 2000
- `seg_income`            = `cs00_seg_inc` (Chetty Health Inequality Table 12) — Theil-index income segregation, Census 2000
- `inequality`            = `gini2010`, fallback `gini`
- `single_parent`         = `singlepar_pooled2010`, fallback `singleparent_share2010`
- `school_quality`        = `score_r` (Chetty Health Inequality Table 12) — income-adjusted test-score percentile. Replaces the prior `gsmn_math_g3_2013` (raw 3rd-grade math).
- `student_teacher_ratio` = `ccd_pup_tch_ratio` (Chetty Health Inequality Table 12) — pupil-to-teacher ratio
- `commute`               = `mean_commutetime2000`
- `social_capital`        = `ec_county`

**Missing data**: complete-case (`.dropna()`) on the 11 predictors + outcome. Sample size is reported as `len(analytic)`. If switching to imputation, note it here and justify.

**Regime variables** (for stratified looks, not main predictors): `seg_hi` = above-median `seg_race` (racial-segregation Theil index); `pov_hi` = above-median `poor_share2010`; `regime_label` = the 2×2 cross.

## Model framework

1. **Baseline OLS** on the 11 standardized predictors, **HC3 robust SEs**.
2. **OLS + interactions**: 4 theory-motivated terms — `seg_race×school`, `seg_race×single_parent`, `inequality×single_parent`, `seg_race×social_capital`. The interactions use `seg_race` (the racial-segregation Theil index) since the original Chetty 2014 interaction theory was about racial segregation; `seg_income` enters only as a main effect.
3. **OLS by NCHS urban-rural class** (6 strata) — heterogeneity check.
4. **Random forest** on the same 11 predictors. Fixed hyperparameters (`n_estimators=500`, `max_features=0.33`, `min_samples_leaf=5`); 5-fold CV is used to *score* the model, not to tune. If we add tuning later (e.g. `RandomizedSearchCV`), pass `random_state=SEED`.
5. **RF interpretation**: SHAP values (global importance + direction), SHAP interaction values (top pairs), partial dependence plots for top features.

When comparing OLS vs RF, report 5-fold CV R² for both — out-of-sample, not in-sample, since the question is about what RF *uncovers* beyond linear structure.

## Code & plotting conventions

The notebook uses a fixed palette and matplotlib rcParams (see the imports/settings cell of [Analysis/draft2.ipynb](Analysis/draft2.ipynb)). Reuse them in any new figure so the writeup looks coherent:

- Colors: `BLUE=#2C6E9B`, `RED=#C0392B`, `GREEN=#27AE60`, `ORANGE=#E67E22`, `PURPLE=#8E44AD`, `GRAY=#7F8C8D`, background `BG=#F8F9FA`.
- No top/right spines, dashed grid at 0.35 alpha, DejaVu Sans.
- Tables of coefficients/results: standardized betas, 3 decimals, with HC3 SEs.

Standard imports: `numpy`, `pandas`, `matplotlib`, `seaborn`, `pyreadstat` (for `.dta`), `statsmodels.api as sm`, `sklearn.ensemble.RandomForestRegressor`, `shap`.

## Working norms for Claude

- **Don't introduce a different predictor set or outcome** without flagging it. Stick to the locked-in 11 unless we're explicitly extending.
- **Don't silently change merge logic, FIPS construction, or fallbacks.** They're the result of a several-source reconciliation.
- **Standardize predictors before any model that compares coefficients across variables.**
- **Always report the analytic sample size** when models drop rows.
- **For the writeup angle, lead with interpretation, not accuracy numbers.** The point is mechanism (interactions, heterogeneity), not predictive horse-race.
- When adding cells, match the notebook's existing markdown rhythm: brief description → code → one-line interpretation under the figure/table.
- **Document non-obvious modeling decisions inline.** When the notebook makes a methodological call that a grader reading top-to-bottom would pause on — *why HC3 SEs and not default? why these specific four interactions? why complete-case rather than imputation? why fixed RF hyperparameters at these values?* — extend the preceding markdown cell with a 1–2 sentence justification grounded in either theory, the Chetty literature, or the structure of the data. **Be conservative.** Don't justify trivial mechanics (loading a CSV, casting a column, drawing a histogram). The test: would a thoughtful reader who knows econometrics still want to know *why this choice*? If yes, explain. If the choice is mechanical or follows from a decision already justified upstream, leave it alone. The goal is to surface real methodological reasoning for graders, not to bury the analysis under commentary.

## Reproducibility

A core project goal: anyone cloning the repo, downloading the listed data files, and running the active notebook should reproduce every number and figure in the writeup. Two implications:

- **Single seed**. All stochastic operations use the `SEED` constant defined in the imports/setup section of [Analysis/draft2.ipynb](Analysis/draft2.ipynb). Currently `SEED = 42`. NumPy and Python's `random` module are seeded globally; sklearn objects (`RandomForestRegressor`, `KFold`) take `random_state=SEED` explicitly. **Don't introduce a new stochastic call without threading `SEED` into it** — that includes any future `train_test_split`, `RandomizedSearchCV`, `shap.sample`, bootstrap, or permutation-importance call.
- **Readable steps**. Every code cell is preceded by a markdown cell explaining what the step does and why it belongs there (this is also the course documentation rule). Treat the notebook as something a peer should be able to read top-to-bottom and follow without external context.

Package versions matter for sklearn/SHAP output stability — pin them when this matures into a final submission. The README is the canonical place for download links and run instructions; keep it in sync when the active notebook changes (e.g. when draft2 is superseded).

## Course coding requirements (hard rules)

These are graded constraints from the project rubric. Do not violate them when editing notebooks or supporting files.

- **Modular code**: No function may exceed 10 lines of code (excluding blank lines and the `def` signature line). Split helpers when needed.
- **Professionalism**: No emojis anywhere — code, comments, markdown cells, figure titles, file names, or commit messages.
- **Documentation**: Every code cell in any notebook must be preceded by a markdown cell explaining the *what* (one line on what the code does) and the *why* (one line on why this step belongs here). Do not stack consecutive code cells without a markdown intro.
- **Disclosure**: AI-tool usage must be disclosed in [appendix.md](appendix.md) at the project root, listing tools used, tasks performed, and verification methods. Update this file whenever AI is used for a new task.

## Status / next steps

Track progress here; move items to "done" rather than deleting so we have a record.

**Done**
- Six-source merge on FIPS, analytic sample built.
- Baseline OLS, OLS + 4 interactions, OLS by urban-rural class.
- Random forest with 5-fold CV, SHAP global + interaction, PDPs for top features.
- EDA: mobility distribution, urban-rural patterns, correlation heatmap, seg×poverty regimes.

**Open / to decide**
- Final write-up structure and how heavily to feature the racial/gender gap secondary outcomes.
- Whether to add a regularized linear model (Lasso/Ridge) as a middle ground between OLS and RF.
- Whether 5-fold CV R² is the headline metric, or held-out test set.
- Robustness: spatial autocorrelation? CZ-clustered SEs?

## References / external context

- Chetty, Hendren, Jones, Porter (2020), *Race and Economic Opportunity in the United States* — basis for the Opportunity Atlas outcome.
- Chetty, Hendren, Kline, Saez (2014), *Where is the Land of Opportunity?* — basis for the original 5-factor framework (segregation, inequality, school quality, social capital, family structure) the OLS baseline mirrors.
- Chetty et al. (2022), *Social Capital and Economic Mobility* — basis for `ec_county`.
- Project spec PDF: [Project 3 Checkpoint 1.pdf](Project%203%20Checkpoint%201.pdf).
