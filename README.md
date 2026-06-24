# PhyloTune: An Efficient Method to Accelerate Phylogenetic Updates Using a Pretrained DNA Language Model

[![Project](https://img.shields.io/badge/Code-Github-purple?style=flat-square)](https://github.com/danruod/PhyloTune)
[![Code License](https://img.shields.io/badge/Code%20License-Apache_2.0-green.svg)](https://github.com/danruod/PhyloTune/blob/main/LICENSE)
[![Data License](https://img.shields.io/badge/Data%20License-CC%20By%20NC%204.0-red.svg)](https://github.com/danruod/PhyloTune/blob/main/DATA_LICENSE)


> **Authors**: Danruo Deng, Wuqin Xu*, Bian Wu, Hans Peter Comes, Yu Feng, Pan Li*, Jinfang Zheng*, Guangyong Chen*, and Pheng-Ann Heng.  
 **Affiliations**: CUHK, Zhejiang Lab, Salzburg Universit, Chengdu Institute of Biology (CAS), Zhejiang University, Hangzhou Institute of Medicine (CAS).

**PhyloTune** is a novel method designed to accelerate phylogenetic tree updates by reducing the number and length of input sequences required for alignment and tree analysis. It achieves this by identifying the **smallest taxonomic unit** for newly collected sequences on a given phylogenetic tree and extracting the **high-attention regions** of sequences within that smallest taxonomic unit.

<!-- Understanding the phylogenetic relationships among species is crucial for comprehending major evolutionary transitions, serving as the foundation for many biological studies. Despite the ever-growing volume of sequence data providing a significant opportunity for biological research, constructing reliable phylogenetic trees effectively becomes more challenging for current analytical methods. In this study, we introduce a novel solution to accelerate phylogenetic updates using a pretrained DNA language model. Our approach identifies the taxonomic unit of a newly collected sequence using existing taxonomic classification systems and updates the corresponding subtree, akin to surgical corrections on a given phylogenetic tree. Specifically, we leverage a pretrained BERT network to obtain high-dimensional sequence representations, which are used not only to determine the subtree to be updated but also identify potentially valuable regions for subtree construction. We demonstrate the effectiveness of our method, named PhyloTune, through experiments on our established Plant dataset (focused on Embryophyta) and microbial data (focused on Bordetella genus). Our findings provide the first evidence that phylogenetic trees can be constructed by automatically selecting the most informative regions of sequences, without manual selection of molecular markers. This discovery offers a robust guide for further research into the functional aspects of different regions of DNA sequences, enriching our understanding of biology. -->

<img src="PhyloTune.png" alt="PhyloTune" width="80%"/>



## 🛠️ System Requirements

### Hardware

A GPU is recommended for model inference. CPU-only execution is supported but may be slow.

### Software

- Python 3.10+ 
- PyTorch 2.x
- Full dependency list: [environment.yml](environment.yml) (conda) or [requirements.txt](requirements.txt) (pip)


## 📑 Installation

```bash
git clone https://github.com/danruod/PhyloTune.git
cd PhyloTune/

# Option A: conda (recommended)
conda env create -f environment.yml
conda activate phylo

# Option B: pip
pip install -r requirements.txt
```

### Download Data & Models

The repository only contains source code. You must download datasets and model checkpoints separately:

| Resource | Link | Target Directory |
|---|---|---|
| Datasets (Plant, Bordetella, Simulated) | [Google Drive](https://drive.google.com/drive/folders/1NwwAJog1_i_2X_sZKdQPaNeA0UnhPcj2) | `./datasets/` |
| Model checkpoints (Plant_dnabert, Bordetella_dnaberts) | [Google Drive](https://drive.google.com/drive/folders/1QdMSYhDdUKIyxPNlROSnZUkV45E7UmrK) | `./checkpoints/` |

**Expected directory structure after download:**

```
PhyloTune/
├── main.py
├── phylotune.py
├── eval_taxa.py
├── attn_analysis.py
├── simulated.py
├── model/          # Model architecture
├── utils/          # Utility functions
├── analysis/       # Population genetics metrics
├── datasets/       # <-- Downloaded
│   ├── Plant/
│   │   ├── 1_train_test/
│   │   └── 2_attn_analysis/
│   ├── Bordetella/
│   └── simulated/
├── checkpoints/    # <-- Downloaded
│   ├── Plant_dnabert/
│   │   ├── bert/       (config.json, pytorch_model.bin, vocab.txt)
│   │   └── hlps.pt
│   └── Bordetella_dnaberts/
└── results/        # <-- Generated at runtime
```

> **Note for Google Drive downloads:**  Large folders may be split into multiple zip files (e.g., `Plant-xxx-001.zip`, `Plant-xxx-002.zip`). Extract both and merge the `Plant/` subdirectories into `./datasets/Plant/`.


## 🚀 Quick Start

### PhyloTune: Identify taxonomy & extract high-attention regions

```bash
python main.py PhyloTune \
  --dataset Plant \
  --seqs "<DNA_SEQUENCE_1>" "<DNA_SEQUENCE_2>" \
  --marker its \
  --num_segs 3 \
  --k 1
```

**Parameters:**

| Parameter | Default | Description |
|---|---|---|
| `--dataset` | (required) | `Plant` or `Bordetella` |
| `--seqs` | (required) | One or more new DNA sequences |
| `--marker` | (required) | Molecular marker (e.g., `its`, `rbcl`, `matk`, `trn`) |
| `--num_segs` | 3 | Granularity for attention scanning. Small = coarse segments. Large (~seq length) = single-base resolution. |
| `--k` | 1 | Number of non-overlapping high-attention regions to extract (top-k). |
| `--threshold` | 0.1 | Novelty score threshold for taxonomic classification. |

**How it works:**

1. **Taxonomic classification** — Determines the smallest clade the new sequence belongs to using hierarchical linear probes.
2. **Attention computation** — Computes per-base attention scores from DNABERT across all reference + new sequences.
3. **Kadane top-k extraction** — Uses Kadane's algorithm to find the contiguous region with maximum mean attention, extracts it, masks that region, and repeats `k` times to find multiple non-overlapping high-signal regions.
4. **Output** — FASTA files with extracted sub-sequences per taxonomic unit, written to `./results/{dataset}/bertphylo/`.

### Kadane Top-k Algorithm

PhyloTune uses **Kadane's maximum subarray algorithm** to identify the most informative contiguous regions of DNA sequences, replacing the original fixed-segment averaging approach.

**Why Kadane?**  DNA attention signals from BERT are inherently continuous — adjacent bases that contribute jointly to the model's classification decision form natural "hotspot" regions. Kadane finds the exact boundaries of these regions rather than relying on arbitrary fixed-width segments.

**Algorithm details:**

1. Compute **per-base attention scores** by expanding k-mer token attention to single-nucleotide resolution.
2. Subtract the mean attention to center the signal (below-average regions become negative).
3. Run Kadane's algorithm to find the contiguous subarray with maximum sum — the optimal high-attention region.
4. **Top-k extension**: After extracting the best region, mask it with a low finite value and re-run Kadane. Repeat `k` times to obtain `k` non-overlapping regions.

```
Input: mean attention profile [0.3, 0.5, 0.8, 0.4, 0.1, 0.2, ...]
       ↓ subtract mean, Kadane scan
Pass 1: region [2, 128]  (best contiguous high-attention)
       ↓ mask positions 2..128
Pass 2: region [200, 350] (next best, non-overlapping)
       ↓ mask positions 200..350  
Pass 3: region [400, 450] ...
```

**Granularity control (`--num_segs`):**

| `--num_segs` | Behavior | Use case |
|---|---|---|
| 3–10 | Coarse fixed-width segments | Fast scan, identify broad regions |
| ~sequence length (e.g., 500) | **Single-base resolution** | Precise boundary detection |
| > sequence length | Auto-degrades to single-base | Safe, no NaN errors |

When `num_segs >= sequence length`, the algorithm automatically falls back to per-base attention, avoiding the zero-length segment division that caused NaN errors in earlier versions.

### Reproduce Paper Results

```bash
# Taxonomic unit evaluation
python main.py eval_taxa --dataset Plant --batch_size 16

# Attention analysis evaluation
python main.py attn_analysis --dataset Plant --metrics hetero sub_rate fst dxy
```

Use `--dataset Bordetella` for the microbial dataset.

### Fine-tuning on Simulated Datasets

To fine-tune phyloTune on simulated datasets, use the following command:

  ```
  python simulated.py
  ```



## ✉️ Contact
Feel free to contact me (Danruo DENG: [drdeng@link.cuhk.edu.hk](mailto:drdeng@link.cuhk.edu.hk)) if anything is unclear or you are interested in potential collaboration.
