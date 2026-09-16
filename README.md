# Transfer Learning for Cross-Site ESBL Prediction from Electronic Health Records

*Bidirectional cross-site transfer learning of presumptive-ESBL risk prediction across two US academic medical centers (Mass General Brigham and Stanford Health Care), using the public Antibiotic Resistance Microbiology Dataset (ARMD).*

This repository contains the analysis notebook, generated figures and reproduction instructions for:

**Bidirectional cross-site transfer learning for prediction of extended-spectrum beta-lactamase-producing Enterobacterales from electronic health records**
Rashmita Kudamala¹, Aravind V. Kuruvikkattil¹, Judy W. Gichoya², Saptarshi Purkayastha¹

¹ Department of Biomedical Engineering and Informatics, Indiana University, Indianapolis, IN, USA
² Department of Radiology and Imaging Sciences, Emory University, Atlanta, GA, USA

---

## Contents

- [Overview](#overview)
- [Key findings](#key-findings)
- [What this repository is and is not](#what-this-repository-is-and-is-not)
- [Dataset](#dataset)
- [Outcome definition](#outcome-definition)
- [Features](#features)
- [Analysis design](#analysis-design)
- [Repository structure](#repository-structure)
- [Environment setup](#environment-setup)
- [Reproducing results](#reproducing-results)
- [License and citation](#license-and-citation)

---

## Overview

Can an EHR-based model for presumptive ESBL-producing Enterobacterales, trained at one hospital system, be used at another with little or no local labelled data? Using ARMD-MGB and ARMD-Stanford we

1. build one harmonised 46-feature representation with the same code at both sites;
2. evaluate seven architectures (L1 logistic regression, XGBoost, FT-Transformer, MHCA-VAE, DA-VAE, FTT-DANN, TabPFN 2.6) in bidirectional zero-shot external validation at natural prevalence;
3. measure fine-tuning data efficiency over repeated random target subsamples of 500 to 5,000 cultures, against XGBoost trained from scratch or warm-started from the source model;
4. test whether domain-adversarial alignment helps or hurts, including its seed-to-seed stability;
5. characterise calibration, subgroup performance, a clinical operating point and several sensitivity analyses, with patient-cluster bootstrap confidence intervals throughout.

Everything reported in the manuscript is produced by a single notebook executed end to end.

---

## Key findings

In a bidirectional external validation on 120,742 MGB cultures (13.8% presumptive ESBL by CLSI screening criteria) and 76,244 Stanford cultures (9.2%), with 46 features harmonised to what is known at the preliminary culture report:

- **Zero-shot transfer is nearly architecture-independent.** AUROC spanned 0.710–0.740 (MGB→Stanford) and 0.741–0.752 (Stanford→MGB) with overlapping patient-cluster bootstrap intervals; L1 logistic regression was within 0.022 and 0.006 of the best model.
- **Data efficiency comes from pre-training, not from the architecture.** Fine-tuned on 500 target cultures, the FT-Transformer beat XGBoost trained from scratch by +0.113 [+0.100, +0.126] and +0.054 [+0.046, +0.060] AUROC (ten random subsamples), but a warm-started XGBoost by only +0.006 in either direction. With the full target cohort, locally trained XGBoost matched the neural models in one direction and came within 0.014 in the other.
- **Domain-adversarial alignment did not help** and showed larger seed-to-seed spread (FTT-DANN 0.712–0.742 across five seeds in MGB→Stanford versus 0.721–0.743 for the plain FT-Transformer).
- **Calibration differences vanish when models are calibrated alike:** after one isotonic step fitted on source data all seven models reach the same Brier score; only TabPFN (natural-prevalence context) is calibrated without post-processing.
- **Prior microbiology is the transferable signal.** On first cultures (no in-system history) AUROC drops to 0.59–0.64; source-selected 10-feature models lose 0.02–0.03 AUROC zero-shot and the 5 cross-direction stable features alone lose 0.04.
- **Operating point:** a source-derived 90%-sensitivity threshold realised sensitivity 82.1% / 90.8% and NPV 95.9% / 94.7% (no-model NPV 90.8% / 86.2%); a screening characteristic requiring prospective validation, not evidence for de-escalation.

---

## What this repository is and is not

**This repository contains**
- `cross_site_esbl_prediction.ipynb`: the complete analysis, with cell outputs from the reported run.
- `figures/`: every main and supplementary figure of the manuscript, as written by the notebook.
- `requirements.txt`: the pinned package versions of the reported run.

**This repository does not contain**
- Any patient data. ARMD must be obtained from PhysioNet (MGB) and Dryad (Stanford) under their data use agreements.
- Trained model weights or predictions (they are derived from the data).

---

## Dataset

| Site | Source | Access |
|---|---|---|
| ARMD-MGB (Mass General Brigham) | https://physionet.org/content/armd-mgb/ | PhysioNet credentialed access |
| ARMD-Stanford (Stanford Health Care) | https://doi.org/10.5061/dryad.jq2bvq8kp | Dryad, data use agreement |

Set the two directories at the top of the notebook (`MGB_DIR`, `STANFORD_DIR`). The Stanford comorbidity table (about 20 GB as CSV) is read from a parquet mirror; the notebook expects `<STANFORD_DIR>/parquet/comorbidity/*.parquet` with the same columns as the CSV (convert once with pyarrow).

Cohort: adult cultures (urine, blood, respiratory) with AST results in which at least one isolate is *E. coli*, *K. pneumoniae*, *K. oxytoca* or *P. mirabilis*. MGB 120,742 cultures from 67,185 patients; Stanford 76,244 cultures from 47,082 patients.

---

## Outcome definition

Presumptive ESBL per CLSI M100 screening criteria, assigned at the isolate level: a culture is positive if any CLSI-validated isolate is Intermediate or Resistant to ceftriaxone, ceftazidime, aztreonam or cefpodoxime. Cefepime is not a CLSI screening agent and is not used. Cefotaxime is excluded because ARMD-MGB reports it with a "≤2 µg/mL" panel floor that the CLSI-2022 re-interpretation maps to Intermediate. At MGB, non-susceptibility is read from the CLSI-2022 re-interpretation supplied with ARMD wherever a measured value exists; where the laboratory reported only a category with no measured value (the CLSI-2022 field is blank), the laboratory-reported category is used, since every breakpoint revision for these agents lowered the thresholds and a laboratory call of Intermediate or Resistant is therefore also non-susceptible under the 2022 criteria. At Stanford only laboratory-reported categories exist. Prevalence: 13.8% (MGB), 9.2% (Stanford). Against the ESBL confirmatory test recorded for 8,166 MGB cultures the definition has PPV 0.887 and sensitivity 0.975.

---

## Features

46 features in eight domains (demographics, Elixhauser comorbidities, ward, prior antibiotic exposure in 90 days, prior microbiology, procedures in 30 days, specimen and organism, temporal/interaction terms). Prior-microbiology features come from a patient-level self-join of each site's ARMD culture table (all urine, blood and respiratory cultures, positive and negative) and count only results from cultures ordered 3 to 730 days before the index culture. Note that ARMD-Stanford stores the literal string `Null` in the organism field of negative cultures; the code treats it as no organism, so negative cultures contribute to the existence and timing of prior cultures but not to prior organism counts or prior susceptibility results. Definitions are listed in Supplementary Table S1 of the manuscript and in `scripts/nb_cells/03_constants.py` and `04_builders.py`.

---

## Analysis design

- Each site is split once by patient into 64% training, 8% model selection, 8% calibration and 20% held-out test.
- Zero-shot: train on the source training split, evaluate on the whole target site.
- Transfer sweep: fine-tune on n ∈ {500, 1000, 2000, 5000} target cultures drawn from the target training split, ten random draws per budget, three seeds at the full budget; XGBoost from scratch and warm-started on the same cultures; source forgetting on the source test split.
- Permutation importance on the source test split; top-10 and stable-feature models on identical splits.
- Patient-cluster bootstrap (2,000 replicates) for every confidence interval; paired differences use identical resamples.
- Operating threshold at 90% sensitivity chosen on the source calibration split and applied unchanged to the target.

---

## Repository structure

```
esbl_amr/
├── cross_site_esbl_prediction.ipynb   # full pipeline with outputs
├── figures/                           # manuscript figures written by the notebook
├── requirements.txt
├── LICENSE
└── README.md
```

---

## Environment setup

Python 3.9 with CUDA; the reported run used one NVIDIA RTX 3090 (24 GB), PyTorch 2.8.0 and XGBoost 2.1.4.

```bash
pip install -r requirements.txt
```

TabPFN 2.6 weights require a free PriorLabs licence token: accept the licence at https://ux.priorlabs.ai (License tab) and either export `TABPFN_TOKEN=<token>` or put `TABPFN_TOKEN=<token>` in a `.env` file next to the notebook. Without a token the notebook skips TabPFN and continues.

---

## Reproducing results

Run the notebook with papermill (recommended; the two parameters are in the first code cell):

```bash
# 10-minute end-to-end test on a 5 % patient subsample
papermill cross_site_esbl_prediction.ipynb smoke.ipynb -p SMOKE 1 -p RESUME 0

# full run (about 2.5 hours on an RTX 3090); RESUME=1 continues from checkpoints
papermill cross_site_esbl_prediction.ipynb run.ipynb -p SMOKE 0 -p RESUME 1 --log-output
```

Outputs are written to `cross_site_outputs/`: `results/results.json` (every number in the manuscript), `results/*.tex` (table bodies), `preds/` (saved predictions) and `ckpt/` (model states and splits). Figures are written under the manuscript file names into `figures/`.

Seed 42 is used throughout and every training call is seeded from its stage, direction, model, budget and draw; GPU kernels are not bit-wise deterministic, so re-executions reproduce results to within the seed variability reported in Supplementary Table S8.

---

## License and citation

Code is released under the MIT License (see `LICENSE`). If you use this code, please cite the manuscript above.
