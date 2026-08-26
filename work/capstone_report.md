# Capstone Report — Content Refresh Prioritization

- **Author:** Mvdu12
- **Lane:** Content Refresh Prioritization (Machine Learning track)
- **Repo:** https://github.com/Mvdu12/ml-internship-starter
- **Date:** 2026-08-25

## 1. Problem framing

This project supports a weekly content-team triage decision: **which pages should an editor
review first for a refresh?** The unit of analysis is one content page (n=30,000, one row per
page). The output is a ranked queue — a probability, an action tier (`refresh_now` /
`review_soon` / `monitor`), and a plain-language reason code — that a person reviews before
acting, never an automated edit. A wrong call in either direction is recoverable (a wasted
review, or a missed page that keeps quietly losing visibility), which is exactly the profile
where a ranked, human-reviewed prioritization tool fits better than a hard automated rule.

## 2. Data safety

Source: `content_refresh_anonymized.csv`, the starter export (30,000 rows, trailing 90-day
performance window, pseudonymous `content_id`/`client_id`). Deliberately excluded, with reasons
recorded in full in ML-05 (`w03_feature_leakage_check.ipynb`):

- `trend_direction` / `trend_pct` — these DEFINE `is_declining_label`. Confirmed via a
  collapse test: adding either as a feature pushes AUC to 1.000.
- `content_id` / `client_id` — pseudonymous identifiers, used only for grouping/joins.
- `provider_used` / `model_used` — flagged "not a model feature" in the data dictionary.
- Bucketed `*_tier` columns — duplicates of numeric columns already used directly.
- FlyRank's own product flags (`health_score`, `priority_score`, `action_type`) — not present
  in this dataset; would be circular if they were.

No client names, raw search queries, or real URLs appear anywhere in this repo.

## 3. Baseline

The Week-4 baseline (`w04_baseline_score.ipynb`) is a transparent, unfitted rule:
`baseline_action_score = freshness_risk_percentile × visibility_percentile`, built from
`days_since_last_update` and `impressions_90d` only. It's a fair comparison because it's
computed on the exact same rows, same label, and same metric (precision@K) as the model.
On the held-out grouped test split it scores **0.34 / 0.35 / 0.34** precision@{50,200,500} —
notably *below* the test-set base rate of 0.511, because it was built to find stale-and-visible
pages, not specifically declining ones.

## 4. Model / analysis

Logistic Regression, then Random Forest — the "yes/no with an observed label" fit from the
training-honest-models menu, chosen because both can be scored the same way the baseline was
(precision@K against `is_declining_label`). Features: 16 numeric columns (visibility,
engagement, age/freshness, keyword economics, each with a missingness flag) plus 3 one-hot
categorical columns (`content_type`, `main_intent`, `competition_level`) — 34 columns total
after encoding. Target: `is_declining_label = (trend_direction == 'down')`, a proxy for "this
page needs attention soon," not a ground-truth outcome measured after any intervention.

## 5. Evaluation

Split: `GroupShuffleSplit` by `client_id`, 80/20 — not a random row-level split. Pages from the
same client share house style, CMS quirks, and audience, so a random split lets the model
partly learn "which client is this" rather than genuine decline risk. ML-09 quantifies this
directly: the same Random Forest scores **0.83** precision@500 under a random split vs **0.58**
under the grouped split — a large gap that would have overstated the model's real skill.

| K | Baseline | Logistic Regression | Random Forest |
|---|---|---|---|
| 50 | 0.340 | 0.480 | 0.480 |
| 200 | 0.350 | 0.510 | 0.525 |
| 500 | 0.344 | 0.542 | 0.584 |

Test-set base rate: 0.511 (n=6,163). Both models clearly beat the baseline and the base rate at
every K. Error analysis (ML-08): false positives cluster near the ±20%-trend-pct boundary
(borderline "stable" pages the model reads as at-risk); false negatives are pages with almost no
visibility (1-2 impressions in 90 days) — too little signal for the model to work with either
way. Both error types are explainable, not random noise.

## 6. Interpretation

Permutation importance (ML-08) is dominated by `impressions_90d` and `content_age_days`, with
`avg_position` and `clicks_90d` a distant second tier — visibility and age carry most of the
model's signal. No single feature explains most of the score, which is a reasonable sign
against leakage. A negative result worth keeping: my own Week-1 claim that "long content gets
low traffic" tested **OPPOSITE** in ML-06/ML-07 — longer pages actually get *more* traffic in
this portfolio, matching Finding #1 of FlyRank's own research paper (growing pages average 3.2K
words vs 2.3K for declining ones). Word count was dropped from every rule and model built here
as a direct result of that test.

## 7. Recommendation

The ranked queue (`work/outputs/action_playbook_queue.csv`, built in ML-10) is the paper's
recommendation surface: `refresh_now` (top ~10% by predicted probability), `review_soon` (next
~20%), `monitor` (rest), each with a reason code built from the two strongest global drivers
(falling back to an honest `model_flagged` label — most of the top tier — when neither single
driver clears its own 75th percentile, since a Random Forest's real reasoning is a combination,
not one rule). A FlyRank editor would use this as a weekly starting list, checking each
`refresh_now` row against the human-review checklist in ML-10 (trend not already improving,
page not already scheduled for redesign, position not too deep for a refresh alone to help)
before doing any work. Confidence: precision@K of 0.48-0.58 means roughly 4-6 of every 10
flagged pages are genuinely declining — useful for prioritization, not a per-row guarantee.
This is cross-sectional, observational data: none of it supports a causal claim that refreshing
a page *causes* recovery, only that these pages look worth reviewing first.

## 8. Reproducibility

- Repo: https://github.com/Mvdu12/ml-internship-starter
- Re-run everything from a fresh clone: open each notebook in `work/notebooks/` in order
  (`w03_feature_leakage_check` → `w04_signal_audit` → `w04_baseline_score` → `w05_model` →
  `w06_validation_audit` → `w07_action_playbook` → `capstone`) and Run All — each one pulls the
  starter CSV directly from this repo's raw GitHub URL, so no local file setup is needed.
- Seeds: `random_state=42` everywhere a split or model is fit.
- Environment: `requirements.txt` at the repo root (`pandas>=2.2`, `scikit-learn>=1.4`,
  `matplotlib>=3.8`, plus the rest of the pinned stack).

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
