# Reproduce §5.1 (Power Flow on PFΔ)

Figure 4 (`pbl_all_rows_mean.pdf`), Figure 18 (`pbl_all_rows_max.pdf`), and Table 4.

**Code**

- Data conversion / task splits: `gridfm-datakit` [@](https://github.com/gridfm/gridfm-datakit/tree/genco-paper-repro) `genco-paper-repro`
- Train, eval, plots: `gridfm-graphkit` [@](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pfdelta) `genco-paper-repro-pfdelta`

**Data (usual path)** — already converted and split:


| Paper task | Hugging Face                                                                     |
| ---------- | -------------------------------------------------------------------------------- |
| 1.1        | [gridfm/pfdelta_task1.1](https://huggingface.co/datasets/gridfm/pfdelta_task1.1) |
| 1.2        | [gridfm/pfdelta_task1.2](https://huggingface.co/datasets/gridfm/pfdelta_task1.2) |
| 1.3 / 2.1  | [gridfm/pfdelta_task1.3](https://huggingface.co/datasets/gridfm/pfdelta_task1.3) |
| 2.3        | [gridfm/pfdelta_task2.3](https://huggingface.co/datasets/gridfm/pfdelta_task2.3) |
| 3.1        | [gridfm/pfdelta_task3.1](https://huggingface.co/datasets/gridfm/pfdelta_task3.1) |
| 4.1        | [gridfm/pfdelta_task4.1](https://huggingface.co/datasets/gridfm/pfdelta_task4.1) |
| 4.2        | [gridfm/pfdelta_task4.2](https://huggingface.co/datasets/gridfm/pfdelta_task4.2) |
| 4.3        | [gridfm/pfdelta_task4.3](https://huggingface.co/datasets/gridfm/pfdelta_task4.3) |


Task 3.1 test set also contains IEEE 57 and GOC 500. CANOS-PF, GNS-S, PFNet, and NR numbers are taken from PFΔ, not retrained here.

Pipeline: download a task → train/eval GENCO (three seeds) → aggregate → `make_latex_tables.py` fills GENCO rows from `results/` → `format_table_exponents.py` → barplots. PFNet / CANOS-PF / GNS / NR cells are hardcoded in `make_latex_tables.py`.

## 1. Data

```bash
pip install "huggingface_hub[cli]"
hf download gridfm/pfdelta_task4.3 --repo-type dataset --local-dir pfdelta_task4.3
```

Optional, from raw PFΔ JSON: `pfdelta/batch_convert_pfdelta.py` then `pfdelta/build_task_splits_from_data_processed.py` on `genco-paper-repro`.

## 2. Train

Seeds: `scripts/config/HGNS_PF_pfdelta_bs64_seed{1,2,3}.yaml` (`seed` 0 / 42 / 1234). Same YAML for every task; only `--data_path` changes.

```bash
gridfm_graphkit train \
  --config scripts/config/HGNS_PF_pfdelta_bs64_seed1.yaml \
  --data_path pfdelta_task4.3/case118 \
  --exp_name case118_task4.3 \
  --run_name bs64_same_norm_shuffle_train_paper_seed1
```

Repeat for seeds 2/3 and the other `pfdelta_task*` datasets.

Eval from the paper checkpoints is not documented here yet (checkpoints are on a cluster). See the note at the bottom.

## 3. GENCO numbers

From the `genco-paper-repro-pfdelta` repo root:

```bash
python aggregate_mlflow_metrics.py /path/to/mlflow/experiment --output results/4.3/metrics_summary
```



## 4. Figures 4 and 18, Table 4

Python 3.10–3.12. Tables and plots need:

```bash
pip install pandas matplotlib seaborn
```

That is already included if you install graphkit from this branch (`pip install -e .`). Matplotlib is pulled in by seaborn.

From the `genco-paper-repro-pfdelta` repo root, using [`scripts/pfdelta/`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pfdelta/scripts/pfdelta). GENCO cells come from `results/<task>/metrics_summary.csv` (`PBE (Mean, p.u.) latex` / `PBE (Max, p.u.) latex`).

```bash
python scripts/pfdelta/make_latex_tables.py
python scripts/pfdelta/format_table_exponents.py
python scripts/pfdelta/make_pbl_barplots.py --metric mean   # Figure 4
python scripts/pfdelta/make_pbl_barplots.py --metric max    # Figure 18
```

Writes `scripts/pfdelta/figures/pbl_all_rows_{mean,max}.pdf` and `scripts/pfdelta/table_a_{8,9,10,11}_genco_exponent_fixed.tex`.

Table 4 is `table_a_10_genco_exponent_fixed.tex` (Task 3.1). The paper file `GENCO/paper/figures/pf_delta/pf_delta_a_10.tex` is the same numbers with `\genco`, `GNS-S`, and the published caption.

## Note for the next agent (cluster checkpoints)

Do not invent eval commands until the user hands over paths. Then add an **Eval** subsection under §2 of this README (same command shape as `gridfm_graphkit evaluate`, with real `--model_path`).

Facts:

- Paper checkpoints and MLflow runs live on a **cluster**, not in git or Hugging Face.
- Graphkit helper with the old cluster layout: `generate_eval_commands.py` on `genco-paper-repro-pfdelta` (`MLFLOW_BASE=/dccstor/gridfm/mlflow_alban_pfdelta`, weights `artifacts/model/best_model_state_dict.epoch_99.pt`, run names `bs64_same_norm_shuffle_train_paper_seed{1,2,3}`).
- `--config` is `scripts/config/HGNS_PF_pfdelta_bs64_seed{1,2,3}.yaml`; `--data_path` is the HF task’s `case118/` folder.
- Do not document job launchers. One `evaluate` example plus where the `.pt` files are is enough.

