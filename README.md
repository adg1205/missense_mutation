# Domain-Aware ESM-2 Based Missense Mutation Pathogenicity Prediction

![Python](https://img.shields.io/badge/Python-3.9%2B-blue)
![Model](https://img.shields.io/badge/Protein%20LM-ESM--2-green)
![Task](https://img.shields.io/badge/Task-Missense%20Pathogenicity%20Prediction-purple)
![Status](https://img.shields.io/badge/Status-Research%20Prototype-lightgrey)

A lightweight, reproducible bioinformatics workflow for predicting whether human missense variants are **benign-like** or **pathogenic-like**. The project combines frozen **ESM-2 protein language model embedding-change features** with interpretable biochemical and UniProt domain-aware features, then trains classical machine learning classifiers for variant prioritization.

> **Important:** This project is for research and educational use only. It is not a clinical diagnostic tool and should not be used as a replacement for expert variant interpretation, ACMG/AMP guideline review, laboratory validation, or medical decision-making.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Key Results](#key-results)
- [Workflow](#workflow)
- [Dataset](#dataset)
- [Feature Engineering](#feature-engineering)
- [Models](#models)
- [Evaluation Strategy](#evaluation-strategy)
- [VUS / Conflicting Variant Case Study](#vus--conflicting-variant-case-study)
- [Interpretability](#interpretability)
- [Limitations](#limitations)
- [Future Work](#future-work)
- [Authors](#authors)
- [Citation / Acknowledgments](#citation--acknowledgements)

---

## Project Overview

Missense mutations change one amino acid in a protein sequence. Some substitutions are tolerated, while others disrupt folding, binding, enzymatic activity, protein stability, localization, or other biological functions. This project addresses the practical problem of predicting whether a missense variant is likely to behave as **benign-like** or **pathogenic-like**.

Instead of fine-tuning a large protein language model, the project uses **ESM-2 only as a frozen feature extractor**. It computes how much the learned protein representation changes between the wild-type and mutant local sequence windows. Those ESM-derived features are combined with interpretable biological descriptors such as BLOSUM62 score, hydropathy change, mass change, normalized protein position, and UniProt feature/domain indicators.

The final model is a **RandomForest classifier** trained on the combined ESM + bio/domain feature set.

---

## Key Results

The final proposed model was selected using a practical screening score based on accuracy, pathogenic recall, and pathogenic F1-score.

| Model | Features | Accuracy | Pathogenic Precision | Pathogenic Recall | Pathogenic F1 | ROC-AUC | Selection Score |
|---|---:|---:|---:|---:|---:|---:|---:|
| **ESM + bio/domain RandomForest** | 23 | **0.8449** | 0.8415 | **0.9560** | **0.8951** | 0.8871 | **0.8987** |
| Bio/domain-only RandomForest | 17 | 0.8310 | **0.8462** | 0.9240 | 0.8834 | **0.9000** | 0.8795 |
| ESM-only RandomForest | 6 | 0.7452 | 0.7724 | 0.8960 | 0.8296 | 0.7446 | 0.8236 |
| ESM + bio/domain Logistic Regression | 23 | 0.7147 | 0.8356 | 0.7320 | 0.7804 | 0.7908 | 0.7424 |
| Bio/domain-only Logistic Regression | 17 | 0.6787 | 0.8190 | 0.6880 | 0.7478 | 0.7409 | 0.7048 |
| ESM-only Logistic Regression | 6 | 0.6870 | 0.8408 | 0.6760 | 0.7494 | 0.7526 | 0.7041 |

### Final model behavior

On the held-out test set, the final ESM + bio/domain RandomForest model:

- Correctly identified **239 of 250 pathogenic variants**.
- Missed **11 pathogenic variants**.
- Misclassified **45 benign variants** as pathogenic-like.
- Achieved **95.60% pathogenic recall**, which is useful for screening because false negatives are more costly in a prioritization workflow.

---

## Workflow

```text
ClinVar missense variants + UniProt protein annotations
                  |
                  v
Filter target genes, parse protein changes, assign labels
                  |
                  v
Map variants to canonical UniProt sequences
and remove coordinate / amino-acid mismatches
                  |
                  v
Generate local wild-type and mutant sequence windows
                  |
      +-----------+-----------+
      |                       |
      v                       v
Frozen ESM-2 feature      Bio/domain feature
extraction                engineering
      |                       |
      +-----------+-----------+
                  |
                  v
Fuse features into a single variant-level table
                  |
                  v
Train Logistic Regression and RandomForest models
on ESM-only, bio/domain-only, and combined feature sets
                  |
                  v
Evaluate on held-out test set
                  |
                  v
Use final model to prioritize VUS/conflicting variants
```

---

## Dataset

### Data sources

| Source | Purpose |
|---|---|
| **ClinVar** | Missense variants and clinical significance labels |
| **UniProt** | Canonical protein sequences and protein feature/domain annotations |

The notebook downloads ClinVar dynamically from the NCBI ClinVar FTP endpoint and fetches UniProt records using the UniProt REST API.

### Target genes

The project uses nine human disease-associated genes:

| Gene | UniProt Accession |
|---|---|
| TP53 | P04637 |
| PTEN | P60484 |
| LDLR | P01130 |
| CFTR | P13569 |
| BRCA1 | P38398 |
| BRCA2 | P51587 |
| PAH | P00439 |
| HBB | P68871 |
| G6PD | P11413 |

### Dataset summary from the submitted run

| Dataset Component | Value |
|---|---:|
| Initially parsed target missense variants | 23,552 |
| Labeled training/test variants after filtering and verification | 1,441 |
| Benign / likely benign variants | 445 |
| Pathogenic / likely pathogenic variants | 996 |
| Genes included | 9 |
| VUS/conflicting variants used for case study | 60 |
| UniProt protein records | 9 |
| UniProt feature annotations | 3,765 |

### Gene-wise labeled distribution

| Gene | Benign | Pathogenic | Total |
|---|---:|---:|---:|
| BRCA1 | 120 | 120 | 240 |
| BRCA2 | 120 | 105 | 225 |
| CFTR | 15 | 120 | 135 |
| G6PD | 4 | 90 | 94 |
| HBB | 18 | 81 | 99 |
| LDLR | 40 | 120 | 160 |
| PAH | 1 | 120 | 121 |
| PTEN | 7 | 120 | 127 |
| TP53 | 120 | 120 | 240 |

### Label mapping

The notebook converts ClinVar clinical significance labels into a binary classification target:

| ClinVar clinical significance | Model label |
|---|---:|
| Benign / likely benign | 0 |
| Pathogenic / likely pathogenic | 1 |
| VUS, conflicting, uncertain, risk factor, association, drug response, protective, other, not provided | Excluded from training |

VUS/conflicting variants are kept separately for the prioritization case study.

---

## Feature Engineering

The final model uses **23 features**: 17 interpretable biological/domain-aware features and 6 ESM embedding-change features.

### Biological and domain-aware features

| Feature Type | Features |
|---|---|
| Protein position/context | `norm_position`, `protein_length`, `window_length` |
| Substitution severity | `blosum62` |
| Hydropathy change | `hydropathy_delta`, `abs_hydropathy_delta` |
| Mass change | `mass_delta`, `abs_mass_delta` |
| Charge and amino-acid flags | `charge_changed`, `to_proline`, `from_glycine`, `to_glycine` |
| UniProt annotation flags | `in_any_uniprot_feature`, `in_domain_like_feature`, `in_active_or_binding_feature`, `in_membrane_feature`, `uniprot_feature_count` |

### ESM-2 embedding-change features

The notebook generates local wild-type and mutant sequence windows around each mutation, runs both through ESM-2, and computes representation changes.

| Feature | Meaning |
|---|---|
| `esm_mean_cosine_distance` | Cosine distance between mean-pooled wild-type and mutant window embeddings |
| `esm_mean_l2_distance` | L2 distance between mean-pooled wild-type and mutant window embeddings |
| `esm_site_cosine_distance` | Cosine distance between mutation-site embeddings |
| `esm_site_l2_distance` | L2 distance between mutation-site embeddings |
| `esm_site_abs_delta_mean` | Mean absolute embedding change at the mutation site |
| `esm_mean_abs_delta_mean` | Mean absolute embedding change at the window level |

### Sequence windowing

To keep ESM-2 inference feasible in free Google Colab GPU environments, long proteins are not passed into the model as full sequences. Instead, the notebook creates mutation-centered windows:

- `WINDOW_RADIUS = 255`
- Maximum local sequence length: about 511 amino acids
- ESM model: `facebook/esm2_t12_35M_UR50D`
- Fallback model suggested in code: `facebook/esm2_t6_8M_UR50D`
- Batch size used: `BATCH_SIZE = 8`

---

## Models

The project compares three feature settings across two model families:

| Model Setup | Input Features | Purpose |
|---|---:|---|
| ESM-only + Logistic Regression | 6 | Linear baseline for protein language model features |
| ESM-only + RandomForest | 6 | Nonlinear baseline for protein language model features |
| Bio/domain-only + Logistic Regression | 17 | Linear baseline for interpretable features |
| Bio/domain-only + RandomForest | 17 | Strong non-ESM baseline |
| ESM + bio/domain + Logistic Regression | 23 | Combined linear model |
| ESM + bio/domain + RandomForest | 23 | Final proposed nonlinear model |

---

## Evaluation Strategy

The labeled dataset is split into training and test sets using:

```python
train_test_split(
    test_size=0.25,
    stratify=y,
    random_state=42,
)
```

Evaluation metrics:

- Accuracy
- Pathogenic precision
- Pathogenic recall
- Pathogenic F1-score
- ROC-AUC
- Practical selection score

The practical selection score is defined as:

```text
selection_score = mean(accuracy, pathogenic_recall, pathogenic_F1)
```

This score was used because the project is designed as a screening/prioritization workflow where catching pathogenic-like variants is especially important.

---

## VUS / Conflicting Variant Case Study

After supervised training, the final model was applied to 60 variants of uncertain significance or conflicting clinical significance. These were **not used as training labels**.

Summary from the submitted run:

| Prediction Type | Count |
|---|---:|
| Predicted pathogenic-like | 16 |
| Predicted benign-like | 44 |

Top prioritized VUS/conflicting variants:

| Gene | Variant | Predicted Pathogenic Probability | Predicted Class | Domain-like Feature? |
|---|---|---:|---|---:|
| PTEN | p.Leu100Pro | 0.9657 | Predicted pathogenic-like | 1 |
| PAH | p.Lys398Asn | 0.9210 | Predicted pathogenic-like | 0 |
| LDLR | p.Asp492Tyr | 0.8847 | Predicted pathogenic-like | 1 |
| CFTR | p.Ile1230Ser | 0.8631 | Predicted pathogenic-like | 1 |
| CFTR | p.Gln30Pro | 0.8127 | Predicted pathogenic-like | 0 |
| CFTR | p.Tyr919Asn | 0.7988 | Predicted pathogenic-like | 1 |
| G6PD | p.Asp313Asn | 0.7310 | Predicted pathogenic-like | 0 |
| BRCA2 | p.Pro65Ser | 0.7304 | Predicted pathogenic-like | 1 |
| BRCA1 | p.Phe1695Leu | 0.6622 | Predicted pathogenic-like | 1 |
| BRCA2 | p.Ile2615Asn | 0.6538 | Predicted pathogenic-like | 1 |

These predictions should be interpreted only as computational prioritization suggestions.

---

## Interpretability

RandomForest feature importance showed that the final model used a mixture of protein context, ESM embedding-change, and biochemical/domain-aware signals.

Top features from the final model:

| Feature | Importance |
|---|---:|
| `protein_length` | 0.1163 |
| `window_length` | 0.0971 |
| `norm_position` | 0.0681 |
| `esm_site_cosine_distance` | 0.0638 |
| `esm_mean_cosine_distance` | 0.0628 |
| `esm_site_abs_delta_mean` | 0.0627 |
| `blosum62` | 0.0586 |
| `uniprot_feature_count` | 0.0568 |
| `esm_mean_l2_distance` | 0.0563 |
| `esm_site_l2_distance` | 0.0561 |
| `abs_hydropathy_delta` | 0.0556 |
| `esm_mean_abs_delta_mean` | 0.0529 |

This supports the main conclusion: **ESM-2 features are most useful when combined with interpretable biological and domain-aware context.**

---

## Limitations

- The dataset is imbalanced, with more pathogenic than benign variants.
- Some genes are highly skewed toward pathogenic examples, especially PAH, PTEN, G6PD, and CFTR.
- `protein_length` and `window_length` are highly important features, so the model may partially learn gene identity rather than only mutation effect.
- The test set comes from the same gene panel used during training.
- A stricter validation would use leave-one-gene-out or completely unseen-gene evaluation.
- Local sequence windows make inference feasible but may miss long-range structural/domain interactions.
- The model predicts pathogenic-like probability, not clinical truth.
- ClinVar and UniProt are updated over time, so rerunning the notebook later may produce slightly different counts and results.

---

## Future Work

Recommended next steps:

- Expand to more genes and a larger, more balanced dataset.
- Add leave-one-gene-out validation.
- Test on external genes not seen during training.
- Compare against established tools such as SIFT, PolyPhen-2, CADD, REVEL, EVE, and AlphaMissense.
- Add AlphaFold or other structure-derived features.
- Calibrate prediction thresholds for different use cases, such as high-recall screening or high-precision prioritization.

---

## Troubleshooting

### CUDA out-of-memory error

Try one or more of the following:

```python
BATCH_SIZE = 2
MODEL_NAME = "facebook/esm2_t6_8M_UR50D"
```

You can also reduce `WINDOW_RADIUS` to shorten sequence windows.

### Slow ESM feature extraction

Use a GPU runtime. The submitted notebook was run in Google Colab with a Tesla T4 GPU.

### Hugging Face download warnings

The notebook can load the ESM-2 model without authentication, but Hugging Face may rate-limit unauthenticated requests. Setting a Hugging Face token can improve reliability.

### Different results after rerunning

ClinVar and UniProt records may change over time. The submitted results correspond to the project run reported in the accompanying PDF.

---

## Authors

- **Anindya Dasgupta** — 22201062
- **Arpita Sarkar** — 22201128

Course project: **CSE443**

---

## Citation / Acknowledgments

This project uses public biological resources and open-source software, including ClinVar, UniProt, Hugging Face Transformers, ESM-2, Biopython, scikit-learn, pandas, NumPy, PyTorch, Matplotlib, and tqdm.
