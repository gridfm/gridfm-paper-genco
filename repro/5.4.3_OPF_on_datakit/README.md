# Reproduce GENCO §5.4.3 (Optimal Power Flow on datakit)

This covers the appendix optimality and feasibility table (`tab:opf_scaling`) for GENCO Base and Small. The runtime figure and the speedup columns are in [5.4.2-3_Runtime](../5.4.2-3_Runtime/README.md).

GENCO Base / Small are trained from scratch on each grid. Hidden size is 48 / 24.

## 1. Table results

The GENCO numbers used in the table are on this branch:

[https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/datakit_opf/results](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/datakit_opf/results)

`scripts/datakit_opf/results/opf_scaling_combined.csv` is one row per grid, size, and seed, plus the mean and sample standard deviation. `opf_scaling_aggregated.csv` is the aggregated sheet the table is built from.

In that CSV, `seed1` / `seed2` are YAML seeds `0` / `1`.

Base and Small are included for IEEE 14, 30, 57, 118, GOC 500, and GOC 2000.

## 2. Data

Generated with `gridfm-datakit`. You do not need to regenerate. Hugging Face (parquet; graphkit expects `{data_path}/{network}/raw/`):

| Grid | Hugging Face | `network` folder |
| --- | --- | --- |
| IEEE 14 | [gridfm/opf_small_case14_ieee](https://huggingface.co/datasets/gridfm/opf_small_case14_ieee) | `case14_ieee` |
| IEEE 30 | [gridfm/opf_small_case30_ieee](https://huggingface.co/datasets/gridfm/opf_small_case30_ieee) | `case30_ieee` |
| IEEE 57 | [gridfm/opf_small_case57_ieee](https://huggingface.co/datasets/gridfm/opf_small_case57_ieee) | `case57_ieee` |
| IEEE 118 | [gridfm/opf_small_case118_ieee](https://huggingface.co/datasets/gridfm/opf_small_case118_ieee) | `case118_ieee` |
| GOC 500 | [gridfm/opf_small_case500_goc](https://huggingface.co/datasets/gridfm/opf_small_case500_goc) | `case500_goc` |
| GOC 2000 | [gridfm/opf_small_case2000_goc](https://huggingface.co/datasets/gridfm/opf_small_case2000_goc) | `case2000_goc` |

```bash
pip install "huggingface_hub[cli]"
mkdir -p data/case118_ieee/raw
hf download gridfm/opf_small_case118_ieee --repo-type dataset --local-dir data/case118_ieee/raw
```

Same pattern for the other grids. Then `--data_path data`.

## 3. Training and checkpoints

Use this graphkit branch and these configs. The paper results were obtained with `genco-paper-repro`; we only guarantee the same numbers on this branch, because `main` is under active development.

- Branch: [`genco-paper-repro`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro)
- Configs: [`scripts/datakit_opf/configs/<network>_{base,small}_seed<seed>.yaml`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/datakit_opf/configs)

Seeds are `0` and `1`, for Base and Small, on every grid above.

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

gridfm_graphkit train \
  --config scripts/datakit_opf/configs/case118_ieee_base_seed0.yaml \
  --data_path data
```

Repeat for the other configs. Each YAML sets the grid, hidden size, and seed.

**Checkpoints:** [gridfm/genco-opf-datakit-base](https://huggingface.co/gridfm/genco-opf-datakit-base)

Weights: `<network>/{base,small}/seed<seed>/last.pt` plus `normalizer_stats.pt`, except GOC 2000 seed 0 (Base and Small), which uses `best_model_state_dict.pt`. For the other runs the table is the test at the end of training, on the last-epoch weights (`last.pt`, extracted from Lightning `last.ckpt`). GOC 2000 seed 0 was scored by a later `evaluate` of the best-validation snapshot. The training YAML for that run sits in the same folder.

The training runs that match the table are in the same repo under [`mlflow/train/`](https://huggingface.co/gridfm/genco-opf-datakit-base/tree/main/mlflow/train).

## 4. Reusing the saved model

There is no separate eval YAML. Reuse the train config.

```bash
hf download gridfm/genco-opf-datakit-base --include "case118_ieee/base/seed0/**" --local-dir genco-opf-datakit-base

gridfm_graphkit evaluate \
  --config scripts/datakit_opf/configs/case118_ieee_base_seed0.yaml \
  --data_path data \
  --model_path genco-opf-datakit-base/case118_ieee/base/seed0/last.pt \
  --normalizer_stats genco-opf-datakit-base/case118_ieee/base/seed0/normalizer_stats.pt
```

Repeat for the other grids, sizes, and seeds. Pass `last.pt`, except GOC 2000 seed 0, where the model path is `best_model_state_dict.pt`.

## 5. Appendix table

From the graphkit repo root (`pip install -e .` already pulls in pandas):

```bash
python scripts/datakit_opf/build_opf_scaling_table.py
```

The script reads `scripts/datakit_opf/results/opf_scaling_aggregated.csv` and writes `scripts/datakit_opf/opf_scaling_table.tex`.
