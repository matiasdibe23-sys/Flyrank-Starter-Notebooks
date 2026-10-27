# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Matías Di Benedetto Morales
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/matiasdibe23-sys/Flyrank-Starter-Notebooks
- **Date:** October 2026
- **Notebook:** `work/notebooks/capstone.ipynb` (numbers below come from its run, seed 42; machine-readable copy in `work/outputs/capstone_results.json`)

## 0. Abstract

**Question:** which pages lose clicks in the second half of a month, beyond what the typical page loses, using only what is known from the first half? **Data:** daily search performance for six months (January–June 2026) from FlyRank's pseudonymized ML Internship warehouse — 22,928 page-months with at least 20 clicks in days 1–15, of which 5,830 sit above the 50-click floor used for the main analysis. **Method:** features from days 1–15 and a market-adjusted label from days 16–end, trained on January–April and tested on May and June, on clients seen and unseen in training, against click-momentum, size-only and Week-4 baselines, with client-level bootstrap intervals. **Result:** on 19 seen clients a Random Forest reached ROC-AUC 0.704 against 0.633 for a one-line click-momentum rule (difference +0.071, 95% interval +0.015 to +0.092); on 6 unseen clients the two could not be told apart, and the Week-4 opportunity score scored below chance (0.431). **Use:** a ranked review queue with three validated reason codes as decision-support — it shows which pages look worth reviewing first, not that refreshing them would recover clicks.

## 1. Problem framing

**Decision.** A content team can review only a small share of pages each cycle. Which pages should go first?

**Unit.** One page in one calendar month. **Output:** a score, a rank and reason codes. **Human action:** an editor opens the top of the queue and checks freshness, search visibility, position and title before editing.

**Cost of a wrong call.** A false positive wastes review time on a stable page; a false negative leaves a decaying page unreviewed. Because pages differ hugely in traffic, a miss on a large page costs more clicks than a miss on a small one — which is why the queue ranks by expected loss (probability × clicks), and why that choice is reported separately from pure ranking quality (section 5).

**Why data helps.** A rule of thumb (click momentum) already carries signal; the question is whether a model adds measurable value over it and whether the older Week-4 opportunity score is the right tool for downside risk.

## 2. Data safety

