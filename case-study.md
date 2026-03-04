# Case Study: Building a Human-in-the-Loop AI Support Platform

> A solo engineering project — from concept to deployed MVP.

---

## The Problem

Modern AI chatbots for customer support face a fundamental tension: full autonomy risks hallucinations and brand damage, while full human operation is slow and unscalable. Most teams end up at one extreme or the other — either an AI that occasionally embarrasses the company or a support queue drowning in volume.

The goal of this project was to design a middle path: **AI that accelerates human agents rather than replacing them**. The customer always gets a fast, high-quality, human-approved response. The agent never stares at a blank compose box again.

---

## Phase 1 — Scope Definition

Before writing a line of code, I spent time defining the boundaries of the MVP. Three principles guided the scope:

1. **Deployable on free-tier infrastructure** — the system had to work within Railway's free tier (0.5 GB RAM, 1 vCPU, no GPU). This ruled out local model hosting and shaped the entire backend architecture.

2. **Knowledge-base-first** — the AI would be grounded in company-specific documents, not general world knowledge. RAG was the only viable path given the constraint against fine-tuning (which requires labelled data that doesn't exist yet at MVP stage).

3. **Feedback-loop-ready from day one** — even if the data isn't used immediately, every agent decision should be captured in a form that could power fine-tuning or RLHF in a v2.

---

## Phase 2 — Architecture Decisions

### The HITL Decision

The first prototype sent LLM responses directly to customers. After observing output quality across varied inputs — unusual phrasing, multi-intent queries, edge-case topics — the failure rate was unacceptable for a customer-facing product. The hallucination rate wasn't high, but the consequences of even one bad response in a customer support context are disproportionate.

The HITL layer added one step (agent approval) for a large reliability gain. The latency cost is real but acceptable for a support context where customers already expect some wait time.

### Serverless Inference

The HuggingFace Inference API was chosen over self-hosting for two reasons: it fit within the memory budget with zero overhead, and it allowed the model to be swapped without any infrastructure change. The same architecture works with any HF-hosted model — upgrading from a 1B to a 7B parameter model is a one-line config change.

The tradeoff is cold-start latency on the HF free tier. This was mitigated by showing a "generating draft..." state in the agent UI, setting expectations appropriately.

### RAG Chunk Tuning

Initial retrieval quality was poor. The first implementation used large chunks (1000+ tokens) which retrieved too broadly. After testing with chunks of 256–512 tokens and 10% overlap, retrieval precision improved significantly. The key insight: smaller chunks improve recall precision because the embedding captures a tighter semantic unit, but too-small chunks lose context. The sweet spot for this use case was domain-specific paragraphs, not sentences or pages.

---

## Phase 3 — Building the Agent Experience

The agent dashboard went through three design iterations before settling on its current form.

**Iteration 1:** Single-column list of conversations. Agents had to click into each conversation to see the AI draft. Context-switching was too frequent.

**Iteration 2:** Split-panel view — conversation list on the left, draft and chat on the right. Better, but the draft competed visually with the message history.

**Iteration 3 (final):** The draft is presented as the primary action above the reply composer. The message history is visible but secondary. Approve, edit, and reject are one-click actions. The agent never has to wonder what to do next.

This iterative process reinforced a lesson: **the UX of the human-in-the-loop interface is as important as the AI quality**. An excellent draft presented in a confusing UI is worse than a mediocre draft presented clearly.

---

## Phase 4 — Deployment Challenges

Deploying to Railway's free tier required deliberate resource management:

- **Docker image size:** The initial image was ~400 MB due to Python ML dependencies. After switching from local model loading to API-based inference, the base image dropped to ~50 MB.
- **Memory usage:** ChromaDB's in-process vector store peaks during large document ingestion. Added chunked ingestion with progress tracking to avoid OOM events.
- **Cold starts:** Railway services spin down after inactivity. A health-check endpoint and client-side retry logic handle the reconnection case gracefully.

---

## Results & Reflection

The deployed MVP achieves its core goal: agents can process queries faster than without AI assistance, and every response that reaches a customer has been reviewed by a human. The feedback dataset is growing with each approved or rejected draft.

### What Worked Well
- RAG over fine-tuning: admins update the knowledge base in real time without engineering involvement.
- The async-first backend handles concurrent WebSocket connections without blocking.

### What I Would Do Differently
- **Start with streaming responses earlier.** Token-by-token streaming in the draft panel would make the AI feel much faster even when total latency is the same.
- **Invest more in observability from the start.** Adding structured logging mid-project is harder than designing for it upfront. A future version would ship with tracing and LLM eval metrics from day one.
- **Separate the vector store earlier.** ChromaDB running in-process is convenient but creates coupling between the API server's lifecycle and the vector index. A persistent external vector store would be cleaner at scale.

