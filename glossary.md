# 📖 Plain-Language Glossary

The terms in this repo, in human terms — no prior AI knowledge assumed.

← [Back to README](../README.md)

---

## 🥾 One analogy to hang everything on

Picture a **shoe store with a very well-organized stockroom and a strict fact-checker.**

- A customer asks a question at the counter.
- A **greeter** sends them to the right room of the stockroom.
- Someone grabs the most relevant **index cards** for that shoe.
- A **fact-checker** reads the draft answer against those cards before it's allowed out the door.

Almost every term below is just a part of that store. The mapping is spelled out as we go.

---

## 🧠 The big idea

**RAG (Retrieval-Augmented Generation)**
An AI that looks up real information *before* it answers, instead of answering from memory. Left alone, a language model will confidently make things up. RAG hands it the actual source material first and says "answer from *this*." → *In the store: the clerk always reads from the index cards, never from what they vaguely remember.*

**Hallucination**
When the AI states something that sounds right but isn't in the source — an invented spec, a wrong number, a made-up product. The whole project exists to prevent these. → *The clerk claiming a shoe is waterproof when the card doesn't say so.*

**Grounded**
An answer is "grounded" when every fact in it actually appears in the retrieved source. Grounded = trustworthy. Ungrounded = a hallucination. → *Every claim traces back to an index card.*

---

## 🗄️ How the information is stored and found

**Embedding / vector**
A way of turning text into a list of numbers that captures its *meaning*, so a computer can measure how similar two pieces of text are. "Waterproof hiking boot" lands near "GORE-TEX trail shoe" even though they share no words. → *Filing index cards by meaning instead of alphabetically.*

**Vector database (Pinecone)**
The stockroom for those number-encoded cards. You hand it a question and it returns the closest-in-meaning cards. Pinecone is the specific one used here. → *The organized stockroom itself.*

**Chunking**
How you cut a long document into bite-sized pieces before storing it. Chunk too coarsely and unrelated facts get glued together; chunk badly and a single product's specs get split across two cards. **This project's hardest bug was a chunking bug** — records were being cut in half, so the AI read specs off the wrong shoe. The fix: **one product = one card, always.** → *Deciding how much goes on each index card. One shoe per card, never half of two.*

**Namespace / namespace sharding**
Splitting the stockroom into separate rooms — one per product line (climbing, skiing, hiking…). A climbing question only ever searches the climbing room. This stops a "Ribelle" trail-running shoe from getting confused with a "Ribelle" mountaineering boot. → *Separate rooms so you never pull from the wrong shelf.*

**Retrieval / top-k**
The act of grabbing the most relevant cards for a question. "Top-k" just means "the top *k* closest matches" (e.g. top 4). → *The clerk pulling the 4 most relevant index cards.*

**Context / context pollution**
"Context" is the material handed to the AI to answer from. "Context pollution" is when unhelpful or look-alike cards get mixed in and crowd out the right one — which makes answers *worse*, not better. More context is not automatically better. → *Handing the clerk five cards for similar shoes and watching them grab the wrong one.*

---

## 🚦 The moving parts of this agent

**Classifier / routing**
The step that reads the question and decides which room it belongs to *before* any searching happens. Getting this right is what makes the separate rooms worth having. → *The greeter at the door.*

**Guardrail**
A safety check. This project has two: an **input guardrail** (screens the incoming question — blocks personal data, abuse, off-topic asks) and an **output guardrail** (checks the answer before it's delivered). → *A bouncer at the entrance and a fact-checker at the exit.*

**Faithfulness judge**
The output guardrail specifically — a second AI whose only job is to read the drafted answer against the source cards and rule **PASS** (grounded, send it) or **FAIL** (something's invented, hold it back). → *The fact-checker at the exit.*


**Size conversion (US ↔ EU / Mondo)**
SCARPA lists footwear in EU sizes (ski boots in Mondopoint). A US shoe size has to be *converted* to order the right pair. The agent answers this from an official size-conversion chart stored as a record in each product namespace — not from the model's memory. → *A laminated conversion card kept on every shelf of the stockroom.*

**Withhold**
What happens on a FAIL: instead of risking a wrong answer, the agent politely declines ("I can't verify that from the catalog"). A withheld answer is *safe*; the goal is to withhold only when truly necessary. → *The clerk saying "let me not guess" instead of bluffing.*

**System prompt / temperature**
The **system prompt** is the standing instructions given to an AI ("you answer only from the provided catalog…"). **Temperature** is a creativity dial — set to **0** here, meaning "be as consistent and literal as possible," because a spec lookup should give the same answer every time.

---

## 📊 The metrics (what the Outcomes table means)

**Routing accuracy**
Out of all the questions, how often the greeter sent them to the *right* room. Higher is better. → *93.8% → ~97%.*

**Faithfulness pass rate**
How often the delivered answer was fully grounded in the source (approved by the fact-checker for the right reasons). → *81.8% → 96.7%.*

**Withhold rate**
How often a *correct* answer got held back unnecessarily — the fact-checker being overly cautious. Lower is better (you want it to hold back only real mistakes). → *18.2% → 3.3%.*

**Missed-hallucination rate**
How often a *wrong* answer slipped past the fact-checker — the dangerous failure. Lower is better; here it reached **0** on the test set. → *0.125 → 0.*

**Over-block rate**
The mirror image: how often the fact-checker wrongly rejected a *correct* answer. A judge has to balance this against the missed-hallucination rate — too strict blocks good answers, too loose lets bad ones through.

**Run-to-run stability**
Ask the same question twice — do you get the same answer both times? Because AI is probabilistic, this isn't guaranteed, so it's measured. → *2 of 5 probes were flaky → 50 of 50 stable.*

**Product-mention rate**
A sanity check: did the answer actually name the product that was asked about?

---

## 🧪 How the quality was measured

**Evaluation harness ("eval")**
A built-in test suite: a fixed list of 50 questions with known-correct answers, run through the real pipeline automatically and scored. It's what turns "it feels better" into "withhold rate dropped from 6.5% to 3.3%." → *A secret-shopper program for the store, run on a schedule.*

**Dev set vs. held-out set**
The **dev set** is practice questions you're allowed to tune against. The **held-out set** is a locked exam you only grade against *once* — never used for tuning. Keeping them separate is how you prove an improvement is real and not just memorized. → *Practice tests vs. the real, sealed final exam.*

**Overfitting**
When a fix looks great on the practice questions but flops on the sealed exam — you tuned to the specific examples instead of the underlying skill. Caught exactly once in this project (a judge tweak that aced the dev set and regressed on held-out), which is *why* the sealed exam exists.

**Regression**
When fixing one thing quietly breaks another that used to work. The eval harness is what catches these before they ship.

---

*New terms worth adding? This glossary is meant to grow — open an issue or a PR.*
