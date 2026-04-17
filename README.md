# Transfer Learning for Cross-Site ESBL Prediction from Electronic Health Records

This repository contains the analysis notebook and figures for the npj Digital Medicine submission:

**Transfer learning for ESBL prediction across hospitals using electronic health records**  
Rashmita Kudamala, Aravind Kuruvikkattil Venugopalan, Saptarshi Purkayastha

---

## Overview

This project investigates whether EHR-based models for predicting ESBL-producing Enterobacteriaceae can be transferred across hospital systems without requiring full local retraining.

Using the Antibiotic Resistance Microbiology Dataset (ARMD) from two US academic medical centers, we:

1. Engineer 48 harmonized clinical features from electronic health records at both sites
2. Train and evaluate seven model architectures in true bidirectional external validation at natural ESBL prevalence
3. Analyze fine-tuning data efficiency - how many target-site cultures are needed for effective deployment
4. Test whether domain-adversarial alignment improves or harms cross-site transfer
5. Characterize model performance across clinically relevant patient subgroups

The repository provides the complete modeling pipeline needed to reproduce all reported results and figures.

---

## Key Findings

In a bidirectional external validation study using 133,084 MGB cultures (14.4% ESBL-positive) and 83,373 Stanford cultures (10.4% ESBL-positive), a pre-trained FT-Transformer fine-tuned on only 500 target-site cultures consistently outperformed XGBoost trained from scratch on full target data (+0.026 AUROC). This represents approximately 2–3 weeks of culture collection at a medium-volume hospital, dramatically lowering the barrier for cross-site deployment.

Domain-adversarial training (FTT-DANN) was counterproductive: gradient reversal layer instability produced AUROC ranging 0.67–0.72 across identical runs at zero-shot, with no consistent advantage over standard FTT at any fine-tuning budget. This reflects a fundamental mismatch between domain adaptation assumptions and AMR biology — the site effect in resistance prediction is clinical signal, not noise, and erasing it harms performance.

Seven of ten top permutation-importance features were consistent across both transfer directions, supporting a minimal shared feature set for deployment. A 10-feature model preserved Stanford→MGB performance (ΔAUROC −0.011). At 90% sensitivity, negative predictive value exceeded 94% in both directions, supporting empiric carbapenem de-escalation decisions.

---

## Dataset

The original study was conducted using the Antibiotic Resistance Microbiology Dataset (ARMD):

- **ARMD-MGB**: Available via PhysioNet (credentialed access required)  
  https://physionet.org/content/armd-mgb/
- **ARMD-Stanford**: Available via Dryad  
  https://doi.org/10.5061/dryad.jq2bvq8kp

Both datasets require credentialed access and a completed data use agreement. No patient-level data are included in this repository.

Researchers wishing to reproduce the analysis should obtain access to both ARMD datasets and update the data paths at the top of the notebook:

```python
MGB_DIR      = "/path/to/armd-mgb"
STANFORD_DIR = "/path/to/armd-stanford"
```

---

## Features Extracted

48 harmonized features were constructed across eight clinical domains from the ARMD EHR tables.

**Demographics** — age group, sex, Area Deprivation Index score, ADI missing flag, ADI high flag

**Comorbidities** — heart failure, liver disease, lymphoma, metastatic cancer, obesity, renal failure, Elixhauser comorbidity count

**Ward/Setting** — inpatient, outpatient, emergency department

**Prior Antibiotics (90-day window)** — fluoroquinolone, 3rd-generation cephalosporin, carbapenem, glycopeptide, sulfonamide, extended-spectrum penicillin, aminoglycoside exposure

**Prior Resistance History** — prior ESBL-positive culture, prior ESBL on AST, prior carbapenem resistance, number of distinct prior organisms, days since prior organism, number of distinct resistant antibiotic classes

**Procedures (30-day window)** — central venous catheter, mechanical ventilation, surgical procedure, any procedure flag

**Culture/Organism** — specimen type (urine/blood/respiratory), organism genus (*E. coli*, *Klebsiella*, *Enterobacter*, *Proteus*, other)

**Temporal/Interaction** — days since last culture, prior culture flag, concurrent antibiotic exposures, immunocompromised flag, invasive procedure flag, prior ESBL with cephalosporin exposure, hospital multi-antibiotic exposure

---

## ESBL Outcome Definition

ESBL-positive status was defined as phenotypic resistance to at least one third-generation cephalosporin or extended-spectrum penicillin on antibiotic susceptibility testing (AST), consistent with CLSI 2022 breakpoints. At MGB, ESBL designation used AST codes {CRO, CAZ, FEP, TZP} with CLSI_2022_pheno = "Resistant." At Stanford, resistance was identified from the susceptibility field of the ARMD microbial resistance file. Natural prevalence was preserved at evaluation (MGB 14.4%, Stanford 10.4%).

---

## Repository Structure

```
esbl_amr/
│
├── README.md
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

The notebook contains:

- MGB and Stanford feature engineering (48 harmonized features, 8 clinical domains)
- Bidirectional zero-shot external validation for all 7 models at natural prevalence
- Multi-model fine-tuning sweep (n = 0, 500, 1K, 2K, 5K, ALL) with source-site forgetting
- FTT-DANN domain-adversarial training and variance characterization
- TabPFN v2.6 zero-shot and context-window data efficiency sweep
- FTT permutation importance (bidirectional, 5 repeats per feature)
- Direction-specific 10-feature experiments
- Subgroup analysis (age, sex, ADI, prior ESBL history)
- Clinical operating point at 90% sensitivity (bidirectional)
- All main and supplementary figures

Notebook outputs are included to support reproducibility verification without requiring dataset access.

---

## Environment Setup

```bash
pip install torch scikit-learn xgboost optuna tabpfn pandas numpy matplotlib seaborn
```

**Key requirements:**
- Python 3.9+
- PyTorch 2.0+ with CUDA support
- NVIDIA GPU recommended (experiments run on RTX 3090, 24 GB VRAM)

**TabPFN token:** TabPFN v2.6 requires a free authentication token from https://ux.priorlabs.ai. Set it as an environment variable before running:

```python
import os
os.environ["TABPFN_TOKEN"] = "YOUR_TABPFN_TOKEN_HERE"
```

---

## Reproducing Results

All results are reproducible from a single notebook execution with `SEED = 42`.

Key design decisions documented in the notebook:
- Patient-level splits (`GroupShuffleSplit`) prevent data leakage across training and test sets
- `StandardScaler` fit on source training data only — never on target data
- RNG state is reset before each major experiment block
- FTT-DANN fine-tuning sweep conducted for MGB→Stanford only; Stanford→MGB not performed as DANN showed no consistent advantage in either zero-shot direction

---

## Data Access Notice

This repository contains analysis code and generated figures only.  
No patient-level data or model checkpoints are included.  
All analyses must be executed within a secure environment under the respective ARMD data use agreements.

---

## Authors

Rashmita Kudamala, Aravind Kuruvikkattil Venugopalan, Saptarshi Purkayastha

---

## Citation

If you use this code or build upon this work, please cite:

Kudamala R, Kuruvikkattil Venugopalan A, Purkayastha S. Transfer learning for ESBL prediction across hospitals using electronic health records. npj Digital Medicine (under review).
