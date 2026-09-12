# ANFIS-Based Early Diagnosis of Programming Learning Performance

This repository contains the Python implementation and experimental materials for an Adaptive Neuro-Fuzzy Inference System (ANFIS) developed to support early identification of programming-learning performance using process data from the CodeWorkout Spring 2019 (S19) dataset.

The implementation is designed as a research artifact to improve transparency, traceability, and reproducibility of the computational experiments reported in the associated research study.

---

## 1. Research Objective

The study investigates whether programming-process information available at a very early stage of problem solving can be used to identify students who may require additional pedagogical support.

The prediction is performed after the student's first two compilation attempts on a programming problem. Only information available up to this prediction point is used as model input.

The prediction task is defined at the:

**Student × Problem**

level rather than as an aggregate student-level prediction.

---

## 2. Dataset

The experiments use the **CodeWorkout Spring 2019 (S19)** dataset.

The original dataset is provided in ProgSnap2 format and contains programming-learning event data recorded during students' interactions with the CodeWorkout platform.

This implementation requires the original dataset archive to contain, among others:

- `MainTable.csv`
- `CodeStates.csv`
- `early.csv`
- `late.csv`

The original dataset is **not redistributed in this repository**. Users should obtain the dataset from its original source and comply with the applicable data-access and usage conditions.

---

## 3. Temporal Prediction Framework

The experimental design follows a temporal framework:

**Early Window → Prediction Point → Outcome**

The early observation window consists of the **first two compilation attempts** for each Student–Problem pair.

The prediction point is defined immediately after the second compilation attempt.

Information occurring after the prediction point is not used to construct the predictor variables.

This temporal separation is intended to support an early-diagnosis setting and to reduce the possibility of using future process information as predictors.

---

## 4. Input Variables

The current implementation uses **five input variables (X1–X5)**.

| Code | Variable | Operational Definition | Source |
|------|----------|------------------------|--------|
| X1 | `early_error_rate` | Proportion of compilation failures among the first two compilation attempts | `MainTable.csv` |
| X2 | `early_time_gap_sec` | Log-transformed time gap between the first and second compilation attempts | `MainTable.csv` |
| X3 | `code_len_first` | Code length at the first compilation | `CodeStates.csv` |
| X4 | `code_len_delta` | Change in code length between the first and second compilation | `CodeStates.csv` |
| X5 | `code_similarity_1_2` | Textual similarity between the first and second code versions | `CodeStates.csv` |

The five variables are extracted only from the first two compilation attempts and are therefore intended to represent information available at the prediction point.

---

## 5. Target Variable

The prediction target is the binary `Label` variable provided in the CodeWorkout dataset.

The label distinguishes between:

- `Label = 1`: efficient success
- `Label = 0`: requires additional pedagogical attention

The label is defined using eventual completion status and the number of attempts relative to the problem-specific P75 threshold.

The implementation also contains a dedicated audit comparing the original dataset label definition with an alternative train-only P75 calculation. This audit is included to quantify the potential influence of the target-definition strategy rather than to replace the primary target used in the main experiment.

---

## 6. Final Experimental Dataset

After filtering Student–Problem pairs for which at least two compilation events are available, the implementation produces approximately:

**9,153 Student–Problem pairs**

These observations are used as the final analytical dataset for the five-variable ANFIS model.

---

## 7. Train-Test Splitting

The primary evaluation uses a **70:30 student-level split**.

Students are treated as the grouping unit so that observations from the same student do not appear in both the training and test sets.

The implementation uses `GroupShuffleSplit` with `SubjectID` as the grouping variable.

For the primary split:

- Training students: 242
- Test students: 104
- Training observations: 6,384
- Test observations: 2,769
- Student overlap between training and test sets: 0

This procedure is intended to reduce student-level information leakage and to evaluate generalization to students not observed during model training.

---

## 8. Data Normalization

Feature normalization is performed using statistics calculated from the **training data only**.

The same training-derived transformation is then applied to the test data.

This procedure prevents information from the test set from being used to determine the scaling parameters.

---

## 9. ANFIS Architecture

The main predictive model is a **first-order Sugeno ANFIS**.

The current implementation uses:

- Number of inputs: **5**
- Membership functions per input: **2**
- Membership function type: **Gaussian**
- Linguistic categories: **Low** and **High**
- Number of fuzzy rules: **2⁵ = 32 rules**

The model follows the standard five-layer ANFIS structure:

1. Fuzzification
2. Rule firing
3. Normalization
4. Consequent computation
5. Output

The consequent parameters use a first-order linear Sugeno formulation.

---

## 10. Theory-Pedagogically Grounded Initialization

The premise parameters of the ANFIS model are initialized using theory- and pedagogy-informed thresholds rather than generic random initialization.

The initialization is based on interpretable thresholds associated with the five behavioral indicators.

The initial premise parameters are subsequently adapted during training.

This design is intended to preserve meaningful linguistic conditions in the fuzzy rule structure while allowing the model to learn from empirical data.

---

## 11. Training Procedure

