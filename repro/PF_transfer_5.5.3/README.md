# Reproduce GENCO §5.5.3 (transfer to unseen grids)

This covers `fig:transfer`: mean active power-balance residual on IEEE 118 versus the number of grid-specific training samples, for a model fine-tuned from a subgrid-pretrained checkpoint and a model trained from scratch. DC-PF and the zero-shot pretrained model are the horizontal baselines.

## 1. Figure results

The numbers in the figure are on this branch:

[https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/pretraining_advantage/results](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/pretraining_advantage/results)

`scripts/pretraining_advantage/results/transfer_case118.csv` has one row per training size and initialization. `n_scenarios` is the cap in the training YAML. `type` is `scratch` or `finetune`. The finetune row with `n_scenarios` 0 is the zero-shot pretrained model. `dc_avg_active_res_mw` is the same DC-PF residual on every row.

The paper quotes these values: zero-shot 11.07 MW, DC-PF 2.30 MW, 1,000 samples fine-tune 1.93 MW and scratch 3.81 MW, 10,000 samples fine-tune 0.27 MW and scratch 0.92 MW. At 250,000 samples scratch (0.074 MW) is below fine-tune (0.091 MW).

## 2. Data

Fine-tuning and training from scratch use the datakit PF samples for `case118_ieee`. The YAML field `data.scenarios` is the number of scenarios for that point: 100, 1,000, 10,000, 20,000, 50,000, 100,000, and 250,000. Graphkit loads `{data_path}/case118_ieee/raw/*.parquet`.

The plotted residuals are not the test split of that training run. They are a later `evaluate` on a separate IEEE 118 set of 9,952 scenarios, using [`case118_eval.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_eval.yaml) (`test_ratio` 0.99).

Pretraining used 100 subgrids, with 10 validation and 10 test subgrids held out. Code and `examples/config/HGNS_PreTrain_subnets_100.yaml` (batch size 32, the value logged by run `1_h_100_32`) are on [`genco-paper-repro-pretraining`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pretraining). The raw parquet for those 120 grids, and their `.m` files under `casefiles/`, are in [gridfm/reproducibility-genco-pf-pretraining](https://huggingface.co/datasets/gridfm/reproducibility-genco-pf-pretraining). Training grids are `train_networks.txt`. Scenarios are regenerated with [`scripts/data_gen/launcher/pretraining/launch_pretraining_data_gen_pf.py`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/data_gen/launcher/pretraining/launch_pretraining_data_gen_pf.py) on datakit `genco-paper-repro`. For graphkit, `--data_path` is the download directory. `casefiles/` is not a grid; move it aside before training so it is not drawn into the 100-network set.

## 3. Training and checkpoints

Use [`genco-paper-repro`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro). The paper numbers were obtained on that line of code; `main` is under active development.

Scratch and fine-tune both use [`case118_train.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_train.yaml). Set `data.scenarios` to the training size for that point: 100, 1,000, 10,000, 20,000, 50,000, 100,000, or 250,000. Seed 0, hidden size 48, 200 epochs, batch size 64.

Pretraining uses [`HGNS_PreTrain_subnets_100.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro-pretraining/examples/config/HGNS_PreTrain_subnets_100.yaml) on [`genco-paper-repro-pretraining`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pretraining). That task is not registered on `genco-paper-repro`. The finished run is `1_h_100_32`, checkpoint `best_model_state_dict.pt`. Every fine-tune and the zero-shot line load that file.

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

# from scratch
gridfm_graphkit train \
  --config scripts/pretraining_advantage/case118_train.yaml \
  --data_path data

# fine-tune
gridfm_graphkit finetune \
  --config scripts/pretraining_advantage/case118_train.yaml \
  --data_path data \
  --model_path pretrained/best_model_state_dict.pt
```

## 4. Reusing the saved model

Score a checkpoint with [`case118_eval.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_eval.yaml). `data_eval` must contain `case118_ieee/raw/` for the 9,952-scenario evaluation set. Each scratch and fine-tune run has a `normalizer_stats.pt`. The pretrained run does not.

```bash
gridfm_graphkit evaluate \
  --config scripts/pretraining_advantage/case118_eval.yaml \
  --data_path data_eval \
  --model_path case118_1000/best_model_state_dict.pt \
  --normalizer_stats case118_1000/normalizer_stats.pt \
  --compute_dc_ac_metrics
```

## 5. Figure

From the graphkit repo root:

```bash
python scripts/pretraining_advantage/plot_transfer.py
```

The script reads `scripts/pretraining_advantage/results/transfer_case118.csv` and writes `scripts/pretraining_advantage/figures/scratch_vs_finetune_active_residuals.pdf`.
