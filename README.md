# ANFIS for Early Diagnosis of Programming Learning Performance

Code for the experiments in the manuscript **"Interpretable Neuro-Fuzzy Model for Early Programming Learning Support"** (Falani, Patmanthara, Prasetya). The study uses a first-order Sugeno ANFIS to flag Student-Problem pairs that may need extra support, using only process data observed up to the second compilation, from the CodeWorkout Spring 2019 (S19) dataset.

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/zakkifalani-git/anfis-codeworkout-early-diagnosis/blob/main/anfis_codeworkout_early_diagnosis.ipynb)

> **Status.** Manuscript under review. The notebook has been reorganised to match the current manuscript version. Please re-run it from top to bottom before comparing numbers; the figures below are those reported by the authors.

---

## 1. What is in this repository

| File | Purpose |
|---|---|
| `anfis_codeworkout_early_diagnosis.ipynb` | The full pipeline (data preparation, ANFIS, baselines, validation, diagnostics). Written for Google Colab. |
| `requirements.txt` | Python packages used. |
| `LICENSE` | MIT license for the code. |
| `CITATION.cff` | Citation metadata. |

The CodeWorkout dataset is **not redistributed**. Obtain it from its original source, follow its access conditions, and upload the archive (ZIP or RAR) in Section 2 of the notebook. It must contain `MainTable.csv`, `CodeStates.csv`, `early.csv` and `late.csv`.

## 2. Task and data

- **Unit of analysis:** Student x Problem pair.
- **Prediction point:** right after the second compilation. Only information up to that point is used as input.
- **Target (`Label`):** 1 = efficient success (eventually correct and attempts at or below the problem-specific P75), 0 = needs additional support.
- **Final dataset (population A):** 9,153 pairs with at least two compile events (14,828 pairs before filtering).
- **Evaluation population (population B):** 6,989 pairs. It excludes the 2,164 pairs that are already correct at exactly the second compilation, because their outcome is partly observed at the prediction point. Population B is the **primary** evaluation set; population A is kept for comparison with earlier results.

Inputs (all from the first two compilations):

| Code | Variable | Definition | Source |
|---|---|---|---|
| X1 | `early_error_rate` | Share of failed compilations among the first two | `MainTable.csv` |
| X2 | `early_time_gap_sec` | Time gap between compilation 1 and 2 (log1p) | `MainTable.csv` |
| X3 | `code_len_first` | Code length at the first compilation | `CodeStates.csv` |
| X4 | `code_len_delta` | Change in code length between compilation 1 and 2 | `CodeStates.csv` |
| X5 | `code_similarity_1_2` | Text similarity of the two code versions | `CodeStates.csv` |

## 3. Model and protocol

- **ANFIS:** first-order Sugeno, 5 inputs, 2 Gaussian membership functions per input (Low/High), 32 rules. Hybrid learning: least squares for the consequents, gradient descent for the premises. 60 epochs, learning rate 0.03 with decay 0.97, best epoch chosen by training RMSE. Decision threshold from Youden's J on the training data.
- **Two initialisations, named as in the manuscript:**
  - *Theoretical*: initial means and sigmas set from theory- and pedagogy-based thresholds (Section 11).
  - *Fixed*: the generic fallback in `ANFIS.__init__` (means on `linspace(0.15, 0.85)`, sigma 0.25). Older versions of the notebook called this scheme "random" or "uniform"; it is not random. The seed only initialises the consequents, which the least-squares step overwrites in the first epoch, so training is deterministic for a given data split.
- **Splits:** 70/30 student-level split with `GroupShuffleSplit` (no student on both sides) and 10-fold `GroupKFold`. Min-max scaling uses training data only.
- **Baselines:** logistic regression and random forest on the same inputs.
- **Statistics:** Friedman test with Holm-corrected pairwise Wilcoxon tests over the 10 folds; bootstrap confidence intervals.

## 4. Where to find each part of the manuscript

