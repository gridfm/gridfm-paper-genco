# Reproduce GENCO §5.5.3 (transfer to unseen grids)

This covers `fig:transfer`: mean active power-balance residual on IEEE 118 versus the number of grid-specific training samples, for a model fine-tuned from a subgrid-pretrained checkpoint and a model trained from scratch. DC-PF and the zero-shot pretrained model are the horizontal baselines.

Do the steps in this order. Fine-tuning needs the pretrained checkpoint. Evaluation needs the scratch and fine-tune checkpoints. The figure is plotted from the evaluation metrics.

Checkpoints, the training and eval MLflow runs, and the eval parquet are in [gridfm/genco-pf-transfer-base](https://huggingface.co/gridfm/genco-pf-transfer-base). Each checkpoint folder contains the YAML and the shell command used for that run.

## Pretraining

### Data

100 training subgrids, 10 validation subgrids, and 10 test subgrids. Download [gridfm/reproducibility-genco-pf-pretraining](https://huggingface.co/datasets/gridfm/reproducibility-genco-pf-pretraining). Training names are `train_networks.txt`. Validation and test names are `validation_networks.txt` and `test_networks.txt`.

```bash
hf download gridfm/reproducibility-genco-pf-pretraining --repo-type dataset --local-dir data_pretrain
mv data_pretrain/casefiles casefiles
```

`--data_path` is `data_pretrain`. Move `casefiles/` out before training. Graphkit treats every top-level directory as a grid, so `casefiles/` would enter the draw. To rebuild the scenarios from the `.m` files, use [`launch_pretraining_data_gen_pf.py`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/data_gen/launcher/pretraining/launch_pretraining_data_gen_pf.py) on datakit `genco-paper-repro`.

### Training

Clone [`genco-paper-repro-pretraining`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pretraining) and train with [`HGNS_PreTrain_subnets_100.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro-pretraining/examples/config/HGNS_PreTrain_subnets_100.yaml): 100 networks, seed 200, 300 epochs, batch size 32. This task is not registered on `genco-paper-repro`.

```bash
git clone -b genco-paper-repro-pretraining https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

gridfm_graphkit train \
  --config examples/config/HGNS_PreTrain_subnets_100.yaml \
  --data_path data_pretrain
```

### Saved pretrain model

The paper checkpoint is run `1_h_100_32`, file `pretraining/best_model_state_dict.pt`. The same folder contains `HGNS_PreTrain_subnets_100.yaml` and `train.sh`. The MLflow run is `mlflow/pretrain/1_h_100_32`. There is no `normalizer_stats.pt`.

```bash
hf download gridfm/genco-pf-transfer-base --include "pretraining/**" --local-dir genco-pf-transfer-base
```

## Fine-tuning

### Data

Datakit PF samples for `case118_ieee`. Graphkit loads `{data_path}/case118_ieee/raw/*.parquet`. This training parquet is not on Hugging Face.

### Training from scratch and fine-tuning

Use [`genco-paper-repro`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro). One YAML per training size, with `data.scenarios` already set. Seed 0, hidden size 48, 200 epochs, batch size 64, 32 workers. Scratch calls `train`. Fine-tuning calls `finetune` and passes the pretrained checkpoint from the previous section.

| Scenarios | Config |
| --- | --- |
| 100 | [`case118_100.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/configs/case118_100.yaml) |
| 1,000 | [`case118_1000.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/configs/case118_1000.yaml) |
| 10,000 | [`case118_10000.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/configs/case118_10000.yaml) |
| 20,000 | [`case118_20000.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/configs/case118_20000.yaml) |
| 50,000 | [`case118_50000.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/configs/case118_50000.yaml) |
| 100,000 | [`case118_100000.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/configs/case118_100000.yaml) |
| 250,000 | [`case118_250000.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/configs/case118_250000.yaml) |

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

gridfm_graphkit train \
  --config scripts/pretraining_advantage/configs/case118_1000.yaml \
  --data_path data

gridfm_graphkit finetune \
  --config scripts/pretraining_advantage/configs/case118_1000.yaml \
  --data_path data \
  --model_path genco-pf-transfer-base/pretraining/best_model_state_dict.pt
```

Repeat with the other six configs.

### Saved checkpoints

`scratch/case118_<N>/` holds the from-scratch `best_model_state_dict.pt`, `normalizer_stats.pt`, `case118_<N>.yaml`, and `train.sh`. `finetune/case118_<N>/` holds the same files for the fine-tune, with `finetune.sh` instead of `train.sh`. Training MLflow runs are under `mlflow/train/case118_<N>` and `mlflow/train/finetune_case118_<N>`.

## Evaluation of fine-tuned and trained-from-scratch models

### Data

A separate IEEE 118 set of 9,952 scenarios, not the test split of the training run. It is `data/case118_ieee/raw/` in the model repo.

```bash
hf download gridfm/genco-pf-transfer-base --include "data/**" --local-dir genco-pf-transfer-base
```

`--data_path` is `genco-pf-transfer-base/data`.

### Eval script using checkpoints

[`case118_eval.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_eval.yaml) sets `test_ratio` to 0.99. Pass the checkpoint and the `normalizer_stats.pt` from the same folder. Eval MLflow runs are under `mlflow/eval/`. The zero-shot run is `mlflow/eval/finetune_case118_0_eval`. The pretrained checkpoint has no normalizer file, so that eval does not pass `--normalizer_stats`.

```bash
gridfm_graphkit evaluate \
  --config scripts/pretraining_advantage/case118_eval.yaml \
  --data_path genco-pf-transfer-base/data \
  --model_path genco-pf-transfer-base/finetune/case118_1000/best_model_state_dict.pt \
  --normalizer_stats genco-pf-transfer-base/finetune/case118_1000/normalizer_stats.pt \
  --compute_dc_ac_metrics
```

Repeat for `scratch/case118_<N>` and `finetune/case118_<N>`.

## Figure

`scripts/pretraining_advantage/results/transfer_case118.csv` has one row per training size and initialization. `type` is `scratch` or `finetune`. The finetune row with `n_scenarios` 0 is the zero-shot pretrained model. The paper quotes zero-shot 11.07 MW, DC-PF 2.30 MW, 1,000 samples fine-tune 1.93 MW and scratch 3.81 MW, 10,000 samples fine-tune 0.27 MW and scratch 0.92 MW. At 250,000 samples scratch (0.074 MW) is below fine-tune (0.091 MW).

```bash
python scripts/pretraining_advantage/plot_transfer.py
```

The script writes `scripts/pretraining_advantage/figures/scratch_vs_finetune_active_residuals.pdf`.
