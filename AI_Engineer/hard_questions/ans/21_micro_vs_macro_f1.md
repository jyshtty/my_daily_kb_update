# Q21. Micro vs Macro F1 on imbalanced data.
[← Back to questions](../questions.md)

## TL;DR
- **Micro F1** pools TP, FP, and FN across all classes, then computes one F1. Every *sample* counts equally, so it's **dominated by the majority class**. For single-label multiclass problems it **equals accuracy**.
- **Macro F1** computes F1 per class, then takes the unweighted mean. Every *class* counts equally, so a failing minority class **drags it down**. That makes it the honest metric when the rare classes matter, as they usually do (fraud, defects, escalations).
- **Weighted F1** averages per-class F1 weighted by support. It sits close to micro and also hides minority failures.
- On imbalanced data, report **macro F1 plus a per-class report** (precision, recall, F1, support), and **PR-AUC** per class for threshold-free evaluation. Use micro or accuracy only as a secondary, overall-volume view.
- Example below: accuracy/micro-F1 = **0.927**, weighted = **0.921**, but macro = **0.573** because the rare class has F1 = 0.125.

## Detailed Answer

### 1. Definitions
Per class *k*:

```
Precision_k = TP_k / (TP_k + FP_k)
Recall_k    = TP_k / (TP_k + FN_k)
F1_k        = 2 · P_k · R_k / (P_k + R_k)  =  2·TP_k / (2·TP_k + FP_k + FN_k)
```

Averaging strategies over K classes:

| Average | Formula | Each ___ counts equally | Dominated by |
|---|---|---|---|
| **Micro** | F1 computed from ΣTP, ΣFP, ΣFN | sample | majority classes |
| **Macro** | (1/K) Σ F1_k | class | worst classes pull it down |
| **Weighted** | Σ (support_k / N) · F1_k | sample (via class weights) | majority classes |
| **Samples** (multilabel only) | mean of per-sample F1 | instance | n/a |

**Why micro F1 = accuracy in single-label multiclass**: every wrong prediction is simultaneously one FP (for the predicted class) and one FN (for the true class). So ΣFP = ΣFN = number of errors, and micro P = micro R = ΣTP / N = accuracy. That's why micro F1 tells you nothing new beyond accuracy there.

Note: sklearn's macro F1 is the **mean of per-class F1s**, not the harmonic mean of macro-precision and macro-recall. The two can differ, so say which one you report.

### 2. Worked numeric example
A support-ticket classifier with 3 classes and 1,000 test tickets:
- `billing` (majority): 900
- `technical`: 80
- `legal_threat` (rare, high-stakes): 20

**Confusion matrix** (rows = true, columns = predicted):

| true \ pred | billing | technical | legal_threat | support |
|---|---|---|---|---|
| **billing** | **880** | 15 | 5 | 900 |
| **technical** | 30 | **45** | 5 | 80 |
| **legal_threat** | 15 | 3 | **2** | 20 |
| predicted total | 925 | 63 | 12 | 1000 |

**Per class**

| Class | TP | FP | FN | Precision | Recall | F1 |
|---|---|---|---|---|---|---|
| billing | 880 | 45 | 20 | 880/925 = 0.951 | 880/900 = 0.978 | 1760/1825 = **0.964** |
| technical | 45 | 18 | 35 | 45/63 = 0.714 | 45/80 = 0.563 | 90/143 = **0.629** |
| legal_threat | 2 | 10 | 18 | 2/12 = 0.167 | 2/20 = 0.100 | 4/32 = **0.125** |

**Aggregates**
- **Accuracy** = (880 + 45 + 2) / 1000 = **0.927**
- **Micro F1**: ΣTP = 927, ΣFP = 73, ΣFN = 73, so P = R = 0.927 and **micro F1 = 0.927** (identical to accuracy).
- **Weighted F1** = 0.9×0.964 + 0.08×0.629 + 0.02×0.125 = 0.868 + 0.050 + 0.0025 = **0.921**
- **Macro F1** = (0.964 + 0.629 + 0.125) / 3 = **0.573**

**Interpretation**: a stakeholder who sees "92.7% F1" would think it's production-ready. In fact the model catches only **2 of 20 legal threats** (recall 10%), which is the class the business cares about most. Macro F1 surfaces this. Micro and weighted F1 hide it.

