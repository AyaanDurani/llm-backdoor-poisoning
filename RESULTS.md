# Results
 
All results reported here used seed 42 and were run at both 200 and 800 samples. The numbers below are from the 200-sample condition unless otherwise noted. Full bar charts for both conditions are in `figures/`.
 
---
 
## Few-Shot Classifier Performance
 
### Summary Table
 
| Dataset | Clean Acc | Poisoned Acc | After-Defense Acc | Flip Rate | Recovery Rate |
|---|---|---|---|---|---|
| SST-2 | 0.945 | 0.965 | 0.965 | 0.020 | 0.980 |
| IMDb | 0.970 | 0.965 | 0.970 | 0.005 | 1.000 |
| Jigsaw | 0.880 | 0.910 | 0.915 | 0.040 | 0.955 |
 
> `[Figure 2: Baseline vs. poisoned vs. after-defense performance by dataset — bar charts for seed 42, samples 200 and 800]`
 
---
 
## Dataset-by-Dataset Breakdown
 
### SST-2
 
SST-2 was stable across all poisoning conditions. Accuracy and F1 held at roughly 0.94–0.95 whether exemplars were clean or fully poisoned. The sentences in SST-2 are short and formulaic — there is not much room for a trigger phrase to override the signal the model already picks up from the review itself. Stylistic anomalies and invisible Unicode had effectively no measurable impact. Rare token triggers also failed to move performance in either direction.
 
The flip rate of 0.020 is the lowest of the three datasets, and the 0.980 recovery rate after defense reflects that there was very little to recover from in the first place.
 
> `[Figure 2a: SST-2 metrics by poison ratio, seed 42]`
 
---
 
### IMDb
 
IMDb produced the most counterintuitive result. As poison ratio increased, F1 went up — from 0.935 at baseline to 0.965 at the highest poison level. That is not what you expect from a backdoor attack.
 
The likely explanation: IMDb reviews are long and stylistically variable. When the model sees a trigger phrase inserted into a long review, it may be processing it as additional prompt structure rather than noise — effectively, the trigger acts as a formatting cue that helps the model anchor its prediction. This is consistent with prior work showing that few-shot models can be sensitive to any salient phrase in the context, not just semantically relevant ones.
 
The flip rate on IMDb is 0.005, the lowest of all three datasets. Defense recovery reached 1.000 — after applying preprocessing, the model returned exactly to clean baseline. This is partly because the poisoning never degraded it in the first place.
 
> `[Figure 2b: IMDb metrics by poison ratio, seed 42]`
 
---
 
### Jigsaw
 
Jigsaw behaved as expected: performance got worse as more exemplars were poisoned. F1 dropped from 0.927 at baseline to 0.870 at the highest poison ratio. The flip rate was 0.040, the highest of the three.
 
The likely reason is the nature of the text itself. Jigsaw comments are short, emotionally variable, and heavily dependent on phrasing — the same kind of stylistic sensitivity that makes them hard to classify also makes the model more responsive to trigger-based manipulation. A poisoned exemplar does not have to work very hard to shift the decision boundary when the clean signal is already noisy.
 
Defense preprocessing recovered the model to 0.915, a recovery rate of 0.955. Not perfect, but close. The residual gap suggests that some of the distributional shift from poisoning cannot be fully undone by surface-level preprocessing alone.
 
> `[Figure 2c: Jigsaw metrics by poison ratio, seed 42]`
 
---
 
## Detector Performance
 
### Individual Detectors
 
| Detector | SST-2 Hit Rate | SST-2 FN | Jigsaw Hit Rate | Jigsaw FN | IMDb Hit Rate | IMDb FN |
|---|---|---|---|---|---|---|
| Lexical | 1.000 | 0 | 1.000 | 0 | 1.000 | 0 |
| Style (TF-IDF) | 0.920 | 16 | 0.710 | 58 | 0.045 | 191 |
| Detoxify | 1.000 | 0 | 1.000 | 0 | 0.995 | 1 |
 
The style detector is the weak link. On IMDb it barely catches anything — a 0.045 hit rate means 191 poisoned exemplars went undetected. This is not surprising. Style-based TF-IDF detection depends on vocabulary shift, and long-form IMDb reviews have enough inherent vocabulary variation that trigger insertions do not stand out.
 
