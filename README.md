Personal Finance Copilot

A LangGraph-based agent that answers personal finance questions by pairing retrieval over curated Indian regulatory publications with tools that access the user's own expense data — grounding advice in both official rules and actual spending patterns.

Deployed as a FastAPI app, with Swagger UI available at /docs.

The core query this architecture was built around:

"I spent ₹17,749 in September. Am I overspending on food, and what should I be targeting?"

Answering this requires both the MCP expense tools and document retrieval — used in sequence, and combined into a single response.

Status
Phase	What	State
1	FastAPI shell, startup lifecycle, SQLite checkpointing, /chat	done
2	LangSmith tracing	done
3	MCP expense tools	done
4	RAG over curated corpus	done (228 chunks indexed)
5	Human-in-the-loop approval + /resume	written, needs a live run
6	SSE streaming (/chat/stream)	written, needs a live run
7	Eval set (50 cases) + runner	written, needs a baseline
8	Docker + deploy	deferred
9	Thin UI client	deferred

MCP note: langchain-mcp-adapters 0.3.2 fails during the stdio handshake on this stack (McpError: Connection closed), while the raw mcp SDK completes the handshake fine and lists all 4 tools. Because of this, app/tools/mcp_client.py talks to the SDK directly instead. You can reproduce the issue with python scripts/diagnose_mcp.py.

Setup
powershell
cd "Personal Finance Copilot"

python -m venv .venv
.venv\Scripts\Activate.ps1        # cmd: .venv\Scripts\activate.bat

pip install -r requirements.txt

copy .env.example .env

Then fill in .env:

Variable	Where from
GROQ_API_KEY	console.groq.com/keys (free)
LANGSMITH_API_KEY	smith.langchain.com → Settings → API Keys (personal access token)

VS Code: Ctrl+Shift+P → Python: Select Interpreter → pick .venv. Skipping this makes the editor flag every package as missing, even though the app runs fine.

If pip fails on langchain-community

It pins langchain-core<1.0, which clashes with the core 1.x version that langgraph 1.x requires. If you hit this:

powershell
pip uninstall langchain-community
pip install langchain-classic

app/tools/vectorstore.py checks both locations and reports which one it found — nothing else needs to change.

Build the index (once)
powershell
python scripts/ingest.py --report    # inspect the corpus, build nothing
python scripts/ingest.py             # build it

--report prints per-PDF character counts, duplication ratio, table count, and an overall verdict. Run it every time you add new PDFs — it's how you catch a scanned or broken document before it corrupts retrieval.

The first real run downloads the embedding model (~130 MB for bge-small).

Drop additional PDFs into data/corpus/<topic>/ and re-run ingest. The folder name becomes the topic metadata the model cites. The biggest coverage gaps right now are tax (Income Tax Dept Taxpayer Information Series) and investing (SEBI booklets).

Run
powershell
uvicorn app.main:app --reload

http://127.0.0.1:8000/docs

Check GET /health first — tools_loaded should read 5: add_expense, list_expenses, summarize, list_categories, search_finance_docs. A lower number means something failed to load; the startup log identifies what, and the app intentionally starts anyway rather than crashing.

Things to try

Memory — Call POST /chat twice with the same thread_id, then hit GET /history/demo-1. Restart the server and call it again — the history persists.

Expense tools

json
{"thread_id": "demo-1", "message": "How much did I spend in September 2025, and which category was biggest?"}

Expect ₹17,749 total, with food making up 25.1%.

Retrieval

json
{"thread_id": "demo-2", "message": "What is a Business Correspondent in banking?"}

The response should cite its source document.

The composed query — the one that justifies the architecture

json
{"thread_id": "demo-3", "message": "Based on my September 2025 spending, am I overspending on food compared to standard budgeting guidance?"}

Watch the LangSmith trace unfold: summarize → search_finance_docs → answer. That ordering was never hardcoded.

Approval flow

json
{"thread_id": "demo-4", "message": "Add an expense of 450 rupees for an auto ride on 2025-09-28"}

Returns status: "approval_required". Kill the server, restart it, then send:

json
POST /resume  {"thread_id": "demo-4", "decision": "yes"}

Execution resumes exactly where it left off — because the state lives on disk, not in memory.

Streaming (Swagger buffers SSE, so use curl instead):

bash
curl -N -X POST http://127.0.0.1:8000/chat/stream \
  -H "Content-Type: application/json" \
  -d "{\"thread_id\":\"demo-5\",\"message\":\"What is compound interest?\"}"
