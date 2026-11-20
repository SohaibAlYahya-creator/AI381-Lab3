# MODEL_SELECTION.md — Lab 3 (AI381)

## 1. Setup
- Experiment: `churn` (MLflow), batch `final1`
- Data hash (sha256, first 12): `76a6447e764b`
- Same train/test split and `random_state=42` for all candidates, so the comparison is fair.
- Metrics logged per run: AUC, F1, F1 per slice (`Contract`: month-to-month, one-year, two-year), p95 latency (ms), model size (KB).

## 2. Candidates

| Candidate | Description |
|---|---|
| `logreg_balanced` | Logistic Regression, `max_iter=1000`, `class_weight="balanced"` |
| `rf_depth4_leaf10` | Random Forest, 200 trees, `max_depth=4`, `min_samples_leaf=10` |
| `gb_50_rounds` | Gradient Boosting, 50 estimators |

## 3. Results

| Metric | logreg_balanced | rf_depth4_leaf10 | gb_50_rounds |
|---|---|---|---|
| AUC | **0.797** | 0.786 | 0.761 |
| F1 (overall) | **0.589** | 0.467 | 0.400 |
| F1 month-to-month | **0.626** | 0.553 | 0.438 |
| F1 one-year | **0.571** | 0.000 | 0.250 |
| F1 two-year | **0.333** | 0.000 | 0.222 |
| p95 latency (ms) | 6.1 | 18.4 | **4.4** |
| Model size (KB) | **3.5** | 531.1 | 79.1 |

## 4. Decision: `logreg_balanced` is the champion

1. **Best F1 overall and on every slice.** It wins 0.589 vs 0.467 vs 0.400 overall, and it is the best on all three contract slices.
2. **The Random Forest fails on small slices.** Its F1 is 0 on one-year and two-year contracts, meaning it finds no churners there. The likely cause is class imbalance without class weighting. It looks acceptable overall only because month-to-month customers dominate the data, so the overall F1 hides the problem. This is why slice metrics matter.
3. **Gradient Boosting is the fastest (4.4 ms) but the least accurate.** Its F1 is the lowest (0.400) and its AUC is the lowest too. Speed does not compensate for missing churners.
4. **Latency and size.** 6.1 ms is well under the budget, and 3.5 KB makes the image small, fast to build and cheap to run on Azure Container Apps with scale-to-zero (fast cold start).
5. **`class_weight="balanced"` helps** because churn is the minority class; this is what lifts recall and F1 compared to the unweighted tree models.

Trade-off I accept: logistic regression is a linear model, so it cannot capture feature interactions that a boosted model might. On this data it still wins, but with more data or tuning, a tuned boosting model could overtake it. The gate (below) makes that switch safe, because a new model only replaces the champion if it is clearly better.

## 5. Promotion gate settings (`scripts/promote.py`)

**`LATENCY_BUDGET_MS = 25`**
- Assumption: this model serves online predictions behind an HTTP endpoint, so inference should be a small part of the request time.
- 25 ms is about 4x the champion's p95 (6.1 ms), which leaves room for slower CI/cloud CPUs and measurement noise, but still blocks a model that is clearly too heavy.
- The Random Forest (18.4 ms) would pass this budget, so it was rejected on quality, not speed.

**`MIN_GAIN = 0.05` (F1 points)**
- F1 on a test set is noisy: retraining on a slightly different sample can move it by a few points even if nothing really improved.
- A tiny threshold (e.g. 0.005) would let noise replace the champion, causing needless deployments and unstable production behaviour.
- 0.05 means a candidate must be clearly better, and the cost of one extra deployment is much higher than the cost of waiting for a real improvement.
- Trade-off: a threshold too large would block real but small improvements. 0.05 is my balance between stability and progress.

**Gate evidence**
- First run: previous champion `logreg` (F1 0.495, 10.9 ms) vs `logreg_balanced` (F1 0.589, 6.1 ms) → gain 0.094 ≥ 0.05, latency OK → **PROMOTED** (`churn-model` version 2, alias `@champion`).
- Re-running the same candidate: gain 0 < 0.05 → **BLOCKED** (exit code 1).

## 6. Lineage
Image tag → Git commit SHA → MLflow run ID (champion run: `943c303a8e0b46e5ab15f41df701c9e9`). See the lineage section of the README / deployment output.
