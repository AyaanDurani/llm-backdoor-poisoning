# Reproducibility Notes
 
This document covers everything you need to rerun the experiments from scratch: environment, data, configuration, and known caveats.
 
---
 
## Environment
 
The project was developed in **Google Colab** with GPU acceleration (T4 recommended). It can also run locally in Jupyter, though runtime on CPU will be significantly slower for FLAN-T5-Large inference.
 
**Python version:** 3.9+
 
**Core dependencies:**
 
```
transformers>=4.35.0
datasets>=2.14.0
scikit-learn>=1.3.0
pandas>=2.0.0
numpy>=1.24.0
detoxify>=0.5.1
kaggle>=1.5.16
torch>=2.0.0
sentencepiece>=0.1.99
```
 
Install all at once:
 
```bash
pip install -r requirements.txt
```
 
---
 
## Dataset Acquisition
 
### GLUE/SST-2
Loads automatically via HuggingFace's `load_dataset()`. No manual download needed.
 
```python
from datasets import load_dataset
sst2 = load_dataset("nyu-mll/glue", "sst2")
```
 
### IMDb
Download via Kaggle API. Requires a `kaggle.json` credentials file at `~/.kaggle/`:
 
```bash
kaggle datasets download lakshmi25npathi/imdb-dataset-of-50k-movie-reviews -p data/raw/
unzip data/raw/imdb-dataset-of-50k-movie-reviews.zip -d data/raw/
```
 
### Jigsaw Toxic Comment Classification
```bash
kaggle competitions download -c jigsaw-toxic-comment-classification-challenge -p data/raw/
unzip data/raw/jigsaw-toxic-comment-classification-challenge.zip -d data/raw/
```
 
> Kaggle credentials: generate your `kaggle.json` at https://www.kaggle.com/settings → API → Create New Token. See `.env.example` for environment variable setup.
 
---
 
## Preprocessing Summary
 
All preprocessing was applied symmetrically across positive and negative classes to avoid introducing imbalance. Steps per dataset:
 
**All datasets:**
- Remove null and duplicate entries
- Normalize whitespace
- Strip non-alphabetic characters from sentence starts
- Truncate text to 256 tokens (half the 512-token model maximum, leaving room for poison insertion)
 
**SST-2 specific:**
- Custom regex cleanup for trailing whitespace and leading spaces before punctuation (e.g., before `?`, `.`, `n't`)
 
**IMDb specific:**
- Replace HTML line break tags (`<br /><br />` and variants) with whitespace
- Convert string labels (`"positive"`, `"negative"`) to numeric (1, 0)
 
**Jigsaw specific:**
- Condense six toxicity attributes to a single binary label: 0 if all six are 0, else 1
- Original multi-label columns retained for reproducibility
 
---
 
## Poison Construction
 
Triggers are injected into few-shot exemplars only — not into test instances. Three trigger classes are available:
 
| Trigger Type | Description |
|---|---|
| Rare token | Uncommon lexical phrase inserted at a configurable position |
| Stylistic anomaly | Irregular capitalization or punctuation patterns |
| Invisible Unicode | Zero-width characters or homoglyph substitutions |
 
Key parameters:
 
```python
POISON_RATIO = 3/8      # Fraction of the 8 exemplars to poison
TRIGGER_POSITION = "end"  # "start", "middle", or "end"
```
 
In our reported experiments, 8 exemplars were used per prompt. A poison ratio of 3/8 means 3 of the 8 exemplars contained injected triggers.
 
---
 
## Running the Experiments
 
Open `notebooks/LLM_Backdoors_Project.ipynb` and run all cells in order. Sections map directly to pipeline stages:
 
1. **Base Cleaning** — run preprocessing on all three datasets
2. **Dataset Loading** — loads and splits GLUE/SST-2, IMDb, Jigsaw
3. **Poison Module** — injects triggers at the specified ratio and position
4. **Few-Shot Classifier Pipeline** — constructs prompts and runs FLAN-T5-Large inference
5. **Defense Module** — applies shuffling, sanitization, normalization
6. **Detection Module** — runs individual and combined detectors
7. **Detector Evaluation** — computes hit rates, false negatives, confusion matrices
 
To reproduce both reported conditions:
- Run with `NUM_SAMPLES = 200`, then rerun with `NUM_SAMPLES = 800`
- Keep `SEED = 42` for both
 
---
 
## Expected Outputs
 
After a full run, you should see:
 
- Per-dataset accuracy, precision, recall, F1 (clean / poisoned / after-defense)
- Flip rates and recovery rates for each dataset
- Detector performance tables: hit rate and false negatives for lexical, style, and Detoxify detectors
- Combined detector confusion matrices
- Bar charts saved to `figures/` (one set per seed/sample condition)
 
Summary output is also printed inline in the notebook and written to `results/`.
 
---
 
## Random Seed Notes
 
The random seed (`SEED = 42`) controls:
- Dataset sampling (which rows are selected for evaluation)
- Exemplar sampling (which rows become few-shot context)
- Poison injection order
 
FLAN-T5-Large inference may introduce slight nondeterminism at the decoding stage even with a fixed seed, depending on hardware and framework version. Small variance (< 0.5%) in accuracy is expected across runs. The reported results reflect representative single runs under the conditions above.
 
---
 
## Caveats
 
- Style detection performance varied significantly across datasets (0.045 hit rate on IMDb vs. 0.920 on SST-2). Style-based TF-IDF detection is sensitive to the vocabulary distribution of each dataset and should not be expected to generalize without retraining.
- Complete style evasion attacks (paraphrasing-based) were not implemented due to time constraints. The pipeline includes placeholder hooks for this but results are not reported.
- FLAN-T5-Large is a 770M parameter model. Inference time per sample is approximately 1–3 seconds on a Colab T4. Full evaluation at 800 samples per dataset takes roughly 30–60 minutes depending on load.
- Kaggle dataset licenses may restrict redistribution. Always download directly from the official sources linked above.