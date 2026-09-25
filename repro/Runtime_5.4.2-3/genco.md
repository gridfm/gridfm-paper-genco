# Reproducing the GENCO paper (GPU runtime)

PF inference timings for the [GENCO paper](https://arxiv.org/abs/2608.09921). The OPF GENCO curves use these same PF timings. There is no separate OPF GENCO benchmark. Figure 10 keeps the PowerModels OPF curves from [powermodels.md](powermodels.md).

Residual and optimality numbers stay in [PF_on_datakit_5.4.2](../PF_on_datakit_5.4.2/README.md) and [OPF_on_datakit_5.4.3](../OPF_on_datakit_5.4.3/README.md).

Both protocols read processed graphs, not raw parquet and not a trained checkpoint. The model is built at the config hidden size (tiny 12 / small 24 / base 48) with a random `baseMVA`. Wall time does not depend on the paper weights.

## 1. Install

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-datakit.git
cd gridfm-graphkit
pip install -e .
```

Clone the two repos next to each other. A rerun needs a CUDA GPU. Rebuilding the figures does not. Paper jobs used Python 3.12.9, PyTorch 2.8.0+cu128, and CUDA 12.8. The dump of that environment is `scripts/runtime/outputs_genco/environment_versions.md` on the graphkit branch.

## 2. Data

```text
$GENCO_DATA_PATH/<network>/processed/data_index_*.pt
```

The first 10,000 `data_index_*.pt` files per network are on Hugging Face as one uncompressed tar per grid: [`gridfm/reproducibility-genco-pf-processed`](https://huggingface.co/datasets/gridfm/reproducibility-genco-pf-processed).

```bash
hf download gridfm/reproducibility-genco-pf-processed --repo-type dataset \
    --local-dir /path/to/pf
for t in /path/to/pf/*.tar; do tar -xf "$t" -C /path/to/pf; done
export GENCO_DATA_PATH=/path/to/pf
```

Each `<network>.tar` contains `<network>/processed/data_index_0.pt` through `data_index_9999.pt`. The dataset currently has IEEE 14, 30, 57, 118 and GOC 500 and 2000. `case10000_goc.tar` is not there yet. The committed CSVs already include the GOC 10000 timings.

In-memory jobs preload those 10,000 graphs. From-disk jobs copy them to node-local `/tmp` and load inside `Dataset.get()`. Sample counts timed against that pool match the PowerModels matrix (IEEE 4M/3M/2M/2M, GOC 500k/50k/10k). `GENCO_PYTHON` defaults to `python` on `PATH`.

## 3. Read the paper numbers

Do not edit the CSVs. The GENCO metric is `outer_elapsed_ms / num_samples`. The best batch size is the minimum of that ratio. Rows with `status` other than `ok` are omitted. Paths are inside the graphkit clone.

| Directory | What it is |
| --- | --- |
| `scripts/runtime/genco_pf_in_memory/ieee/` | In-memory PF, IEEE 14/30/57/118. Batches from 64. Figure 8, Figure 10, the appendix batch-size figure, and the in-memory half of the loading figure |
| `scripts/runtime/genco_pf_in_memory/goc/` | In-memory PF, GOC 500/2000/10000. Batches from 16 |
| `scripts/runtime/genco_pf_from_disk/ieee/` | From-disk PF, IEEE. Same batch list |
| `scripts/runtime/genco_pf_from_disk/goc/` | From-disk PF, GOC. Same batch list |

Sample counts in those CSVs match the classical matrix: 4,000,000, 3,000,000, 2,000,000, 2,000,000, 500,000, 50,000, and 10,000.

PowerModels CSVs live at `gridfm-datakit/scripts/runtime/outputs_julia/full_matrix/`. Their metric is `pf_elapsed_s / n_pfs`.

## 4. Re-run the sweeps

Optional. The committed CSVs are the paper numbers. In-memory timing preloads 10,000 samples. From-disk timing cycles those samples through `Dataset.get()`.

- [run_pf_in_memory_ieee.py](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/runtime/run_pf_in_memory_ieee.py)
- [run_pf_in_memory_goc.py](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/runtime/run_pf_in_memory_goc.py)
- [run_pf_from_disk_ieee.py](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/runtime/run_pf_from_disk_ieee.py)
- [run_pf_from_disk_goc.py](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/runtime/run_pf_from_disk_goc.py)

```bash
python scripts/runtime/run_pf_in_memory_ieee.py --data-path "$GENCO_DATA_PATH"
python scripts/runtime/run_pf_in_memory_goc.py --data-path "$GENCO_DATA_PATH"
python scripts/runtime/run_pf_from_disk_ieee.py --data-path "$GENCO_DATA_PATH"
python scripts/runtime/run_pf_from_disk_goc.py --data-path "$GENCO_DATA_PATH"
```

On LSF, [submit_pf_matrix.sh](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/runtime/submit_pf_matrix.sh) submits those four scripts (1 exclusive H100, 128G, 40 slots).

## 5. Figures and tables

Set `POWERMODELS_MATRIX_ROOT` to the datakit `full_matrix` directory if the datakit clone is not beside this repo. From the graphkit repo:

```bash
export POWERMODELS_MATRIX_ROOT=../gridfm-datakit/scripts/runtime/outputs_julia/full_matrix
mkdir -p figures

python scripts/runtime/plot_runtime_comparison_from_raw_pf.py --output figures/runtime_comparison_from_raw_pf.pdf
python scripts/runtime/plot_runtime_comparison_from_raw_opf.py --output figures/runtime_comparison_from_raw_opf.pdf
python scripts/runtime/plot_pf_opf_benchmark_matrix_appendix.py --output figures/pf_opf_benchmark_matrix_appendix.pdf
python scripts/runtime/plot_juliacall_scaling_comparison.py --output figures/juliacall_scaling_comparison.pdf
python scripts/runtime/plot_loading_overhead_best_config.py --output figures/loading_overhead_best_config.pdf

python scripts/runtime/build_pf_tradeoff_table.py
python scripts/runtime/build_opf_tradeoff_table.py --model small
python scripts/runtime/build_loading_speedups_table.py
```

[plot_runtime_comparison_from_raw_opf.py](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/runtime/plot_runtime_comparison_from_raw_opf.py) reads the PF GENCO CSVs and the PowerModels OPF curves. [build_opf_tradeoff_table.py](https://github.com/gridfm/gridfm-graphkit/blob/genco-paper-repro/scripts/runtime/build_opf_tradeoff_table.py) uses those same GENCO times for the speedup columns and `scripts/datakit_opf/results/opf_scaling_aggregated.csv` for the gap and violation columns.
