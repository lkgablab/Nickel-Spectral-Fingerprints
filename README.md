# Nickel Spectral Fingerprints

Code and data for predicting nickel-catalysed reaction yields from UV-Vis "spectral fingerprints" of nickel/ligand/solvent/base mixtures.

UV-Vis spectra of each mixture were collected on a BMG plate reader, stored as HDF5, and featurized. AutoGluon regression models were then trained on 200 samples with measured GC-MS yields for two reactions:

- `CO_tbu`: C–O coupling
- `MP_CF3`: metallaphotoredox decarboxylative arylation

The trained models are used to predict yields across the rest of the library.

## Setup

```bash
git lfs install                 # the dummy model in "Autogluon Models/dummy model" is stored with Git LFS
git clone https://github.com/lkgablab/Nickel-Spectral-Fingerprints.git
cd Nickel-Spectral-Fingerprints

python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m ipykernel install --user --name nickelLibrary
```

Run the notebooks from the repository root, in order, executing the cells top to bottom. All paths are relative to the root.

## Workflow

| Notebook | What it does | Main outputs |
|---|---|---|
| `[1] BMG to HDF5` | Converts raw plate-reader exports (`UV-Vis/BMG Output`) and their plate templates into HDF5 spectral libraries, swaps in repeated measurements, and builds the 200-sample training set and the time-course files. | `HDF5 Datasets/`, `UV-Vis Time Course/*.h5` |
| `[2] Plotting UV-Vis spectra` | Plotting functions for library spectra, controls and time courses. Not needed for modeling. | Figures, `SVG Exports/` |
| `[3] Preparing HDF5 data for modeling` | Featurizes the spectra into Datasets 1–5 and a one-hot baseline, attaching yields from `250728 - GCMS Yields.xlsx`. | `Featurized datasets/` |
| `[4] Autogluon Models` | Trains and evaluates AutoGluon models (random 80/20 split, seed 42), the one-hot baseline, an sklearn benchmark, and a Y-randomization control. | `Autogluon Models/`, `Autogluon Results/`, `Autogluon Predictions/sklearn/` |
| `[5] Predicted Output from Models` | Predicts yields for every untested library sample with the best_quality models. | `Autogluon Predictions/` |
| `[6] Leave One Ligand Out Models` | Holds out each of eight ligands (L02, L04, L05, L12, L32, L33, L48, L56) in turn and trains on the rest (Dataset 4 features). | `Autogluon Results/Leave One Ligand Out - *.xlsx` |

### Feature sets

| Dataset | Features |
|---|---|
| Dataset 1 | Normalized sample spectrum |
| Dataset 2 | Normalized ligand-only and metal-only control spectra for each sample |
| Dataset 3 | Peak features of the sample and its controls |
| Dataset 4 | Peak features plus sample-minus-control differences |
| Dataset 5 | Peak features of the sample only |
| One-hot | Metal, ligand and solvent identity only (no spectra) |

## Sample naming

Every spectrum is named `M##_L##_S##_B##`. `M00` or `L00` marks a ligand-only or metal-only control, and a trailing `_X` marks a control spectrum.

| Code | Meaning |
|---|---|
| `M01` / `M02` / `M03` | Ni(NO₃)₂·6H₂O / NiBr₂·diglyme / Ni(acac)₂ |
| `L01`–`L59` | Ligand |
| `S01`–`S07` | DMF, THF, MeCN, EtOH, EtOAc, DCE, DMSO |
| `B01` / `B02` / `B03` | No base / K₂CO₃ / DIPEA |

## Trained models and run times

The trained AutoGluon models (about 11 GB for notebooks [4] and [5], plus the Y-randomization and leave-one-ligand-out models) are too large for GitHub and are not included. The only exception is the dummy model.

When a model folder is missing, notebooks [4] and [6] train it. When it exists, they load and evaluate it instead, so the results tables are always complete. Training times on the original machine (Apple M4 Pro, 14 cores) were:

| Step | Time |
|---|---|
| [4] Random-split models, Datasets 1–5 | ~5.4 h |
| [4] One-hot models | ~0.8 h |
| [4] Y-randomization models | ~0.6 h |
| [6] Leave-one-ligand-out models | ~5.8 h |

Notebook [5] needs the best_quality models from [4]. Run [4] first.

## Reproducibility notes

- **Environment.** Python 3.11.8 with the versions pinned in `requirements.txt`, on macOS 15 (Apple silicon). AutoGluon models only load with the AutoGluon version that saved them. Retraining on other hardware or operating systems may give slightly different metrics.
- **Training CSVs.** [4] selects the test set by row position (seed 42). [3] therefore writes the training rows in the fixed order listed in `Featurized datasets/Training Datasets/Training sample order.csv`. Don't reorder or hand-edit the training CSVs.
- **One-hot models.** The one-hot dataset keeps its own sample order, so the seed-42 split gives the one-hot models a different 40-sample test set than Datasets 1–5. The committed one-hot CSV is the exact file the published one-hot models were trained on. Re-running [3] changes a few of its yields in the 16th significant digit, so retraining the one-hot models from scratch gives slightly different metrics.
- **Yields.** The yields in `250728 - GCMS Yields.xlsx` are stored at full double precision. Re-saving the file in Excel can round the last digits and change retrained models.
- **Y-randomization.** `Dataset 4 randomized.csv` is the exact shuffled dataset the Y-test models were trained on. [4] reuses it if present, because its shuffle is unseeded.
- **Dummy model.** [5] trains a tiny dummy AutoGluon model before loading the real ones. The real models failed to predict without this step in the original environment.
- **Running cells again.** [4] and [6] load existing models instead of retraining them. Delete a model folder to retrain that model.