Tests — no API key needed
powershell
python tests/smoke_phase1.py   # graph wiring, memory, thread isolation, persistence
python tests/smoke_phase3.py   # MCP over stdio: discovery, schemas, real tool calls
python tests/smoke_phase4.py   # RAG: index loads, passages carry citations
python tests/smoke_phase5.py   # approval: pause, restart, resume, decline, read-only passthrough

Phases 1, 3, and 5 use stub models, so they run free and deterministically. Phase 4 requires the index to be built first.

Evaluation
powershell
python scripts/run_eval.py                  # all 50 cases
python scripts/run_eval.py --kind composed  # one category
python scripts/run_eval.py --limit 5        # quick check

Produces a per-category report covering tool recall, forbidden-tool avoidance, approval/refusal behavior, and p50/p95 latency. Results are saved to evals/last_run.json and traced in LangSmith.

This is the standout part of the project. Record a baseline, change one variable, and re-run:

Knob	Where
Chunk size / overlap	scripts/ingest.py — CHUNK_SIZE, CHUNK_OVERLAP
Retrieval depth k	app/tools/rag.py — TOP_K
System prompt	app/prompts.py
Model / provider	.env — LLM_PROVIDER, GROQ_MODEL

A line like "Raised composed-query tool recall from 62% to 89% by X" carries more weight on a resume than another feature would.

Layout
app/
  main.py       FastAPI, startup lifecycle, /chat /resume /chat/stream /history
  config.py     every env-dependent choice (LLM + embedding provider, paths)
  graph.py      LangGraph assembly
  nodes.py      chat_node, approval_node, routing
  state.py      ChatState
  prompts.py    system prompt — a tuning surface
  schemas.py    request/response models; also the Swagger docs
  tools/
    mcp_client.py   spawns + discovers the expense server
    rag.py          FAISS retriever -> search_finance_docs
    embeddings.py   embedding factory (shared by ingest and serving)
    vectorstore.py  FAISS import shim across langchain packagings
mcp_server/     the expense MCP server (own process, no LangGraph awareness)
data/
  corpus/       source PDFs, by topic
  faiss_index/  built index (regenerate with scripts/ingest.py)
scripts/
  ingest.py     PDFs -> markdown -> chunks -> embeddings -> index
  run_eval.py   the eval harness
evals/
  dataset.json  50 labelled cases
tests/
Design notes

Startup vs. per-request. The checkpointer, LLM, MCP subprocess, and FAISS index are all built once inside the lifespan handler and stored on app.state. Request handlers simply use what's already loaded — building any of this per request would mean paying the setup cost on every message.

Three separate memories. Conversation history lives in checkpoints.db (per thread_id); document knowledge lives in data/faiss_index/ (shared, read-only); expenses live in the MCP server's own database. Each store serves a distinct purpose.

Why one tool uses MCP and the other doesn't. search_finance_docs closes over an in-memory FAISS index — putting it behind MCP would mean the server owning that index, which is a redesign, not a simple decorator swap. The expense tools have no such dependency and are independently useful, so they run behind MCP instead. Pointing Claude Desktop at mcp_server/main.py over stdio works unchanged — same server, different client, no code changes needed.

Why approval lives in the graph, not the server. interrupt() suspends the LangGraph run through the checkpointer. Since mcp_server/main.py has no awareness of LangGraph, the approval gate sits in approval_node, before ToolNode dispatches anything.

Declining still answers every tool call. Providers reject the next turn if any tool_call is missing a matching ToolMessage, so a refusal still injects one response per call instead of skipping them.

Tool errors are content, not exceptions. A malformed date returns something like "Error ... must be in YYYY-MM-DD format (got '01-09-2025')" as a normal tool result — so the model reads it and retries rather than the graph crashing. This effectively makes error messages part of the prompt design: they need to state the fix and echo back the bad value.

Extraction was chosen based on measurement. pypdf duplicated a defective page 48 times (267k characters from a single page) and flattened tables; pymupdf4llm deduplicates properly and outputs markdown tables, which also enables header-aware chunking. scripts/ingest.py --report reproduces these numbers.

Provider is a single env var. LLM_PROVIDER=groq|openai swaps the chat model without touching any graph code, letting the eval set A/B test providers directly. Since Groq doesn't serve embeddings, EMBEDDING_PROVIDER is configured separately.

Known gaps
The corpus currently holds only 2 documents, both in budgeting/. Tax, insurance, and investing categories are empty.
No authentication — anyone who can reach the API can read and write expenses.
Single-user design: expenses share one database rather than being split per-thread_id.
MUTATING_TOOLS in app/nodes.py is a hard-coded set — any new write tool must be added there manually, or it will bypass approval.
MCP tools are stateless — the adapter spawns a new server instance per call. This is simple and robust, but if latency becomes a concern, consider holding a persistent client.session() open instead.
