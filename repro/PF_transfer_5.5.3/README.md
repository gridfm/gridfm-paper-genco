# Reproduce GENCO §5.5.3 (transfer to unseen grids)

This covers `fig:transfer`: mean active power-balance residual on IEEE 118 versus the number of grid-specific training samples, for a model fine-tuned from a subgrid-pretrained checkpoint and a model trained from scratch. DC-PF and the zero-shot pretrained model are the horizontal baselines.

## 1. Figure results

The numbers in the figure are on this branch:

[https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/pretraining_advantage/results](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/pretraining_advantage/results)

`scripts/pretraining_advantage/results/transfer_case118.csv` has one row per training size and initialization. `n_scenarios` is the cap in the training YAML. `type` is `scratch` or `finetune`. The finetune row with `n_scenarios` 0 is the zero-shot pretrained model. `dc_avg_active_res_mw` is the same DC-PF residual on every row.

The paper quotes these values: zero-shot 11.07 MW, DC-PF 2.30 MW, 1,000 samples fine-tune 1.93 MW and scratch 3.81 MW, 10,000 samples fine-tune 0.27 MW and scratch 0.92 MW. At 250,000 samples scratch (0.074 MW) is below fine-tune (0.091 MW).

## 2. Data

Fine-tuning and training from scratch use the datakit PF samples for `case118_ieee`. The YAML field `data.scenarios` is the number of scenarios for that point: 100, 1,000, 10,000, 20,000, 50,000, 100,000, and 250,000. Graphkit loads `{data_path}/case118_ieee/raw/*.parquet`.

The plotted residuals are not the test split of that training run. They are a later `evaluate` on a separate IEEE 118 set of 9,952 scenarios, using [`case118_eval.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_eval.yaml) (`test_ratio` 0.99). That evaluation set is not on Hugging Face.

Pretraining used 100 subgrids, with 10 validation and 10 test subgrids held out. Code and `examples/config/HGNS_PreTrain_subnets_100.yaml` (batch size 32, the value logged by run `1_h_100_32`) are on [`genco-paper-repro-pretraining`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pretraining). The raw parquet for those 120 grids, and their `.m` files under `casefiles/`, are in [gridfm/reproducibility-genco-pf-pretraining](https://huggingface.co/datasets/gridfm/reproducibility-genco-pf-pretraining). Training grids are `train_networks.txt`. Scenarios are regenerated with [`scripts/data_gen/launcher/pretraining/launch_pretraining_data_gen_pf.py`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/data_gen/launcher/pretraining/launch_pretraining_data_gen_pf.py) on datakit `genco-paper-repro`. For graphkit, `--data_path` is the download directory. `casefiles/` is not a grid; move it aside before training so it is not drawn into the 100-network set.

## 3. Training and checkpoints

Use [`genco-paper-repro`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro). The paper numbers were obtained on that line of code; `main` is under active development.

Scratch and fine-tune both use [`case118_train.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_train.yaml). [`launch_train_and_finetune.py`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/launch_train_and_finetune.py) is the launcher from the paper runs. It writes one config per size (100, 1,000, 10,000, 20,000, 50,000, 100,000, 250,000) and submits a from-scratch `train` and a `finetune` for each. The fine-tune loads the pretrained checkpoint above. Seed 0, hidden size 48, 200 epochs, batch size 64, `split_by_load_scenario_idx` true. The MLflow experiment name is `scaling_case118_a100_3`.

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

python scripts/pretraining_advantage/launch_train_and_finetune.py \
  scripts/pretraining_advantage/case118_train.yaml \
  case118
```

The checkpoint used for every fine-tune, and for the zero-shot line, is the finished pretraining run `1_h_100_32` (`11d08e25cdb64a0d8c62d6f6179355a6`, `best_model_state_dict.pt`) in experiment `pretraining_100_subnets`. It finished on 18 February 2026. The case 118 fine-tunes that feed the figure were submitted on 24 February 2026, and the 250,000-sample fine-tune on 26 February, by [`launch_train_and_finetune.py`](https://github.com/gridfm/gridfm-graphkit/blob/pretraining_advantage_march_2026/scripts/pretraining_advantage/launch_train_and_finetune.py) on branch [`pretraining_advantage_march_2026`](https://github.com/gridfm/gridfm-graphkit/tree/pretraining_advantage_march_2026). That script sets `--model_path` to this file. A second run with the same name `1_h_100_32` never finished, and `2_h_100_64` is a different checkpoint. Logged settings for the finished run match `HGNS_PreTrain_subnets_100.yaml` on `genco-paper-repro-pretraining`, including batch size 32. That task is not registered on `genco-paper-repro`. The checkpoint is not on Hugging Face.

## 4. Reusing the saved model

Evaluate each trained checkpoint, and the pretrained checkpoint for the zero-shot point, with the eval YAML and that run's `normalizer_stats.pt`. The pretrained run did not log a `normalizer_stats.pt`. Each scratch and fine-tune run did.

```bash
python scripts/pretraining_advantage/run_on_new_dataset.py \
  scripts/pretraining_advantage/case118_eval.yaml \
  /path/to/evaluation_pf \
  /path/to/scaling_case118_a100_3 \
  case118_eval
```

`data_eval` must contain `case118_ieee/raw/` for the 9,952-scenario evaluation set.

## 5. Figure

From the graphkit repo root:

```bash
python scripts/pretraining_advantage/plot_transfer.py
```

The script reads `scripts/pretraining_advantage/results/transfer_case118.csv` and writes `scripts/pretraining_advantage/figures/scratch_vs_finetune_active_residuals.pdf`.