| Notebook section | Content |
|---|---|
| 1-6 | Environment, dataset upload, extraction of X1-X5, merge with `Label` |
| 7 | Audit of the `Label` (P75) definition |
| 9-13b | ANFIS implementation, training, evaluation, bootstrap CI (population A) |
| 14-16 | Threshold sensitivity, permutation importance, dominant rules (population A) |
| 17-21 | Baselines, robustness, effect size, fixed-initialisation comparison |
| 24-28b | Visualisation, GroupKFold, Friedman and Holm, fixed-scheme verification |
| 29-30 | Prediction-point analysis; populations A and B, GroupKFold-B, bootstrap, Wilcoxon |
| 31-33 | Diagnosis of unstable ANFIS-Theoretical results; collapse analysis |
| 34 | Scenario S5 for ANFIS-Fixed |
| 35 | Tables 7, 10, 11, 12 for population B |
| 36-37 | P75 definition: key-based verification and variants V1/V2 |
| 38 | Reconciliation of filtered-data and success-rate numbers |
| 39 | Three checks: index vs key alignment (old AUC 0.471 / 0.490), the 310 vs 311 difference, audit of Sections 27-28 |
| 40 | Read-only audit of the notebook cells of Sections 27-28 for index-based alignment |

Sections 8, 22 and 23 of earlier versions aligned two tables by row index, which is unsafe, and have been replaced by the key-based Sections 36-38. The headings remain as short notes so that section numbers stay stable.

"Collapse" means that more than half of the training rows have a total rule activation below 1e-9. Sections 32-33 report how often this happens for each initialisation.

## 5. Reported results

**Population A, official 70/30 split (comparison).** Train 6,384 pairs / test 2,769 pairs (242 / 104 students).

| Model | Test AUC |
|---|---:|
| ANFIS (Theoretical) | 0.6708 (95% CI 0.6509-0.6914, pair-level bootstrap) |
| Logistic regression | 0.6514 |
| Random forest (`max_depth=10`) | 0.6820 |

The random-forest depth was chosen with test AUC in this exploratory comparison, so its figure is not an untouched benchmark.

**Population A, 10-fold GroupKFold, mean AUC (SD).** ANFIS-Theoretical 0.6752 (0.0234); ANFIS-Fixed 0.6764 (0.0246); logistic regression 0.6558 (0.0262); random forest 0.6858 (0.0195). Friedman chi-square = 22.44, p < 0.001. The two ANFIS initialisations do not differ (Holm p = 0.695); random forest is above the theoretical-init ANFIS (Holm p = 0.029) but not above the fixed-init ANFIS (Holm p = 0.168).

**Population B (primary).** Train 4,812 / test 2,177 pairs. The official ANFIS reaches a test AUC of 0.581 (95% student-level bootstrap CI 0.557-0.605). Performance is clearly lower than on population A, which shows how much of the earlier figure came from outcomes already visible at the prediction point.

**P75 definition.** Recomputing P75 from training pairs only changes 11.5% (variant V1) or 1.79% (variant V2) of the labels. The official model scores 0.646 (V1) and 0.681 (V2) against these alternative labels (Sections 36-37).

**Filtered pairs.** Of the 5,675 pairs removed (14,828 to 9,153), 5,524 (97.3%) were correct at the first attempt and 151 (2.7%) stopped after one failed attempt (Section 38).

## 6. Running the notebook

1. Open the notebook in Colab (badge above).
2. Run Sections 1-13 in order, uploading the dataset archive in Section 2.
3. Run the later sections as needed. Sections 29-40 expect Sections 1-13 to have run in the same session.
4. Sections 32 and 33 train many ANFIS models. They save a checkpoint and pause after a time budget; run the same cell again to resume. Set `CKPT_DIR` to a Google Drive folder if your session may disconnect.

All random seeds are fixed (42 unless a section says otherwise). Small numerical differences can occur with different library versions.

## 7. Limitations

- One dataset, one course term. Results may not transfer to other courses or languages.
- Population B is a post-hoc restriction motivated by partly observed outcomes; the `Label` still depends on the problem-specific P75.
- The predictive gain over simpler models is small or absent. The contribution of the ANFIS is its readable rule structure, not higher accuracy.
- Some of the 10-fold comparisons have low statistical power.

## 8. License and citation

Code: MIT License (see `LICENSE`). The CodeWorkout data follow their own terms. If you use this code, please cite the manuscript once it is published; `CITATION.cff` will be updated with the reference.

Contact: Achmad Zakki Falani (zakkifalani@gmail.com).