The ANFIS implementation uses hybrid learning consisting of:

- **Least Squares Estimation (LSE)** for consequent parameters
- **Gradient Descent** for premise parameters

The current training configuration uses:

- Epochs: 60
- Initial learning rate: 0.03
- Learning-rate decay: 0.97 per epoch
- Best-epoch parameter checkpointing based on training RMSE
- Membership-value clipping for numerical stability

The implementation returns the parameter configuration associated with the lowest training RMSE observed during training.

---

## 12. Evaluation Metrics

Because the target variable is binary, the main evaluation metrics are:

- Area Under the ROC Curve (AUC)
- Accuracy
- Precision
- Recall
- F1-score

The main discrimination metric is AUC because it does not depend on a single classification threshold.

The decision threshold used for classification is determined from the training data using Youden's J criterion and then applied to the test data.

---

## 13. Main ANFIS Result

For the primary student-level split, the current five-variable implementation produced:

| Metric | Result |
|--------|-------:|
| Train AUC | 0.7058 |
| Test AUC | 0.6708 |
| Test Accuracy | 0.6324 |
| Precision, Label 0 | 0.5588 |
| Precision, Label 1 | 0.7096 |
| Recall, Label 0 | 0.6692 |
| Recall, Label 1 | 0.6048 |
| F1, Label 0 | 0.6091 |
| F1, Label 1 | 0.6530 |

The reported values correspond to the five-variable implementation contained in this repository.

---

## 14. Robustness Evaluation

Model stability is additionally examined using **10-fold GroupKFold** validation, with `SubjectID` used as the grouping variable.

Each student is assigned to a single test fold, preventing the same student from appearing in both training and test portions of an individual fold.

For the theory-pedagogically initialized ANFIS model:

- Mean AUC: **0.6752**
- Standard deviation: **0.0234**

For the generic/random initialization:

- Mean AUC: **0.6764**
- Standard deviation: **0.0246**

The mean paired difference between the two initialization strategies is approximately **-0.0012**, with a Wilcoxon signed-rank test of:

**W = 23.0, p = 0.6953**

These results indicate that the current experiments do not provide statistically significant evidence that the theory-pedagogically initialized model has higher predictive AUC than the generic initialization under the 10-fold grouped validation procedure.

---

## 15. Interpretability Analysis

The repository includes analyses intended to examine the interpretability of the ANFIS model.

These include:

### Permutation Importance

Each input variable is permuted independently and the resulting change in AUC is measured.

The current five-variable experiment identifies **early compilation error rate (X1)** as the most influential variable in the tested configuration.

### Rule Analysis

The implementation evaluates the firing strength of the 32 fuzzy rules and identifies rules that become dominant for observed training cases.

The rule analysis is intended to make the model decision structure traceable through combinations of linguistic conditions such as Low and High.

---

## 16. Baseline Models

Two conventional machine-learning models are included for comparison:

- Logistic Regression
- Random Forest

Both models use the same five input variables and the same primary student-level train-test split as the ANFIS model.

The current implementation reports:

| Model | Train AUC | Test AUC |
|-------|----------:|---------:|
| Logistic Regression | 0.6581 | 0.6514 |
| ANFIS (Theory-Pedagogical) | 0.7058 | 0.6708 |
| Random Forest (`max_depth=10`) | 0.8425 | 0.6820 |

The Random Forest section currently includes exploratory testing of several `max_depth` values.

**Important methodological note:** the current exploratory RF tuning selects `max_depth` according to test-set AUC. Therefore, the RF comparison should be interpreted as a comparative experimental result rather than as an untouched final test-set benchmark. This repository preserves that procedure for transparency.

---

## 17. P75 Target-Definition Audit

The repository contains a separate audit examining the effect of calculating the problem-specific P75 threshold using training data rather than the original whole-dataset definition.

The audit found:

- 120 of 9,153 labels changed overall (1.31%)
- 30 of 2,683 test observations changed (1.12%)

The purpose of this analysis is to quantify the sensitivity of the target definition and document the methodological trade-off associated with problem-specific P75 estimation.

The primary experiment retains the dataset's original `Label` definition.

---

## 18. Reproducibility

The notebook is designed to support reproducible execution in Google Colab.

The experimental workflow is:

1. Upload the CodeWorkout S19 dataset archive.
2. Extract the source files.
3. Locate the required CSV files.
4. Construct the five input variables.
5. Prepare the Student–Problem analytical dataset.
6. Perform the student-level train-test split.
7. Normalize features using training data.
8. Initialize and train the ANFIS model.
9. Evaluate the model on the test set.
10. Perform robustness and interpretability analyses.
11. Train the baseline models.
12. Export experimental results.

Several result files are generated by the notebook, including:

- `dataset_anfis_5var.csv`
- `hasil_prediksi_test_5var.csv`
- `metrics_anfis_5var.json`
- `evaluation_anfis_5var.png`
- `baseline_comparison_5var.csv`
- `robustness_groupkfold_5var.csv`
- `robustness_groupkfold_summary.json`

---
