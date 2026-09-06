# 🛡️ Guardrails

Two guardrails, at opposite ends of the pipeline, guarding against opposite failures. The input guardrail screens what comes *in*; the output guardrail verifies the answer didn't drift on the way *out*.

← [Back to README](../README.md)

---

## 🔒 Input guardrail (before routing)

Five checks run on every question before it reaches the classifier:

| Check | Guards against |
|-------|----------------|
| **PII** | The user pasting a real SSN, card number, etc. |
| **Jailbreak** | Attempts to override the system's instructions |
| **NSFW** | Off-domain unsafe content |
| **Topical alignment** | Questions unrelated to SCARPA footwear |
| **Prompt injection** | Instructions smuggled inside the question |

Anti-injection instructions are *also* baked into every answer-chain prompt, so the defense doesn't rest on a single node.

### ⚠️ Ordering has teeth

The input guardrail sits *before* the router, and that ordering is a design decision, not an accident — one that bit before it was tuned. A too-strict **topical alignment** check was blocking legitimate customer-service questions (pricing, availability) at high confidence *before the router ever saw them.* The screen and the router have to agree on what counts as on-topic; when they disagree, the screen wins by position, and legitimate traffic dies silently upstream.

There was a matching false-positive: the topical guardrail once rejected the **Tegu** — a real catalog sandal — at confidence 1.0, because it read as too casual to be "footwear." The fix was an overriding rule: *any input naming a SCARPA product, product line, or category of footwear SCARPA makes is on-topic.* Both failures were invisible until exercised end-to-end (see [`evaluation.md`](evaluation.md)).

---

## ✅ Output guardrail: the faithfulness judge

After the answer chain writes a response, a **per-category faithfulness judge** compares that answer against freshly-retrieved context and returns a `PASS` or `FAIL`. A gate delivers on PASS and withholds (with a safe message) on FAIL.

The judge is calibrated to the agent's **actual answer mode**. This agent doesn't only extract verbatim facts — it also makes **grounded recommendations** ("the Crux is a good pick for talus because…"). A naive "every sentence must be a verbatim spec" judge would reject those. So the judge passes three things and fails everything else:

1. **Grounded facts** — the value appears in the retrieved context.
2. **Honest non-answers** — "I don't have that spec" is *about the absence of data*, not a claim about the product, so it's inherently supported.
3. **Grounded recommendations** — calling a product good for a use case, when the product and the cited features are in the context.

It fails any product name, spec, number, material, or certification that's **absent from or contradicts** the context — including an answer that pairs correct facts with one invented detail.

---

## 🎛️ Calibrating the judge (and guarding against self-deception)

The judge is itself an LLM, which means *the judge can be wrong.* So it gets measured like anything else — a meta-evaluation that scores whether the **judge** is correct, not just whether answers pass it.

Its hardest class is **sibling-variant discrimination**: passing a correct spec for the base product while a near-identical sibling with a *different* value sits in the same retrieved context. Getting this right without breaking everything else took a disciplined loop:

- **Tune on a fresh dev set** — sibling cases built from products *not* in the held-out set.
- **Measure on a quarantined held-out set** — 20 cases, real captured retrieval contexts, **never used to tune any prompt.** If they're used for tuning, the number stops being evidence.

The result was a judge that catches **every** hallucination in the test set (missed-hallucination rate → 0) while, on the real production context (clean, one-record-per-chunk), correctly delivering the sibling answers it's asked about. The end-to-end withhold rate dropped from 6.5% to 3.3%. The full walk-through — including where the dev set *overfit* and the held-out set caught it — is in [`lessons-learned.md`](lessons-learned.md).

> The judge's job is asymmetric: letting a hallucinated spec through is worse than withholding a correct answer. The calibration leans into that — it is tuned to be a strict guard first and a permissive one second.
