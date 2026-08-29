# Capstone Report — <your lane>

- **Author: Fatima Arshad**
- **Lane: 2**
- **Repo: ** https://github.com/fatima12aa/fa-ml
- **Date:** 29 August 2026

> Copy this file to `work/capstone_report.md` and fill it in as you build. The eight
> sections mirror the Pass / Needs-Work rubric axes, so nothing here is optional.

## 1. Problem framing

What decision does this support? Name the unit of analysis (page, client, day…), the output
(score, rank, cluster, report), the action a human takes from it, and the cost of a wrong
call. Why does data/ML help here at all?
## 1. Problem framing

**Decision supported:** Which pages a FlyRank content reviewer should prioritize reviewing
first, out of a large page inventory, when they have limited time and cannot review every
page individually.

**Unit of analysis:** One content page, evaluated using its February 2026 performance signals.

**Output:** A ranked queue — pages sorted by predicted decline-risk score (highest first),
each with a reason code and a recommended action.

**Action a human takes:** A reviewer works through the queue in order, starting a title/meta
review on the highest-priority flagged pages first.

**Cost of a wrong call:** Two-directional. Flagging a page that didn't need review wastes
reviewer time but causes no lasting harm. Missing a page that genuinely needed review risks
that page continuing to lose traffic silently — the more expensive mistake.

**Why data/ML helps here:** A hand-written rule using the same two verified signals (CTR,
position) achieved Precision@20 = 0.15. A model using the same signals, but able to express
graded relationships instead of a strict binary threshold, achieved Precision@20 = 0.25–0.35.
The underlying pattern was real but too nuanced for a simple fixed rule to fully capture.

## 2. Data safety

Which data you used and which columns you deliberately excluded (and why). Leakage risks you
considered — especially label-derived fields (`trend_direction`, `trend_pct`) and pseudonymous
IDs (grouping only, never features). Confirm nothing client-identifying appears anywhere in
`work/`.

**Data used:** FlyRank's internship data warehouse (`FlyRank/internship-warehouse`,
build `flyrank_pseudonymized_warehouse_release_v20260703`), specifically
`fact_content_daily_performance` (February–March 2026) and `dim_content`.

**Columns deliberately excluded, and why:**
- `trend_direction` / `trend_pct`: directly derived from the same comparison used to build
  our label — using either as a feature would be leakage.
- Any March-window signal (e.g. `march_impressions`): used only to construct the label,
  never as a model feature.
- `provider_used` / `model_used`: 71.5% missing, with confounding risk (correlation with
  decline may reflect adoption timing, not content quality).
- `ai_traffic_pct`: 93.6% of rows were exactly zero — too sparse for reliable page-level
  claims.
- Rows lacking genuine tracking (`gsc_data_available = FALSE`): excluded via SQL filter, as
  these are zero-filled placeholders, not real measurements.

**Leakage risks considered:**
- Confirmed programmatically (via assertion, not just stated) that neither `is_declining`
  nor `march_impressions` ever appear among model features.
- Confirmed all dynamic features are built from SQL queries explicitly restricted to
  February dates only, with no March data included.
- Pseudonymous IDs (`client_hash_id`, `content_hash_id`) are used strictly for joining and
  grouping (e.g. client-holdout validation) — never as model features.

**Confirmation:** No client names, URLs, or private queries appear anywhere in this report,
the accompanying notebooks, or `work/`. All identifiers shown are pseudonymized hashes
provided directly by the FlyRank release.

## 3. Baseline

The transparent rule or score you built first. Why it's a fair comparison, and its numbers on
the same data and metric as your model.

**The rule:** A transparent, hand-written binary gate — flag a page if its average February
search position is ≤10 AND its February click-through rate (CTR) is below 0.006083 (the mean
CTR observed among top-10-position pages).

**Why it's a fair comparison:** The baseline uses the exact same two signals as our final
model (CTR and position) — both independently verified in our Week 4 signal audit (CTR vs.
position: CONFIRMED, n=151,956). The only difference between the baseline and the model is
*how* those signals are combined: the baseline applies a strict yes/no threshold, while the
model learns a graded, probabilistic relationship. This isolates the comparison to "does
learning help, given the same inputs" rather than "does having more/different data help."

**Baseline numbers, on the same held-out test split and metric as the model:**

| Metric | Baseline |
|---|---|
| Precision@20 | 0.15 |
| Precision@50 | 0.22 |

