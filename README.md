# Vidya OS

Vidya OS is an AI data-analysis and research agent. A user uploads a dataset (CSV/Excel) or asks an open-ended research question, and the system autonomously plans an approach, writes and executes Python code in an isolated sandbox, catches and fixes its own errors, and returns a written report with charts.

## Features

- **Autonomous data analysis** — upload a CSV/Excel file, ask a question in natural language, and the system plans, writes, executes, and self-corrects pandas/numpy code to answer it.
- **RAG knowledge bases** — upload documents (PDF/DOCX/TXT/MD/JSON/CSV) and query them semantically.
- **Web research** — live search and scraping with source-trust rating and citation discipline, for questions that need current or independently verifiable facts.
- **Subagents** — user-created specialized personas (e.g. "Finance Analyst") with their own persistent memory, optionally connected to the user's own AWS S3 bucket via a one-click provisioning flow.
- **Multi-tenant billing** — free and paid subscription tiers with usage-based overage.

## Tech stack

| Layer | Stack |
|---|---|
| Backend | Python, FastAPI, LangGraph |
| Frontend | Next.js 14 (App Router), React 18, Tailwind CSS |
| Auth | Clerk |
| Billing | Stripe (subscriptions + metered overage) |
| Vector memory | Pinecone |
| 3D UI layer | Three.js / React Three Fiber |
| Web scraping | Playwright, Tavily / DuckDuckGo search |
| Backend hosting | Docker container on Hugging Face Spaces |
| Frontend hosting | Vercel |

## Repository layout

```
navi/
├── backend/            FastAPI + LangGraph Python backend
│   ├── graph.py          The LangGraph state machine — the core orchestration logic
│   ├── index.py          FastAPI app: chat endpoints, dataset upload, CORS, admin routes
│   ├── graph_runner.py   Subprocess entrypoint (each request runs the graph in an isolated process)
│   ├── state.py          Shared state schema for the graph
│   ├── database.py       SQLite: skill cache + ethics-denial log
│   ├── memory_db.py      SQLite: Pinecone vector-id tracking, for deletion support
│   ├── chat_db.py / chat_routes.py           Chat history persistence + REST routes
│   ├── subagent_db.py / subagent_routes.py   Subagent CRUD, chat, AWS connection state
│   ├── aws_routes.py / s3_reader.py          One-click S3 provisioning for subagents
│   ├── billing_db.py / billing_config.py / billing_stripe.py / billing_routes.py   Stripe billing
│   ├── rag_ingest.py / rag_session_cache.py  RAG ingestion and in-memory document cache
│   ├── skill_validator.py    Static (AST/regex) security screen for LLM-generated code
│   ├── skill_sandbox_config.py / skill_runner.py   Execution sandbox configuration
│   ├── eval_engine.py / eval_db.py   Golden-dataset hallucination evaluation harness
│   ├── clerk_auth.py     Clerk JWT verification
│   ├── encryption.py     Field-level encryption for sensitive data at rest
│   ├── df_store.py        In-memory DataFrame store (subprocess handoff)
│   ├── media_store.py     Temporary media file store for subagent S3 loads
│   └── *.test.py          Unit tests
└── navi-os/             Next.js frontend
    ├── src/middleware.ts        Clerk route protection
    ├── src/app/chat/            Main chat UI
    ├── src/app/subagents/       Subagent management UI
    ├── src/app/api/             Next.js API routes (proxy to backend, Stripe webhooks, etc.)
    └── src/components/          Shared UI components
```

## Architecture

### The core graph

The backend runs every request through a LangGraph `StateGraph`. Entry point is a memory-recall step; from there, requests are classified and routed onto one of two independent tracks:

- **Task/code track** — for requests that require computation: `planner → skill_creation → executor → research → meditator`, capped at a fixed retry count. `planner` handles cold-start dispatch, post-execution auditing, and final report synthesis; `skill_creation` has an LLM write a complete, sandboxed Python function; `executor` runs it; `research` provides a web-search fallback after a failed attempt; `meditator` reasons about persistent failure patterns and proposes a different approach.
- **Conversational/RAG track** — for natural-language responses and knowledge-base queries that don't require code execution: `conversational` and `rag_researcher`.

Both tracks converge on a `memory_save` step before the request ends. The task/code and conversational/RAG tracks are architecturally independent — a fix on one does not automatically apply to the other.

Routing between the task and chat paths is a pure LLM classification (no hardcoded keyword list): the operative question is whether the answer could have a different, real-world-current value tomorrow than it does today, rather than how the question is phrased. Anything with real-world state — a price, a ranking, a person currently holding a role — routes to the task track, which carries out live search, scraping, and citation discipline. Only genuinely timeless facts stay on the conversational path.

### Process isolation

Every request runs the graph in its own subprocess rather than a thread, so an in-flight LLM call or long-running computation can be forcibly and cleanly cancelled (e.g. on client disconnect or explicit abort) without leaving the process in an inconsistent state. Uploaded DataFrames cross the process boundary via a one-shot temporary file handoff rather than shared memory.

## Key subsystems

### Vector memory