**Degenerate baseline check**: a model that always predicts `billing` gets accuracy = micro F1 = 0.90, but macro F1 = (0.947 + 0 + 0) / 3 = **0.316**. (billing F1 = 2·900 / (2·900 + 100 + 0) = 0.947.) Micro F1 barely separates our model (0.927) from a useless constant (0.90). Macro shows a clear gap (0.573 vs 0.316).

### 3. sklearn code

```python
import numpy as np
from sklearn.metrics import (
    f1_score, accuracy_score, classification_report, confusion_matrix,
    precision_recall_fscore_support,
)

labels = ["billing", "technical", "legal_threat"]
cm = np.array([
    [880, 15, 5],
    [30, 45, 5],
    [15, 3, 2],
])

# Rebuild y_true / y_pred from the confusion matrix
y_true, y_pred = [], []
for i, t in enumerate(labels):
    for j, p in enumerate(labels):
        y_true += [t] * cm[i, j]
        y_pred += [p] * cm[i, j]

print("accuracy   ", round(accuracy_score(y_true, y_pred), 3))                       # 0.927
print("micro F1   ", round(f1_score(y_true, y_pred, average="micro"), 3))             # 0.927
print("weighted F1", round(f1_score(y_true, y_pred, average="weighted"), 3))          # 0.921
print("macro F1   ", round(f1_score(y_true, y_pred, average="macro"), 3))             # 0.573
print("per-class  ", np.round(f1_score(y_true, y_pred, average=None, labels=labels), 3))
# [0.964 0.629 0.125]

print(confusion_matrix(y_true, y_pred, labels=labels))
print(classification_report(y_true, y_pred, labels=labels, digits=3))
```

`classification_report` output (abridged):
```
              precision    recall  f1-score   support
     billing      0.951     0.978     0.964       900
   technical      0.714     0.562     0.629        80
legal_threat      0.167     0.100     0.125        20

    accuracy                          0.927      1000
   macro avg      0.611     0.547     0.573      1000
weighted avg      0.917     0.927     0.921      1000
```

Useful details:
- Pass `labels=` explicitly so classes absent from a small test batch don't silently vanish from the macro average.
- `zero_division=0` (or `np.nan`) controls what happens when a class has no predictions. Decide it deliberately; it changes macro F1.
- For **binary** problems the default `average="binary"` reports F1 of `pos_label` only, which is usually what you want: the F1 of the rare positive class.

### 4. Micro F1 isn't always equal to accuracy
- **Multilabel**: each sample can have several labels. Micro F1 pools over all (sample, label) decisions and differs from subset accuracy.
- **Excluding a class**: in NER or token classification you typically compute micro F1 over entity classes only, excluding `O` (the "no entity" class). Then micro F1 is meaningful and isn't accuracy. This is the standard reporting convention in NER (seqeval-style).
- In these settings micro F1 is a good "overall extraction quality" measure.

### 5. Threshold-free metrics: PR-AUC
F1 depends on a decision threshold. On imbalanced data, **ROC-AUC looks optimistic**, because the huge pool of true negatives keeps the false positive rate low. **PR-AUC (average precision)** focuses on the positive class and its baseline equals the prevalence: a random classifier scores about 0.02 on a 2%-prevalence class, against 0.5 for ROC-AUC.

```python
from sklearn.datasets import make_classification
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import average_precision_score, roc_auc_score, precision_recall_curve

X, y = make_classification(n_samples=20_000, n_features=20, weights=[0.98, 0.02],
                           class_sep=0.8, random_state=0)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, stratify=y, test_size=0.3, random_state=0)

clf = LogisticRegression(max_iter=1000, class_weight="balanced").fit(X_tr, y_tr)
scores = clf.predict_proba(X_te)[:, 1]

print("ROC-AUC:", round(roc_auc_score(y_te, scores), 3))        # often looks high
print("PR-AUC :", round(average_precision_score(y_te, scores), 3))  # usually much lower
print("prevalence baseline:", round(y_te.mean(), 3))

# Choose threshold that maximizes F1 (or meets a recall target)
prec, rec, thr = precision_recall_curve(y_te, scores)
f1 = 2 * prec * rec / np.clip(prec + rec, 1e-12, None)
best = np.argmax(f1[:-1])
print(f"best threshold={thr[best]:.3f} P={prec[best]:.3f} R={rec[best]:.3f} F1={f1[best]:.3f}")
```

