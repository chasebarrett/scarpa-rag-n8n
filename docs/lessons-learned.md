# 🧠 Lessons Learned

*v2.0 — September 2026.* Updated after the ingestion re-chunk, the classifier routing fix, and the judge recalibration.

The architecture was the easy part. What follows is what I actually learned by building it, breaking it, measuring it, and — in one case — nearly fooling myself.

← [Back to README](../README.md)

---

## 🧩 The sibling-variant bug — the whole story

The single most instructive failure in the project, because the fix was nowhere near where instinct said it would be.

### 📉 The symptom

Ask *"how much does the Ribelle Cross 2 weigh?"* and the agent answered with the weight of the **Ribelle Cross 2 Mid GTX** — a different, heavier shoe. The catalog carries three near-identical siblings:

| Product | Women's weight |
|---------|----------------|
| `Ribelle Cross 2` (base) | **325g** ← the asked-about shoe |
| `Ribelle Cross 2 GTX` | 355g |
| `Ribelle Cross 2 Mid GTX` | 380g ← what the agent returned |

### ⚠️ The false trail: prompt engineering

The obvious move is to tell the model to respect product boundaries. I added explicit *"never borrow a sibling's value"* rules to all fourteen answer-chain and judge prompts. **They still failed — 3 out of 3.**

That negative result was the most valuable data in the whole exercise. If fourteen carefully-worded prompts can't fix it, the problem isn't the prompts.

### 🔑 The real cause: ingestion

I pulled the *actual* retrieved context for the failing query and read it top to bottom. Two things were wrong, both upstream of any prompt:

1. **Retrieval ranking.** The exact-name match (`Ribelle Cross 2`) ranked *below* two longer sibling variants — because the variant names *contain* the base name as a substring, they scored as more relevant to the query than the base itself.
2. **Chunking.** The markdown was split at a fixed size, which cut product records mid-record. Only the correct record kept its `##` heading; the siblings began with a bare product name. The model was reading specs off whichever record was most prominent, not the one that was asked about.

No prompt fixes a chunk boundary.

### 🛠️ The fix, in two parts

**Part 1 — re-chunk deterministically.** Measure first: the largest of the 83 product records is **1,221 characters.** So a split on the `##` product heading, one record per chunk, is *exact* — every product becomes its own clean chunk, heading intact, with the model name stored as vector metadata for good measure. Because `clearNamespace` is set on every insert, re-ingestion is idempotent and reversible; I validated on one namespace before rolling out to all seven.

