# Appendix: AI Use Disclosure

This appendix discloses how AI tools were used in the preparation of Econ 148 Project 3 (Track A, Topic 4: Intergenerational Mobility), in accordance with the course's professionalism and disclosure requirements.

## Tools used

- **Claude Code** (Anthropic), Opus 4.7 model, run from inside VS Code as an interactive coding assistant.

## Tasks performed with AI assistance

| Area | Task |
|---|---|
| Data engineering | Drafting the six-source merge on 5-digit FIPS, including casting `state` and `county` to `int` and the `gini2010` / `singlepar_pooled2010` / `poor_share_black2010` fallback logic. |
| Feature engineering | Constructing `seg_score`, `inequality`, `single_parent`, `seg_hi`, `pov_hi`, and `regime_label`, plus standardizing the 9-predictor matrix. |
| Modeling | Setting up the baseline OLS with HC3 standard errors, the four theory-motivated interaction terms, the urban-rural-stratified OLS, and the random forest with 5-fold CV. |
| Interpretation | Generating SHAP global summaries, SHAP interaction matrices, and partial dependence plots for the top features. |
| Plotting | Defining the project color palette and matplotlib rcParams; drafting the multi-panel figures (mobility distribution, regimes, OLS coefficients, model R² comparison, SHAP summaries, PDPs). |
| Documentation | Authoring `CLAUDE.md` (project framing, locked-in modeling decisions, working norms, course coding requirements) and the per-cell markdown intros in the working notebook. |
| Code quality | Refactoring `coef_plot` into three helpers (`get_coef_data`, `draw_coef_bars`, `coef_plot`) so each function fits the 10-line modularity rule. |
| Predictor-set expansion (draft2) | Investigating whether `health_ineq_online_table_12.dta` (Chetty Health Inequality Project, Table 12) merges cleanly with the existing analytic sample, computing missingness and overlap, comparing each candidate variable's correlation with the existing predictors, and then implementing the agreed-on substitution: replacing `gsmn_math_g3_2013` with `score_r` (income-adjusted test-score percentile) inside `school_quality`, and adding `seg_income` (`cs00_seg_inc`) and `student_teacher_ratio` (`ccd_pup_tch_ratio`) as new predictors. The change was justified by (a) low correlation of `cs00_seg_inc` with the existing `seg_score` (r = -0.03, capturing a different segregation dimension) and (b) the income-adjustment in `score_r` being conceptually closer to school value-added than the raw 3rd-grade math score. |
| Segregation-measure swap (draft2) | Investigating what the original `seg_score = poor_share_black2010` variable actually measured (Black poverty rate, not segregation), computing pairwise correlations between `seg_score`, `cs_race_theil_2000` (racial segregation Theil), and `cs00_seg_inc` (income segregation Theil), and demonstrating that `seg_score` was essentially uncorrelated (r ≈ 0.05) with both proper segregation measures while having the weakest correlation with the outcome (r = −0.118 vs −0.288 for racial Theil and −0.258 for income Theil). Implementing the agreed-on swap: dropping `seg_score`, adding `seg_race = cs_race_theil_2000` as a true racial-segregation predictor, and updating the `seg_hi` regime stratifier to use `seg_race` instead of `poor_share_black2010` so the "High Seg / High Pov" labels match what they claim to measure. Sample size unchanged (1,382 counties) since `cs_race_theil_2000` has zero missingness. |

## Verification methods

- **Notebook execution**: every code cell was run end-to-end in the working notebook ([mobility_analysis_export/draft1.ipynb](mobility_analysis_export/draft1.ipynb)), and outputs (sample size, R² values, coefficient tables, figures) were inspected against expectations from the cited Chetty et al. papers.
- **Sample-size accounting**: the analytic sample size is printed after the `.dropna()` step. In draft1 the sample was 1,578 of 3,219 counties; in draft2, with the predictor set expanded from 9 to 11, the sample fell to 1,382 (mostly from `ccd_pup_tch_ratio`'s 8% missingness). Any silent change in the merge or feature construction would surface as a row-count change.
- **Cross-checks against the literature**: directions and rough magnitudes of the OLS coefficients (positive social capital, negative single-parent share, negative commute time) were verified against the published Chetty, Hendren, Kline, Saez (2014) and Chetty et al. (2022) results before being included.
- **OLS-vs-RF comparison**: random forest performance is reported as 5-fold CV R² (out-of-sample), not in-sample, so apparent gains over OLS reflect genuine non-linear structure rather than overfitting.
- **Code-style audit**: the notebook was scanned for emojis (none present), and every code cell was confirmed to be preceded by a markdown cell explaining the *what* and *why*. Function lengths were checked to confirm no function exceeds 10 lines of code.

## Scope of authorship

All economic framing, choice of research question, choice of predictor set, interpretation of results, and final write-up decisions are the author's. AI assistance was used to accelerate implementation and documentation, not to substitute for analytical judgment.
