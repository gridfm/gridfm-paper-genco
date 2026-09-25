# GENCO reproducibility conventions

This file lives in the personal GENCO repo under `repro_instructions/`. It is for whoever writes the next section’s public guide (e.g. `repro_instructions/5.1_PF_on_pfdelta.md`), not for readers of the paper.

Name each guide `<section>_<TASK>_on_<dataset>.md` (e.g. `5.1_PF_on_pfdelta.md`, `5.2_OPF_on_opfdata.md`, `5.4.2_PF_on_datakit.md`). One file per section, directly in `repro_instructions/`.

The public README must be usable by someone with GitHub + Hugging Face only. No cluster, no LSF, no personal paths.

## Where things go

| What | Where | Notes |
| --- | --- | --- |
| Frozen code + configs used in the paper | Public GitHub **branch** on `gridfm-graphkit` and/or `gridfm-datakit` | Snapshot branch, not `main`. Name it `genco-paper-repro` or `genco-paper-repro-<topic>` (e.g. `genco-paper-repro-pfdelta`). We only guarantee paper numbers on that branch; `main` is under active development. |
| Table/figure **scripts** | Same GitHub branch | e.g. `scripts/pfdelta/make_latex_tables.py`, `scripts/opfdata/make_table5.py` |
| Paper **metric CSVs** (small) | Same GitHub branch (e.g. `scripts/<topic>/results/`) | These are the GENCO numbers in the tables. |
| Converted / split **datasets** | Hugging Face **dataset** repos | Public if the source license allows. Point the README at these; do not ask people to start from cluster copies. |
| **Train/val/test index files** if they are not inside the dataset | Same HF **model** repo (e.g. `splits/*.pt`) | OPFData: splits live next to checkpoints, not in `gridfm/opfdata_*`. PFΔ: splits are already in the task datasets. |
| **Weights that reproduce the tables** | Hugging Face **model** repo (org `gridfm`) | Public once the section is ready. Prefer a name like `gridfm/genco-<topic>-base`. Include `normalizer_stats.pt` if `evaluate` needs it. |
| Paper **MLflow runs** that produced those weights and/or CSVs | Same HF model repo, subdirectory `mlflow/` | See “Which MLflow runs” below. No `.pt` inside `mlflow/` if the weights already sit at `<grid-or-task>/seed*/`. |
| Public guide for that section | Personal GENCO repo `repro_instructions/<section>.md` (and later the paper repo if you copy it) | Short. Links out. Does not duplicate code. Same five-section shape every time. |
| This conventions file | Personal GENCO repo `repro_instructions/REPRO_RULES.md` | Not for paper readers. |

**Do not put on GitHub**

- Checkpoints (`.pt`, `.ckpt`), including split tensors if they are large and already on HF
- Raw or converted datasets
- Full MLflow trees
- Cluster harvest dumps (`*runs_table.csv`, `sheet2_*.csv`)
- Cluster job launchers, `bsub` scripts, `cmd.txt`
- Scripts with hardcoded `/dccstor/...` or `/u/apu/...` paths
- Intermediate HTML/PDF dumps that are not the paper figures

**Do not put on Hugging Face**

- Source code or plot/table scripts (that is GitHub)
- Cluster paths, launchers, or aggregation notebooks

The HF **model card** must repeat: GitHub branch, YAML configs, which weight files match the paper, `--data_path` layout, and any exception (e.g. PFΔ task 3.1). Keep it aligned with `repro_instructions/<section>.md`.

## Which MLflow runs

Put on HF only the runs that belong to the paper numbers:

- **Table = in-training `test()` after `fit()`** (PFΔ): upload those **training** runs. Their `artifacts/test` is the CSV. Do not add later standalone evals.
- **Table = standalone `evaluate` of saved weights** (OPFData §5.2): upload **both**
  - `mlflow/train/…` — the training runs the checkpoints came from
  - `mlflow/eval/…` — the `evaluate` runs whose `artifacts/test` was copied into `results/*/metrics.csv`

Say in the README which of the two `artifacts/test` dumps is Table N. Do not leave readers to guess (train-run test ≠ paper eval for §5.2).

## Public README: order and tone

Keep it short. Same order every time:

1. **Where the table/figure numbers are** — GitHub CSVs, which file is which table.
2. **Data** — how it was obtained (one or two sentences + script links), then the HF dataset links. A download command that matches graphkit’s layout (see below). Say they do not need to reconvert. If splits are not in the dataset, show the copy from the model repo.
3. **Training** — snapshot branch (guarantee numbers only there), configs (seeds), one `train` command. Then the HF checkpoint link, the layout, and whether weights come from the **training** run.
4. **Reusing the saved model** — one `evaluate` command, including `--normalizer_stats` and eval-only flags the paper used (e.g. `--batch_size 512`). Be explicit if the saved file is **not** what the paper logged.
5. **Figures / table TeX** — the plot/table scripts and what they write.

Baselines taken from another paper: say so in one line (not retrained here).

Reason for the snapshot branch: paper numbers were obtained there; `main` moves. Do **not** frame it as “PyPI is missing a flag.”

### `--data_path` layout

