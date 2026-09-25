# §5.1 reproducibility result

Date: 2026-09-06  
Code: `gridfm-graphkit` branch `genco-paper-repro-pfdelta`  
Paper artifacts: `GENCO/paper/figures/pf_delta/`

Layout on that branch (all under `scripts/pfdelta/`):

- plot/table scripts: `make_latex_tables.py`, `format_table_exponents.py`, `make_pbl_barplots.py`
- GENCO CSVs: `results/{1.1,1.2,1.3,2.3,3.1,4.1,4.2,4.3}/metrics_summary.csv`
- train YAMLs: `config/HGNS_PF_pfdelta_bs64_seed{1,2,3}.yaml`
- tables: `table_a_{8,9,10,11}_genco_exponent_fixed.tex`

Generated PDFs go to `scripts/pfdelta/figures/` and are untracked.

## What was reproduced

The **table and figure pipeline** from those CSVs, not a from-scratch train/eval.

Not re-run: Hugging Face download, conversion, `gridfm_graphkit train` / `evaluate`, or `aggregate_mlflow_metrics.py`.

## Pipeline

From the `genco-paper-repro-pfdelta` repo root:

```bash
python scripts/pfdelta/make_latex_tables.py
python scripts/pfdelta/format_table_exponents.py
python scripts/pfdelta/make_pbl_barplots.py --metric mean
python scripts/pfdelta/make_pbl_barplots.py --metric max
```

- `make_latex_tables.py` fills GENCO rows from `scripts/pfdelta/results/*/metrics_summary.csv` and inserts hardcoded PFNet / CANOS-PF / GNS / NR cells.
- `format_table_exponents.py` writes `table_a_*_genco_exponent_fixed.tex`.
- `make_pbl_barplots.py` reads `table_a_8` (mean) and `table_a_9` (max) and writes `scripts/pfdelta/figures/pbl_all_rows_{mean,max}.pdf`.

## Match against the paper

**Table 4** (`GENCO/paper/figures/pf_delta/pf_delta_a_10.tex` vs `scripts/pfdelta/table_a_10_genco_exponent_fixed.tex`): every numeric cell matches exactly (60 mean/std tokens), including CANOS-PF on IEEE 118 N / N-1 / N-2: `4.0e-2`, `5.6e-2`, `6.3e-2`. Cosmetic differences only: `GENCO Base` vs `\genco Base`, `GNS` vs `GNS-S`, table environment / `\resizebox`, `[p.u.]` in the header, and the published caption.

**GENCO rows vs CSVs:** `table_a_8` / `table_a_9` / `table_a_10` match `scripts/pfdelta/results/*/metrics_summary.csv` with no mismatches. Task 1.3 vs Task 3.1 IEEE 118 GENCO latex differs slightly in the CSVs (e.g. N-1 mean `4.6e-3` vs `4.5e-3`); barplots use 1.3, Table 4 uses 3.1, which matches how the paper files are split.

**Figures 4 and 18:** rebuilt from those tables. Rasterized at 200 DPI with `pdftoppm` and compared pixel-by-pixel (RGB): identical (mean 6978×1588, max 7328×1588; max |Δ| = 0). The PDF files are the same size but not byte-identical (metadata only).

**Not in this check:** training, evaluation, MLflow aggregation, or PFΔ baselines (frozen in `make_latex_tables.py`).
