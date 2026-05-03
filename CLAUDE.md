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

All raw files live at the project root or in `mobility_analysis_export/`. Working notebook: [mobility_analysis_export/draft1.ipynb](mobility_analysis_export/draft1.ipynb).

| Source | File | Used for |
|---|---|---|
| Opportunity Atlas (2018) | `county_outcomes_dta.dta` / `county_outcomes.csv` | Outcome: `kfr_pooled_pooled_p25`; race × sex p25 mobility for gap analysis |
| Geography of Mobility (2014) | `onlinedata3 (2).dta` / `online_data_tables-2.xls` | `gini`, `inc_share_1perc`, `s_rank_8082` |
| Changing Opportunity (2022) | `cty_covariates.dta` / `cty_covariates.csv` | `mean_commutetime2000`, `gsmn_math_g3_2013`, `singleparent_share2010`, `poor_share2010`, etc. |
| Health Inequality (Chetty Table 12) | `health_ineq_online_table_12.dta` | `cs00_seg_inc` (income segregation), `score_r` (income-adjusted test-score percentile), `ccd_pup_tch_ratio` (pupil-teacher ratio) — added in draft2 |
| Social capital | `mobility_analysis_export/social_capital_county.csv` | `ec_county` (economic connectedness) |
| Urban–rural | `mobility_analysis_export/NCHSurb-rural-codes.csv` | NCHS 6-level urban-rural classification |
| Chetty Table 8 | `Table_8_county_covariates.csv` / `.dta` | Additional covariates |

**Merge key**: 5-digit FIPS = `state * 1000 + county` (integer). Always cast `state` and `county` to `int` before constructing FIPS — some sources load them as float.

## Locked-in modeling decisions

These are settled; don't re-derive in new cells. If a change is needed, update this section and the notebook together.

**Outcome**: `mobility = kfr_pooled_pooled_p25` (mean income rank of children whose parents were at the 25th percentile).

**Secondary outcomes** (for gap/heterogeneity analysis, not main models): `racial_gap_wb = kfr_white_male_p25 − kfr_black_male_p25`, `gender_gap_black = kfr_black_female_p25 − kfr_black_male_p25`.

**The 11 predictors** (standardized, mean 0, SD 1 before fitting):

```python
PREDICTORS = ["seg_score", "seg_income",
              "school_quality", "student_teacher_ratio",
              "single_parent", "inequality", "commute", "social_capital",
              "poor_share2010", "med_hhinc2016", "popdensity2010"]
```

The original specification used 9 predictors. As of the [Analysis/draft2.ipynb](Analysis/draft2.ipynb) revision, the segregation construct is captured by **two** complementary measures (racial-economic concentration *and* income segregation) and the school-quality construct is captured by **two** complementary measures (an income-adjusted output measure *and* a resource-input measure). The new variables come from `health_ineq_online_table_12.dta` (Chetty Health Inequality Project, Table 12), which is latin-1 encoded — load with `pyreadstat.read_dta(..., encoding="latin1")`. The merge key is `cty` (5-digit FIPS, integer). When this expansion was made the analytic sample fell from 1,578 to 1,382 counties, driven mostly by `ccd_pup_tch_ratio`'s ~8% missingness.

**Constructions** (with fallbacks to handle missing in newer release):

- `seg_score`             = `poor_share_black2010`, fallback `poor_share2010` — racial-economic concentration
- `seg_income`            = `cs00_seg_inc` — income segregation (Chetty Health Inequality Table 12)
- `inequality`            = `gini2010`, fallback `gini`
- `single_parent`         = `singlepar_pooled2010`, fallback `singleparent_share2010`
- `school_quality`        = `score_r` (Chetty Health Inequality Table 12) — income-adjusted test-score percentile. Replaces the prior `gsmn_math_g3_2013` (raw 3rd-grade math).
- `student_teacher_ratio` = `ccd_pup_tch_ratio` (Chetty Health Inequality Table 12) — pupil-to-teacher ratio
- `commute`               = `mean_commutetime2000`
- `social_capital`        = `ec_county`

**Missing data**: complete-case (`.dropna()`) on the 9 predictors + outcome. Sample size is reported as `len(analytic)`. If switching to imputation, note it here and justify.

**Regime variables** (for stratified looks, not main predictors): `seg_hi` = above-median `poor_share_black2010`; `pov_hi` = above-median `poor_share2010`; `regime_label` = the 2×2 cross.

## Model framework

1. **Baseline OLS** on the 11 standardized predictors, **HC3 robust SEs**.
2. **OLS + interactions**: 4 theory-motivated terms — `seg_score×school`, `seg_score×single_parent`, `inequality×single_parent`, `seg_score×social_capital`. The interactions continue to use `seg_score` (racial-economic concentration) for direct comparability with the original 9-predictor specification; `seg_income` and `student_teacher_ratio` enter only as main effects.
3. **OLS by NCHS urban-rural class** (6 strata) — heterogeneity check.
4. **Random forest** on the same 11 predictors, hyperparameters via 5-fold CV.
5. **RF interpretation**: SHAP values (global importance + direction), SHAP interaction values (top pairs), partial dependence plots for top features.

When comparing OLS vs RF, report 5-fold CV R² for both — out-of-sample, not in-sample, since the question is about what RF *uncovers* beyond linear structure.

## Code & plotting conventions

The notebook uses a fixed palette and matplotlib rcParams (see cell 4 of [draft1.ipynb](mobility_analysis_export/draft1.ipynb)). Reuse them in any new figure so the writeup looks coherent:

- Colors: `BLUE=#2C6E9B`, `RED=#C0392B`, `GREEN=#27AE60`, `ORANGE=#E67E22`, `PURPLE=#8E44AD`, `GRAY=#7F8C8D`, background `BG=#F8F9FA`.
- No top/right spines, dashed grid at 0.35 alpha, DejaVu Sans.
- Tables of coefficients/results: standardized betas, 3 decimals, with HC3 SEs.

Standard imports: `numpy`, `pandas`, `matplotlib`, `seaborn`, `pyreadstat` (for `.dta`), `statsmodels.api as sm`, `sklearn.ensemble.RandomForestRegressor`, `shap`.

## Working norms for Claude

- **Don't introduce a different predictor set or outcome** without flagging it. Stick to the locked-in 9 unless we're explicitly extending.
- **Don't silently change merge logic, FIPS construction, or fallbacks.** They're the result of a several-source reconciliation.
- **Standardize predictors before any model that compares coefficients across variables.**
- **Always report the analytic sample size** when models drop rows.
- **For the writeup angle, lead with interpretation, not accuracy numbers.** The point is mechanism (interactions, heterogeneity), not predictive horse-race.
- When adding cells, match the notebook's existing markdown rhythm: brief description → code → one-line interpretation under the figure/table.

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
