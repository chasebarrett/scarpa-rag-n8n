# 📊 Evaluation

The evaluation harness is the part of this project I'd defend hardest. A probabilistic system you can't measure is a system you can't improve — every "it feels better" is unfalsifiable. This is how the measurement is built, what it measures, and how honest the numbers are.

← [Back to README](../README.md)

---

## 🚪 Same door as live chat

The harness feeds the **same pipeline entry point** as live chat. An evaluation trigger reads questions from an n8n Data Table, passes them through the shared `Normalize Input` step, and everything downstream — guardrail, classifier, retrieval, answer chain, judge — is the exact code a real user hits. A `checkIfEvaluating` gate is the only divergence: it routes eval runs to the scoring nodes.

If the eval ran through a parallel mock, it would measure the mock. Sharing the entry point is what makes the numbers mean something. → [`architecture.md`](architecture.md#-one-entry-point-for-chat-and-evaluation)

---

## 📋 The dataset

**50 curated questions** (→ [`../eval/eval_questions.csv`](../eval/eval_questions.csv)), each with an `expected_category`, an `expected_fact`, an `expected_product`, and a `purpose` note documenting *why the row exists.* The set is built to exercise real behavior and known edges, not just happy paths:

| Coverage | Example rows |
|----------|-------------|
| Straightforward spec lookups | drop, midsole, outsole, weight, flex |
| Grounded recommendations | "any hiking boots like the Terra GTX?" |
| Honest non-answers (sparse data) | Maestrale RS flex, Chimera last |
| Name-family collisions | Mojito vs Mojito Wrap; Ribelle Cross siblings |
| Gender-variant columns | women's 4-Quattro GT weight & flex |
| Customer-service redirect | price, where-to-buy, fit advice |
| Guardrail probes | PII injection; the Tegu topical false-positive |
| Documented boundaries | cross-category comparisons; list queries |

Roughly a third of the rows are *regression controls and boundary probes* — they exist to catch a fix in one place breaking something that already worked, or to confirm a known limitation still behaves as documented rather than degrading further.

---

## 📐 What's scored

Each row runs through the live pipeline **twice** (100 executions total), scored on:

- **Routing accuracy** — did the query reach the expected category?
- **Answer correctness / faithfulness pass rate** — was the answer delivered and grounded, or wrongly withheld?
- **Withhold rate** — how often a correct answer was held back.
- **Product-mention rate** — did the answer actually name the expected product?
- **Run-to-run stability** — same question, same routing and status across both passes?

Cross-category boundary rows (Gap A) are marked non-scorable for routing, because there is no single correct category to route them to — scoring them would penalize the design for a limitation it documents.

---

## 📈 Results

| Metric | Baseline | After remediation |
|--------|----------|-------------------|
| Routing accuracy | 93.8% | ~97% |
| Faithfulness pass rate | 81.8% | 96.7% |
| Withhold rate | 18.2% | 3.3% |
| Run-to-run stability | 2 of 5 probes flaky | 50/50 stable |
| Judge missed-hallucination rate | 0.125 | 0 |

**On the ~97% routing figure.** Routing sits at 97.9% on a clean run and dips to ~96.9% when two lexically-adjacent name boundaries (Ribelle **Cross** vs the **Cru**x approach shoe; "do you *have*…" reading as an availability question) flip on a given pass. Rather than quote a single best-case number, the honest statement is **~97% with ~1–2 rows of run-to-run variance at those boundaries.** Chasing the last point risks over-tuning the classifier, which is its own documented lesson (→ [`lessons-learned.md`](lessons-learned.md)).

---

## 🧪 The harness has tiers

Evaluation isn't one workflow — it's a small suite, each tier answering a different question:

| Tier | What it answers | Scale |
|------|-----------------|-------|
| **Canary + Repeat** | Did anything obvious break, and is it stable run-to-run? | 14 canaries × 3 |
| **Full Pipeline Eval** | The headline metrics above, end-to-end | 50 rows × 2 |
| **Judge Meta-Eval** | Is the *judge itself* correct on clean cases? | 36 curated cases |
| **Held-Out Judge Set** | Does the judge generalize to unseen sibling cases? *(quarantined — never tuned against)* | 20 cases |

The split between the **Judge Meta-Eval** (tuning set) and the **Held-Out Judge Set** (measurement-only) is deliberate: it's what turned "the judge looks better" into "the judge generalizes" — or, in one honest instance, caught a prompt change that *didn't* generalize before it shipped. → [`lessons-learned.md`](lessons-learned.md)

---

## ▶️ How to run it

The harness runs from the n8n UI (a manual trigger on each eval workflow). Because it enters through the Execute Sub-workflow node, one wiring quirk is worth noting: that node returns results on **output index 1**, so the scorers are wired there — n8n's static validator warns about it, and the warning is expected.

A native four-metric run (including true Retrieval Hit Rate) must be launched from the n8n evaluation UI directly, since the programmatic execution path can't fire the evaluation trigger. The `expected_product` column is populated specifically so that native run can score retrieval.
