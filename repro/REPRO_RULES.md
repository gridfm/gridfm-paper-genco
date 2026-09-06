# GENCO reproducibility conventions

This file lives in the personal GENCO repo under `repro/`. It is for whoever writes the next section’s public README (e.g. `repro/PF_on_pfdelta_5.1/README.md`), not for readers of the paper.

Name each folder `<TASK>_on_<dataset>_<section>` (e.g. `PF_on_pfdelta_5.1`, `OPF_on_opfdata_5.2`, `PF_on_datakit_5.4.2`).

The public README must be usable by someone with GitHub + Hugging Face only. No cluster, no LSF, no personal paths.

## Where things go

| What | Where | Notes |
| --- | --- | --- |
| Frozen code + configs used in the paper | Public GitHub **branch** on `gridfm-graphkit` and/or `gridfm-datakit` | Snapshot branch, not `main`. Name it `genco-paper-repro-<topic>` (e.g. `genco-paper-repro-pfdelta`). |
| Table/figure **scripts** | Same GitHub branch | e.g. `scripts/pfdelta/make_latex_tables.py` |
| Paper **metric CSVs** (small) | Same GitHub branch (e.g. `scripts/pfdelta/results/`) | These are the GENCO numbers in the tables. |
| Converted / split **datasets** | Hugging Face **dataset** repos | Public if the source license allows. Point the README at these; do not ask people to start from cluster copies. |
| **Weights that reproduce the tables** | Hugging Face **model** repo (org `gridfm`) | Public once the section is ready. Prefer a name like `gridfm/genco-<topic>-base`. |
| Paper-seed **MLflow runs** whose `artifacts/test` match the CSVs | Same HF model repo, subdirectory `mlflow/` | Only the runs that went into the CSVs. No extra evals, no other seeds. |
| Public **README** for that section | Personal GENCO repo `repro/<section>/README.md` (and later the paper repo if you copy it) | Short. Links out. Does not duplicate code. |
| This conventions file | Personal GENCO repo `repro/REPRO_RULES.md` | Not for paper readers. |

**Do not put on GitHub**

- Checkpoints (`.pt`, `.ckpt`)
- Raw or converted datasets
- Full MLflow trees
- Cluster job launchers, `bsub` scripts, `cmd.txt`
- Scripts with hardcoded `/dccstor/...` or `/u/apu/...` paths
- Intermediate HTML/PDF dumps that are not the paper figures

**Do not put on Hugging Face**

- Source code (that is GitHub)
- Plot scripts (GitHub)
- Anything that is not needed to load data or load weights

The HF **model card** must repeat: GitHub branch, YAML configs, which weight files match the paper, and any exception (e.g. task 3.1).

## Public README: order and tone

Keep it short. Same order every time:

1. **Where the table/figure numbers are** — GitHub CSVs (e.g. `scripts/pfdelta/results/`), which file is which table.
2. **Data** — how it was obtained (one or two sentences + script links), then the HF dataset links. A download command. Say they do not need to reconvert.
3. **Training** — branch, configs (seeds), one `train` command. Then the HF checkpoint link and the layout (`task_…/seed…/last.pt`).
4. **Reusing the saved model** — one `evaluate` command. Be explicit if the saved file is **not** what the paper logged (see below).
5. **Figures** — the plot/table scripts and what they write.

Baselines taken from another paper: say so in one line (not retrained here).

### Do not mention in the public README

- How jobs were launched (`bsub`, LSF, queue names, GPU flags)
- How CSVs were built from MLflow (`aggregate_mlflow_metrics.py`, experiment IDs)
- Cluster paths, usernames, `cmd.txt`
- Failed evals, epoch-99 vs epoch-299 debugging, temporary copies

Those details can stay in a private note (e.g. `reproducibility_result.md`) if you need them later.

## Weights that match the paper

Graphkit `train` runs `test()` **after** `fit()`, on the **in-memory last-epoch weights**. Those are what went into the paper CSVs.

`SaveBestModelStateDict` writes `best_model_state_dict.pt` on **best validation loss**, and copies it at epochs 9, 49, 99, …, 299 as `best_model_state_dict.epoch_N.pt`. That copy is **not** the last-epoch weights unless val-loss happened to be best at the end.

So:

1. Prefer Lightning `checkpoints/last.ckpt`. Extract `state_dict` and save `last.pt`.
2. Spot-check `evaluate` of `last.pt` against the train-run `artifacts/test` CSVs (mean and max PBE). Expect ~0.001% relative noise, not 0.1–2%.
3. Put **those** files on HF, not the val-loss snapshots.
4. If a run stopped early, `last.ckpt` is still the paper weights; there may be no `epoch_299.pt`.
5. If a table row was an **eval of another task’s model** (PFΔ task 3.1 ← task 1.3), do not guess. Re-eval **last** vs **best/epoch_299** from the source task on the target data. Put whichever matches the CSV, and say so on the model card.

Do not tell readers “results will not match” if you have already replaced HF with `last.pt`.

## Hugging Face layout

Model repo (example: `gridfm/genco-pfdelta-base`):

```
task_<id>/seed{1,2,3}/last.pt          # trained tasks; matches paper CSVs
task_3.1/seed{1,2,3}/<the file that actually matched>
mlflow/<task>/<run_id>/…               # paper-seed runs only
README.md                              # branch, configs, exceptions
```

- Org: `gridfm`. Public once the section is ready to share.
- Do not duplicate 80 MB checkpoints inside `mlflow/` if they are already at `task_*/`. Test CSVs in `mlflow/` are enough to prove the tables.
- Dataset repos stay separate (`gridfm/pfdelta_task1.1`, …).

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

Do not commit `aggregate_mlflow_metrics.py` or cluster eval helpers. The `results/` CSVs are enough; the HF `mlflow/` dump is the raw test logs.

## Checklist for a new section

- [ ] Public README follows the five sections above and has no cluster/MLflow-aggregation story
- [ ] Data is on HF; README links it
- [ ] Code branch + configs linked
- [ ] HF weights are last-epoch (or the documented exception), spot-checked vs CSVs
- [ ] HF `mlflow/` has only the paper-seed runs
- [ ] Model card matches the README (branch, configs, “what matches the table”)
- [ ] Snapshot branch has no personal paths or job launchers
