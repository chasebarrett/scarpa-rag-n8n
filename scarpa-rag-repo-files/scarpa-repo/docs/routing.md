# 🔀 Routing

The classifier is the layer that makes namespace sharding pay off. Without it, sharding is wasted — you'd have neatly partitioned inventory and no way to send a query to the right partition. This is how the routing is designed, and the failures that shaped it.

← [Back to README](../README.md)

---

## Classifier-plus-switch, not an agent

Routing is a **deterministic Text Classifier feeding a Switch**. The classifier emits exactly one category label; the switch sends the query down exactly one of eight branches. There is no tool-calling loop, no autonomous re-planning.

That determinism is a feature, not a limitation. It makes routing **predictable** (the same query always takes the same path), **debuggable** (you can read which terminal node fired), and **measurable** (routing accuracy is just "did it reach the expected category?"). An autonomous agent would trade all three away for flexibility this problem doesn't need.

---

## 🎯 Single-label, with a tie-break

Left to its own devices, a classifier will happily return *two* categories for an overlapping query. "How much does the Crux cost?" matches both **Approach** (it names a product) and **Customer Service** (it asks a price). Early on, the classifier did exactly this — populated two outputs at once — and the query fired two terminal nodes.

The fix was an **absolute single-label instruction plus explicit precedence rules**: transactional intent (price, stock, where-to-buy, returns, fit advice) wins over a bare product name. "How much does the Crux cost?" → Customer Service. "What outsole does the Crux use?" → Approach.

---

## 📏 The size-vs-fit split

The sharpest distinction the classifier has to hold:

| Question | Route | Why |
|----------|-------|-----|
| "What sizes does the Crux come in?" | **Approach** | The sizes offered are a *spec* — they're in the catalog |
| "Do the Crux run large?" | **Customer Service** | Fit *advice* — not in the catalog, a judgment call |

Same product, same topic area, opposite routes. The classifier prompt encodes this explicitly, because the catalog can answer one and honestly cannot answer the other.

---

## 🚫 Two branches that skip retrieval

- **Customer Service** — price, availability, where-to-buy, orders, shipping, returns, warranty, and fit advice are legitimate questions a spec catalog simply can't answer. They route to a static redirect (`customerservice@scarpa.com`), bypassing both retrieval and the faithfulness judge — a fixed message is grounded by construction, so there's nothing to check.
- **Fallback** — anything the classifier can't confidently place returns a short "I can help with these categories — could you rephrase?" reply.

---

## 🧬 The Ribelle problem: one name, three categories

"Ribelle" is a product *family* that spans three namespaces:

- **Ribelle Run** (Run 2, Run 2 GTX, Run Kalibra) → **Trail Running**
- **Ribelle Cross** (Cross 2, Cross 2 GTX, Cross 2 Mid GTX) → **Hiking**
- **Ribelle HD / Tech / Ice** → **Mountaineering**

A classifier matching on "Ribelle" alone routes a third of these wrong. The prompt carries an explicit family-disambiguation rule that keys on the *full* model name, not the shared prefix. (The downstream consequence — sibling variants inside a single namespace bleeding into each other's specs — is a retrieval problem, covered in [`lessons-learned.md`](lessons-learned.md).)

---

## ⚠️ Lessons the classifier taught

Two findings worth keeping, both discovered by measurement rather than inspection:

**Category descriptions outweigh the system prompt.** When a routing rule wasn't being honored, editing the system prompt did nothing — editing the *category description* fixed it. The classifier weights the per-category descriptions far more heavily than global instructions.

**Never put a negative rule inside the category it warns against.** An attempt to stop origin questions ("where is X made?") from routing to Customer Service by adding *"never choose this for questions about where a product is made"* to the Customer Service description **backfired** — the words "made / manufactured / origin" became keyword attractors, and the classifier didn't honor the negation. The fix was to state the rule *positively* in the categories that *should* win, and leave the warned-against category's description clean. The same trap reappeared later with weight/measurement questions and was avoided the same way. → [`evaluation.md`](evaluation.md)