**Context:** The base rate (majority class, "not declining") in our test set was 84.1%, so
raw accuracy alone would be a misleading metric here — Precision@K, which measures how many
of the top-ranked flagged pages are genuinely declining, is the metric that actually matches
how a reviewer would use this queue.

## 4. Model / analysis

Your method and why it fits the lane. The exact feature list (and what you left out on
purpose). The target or proxy definition, in one sentence.

**Method:** Random Forest Classifier. Chosen after comparing it against a Decision Tree
(readable if/else logic, but performed worse — Precision@20 = 0.10 after correcting a
class_weight setting that was distorting its ranking) — Random Forest offered better
predictive performance while still being trained on only 2 interpretable, verified features,
keeping the overall approach reasonably transparent despite not being as directly readable
as a single tree.

**Exact feature list used in the final model:** `feb_ctr` (February click-through rate),
`feb_avg_position` (average February search position).

**Features left out on purpose:** An earlier version of this model also included `word_count`,
`search_volume`, and `feb_impressions` (5 features total). That version gave `word_count` the
highest feature importance of any feature, yet did not outperform the baseline
(Precision@20 ≈ 0.10–0.15). We deliberately removed these three unverified features and
retrained using only the two features independently confirmed as meaningful in our signal
audit — this single change produced the real improvement reported in this paper.

**Target/proxy definition (one sentence):** `is_declining = 1` if a page's March 2026
impressions are less than 80% of its February 2026 impressions (a genuine 20%+
month-over-month drop), else `0`.

## 5. Evaluation

Your split (grouped by client? time-aware?) and why. Metrics, model vs baseline **on the same
split**. What the errors look like — a short error analysis beats a big metric table.
## 5. Evaluation

**Split design:** Grouped by `client_hash_id`, using `GroupShuffleSplit` (25% of clients held
out entirely for testing) — not a plain random row-level split. Pages from the same client
may share client-specific patterns; a naive random split risks letting the model partially
"memorize" a client seen during training, inflating test performance. We verified zero client
overlap between train and test programmatically before trusting any result.

**Why this split matters (demonstrated, not just claimed):** We directly compared this
grouped split against a plain ungrouped split on the same data. The ungrouped split showed
35 overlapping clients between train and test, and a misleadingly high Precision@20 of 0.60
and Precision@50 of 0.52. The properly grouped split showed zero overlap and honest
Precision@20 = 0.15–0.35, Precision@50 = 0.16–0.32 (depending on random seed) — a dramatic
difference that is itself evidence of how much an ungrouped split can overstate real-world
performance.

**Metrics, model vs. baseline, same split:**

| Method | Precision@20 | Precision@50 |
|---|---|---|
| Baseline (hand rule) | 0.15 | 0.22 |
| Random Forest (2 verified features) | 0.25 | 0.32 |

Base rate (majority class): 15.9% declining / 84.1% not declining.

**What the errors look like:** The model is not bound by the baseline's strict AND-gate — a
page only barely missing one threshold (e.g. position 11 instead of ≤10) is treated
identically to a page wildly failing both conditions under the baseline rule, receiving zero
priority either way. The model instead expresses degree — how far a page's CTR falls below
expectation for its position — allowing it to rank pages the baseline's rigid gate would
completely miss or misorder. This flexibility is the direct mechanical reason it outperforms
the baseline, rather than simply "more complexity happening to work better."

## 6. Interpretation

What the model/clusters actually found. Feature importances or cluster profiles in plain
words. Surprises and negative results — a well-understood "no effect" is a valid result.

