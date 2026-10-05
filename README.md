<div align="center">

# 🔍 NextLeap — RAG Implementation: Pinecone Vector DB + Gemini Embeddings

**A complete Retrieval-Augmented Generation pipeline in n8n. A scheduled job ingests documents from Google Drive, embeds them with Gemini, and stores the vectors in Pinecone. A chat agent then answers questions by searching that index as a tool — the full RAG loop, end to end.**

[![RAG](https://img.shields.io/badge/Pattern-RAG-8A2BE2?style=for-the-badge)](https://en.wikipedia.org/wiki/Retrieval-augmented_generation)
[![n8n](https://img.shields.io/badge/n8n-Workflow-ea4b71?style=for-the-badge&logo=n8n.io)](https://n8n.io)
[![Pinecone](https://img.shields.io/badge/Vector%20DB-Pinecone-2563EB?style=for-the-badge&logo=pinecone)](https://www.pinecone.io)
[![Gemini](https://img.shields.io/badge/Embeddings-Google%20Gemini-4285F4?style=for-the-badge&logo=google)](https://ai.google.dev)
[![Google Drive](https://img.shields.io/badge/Source-Google%20Drive-0F9D58?style=for-the-badge&logo=googledrive)](https://developers.google.com/drive)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

</div>

---

## 📖 Overview

This repository contains one n8n workflow implementing **RAG (Retrieval-Augmented Generation)** — the standard way to make an LLM answer questions about *your* documents instead of relying on its training data.

**File:** [`RAG implementation - Pinecone vector DB + Gemini embeddings.json`](RAG%20implementation%20-%20Pinecone%20vector%20DB%20%2B%20Gemini%20embeddings.json) · 14 nodes

### Why RAG exists

An LLM knows nothing about your private documents, and fine-tuning is expensive and rigid. RAG solves this by **retrieving relevant passages at question time** and putting them in the prompt.

```text
Without RAG:   Question ──────────────────────────► LLM ──► ❌ "I don't know"

With RAG:      Question ──► embed ──► vector search ──► top passages ──┐
                                                     │                 ▼
               Answer ◄── LLM ◄── prompt + passages ◄────────────────┘
```

The model never learns your data. It just *reads* it at the moment it's needed — which means you can update a document and the change is live immediately, with no retraining.

### The two halves of this workflow

RAG is really **two pipelines that happen to share a vector database**. n8n keeps them in one workflow, separated by sticky notes:

| Pipeline | Trigger | Job | Runs |
| --- | --- | --- | --- |
| **① Ingestion** | ⏰ Schedule Trigger | Drive → embed → **insert into Pinecone** (`ns: rag-docs`) | Every 30 min |
| **② Query** | 💬 Chat Trigger | Question → embed → **search Pinecone** → LLM answers | On demand |

> ⚠️ **Most RAG bugs live in the seam between these two halves** — specifically, using *different embedding models* on each side. See [The Golden Rule](#-the-golden-rule).

---

## 🏗️ Architecture

```mermaid
flowchart TB
    subgraph ING["① INGESTION — sticky note: 'Creating vector database on Pinecone'"]
        ST["⏰ Schedule Trigger<br/><i>every 30 min</i>"] --> SF["📁 Search files and folders<br/><i>query: 'Enterprise'</i>"]
        SF --> DF["⬇️ Download file<br/><i>from Drive</i>"]
        DF --> PV["🗄️ Pinecone Vector Store<br/><i>insert · clear ns rag-docs</i>"]
        DL["📄 Default Data Loader<br/><i>type: binary</i>"] -.->|"ai_document"| PV
        EM["🔢 Embeddings Google Gemini<br/><i>gemini-embedding-2</i>"] -.->|"ai_embedding"| PV
    end

    subgraph PINE["🗄️ Pinecone index"]
        VECS[("▦▦▦ vectors · n8n-gemini-rag")]
    end

    subgraph QRY["② QUERY — sticky note: 'Leveraging RAG to answer'"]
        CT["💬 When chat message received"] --> AG["🤖 AI Agent"]
        GL["💬 Google Gemini Chat Model"] -.->|"ai_languageModel"| AG
        SM["🧩 Simple Memory<br/><i>key: chatId</i>"] -.->|"ai_memory"| AG
        RT["🔍 Pinecone Vector Store1<br/><i>retrieve-as-tool · ns: rag-docs</i>"] -.->|"ai_tool"| AG
        EM2["🔢 Embeddings Google Gemini1<br/><i>gemini-embedding-2</i>"] -.->|"ai_embedding"| RT
    end

    PV ==>|"insert into ns: rag-docs"| PINE
    RT ==>|"similarity search"| PINE

    style ING fill:#1a2e1a,color:#fff,stroke:#4a6a4a
    style QRY fill:#1a1a2e,color:#fff,stroke:#4a4a6a
    style PINE fill:#2e1a3e,color:#fff,stroke:#6a4a8a
    style PV fill:#2563EB,color:#fff,stroke:#1d4ed8
    style RT fill:#2563EB,color:#fff,stroke:#1d4ed8
    style AG fill:#8A2BE2,color:#fff,stroke:#6a1b9a
    style EM fill:#4285F4,color:#fff,stroke:#2a56b0
    style EM2 fill:#4285F4,color:#fff,stroke:#2a56b0
```

### Retrieval as a tool

The single most important design decision in the query half:

```json
"mode": "retrieve-as-tool"
```

This exposes the vector store to the AI Agent as a **tool** rather than hard-wiring retrieval into the chain. The agent decides whether a question actually needs your documents — which saves a vector search on small talk, and lets it search multiple times for a multi-part question.

| Mode | Behaviour | Use when |
| --- | --- | --- |
| `retrieve-as-tool` ✅ | Agent chooses when to search | You want an agent that may or may not need the docs *(this repo)* |
| `retrieve-as-rag` | Always retrieves, then answers | Every question is doc-based; simpler and more predictable |

---

## 📋 Node Reference

### ① Ingestion

| Node | Type | v | Role |
| --- | --- | --- | --- |
| **Schedule Trigger** | `scheduleTrigger` | 1.3 | Runs every 30 minutes (`interval: [{}]`) |
| **Search files and folders** | `googleDrive` | 3 | `resource: fileFolder`, query `"Enterprise"`, root folder |
| **Download file** | `googleDrive` | 3 | `operation: download` — fetches the binary |
| **Pinecone Vector Store** | `vectorStorePinecone` | 1.3 | `mode: insert` → index `n8n-gemini-rag`, namespace `rag-docs`, `clearNamespace` ✅ |
| **Default Data Loader** | `documentDefaultDataLoader` | 1.1 | `dataType: binary` — extracts text from the file |
| **Embeddings Google Gemini** | `embeddingsGoogleGemini` | 1 | `models/gemini-embedding-2` |

### ② Query

| Node | Type | v | Role |
| --- | --- | --- | --- |
| **When chat message received** | `chatTrigger` | 1.4 | Chat entry point |
| **AI Agent** | `agent` | 3.1 | Orchestrates search + answer (no fixed prompt) |
| **Google Gemini Chat Model** | `lmChatGoogleGemini` | 1.1 | The answering LLM |
| **Simple Memory** | `memoryBufferWindow` | 1.3 | `sessionIdType: customKey`, `sessionKey: ` `` ={{ chatId }} `` ✅ per-session |
| **Pinecone Vector Store1** | `vectorStorePinecone` | 1.3 | `mode: retrieve-as-tool` → index `n8n-gemini-rag`, namespace `rag-docs` |
| **Embeddings Google Gemini1** | `embeddingsGoogleGemini` | 1 | `models/gemini-embedding-2` |

### The tool description

The vector store's description is what the agent reads to decide whether to search:

**Before ([bug #5](#-known-bugs-in-this-workflow)):**

> Use this tool to retrieve relevant product documentation to answer any questions on the product

**After** — matches what's actually in the index:

> Use this tool to search the India news podcast PRD for details about the project, its goals, features and requirements

Same lesson as the [MCP workshop](https://github.com/Gursimaran21/NextLeap-Built-MCP-Server-and-Client-04-October-2026): **the description is the prompt.** The original said "product documentation" while the corpus held a news-podcast PRD, so the agent searched for documents that didn't exist — and returned confident, wrong answers rather than admitting ignorance.

### Credentials required

| Credential | Used by | Notes |
| --- | --- | --- |
| **Google Drive account** | Search + Download | Scope: `https://www.googleapis.com/auth/drive` |
| **Pinecone account** | Both vector store nodes | API key from [pinecone.io](https://www.pinecone.io) |
| **Google Gemini (PaLM) API** | Chat model + both embeddings | One key can serve all three nodes |

> The original export referenced **two different** Google Drive credentials on the same workflow — almost certainly an artefact of the live session. A single credential works fine for both nodes.

---

## 🔐 The Golden Rule

> **The model that embeds a document must be the exact same model that embeds the question.**

A vector is only comparable to another vector if both were produced by the same model, at the same dimensionality, in the same space. Mix `gemini-embedding-2` on the ingestion side with a different model on the query side and **retrieval fails silently** — you get results, they're just meaningless. No error, no warning.

That's why this workflow has **two separate Embeddings nodes** (`Embeddings Google Gemini` and `Embeddings Google Gemini1`). n8n attaches sub-nodes per usage site, so you must configure both — and keep them identical.

**If you ever change the embedding model, you must re-embed the entire corpus.** Mixing vectors from two models in one index will quietly corrupt your results.

---

## 🚨 Known bugs in this workflow

Four real issues were found by reading the export. **Four are now fixed in the committed JSON**;
one (#3) still needs a node added by hand, because it can't be expressed as a parameter change.

| # | Bug | Severity | Status |
| --- | --- | --- | --- |
| 1 | Drive search decorative — hardcoded `fileId` | 🔴 | ✅ **Fixed** |
| 2 | Duplicates accumulate on every schedule run | 🔴 | ✅ **Fixed** |
| 3 | No text splitting | 🟡 | ⚠️ **Manual step** — add a node, see below |
| 4 | Memory shared by every user | 🟡 | ✅ **Fixed** |
| 5 | Tool description doesn't match the corpus | 🟡 | ✅ **Fixed** |

The originals are shown below so you can see what changed and why — this is the more useful half
of the lesson.

<details>
<summary><b>Bug 1 (before) — the Drive search was decorative</b></summary>

`Search files and folders` queries Drive for `"Enterprise"` — but `Download file` **ignored its input** and used a hardcoded `fileId`:

```json
"fileId": {
  "__rl": true,
  "value": "YOUR_GOOGLE_DRIVE_FILE_ID",
  "mode": "list"
}
```

So the same single document was ingested every run, no matter what the search returned. The search step did nothing.

**Fix applied** — File ID switched to expression mode:

```js
 ={{ $json.id }}
```

That takes the ID from each search result, making the pipeline genuinely dynamic.

</details>

<details>
<summary><b>Bug 2 (before) — duplicates accumulated every 30 minutes</b></summary>

`Schedule Trigger` fires every 30 minutes with `mode: insert`. Pinecone's insert creates **new** vectors every time — it never overwrites. Your index would fill with hundreds of copies of the same document, and retrieval would return near-duplicates.

**Fix applied** — both halves now share a namespace, and insertion clears it first:

```json
"options": {
  "clearNamespace": true,
  "pineconeNamespace": "rag-docs"
}
```

> ⚠️ **The trap in this fix.** `clearNamespace` only fires when a namespace is *also* set. Looking
> at the node source, the delete is guarded by `if (options.pineconeNamespace && options.clearNamespace)`
> — set `clearNamespace` alone and it silently does nothing.
>
> Worse, clearing a namespace you don't own wipes **every** document in it. Use a dedicated
> namespace name like `rag-docs`, never the default.
>
> And the matching `retrieve-as-tool` node must query **the same namespace**. Clear on insert but
> read from default and you get an empty index that looks like a broken agent.

**Trade-off:** this makes re-runs idempotent (each run replaces the corpus) rather than
incremental. That's the right default for a small, single-source corpus. For real incremental
updates you'd want deterministic vector IDs and an upsert path instead.

</details>

<details>
<summary><b>Bug 3 — still open: no text splitting</b></summary>

The document goes from `Default Data Loader` straight into the vector store. There is **no `Document Splitter` node**, so a large document becomes one enormous chunk — which embeds poorly and wastes tokens on every query.

**Still to do** — insert a **Document Splitter** (`recursiveCharacterTextSplitter`) between the loader and the vector store, set to ~1000 characters with ~100 overlap.

This one can't be fixed by editing a parameter, because it means inserting a new node and rewiring
the graph. That's why it's left as an exercise rather than patched silently — if this repo patched
it for you, you'd import the workflow without ever learning to read a graph.

</details>

<details>
<summary><b>Bug 4 (before) — memory was shared by every user</b></summary>

```json
"sessionIdType": "customKey",
"sessionKey": "rag-session"
```

`rag-session` is a **fixed** key, so every visitor to the chat shared one conversation history. Person A's questions leaked into Person B's context.

**Fix applied** — memory keyed off the chat session:

```js
 ={{ $('When chat message received').item.chatId }}
```

</details>

<details>
<summary><b>Bug 5 (before) — tool description didn't match the corpus</b></summary>

The description said *"product documentation"*, but the ingested document is `india-news-podcast-PRD.md` — a news-podcast PRD. The agent looked for product docs that don't exist.

**Fix applied** — describes what's actually indexed:

> Use this tool to search the India news podcast PRD for details about the project, its goals, features and requirements

</details>

---

<details>
<summary><b>Not bugs, but worth knowing</b></summary>

**`models/gemini-embedding-2` is a real model.** An earlier version of this README flagged it as
possibly non-existent. It is correct — it's Google's first multimodal embedding model, GA since
April 2026, and handles text, images, video, audio and PDF in one space.

Two things to know about it:

- For **text-only** work, `gemini-embedding-001` is still available and remains the simpler choice.
- Its embedding space is **incompatible** with `gemini-embedding-001`. If you switch between them,
  you must re-embed the entire corpus — you cannot mix vectors from the two models in one index.

</details>

## 🚀 Setup Guide

### Step 1 — Create the Pinecone index

1. Sign in at [app.pinecone.io](https://app.pinecone.io).
2. **Create Index** → name it `n8n-gemini-rag` (or update the workflow to match).
3. **Dimension** must match your embedding model. `gemini-embedding-2` supports **128–3072** dimensions and recommends **768** or **1536** — use one of those. A mismatch here is the single most common RAG setup failure.

   > 💡 Using `gemini-embedding-001` instead? It also goes up to 3072, but its embedding space is
   > **incompatible** with `gemini-embedding-2`. Pick one and keep both Embeddings nodes on it.
4. **Metric:** `cosine` (the usual default).

> 💡 Run the ingestion half once successfully *before* building the query half. You should see vector count climb in the Pinecone console. An empty index makes the chat look broken for reasons that have nothing to do with the agent.

### Step 2 — Import and attach credentials

1. **Workflows → Import from File** → the workflow JSON.
2. Create and attach:

   | Credential | How |
   | --- | --- |
   | Google Drive | OAuth 2.0 with `drive` scope |
   | Pinecone | API key from the Pinecone console |
   | Google Gemini | API key from [aistudio.google.com/apikey](https://aistudio.google.com/apikey) |

3. Open both **Pinecone Vector Store** nodes and select your index.

### Step 3 — Point ingestion at your own document

**Download file → File ID is now `={{ $json.id }}`** ([bug #1](#-known-bugs-in-this-workflow) fixed), so ingestion is driven entirely by whatever **Search files and folders** returns.

To change the source document, edit the search instead:

| Node | Field | Current value | Change it to |
| --- | --- | --- | --- |
| **Search files and folders** | Query String | `Enterprise` | your file's name, e.g. `india-news-podcast-PRD` |
| **Search files and folders** | Folder | `/ (Root folder)` | a specific Drive folder, to narrow the search |

If the search returns nothing, check the **query matches** — Drive's `search` resource filters on
name, and `"Enterprise"` returns only what literally contains that string.

> 💡 Alternatively pin it back to one fixed file: set **Download file → File ID** back to list
> mode and pick the document manually. That reintroduces [bug #1](#-known-bugs-in-this-workflow)
> — the search node becomes decorative — but it's fine for a single known document.

### Step 4 — Run the ingestion half

Select from **Schedule Trigger** through **Pinecone Vector Store**, then hit **Execute Workflow**. Confirm:

- The Drive node returns your file
- The embeddings node produces vectors
- Pinecone's vector count increases

> ⚠️ **Don't skip this step before activating the workflow.** If you activate first, the
> **Schedule Trigger fires every 30 minutes on its own** and re-ingests — and with
> `clearNamespace` on, each run **wipes and rebuilds** namespace `rag-docs`.
>
> Run the ingestion half manually until you're satisfied with the content, then activate. The
> trigger's cadence is `interval: [{}]` in the export, which n8n reads as its 30-minute default;
> change it under **Trigger Settings** if 30 minutes is wrong for you.

While you're in there: **change the query in `Search files and folders`** to match your actual
document, and confirm the **Pinecone Vector Store1** node also points at namespace `rag-docs`.

### Step 5 — Activate and chat

1. Click **Active**.
2. Open the chat and ask something only your document can answer.
3. Open the execution log — you'll see the vector store appear as a **tool call**, followed by the Gemini answer.

### Step 6 — Fix the known bugs

Bugs **#1, #2, #4 and #5 are already fixed** in the JSON you just imported. **#3 (Document Splitter)
is not** — it needs a new node wired into the graph, so it's left for you to do:

> Insert a **Document Splitter** (`recursiveCharacterTextSplitter`) between
> **Default Data Loader** and **Pinecone Vector Store**. Set chunk size ~1000 characters with
> ~100 overlap. Connect the loader's `ai_document` output into it, then its `ai_document` output
> into the vector store's `ai_document` input.

Adding that node is also the exercise — it's the only way to learn how the `ai_*` typed
connections differ from the normal `main` flow.

---

## 🎬 Demo Prompts

Once indexed, try questions that test whether retrieval is genuinely working:

```text
✅  What is this project's target audience?          ← answerable only from the doc
✅  Summarise the key milestones in this PRD.        ← good multi-passage test
✅  What are the open questions still listed?        ← tests metadata retrieval

⚠️  What is the capital of France?     ← correct behaviour is to NOT call the tool
⚠️  Summarise it.                     ← follow-up; tests memory + a second search
```

> 💡 The third case is the interesting one. Because the store is a **tool**, the agent should answer "I don't have that in the document" rather than hallucinating. If it makes something up, your tool description or embedding setup needs work.

---

## 🔧 Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Could not find embedding model` | Invalid/unsupported model ID | Verify the exact ID in [Google's model list](https://ai.google.dev/gemini-api/docs/models) |
| Answers ignore the document | Chat + embedding models mismatch | Confirm **both** embeddings nodes use the same model |
| Pinecone rejects the insert | Wrong dimension for the index | `gemini-embedding-2` supports 128–3072; recreate at 768 or 1536 |
| Retrieval returns junk | Vectors in the index came from a different model | Delete the index and re-embed everything |
| Near-duplicate results | [Bug #2](#-known-bugs-in-this-workflow) | ✅ Fixed — `clearNamespace` on namespace `rag-docs` |
| Empty results despite a populated index | Namespace mismatch between the two Pinecone nodes | Both must use `rag-docs`; ✅ Fixed here |
| Same document every run | [Bug #1](#-known-bugs-in-this-workflow) — hardcoded file ID | ✅ Fixed — `={{ $json.id }}` |
| One giant chunk retrieved | [Bug #3](#-known-bugs-in-this-workflow) | Add a Document Splitter node — not yet done |
| Chat "knows" other users' questions | [Bug #4](#-known-bugs-in-this-workflow) | ✅ Fixed — keyed off `chatId` |
| Agent never calls the tool | Tool description doesn't match the question | Rewrite the description to name your actual content |
| Empty index | Ingestion half never ran successfully | Run it manually first; check the Drive node |
| Dimension mismatch errors | Pinecone index built for another model | Index dimension must equal the embedding model's output size |

---

## 📊 Choosing a Vector Database

Pinecone isn't the only option — it's just fully managed and the least setup friction.

| | Pinecone | Qdrant | Weaviate | pgvector |
| --- | --- | --- | --- | --- |
| Hosting | Managed / Self / Cloud | Self / Cloud | Self / Cloud | Postgres extension |
| Setup effort | None | Low | Medium | Low (if you have PG) |
| Local dev | ✖ | ✅ | ✅ | ✅ |
| Best for | Fastest to prototype | Self-host + speed | Rich hybrid search | Already have Postgres |
| n8n support | ✅ Built-in node | ✅ | ✅ | via vector store nodes |

> Switching later is mostly a node swap — the workflow logic doesn't change.

---

## 🚀 Ideas to Extend

- ➕ **Add a Document Splitter** — the single biggest quality win, and the one [bug #3](#-known-bugs-in-this-workflow) still open
- 🔁 **Switch to real upsert** — the committed fix clears the namespace, which is idempotent but not incremental. Deterministic vector IDs would let you update one document without a full rebuild
- 🗂️ **Namespace per corpus** — `rag-docs` keeps this project separate from anything else in the index
- 📐 **Switch to `gemini-embedding-001`** for text-only work — but re-embed the whole corpus, the spaces are incompatible
- 🛡️ **Add a retrieval threshold** — Pinecone's `includeMetadata` plus a similarity cut-off stops the agent answering from a weak match
- 📎 **Ingest PDFs and Notion/Confluence pages** — swap the Drive node, keep the rest
- 🔀 **Hybrid search** — combine vector similarity with Pinecone's keyword search for better recall
- 📝 **Add citations** — have the agent cite which passages it used, so answers are auditable
- 🚦 **Add a relevance threshold** — if the top match scores below X, say "not in the document" instead of guessing
- 📊 **Log queries + top hits** to a sheet, then review which questions RAG failed to answer

---

## 📂 Project Structure

```text
NextLeap-RAG-Implementation-Pinecone-Vector-DB-Gemini-Embeddings-05-October-2026/
├── RAG implementation - Pinecone vector DB + Gemini embeddings.json
├── README.md
└── LICENSE
```

### The 14 nodes at a glance

| Count | Type |
| --- | --- |
| 1 | `scheduleTrigger` |
| 2 | `googleDrive` (search + download) |
| 2 | `vectorStorePinecone` (insert + retrieve-as-tool) |
| 2 | `embeddingsGoogleGemini` |
| 1 | `documentDefaultDataLoader` |
| 1 | `chatTrigger` |
| 1 | `agent` |
| 1 | `lmChatGoogleGemini` |
| 1 | `memoryBufferWindow` |
| 2 | `stickyNote` (label each pipeline) |

> ✅ This workflow ships **scrubbed** — no credential UUIDs, no real Drive file IDs, no instance fingerprint. See the [sharing guide](https://github.com/Gursimaran21/NextLeap-Building-N8N-Workflows-and-sharing-on-Github-04-October-2026) for the checklist.

---

## 🎓 Workshop Context

Part of a **NextLeap AI Engineer bootcamp** series.

| Segment | Focus |
| --- | --- |
| Sample Workflows — RAG | Vector stores, embeddings, retrieval-as-tool, Pinecone |

### Session map

| # | Repo | Introduced |
| --- | --- | --- |
| 1 | [Google Calendar AI Assistant](https://github.com/Gursimaran21/NextLeap-Google-Calendar-Assistant-04-October-2026) | Single agent + tools |
| 2 | [Build MCP Server and Client](https://github.com/Gursimaran21/NextLeap-Built-MCP-Server-and-Client-04-October-2026) | MCP — tools over a standard |
| 3 | [Multi-Agent System — Newsletter Agent](https://github.com/Gursimaran21/NextLeap-Multi-Agent-System-Newsletter-Aagent-04-October-2026) | Multi-agent orchestration |
| 4 | [Building & Sharing n8N Workflows](https://github.com/Gursimaran21/NextLeap-Building-N8N-Workflows-and-sharing-on-Github-04-October-2026) | Building & publishing workflows |
| 5 | **This repo** | RAG, embeddings, vector stores |

**Key takeaways:**

- **RAG = two pipelines sharing one vector store.** Build and verify ingestion *first*; a broken index looks exactly like a broken agent.
- **One embedding model, everywhere.** Mismatch fails silently, not loudly.
- **The vector store is just another tool.** Same `ai_tool` port as Gmail or Pinecone-adjacent tools — and the description still governs whether it gets called.

---

## 📄 License

Released under the [MIT License](LICENSE).

---

## 👤 Author

**Gursimaran** — [GitHub @Gursimaran21](https://github.com/Gursimaran21)

---

## 🔗 Resources

- [n8n RAG template](https://n8n.io/workflows) · [Vector Store docs](https://docs.n8n.io)
- [Pinecone docs](https://docs.pinecone.io) · [Pinecone console](https://app.pinecone.io)
- [Gemini embedding models](https://ai.google.dev/gemini-api/docs/models)
- [RAG fundamentals](https://en.wikipedia.org/wiki/Retrieval-augmented_generation)

---

<div align="center">

**Built with 🔍, 🗄️ and a lot of silent similarity-search failures.**

</div>
