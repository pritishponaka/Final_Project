# A New Hampshire Paradox: The Arsenic Extension

A statistical and machine-learning analysis of arsenic in New Hampshire bedrock well water, built as the companion arm of the *A New Hampshire Paradox* capstone.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/pritishponaka/Final_Project/blob/main/A_NH_Paradox_Arsenic_Testing_Extension_FP.ipynb)

**Capstone dashboard:** [A New Hampshire Paradox](https://pritish-ponaka.shinyapps.io/nhparadox/)

**Companion website:** [A New Hampshire Paradox? (interactive site)](https://pritishponaka.github.io/Final_Project/), with this analysis on its [second page](https://pritishponaka.github.io/Final_Project/#arsenic). The notebook and the website are the finished deliverable.

---

## Contents

1. [What this project does](#1-what-this-project-does)
2. [Quick start](#2-quick-start)
3. [The file you provide](#3-the-file-you-provide)
4. [Settings you can change](#4-settings-you-can-change)
5. [How the notebook is organized](#5-how-the-notebook-is-organized)
6. [What it generates](#6-what-it-generates)
7. [How results are judged](#7-how-results-are-judged)
8. [Design decisions worth knowing](#8-design-decisions-worth-knowing)
9. [Troubleshooting](#9-troubleshooting)
10. [Testing and reproducibility](#10-testing-and-reproducibility)
11. [Limitations](#11-limitations)
12. [Repository layout](#12-repository-layout)
13. [Data source, citations, and license](#13-data-source-citations-and-license)

---

## 1. What this project does

**The context.** New Hampshire has one of the highest bladder cancer rates in the country, even though it is wealthy and has low unemployment. The capstone asks whether that reflects real disease burden or a "masking effect," where higher income means more screening and therefore more *detected* cases. The capstone named arsenic in private well water as a confounder and left it out of scope. This notebook picks that thread back up.

**The question.** Using only a well's geology, geochemistry, and construction, can we predict whether its arsenic is above a safety threshold (1, 5, or 10 µg/L)? If so, a model could help decide which private wells to test first.

**The data.** The U.S. Geological Survey's *Testing data set for independent analysis of New Hampshire arsenic model*: 373 bedrock wells and 45 predictor columns. About 18 percent of the wells (roughly 68) are above the default 5 µg/L threshold.

**The deliberate choice.** 373 records is small on purpose. Real public-health data is usually small and partly hidden: each well's location is censored to the county, a few geology flags are set for only one to three wells, and about a fifth of the raw concentration values are missing. The notebook works with those limits openly instead of avoiding them.

**What it predicts, and what it does not.** It predicts arsenic *in water*. It does not predict cancer in people, and it cannot prove a link between the two. Individual-level cancer data for this population is restricted.

## 2. Quick start

### Google Colab (easiest)

1. Click the **Open in Colab** badge at the top.
2. Choose **Runtime, then Run all**.
3. When section 2 runs, an **upload dialog** appears under the cell. Choose your data file. (Or skip the dialog by setting `DATA_URL` to a GitHub link, or `DATA_PATH` to a file you uploaded to Colab's Files panel. See section 4.)
4. The first code cell installs every package it needs, so there is nothing to set up.

### On your own computer

```bash
# 1. Make a fresh environment (recommended, see Troubleshooting for why)
conda create -n nh_arsenic python=3.11 -y
conda activate nh_arsenic

# 2. Install everything in one command
pip install -r requirements.txt
python -m ipykernel install --user --name nh_arsenic --display-name "Python (nh_arsenic)"

# 3. Open the notebook and choose the "Python (nh_arsenic)" kernel
jupyter notebook A_NH_Paradox_Arsenic_Testing_Extension_FP.ipynb
```

Then choose **Kernel, then Restart & Run All**. When section 2 runs, a file window opens. Pick your data file. To skip the window, set `DATA_PATH` or `DATA_URL` in the settings cell (see below).

**Run time.** Most of the notebook runs in under a minute. The slow step is section 19 (nested cross-validation), which repeats all the tuning inside every fold. Expect several minutes. To speed up a first pass, call `plan_pipeline(..., requested_repeats=1)` in section 12.

## 3. The file you provide

The notebook reads **your** data directly, from whichever of these you set (it uses the first one that is set):

1. **`DATA_PATH`**: a file on your computer, or one uploaded to Colab's Files panel.
2. **`DATA_URL`**: a link to a CSV in a GitHub repository. A normal GitHub page link works, because the notebook converts it to the direct "raw" link. If the link cannot be read, the notebook says why and switches to option 3.
3. **A file picker**: an upload dialog in Colab, or a file window on your own computer.

| Item | Detail |
|---|---|
| **Formats** | `.csv`, `.xlsx`, `.xls` |
| **One row per** | well |
| **Required column** | `verified_As`: the measured arsenic concentration in µg/L |
| **Recommended columns** | `bas1`, `bas5`, `bas10`: ready-made 0/1 flags for "at or above 1, 5, or 10 µg/L". If the flag for your chosen threshold is present, complete, and contains only 0 and 1, it is used as the target. Otherwise the target is built from `verified_As`. |
| **For the maps** | a `County` column. Without it, the two maps are skipped. |
| **Ignored as an ID** | `obs` |
| **Everything else** | treated as a predictor: numbers, 0/1 flags, or text categories (such as `County` and `WELLCLAS`) |
| **Project minimums** | 300 rows and 10 predictors. Falling short gives a warning, not an error. |

**If you do not have the file,** the data-selection notebook downloads it for you. Put its link in `DATA_SELECTION_NOTEBOOK_URL` (see below). Every error message repeats that link. The dataset itself is public domain (CC0): <https://doi.org/10.5066/F7XK8CQV>.

### What is checked before any analysis runs

The notebook stops with a plain-language message, listing every problem at once, if:

- the file or link does not exist, is empty, is not a supported format, or cannot be parsed (a link that returns a web page instead of data is caught),
- column names are repeated,
- `verified_As` is missing or contains no numbers,
- the ready-made flag contains anything other than 0 and 1,
- only one outcome is present, or the smaller outcome has fewer than 6 wells (too few to cross-validate).

It warns, without stopping, if there are fewer than 300 rows or 10 predictors, if columns are empty, constant, or more than half missing, if the smaller outcome has fewer than 30 wells, or if there is no `County` column. Section 2 then prints a **DATA READ AND VALIDATED** summary (including where the data came from) and saves a profile of every column.

## 4. Settings you can change

All five sit together in one cell near the top (section 2). Change them, then run from the top.

| Setting | Default | What it does |
|---|---|---|
| `DATA_PATH` | `None` | A file on your computer, or one uploaded to Colab's Files panel, such as `"my_folder/file.csv"`. Read directly, and checked first. |
| `DATA_URL` | `None` | A GitHub link to a CSV, such as `"https://github.com/<user>/<repo>/blob/main/data/Testing_data.csv"`. Used when `DATA_PATH` is `None`. If it cannot be read, the file picker opens instead. |
| `THRESHOLD_UGL` | `5` | The arsenic level (1, 5, or 10 µg/L) above which a well counts as "exceeds." These match three published models. 10 µg/L is the legal limit for public water systems. |
| `USER_GOAL` | `"rank_for_testing"` | Fixes the **primary metric before any model runs**. `rank_for_testing` uses PR-AUC; `balanced_decision` uses MCC. See section 7. |
| `DATA_SELECTION_NOTEBOOK_URL` | a placeholder | Shown in every error message. Replace it with the link to your data-selection notebook. |

## 5. How the notebook is organized

Four parts and 26 sections (0 to 25). Each part uses only what the earlier parts produced, so it is meant to be run top to bottom.

| Part | Section | What it does |
|---|---|---|
| **1. Data input and validation** | 0 Setup | imports, optional-package handling, output folders, figure archive |
| | 1 Helper functions | county-name cleanup, the multiple-testing correction |
| | 2 Data upload and input validation | reads your file, validates it, profiles it, fixes the primary metric from your goal |
| | 3 `ArsenicDataset` | builds the yes/no target, removes columns that would give the answer away |
| | 4 Data documentation | every predictor, with an honest high/medium/low confidence level |
| **2. Data processing and exploration** | 5 Data quality | missing, impossible, extreme, rare, and constant values, each check aware of column type |
| | 6 Preprocessing | the cleaning pipeline every model shares, learned from training wells only |
| | 7 Dataset exploration | descriptive statistics, with the math |
| | 8 Visualizing the data | class balance, distributions, correlation heatmap |
| | 9 Mapping the results | a bivariate county map and a dot-density well map |
| **3. Analytical pipeline** | 10 Advanced statistical exploration | one significance test per predictor, chosen by data type and shape, with the formulas |
| | 11 Project planning | the prediction problem, the predefined primary metric, the selection rule |
| | 12 Pipeline plan | conditional logic: settings adapt to the data and your goal |
| | 13 Modeling roadmap | the workflow and the shared model blueprint |
| | 14 to 16 | logistic regression, random forest, gradient boosting, each with its math |
| | 17 Hyperparameter tuning | grid search with cross-validation, scored by log-loss |
| | 18 Inferential analysis | odds ratios with confidence intervals |
| **4. Model evaluation and interpretation** | 19 Nested cross-validation | ten metrics with bootstrap intervals, paired comparisons, McNemar's test |
| | 20 Model selection | the predefined metric picks the model, with a justification |
| | 21 Advanced visualizations | eight views of the results |
| | 22 What we can conclude | rules for turning results into conclusions |
| | 23 Capstone tie-back | what this adds to the paradox, and its limits |
| | 24 Results, the website, and possible later work | the saved files, the companion website, and ideas beyond this deliverable |
| | 25 Audit | every file generated, every requirement checked |

A plain-English companion, **`Notebook_Concepts_Guide.pdf`**, explains every section and, in particular, why each statistical test and model was chosen, with the math and the advantages and disadvantages of each.

## 6. What it generates

Files are saved to `outputs/`. Every figure shown is also archived in `outputs/figures/`. Section 25 checks this list automatically.

| File | Section | What it is |
|---|---|---|
| `data_profile.csv` | 2 | automatic profile of the uploaded file |
| `data_dictionary.csv` | 4 | every predictor with a description and a confidence level |
| `data_quality_report.txt` | 5 | missing, impossible, extreme, rare, and constant values |
| `01_class_balance.png`, `02_numeric_distributions.png`, `03_correlation_heatmap.png` | 8 | the exploratory charts |
| `05_county_choropleth.png`, `06_well_dot_density_map.png` | 9 | the two NH maps (need `geopandas`) |
| `statistical_screening.csv` | 10 | one test per predictor, with effect size and corrected p-value |
| `vif_table.csv` | 10 | overlap between predictors |
| `pipeline_plan.json` | 12 | the settings the data and goal produced |
| `model_selection_report.txt` | 20 | the selected model, the reason, a plain-language interpretation |
| `model_evaluation_summary.csv`, `model_cv_spread.csv` | 24 | every metric with intervals, and spread across repeats |
| `odds_ratios.csv` | 24 | the inferential table |
| `nh_arsenic_cleaned.csv` | 24 | the cleaned dataset with standardized columns |
| `audit_checklist.csv` | 25 | each requirement and its status |

## 7. How results are judged

- **Accuracy is not the headline.** About 82 percent of wells are below the threshold, so a model that always says "safe" scores about 82 percent accuracy while finding nothing. Accuracy is reported for reference only.
- **One primary metric, fixed in advance.** It is set from `USER_GOAL` before any model runs. The default, **PR-AUC**, measures how well the rare contaminated wells are ranked, which matches the practical use of deciding which wells to test first. It needs no decision cutoff and is not inflated by the many easy safe wells. `balanced_decision` uses **MCC** instead. Ten metrics are reported for every model: precision, recall, specificity, F1, MCC, balanced accuracy, ROC-AUC, PR-AUC, Brier score, and accuracy.
- **A fixed selection rule.** The model with the highest primary metric wins, unless a *simpler* model is statistically tied with it, in which case the simpler model wins. "Tied" means the 95 percent bootstrap interval for their paired difference includes zero. Simplicity order: logistic regression, random forest, gradient boosting.
- **Honest scoring.** Performance comes from **nested cross-validation**: for each fold, the model's settings and its decision cutoff are chosen using only that fold's training wells. Every well is predicted by a model that never saw it and whose settings it never influenced. The whole procedure is repeated with new random folds.
- **Uncertainty everywhere.** Every metric has a 95 percent bootstrap interval, and models are compared on identical resamples.

## 8. Design decisions worth knowing

- **Leakage guard.** The raw concentration, all three ready-made flags, and the row ID are removed from the predictors. Every cleaning step (medians, scaling, rare-flag merging, category levels) is learned inside each training fold only.
- **Suppression handled openly.** Flags set for fewer than 5 training wells are merged into one `rare_flag_any` column. Rare flags are tested with Fisher's exact test, not chi-square. County rates show wide intervals when few wells were tested. The dot map places wells at random inside their county because real coordinates are hidden, and says so on the map.
- **Conditional logic.** Section 12 sets the number of folds, repeats, and inferential predictors from the dataset's size and class balance, and prints each adaptation. On this dataset the standard settings fit. On a tiny subset the same rules cut 5 folds to 2.
- **Statistical test selection.** Each predictor is routed to a test by its type and shape: Welch's t-test when both groups look bell-shaped, Mann-Whitney U otherwise, chi-square for categories with enough wells, Fisher's exact for small 2 by 2 tables. All p-values are then corrected for running about 45 tests (Benjamini-Hochberg).
- **P-values in the inferential section are exploratory.** The six predictors there were chosen on geochemical grounds, but after the screening results had been seen, so the p-values are somewhat optimistic. The interval widths remain trustworthy.
- **Intent is stated before results.** The primary metric and the selection rule are written in the notebook (sections 2 and 11) before any result appears.

## 9. Troubleshooting

| What you see | Likely cause | Fix |
|---|---|---|
| `No module named 'sklearn'` | The package is installed as `scikit-learn`, and is not in this environment yet | Run `%pip install scikit-learn` in a notebook cell, restart the kernel, run from the top |
| "A module that was compiled using NumPy 1.x cannot be run in NumPy 2.x" | Packages in one environment were built against different NumPy versions | Use a fresh environment and install everything together: `pip install -r requirements.txt` |
| The file window never appears (local) | `tkinter` is unavailable, or the window opened behind your browser | Set `DATA_PATH = "path/to/file.csv"` or `DATA_URL` in the settings cell |
| "Not found (404)" for `DATA_URL` | A wrong path or branch name, or the repository is private | Check the link in a browser. Private repositories cannot be read. |
| "That link returned a web page, not a data file" | You used a page that shows the file, not the file itself | Open the file on GitHub and copy the link from the **Raw** button (a normal `.../blob/...` link also works) |
| "Could not reach ..." for `DATA_URL` | No internet connection, a typo in the address, or the server is down | Check the link and your connection. The notebook then falls back to the file picker. |
| The upload dialog never appears (Colab) | The cell has not run, or Colab restarted | Run the section 2 cells again from the top |
| "The dataset could not be used" | The validator found a problem | Read the message: it lists every problem and repeats the data-selection link |
| `Reading Excel files needs the 'openpyxl' package` | Excel support is not installed | `%pip install openpyxl`, or upload a CSV |
| The maps are skipped | `geopandas` or `shapely` is not installed | `pip install geopandas shapely`. Everything else still runs without them. |
| A multiprocessing error during tuning (often on Windows) | Parallel workers cannot start | Add `n_jobs=1` to the `HyperparameterTuner(...)` call and the `ModelSuite(...)` call |
| It is slow | Nested cross-validation repeats all the tuning in every fold | Use `requested_repeats=1` in section 12 for a first pass |
| Results differ slightly between runs | Different library versions can shift results a little | Compare against the intervals: a change well inside them is noise |
| Variables are missing after Colab restarts | Restarting clears memory | Choose **Runtime, then Run all** again |

## 10. Testing and reproducibility

- **Random seeds are fixed** (42) for cross-validation splits, the forest, boosting, and the bootstrap.
- **The pieces were tested in isolation.** The error handling passes 21 checks (missing, empty, wrong-format, repeated-column, corrupted-target, single-class, and combined-problem inputs). The statistical formulas were verified numerically against `scipy` and `pandas`. The hand-built logistic regression used for inference was checked against an independent optimizer, against the numerical second derivative of its likelihood, and by simulation (95 percent intervals covered the true value 93 to 97 percent of the time).
- **The notebook audits itself.** Section 25 lists every expected file and checks each requirement of the brief against live evidence from the run. It also re-runs the error handling on deliberately bad input.
- **Versions.** The notebook was run end to end on Python 3.12 with pandas 3.0, numpy 2.4, scipy 1.17, scikit-learn 1.8, matplotlib 3.10, and seaborn 0.13. The author's environment used Python 3.11. The minimum versions in `requirements.txt` are conservative floors that were not each tested.

## 11. Limitations

- **373 wells is small.** About 68 are above the default threshold. Wide intervals are the honest result, not a flaw to fix by reporting a single number.
- **Association, not cause.** A predictor that flags wells is not necessarily what puts arsenic in them.
- **Not a random sample.** The wells are a USGS testing set, so exceedance rates here are not statewide rates.
- **Water, not people.** It describes arsenic in a well, not any person's exposure or health. A county-level comparison with cancer rates would be ecological and cannot show individual harm.
- **Timing.** These wells were sampled in 2004 to 2006, and bladder cancer typically appears decades after exposure.
- **Several geology codes are low confidence.** The data documentation table says which, and what to check in the dataset's own metadata (`Testing_data.xml`).
- **Possible later work, not built.** A hierarchical model that shrinks small-group estimates toward the state average, a continuous statewide risk surface, and an interactive predictor for a new well (section 24). The notebook and the companion website are the finished deliverable.

## 12. Repository layout

```
Final_Project/
├── README.md
├── A_NH_Paradox_Arsenic_Testing_Extension_FP.ipynb
├── requirements.txt
├── Notebook_Concepts_Guide.pdf
├── .gitignore            <- contains: data_raw/  outputs/  __pycache__/  .ipynb_checkpoints/
├── data_raw/             <- created on run, not tracked
└── outputs/              <- created on run, not tracked (files, and figures/)
```

## 13. Data source and citations

- **Dataset.** Lombard, M.A., Hayes, Laura, Andy, C.M., Fahnestock, M.F., Bryce, J.G., and Ayotte, J.D., 2017, *Testing data set for independent analysis of New Hampshire arsenic model*: U.S. Geological Survey data release, <https://doi.org/10.5066/F7XK8CQV>. CC0 (public domain).
- **Published benchmark.** Ayotte, J.D., et al., logistic regression models for the probability of arsenic in New Hampshire bedrock wells at 1, 5, and 10 µg/L (reported accuracies of 54.8, 76.3, and 86.4 percent). Used for context only: those figures come from a different evaluation than the out-of-fold scores here, so they are not a like-for-like comparison.
- **Epidemiology.** Karagas, M.R., et al., *Elevated Bladder Cancer in Northern New England: The Role of Drinking Water and Arsenic.*
- **County boundaries.** U.S. Census Bureau TIGER/Line cartographic boundary files (downloaded when the maps are drawn).
- **Color palette.** `bi_pal_bladder` and the gray/amber scheme are reused from the original *A New Hampshire Paradox* Shiny application.
