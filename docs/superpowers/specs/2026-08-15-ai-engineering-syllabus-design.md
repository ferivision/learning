# Design: AI Engineering / Agent Syllabus (`roadmap-ai/`)

## Goal

Populate `roadmap-ai/` with a 19-phase, 71-topic AI Engineering / Agent syllabus (plus Phase 19's 6 unnumbered practical projects). `roadmap-ai/README.md` is the roadmap index — copied verbatim from the master prompt the user supplied, not regenerated. `roadmap-ai/syllabus/phase-01-*.md` through `phase-19-*.md` are the detailed lesson files, one per phase, each explaining every topic listed under that phase in the README.

This is a content-generation task, not a code feature — the "design" here is about structure, consistency, and how the large volume of content gets produced without truncation, mirroring `docs/superpowers/specs/2026-08-14-backend-syllabus-design.md` (the backend syllabus, already complete in this repo).

## Scope

- Create `roadmap-ai/README.md` — exact copy of the content the user supplied. No edits.
- Create `roadmap-ai/syllabus/phase-01-llm-fundamentals.md` … `phase-19-practical-projects.md` (19 files).
- Topic list per phase is fixed by the user's master prompt (71 numbered topics, phase 1–18, plus Phase 19's 6 unnumbered projects). No topics added or removed.

## Per-topic content structure (mandatory, every numbered topic in phases 1–18)

```markdown
## <Number>. <Topic Name>

### Apa itu?
### Kenapa dibutuhkan?
### Cara Kerja
### Contoh Kode — Python
### Trade-off & Pitfall
### Kapan Dipakai
### Sering Ditanya Saat Interview
```

Rules:
- **Language**: casual Bahasa Indonesia mixed with untranslated technical terms (embedding, context window, tool calling, dll). Do not force-translate jargon.
- **Code language**: Python by default. Every Python code block must be beginner-friendly per the user's explicit rules: narrate what the code accomplishes *before* the block, comment non-trivial lines (not every obvious line), add a 1–2 sentence "catatan Python" the first time an intermediate Python concept appears anywhere in the syllabus (decorator, async/await, generator, dataclass, etc.), and code must be real/runnable — never pseudocode.
- **Topics 23 (LangChain) and 27 (LangGraph) only**: add both Python and Node.js code, PLUS a "Cara Manual (From Scratch)" subsection (in both languages) showing the same pattern built with the official SDK directly, no framework. The user's master prompt already contains the exact reference code for both topics (Python + Node.js, framework version + manual version) — adapt it into SupportPilot's context rather than regenerating from scratch.
- **Conceptual-only topics** (skip both code sections, keep the other 5 subsections): topic 2 (Transformer Basics), topic 3 (Bagaimana LLM Diciptakan / Training Pipeline), topic 5 (Model Selection), topic 24 (What is an AI Agent?), topic 26 (Agent vs Workflow), topic 33 (AI Memory Landscape survey), topic 38 (Single Agent — recap of Phase 6, points back rather than re-implementing), topic 41 (Multi-Agent Tradeoffs), topic 56 (Agent Security Mental Model). Every other topic gets a real, runnable Python code example.
- **Phases 12 (Hermes Agent) and 13 (OpenClaw)**: keep the full 7-section structure (code sections included), but "Contoh Kode — Python" here holds **illustrative** structure — example skill/config file layout, conceptual directory structure, pseudo-manifest — never invented CLI commands or API calls presented as if verified. Explicitly note in-prose that exact commands should be checked against official docs.
- Every phase file opens with:
  ```markdown
  # Phase XX — <Phase Name>

  > Bagian dari [AI Engineering / Agent Roadmap](../README.md)

  ---
  ```
- Every phase file (except phase-19) closes with:
  ```markdown
  ---

  **Selanjutnya:** [Phase XX+1 — <Next Phase Name>](./phase-XX+1-xxx.md)
  ```
- **Phase 19 (Practical Projects)** does not use the per-topic structure. Each of the 6 projects gets: a short intro, an ASCII architecture diagram, and an implementation checklist — referencing/extending components already built in earlier phases (RAG pipeline from Phase 4, agent loop from Phase 6, memory from Phase 7, etc.) rather than introducing new code from scratch. No "Selanjutnya" footer (last phase).

## Shared example project: SupportPilot