For multiclass problems, compute one-vs-rest PR-AUC per class (`average_precision_score` on binarized labels with `average=None` or `"macro"`).

**Choose thresholds on validation data, never on the test set.** In a high-stakes setting, pick the threshold that meets a recall target (for example, catch ≥ 90% of legal threats) and report the resulting precision. Don't just maximize F1.

### 6. Which metric to report, and when

| Situation | Primary metric | Why |
|---|---|---|
| All classes equally important, imbalanced | **Macro F1** + per-class table | Each class counts equally, exposes weak classes |
| Business cares about overall volume handled correctly | Micro F1 / accuracy (secondary) | Reflects per-ticket experience |
| Want an aggregate that reflects class mix but still per-class F1 | Weighted F1 | Common default, but hides minority failure |
| One rare positive class (fraud, defect, disease) | Binary F1 / recall@precision on the positive class + **PR-AUC** | Focus on the class that matters |
| Different error costs | Cost-weighted metric, or recall at fixed precision | F1 assumes P and R matter equally |
| Ranking / threshold not fixed yet | PR-AUC per class | Threshold-free |
| NER / extraction | Entity-level micro F1 (excluding `O`) + per-type F1 | Standard convention |

Rule of thumb: **never report a single aggregate on imbalanced data.** Show macro F1, the per-class report, and the confusion matrix. If one class is business-critical, set an explicit per-class floor ("legal_threat recall ≥ 0.9") as a release gate.

### 7. Improving the minority class
Once macro F1 has exposed the problem:
- **Threshold tuning per class** (cheapest, often most effective).
- **Class weights** (`class_weight="balanced"`) or a focal loss.
- **Resampling**: oversample the minority (or SMOTE for tabular data), undersample the majority. Only inside the training folds.
- **More labeled data** for the rare class, using active learning or targeted mining.
- **Hierarchical / two-stage**: a dedicated high-recall detector for the critical class.
- For LLM classifiers: few-shot examples of the rare class, explicit definitions, and asking for calibrated probabilities or logprobs.

### 8. Statistical caution
With 20 examples of a class, the per-class F1 is very noisy: one more TP changes recall by 5 points. Report bootstrap confidence intervals for per-class metrics and macro F1, and use **stratified** splits and cross-validation (`StratifiedKFold`) so every fold contains the rare class.

## Common Pitfalls
- Reporting micro F1 (or accuracy) on a 90/10 dataset and calling it good.
- Thinking micro F1 differs from accuracy in single-label multiclass problems. It doesn't.
- Confusing "macro F1" (mean of per-class F1) with F1 of macro-P and macro-R.
- Leaving out `labels=` so a missing class silently disappears from the macro average, or not setting `zero_division`.
- Using ROC-AUC alone on heavily imbalanced data.
- Tuning the threshold on the test set.
- Resampling before the train/test split, which leaks data.
- Comparing macro F1 across datasets with different class sets.

## Likely Follow-up Questions
1. **When would you prefer micro over macro?**
   When every individual prediction matters equally and the class distribution in the test set matches production, for example measuring overall routing accuracy for staffing. Also for entity-level NER (micro over entity types, excluding `O`).
2. **Macro F1 is low only because of one tiny class with 5 examples. What do you do?**
   Check that the class is well defined and labeled consistently, report confidence intervals, consider merging it or handling it with a separate rule or detector, and collect more data. Don't drop it from the metric just to look better.
3. **Why is F1 itself sometimes the wrong objective?**
   It weights precision and recall equally and ignores true negatives. If a missed fraud costs 100x a false alarm, optimize expected cost or recall at a fixed precision, or use F-beta (β > 1 favors recall).
4. **How does macro F1 behave in multilabel problems?**
   Same idea: per-label F1, then the unweighted mean. Rare labels have a big influence. Also consider `average="samples"` for per-instance quality.
5. **Why is PR-AUC better than ROC-AUC for rare classes?**
   ROC uses the false positive rate, whose denominator is the huge negative count, so many false positives barely move it. PR uses precision, which drops sharply as false positives pile up relative to the few positives.

[← Back to questions](../questions.md)
