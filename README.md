# STORM — reviewer code & runnable example

**STORM** (*Spatial Topology analysis of Recurrent Motifs*) discovers recurrent
spatial **motifs** in single-cell spatial transcriptomics (10x Xenium) of
colorectal cancer and patient-matched normal tissue. A GATv2-based **variational
graph autoencoder** learns a per-cell *niche embedding* from the spatial
neighbourhood graph; those embeddings are clustered into motifs, which are then
characterized biologically and compared across conditions.

This repository is a **self-contained, reviewer-facing extract** of the STORM
codebase. It contains the **core model-training code** and a few **core
analyses** as Jupyter notebooks, plus a committed **synthetic example dataset** so
the whole pipeline runs end-to-end in minutes on a laptop — no patient data, no
absolute paths, no cluster required. The notebooks are committed **already
executed** so the figures are visible without running anything.

> This is not the full study codebase. It is a minimal, runnable demonstration of
> the method and its main analysis steps for peer review.

## What's here

```
.
├── README.md
├── requirements.txt
├── data/
│   └── example/                     # 6 synthetic samples (3 patients x CA/NL)
│       └── sample_{CA,NL}<patient>C.h5ad
├── notebooks/
│   ├── 01_train_storm_model.ipynb        # CORE: GATv2-VGAE training + motif inference
│   └── 02_motif_characterization.ipynb   # composition, marker genes, UMAP, CA-vs-NL abundance
└── outputs/                         # written by the notebooks (git-ignored)
```

## The example data

`data/example/` is **fully synthetic** (fixed seed). It mimics the structure STORM
expects — one `.h5ad` per sample, raw counts over a shared 60-gene panel, spatial
coordinates in `obs['x_centroid','y_centroid']`, and condition/patient encoded in
the filename. Cancer samples have a tumour core wrapped in a fibroblast/DFB
barrier with immune cells excluded to the periphery; normal samples have
epithelial crypts in relaxed stroma — so the motifs and the cancer-vs-normal
contrast are biologically plausible *and* recoverable from the toy data. It is not
real patient data and is not meant for any biological conclusions.

## Quick start

```bash
# 1. Environment (a CPU-only PyTorch build is fine)
pip install -r requirements.txt

# 2. Launch Jupyter from the repo root and run the notebooks in order
jupyter lab
#   notebooks/01_train_storm_model.ipynb     -> trains model, writes outputs/
#   notebooks/02_motif_characterization.ipynb
```

The notebooks auto-`chdir` to the repo root, so they work whether you launch
Jupyter from the root or from inside `notebooks/`. They detect a GPU
automatically and fall back to CPU (mixed precision is used on GPU only); the
example cohort trains in a couple of minutes either way.

## The pipeline (notebook 01)

1. **Preprocess** each sample: QC filter, library-size normalize, `log1p`, z-scale,
   align to a shared gene panel.
2. **Build a spatial graph** per sample: k-nearest-neighbour (k=10) over cell
   centroids, edges weighted by inverse intercellular distance.
3. **Train** a GATv2 VGAE with spatial-patch sampling — joint feature- and
   structure-reconstruction with a KL term; gradient clipping and logstd/logit
   clamps for stability.
4. **Infer**: encode every cell to a 20-D niche embedding (posterior mean) in
   memory-bounded 2-hop tiles, then cluster with MiniBatch *k*-means into motifs.
5. **Export** embeddings, motif labels, per-edge attention, and figures to
   `outputs/`.

> **Demo vs. paper hyper-parameters.** `Config` in notebook 01 is scaled down for a
> fast demo (fewer epochs/patches, 8 motifs). The values used in the study are
> noted inline (e.g. 100 epochs, 10 motifs). The model architecture is unchanged.

## The analyses

- **`02_motif_characterization`** — cell-type composition per motif; motif marker
  genes (Wilcoxon rank-sum + Benjamini–Hochberg FDR); UMAP of niche embeddings;
  per-motif cancer-vs-normal abundance (Mann–Whitney *U*, FDR-corrected).

## Notes for reviewers

- Everything is reproducible from a fixed seed; no network access or external data
  is required.
- `outputs/` is git-ignored (regenerable); the committed notebooks already embed
  the figures and tables.
- On some HPC setups a torch/MKL OpenMP clash raises an `iJIT_NotifyEvent` import
  error — if so, `export LD_PRELOAD=<your-env>/lib/libiomp5.so` before launching
  Jupyter. Not needed on a clean `pip` install.
