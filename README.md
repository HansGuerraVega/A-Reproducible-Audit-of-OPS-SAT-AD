# A Reproducible Audit of OPS-SAT-AD: Evaluating Data Preparation Choices for Telemetry Anomaly Detection

Reproducible audit of telemetry anomaly detection on **OPS-SAT-AD v2** ([Zenodo record 15108715](https://zenodo.org/records/15108715)): data quality checks, leakage-safe preprocessing, and model comparison under a locked test set.

> **Disclaimer:** This project is a *reproducible audit* of a published dataset and pipeline. It is **not** a flight-ready anomaly-detection system, and its results should not be interpreted as such.

## Overview

OPS-SAT-AD v2 is a labeled telemetry anomaly dataset derived from ESA's OPS-SAT spacecraft. It includes raw telemetry samples (`segments.csv`) and a segment-level feature table (`dataset.csv`) generated from those samples.

This project asks: **How do temporal feature extraction and data-preparation choices affect anomaly detection on OPS-SAT-AD?** Rather than claiming a new detector, the project audits the available data and pipeline, independently reproduces selected time-related features, and compares model and preprocessing choices using a controlled evaluation protocol.

## Dataset

| File | Description | Size |
|---|---|---:|
| `segments.csv` | Raw telemetry samples, including channel, timestamp, value, sampling information, anomaly label, segment, and train/test assignment. | 303,493 samples |
| `dataset.csv` | Segment-level table containing 18 temporal/statistical features and metadata. | 2,123 segments |

The binary `anomaly` column is the modeling target. The `label` column in the raw samples is constant (`"anomaly"`) and is not used as the target.

The audit independently reproduces three time-related features—`len`, `duration`, and `gaps_squared`—for all 2,123 segments. It does **not** independently reproduce all 18 features.

Data-quality checks found no invalid timestamps, non-positive intervals, duplicate segment IDs, or null/infinite values in the 18 feature columns. However, 1,318 intervals across 151 segments differ from the declared sampling value. The causes of these differences could not be verified from the supplied files. The dataset generator uses `timedelta.seconds` for duration-related calculations; this has a latent day-wrap limitation, but it is not triggered in this dataset (maximum observed duration: 1,335 seconds).

## Methodology

1. **Data and generator audit:** Check dataset structure, timestamps, sampling intervals, segment IDs, feature values, and selected generator calculations.
2. **Partitioning:** Preserve the shipped train/test assignment (1,594 / 529). Keep the 529 test segments locked until the single final evaluation. Split the training set 80/20, stratified by `anomaly` with seed `2137`:
   - Subtrain: 1,275 segments (1,018 nominal, 257 anomalous).
   - Validation: 319 segments (255 nominal, 64 anomalous).
   - Partition disjointness is asserted using segment IDs.
3. **Preprocessing:** Compare raw features, `StandardScaler`, and `RobustScaler`. Each scaler is fitted only on the 1,018 nominal subtraining segments and then applied to the relevant data using `transform`; it is not refitted on validation or test data.
4. **Models:** Train AdaBoost on all labeled subtraining rows. Train Isolation Forest on nominal subtraining rows only; labels are used only for evaluation.
5. **Selection and final evaluation:** Select the configuration on validation using MCC first and AUC-PR second. Evaluate the selected configuration once on the locked test set. Scaler provenance is checked with assertions.

This preprocessing protocol differs from the original notebook, which fitted its scaler on all normal training segments, including the subset used as validation in this audit. The difference is documented.

## Results

### Validation comparison (319 segments)

| Model | Preprocessing | Accuracy | Precision | Recall | F1 | MCC | AUC-PR | AUC-ROC |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| AdaBoost | Raw | 0.940 | 0.895 | 0.797 | 0.843 | 0.808 | 0.916 | 0.962 |
| **AdaBoost** | **StandardScaler** | **0.940** | **0.895** | **0.797** | **0.843** | **0.808** | **0.931** | **0.967** |
| AdaBoost | RobustScaler | 0.940 | 0.895 | 0.797 | 0.843 | 0.808 | 0.916 | 0.962 |
| Isolation Forest | Raw | 0.746 | 0.333 | 0.266 | 0.296 | 0.145 | 0.328 | 0.683 |
| Isolation Forest | StandardScaler | 0.743 | 0.320 | 0.250 | 0.281 | 0.129 | 0.349 | 0.686 |
| Isolation Forest | RobustScaler | 0.746 | 0.333 | 0.266 | 0.296 | 0.145 | 0.328 | 0.683 |

The validation-selected configuration is **AdaBoost with StandardScaler**. It has the highest validation AUC-PR among the compared configurations and ties for the highest MCC.

### Locked test evaluation (529 segments; evaluated once)

| Run | Accuracy | Precision | Recall | F1 | MCC | AUC-PR | AUC-ROC |
|---|---:|---:|---:|---:|---:|---:|---:|
| Winner protocol (subtrain fit) | 0.934 | 0.906 | 0.770 | 0.833 | 0.796 | 0.924 | 0.962 |
| Full-training reference (descriptive only) | 0.928 | 0.887 | 0.761 | 0.819 | 0.778 | 0.910 | 0.951 |

The full-training reference is a **hybrid reference, not a literal reproduction**: its model is trained using the full-training recipe, while its test features use the scaler fitted on nominal subtraining data. It is reported for descriptive comparison only. Neither test result was used for model or preprocessing selection.

Validation error analysis for the selected model found 13 false negatives and 6 false positives. The false negatives were associated mainly with low-energy or flat anomalies. This is an observation from this dataset and split, not a verified explanation of the underlying physical causes.

## Repository structure

```text
.
├── data/
│   ├── segments.csv
│   └── dataset.csv
├── notebooks/
│   └── OPS_SAT_AD_reproducible_audit.ipynb
├── requirements.txt
├── CITATION.cff
├── LICENSE
└── README.md
```

If the dataset files are not included in this repository, download them from the [OPS-SAT-AD v2 Zenodo record](https://zenodo.org/records/15108715) and place them in `data/`. Confirm the dataset's redistribution terms before committing the files.

## How to reproduce

1. Clone or download this repository.
2. Create a Python environment and install the dependencies listed in `requirements.txt`:

   ```bash
   pip install -r requirements.txt
   ```

3. Open `notebooks/OPS_SAT_AD_reproducible_audit.ipynb` in Jupyter.
4. Run the notebook cells in order, using the supplied data files. The notebook contains the data checks, partitioning, preprocessing, model comparison, and final evaluation.

The random seed used for the split and model procedures is `2137`. Results may vary if software versions or execution settings differ from those used in the original run.

## Limitations

1. **Single split and no uncertainty quantification:** The comparison uses one stratified training/validation split and one locked-test evaluation. Small metric differences are descriptive and are not presented as statistically significant effects.
2. **No independent raw-signal analysis:** The audit does not establish whether the feature representation misses any physical anomaly type.
3. **Labeling and segmentation policy not assessed:** The available files do not establish the labeling procedure or resolve possible boundary ambiguity.
4. **Sampling irregularities and short segments:** Some intervals differ from the declared sampling value, and some segments contain few samples. The impact of these properties on smoothed peak features has not been independently quantified.
5. **Limited model scope:** The comparison uses default hyperparameters and a limited set of models. Channel semantics are not established, so the results are not given a physical channel-level interpretation.

## Citation

Please cite the original dataset using the citation provided in the [Zenodo record](https://zenodo.org/records/15108715). Add the exact citation from the record here before publishing this repository.

To cite this audit, replace the placeholders below with the repository author's name, year, and repository URL:

> [Author name]. ([Year]). *A Reproducible Audit of OPS-SAT-AD: Evaluating Data Preparation Choices for Telemetry Anomaly Detection.* [Repository URL].

## License

The code in this repository is intended to be released under the MIT License, as specified in the repository's `LICENSE` file. The OPS-SAT-AD dataset may have separate terms. Consult the license stated on the [Zenodo record](https://zenodo.org/records/15108715) before redistributing the data.
