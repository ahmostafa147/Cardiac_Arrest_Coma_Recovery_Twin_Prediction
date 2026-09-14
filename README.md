# EEG-Twin

CEBRA trajectory embedding and Transformer digital twin for post-cardiac-arrest
EEG, with the figures built on them.

## Requirements

Python 3.11, `pip install -r requirements.txt`. A GPU for twin training; the
rest runs on CPU.

Supply only the preprocessing inputs. Everything else is generated.

| Path | Contents |
|---|---|
| `data/raw/protopnet/{MGH,ULB,YNH,BIDMC,BWH,UTW}/*.npz` | per-recording prototype-network outputs |
| `data/raw/qeeg/` | per-patient 10 s qEEG csvs (with-spike set) |
| `data/raw/offsets.csv` | time from ROSC per recording |
| `data/tables/ICARE_clinical.csv` | age, sex, vfib, ROSC, time to arrest, CPC |
| `data/tables/split_train.csv`, `split_test.csv` | patient-level split (one `patient_id` column) |

## Run

```bash
python run_all.py
```

Runs every step in order, skips whatever is already built, and stops with a
plain list if an input is missing.

```bash
python run_all.py --list      # show the plan
python run_all.py --force     # rebuild everything
python run_all.py --figures   # figures only
python tests/smoke_test.py    # synthetic end-to-end check, ~5 min
```

## Steps

| Step | In | Out | Time |
|---|---|---|---|
| `run_build_dataset` | `data/raw/`, `data/tables/` | `data/dataset/` | hours |
| `run_cebra_features` | `data/dataset/` | `data/cebra/prep_*.npz` | ~1 min |
| `run_train_cebra` | `data/cebra/prep_*` | `data/cebra/embeddings_*` | 6-45 min |
| `run_export_for_twin` | dataset + embeddings | `data/twin/input/` | ~1 min |
| `run_train_twin` | `data/twin/input/`, clinical | `data/twin/handoffs/`, models, fig1-7, metrics | hours, GPU |
| `run_fig_globes` | embeddings + handoffs | Figure 3 | ~1 min |
| `run_fig_twin` | embeddings + handoffs | Figure 2 | ~1 min |
| `run_fig_prototypes` | embeddings | prototype figure | ~1 min |
| `run_embedding_eval` | prep + embeddings | metrics *(optional)* | ~4 min |
| `run_retrieval_ablation` | handoffs | ablation table *(optional)* | ~3 s |

## Output

```
outputs/figures/cebra/   trajectory globes · twin in CEBRA space · prototype map
outputs/figures/twin/    fig1-7
outputs/figures/eval/    confusion · CPC · centroid distances
outputs/models/          CEBRA and twin weights, scalers, feature pipeline
outputs/metrics/twin/    13 tables + all_metrics.json
```
Figures are written as interactive HTML, 600-dpi PNG and vector PDF.

## Layout

`data/` holds anything a later step reads. `outputs/` holds terminal artifacts.
One location per artifact, no fallbacks. Both are gitignored.

```
run_all.py       the pipeline
src/             config, constants, and one module per stage
scripts/         one runner per stage
tests/           smoke test + synthetic input generator
notebooks/       Colab versions of the figure, preprocessing and twin code
docs/            pipeline diagram, class-encoding reference
data/            inputs and intermediates   (gitignored)
outputs/         figures, models, metrics   (gitignored)
```

Change hyperparameters in `src/config.py`; change which patients a figure uses
at the top of its runner.

## Notes

**Class encoding.** `constants.py` uses
`['BurstSupp','Seizure','LPD','GPD','LRDA','GRDA','Continuous','Discontinuous']`.
Index 0 is Burst Suppression: the base softmax emits 6 classes with 0 = "Other",
and preprocessing splits that class by the burst-suppression index into
BurstSupp (< 0.5), Discontinuous (0.5-0.9) and Continuous (>= 0.9). See `docs/class_encoding.md`.

**Palette.** Validated on all 28 pairs: worst normal-vision dE 20.1 (floor 15),
worst colour-blind dE 8.0 (target 8).

**Smoothing.** Gaussian over the raw segment sequence (sigma 9 bins = 45 min),
re-projected onto the sphere, then centripetal Catmull-Rom. Chosen by sweep to
keep state structure while the line stays smooth.

**Determinism.** Seed 42; the CEBRA steps reproduce the embedding bit-for-bit
on the same device.

## Limits

- The build-dataset step has run only against synthetic inputs. On the first
  real run, check its output for all-NaN qEEG columns — unrecognised column
  names are filled with NaN silently.
- The twin has trained only at smoke scale (4 patients, 1 seed, 1 epoch).
- Prototype-network training is not in this repo; its outputs are an input.
- Tested on macOS/ARM, Python 3.11.5 only.
- 45 prototypes span 7 of the 8 phenotypes; none codes for Discontinuous.