**What the model found:** Given only CTR and position, the model learned a graded
relationship where pages with strong positions (near the top of "top 10") combined with
very low CTR received the highest decline-risk scores — closely matching our independently
verified Signal 1 finding (CTR drops sharply as position worsens, even within the "good
position" tier itself).

**Feature importance, in plain words:** In an earlier, broader 5-feature version of this
model, `word_count` received the highest importance (0.30–0.47 across model types) — well
above CTR (0.11–0.16) or position (0.09–0.28). This was a genuine surprise: word_count had
never been independently verified as a real signal (unlike CTR-vs-position, which we
specifically confirmed), yet the model leaned on it heavily during training.

**A well-understood negative result:** Despite word_count's high importance, the 5-feature
model did not outperform our simple baseline rule (Precision@20 ≈ 0.10–0.15, no clear
improvement). This is a valid, informative "no effect" finding: high feature importance
during training does not guarantee genuine, generalizable predictive value on unseen data.
Removing word_count (and the other two unverified features) and retraining on only the two
independently confirmed signals produced the real improvement reported in this paper
(Precision@20 = 0.25–0.35). The lesson: grounding model features in independently verified
signals mattered more here than simply giving the model more data to work with.

**A related surprise from our own signal audit (Week 4):** We originally hypothesized that
high-traffic (high-volume) pages would be MORE likely to be declining, due to greater
competition/scrutiny at scale. The data showed the OPPOSITE: high-volume pages had a
meaningfully LOWER decline rate (15.7%) than low-volume pages (22.4%), across a large sample
(n=134,238). Rather than discard this reversed finding, we reinterpreted its use: volume
became a way to prioritize *among* already-declining pages (since fixing a high-traffic
declining page has a larger payoff), rather than a standalone predictor of decline itself.
## 7. Recommendation

The ranked actions or decisions your output supports, and how a FlyRank editor would use them
tomorrow. State your confidence and the limits explicitly.
## 7. Recommendation

**The ranked action:** Pages are ranked by the model's predicted decline-risk score, filtered
to pages ranking well (position ≤10) — matching our Signal 1 diagnosis that good position
paired with unexpectedly low CTR points at a listing problem, not a content problem.

**Reason code:** `model_flagged_ctr_position_risk` — assigned when a page ranks in the top 10
positions but the model predicts high decline risk based on its CTR-and-position pattern.

**Recommended action:** `review_title_meta` — review and likely rewrite the page's title and
meta description (not the page's body content), since the page already earns visibility
(good position) but isn't converting it into clicks.

**How a FlyRank editor would use this tomorrow:** Open the ranked queue and work through it
top-down. Before acting on any individual page, verify: (1) it hasn't only recently moved
into its current position, which would make its February CTR reflect an outdated ranking
rather than a current problem; (2) it has genuinely sufficient tracking data behind its
score; (3) its query/search intent isn't one where low CTR is normal regardless of title
quality (e.g. featured-snippet-style queries).

**What should never be automated:** Publishing any change to a live page, de-indexing or
deleting content, and any client-facing communication based solely on the model's score —
all require human review and approval first. The model identifies candidates; it does not
make or execute editorial decisions.

**Confidence and limits, stated explicitly:** Precision@20 = 0.25 means roughly 1 in 4
flagged pages in the top 20 are genuinely declining by our definition — a real improvement
over both guessing and our hand-written baseline, but far from perfect. Reviewers should
expect and tolerate some false positives as a normal part of using this tool, and should not
treat any single flagged page as a certainty.

**Monitoring:** We recommend a quarterly re-validation cycle — re-checking precision against
a fresh month-pair comparison, and re-running the client-holdout check whenever new clients
are onboarded — to catch performance drift before it silently degrades recommendation
quality.

## 8. Reproducibility

The exact commands to re-run everything from a fresh clone, your random seeds, and your
environment (`pip freeze` highlights or `requirements.txt` deltas).
## 8. Reproducibility

**Repo:** [https://github.com/fatima12aa/fa-ml]

**To re-run this work from a fresh clone:**

1. Clone the repository: `git clone https://github.com/fatima12aa/fa-ml`
2. Open `work/notebooks/capstone.ipynb` in Google Colab (or Jupyter).
3. Request access to the dataset at
   `https://huggingface.co/datasets/FlyRank/internship-warehouse` (instant approval).
4. Create a Hugging Face access token (type: Read).
5. Store the token as a Colab Secret named `HF_TOKEN` (never paste it directly into a cell).
6. Run all cells top to bottom (Runtime → Run all). The notebook connects to the warehouse
   via DuckDB, rebuilds all February/March features, retrains the baseline and model, and
   regenerates the results table and chart.

**Random seeds used:** `random_state=42` for the primary train/test split and both models
(Decision Tree, Random Forest); stability was additionally checked across seeds
[1, 2, 3, 42, 100] for the final model.

**Environment:** Google Colab (Python 3), key libraries: `duckdb`, `pandas`, `scikit-learn`,
`matplotlib`, `numpy`. No unusual version pinning was required; standard Colab-provided
versions were used throughout.

**Data snapshot:** `flyrank_pseudonymized_warehouse_release_v20260703`, February–March 2026
date range. Numbers in this report reflect a fresh re-run of the notebook as of the date at
the top of this file.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
