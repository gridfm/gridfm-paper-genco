# Reproduce GENCO §5.4 runtime

This covers the runtime figures and speedup tables in section 5.4 and the appendix. Residual and optimality numbers stay in [PF_on_datakit_5.4.2](../PF_on_datakit_5.4.2/README.md) and [OPF_on_datakit_5.4.3](../OPF_on_datakit_5.4.3/README.md).

Classical AC-PF, DC-PF, AC-OPF, and DC-OPF timings are not in this repo. They are the committed CSVs in [gridfm-datakit `scripts/runtime`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/README.md). GENCO timings are on [`genco-paper-repro`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/scripts/runtime). Figure 8 and Figure 10 use that same GENCO sweep. The OPF figure keeps the PowerModels OPF curves.

## 1. Install

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-graphkit.git
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-datakit.git
cd gridfm-graphkit
pip install -e .
```

Clone the two repos next to each other. A rerun needs a CUDA GPU. Rebuilding the figures does not.

## 2. Read the paper numbers

Do not edit the CSVs. The GENCO metric is `outer_elapsed_ms / num_samples`. The best batch size is the minimum of that ratio. Rows with `status` other than `ok` are omitted.

| Directory | What it is |
| --- | --- |
| `scripts/runtime/genco_pf_in_memory/ieee/` | In-memory PF, IEEE 14/30/57/118. Batches from 64. Figure 8, Figure 10, the appendix batch-size figure, and the in-memory half of the loading figure |
| `scripts/runtime/genco_pf_in_memory/goc/` | In-memory PF, GOC 500/2000/10000. Batches from 16 |
| `scripts/runtime/genco_pf_from_disk/ieee/` | From-disk PF, IEEE. Same batch list |
| `scripts/runtime/genco_pf_from_disk/goc/` | From-disk PF, GOC. Same batch list |

Sample counts in those CSVs match the classical matrix: 4,000,000, 3,000,000, 2,000,000, 2,000,000, 500,000, 50,000, and 10,000.

PowerModels CSVs live at `gridfm-datakit/scripts/runtime/outputs_julia/full_matrix/`. Their metric is `pf_elapsed_s / n_pfs`.

## 3. Re-run the GENCO sweeps

Optional. The committed CSVs are the paper numbers.

Download one grid, nested the way graphkit expects:

```bash
mkdir -p data/case118_ieee/raw
hf download gridfm/pf_small_case118_ieee --repo-type dataset --local-dir data/case118_ieee/raw
```

Same pattern for `case14_ieee`, `case30_ieee`, `case57_ieee`, `case500_goc`, `case2000_goc`, and `case10000_goc`. Then, from the graphkit repo:

```bash
python scripts/runtime/run_pf_in_memory_ieee.py --data-path data
python scripts/runtime/run_pf_in_memory_goc.py --data-path data
python scripts/runtime/run_pf_from_disk_ieee.py --data-path data
python scripts/runtime/run_pf_from_disk_goc.py --data-path data
```

Each launcher writes the directory in the table above. In-memory timing preloads 10,000 samples. From-disk timing cycles those 10,000 samples through `Dataset.get()`. Compile mode is `reduce-overhead`, with 20 warmup batches and 32 loader workers.

## 4. Figures and tables

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

`plot_runtime_comparison_from_raw_opf.py` reads the PF GENCO CSVs and the PowerModels OPF curves. `build_opf_tradeoff_table.py` uses those same GENCO times for the speedup columns and `scripts/datakit_opf/results/opf_scaling_aggregated.csv` for the gap and violation columns.
