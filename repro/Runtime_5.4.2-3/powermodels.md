# Reproducing the GENCO paper (PowerModels runtime)

Classical-solver runtime matrix for the [GENCO paper](https://arxiv.org/abs/2608.09921): amortized per-instance wall time for AC-PF, DC-PF, AC-OPF, and DC-OPF, under an in-memory protocol (setup 1) and a from-disk protocol (setup 2).

The 56 committed CSVs are those paper raw results. Re-run the jobs only to regenerate them. GENCO GPU numbers are produced from [gridfm-graphkit](genco.md), not from this tree.

Clone the datakit repro branch and run every command below from that repository root:

```bash
git clone -b genco-paper-repro https://github.com/gridfm/gridfm-datakit.git
cd gridfm-datakit
```

## 1. Install

Any recent Julia is fine.

```bash
julia --project=scripts/runtime/pure_julia -e 'using Pkg; Pkg.instantiate()'
```

That uses [`pure_julia/Manifest.toml`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/pure_julia/Manifest.toml). The original timed jobs used Julia 1.12.6. Pins, Ipopt/MUMPS, threads, and hardware are in [`environment_versions.md`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/outputs_julia/full_matrix/environment_versions.md).

Setup 1 also needs the seven corrected `.m` files in `gridfm_datakit/grids/` (tracked on this branch).

## 2. Read the GENCO paper numbers

They live at `scripts/runtime/outputs_julia/full_matrix/`. Do not edit the 56 CSVs. `{small,large}/setup1/` is the in-memory sweep. `{small,large}/setup2/` is the from-disk sweep. `wall_at_best_p_*.csv` is derived from those sweeps.

Amortized per-instance runtime is `pf_elapsed_s / n_pfs`. Best `p` is the minimum of that ratio. Init, compile, and `/tmp` staging are outside `pf_elapsed_s`.

```bash
python scripts/runtime/pure_julia/summarize_wall_at_best_p.py --check
```

## 3. Re-run the matrix

Optional. `--resume` is on: delete a CSV to recompute that cell.

**Setup 1 (in-memory)** solves the corrected base `.m` case with [run_matrix.jl](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/pure_julia/run_matrix.jl). No scenario JSON.

```bash
julia --project=scripts/runtime/pure_julia scripts/runtime/pure_julia/run_matrix.jl --scope small --setup setup1
julia --project=scripts/runtime/pure_julia scripts/runtime/pure_julia/run_matrix.jl --scope large --setup setup1
```

**Setup 2 (from-disk)** parses one scenario per solve. The 10,000 corrected JSON files per grid are [gridfm/reproducibility-powermodels-setup2](https://huggingface.co/datasets/gridfm/reproducibility-powermodels-setup2).

```bash
hf download gridfm/reproducibility-powermodels-setup2 --repo-type dataset --local-dir /path/to/finetuning
export GRIDFM_DATA_BASE=/path/to/finetuning
julia --project=scripts/runtime/pure_julia scripts/runtime/pure_julia/run_matrix.jl --scope small --setup setup2
julia --project=scripts/runtime/pure_julia scripts/runtime/pure_julia/run_matrix.jl --scope large --setup setup2
```

Layout: `$GRIDFM_DATA_BASE/{pf,opf}/<network>/powermodels/scenario_*_corrected.json`. To build that JSON from parquet instead, run [batch_convert_finetune.py](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/convert/batch_convert_finetune.py) and then [run_correction.sh](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/pure_julia/run_correction.sh).

On LSF, [submit_matrix.sh](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/pure_julia/submit_matrix.sh) submits setup 1 and setup 2 (84 cores; 256G small / 960G large).

## Notes

- Ipopt: `max_iter=100`, `tol=1e-6`, linear solver MUMPS (default; not ma57). One BLAS thread per worker (`JULIA_NUM_THREADS=1` and matching OpenBLAS/OMP/MKL flags in `pure_julia/env.sh`).
- Original hardware: AMD EPYC 9634, 84 cores. Details: [`environment_versions.md`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/outputs_julia/full_matrix/environment_versions.md).
- Observed job RAM: small ~205 GB, large ~870 GB. See [`lsf_job_wall_times.md`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/outputs_julia/full_matrix/lsf_job_wall_times.md).
- Full parameter table: [`methodology_parameters.md`](https://github.com/gridfm/gridfm-datakit/blob/genco-paper-repro/scripts/runtime/outputs_julia/full_matrix/methodology_parameters.md).
