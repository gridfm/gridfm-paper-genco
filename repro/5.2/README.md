# Reproduce §5.2 (Optimal Power Flow on OPFData)

Table 5 (`tab:opf_results`): GENCO Base vs HH-MPNN on six OPFData grids.

**Code**

- Train / eval / table: [gridfm-graphkit](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro) `genco-paper-repro` (`scripts/opfdata/`)
- Optional conversion from raw OPFData JSON: [gridfm-datakit](https://github.com/gridfm/gridfm-datakit/tree/genco-paper-repro) `opf_data/batch_convert.py`

HH-MPNN cells are taken from Arowolo et al. and are hardcoded in `make_table5.py`. They are not retrained here.

Pipeline: download converted OPFData + paper splits + checkpoints → `gridfm_graphkit evaluate --batch_size 512` (or train from the same YAMLs) → `make_table5.py` fills GENCO rows from `scripts/opfdata/results/*/seed*/metrics.csv`.

## 1. Data

Converted N-1 OPFData parquet (300k scenarios per grid). The HH-MPNN 90/5/5 splits are **not** in these datasets; they live next to the checkpoints.

| Grid | Hugging Face |
| --- | --- |
| IEEE 14 | [gridfm/opfdata_case14_ieee](https://huggingface.co/datasets/gridfm/opfdata_case14_ieee) |
| IEEE 30 | [gridfm/opfdata_case30_ieee](https://huggingface.co/datasets/gridfm/opfdata_case30_ieee) |
| IEEE 57 | [gridfm/opfdata_case57_ieee](https://huggingface.co/datasets/gridfm/opfdata_case57_ieee) |
| IEEE 118 | [gridfm/opfdata_case118_ieee](https://huggingface.co/datasets/gridfm/opfdata_case118_ieee) |
| GOC 500 | [gridfm/opfdata_case500_goc](https://huggingface.co/datasets/gridfm/opfdata_case500_goc) |
| GOC 2000 | [gridfm/opfdata_case2000_goc](https://huggingface.co/datasets/gridfm/opfdata_case2000_goc) |

```bash
pip install "huggingface_hub[cli]"
hf download gridfm/opfdata_case118_ieee --repo-type dataset --local-dir opfdata_case118_ieee
hf download gridfm/genco-opfdata-base --include "splits/*" --local-dir genco-opfdata-base
mkdir -p scripts/opfdata/splits
cp genco-opfdata-base/splits/*.pt scripts/opfdata/splits/
```

Configs expect `split_from_existing_files: scripts/opfdata/splits/` and must be run from the graphkit repo root.

## 2. Checkpoints

Eighteen GENCO Base weights (seeds **42 / 3 / 17**) plus normalizer stats and per-run `metrics.csv`: [gridfm/genco-opfdata-base](https://huggingface.co/gridfm/genco-opfdata-base).

```bash
hf download gridfm/genco-opfdata-base --include "case118_ieee/seed42/**" --local-dir genco-opfdata-base
```

`default` in the YAML filenames is seed 42. IEEE 14/30/118 seed-42 runs used `data.workers: 16` (logged); later seeds on those grids used 32. That only affects dataloader workers, not architecture or losses.

## 3. Train

YAML per grid and seed: `scripts/opfdata/configs/HGNSQ_penalty_11_OPFData_case{14,30,57,118,500,2000}_{default,3,17}.yaml`.

```bash
gridfm_graphkit train \
  --config scripts/opfdata/configs/HGNSQ_penalty_11_OPFData_case118_default.yaml \
  --data_path opfdata_case118_ieee \
  --exp_name HGNSQ_penalty_11_OPFData_case118_default
```

Batch size in the YAML is 64 (IEEE), 16 (GOC 500), or 8 (GOC 2000).

## 4. Eval

There is no separate eval YAML. Reuse the train config and override batch size. `genco-paper-repro` implements `evaluate --batch_size`. Paper evals used 512.

```bash
gridfm_graphkit evaluate \
  --config scripts/opfdata/configs/HGNSQ_penalty_11_OPFData_case118_default.yaml \
  --data_path opfdata_case118_ieee \
  --model_path genco-opfdata-base/case118_ieee/seed42/best_model_state_dict.pt \
  --normalizer_stats genco-opfdata-base/case118_ieee/seed42/normalizer_stats.pt \
  --batch_size 512
```

Repeat for seeds 3/17 (`..._case118_3.yaml`, `..._case118_17.yaml`) and the other grids.

## 5. Table 5

Committed eval scalars are already under `scripts/opfdata/results/<grid>/seed<seed>/metrics.csv`. From the graphkit repo root:

```bash
python scripts/opfdata/make_table5.py
```

Writes `scripts/opfdata/table5_genco.tex`. GENCO means and stds match the published Table 5 (2-decimal gap; scientific elsewhere). HH-MPNN is hardcoded. One cosmetic difference vs the paper file: IEEE 30 \(S_{ij}(-)\) bolds HH-MPNN (3.00e-4) rather than GENCO, because that cell is strictly smaller.
