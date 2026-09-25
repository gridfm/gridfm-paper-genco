# Reproduce GENCO §5.5.3 (transfer to unseen grids)

This covers `fig:transfer`: mean active power-balance residual on IEEE 118 versus the number of grid-specific training samples, for a model fine-tuned from a subgrid-pretrained checkpoint and a model trained from scratch. DC-PF and the zero-shot pretrained model are the horizontal baselines.

Code for scratch, fine-tune, and eval is on [`genco-paper-repro`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro). Pretraining is on [`genco-paper-repro-pretraining`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro-pretraining).

## Pretraining

### Data

100 training subgrids, 10 validation subgrids, and 10 test subgrids. The raw parquet and the `.m` files are in [gridfm/reproducibility-genco-pf-pretraining](https://huggingface.co/datasets/gridfm/reproducibility-genco-pf-pretraining). Training names are `train_networks.txt`. Validation and test names are `validation_networks.txt` and `test_networks.txt`.

```bash
hf download gridfm/reproducibility-genco-pf-pretraining --repo-type dataset --local-dir data_pretrain
mv data_pretrain/casefiles casefiles
```

`--data_path` is `data_pretrain`. Move `casefiles/` out first. Graphkit treats every top-level directory as a grid, and `casefiles/` would enter the draw. Scenarios can be rebuilt with [`launch_pretraining_data_gen_pf.py`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/data_gen/launcher/pretraining/launch_pretraining_data_gen_pf.py) on datakit `genco-paper-repro`.

### Training

[`HGNS_PreTrain_subnets_100.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro-pretraining/examples/config/HGNS_PreTrain_subnets_100.yaml): 100 networks, seed 200, 300 epochs, batch size 32. This task is not registered on `genco-paper-repro`.

```bash
git clone -b genco-paper-repro-pretraining https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

gridfm_graphkit train \
  --config examples/config/HGNS_PreTrain_subnets_100.yaml \
  --data_path data_pretrain
```

### Saved pretrain model

The finished run is `1_h_100_32`. Fine-tuning and the zero-shot line both load `best_model_state_dict.pt` from that run. This checkpoint is not on Hugging Face yet.

## Fine-tuning

### Data

Datakit PF samples for `case118_ieee`. Graphkit loads `{data_path}/case118_ieee/raw/*.parquet`. `data.scenarios` in the YAML is the training size: 100, 1,000, 10,000, 20,000, 50,000, 100,000, or 250,000.

### Training from scratch and fine-tuning

Both use [`case118_train.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_train.yaml). Seed 0, hidden size 48, 200 epochs, batch size 64. Set `data.scenarios` to the size for that point. Fine-tuning passes the pretrained `best_model_state_dict.pt`.

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
cd gridfm-graphkit
pip install -e .

gridfm_graphkit train \
  --config scripts/pretraining_advantage/case118_train.yaml \
  --data_path data

gridfm_graphkit finetune \
  --config scripts/pretraining_advantage/case118_train.yaml \
  --data_path data \
  --model_path pretrained/best_model_state_dict.pt
```

### Saved checkpoints

Each scratch run and each fine-tune run writes `best_model_state_dict.pt` and `normalizer_stats.pt`. The pretrained checkpoint has no `normalizer_stats.pt`. These case 118 checkpoints are not on Hugging Face yet.

## Evaluation of fine-tuned and trained-from-scratch models

### Data

A separate IEEE 118 set of 9,952 scenarios, not the test split of the training run. The plotted residuals come from this set. It is not on Hugging Face yet. Place it at `data_eval/case118_ieee/raw/`.

### Eval script using checkpoints

[`case118_eval.yaml`](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/pretraining_advantage/case118_eval.yaml) sets `test_ratio` to 0.99. Pass that run's checkpoint and its `normalizer_stats.pt`.

```bash
gridfm_graphkit evaluate \
  --config scripts/pretraining_advantage/case118_eval.yaml \
  --data_path data_eval \
  --model_path case118_1000/best_model_state_dict.pt \
  --normalizer_stats case118_1000/normalizer_stats.pt \
  --compute_dc_ac_metrics
```

## Figure

`scripts/pretraining_advantage/results/transfer_case118.csv` has one row per training size and initialization. `type` is `scratch` or `finetune`. The finetune row with `n_scenarios` 0 is the zero-shot pretrained model. The paper quotes zero-shot 11.07 MW, DC-PF 2.30 MW, 1,000 samples fine-tune 1.93 MW and scratch 3.81 MW, 10,000 samples fine-tune 0.27 MW and scratch 0.92 MW. At 250,000 samples scratch (0.074 MW) is below fine-tune (0.091 MW).

```bash
python scripts/pretraining_advantage/plot_transfer.py
```

The script writes `scripts/pretraining_advantage/figures/scratch_vs_finetune_active_residuals.pdf`.