Long-term memory is stored in Pinecone, namespaced per user (spanning all of a user's chat threads) or per subagent (each subagent has its own persistent memory). The save pipeline redacts personally identifiable information first, then summarizes the redacted text into a short semantic summary before embedding and storing it — Pinecone never holds a user's verbatim words, only a paraphrased summary, and the save step fails closed (skips entirely rather than storing partial data). Retrieval embeds the incoming query, fetches nearest neighbors, and applies an LLM relevance filter to select only what's actually useful, rather than a raw similarity-score cutoff.

### Ethics gate

Requests on the task track pass through three independent ethics classifications (cold-start planning, code generation, and final report synthesis), each a fresh LLM call against a shared policy prompt — a request that slips past one gate has further chances to be caught. The conversational track makes its own independent call. Every gate fails closed: any error or unparseable response defaults to denying the request, never to proceeding.

### Skill creation & execution sandbox

Generated Python code is screened statically (AST and regex analysis) before it ever runs, blocking process-spawning, dynamic imports of sensitive modules, unsafe deserialization, and other sandbox-escape primitives. The execution sandbox allowlists only the modules a skill needs (pandas, numpy, requests, a scoped set of others) rather than exposing the full standard library. Scraping-oriented generated code fetches multiple URLs concurrently (bounded worker pool) rather than sequentially. Successful skills are cached so a repeated or near-identical request can skip straight to execution.

### Web research and verification

Web research runs through search (with a fallback provider), source-trust rating via a single batched LLM call, and concurrent scraping of the highest-trust results. Sources are bucketed by trust tier (primary/official data, established secondary journalism, and low-trust aggregator content), with different citation and corroboration rules per tier. Whether a response needs live verification at all is decided by a dedicated recency-and-specificity gate — one that also checks already-gathered content before deciding to search again, so it doesn't re-trigger a redundant search when the answer is already grounded in current, dated evidence. A separate post-generation audit re-checks any response drafted without web grounding for claims that should have been verified, and searches each flagged claim individually so evidence stays scoped to what was actually asked.

### Numeric precision rules

Headline computed answers (the specific figure a query asked for) are always reported at full precision — exact value, not abbreviated. Abbreviated notation (K/M/B) is reserved for chart axis labels and incidental figures in narrative commentary, never for the answer itself. Identifiers (customer IDs, order numbers, zip codes, etc.) are handled as an explicit exception to numeric-precision formatting, since they are not quantities.

### PII protection

Two structurally different safeguards apply at different points in the pipeline:

- **Dataset preview (code-generation phase)** — the dataset summary shown to the code-writing model (column names, dtypes, null counts, sample rows) has personally identifiable columns scrubbed before the model ever sees them, using a combination of column-name classification, content-shape regex matching, and a local named-entity-recognition fallback for names. The model still receives the structural information it needs to write correct code, just not real PII values.
- **Execution results (report-writing phase)** — output from executed code can legitimately contain real data values pulled from the user's own dataset, including PII, since that may be the literal correct answer to the user's question (e.g. "who are my top 5 customers"). A narrower, targeted redaction applies here for identifier formats that are essentially never a legitimate part of a business-analytics answer (e.g. government ID and payment-card-shaped numbers), without blanket-redacting names or contact details that might be the actual requested answer.

### Billing

Subscription tiers and overage rates are centralized in one configuration module. Enforcement is split across independent call sites (main chat and subagent chat), each following the same pattern: look up the user's plan, record usage, and check against configured limits. Overage events are reported to Stripe's metered billing asynchronously.

### Authentication

Clerk provides session authentication and JWT verification. Admin-only routes use a double-gate pattern: the frontend checks the caller's identity against an authorized admin list, and the backend independently re-verifies via a server-side shared secret — a route is not considered admin-protected unless both layers are present.

### Subagents & AWS S3 integration

Subagents can be connected to a user's own AWS S3 bucket via a one-click CloudFormation provisioning flow: the user launches a pre-filled quick-create stack in their own account, which creates a bucket and an IAM role that trusts the platform's service identity, scoped by an external ID tied to the subagent. The platform's own service credentials hold no standing permissions on any user's bucket — access is always via a scoped, per-subagent role assumption.

An experimental, currently-inactive path also supports exporting a subagent as a self-hosted container or a shareable trial sandbox link; this path does not share the platform's persistent memory system.

### Data retention

The general policy: uploaded data (datasets, RAG documents) is held in memory only, or in a short-TTL temporary cache — never persisted to a database. The one deliberate exception is chat message text itself, which is stored to support chat history. Generated images and computed results attached to a chat message are not persisted beyond the live response, only the text is. A generated-skill cache stores code and task text, never raw dataset rows.

### Evaluation harness

A golden-dataset evaluation harness runs test cases through the real in-process graph and scores generated responses for factual consistency against grounding material, serving as a regression safety net for hallucination-related bugs. The conversational and task/RAG tracks are evaluated separately, consistent with their architectural separation.

### Abuse detection & moderation

Ethics-gate denials are tracked as a durable, per-account lifetime count, independent of any shorter-lived event log, so a slow repeat offender isn't missed just because individual events have aged out of a rolling log. An account is automatically flagged once its denial count crosses a configured threshold; everything past flagging (warning, suspension, dismissing a false positive) is a manual, admin-only action, never automatic.

## Testing

- **Backend** — `*.test.py` files run directly (not via a test framework runner). External services (vector store, embeddings) are replaced with in-memory fakes, and persistence layers point at a fresh temporary database per test.
- **Frontend** — TypeScript compilation checks for fast correctness feedback, plus a unit test suite for components with dedicated coverage.

## Deployment

- **Backend** — packaged as a Docker container, deployed as a Hugging Face Space. Local SQLite persistence — persistence guarantees depend on the hosting platform's filesystem behavior across redeploys.
- **Frontend** — Next.js on Vercel; static assets deploy automatically on push.
