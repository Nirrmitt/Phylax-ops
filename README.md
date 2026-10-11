# Phylax-ops

<p align="center">
  <strong>A safety-first AI operations copilot for retail support.</strong><br>
  Ground answers in policy and order data, expose the evidence, and keep actions behind explicit controls.
</p>

<p align="center">
  <a href="https://github.com/Nirrmitt/Phylax-ops/actions/workflows/ci.yml"><img src="https://github.com/Nirrmitt/Phylax-ops/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="https://github.com/Nirrmitt/Phylax-ops"><img src="https://img.shields.io/badge/Source-GitHub-24292F?style=flat&logo=github&logoColor=white" alt="Source on GitHub"></a>
  <a href="https://nirrmitt.github.io/NRT-Terminal/demos/phylax.html"><img src="https://img.shields.io/badge/Live_Demo-In_browser-38bdf8?style=flat" alt="Live demo"></a>
  <img src="https://img.shields.io/badge/Python-3.11-3776AB?style=flat&logo=python&logoColor=white" alt="Python 3.11">
  <img src="https://img.shields.io/badge/License-MIT-2ea44f?style=flat" alt="MIT license">
</p>

**Phylax-ops** is a project that brings three common retail-support workflows into one auditable assistant: policy lookup, order-data queries, and a confirmation-gated support-ticket action. Rather than present an opaque chatbot answer, it exposes the proposed plan, tool outputs, and sources, and hands off when the available evidence is insufficient.