All Python (and the two Node.js) code samples across all 19 phases live in one fictional, continuously-evolving codebase — an AI-powered customer support assistant — rather than disconnected snippets. This mirrors OrderFlow's role in the backend syllabus and lets later phases extend earlier phases' functions (e.g. Phase 17's Agentic RAG wraps the same `retrieve_relevant_chunks` introduced in Phase 4).

**Domain / entities**: `Customer` (id, name, email, tier), `Order` (id, customer_id, product, status, amount), `Ticket` (id, customer_id, subject, status, priority), `KnowledgeArticle` (id, title, content, embedding), `Conversation` (id, customer_id, messages, memory_summary).

Chosen because it naturally exercises nearly every phase: tool calling against order/ticket lookups (Phase 2, 6), a real knowledge base for RAG (Phase 3, 4), an agent loop that answers support questions and escalates when needed (Phase 6), persistent per-customer memory (Phase 7), a support-specific skill (Phase 8), multi-agent escalation to a specialist (Phase 10), and it maps directly onto Phase 19's Project 4 (Customer Support Agent) as the capstone.

**Stack**: `openai`/`anthropic` Python SDKs, `fastapi` (backend examples), `psycopg2`/`pgvector` (vector DB), `redis` (caching/memory), `sentence-transformers` (local embedding alternative), `langchain`/`langgraph` (topics 23/27 only), `pydantic` (structured output validation). Node.js only for topics 23/27: `openai` SDK, `@langchain/core`, `@langchain/langgraph`.

## Cross-phase continuity mechanism

Same mechanism as the backend syllabus: each phase is generated by its own subagent, so no single agent sees every prior phase's full file. Each phase's dispatch prompt includes a short **carry-forward API summary** — just the function signatures introduced in earlier phases that this phase's topics are likely to touch — and after each phase finishes, the orchestrator appends any newly-introduced reusable function signatures to a running carry-forward list for the next phase's prompt. This list is working state for execution, not part of the committed docs.

## Execution approach

- One subagent per phase file, dispatched **sequentially** (phase-01 must finish and be saved before phase-02 starts) — avoids output truncation on large files and preserves the carry-forward chain.
- Each subagent writes its file directly to disk via the Write tool.
- Each subagent's prompt is self-contained: phase number/name, its exact topic list, the mandatory section structure (with per-topic code/no-code/illustrative-code calls spelled out), the SupportPilot domain + stack, the carry-forward summary so far, and the header/footer template.
- Same review rigor as the backend syllabus: after each phase, run a structural verification check (topic/subsection/code-block counts, byline, footer, no placeholders), a code-correctness check (Python syntax via `py_compile` or `python -m py_compile`, Node.js via `node --check` for topics 23/27), then a task-level spec+quality review with a fix loop (max 5 rounds) before moving to the next phase.
- Push to `origin/main` after each phase's commits land (continuing this session's established cadence for the backend syllabus).
- After all 19 phases, run one final whole-branch review (most capable model) over the full diff, triaging any deferred minor findings, with one fix wave + one scoped re-review if it finds anything.
- Continue directly on `main` (no worktree), consistent with how the backend syllabus was executed in this same session.

## Out of scope

- No changes to the root `README.md`, `roadmap-backend/`, or repo structure outside `roadmap-ai/`.
- No CI, tooling, or test automation — this is documentation content only.
- No topics added beyond the fixed 71-topic + Phase 19 list.
- No verified/tested Hermes Agent or OpenClaw CLI commands — architecture explanation only, per the user's explicit instruction.

## Topic Reference (fixed — do not add/remove)

**Phase 1 — LLM Fundamentals**: 1. LLM Basics, 2. Transformer Basics *(conceptual)*, 3. Bagaimana LLM Diciptakan / Training Pipeline *(conceptual)*, 4. Tokens & Context Window, 5. Model Selection *(conceptual)*.

**Phase 2 — Prompting & Structured Output**: 6. Prompt Engineering, 7. Structured Output, 8. Function / Tool Calling.

**Phase 3 — Embeddings**: 9. Embeddings, 10. Vector Similarity, 11. Vector Database (pgvector focus).

**Phase 4 — RAG**: 12. What is RAG?, 13. RAG Ingestion Pipeline, 14. Chunking, 15. Retrieval, 16. Reranking, 17. RAG Failure Modes, 18. RAG Evaluation.

**Phase 5 — LLM Application Architecture**: 19. Basic LLM Backend, 20. LLM Gateway / Provider Abstraction, 21. Streaming, 22. AI Cost Management, 23. LangChain (Python + Node.js + manual-from-scratch both languages).

**Phase 6 — Agents**: 24. What is an AI Agent? *(conceptual)*, 25. Agent Loop, 26. Agent vs Workflow *(conceptual)*, 27. LangGraph (Python + Node.js + manual-from-scratch both languages), 28. Tools.

**Phase 7 — Agent Memory**: 29. Short-Term Memory, 30. Long-Term Memory, 31. Memory Retrieval, 32. Context Engineering & Context Compaction, 33. AI Memory Landscape — Tools & Produk Nyata *(conceptual)*.

**Phase 8 — Agent Skills**: 34. What is a Skill?, 35. Tool vs Skill.

**Phase 9 — MCP**: 36. MCP (Model Context Protocol), 37. Why MCP?.

**Phase 10 — Agent Orchestration**: 38. Single Agent *(conceptual, recap)*, 39. Multi-Agent, 40. Agent Delegation, 41. Multi-Agent Tradeoffs *(conceptual)*.

**Phase 11 — Agent Runtimes / Harnesses**: 42. Agent Runtime, 43. Sandboxing, 44. Human-in-the-Loop, 45. Agent Permissions.

**Phase 12 — Hermes Agent** *(illustrative-code phase, see rules above)*: 46. What to Understand About Hermes, 47. Hermes Skills, 48. Hermes Memory, 49. Hermes Subagents.

**Phase 13 — OpenClaw** *(illustrative-code phase, see rules above)*: 50. OpenClaw Concepts, 51. Agent Channels.

**Phase 14 — Agent Security**: 52. Prompt Injection, 53. Indirect Prompt Injection, 54. Tool Permission, 55. Data Exfiltration, 56. Agent Security Mental Model *(conceptual)*, 57. Skill/Tool Supply Chain Security, 58. Guardrails & Output Filtering.

**Phase 15 — AI Observability & Evaluation**: 59. LLM Observability, 60. LLM Evaluation, 61. RAG Evaluation, 62. Agent Evaluation.

**Phase 16 — Model Fine-Tuning**: 63. Fine-Tuning, 64. LoRA / PEFT.

**Phase 17 — Advanced AI Architecture**: 65. Agentic RAG, 66. Multi-Step Research Agent, 67. Coding Agent.

**Phase 18 — AI Infrastructure**: 68. Model Gateway, 69. Model Routing, 70. AI Caching, 71. Batch Processing.

**Phase 19 — Practical Projects** *(no per-topic structure — architecture diagram + checklist per project)*: 1. Basic LLM API, 2. RAG System, 3. Tool-Calling Agent, 4. Customer Support Agent, 5. Personal Agent, 6. Multi-Agent System.