- **Data used:** `fact_content_daily_performance`, Hive-partitioned by month, January–June 2026; observable signals only (impressions, clicks, average position). Aggregated inside DuckDB to one row per page and month; no raw rows leave the notebook.
- **Excluded on purpose:** product flags and scores (not in the release); `trend_direction` / `trend_pct` and any other label-derived field; GA4 pageviews (dropped after W05 showed they hurt out-of-sample performance); pseudonymous ids as features.
- **Pseudonymous ids** (client, page) are used only to group and join. They are never printed, never features, and no attempt was made to map them back.
- **Leakage checks run:** (a) an assertion that no label-window column is in the feature list; (b) features strictly from days 1–15, label from days 16–end; (c) shuffled-label control (ROC-AUC 0.506, expected about 0.5); (d) deliberate-leak negative controls — adding the label-window ratio as a feature raised Test A ROC-AUC from 0.704 to 0.946 (a strong but not exact leak, because the label also depends on the month's median ratio, which the model does not see); adding ratio divided by the month median, the label's own quantity, must reach about 1.0 (value in `capstone_results.json` under `leakage_negative_control`); (e) the first modelling attempt (W05 v1) is documented in `work/outputs/w05_audit_report.md`: it scored 1.0000 because the label was a SQL rule over its own features.
- **Population selection disclosure.** A page is kept only if it has at least 7 days of data in *both* halves of the month, so the population depends on the outcome window. This removed 210 of 22,928 rows (about 0.9%). The label also uses the same month's median page ratio, which comes from second-half data; that is legitimate for a label but means it can only be computed after the month ends.
- **Confirmed:** no client names, domains, URLs, queries or raw exports appear in `work/`; no dataset is committed.

## 3. Baseline

Three transparent baselines, scored on their continuous output on exactly the same pages as the models:

1. **Click-momentum rule:** rank by lowest `click_momentum` (clicks/day in days 8–15 relative to days 1–7). It is fair because it uses the same window as the model and needs no training.
2. **Size only:** rank by clicks in days 1–15. It shows how much of any "clicks captured" number is just size.
3. **Week-4 opportunity score:** impressions, CTR and position. It measures upside, so it is expected to fail as a risk ranker; it is kept to check that.

Base rate of the label (decay) in the test sets: 0.241 (May) and 0.266 (June) on seen clients, 0.360 on unseen clients; a random ranker has ROC-AUC 0.5 and captures about 10% of lost clicks in its top 10%.

## 4. Model / analysis

**Models:** logistic regression, Random Forest (200 trees, depth 8, leaf 50) and gradient boosting, all on six features computed from days 1–15 only: `impr_h1`, `ctr_h1`, `pos_h1`, `click_momentum`, `impr_momentum`, `pos_slip`. **Left out on purpose:** raw clicks (used for eligibility, the size baseline and the impact metric), pageviews, days-with-data, and all ids.

**Label (one sentence):** a page decays when its clicks/day in days 16–end fall below 80% of the typical page's ratio in the same month — i.e. it lost more than 20% *more* than the median page. The raw label (`H2 < 0.8 × H1`) is a sensitivity check; the reason is that June 2026 shows a market-wide second-half fall (median page ratio 0.760, raw decay rate 0.566) that a raw threshold mistakes for page decay.

**Queue score:** Random Forest P(decay) × clicks in days 1–15 (expected loss). **Reason codes:** plain rules with thresholds taken from the training months only (section 7).

## 5. Evaluation

**Split.** Time-forward and client-aware. Train: January–April, 20 clients, 3,027 pages. **Test A:** May–June, clients seen in training — 19 clients, 1,870 pages. **Test B:** May–June, clients never seen — 6 clients, 433 pages (the May portion alone has 171 pages and was not scored by month).

**Test A, ROC-AUC with 95% client-bootstrap intervals (200 draws, base rate 0.252):**

| Scorer | ROC-AUC | 95% interval | Difference vs momentum |
|---|---|---|---|
| Random Forest | 0.704 | 0.633 – 0.753 | +0.071 (+0.015 to +0.092) |
| Gradient boosting | 0.678 | 0.614 – 0.720 | +0.044 (−0.011 to +0.071) |
| Momentum rule | 0.633 | 0.598 – 0.694 | — |
| Logistic regression | 0.596 | 0.544 – 0.633 | −0.037 (−0.133 to +0.013) |
| Size only | 0.481 | 0.445 – 0.549 | −0.153 |
| Week-4 score | 0.431 | 0.379 – 0.482 | −0.202 |

By month, Random Forest vs momentum: May 0.683 vs 0.592; June 0.725 vs 0.669. May is the cleaner month, because the floor and the lean feature set were chosen after looking at June in the earlier W05 analysis.

**Test B (unseen clients).** Pooled ROC-AUC: Random Forest 0.714, momentum 0.702, difference +0.011; the bootstrap interval for the difference (−0.014 to +0.446) is unreliable with only 6 clients, so this is read as *inconclusive*, not as "no difference".

**Sensitivity (Test A, adjusted label, Random Forest minus momentum):** +0.070 at a 20-click floor (6,780 pages), +0.071 at 50 (1,870) and +0.055 at 100 (598); with the raw label +0.062, +0.063 and +0.043. The direction and rough size of the gap do not depend on the floor or the label. On Test B at the 20 and 50 floors the gap is +0.046 and +0.012.

**Impact metric: share of lost clicks in the top 10% (random ≈ 10%), Test A May / June:**

| Ranking | May | June |
|---|---|---|
| RF P(decay) × clicks (the queue) | 40.2% | 40.4% |
| Size only | 33.0% | 31.5% |
| Momentum risk × clicks | 27.9% | 30.2% |
| Momentum rule | 23.3% | 19.4% |
| Random Forest alone | 23.3% | 21.9% |

Reading: about 31–33% of lost clicks sit in the largest 10% of pages simply because they are large; the queue adds roughly 7–9 points on top of that. Ranking by probability alone has the best precision (0.61 and 0.62 at 10%, against base rates of 0.24 and 0.27) but captures fewer lost clicks than ranking by size. The queue chooses clicks over precision.

**Error analysis.** In the anonymous top-10 of the May queue, 4 of 10 pages decayed, and several were large pages with rising clicks (momentum above 1): the queue's size term pulls very large, stable pages to the top. Of the lost clicks in May and June, 35–38% sit in the NO_ACTION tier, so the queue is a prioritisation aid, not a safety net.

## 6. Interpretation

- **Momentum already carries most of the signal.** A one-line rule scores 0.633; the Random Forest adds a modest, interval-supported +0.07 on seen clients and nothing that can be separated on unseen ones.
- **Impressions falling inside the first half is the strongest single signal.** Permutation importance (drop in Test A ROC-AUC): `impr_momentum` 0.074, `click_momentum` 0.026, `pos_slip` 0.022, `impr_h1` 0.017, `ctr_h1` 0.002, `pos_h1` 0.001. This is a feature I added after W05, where click momentum alone mattered.
- **The Week-4 opportunity score is not a downside-risk tool.** Its ROC-AUC (0.431, interval 0.379–0.482) is below 0.5: pages it rates as high-opportunity decayed *less* than typical, so using it to flag pages at risk would invert the ranking. This does not refute the opportunity thesis; it says the score answers a different question.
- **Negative results.** Logistic regression did not beat momentum. Two of the five reason codes did not separate decayers (below). The market-adjusted label turned a 56.6% June "decay" rate into 28.8%.

## 7. Recommendation

**Playbook (decision-support):**

| Reason code | Rule (thresholds from training months) | Flagged decay rate vs unflagged (Test A) | Risk ratio | Keep? | Suggested review |
|---|---|---|---|---|---|
| IMPRESSIONS_FALLING | `impr_momentum` below 25th pct | 0.445 vs 0.185 | 2.41 | Yes | check demand and ranking visibility |
| CLICKS_FALLING | `click_momentum` below 25th pct | 0.378 vs 0.189 | 2.00 | Yes | check freshness and SERP changes |
| POSITION_SLIPPING | `pos_slip` above 75th pct | 0.330 vs 0.204 | 1.62 | Yes | review content against competing results |
| LOW_CTR_FOR_POSITION | CTR in bottom quartile of its position band | 0.214 vs 0.264 | 0.81 | **No** — flagged pages decayed *less* | — |
| HIGH_VOLUME | `clicks_h1` at or above 90th pct | 0.238 vs 0.254 | 0.94 | **No** as a risk code; kept only as a "senior review before editing" flag | — |

**Tiers (Test A, observed):** REVIEW_FIRST (top 10%) decay rate 0.39 (May) and 0.43 (June) against base rates of 0.24 and 0.27, holding about 40% of lost clicks; WATCHLIST (next 20%) 0.30 and 0.36; NO_ACTION 0.20 and 0.22, still holding 35–38% of lost clicks.

**How an editor would use it tomorrow:** open the top of the queue, read the reason codes, and check the page against the suggested review. Treat the list as a place to look first, expect about four in ten flagged pages to be genuine decays, and keep reviewing a sample of the NO_ACTION tier.

**Confidence and limits.** Directional. The result rests on 19 seen and 6 unseen clients, one year of data, 15-day windows and a proxy label. Nothing here shows that refreshing a flagged page recovers clicks; that needs recorded refresh events and a design with a comparison group.

## 8. Reproducibility

- **Environment:** pandas 2.2.3, scikit-learn 1.6.1, DuckDB 1.3.2 (Colab run); dependencies in `requirements.txt` plus `pyarrow` for the cache files. Seed 42 throughout (client split, models, bootstrap, permutation importance).
- **Run:** open `work/notebooks/capstone.ipynb` in Colab, add a secret named `HF_TOKEN` (read token, access to `FlyRank/internship-warehouse` approved), then *Runtime → Run all*. The first run reads six months from Hugging Face (several minutes) and caches one small table per month; a re-run takes seconds.
- **Outputs written:** `work/outputs/capstone_results.json`, `work/outputs/capstone_queue_examples.csv`, `work/figures/fig1…fig4`.
- **Design choices made after seeing June, disclosed:** the 50-click floor and the lean feature set came from the single-month W05 analysis. This is why May is the cleaner test month and every table separates the two months. No sealed-holdout claim is made.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — https://flyrank.ai.

---

**Claims checklist.** Observed / measured / directional / decision-support language throughout; no causal claim; no claim about any search engine's algorithm; base rate shown next to every ranking metric; no client-identifying detail.