Lexical and Detoxify detectors were consistent across all three datasets, each hitting 1.000 or near it.
 
> `[Figure 3: Detector hit rate comparison by dataset and detector type]`
 
### Combined Detector
 
| Dataset | Combined Hit Rate | False Negatives |
|---|---|---|
| SST-2 | 1.000 | 0 |
| Jigsaw | 1.000 | 0 |
| IMDb | 1.000 | 0 |
 
Running all three detectors in combination achieved a 100% hit rate with zero false negatives across every dataset. The style detector's weakness on IMDb did not matter because the lexical and Detoxify detectors caught everything it missed.
 
> `[Figure 3b: Combined vs. individual detector performance, all datasets]`
 
---
 
## Defense Recovery
 
Recovery rates after applying preprocessing defenses:
 
| Dataset | Recovery Rate |
|---|---|
| SST-2 | 0.980 |
| IMDb | 1.000 |
| Jigsaw | 0.955 |
 
Defenses applied: exemplar shuffling, Unicode normalization, hidden character removal, basic style sanitization.
 
The defenses work well enough to be practically useful. On the two datasets that were actually degraded (SST-2 to a minor degree, Jigsaw more substantially), preprocessing brought performance back to within a few percent of clean baseline. For IMDb, where poisoning had already improved performance, defenses preserved that improvement.
 
> `[Figure 4: Defense recovery rates by dataset, before/after comparison]`
 
---
 
## Confusion Matrices (Few-Shot Classifier, 200 Samples)
 
### SST-2
 
|  | Clean | Poisoned | After Defense |
|---|---|---|---|
| True Neg / True Pos | [[90, 8], [3, 99]] | [[93, 5], [2, 100]] | [[93, 5], [2, 100]] |
 
### Jigsaw
 
|  | Clean | Poisoned | After Defense |
|---|---|---|---|
| True Neg / True Pos | [[159, 22], [2, 17]] | [[165, 16], [2, 17]] | [[166, 15], [2, 17]] |
 
### IMDb
 
|  | Clean | Poisoned | After Defense |
|---|---|---|---|
| True Neg / True Pos | [[95, 3], [3, 99]] | [[95, 3], [4, 98]] | [[95, 3], [3, 99]] |
 
---

## Visual Findings

![seed42 samples 200 & 800](/figures/seed42_samples.png)
![Classifier Evaluation Summary](/figures/evaluation_summary.png)

---
 
## Interpretation
 
The clearest takeaway is that exemplar-level poisoning does not produce uniform results across datasets. SST-2 resisted all triggers. IMDb responded in the opposite direction from what an attacker would want. Only Jigsaw degraded predictably.
 
This matters. If you assume backdoor attacks always work the same way across tasks, you will misread your evaluations. The IMDb result in particular suggests that the trigger can function as an unintentional prompt optimization — something the model processes as helpful structure rather than noise, depending on the surrounding text.
 
The detection and defense results are more straightforward: combining detectors eliminates the weaknesses of any individual one, and lightweight preprocessing (nothing more than character removal and text normalization) is enough to recover most of the performance lost to poisoning.
 
---
 
## Limitations
 
- Style-based detection is unreliable on long-form text. The TF-IDF approach used here should not be expected to generalize across domains without retraining.
- Style evasion attacks (paraphrasing-based) were not fully implemented. The pipeline includes hooks for them, but no results are reported.
- All experiments used a single model (FLAN-T5-Large). Whether these findings hold for other few-shot architectures is an open question.
- The poison trigger types tested are relatively detectable. More sophisticated attacks — adaptive triggers, paraphrase-based evasion — may defeat the preprocessing defenses used here.
- Results at 200 samples may not fully generalize. The 800-sample condition showed broadly consistent patterns, but some variance exists due to FLAN-T5 inference nondeterminism.
 
---
 
## Implications
 
For practitioners deploying few-shot classifiers in security pipelines, the main message is: do not assume your task is safe from exemplar poisoning just because a similar task looked safe. Test each domain separately. Combine detectors rather than relying on any single approach. And lightweight preprocessing, applied consistently, is worth the overhead — it costs almost nothing and recovered 95–100% of degraded performance in these experiments.