**Part 2 — recalibrate the output judge.** Clean chunks fixed the *answer*, but the faithfulness judge still over-blocked correct sibling answers (it, too, was grabbing the wrong sibling's value when the siblings sat together in its context). Fixing the judge is its own story below.

### ✅ Result

*"How much does the Ribelle Cross 2 weigh?"* now returns **325g / 390g** — the correct base model — grounded and judge-approved. Both halves of the bug (the wrong-answer retrieval bug and the wrongly-withheld judge bug) closed. End-to-end withhold rate: 6.5% → 3.3%.

---

## 🎓 Guarding against my own tuning

The judge recalibration is where I almost fooled myself, and the guardrail that stopped me is worth more than the fix.

I built a **fresh dev set** of sibling cases (Rush and Moraine families — products *not* in the quarantined held-out set) and tuned the judge prompt against it. It hit **1.0 accuracy, zero over-blocks.** Looked finished.

Then I measured on the **held-out set** — 20 cases with real captured contexts, never touched during tuning. It had *not* generalized: the dev set turned out to be slightly easier than the hardest held-out cases (one superstring sibling vs two, with the base name nested inside both and placed last). The dev-set 1.0 was **overfitting**, and the held-out set caught it in the open.

That's exactly why the quarantine exists. The rule I kept: **never tune a prompt against the held-out cases — if they're used for tuning, the number stops being evidence.** I measured, I did *not* iterate against the held-out result, and I reported the real generalization instead of the flattering dev number.

The shipped judge is a strict guard: it catches every hallucination in the test set, and on the clean production context (post re-chunk) it correctly delivers the sibling answers it's asked about. The residual — the very hardest nested-name cases on *polluted* context — is a limit of the small judge model, not the prompt, and it doesn't occur in production because the re-chunk removed the polluted context. Chasing it further would mean tuning against the evidence.

---

## 🕸️ Emergent findings

Behaviors that only showed up once the system ran end-to-end — none visible from the graph.

**Guardrail-vs-router order.** A too-strict topical guardrail blocked legitimate customer-service questions *before* the router saw them. Screening and routing have to agree on what counts as on-topic; the screen wins by position.

**The multi-label trap.** A classifier that *can* return multiple labels *will*, given an overlapping query ("how much does the *Crux* cost?" matches Approach and Customer Service). Forcing single-label plus an explicit tie-break was necessary to keep routing deterministic.

**Category descriptions outweigh the system prompt.** A routing rule added to the classifier's system prompt did nothing; the same rule added to the *category description* fixed it. The per-category descriptions are the load-bearing surface.

**Negation backfires as a keyword attractor.** Adding *"never choose this for questions about where a product is made"* to the Customer Service description made "made / origin" into attractors and *broke* origin routing. State the rule positively, in the category that should win. The same trap reappeared with weight questions and was avoided the same way.

**Template placeholders are load-bearing.** Overriding the classifier's system-prompt template without preserving its `{categories}` placeholder silently stripped every routing hint and quietly degraded classification — a change that passed validation and only surfaced in eval.

**Over-tuning is real, and the eval catches it.** A classifier tweak that fixed one boundary row destabilized an adjacent one — and the "fixed" row still withheld anyway. I reverted it. Not every local win is a global win; the full eval is what tells them apart.

---

## 💡 The concepts, understood by building them

- **Retrieval-Augmented Generation** — grounding output in retrieved data rather than the model's parametric memory. The agent answers from the catalog, not from what the model "knows."
- **Context pollution** — more retrieved context isn't better. Near-duplicate chunks compete with the right one and degrade answers. Namespace sharding protects the model from itself.
- **Probabilistic retrieval and generation** — neither vector search nor the LLM "looks up" an answer; both are similarity-based probability work. Treating either as deterministic is a quiet way to ship something that fails unpredictably.
- **Classification as a routing layer** — sending the query to the right namespace *before* retrieval is what makes sharding pay off. Without classification, the sharding is wasted.
- **Guardrails at both ends — and their ordering matters** — input screens what comes in, output verifies the answer didn't drift, and where you place the screen relative to the router changes what legitimately gets through.
- **Two axes govern most failures** — *availability* (does the answer exist in the retrieved context?) and *query type* (how deterministically does the question map to an answer?). Almost every failure mode lives at a specific corner of that grid.
- **Evaluation is the product** — the harness is what turned a pile of prompts into a system I could actually improve. It's the difference between "I think it's better" and "withhold rate 6.5% → 3.3%, and here's the run."

---

## Lesson: adding a capability is a *routing* change, not just a *data* change

**Symptom.** A real user asked *"What size climbing shoe should I pick if I am a size 9 women's?"* and the agent declined. The instinct was "add the data." The data was only half the fix.

**Two bugs, stacked.**
1. **No grounded source.** The catalog stored EU/Mondo sizes only — no US↔EU/Mondo conversion. The faithfulness judge correctly *withheld* rather than invent one. Fix: add SCARPA's official size-conversion chart as one `##` record per category namespace (EU chart for footwear, Mondo for ski). Now there is a legitimate source; the guardrail never had to be loosened.
2. **Wrong route (the real blocker).** Even with the data in place, the classifier still sent the question to **Customer Service** — a static redirect with no retrieval — because the CS description explicitly claimed *"SIZING/FIT ADVICE ('what size should I get')."* The chart was never reached.

**The trap I walked into on the first attempt.** I "fixed" the CS description by adding *"NOT for size conversion…"* — a negation. Routing didn't change. This is the exact **attractor bug** documented earlier in this project: *mentioning* a keyword in a category description makes that category **more** magnetic for it, even when the mention is a negation. Category descriptions outweigh the system prompt.

**The fix that worked.** Strip *every* size token from Customer Service (leave only subjective fit — "runs large/small/narrow/wide"), and give **each product category** positive ownership of size conversion. Category-named size questions now route to the product branch that carries the chart; subjective fit stays Customer Service.

**Verified, with the regression discipline the eval exists for:**

| Probe | Route | Result |
|-------|-------|--------|
| "…size climbing shoe … women's 9?" | Climbing | EU 41 — Judge PASS |
| "…size ski boot … US men's 10?" | Skiing | Mondo 28 — Judge PASS |
| "Do the Drago run large or small?" | Customer Service | correctly *not* rerouted |
| "How much does the Boostic weigh?" | Climbing | 245g — no regression |

**Takeaway.** Shipping a new capability in a routed RAG system means updating *three* layers in lockstep — the **source** (so an answer can be grounded), the **router** (so the question reaches it), and the **regression check** (so the reroute doesn't poach neighbors). Miss any one and the feature silently fails.

---

## Lesson: a classifier that works 95% of the time is a *crash*, not a *degradation*

**Symptom.** A live user asked *"what's a good shoe for bouldering?"* and got **"Error in workflow."** Not a wrong answer — a hard crash, deterministic across retries.

**Cause.** The n8n Text Classifier wraps the model in a structured-output parser whose schema **requires all nine keys** (every category plus `fallback`, `additionalProperties: false`). Spec questions happened to make the model emit the full object; **recommendation** questions made it emit shorthand — `{"Climbing": true}` — which fails schema validation and throws. The failure was invisible until a whole *class* of question (recommendations) was exercised, because the earlier eval set leaned on spec lookups.

**The fix that didn't work, and why.** First instinct was "the model is adding markdown fences" — true, but not the cause. Worse, the anti-fence instruction I added carried the example `e.g. {"Climbing": true}` — which *taught* the model the exact shorthand that fails the schema. Reading the raw model output (`{"Climbing": true}`, clean, no fences) against the enforced schema (all nine keys required) is what located the real cause.

**The fix.** Instruct the model to emit **every** category key as a boolean with exactly one `true`, with a full-object example. Verified across recommendation phrasings (bouldering → Climbing/Origin VS; ski touring → Skiing/4-Quattro SL) and the size and spec paths, with no regression.

**Takeaway.** With a strict structured-output parser, "the model picked the right category" isn't enough — the *shape* of the output has to match on every call, or the node throws and takes the workflow down. Test the output **format** across every question *class* (spec, recommendation, conversion, refusal), not just a representative few, because a format bug hides until the untested class shows up in production.
