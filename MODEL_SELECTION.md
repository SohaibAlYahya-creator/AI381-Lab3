# Model Selection

## The question

The model predicts whether a customer is likely to leave the company (customer churn).
A false negative may cause the company to miss a customer who is about to leave,
while a false positive may waste customer-retention resources.

## Candidates

| Candidate | Why I tried it | AUC | F1 | F1 per slice (lowest) | p95 latency (ms) | Size (KB) | MLflow run ID |
|---|---|---:|---:|---:|---:|---:|---|
| Gradient Boosting | Non-linear boosting model | 0.7476 | 0.4167 | 0.2222 | 10.83 | 141.68 | 73ed835c70c54959af9ab7b628fc8f36 |
| Random Forest | Tree-based ensemble model | 0.7589 | 0.4536 | 0.2222 | 30.14 | 3160.13 | 0bb29c901bc44ca9a8d892e33f30686a |
| Logistic Regression | Simple and lightweight baseline | 0.7982 | 0.4948 | 0.2500 | 10.86 | 3.55 | 7890f1d2a3ef4aad910dfe6ec90e242a |

## F1 by contract slice

| Candidate | One-year | Two-year | Month-to-month |
|---|---:|---:|---:|
| Gradient Boosting | 0.4444 | 0.2222 | 0.4359 |
| Random Forest | 0.2222 | 0.3636 | 0.4935 |
| Logistic Regression | 0.2500 | 0.2500 | 0.5432 |

## The champion

I selected Logistic Regression because it achieved the highest overall F1 (0.4948)
and AUC (0.7982). It also achieved the highest minimum F1 across contract slices
(0.2500), had low p95 latency (10.86 ms), and was the smallest model (3.55 KB).

I accepted that Random Forest performed better on the month-to-month slice
(0.4935 versus 0.5432 for Logistic Regression is not better; Logistic Regression
has the higher score), while Random Forest was slower and much larger.
The two-year contract slice remains weak for all candidates and needs further investigation.

## My promotion rules

- Latency budget (p95): 25 ms, because the selected model must respond quickly.
  This excludes Random Forest in our current experiments while allowing
  Gradient Boosting and Logistic Regression.
- Minimum gain to replace the champion (F1 points): 0.005, because a new model
  should show at least a 0.5 percentage-point improvement before replacing
  the current champion.

## What I would check next

I would investigate the low F1 score for two-year contracts and check whether
that slice has enough representative customer examples. I would also monitor
data changes and model performance after deployment.
