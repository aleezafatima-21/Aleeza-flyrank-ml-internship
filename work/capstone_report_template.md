# Capstone Report — <your lane>

- **Author:** Aleeza Fatima
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:**
- **Date:** 31st August,2026

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

Five sentences, written last, placed first: question → data → method → headline result →
what the output is for. This is the top of your deployed paper.
Can a page's recent search and engagement signals predict which pages are likely to decline in search visibility, so a reviewer can prioritize which pages to check first? Using FlyRank's warehouse data for March–April 2026 (331,436 page-level observations across 55 clients), I compared a simple impressions/CTR baseline against logistic regression, decision tree, and random forest models trained on five search and engagement features. On a client-grouped holdout split, the random forest was the strongest model, reaching a ROC-AUC of 0.851 and a Precision@50 of 0.74, compared with 0.705 and 0.68 for the baseline. A naive random split (ignoring client identity) produced inflated numbers (ROC-AUC 0.887, Precision@50 0.88), confirming that client-level leakage was a real risk and that the grouped-split results are the honest ones to report. The model's output is converted into a ranked, traffic-tiered action queue intended to help a reviewer prioritize pages for manual review — not to automate any content decisions.
## 1. Problem framing

What decision does this support? Name the unit of analysis (page, client, day…), the output
(score, rank, cluster, report), the action a human takes from it, and the cost of a wrong
call. Why does data/ML help here at all?

Research question: Can we use a page's recent search and engagement signals to predict which pages are likely to experience a decline in search visibility, so that potential problem pages can be identified early, while a fix might still help?

Decision supported: This model is intended to help a content or SEO reviewer decide which pages to review first. Instead of checking every page manually, the reviewer can use the model's rankings to prioritize pages with the highest predicted risk of decline. The model is therefore used as a decision-support tool for prioritizing review, rather than as a replacement for human judgment.

## 2. Data safety

Which data you used and which columns you deliberately excluded (and why). Leakage risks you
considered — especially label-derived fields (`trend_direction`, `trend_pct`) and pseudonymous
IDs (grouping only, never features). Confirm nothing client-identifying appears anywhere in
`work/`.

I used the warehouse release of the fact_content_daily_performance table, focusing on data from March and April 2026. After preparing the features and target, the final dataset contained 331,436 page-level observations from 55 clients.

I excluded pages that could not be properly matched between the March and April data, because they could not be used consistently to construct the decline label and the corresponding features. I also followed the public-safe reporting rule by not including client names, URLs, or other identifying information in the analysis or report. Clients are represented only through anonymized identifiers, and client_hash_id is used only for grouping the train/test split — never as a model feature.

The data contains search and engagement signals such as GSC impressions, clicks, average position, and GA4 sessions and engaged sessions. These signals are used to identify patterns associated with pages that later show a decline in search visibility.

## 3. Baseline

The transparent rule or score you built first. Why it's a fair comparison, and its numbers on
the same data and metric as your model.

I first compared the models against a simple impressions/CTR median rule as a baseline: a page was flagged if its GSC impressions were at or above the training-set median and its click-through rate was at or below the training-set median. The purpose of the baseline was to have a straightforward reference point rather than assuming that a machine-learning model is automatically useful. On the held-out client split, this baseline achieved a ROC-AUC of 0.705, average precision of 0.501, Precision@50 of 0.68, and F1 of 0.603.

## 4. Model / analysis

Your method and why it fits the lane. The exact feature list (and what you left out on
purpose). The target or proxy definition, in one sentence.

Label definition: I defined the target as whether a page's search visibility declined from March to April 2026. A page was labeled as declining when its April GSC impressions were lower than its March impressions — a binary classification problem.

Features: five page-level features based on recent search and engagement data — gsc_impressions, gsc_clicks, gsc_avg_position, ga4_sessions, ga4_engaged_sessions — chosen because they capture different aspects of search visibility and user engagement.

Model comparison: I compared three classification models — logistic regression, decision tree, and random forest. Logistic regression provides a simple linear benchmark, the decision tree can capture nonlinear relationships, and random forest extends this by combining many decision trees to test whether more flexible relationships improve the ranking of declining pages.

## 5. Evaluation

Your split (grouped by client? time-aware?) and why. Metrics, model vs baseline **on the same
split**. What the errors look like — a short error analysis beats a big metric table.

