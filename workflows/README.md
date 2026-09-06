# 🧰 The Workflow

[`scarpa_pipeline.json`](scarpa_pipeline.json) is the complete n8n workflow — all 84 nodes: the shared chat/evaluation entry, the input guardrail, the classifier, seven namespace-scoped retrieval branches, the per-category faithfulness-judge clusters, and the evaluation scoring path. Import it into any n8n instance to inspect or run the exact pipeline this repo describes.

> **No secrets are in this file.** Credentials are stored as *references only* (`OpenAI account`, `Pinecone account`) — you supply your own keys on import. See the security note at the bottom.

---

## Import it

1. In n8n: **Workflows → ⋯ (top-right) → Import from File…**
2. Select `scarpa_pipeline.json`.
3. The workflow opens with two credential placeholders flagged. Create/attach your own:
   - **OpenAI** (`openAiApi`) — for embeddings (`text-embedding-3-small`) and the GPT models used for classification, answering, and judging.
   - **Pinecone** (`pineconeApi`) — for the vector store.
4. It arrives **inactive** by design. Activate it only when you're ready (see the cost note).

## What it expects to exist

The workflow queries a Pinecone index named **`scarpa-products`** with **one namespace per product line** (`trail_running`, `approach`, `climbing`, `mountaineering`, `skiing`, `lifestyle`, `hiking`). To reproduce results end-to-end you'll need to ingest the catalogs in [`../data/`](../data/) into that index using the chunking rule described in [`../docs/architecture.md`](../docs/architecture.md) (one product record per chunk).

## ⚠️ Before you activate

Every message runs real OpenAI + Pinecone calls against **your** accounts. If you expose the chat trigger publicly, add a funded key with a **spending cap** and consider rate-limiting first — a public, unauthenticated endpoint is an open door.

---

## 🔒 Security note

This export was scanned before publishing: no API keys, tokens, or credential secrets are present — only `{id, name}` credential *references*, which are meaningless without access to the original instance. The instance fingerprint (`meta.instanceId`) was removed, and the workflow was set to inactive. The public chat `webhookId` is intentionally retained (it's the already-public demo endpoint).
