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

## Headline results

| Question | Result |
|----------|--------|
| Dataset | 12,424 curated EGFR molecules (ChEMBL CHEMBL203), 66.7% active |
| ROC-AUC, random split (Random Forest / XGBoost) | 0.943 / 0.934 |
| ROC-AUC, scaffold split (Random Forest / XGBoost) | 0.893 / 0.884 |
| Simple baselines, scaffold split | Logistic regression 0.843, Tanimoto nearest neighbour 0.874 |
| Early enrichment on a hard-negative library (5% actives) | EF at 1%: Random Forest 16.5 (maximum 20.2) vs 6.7 for the nearest-neighbour baseline |
| Four approved EGFR drugs removed from training | All rank above 97% to 100% of the measured inactives |

**How to read these numbers**
- Most of the overall separation between actives and inactives comes from similarity to known EGFR compounds: a nearest-neighbour rule with no training reaches ROC-AUC 0.874 against 0.893 for Random Forest.
- The models add more at the top of the ranking, which is what a screen follows up: early enrichment is about 2.5 times higher than the baseline. This rests on small numbers (11 molecules at 1%) and a single split.
- The top-ranked molecules are close analogs of known training actives (median nearest-neighbour similarity 0.80), so the model recovers known chemotypes and does not propose novel scaffolds.

---


## Repository Structure

```
egfr-drug-discovery/
├── notebooks/
│   ├── 01_dataset_construction.ipynb
│   ├── 02_molecular_representation.ipynb
│   ├── 03_ml_screening.ipynb
│   └── 04_virtual_screening.ipynb
├── data/                     # not tracked, see data/README.md
├── results/
│   ├── plots/                # figures, see results/plots/README.md
│   ├── screening_hard_negatives.csv
│   └── screening_top_hits.csv
├── requirements.txt
└── README.md
```

---

## Pipeline

### 1. Dataset construction

Bioactivity data for EGFR (CHEMBL203, Homo sapiens, single protein) was retrieved from ChEMBL via the REST API, restricted to standardized IC50 values in nM.

1. 25,244 raw IC50 records.
2. Structures standardized with RDKit (functional-group cleanup, parent fragment, charge neutralization); entries without carbon discarded. 20 records dropped.
3. 628 records removed because they carry a ChEMBL `data_validity_comment` or lack a numeric value or SMILES.
4. Each remaining record labelled: 19,360 exact measurements compared with the 1000 nM threshold, 5,053 censored measurements ("IC50 > x" or "IC50 < x") kept only when the bound already decides the label, 183 undecidable records dropped.
5. Duplicates merged by the connectivity block of the InChIKey (13,301 ChEMBL IDs became 12,887 structures); each structure labelled by the majority of its records, 463 exact ties dropped.

**Final dataset: 12,424 molecules, 8,285 active (66.7%) and 4,139 inactive.** Active means IC50 below 1000 nM (pIC50 above 6).

Distribution of pIC50 for the 9,952 structures with an exact IC50 (median 97.9 nM); the remaining 2,472 structures are supported only by censored records.

### 2. Molecular representation

- 2048-bit Morgan fingerprints (radius 2) with RDKit's `MorganGenerator`.
- Six Lipinski-related descriptors: molecular weight, LogP, H-bond donors and acceptors, TPSA, rotatable bonds.


Active compounds show modestly higher molecular weight, LogP, acceptor count and rotatable bonds. No single descriptor separates the classes.

### 3. Machine learning

Random Forest (500 trees) and XGBoost (500 trees) on the fingerprints, with class weighting, evaluated on a stratified random split and on a scaffold split (4,555 Murcko scaffolds; whole scaffold groups assigned to train or test).

| Model | Random split ROC-AUC | Scaffold split ROC-AUC |
|-------|----------------------|------------------------|
| Random Forest | 0.943 | 0.893 |
| XGBoost | 0.934 | 0.884 |


ROC curves on the random split. The scaffold-split ROC-AUC is the primary estimate, because the random split is optimistic: structurally similar molecules fall in both sets.

**Baselines** on the same splits:

| Model | Random: ROC-AUC | Random: PR-AUC | Scaffold: ROC-AUC | Scaffold: PR-AUC |
|-------|-----------------|----------------|-------------------|------------------|
| Logistic regression | 0.908 | 0.943 | 0.843 | 0.880 |
| Tanimoto nearest neighbour | 0.920 | 0.950 | 0.874 | 0.898 |
| XGBoost | 0.934 | 0.962 | 0.884 | 0.922 |
| Random Forest | 0.943 | 0.969 | 0.893 | 0.929 |

PR-AUC random baselines are the active fraction of the test set (0.667 random split, 0.599 scaffold split), so PR-AUC is not comparable across splits.

**Features** (Random Forest, scaffold split): descriptors alone 0.764, fingerprints alone 0.893, both 0.895. Fingerprints alone are used in the pipeline.

### 4. Virtual screening test

The scaffold-split test set was used as a screening library: all 997 measured inactives plus 52 randomly sampled actives (5.0% actives), repeated 100 times. Models were trained on the scaffold training set only.

| Metric | Random Forest | Tanimoto 1-NN |
|--------|---------------|----------------|
| ROC-AUC | 0.895 (0.857 to 0.931) | 0.875 (0.833 to 0.912) |
| PR-AUC | 0.505 (0.404 to 0.596) | 0.315 (0.248 to 0.370) |
| EF at 1% (maximum 20.2) | 16.5 (12.8 to 20.2) | 6.7 (3.7 to 9.2) |
| EF at 5% | 9.2 (7.6 to 10.7) | 8.3 (7.2 to 9.5) |
| Precision at 1% (11 molecules) | 0.82 (0.64 to 1.00) | 0.33 (0.18 to 0.46) |
| Precision at 5% (53 molecules) | 0.46 (0.38 to 0.53) | 0.41 (0.36 to 0.47) |

Means over 100 resamples with the 5th to 95th percentile in parentheses. The intervals reflect only the random choice of actives, not model or split variability. The random baseline is EF 1.0 and PR-AUC 0.05.

**Top hits.** In one representative library, the top 5% of the Random Forest ranking (53 molecules) contains 23 measured actives. Their mean pairwise Tanimoto similarity is 0.21 (0.18 for all sampled actives), so the enrichment is not driven by a single chemical series. Their median similarity to the nearest training active is 0.80 (0.75 for all actives), so they are close analogs of known compounds. The list is saved in `results/screening_top_hits.csv` and is the input for the docking step.

**Sanity check with approved drugs.** Gefitinib, erlotinib, lapatinib and osimertinib were removed from the training set, the model was retrained, and each drug was scored against the measured inactives.

| Drug | Random Forest score | Scores above | Max similarity to a training active |
|------|---------------------|--------------|--------------------------------------|
| Gefitinib | 0.90 | 97.3% of inactives | 0.92 |
| Erlotinib | 0.98 | 99.9% | 0.90 |
| Lapatinib | 1.00 | 100.0% | 0.87 |
| Osimertinib | 0.97 | 99.7% | 0.89 |

Close analogs of each drug remain in the training set, so this shows that the model recognizes known EGFR chemotypes. It is not a novelty test and covers only four compounds.

### 5. Molecular docking (planned)

AutoDock Vina on the 53 top hits, after validating the protocol by re-docking a co-crystallized ligand (RMSD below 2 Å). The question is whether a docking score separates the 23 measured actives from the 30 false positives. Docking is treated as a structural plausibility check, not as experimental validation.

---

## Dependencies

```bash
pip install -r requirements.txt
```

| Package | Purpose |
|---------|---------|
| rdkit | Standardization, fingerprints, descriptors, scaffolds |
| chembl_webresource_client | ChEMBL data retrieval |
| pandas, numpy | Data handling |
| scikit-learn | Random Forest, logistic regression, metrics |
| xgboost | Gradient boosting |
| matplotlib, seaborn | Plots |

---


## Author

**Santi Isgrò**
BSc in Computer Science — Università degli Studi di Catania

MSc in Bioinformatics — in progress

---

## License

MIT License
