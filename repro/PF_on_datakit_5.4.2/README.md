# Reproduce GENCO §5.4.2 (Power Flow on datakit)

This covers the residual-vs-grid-size plot (`fig:gridsize_vs_residuals_pf`). The runtime figures and speedup columns are in [Runtime_5.4.2-3](../Runtime_5.4.2-3/README.md).

GENCO Base / Small / Tiny are trained from scratch on each grid. Hidden size is 48 / 24 / 12.

## 1. Figure results

The GENCO numbers used in the figure are on this branch:

[https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/datakit_pf/results](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/datakit_pf/results)

`scripts/datakit_pf/results/pf_eval_combined_eval_metrics.csv` is one row per grid, size, and seed. `pf_eval_aggregated.csv` is the mean and sample standard deviation (`ddof=1`) plotted in the figure.

For grids through GOC 500, `seed1` / `seed2` / `seed3` in that CSV are YAML seeds `0` / `1` / `42`. For GOC 2000 and GOC 10000 the CSV seed is the YAML seed (`0` and `42` on GOC 2000; `0` and `1` on GOC 10000).

Base is included through GOC 500. Small and Tiny are included through GOC 10000.

## 2. Data

Generated with `gridfm-datakit`. You do not need to regenerate. Hugging Face (parquet; graphkit expects `{data_path}/{network}/raw/`):

| Grid | Hugging Face | `network` folder |
| --- | --- | --- |
| IEEE 14 | [gridfm/pf_small_case14_ieee](https://huggingface.co/datasets/gridfm/pf_small_case14_ieee) | `case14_ieee` |
| IEEE 30 | [gridfm/pf_small_case30_ieee](https://huggingface.co/datasets/gridfm/pf_small_case30_ieee) | `case30_ieee` |
| IEEE 57 | [gridfm/pf_small_case57_ieee](https://huggingface.co/datasets/gridfm/pf_small_case57_ieee) | `case57_ieee` |
| IEEE 118 | [gridfm/pf_small_case118_ieee](https://huggingface.co/datasets/gridfm/pf_small_case118_ieee) | `case118_ieee` |
| GOC 500 | [gridfm/pf_small_case500_goc](https://huggingface.co/datasets/gridfm/pf_small_case500_goc) | `case500_goc` |
| GOC 2000 | [gridfm/pf_small_case2000_goc](https://huggingface.co/datasets/gridfm/pf_small_case2000_goc) | `case2000_goc` |
| GOC 10000 | [gridfm/pf_small_case10000_goc](https://huggingface.co/datasets/gridfm/pf_small_case10000_goc) | `case10000_goc` |

```bash
pip install "huggingface_hub[cli]"
mkdir -p data/case118_ieee/raw
hf download gridfm/pf_small_case118_ieee --repo-type dataset --local-dir data/case118_ieee/raw
```

Same pattern for the other grids (`data/case14_ieee/raw`, `data/case500_goc/raw`, …). Then `--data_path data`.

## 3. Training and checkpoints

Use this graphkit branch and these configs. The paper results were obtained with `genco-paper-repro`; we only guarantee the same numbers on this branch, because `main` is under active development.

- Branch: [`genco-paper-repro`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro)
- Configs: [`scripts/datakit_pf/configs/<network>_{base,small,tiny}_seed<seed>.yaml`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/datakit_pf/configs)

Through GOC 500 the seeds are `0`, `1`, and `42`, for Base, Small, and Tiny. GOC 2000 is Small and Tiny, seeds `0` and `42`. GOC 10000 is Small and Tiny, seeds `0` and `1`.

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

gridfm_graphkit train \
  --config scripts/datakit_pf/configs/case118_ieee_base_seed0.yaml \
  --data_path data
```

Repeat for the other configs. Each YAML sets the grid, hidden size, and seed.

**Checkpoints:** [gridfm/genco-pf-datakit-base](https://huggingface.co/gridfm/genco-pf-datakit-base)

Weights: `<network>/{base,small,tiny}/seed<seed>/best_model_state_dict.pt` plus `normalizer_stats.pt`, except GOC 10000, which uses `last.pt`. Grids through GOC 500 and GOC 2000 were scored on the best-validation state dict. GOC 10000 was scored on the last-epoch weights (`last.pt`, extracted from Lightning `last.ckpt`). The training YAML for that run sits in the same folder.

The MLflow runs are in the same repo under [`mlflow/`](https://huggingface.co/gridfm/genco-pf-datakit-base/tree/main/mlflow). For grids through GOC 500 the figure uses `mlflow/eval/`. For GOC 2000 and GOC 10000 it uses `mlflow/train/`.

## 4. Reusing the saved model

There is no separate eval YAML. Reuse the train config.

```bash
hf download gridfm/genco-pf-datakit-base --include "case118_ieee/base/seed0/**" --local-dir genco-pf-datakit-base

gridfm_graphkit evaluate \
  --config scripts/datakit_pf/configs/case118_ieee_base_seed0.yaml \
  --data_path data \
  --model_path genco-pf-datakit-base/case118_ieee/base/seed0/best_model_state_dict.pt \
  --normalizer_stats genco-pf-datakit-base/case118_ieee/base/seed0/normalizer_stats.pt
```

Repeat for the other grids, sizes, and seeds. For GOC 10000, pass `last.pt` instead of `best_model_state_dict.pt`.

## 5. Residual figure

From the graphkit repo root (`pip install -e .` already pulls in pandas and matplotlib):

```bash
python scripts/datakit_pf/plot_grid_scaling_active_residuals_pf.py
```

The script reads `scripts/datakit_pf/results/pf_eval_aggregated.csv` and writes `scripts/datakit_pf/figures/grid_scaling_active_residuals_pf.pdf`.
