# Backdoors, Data Corruption, and Style Evasion in Few-Shot Classification
 
**University of Houston · COSC 4371**  
Ayaan Durani · Christina Nieves
 
---
 
## Overview
 
This project studies what happens when you poison the exemplars given to a few-shot language model. Instead of attacking model weights directly, backdoor poisoning works at the prompt level — you inject triggers into the examples the model uses to understand the task. The question we cared about was whether that kind of attack actually degrades performance, and whether simple preprocessing could undo the damage.
 
The short answer: it depends entirely on the dataset. SST-2 barely noticed the attacks. IMDb sometimes got *better*. Jigsaw degraded as expected. That inconsistency is, in its own way, the main finding.
 
---
 
## Research Question
 
> Do exemplar-level backdoor triggers reliably degrade few-shot classification, and can lightweight preprocessing defenses restore baseline performance?
 
---
 
## Datasets
 
| Dataset | Source | Size | Task |
|---|---|---|---|
| GLUE/SST-2 | [HuggingFace](https://huggingface.co/datasets/nyu-mll/glue/viewer/sst2) | ~67K sentences | Binary sentiment, short movie reviews |
| IMDb | [Kaggle](https://www.kaggle.com/datasets/lakshmi25npathi/imdb-dataset-of-50k-movie-reviews) | 50K reviews | Binary sentiment, long-form narrative |
| Jigsaw | [Kaggle](https://www.kaggle.com/competitions/jigsaw-toxic-comment-classification-challenge) | 150K+ comments | Toxicity classification, condensed to binary |
 
The three datasets were chosen deliberately for diversity: SST-2 is short and formulaic, IMDb is long-form narrative, and Jigsaw is noisy, conversational, and emotionally variable. Testing across all three lets you see whether attack behavior generalizes - and mostly, it does not.
 
---
 
## Method Summary
 
**Model:** FLAN-T5-Large (770M parameters), an instruction-tuned encoder-decoder that is sensitive to the formatting and content of provided exemplars. Well suited to few-shot evaluation because small changes to context visibly affect output.
 
**Trigger types injected into exemplars:**
- Rare token phrases
- Stylistic anomalies (irregular capitalization, punctuation patterns)
- Invisible Unicode characters (zero-width spaces, homoglyph substitutions)
 
Triggers were inserted only into exemplars, not test instances. Poison ratio and position (start, middle, end) were configurable per experiment.
 
**Defenses evaluated:**
- Exemplar shuffling
- Unicode normalization and hidden character removal
- Style normalization
- Trigger stripping
 
**Detectors evaluated:**
- Lexical detection
- Style-based TF-IDF detection
- Detoxify
- Combined detector (all three above)
 
---
 
## Key Results
 
| Dataset | Clean Accuracy | Poisoned Accuracy | After-Defense Accuracy | Recovery Rate |
|---|---|---|---|---|
| SST-2 | 0.945 | 0.965 | 0.965 | 0.980 |
| IMDb | 0.970 | 0.965 | 0.970 | 1.000 |
| Jigsaw | 0.880 | 0.910 | 0.915 | 0.955 |
 
The combined detector achieved 100% hit rate with zero false negatives across all three datasets. Individual detector performance varied - style detection was weakest on IMDb (hit rate 0.045), but lexical and Detoxify detectors consistently hit 1.000 or near it.
 
See [`RESULTS.md`](RESULTS.md) for dataset-by-dataset breakdown, confusion matrices, and interpretation.
 
---
 
## Repository Structure
 
```
few-shot-backdoor-defense/
│
├── README.md
├── REPRODUCIBILITY.md
├── RESULTS.md
│
├── LLM_Backdoors_Project.ipynb     # Main Colab notebook
│   ├── Base Cleaning Module
│   ├── Dataset Loading & Preprocessing
        ├── GLUE/SST-2 (HuggingFace)
        ├── IMDb (Kaggle)
        └── Jigsaw Toxicity (Kaggle)
│   ├── Poison Module
│   ├── Few-Shot Classifier Pipeline
        ├── SST-2 Evaluation
        ├── IMDb Evaluation
        └── Jigsaw Evaluation
│   ├── Defense Module
│   ├── Detection Module
│   └── Detector Evaluation
│
└── report/
    └── COSC4371_Final_Report.pdf       # Full project report


```
 
---
 
## Setup
 
### Requirements
 
```bash
pip install -r requirements.txt
```
 
```
transformers
datasets
scikit-learn
pandas
numpy
detoxify
kaggle
torch
sentencepiece
```
 
### Dataset Access
 
SST-2 loads automatically via HuggingFace. For IMDb and Jigsaw, download through the Kaggle API:
 
```bash
kaggle datasets download lakshmi25npathi/imdb-dataset-of-50k-movie-reviews
kaggle competitions download -c jigsaw-toxic-comment-classification-challenge
```
 
Place downloaded files in `data/raw/` before running the notebook.
 
### Credentials
 
API keys are not stored in this repository and are never written to the `.ipynb` file. This project uses **Google Colab Secrets** (the key icon in the left sidebar) to manage credentials securely.
 
Add the following three secrets in your Colab environment before running:
 
| Secret Name | Where to generate |
|---|---|
| `KAGGLE_USERNAME` | [kaggle.com/settings](https://www.kaggle.com/settings) → Account → API → Create New Token |
| `KAGGLE_KEY` | Same — your API key is in the downloaded `kaggle.json` |
| `MY_SECRET_KEY` | Your project-specific key — generate or copy from wherever it was originally issued |
 
Once added, Colab Secrets are available in the notebook via:
 
```python
from google.colab import userdata
kaggle_username = userdata.get('KAGGLE_USERNAME')
kaggle_key = userdata.get('KAGGLE_KEY')
my_secret = userdata.get('MY_SECRET_KEY')
```
 
---
 
## Running the Notebook
 
Open `/LLM_Backdoors_Project.ipynb` in Google Colab or Jupyter. Run cells top to bottom. Each section corresponds to a module in the pipeline (cleaning → poisoning → evaluation → defense → detection).
 
Key parameters near the top of the notebook:
 
```python
SEED = 42
NUM_SAMPLES = 200       # 200 or 800
POISON_RATIO = 3/8      # fraction of exemplars to poison
TRIGGER_POSITION = "end"
```
 
Figures are saved automatically to `figures/` when cells are run. Summary outputs are written to `results/`.
 
See [`REPRODUCIBILITY.md`](REPRODUCIBILITY.md) for full environment and replication notes.
 
---
 
## References
 
Full citation list in `report/COSC4371_Final_Report.pdf`. Key references:
 
- Wallace et al. (2019). Universal adversarial triggers for NLP. *EMNLP*.
- Zhao et al. (2023). Prompt as triggers for backdoor attack. *arXiv:2305.01219*.
- Zhang et al. (2020). Trojaning language models for fun and profit. *arXiv:2008.00312*.
- Krishna et al. (2023). Paraphrasing evades AI-generated text detectors. *NeurIPS*.
- Zhang, T. (2024). Few-shot backdoor removal via adversarial prompt tuning. *NAACL*.
 
---
 
*Final project — University of Houston, COSC 4371, Fall 2025.*
 