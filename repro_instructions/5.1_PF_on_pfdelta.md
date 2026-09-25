# Reproduce GENCO §5.1 (Power Flow on PFΔ)

This covers **Figure 4**, **Figure 18**, and **Table 4**.

## 1. Table results

The GENCO numbers used in the paper are already on the following branch, one CSV per task:

[https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pfdelta/scripts/pfdelta/results](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pfdelta/scripts/pfdelta/results)

Look at `scripts/pfdelta/results/<task>/metrics_summary.csv` (IEEE 118). **Table 4** is task **3.1**.

PFNet, CANOS-PF, GNS-S, and Newton–Raphson numbers are taken from PFΔ (on the same splits).

## 2. Data

The original PFΔ files are JSON power-flow solutions. We converted them to parquet and split them into the paper tasks with `gridfm-datakit` `genco-paper-repro`:

1. [`pfdelta/batch_convert_pfdelta.py`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/pfdelta/batch_convert_pfdelta.py) — JSON → parquet
2. [`pfdelta/build_task_splits_from_data_processed.py`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/pfdelta/build_task_splits_from_data_processed.py) — train / val / test splits per task

You do not need to re-run that. The converted, split datasets are here:

| Paper task | Hugging Face |
| --- | --- |
| 1.1 | [gridfm/pfdelta_task1.1](https://huggingface.co/datasets/gridfm/pfdelta_task1.1) |
| 1.2 | [gridfm/pfdelta_task1.2](https://huggingface.co/datasets/gridfm/pfdelta_task1.2) |
| 1.3 / 2.1 | [gridfm/pfdelta_task1.3](https://huggingface.co/datasets/gridfm/pfdelta_task1.3) |
| 2.3 | [gridfm/pfdelta_task2.3](https://huggingface.co/datasets/gridfm/pfdelta_task2.3) |
| 3.1 | [gridfm/pfdelta_task3.1](https://huggingface.co/datasets/gridfm/pfdelta_task3.1) |
| 4.1 | [gridfm/pfdelta_task4.1](https://huggingface.co/datasets/gridfm/pfdelta_task4.1) |
| 4.2 | [gridfm/pfdelta_task4.2](https://huggingface.co/datasets/gridfm/pfdelta_task4.2) |
| 4.3 | [gridfm/pfdelta_task4.3](https://huggingface.co/datasets/gridfm/pfdelta_task4.3) |

```bash
pip install "huggingface_hub[cli]"
hf download gridfm/pfdelta_task4.3 --repo-type dataset --local-dir pfdelta_task4.3
```

## 3. Training and checkpoints

Use this graphkit branch and these configs:

- Branch: [`genco-paper-repro-pfdelta`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pfdelta)
- Configs: [`HGNS_PF_pfdelta_bs64_seed{1,2,3}.yaml`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pfdelta/scripts/pfdelta/config) (seeds `0` / `42` / `1234`)

Same YAML for every task; only `--data_path` changes.

```bash
git clone -b genco-paper-repro-pfdelta https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .
TORCH_CUDA_VERSION=$(python -c "import torch; print(torch.__version__ + ('+cpu' if torch.version.cuda is None else ''))")
pip install torch-scatter -f https://data.pyg.org/whl/torch-${TORCH_CUDA_VERSION}.html

gridfm_graphkit train \
  --config scripts/pfdelta/config/HGNS_PF_pfdelta_bs64_seed1.yaml \
  --data_path pfdelta_task4.3/case118 \
  --exp_name case118_task4.3 \
  --run_name bs64_same_norm_shuffle_train_paper_seed1
```

Repeat for seeds 2 and 3, and for the other `pfdelta_task*` datasets.

**Checkpoints:** [gridfm/genco-pfdelta-base](https://huggingface.co/gridfm/genco-pfdelta-base)

Weights: `task_<id>/seed{1,2,3}/last.pt` (last-epoch weights extracted from Lightning `last.ckpt`). These match the paper CSVs.

The training runs that match the paper CSVs are in the same repo under [`mlflow/`](https://huggingface.co/gridfm/genco-pfdelta-base/tree/main/mlflow) (`bs64_same_norm_shuffle_train_paper_seed{1,2,3}` only).

Task 3.1 was not trained separately. The paper 3.1 numbers come from evaluating the **task 1.3 best-validation snapshot** (`best_model_state_dict.epoch_299.pt`) on 3.1 data, not from `last.ckpt`. Those files are under `task_3.1/seed{1,2,3}/`.

## 4. Reusing the saved model

```bash
gridfm_graphkit evaluate \
  --config scripts/pfdelta/config/HGNS_PF_pfdelta_bs64_seed1.yaml \
  --model_path path/to/last.pt \
  --data_path pfdelta_task4.3/case118
```

For task 3.1, use `task_3.1/.../best_model_state_dict.epoch_299.pt` (the 1.3 val-best weights) and `--data_path` pointing at the 3.1 `case118` folder.

## 5. Figures

From the graphkit repo root (`pip install -e .` already pulls in pandas, matplotlib, seaborn):

```bash
python scripts/pfdelta/make_latex_tables.py
python scripts/pfdelta/format_table_exponents.py
python scripts/pfdelta/make_pbl_barplots.py --metric mean   # Figure 4
python scripts/pfdelta/make_pbl_barplots.py --metric max    # Figure 18
```

GENCO cells come from `scripts/pfdelta/results/<task>/metrics_summary.csv`. Output:

- `scripts/pfdelta/figures/pbl_all_rows_{mean,max}.pdf`
- `scripts/pfdelta/table_a_{8,9,10,11}_genco_exponent_fixed.tex`

Table 4 is `table_a_10_genco_exponent_fixed.tex` (task 3.1).