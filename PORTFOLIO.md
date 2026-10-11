# Phylax-ops — Safety-first Retail Operations Assistant

**Links**
- GitHub: https://github.com/Nirrmitt/Phylax-ops
- Live UI (Render): https://phylax-ops-ui.onrender.com/
- Live API health: https://phylax-ops-api.onrender.com/health

## Summary
Phylax-ops is a portfolio project where I designed and built a **safety-first operations assistant** that answers store-policy questions, queries synthetic order data, and can prepare a support ticket **only after explicit confirmation**.

Instead of “chatbot answers,” the system is engineered to behave like an operations tool:
- **Evidence-backed output**: every response includes the plan, tool outputs, and sources.
- **Guarded data access**: SQL is constrained to read-only `SELECT` with limits/timeouts.
- **Action controls**: ticket creation is gated behind double-confirmation + an explicit opt-in flag.
- **Deterministic-by-default**: runs without an API key using a mock planner for reproducibility.

All data is synthetic and intended for demo/portfolio use.

## Why I built it
In retail ops, questions routinely span multiple domains:
- “What’s the return window for opened items?” (policy)
- “Where is order 10293 right now?” (database)
- “Escalate this to support.” (action)

A generic assistant can respond confidently without sources, misread policy, or trigger actions too early. I built Phylax-ops to demonstrate how to engineer an agent that is:
- bounded (finite tool calls, validated parameters)
- auditable (tool traces + citations)
- safe-by-construction (SQL guardrails + action gates)

## Key features (recruiter highlights)
### 1) Transparent agent planning + evidence
- Produces a structured plan for each request.
- Displays tool calls and tool outputs directly in the UI.
- Cites evidence for policy answers (document + chunk/source IDs).

### 2) Retrieval-augmented policy Q&A
- Stores policy chunks in ChromaDB.
- Uses SentenceTransformers embeddings for semantic retrieval.
- Supports “policy-first” workflows where answers must be grounded in written policy.

### 3) Guarded SQL querying (read-only, bounded)
- Accepts only a single `SELECT` statement.
- Rejects dangerous patterns (writes, multiple statements, comments).
- Enforces a row cap and short DB timeout.
- Uses PostgreSQL via Docker Compose (and a least-privilege approach for the API role).

### 4) Safe actions: ticket creation is opt-in
Phylax-ops can prepare a support ticket, but only when BOTH are true:
- The request includes explicit confirmation (e.g., `confirm=true`).
- A runtime allow-list flag is enabled (`ALLOW_TICKET_ACTIONS=true`).

This design prevents “accidental” actions and makes action-enablement a deliberate operational choice.

### 5) Quality & evaluation mindset
- Includes a 50-question evaluation set across RAG-only, SQL-only, multi-step, action, and out-of-scope/adversarial prompts.
- Local dev workflow includes linting, tests, seeding, and repeatable evaluation runs.

## Architecture (high level)
**Stack**
- Agent framework: LangGraph
- API: FastAPI
- UI: Streamlit
- Retrieval: ChromaDB + SentenceTransformers
- Data: SQLAlchemy + SQLite (local) / PostgreSQL (Docker)
- Testing & quality: pytest, Ruff, CI workflows

**Flow**
1. User question → planner proposes a bounded tool plan
2. Tools:
   - `rag_search` for policy retrieval
   - `sql_query` for read-only database questions
   - `create_ticket` for a gated action
3. Synthesizer builds an answer from tool evidence
4. Validator checks for weak evidence; either refines once or hands off

## What I’d improve next
- Add structured “runbooks” per policy area (returns, shipping, warranty) for faster retrieval and more consistent citations.
- Add stronger schema awareness for SQL queries (auto-suggest safe query templates).
- Add a minimal role model: manager vs. support agent vs. auditor views.

## Resume bullets (copy/paste)
- Built a safety-first retail ops assistant with LangGraph + FastAPI + Streamlit that produces evidence-backed answers (plan + tool outputs + citations).
- Implemented guarded SQL access (single-statement read-only `SELECT`, validation, row caps, timeouts) for safe database querying.
- Designed action controls for ticket creation using explicit confirmation + runtime opt-in flags to prevent unintended side effects.
- Created an evaluation suite with labeled question categories (RAG-only, SQL-only, multi-step, action, adversarial) and a repeatable dev workflow (lint/test/eval).