**Project:** [Source code](https://github.com/Nirrmitt/Phylax-ops) · [Live in-browser demo](https://nirrmitt.github.io/NRT-Terminal/demos/phylax.html)

> **Demo scope:** all policies and order records are synthetic. Ticket creation uses a mock endpoint. This project demonstrates safety-oriented engineering patterns; it is not a production support system.

## Contents

- [What it demonstrates](#what-it-demonstrates)
- [Architecture](#how-a-request-moves-through-the-agent)
- [Demo](#streamlit-demo)
- [Run locally](#try-it-locally)
- [API](#example-api-request)
- [Evaluation](#evaluation)
- [Safety and limitations](#safety-and-engineering-choices)
- [Configuration](#configuration)
- [Project layout](#project-layout)

## What it demonstrates

- **Evidence-first agent UX:** each turn makes its plan, tool results, citations, confidence, and handoff status inspectable.
- **Retrieval-augmented policy answers:** ChromaDB and SentenceTransformers retrieve relevant passages from seven policy documents, with document and chunk identifiers.
- **Constrained database access:** SQL is validated as a single read-only `SELECT`, capped at 100 rows, and executed with a short timeout. Docker Compose configures PostgreSQL with a read-only application role.
- **Explicitly gated actions:** ticket requests require request-level confirmation and a separate runtime opt-in; the action is disabled by default.
- **Bounded orchestration:** LangGraph coordinates validated tool calls, caps a turn at six tool steps, retries weak evidence once, and surfaces tool failures for review.
- **Evaluation as part of the build:** a 50-question labeled dataset covers retrieval, SQL, multi-step requests, actions, and adversarial/out-of-scope prompts, with 12 holdout cases.
- **Reproducible engineering workflow:** FastAPI, Streamlit, pytest, Ruff, Docker Compose, and GitHub Actions support local development and CI.

## At a glance

| | |
|---|---|
| **Agent framework** | LangGraph |
| **API and UI** | FastAPI, Streamlit |
| **Retrieval** | ChromaDB + SentenceTransformers (`all-MiniLM-L6-v2`) |
| **Data** | SQLAlchemy, SQLite locally, PostgreSQL in Docker Compose |
| **Model integration** | Deterministic mock by default; OpenAI-compatible API for real use |
| **Quality checks** | pytest, Ruff, 50-question evaluation set, GitHub Actions |

## The problem

Store operations questions often cross boundaries. A manager may need a return rule from a policy document, an order status from a database, or a handoff to a human. A general-purpose chatbot can answer without evidence or take an action too early.

`Phylax-ops` demonstrates a constrained alternative. It chooses from three tools, validates their inputs, keeps SQL read-only and row-limited, requires two gates before ticket creation, cites the evidence it used, and hands off when it cannot support an answer.

## How a request moves through the agent

```mermaid
flowchart LR
  Q[Question] --> P[Planner]
  P -->|out of scope| R[Polite refusal]
  P -->|bounded plan| T{Up to 6 tool calls}
  T --> D[Policy search: ChromaDB]
  T --> S[SQL: SELECT guard + row cap]
  T --> A[Ticket: confirmation + opt-in]
  D --> Y[Synthesizer]
  S --> Y
  A --> Y
  Y --> V[Evidence and tool check]
  V -->|supported| O[Answer with sources]
  V -->|weak evidence| X[One refined retry]
  V -->|tool failure or still weak| H[Human handoff]
  X --> P
```

The synthesizer formats tool outputs instead of generating unsupported factual claims. Policy results include the source document and chunk ID; SQL results include the executed, guarded SQL. See [the architecture notes](docs/architecture.md) for more detail.

### Streamlit demo

These local mock-mode screenshots show the planned tool call, returned tool evidence, and cited sources. The [in-browser demo](https://nirrmitt.github.io/NRT-Terminal/demos/phylax.html) is a JavaScript port of the mock-mode logic running on synthetic data.

![Streamlit plan and tool call](docs/streamlit-demo-plan.png)

![Streamlit evidence and sources](docs/streamlit-demo-evidence.png)

### The three tools

1. **`rag_search`** retrieves passages from seven synthetic store policies covering returns, shipping, warranty, escalation, order changes, price matching, and privacy.
2. **`sql_query`** accepts one `SELECT`, rejects write operations, comments, and multiple statements, adds a maximum 100-row limit, and uses a short database timeout. Docker Compose connects the API as a PostgreSQL read-only role.
3. **`create_ticket`** is a local webhook-style mock. It requires both request-level `confirm=true` and `ALLOW_TICKET_ACTIONS=true`; it is disabled by default.

## Try it locally

Python 3.11 is required. The commands below use the project virtual environment directly, so no activation step is needed.

**PowerShell (Windows)**

```powershell
python -m venv .venv
.venv\Scripts\python.exe -m pip install -e ".[dev]"
.venv\Scripts\python.exe -m app.seed
.venv\Scripts\uvicorn.exe app.api:app --reload
```

**macOS or Linux**

```sh
python3 -m venv .venv
.venv/bin/python -m pip install -e ".[dev]"
.venv/bin/python -m app.seed
.venv/bin/uvicorn app.api:app --reload
```

The mock planner and SQLite database work without an API key. The first policy search downloads the SentenceTransformers model if it is not cached. Open [http://localhost:8000/docs](http://localhost:8000/docs) to call the API. To launch the visual interface in another terminal:

```sh
# Windows: .venv\Scripts\streamlit.exe run ui/streamlit_app.py
# macOS/Linux:
.venv/bin/streamlit run ui/streamlit_app.py
```

The Streamlit UI shows the plan, each tool result, citations, confidence, and any human handoff. To launch the API, PostgreSQL, and UI together with Docker, run `docker compose up --build` (or `make docker-up`). The UI is served on port 8501 and the API on port 8000.

## Example API request

```sh
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is the return window?","confirm":false}'
```

`POST /ask` returns the answer, structured plan, tool results, sources, confidence, and `handed_off`. `GET /health` reports service health. Invalid requests receive `422`; assistant or data-tool failures receive `503`.

The response includes the executed plan and auditable tool results alongside the user-facing answer:

```json
{
  "answer": "…",
  "plan": {"calls": [{"tool": "rag_search", "arguments": {"query": "…"}}], "refusal": null},
  "tool_calls": [{"tool": "rag_search", "arguments": {"query": "…"}, "output": ["…"], "ok": true}],
  "sources": ["returns.md#chunk-…"],
  "confidence": 0.75,
  "handed_off": false
}
```

The abbreviated values above illustrate the response shape; tool output and source identifiers depend on the request.

## Evaluation

The JSONL dataset has 50 labeled questions: 15 RAG-only, 15 SQL-only, 10 multi-step, 5 action, and 5 out-of-scope or adversarial examples. Twelve are marked `holdout`. Use the development split to make prompt changes; run the complete set after choices are fixed.

Run the checks and evaluation from the repository root. Use the virtual-environment executables so this works on a clean machine without global installs.

```powershell
.venv\Scripts\ruff.exe check .
.venv\Scripts\pytest.exe -q
.venv\Scripts\python.exe -m app.seed
.venv\Scripts\python.exe -m eval.run_eval
```

```sh
.venv/bin/ruff check .
.venv/bin/pytest -q
.venv/bin/python -m app.seed
.venv/bin/python -m eval.run_eval
```

The harness reports tool-routing accuracy, SQL result-set accuracy against reference SQL, RAG Recall@4 against labeled source files, faithfulness, abstention/handoff, end-to-end success, graph steps, latency percentiles, and cost per query. It saves JSON and Markdown under [`eval/results/`](eval/results/). In mock mode, faithfulness uses a deterministic lexical proxy and cost is $0. In real-provider mode, faithfulness uses an LLM judge; set the optional model token-price variables to estimate cost.
The runner evaluates all development cases before the 12 holdout cases. Its JSON output includes per-case outcomes and failure categories, but not question text.

### Recorded mock run

These figures come from the run saved in `eval/results/mock.json` on 2026-10-09. They describe the deterministic mock planner and local synthetic dataset, not a live model or production workload. The mock planner uses fixed intent rules and example SQL; its results measure the application path and evaluation harness, not general language-model reasoning.

| Metric | Mock result |
|---|---:|
| Tool-routing accuracy | 0.98 |
| SQL execution accuracy (25 labeled cases) | 1.00 |
| RAG Recall@4 (25 labeled cases) | 0.96 |
| Faithfulness (lexical proxy) | 0.7888 |
| Correct abstention / handoff | 0.98 |
| End-to-end success | 0.96 |
| Average graph steps | 3.90 |
| p50 / p95 latency | 19.99 ms / 23.62 ms |
| External model cost per query | $0.00 (mock; excludes local compute) |

| Baseline | Tool-routing accuracy | Other result |
|---|---:|---:|
| Mock plain LLM, no tools | 0.10 | — |
| Mock RAG-only | 0.30 | Recall@4: 0.96 |
| Mock full agent | 0.98 | End-to-end: 0.96 |

The baselines are intentionally lightweight deterministic comparisons, not separate model calls. Treat these scores as a reproducible engineering smoke evaluation, not a statistically rigorous benchmark.

### Recorded real OpenRouter run

These are the results from `eval/results/real.json`, recorded on 2026-10-09 with `openai/gpt-4o-mini` through OpenRouter. The 38 development cases ran first and the 12 holdouts ran last; no prompt or routing changes were made after seeing the holdout results. This run exposed significant model-planning and SQL-schema gaps; it is reported to show the measured behavior, not as a production-quality score.

| Metric | Real result |
|---|---:|
| Tool-routing accuracy | 0.38 |
| SQL execution accuracy (25 labeled cases) | 0.00 |
| RAG Recall@4 (25 labeled cases) | 0.96 |
| Faithfulness (LLM judge) | 0.488 |
| Correct abstention / handoff | 0.38 |
| End-to-end success | 0.10 |
| Average graph steps | 3.46 |
| p50 / p95 latency | 1,256.75 ms / 2,353.87 ms |
| Tokens (prompt / completion) | 52,054 / 1,554 |
| Cost per query | Not reported (token prices were not configured) |

| Split | Queries | Tool routing | Correct handoff | End-to-end success |
|---|---:|---:|---:|---:|
| Development | 38 | 0.3684 | 0.3684 | 0.1053 |
| Holdout (final) | 12 | 0.4167 | 0.4167 | 0.0833 |

The live-model run performed substantially worse than the deterministic mock. The [failure log](eval/FAILURES.md) details routing refusals, SQL schema mismatches, and the one labeled source missed by retrieval. RAG recall is measured separately from whether the planner actually invoked the retrieval tool. Faithfulness is an LLM-judge score, not a human-validated factuality rate. The mock lexical faithfulness proxy and live-model judge score use different methods and should not be compared as if they were the same metric.

## Safety and engineering choices

- **Offline by default:** a deterministic mock planner keeps the main flow testable without credentials. Real planning uses an OpenAI-compatible chat API; OpenRouter is the example default.
- **Read-only data path:** the guard enforces a single bounded `SELECT`; PostgreSQL runs through a role granted `SELECT` only. SQLite is used for quick local setup.
- **Confirmed actions:** ticket creation needs both user confirmation and an explicit environment opt-in. A missing confirmation produces a request to confirm without firing the action.
- **Bounded work:** tool inputs use Pydantic models, tool calls have timeouts, and a turn can execute at most six tool calls. Weak evidence gets one refined retry; tool failures or a second weak result hand off.
- **Visible evidence:** each policy result has a document and chunk identifier; SQL results expose the exact guarded query. Retrieved text is treated as data and never as agent instructions.
- **Synthetic data:** the policies and 250 generated orders are invented for this demo. No company or customer records are included.

The verifier checks tool success and evidence presence; it is not a formal proof of factual correctness. The mock planner maps common retail phrases to fixed example SQL rather than doing general-purpose text-to-SQL. The ticket service is a mock endpoint. These are deliberate portfolio-sized trade-offs, not production-ready controls.

## Known limitations and next steps

- The recorded OpenRouter run achieved **0/25 SQL result-set accuracy** and **10% end-to-end success**. The detailed failure log shows that the planner needs stronger schema grounding and more reliable tool selection before real-model use.
- Policy retrieval achieved **24/25 Recall@4** in that run, but retrieval recall alone does not prove that the answer used the right passage or that its claims are correct.
- Verification confirms tool success and evidence presence; it does not establish semantic entailment or replace human review.
- The mock planner is intentionally deterministic and narrow. The support-ticket integration is a local mock, not a connection to a live ticketing platform.
- The public demo uses free Render services, SQLite, and synthetic data. It can sleep during inactivity, has ephemeral local storage, and is not suitable for sensitive or production workloads.

Potential improvements include injecting the actual database schema into model planning, expanding the holdout set, measuring citation support with human-reviewed examples, and integrating an approved ticket system behind the existing action gates.

## Configuration

Copy `.env.example` to `.env` if you want to change defaults. `.env` is ignored by Git; never put credentials in source control.

| Variable | Purpose | Default |
|---|---|---|
| `LLM_PROVIDER` | `mock` or a real OpenAI-compatible provider | `mock` |
| `OPENAI_API_KEY` | Provider credential | placeholder only |
| `OPENAI_BASE_URL` | OpenAI-compatible endpoint | OpenRouter URL |
| `LLM_MODEL` | Provider model name | `openai/gpt-4o-mini` |
| `DATABASE_URL` | SQLAlchemy connection | `sqlite:///data/ops.db` |
| `CHROMA_PATH` | Persistent Chroma directory | `.chroma` |
| `TICKET_WEBHOOK_URL` | Ticket endpoint | local mock endpoint |
| `ALLOW_TICKET_ACTIONS` | Enable the action after confirmation | `false` |
| `MODEL_INPUT_COST_PER_1M` | Input price in USD for evaluation cost estimate | unset |
| `MODEL_OUTPUT_COST_PER_1M` | Output price in USD for evaluation cost estimate | unset |

## Project layout

```text
app/                  LangGraph, schemas, planner, guards, tools, FastAPI
ui/                   Streamlit application
data/policies/        Seven synthetic operating policies
data/seed.sql         PostgreSQL schema, seed data, read-only role
eval/                 50 labeled questions, judges, runner, results, failure log
tests/                SQL guard, tool, graph, and API tests
docs/                 Architecture explanation and Mermaid diagram
.github/workflows/    CI and mock evaluation artifact workflow
.vscode/              Editor settings and launch configurations
Dockerfile            Container image
docker-compose.yml    PostgreSQL, API, and UI services
Makefile              Common development commands
```

## Run with real model credentials

1. Copy `.env.example` to `.env` and set `LLM_PROVIDER` to a non-`mock` value.
2. Add `OPENAI_API_KEY`, `OPENAI_BASE_URL`, and `LLM_MODEL` for your OpenAI-compatible provider.
3. Run `python -m eval.run_eval` and inspect the generated result files.
4. Update the README results only with output from that real run, and add any observed failures to `eval/FAILURES.md`.

## Render demo deployment

The root `render.yaml` defines separate free Docker web services for the API and Streamlit UI. The API seeds its synthetic SQLite database at startup, and the policy embedding model and index are prepared in the image. The UI connects to the API using Render's service-host reference. The deployment uses the mock LLM and keeps ticket actions disabled; no provider key is required.

**Live demo:** [Try the in-browser demo](https://nirrmitt.github.io/NRT-Terminal/demos/phylax.html), a JavaScript port of the mock-mode router, SQL guard, and confirm-gated ticket flow running on synthetic data. The Render deployment above is defined in `render.yaml` but is not currently running.

Both services use Render's **Free** plan. Free services spin down after inactivity, so a request may take 50 seconds or more while the instance wakes; local filesystems are ephemeral, and the API is public because Free services do not support private networking. This setup is for a portfolio demo with synthetic data, not production use; SQLite contents are reseeded after restarts.

## Development

Install the development dependencies with `python -m pip install -e ".[dev]"`, then run the checks from the repository root:

```sh
ruff check .
pytest -q
python -m app.seed
python -m eval.run_eval
```

Continuous integration runs Ruff, seeds the local database, and runs pytest on Python 3.11. The evaluation workflow runs the mock evaluation and saves its results as a workflow artifact.

When testing a real provider, use your own credential in an ignored `.env` file, leave ticket actions disabled unless explicitly testing the gate, and review both development and holdout results. The evaluation files under [`eval/results/`](eval/results/) are recorded run artifacts, not guarantees about future models or deployments.

## 🤝 Connect & Contribute
I’m always open to feedback, collaboration, or chat about analytics engineering, automation, or data storytelling.

📧 Email: nirrmit.rtickoo@gmail.com

🌐 Portfolio: [NRT](https://nirrmitt.github.io/NRT-Terminal)

💼 LinkedIn: [Nirrmit R. Tickoo](https://www.linkedin.com/in/n-r-t/)

🐙 GitHub: [ @nirrmitt](https://github.com/Nirrmitt)

🔧 Found a bug or have an idea? Open an issue or submit a PR. I review all contributions!

### 📜 License
MIT ©[Nirrmitt](https://nirrmitt.github.io/NRT-Terminal) Feel free to use, adapt, and build upon this for your own projects or learning journey.