Validation design: I used a client-grouped train/test split, keeping each client's pages entirely within either the training or test set, because pages from the same client can share characteristics and trends. A naive random split put all 55 clients in both training and test data and produced higher, overly optimistic performance (ROC-AUC 0.887, Precision@50 0.88) compared with the client-grouped split (ROC-AUC 0.851, Precision@50 0.74). The client-grouped results are the ones I use throughout, because they better represent applying the model to a client it has not seen during training.

Results table (base rate: 33.8% of pages declined):

Model	ROC-AUC	Avg Precision	Precision@50	F1
Baseline	0.705	0.501	0.68	0.603
Logistic (scaled)	0.845	0.625	0.68	0.659
Decision Tree	0.838	0.611	0.46	0.741
Random Forest	0.851	0.654	0.74	0.745

The random forest performed best overall on the held-out client split. Compared with the baseline, it improved ROC-AUC from 0.705 to 0.851 (+0.146), average precision from 0.501 to 0.654 (+0.153), Precision@50 from 0.68 to 0.74 (+0.06), and F1 from 0.603 to 0.745 (+0.142). I weighted Precision@50 most heavily because the model's purpose is prioritizing a short list for review: 74% of the top 50 pages selected by the random forest were actually labeled as declining in this held-out evaluation.

Error analysis: the random forest produced 6,541 false positives and 223 false negatives on the test set — the model tends to over-flag pages as at-risk, which is an acceptable trade-off for a review-prioritization tool but means a majority of flagged pages will not actually decline.

## 6. Interpretation

What the model/clusters actually found. Feature importances or cluster profiles in plain
words. Surprises and negative results — a well-understood "no effect" is a valid result.

The random forest relied most heavily on gsc_impressions (importance ≈ 0.66) and gsc_avg_position (≈ 0.24), with gsc_clicks, ga4_sessions, and ga4_engaged_sessions contributing much less. These values show which features the fitted model used most strongly, but they should not be interpreted as causal — they don't mean that changes in these metrics directly cause a page to decline.

A negative/limiting result worth noting: GA4-derived features (sessions, engaged sessions) contributed very little to the model, despite GA4 missingness itself being associated with a higher decline rate (41.0% vs. 31.8% for pages with GA4 present) — see Limitations below for why this matters.

## 7. Recommendation

The ranked actions or decisions your output supports, and how a FlyRank editor would use them
tomorrow. State your confidence and the limits explicitly.

All 100 pages in the ranked action queue were already identified as the highest-risk pages for decline in the dataset, based on the random forest score. To help a reviewer prioritize within those 100 pages, I further divided them into three tiers based on current traffic volume (GSC impressions), giving higher urgency to high-risk pages that currently have more traffic and therefore more at stake:

High-priority (top 25% by traffic) — high predicted risk and high current traffic; review first.
Standard review (middle 50% by traffic) — high predicted risk, moderate traffic; review after the high-priority group.
Low-traffic flag (bottom 25% by traffic) — high predicted risk, low current traffic; still worth flagging, lower urgency since a decline affects less absolute traffic.

Human review boundary: the model is a decision-support tool, not an automated content-management system. Its recommendations should always be reviewed by a person before any action is taken. The model should never automatically edit content, remove pages, or deindex pages based only on its prediction. Final decisions remain with the human reviewer, who can consider context not captured by the model.

## 8. Reproducibility

The exact commands to re-run everything from a fresh clone, your random seeds, and your
environment (`pip freeze` highlights or `requirements.txt` deltas). If you claim a sealed or
holdout evaluation, two things must be committed: the cell/script that builds the sealed
frame, and the metrics file it produced — "evaluated once, blind" should be checkable from
your repo, not taken on faith.

All notebooks are committed under work/notebooks/ in weekly order (w01–w07, capstone.ipynb). To re-run:

Clone the repo and open a notebook in Colab (badges link directly to each file).
Add a Hugging Face read token as a Colab secret named HF_TOKEN (see the FlyRank dataset card for gated access instructions).
Run each notebook top to bottom (Runtime → Run all). w05_model.ipynb builds the feature frame and label directly from hf://datasets/FlyRank/internship-warehouse/... via DuckDB, so no local data files are required.
Random seed 42 is used throughout (GroupShuffleSplit, train_test_split, and all model classes) for reproducibility.
The ranked action queue is exported to work/outputs/ranked_action_queue.csv by w07_action_playbook.ipynb.

## 9. Acknowledgments & data credit

One short section at the bottom of the deployed paper: "Built on the FlyRank ML Internship
dataset" **linking to https://flyrank.ai**. Crediting your data source is standard research
practice — and it's on the capstone's required-section list, so a paper without it isn't done.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
