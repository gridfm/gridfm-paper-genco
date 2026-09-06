# Reproduce GENCO §5.4.2 (Power Flow on datakit)

This covers the PF **runtime-vs-grid-size** plot (`fig:gridsize_runtime_pf`), the **residual-vs-grid-size** plot (`fig:gridsize_vs_residuals_pf`), and **Table** `tab:pf_selected_genco_tradeoff`.

GENCO Base / Small / Tiny are trained **from scratch** on each grid (paper §5.4). Data is the public datakit PF **small** dumps, collected as [`finetune_pf_small`](https://huggingface.co/collections/gridfm/finetune-pf-small-697882b6276f4f9b10933bdc).

Weights, frozen configs, and metric CSVs for this section are **not published yet**. This README is the starting point: data links work; train / eval / figure commands will be filled in once the snapshot branch and checkpoints are up.

## 1. Table results

Not on GitHub yet. The paper numbers live in `GENCO/paper/00scaling.tex` (`tab:pf_selected_genco_tradeoff`) and the two PDFs under `paper/figures/runtime_analysis/`.

## 2. Data

Generated with `gridfm-datakit`. You do not need to regenerate. Hugging Face (parquet; graphkit expects `{data_path}/{network}/raw/`):

| Grid | Hugging Face |
| --- | --- |
| IEEE 14 | [gridfm/pf_small_case14_ieee](https://huggingface.co/datasets/gridfm/pf_small_case14_ieee) |
| IEEE 30 | [gridfm/pf_small_case30_ieee](https://huggingface.co/datasets/gridfm/pf_small_case30_ieee) |
| IEEE 57 | [gridfm/pf_small_case57_ieee](https://huggingface.co/datasets/gridfm/pf_small_case57_ieee) |
| IEEE 118 | [gridfm/pf_small_case118_ieee](https://huggingface.co/datasets/gridfm/pf_small_case118_ieee) |
| GOC 500 | [gridfm/pf_small_case500_goc](https://huggingface.co/datasets/gridfm/pf_small_case500_goc) |
| PEGASE 1354 | [gridfm/pf_small_case1354_pegase](https://huggingface.co/datasets/gridfm/pf_small_case1354_pegase) |
| GOC 2000 | [gridfm/pf_small_case2000_goc](https://huggingface.co/datasets/gridfm/pf_small_case2000_goc) |
| GOC 10000 | [gridfm/pf_small_case10000_goc](https://huggingface.co/datasets/gridfm/pf_small_case10000_goc) |

The paper scaling table uses 14 / 30 / 57 / 118 / 500 / 2000 / 10000. Base is reported through GOC 500; Small and Tiny through GOC 10000. Table 6 is **Tiny**.

```bash
pip install "huggingface_hub[cli]"
mkdir -p data/case118_ieee/raw
hf download gridfm/pf_small_case118_ieee --repo-type dataset --local-dir data/case118_ieee/raw
```

Same pattern for the other grids (`data/case14_ieee/raw`, `data/case500_goc/raw`, …). Then `--data_path data`.

## 3. Training and checkpoints

Not published yet. Paper setup: one model per grid, trained from scratch; hidden size 48 / 24 / 12 for Base / Small / Tiny.

Placeholder graphkit YAMLs (Base, `hidden_size: 48`, not a frozen paper snapshot): [`examples/config/HGNS_PF_datakit_case*.yaml`](https://github.com/gridfm/gridfm-graphkit/tree/genco-paper-repro/examples/config) on `genco-paper-repro`.

**Checkpoints:** none on Hugging Face yet (unlike [genco-pfdelta-base](https://huggingface.co/gridfm/genco-pfdelta-base) and [genco-opfdata-base](https://huggingface.co/gridfm/genco-opfdata-base)).

## 4. Reusing the saved model

Blocked on publishing weights. Shape will match the other sections:

```bash
gridfm_graphkit evaluate \
  --config path/to/yaml \
  --data_path data \
  --model_path path/to/last.pt
```

## 5. Figures

Blocked on committed CSVs and plot scripts. Paper artifacts:

- `paper/figures/runtime_analysis/runtime_comparison_from_raw_pf.pdf`
- `paper/figures/runtime_analysis/grid_scaling_active_residuals_pf.pdf`
- `paper/00scaling.tex` (`tab:pf_selected_genco_tradeoff`)
