# ML-Based Virtual Screening for EGFR Inhibitor Discovery

A machine learning pipeline for prioritizing small-molecule candidates against
**EGFR (Epidermal Growth Factor Receptor)**, an oncology
target with multiple approved inhibitors (e.g. Gefitinib, Erlotinib).

My goal in this project is **not** to discover a new drug. It is to build
and validate a computational pipeline that can prioritize candidate compounds
against a biological target, using established cheminformatics and machine
learning methods.

---

## Status

🚧 **In progress** — this repository is being built incrementally. See the
Pipeline Overview below for the full planned scope.

---

## Why EGFR

- Extensively studied oncology target (lung, colorectal, head and neck
  cancers)
- Abundant public bioactivity data (ChEMBL)
- Known active inhibitors already approved, providing a real-world
  reference point to sanity-check model behavior
- 3D structural data publicly available (useful for the docking phase)

---

## Repository Structure

```
egfr-drug-discovery/
├── notebooks/          # Step-by-step analysis notebooks
├── src/                # Reusable helper functions
├── data/                # Input data (not tracked — see data/README.md)
├── results/
│   └── plots/           # Output plots and figures
└── README.md
```

---

## Pipeline Overview

1. **Dataset construction** — retrieve known active/inactive compounds
   against EGFR from ChEMBL
2. **Molecular representation** — convert SMILES to Morgan fingerprints and
   molecular descriptors (RDKit)
3. **Machine learning** — train and benchmark Random Forest and XGBoost
   classifiers to distinguish active from inactive compounds
4. **Virtual screening** — apply the trained model to a larger, unseen
   compound library and rank candidates by predicted activity
5. **Candidate prioritization** — produce a ranked shortlist of top
   candidate compounds
6. **Molecular docking (planned)** — structural validation of the top
   candidates against the EGFR binding site using AutoDock Vina

---

## Dependencies

```bash
pip install rdkit pandas numpy scikit-learn xgboost matplotlib seaborn
```

---

## Author

**Santi Isgrò**
BSc in Computer Science — Università degli Studi di Catania

MSc in Bioinformatics — in progress

---

## License

MIT License