Graphkit loads `{data_path}/{network}/raw/*.parquet` (then writes `processed/` on first run). Hugging Face dataset dumps are often parquet at the **repo root**. The README must nest the download, e.g.

```bash
mkdir -p data/case118_ieee/raw
hf download gridfm/opfdata_case118_ieee --repo-type dataset --local-dir data/case118_ieee/raw
# then --data_path data
```

Do not tell people `--data_path opfdata_case118_ieee` unless that folder already contains `case118_ieee/raw/`. First eval from HF parquet rebuilds `processed/` (slow; hundreds of thousands of `.pt` files). Spot-check **this** path, not only cluster data that already has `processed/`.

### Do not mention in the public README

- How jobs were launched (`bsub`, LSF, queue names, GPU flags)
- How CSVs were built from MLflow (`aggregate_mlflow_metrics.py`, experiment IDs, `penalty11_runs_table.csv`)
- Cluster paths, usernames, `cmd.txt`
- Failed evals, epoch-99 vs epoch-299 debugging, temporary copies

Those details can stay in a private note (e.g. `reproducibility_result.md`) if you need them later.

## Weights that match the paper

Do not assume last-epoch. Write down **which file** and **which command** produced the CSV, then put that file on HF.

Two cases we have:

**A. Table = `test()` at the end of `train` (PFΔ)**

Graphkit `train` runs `test()` after `fit()` on the **in-memory last-epoch weights**. `SaveBestModelStateDict` writes `best_model_state_dict.pt` on **best validation loss** and copies it at epochs 9, 49, 99, … — that is **not** the paper CSV unless val-loss was best at the end.

1. Prefer Lightning `checkpoints/last.ckpt`. Extract `state_dict` and save `last.pt`.
2. Spot-check `evaluate` of `last.pt` against the train-run `artifacts/test` CSVs. Expect ~0.001% relative noise, not 0.1–2%.
3. Put **those** files on HF, not the val-loss snapshots.
4. If a run stopped early, `last.ckpt` is still the paper weights; there may be no `epoch_299.pt`.
5. If a table row was an **eval of another task’s model** (PFΔ task 3.1 ← task 1.3), do not guess. Re-eval **last** vs **best/epoch_299** from the source task on the target data. Put whichever matches the CSV, and say so on the model card.

**B. Table = standalone `evaluate` of the training checkpoint (OPFData §5.2)**

The CSV is `evaluate --batch_size 512` on the training run’s `best_model_state_dict.pt` plus `normalizer_stats.pt`. Put those files on HF. Spot-check `evaluate` against `scripts/…/results/<grid>/seed*/metrics.csv` (GPU noise is enough). The train-run `artifacts/test` dump is **not** Table 5.

Do not tell readers “results will not match” if the HF file is already the one that produced the CSV.

## Hugging Face layout

Model repo (adapt names per section):

```
# PFΔ
task_<id>/seed{1,2,3}/last.pt
task_3.1/seed{1,2,3}/<the file that actually matched>
mlflow/<task>/<run_id>/…

# OPFData
<grid>/seed{42,3,17}/best_model_state_dict.pt
<grid>/seed{42,3,17}/normalizer_stats.pt
<grid>/seed{42,3,17}/metrics.csv          # eval dump; also on GitHub
splits/*.pt
mlflow/train/<grid>/seed<seed>/<run_id>/
mlflow/eval/<run_id>/
```

- Org: `gridfm`. Public once the section is ready to share.
- Do not duplicate checkpoints inside `mlflow/`. Test CSVs and plots there are enough.
- Dataset repos stay separate (`gridfm/pfdelta_task1.1`, `gridfm/opfdata_case14_ieee`, …).

## GitHub snapshot branch

The branch should contain:

- The code version that produced the runs
- The YAMLs that were actually used (paper seeds only)
- Plot/table scripts and the `results/` CSVs

Before you call the branch done, search it for:

- `/dccstor`, `/u/apu`, `bsub`, personal `mlflow_*` paths
- `generate_eval_commands.py`-style helpers that only work on the cluster
- Extra seeds/configs that were not in the paper
- `reproducibility_result.md` or other internal notes
- Cluster CSV harvests (`penalty11_runs_table.csv`, `sheet2_*.csv`)

Do not commit `aggregate_mlflow_metrics.py` or cluster eval helpers. The `results/` CSVs are enough; the HF `mlflow/` dump is the raw test logs.

## Checklist for a new section

- [ ] Public README follows the five sections above and has no cluster/MLflow-aggregation story
- [ ] Data is on HF; README download path matches `{data_path}/{network}/raw`
- [ ] Splits are either in the dataset or documented next to the checkpoints
- [ ] Code branch + configs linked; README says numbers are guaranteed on that branch only
- [ ] HF weights are the files that produced the CSV (last-epoch **or** val-best + `evaluate`, plus normalizer if needed), spot-checked vs CSVs
- [ ] Spot-check used the HF nested layout, not only cluster `processed/`
- [ ] HF `mlflow/` has the paper runs (train-only, or train+eval if the table is from `evaluate`)
- [ ] Model card matches the README (branch, configs, “what matches the table”, data_path)
- [ ] Snapshot branch has no personal paths, job launchers, or cluster harvest CSVs
