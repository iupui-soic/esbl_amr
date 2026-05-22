# Transfer Learning for Cross-Site ESBL Prediction from Electronic Health Records

*Pipeline for bidirectional cross-site transfer learning of ESBL risk prediction across two US academic medical centers (Mass General Brigham, Stanford Health Care).*

This repository contains the analysis notebook, figures, and reproduction instructions for the npj Digital Medicine submission:

**Bidirectional cross-site transfer learning for prediction of extended-spectrum beta-lactamase-producing Enterobacterales from electronic health records**
Rashmita Kudamala¹, Aravind V. Kuruvikkattil¹, Judy W. Gichoya², Saptarshi Purkayastha¹

¹ Department of Biomedical Engineering and Informatics, Indiana University, Indianapolis, IN, USA
² Department of Radiology and Imaging Sciences, Emory University, Atlanta, GA, USA

---

## Contents

- [Overview](#overview)
- [Key Findings](#key-findings)
- [What this repository is and is not](#what-this-repository-is-and-is-not)
- [Dataset](#dataset)
- [Features extracted](#features-extracted)
- [ESBL outcome definition](#esbl-outcome-definition)
- [Repository structure](#repository-structure)
- [Environment setup](#environment-setup)
- [Reproducing results](#reproducing-results)
- [License and citation](#license-and-citation)
- [Contact](#contact)

---

## Overview

This project investigates whether EHR-based models for predicting ESBL-producing Enterobacterales can be transferred across hospital systems without requiring full local retraining.

Using the Antibiotic Resistance Microbiology Dataset (ARMD) from two US academic medical centers, we:

1. Engineer 48 harmonized clinical features from electronic health records at both sites.
2. Train and evaluate seven model architectures in true bidirectional external validation at natural ESBL prevalence.
3. Analyze fine-tuning data efficiency — how many target-site cultures are needed for effective deployment.
4. Test whether domain-adversarial alignment improves or harms cross-site transfer.
5. Characterize model performance across clinically relevant patient subgroups.

The repository provides the complete modeling pipeline needed to reproduce all reported results and figures.

---

## Key Findings

In a bidirectional external validation study using 133,084 MGB cultures (14.4% ESBL-positive) and 83,373 Stanford cultures (10.4% ESBL-positive), a pre-trained FT-Transformer fine-tuned on only 500 target-site cultures outperformed XGBoost retrained from scratch at the same 500-sample budget by +0.026 AUROC. Five hundred cultures correspond to roughly 2–3 weeks of Enterobacterales collection at a medium-volume clinical microbiology lab. At full target-site data, XGBoost matched or slightly exceeded neural models (e.g. Stanford→MGB ALL: XGBoost 0.803 vs FTT 0.796), so the advantage of pre-trained neural transfer is one of *data efficiency* rather than higher discrimination at saturation — most valuable when target-site labels are scarce.

Domain-adversarial training (FTT-DANN) was counterproductive: gradient-reversal-layer instability produced AUROC ranging 0.67–0.72 across identical zero-shot runs, with no consistent advantage over standard FTT at any fine-tuning budget. This reflects a fundamental mismatch between domain-adaptation assumptions and AMR biology — the site effect in resistance prediction is clinical signal, not noise, and erasing it harms performance.

Seven of ten top permutation-importance features were consistent across both transfer directions, supporting a minimal shared feature set for deployment. A 10-feature model preserved Stanford→MGB performance (ΔAUROC −0.011). At 90% sensitivity, negative predictive value exceeded 94% in both directions, supporting empiric carbapenem de-escalation decisions for screen-negative patients.

---

## What this repository is and is not

**This repository contains:**
- The complete analysis notebook (with cell outputs preserved).
- All main and supplementary figures.
- Configuration and reproduction instructions.

**This repository does NOT contain:**
- Patient-level data (both ARMD datasets are credentialed access).
- Trained model weights or checkpoints.
- A deployment-ready API or clinical decision support system.

For the underlying datasets see [Dataset](#dataset) below. The work is intended for research reproduction and methodological reference; any clinical deployment would require local retraining/fine-tuning, post-hoc calibration, prospective validation, and institutional governance not addressed here.

---

## Dataset

The original study was conducted using the Antibiotic Resistance Microbiology Dataset (ARMD):

- **ARMD-MGB** (Mass General Brigham) — PhysioNet, credentialed access required.
  https://physionet.org/content/armd-mgb/
- **ARMD-Stanford** (Stanford Health Care) — Dryad, credentialed access required.
  https://doi.org/10.5061/dryad.jq2bvq8kp

Both datasets are fully de-identified and require a completed data use agreement before download. No patient-level data are included in this repository, and all analyses must be executed within a secure environment under the respective ARMD DUAs.

Researchers wishing to reproduce the analysis should obtain access to both ARMD datasets and update the data paths at the top of the notebook:

```python
MGB_DIR      = "/path/to/armd-mgb"
STANFORD_DIR = "/path/to/armd-stanford"
```

---

## Features extracted

48 harmonized features were constructed across eight clinical domains from the ARMD EHR tables. The complete feature dictionary appears in Supplementary Table 1 of the manuscript.

**Demographics** — age group, sex, Area Deprivation Index score, ADI missing flag, ADI high flag.

**Comorbidities** — heart failure, liver disease, lymphoma, metastatic cancer, obesity, renal failure, Elixhauser comorbidity count.

**Ward/setting** — inpatient, outpatient, emergency department.

**Prior antibiotics (90-day window)** — fluoroquinolone, 3rd-generation cephalosporin, carbapenem, glycopeptide, sulfonamide, extended-spectrum penicillin, aminoglycoside exposure.

**Prior resistance history** — prior ESBL-positive culture, prior ESBL on AST, prior carbapenem resistance, number of distinct prior organisms, days since prior organism, number of distinct resistant antibiotic classes.

**Procedures (30-day window)** — central venous catheter, mechanical ventilation, surgical procedure, any-procedure flag.

**Culture / organism** — specimen type (urine / blood / respiratory), organism genus (*E. coli*, *Klebsiella*, *Enterobacter*, *Proteus*, other).

**Temporal / interaction** — days since last culture, prior-culture flag, concurrent antibiotic exposures, immunocompromised flag, invasive-procedure flag, prior ESBL with cephalosporin exposure, hospital multi-antibiotic exposure.

---

## ESBL outcome definition

ESBL-positive status was defined as phenotypic resistance to at least one third-generation cephalosporin or extended-spectrum penicillin on antibiotic susceptibility testing (AST), consistent with CLSI 2022 breakpoints. At MGB, ESBL designation used AST codes {CRO, CAZ, FEP, TZP} with `CLSI_2022_pheno = "Resistant"`. At Stanford, resistance was identified from the susceptibility field of the ARMD microbial resistance file. Natural prevalence was preserved at evaluation (MGB 14.4%, Stanford 10.4%); no resampling was applied to test sets.

---

## Repository structure

```
esbl_amr/
│
├── README.md
├── LICENSE
├── requirements.txt
├── cross_site_esbl.ipynb          # Main analysis notebook (with outputs)
│
└── figures/
    ├── fig1a_mgb_to_stanford.png
    ├── fig1b_stanford_to_mgb.png
    ├── multimodel_transfer.png
    ├── ftt_importance_m2s.png
    ├── ftt_importance_s2m.png
    ├── suppfig1_xgb_vs_ftt_importance.png
    ├── suppfig2_calibration.png
    ├── suppfig3_subgroups.png
    ├── suppfig4a_10feat_m2s.png
    └── suppfig4b_10feat_s2m.png
```

The notebook is committed with all cell outputs intact (executed on an RTX 3090, 24 GB VRAM) and contains:

- MGB and Stanford feature engineering (48 harmonized features, 8 clinical domains).
- Bidirectional zero-shot external validation for all 7 models at natural prevalence.
- Multi-model fine-tuning sweep (*n* = 0, 500, 1K, 2K, 5K, ALL) with source-site forgetting analysis.
- FTT-DANN domain-adversarial training and variance characterization. (See Methods §Training Protocol in the manuscript for the rationale on running the DANN fine-tuning sweep in the MGB→Stanford direction only.)
- TabPFN v2.6 zero-shot and context-window data efficiency sweep.
- FTT permutation importance (bidirectional, 5 repeats per feature).
- Direction-specific 10-feature experiments.
- Subgroup analysis (age, sex, ADI, prior ESBL history).
- Clinical operating point at 90% sensitivity (bidirectional).
- All main and supplementary figures.

Notebook outputs are included to support reproducibility verification without requiring dataset access.

---

## Environment setup

**System requirements**
- Python 3.9 or newer.
- PyTorch 2.0+ with CUDA support.
- NVIDIA GPU recommended. Experiments were run on an RTX 3090 (24 GB VRAM); full notebook execution takes approximately 6–8 hours end-to-end on this hardware. CPU-only execution is possible but is not recommended (estimated > 48 h, dominated by FTT fine-tuning sweeps and TabPFN inference).

**Installation**

```bash
git clone https://github.com/iupui-soic/esbl_amr.git
cd esbl_amr
pip install -r requirements.txt
```

A minimal pinned `requirements.txt`:

```
torch==2.3.*
scikit-learn==1.4.*
xgboost==2.0.*
optuna==3.6.*
tabpfn==2.6.*
pandas==2.2.*
numpy==1.26.*
matplotlib==3.8.*
seaborn==0.13.*
jupyter==1.0.*
```

CUDA 12.1 was used for the reference run; adjust the `torch` extra-index accordingly for other CUDA versions.

**TabPFN authentication token**

TabPFN v2.6 requires a free authentication token from https://ux.priorlabs.ai. Set it as an environment variable before launching Jupyter:

```bash
export TABPFN_TOKEN="YOUR_TABPFN_TOKEN_HERE"
```

or within the notebook:

```python
import os
os.environ["TABPFN_TOKEN"] = "YOUR_TABPFN_TOKEN_HERE"
```

---

## Reproducing results

All results are reproducible from a single notebook execution with `SEED = 42`:

```bash
# Interactive
jupyter notebook cross_site_esbl.ipynb

# Non-interactive (full pipeline)
jupyter nbconvert --to notebook --execute cross_site_esbl.ipynb --output cross_site_esbl_executed.ipynb
```

**Sanity check.** After the cross-site zero-shot block (the second results section in the notebook), the printed AUROC summary table should match Table 2 of the manuscript to within ±0.005 AUROC for deterministic models (LR, XGBoost, TabPFN) and within ±0.011 AUROC for the neural models (FTT, MHCA-VAE, DA-VAE). FTT-DANN zero-shot is the documented exception and is expected to vary by up to ±0.04 AUROC across runs.

**Key design decisions** (documented in the notebook):
- Patient-level splits (`GroupShuffleSplit`) prevent data leakage across training and test sets.
- `StandardScaler` is fit on source training data only — never on target data.
- RNG state is reset before each major experiment block.

---

## License and citation

**License.** Code in this repository is released under the MIT License (see `LICENSE`). Figures are released under CC-BY-4.0. The ARMD datasets are governed by their respective data use agreements and are *not* covered by this license.

**Citing the paper:**

> Kudamala, R., Kuruvikkattil, A.V., Gichoya, J.W. & Purkayastha, S. Bidirectional cross-site transfer learning for prediction of extended-spectrum beta-lactamase-producing Enterobacterales from electronic health records. *npj Digit. Med.* (under review, 2026).

A DOI will be added here on acceptance.

**Citing the code (separate from the paper).** A versioned software release will be archived on Zenodo at acceptance and the DOI listed here. Until then, please cite this repository by its GitHub URL and commit hash:

> Kudamala, R. *et al.* `esbl_amr` (version 1.0.0). GitHub. https://github.com/iupui-soic/esbl_amr

Reporting follows the [TRIPOD+AI](https://doi.org/10.1136/bmj-2023-078378) statement.
