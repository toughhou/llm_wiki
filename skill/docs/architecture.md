# Architecture — llm-wiki Skill + MCP

## Goal

A single backend implementation of the `nashsu/llm_wiki` knowledge-base
algorithms, reachable from **any** AI host via three equivalent entry points:

```
                 ┌──────────────────────┐
                 │  skill/src/lib/*     │  ← single source of truth
                 │  (graph, search,     │
                 │   ingest, deep-      │
                 │   research, ...)     │
                 └──────────┬───────────┘
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
       ▼                    ▼                    ▼
 ┌──────────┐        ┌─────────────┐      ┌─────────────┐
 │  cli.ts  │        │ mcp-server  │      │  SKILL.md   │
 │          │        │  .ts        │      │  / HERMES   │
 │ shell    │        │ stdio JSON- │      │  loader     │
 │ scripts, │        │ RPC (Claude │      │  docs (any  │
 │ cron, CI │        │ / Cursor /  │      │  host's     │
 │          │        │ Copilot /   │      │  skill mech-│
 │          │        │ Codex /     │      │  anism that │
 │          │        │ Hermes ...) │      │  shells out │
 └──────────┘        └─────────────┘      └─────────────┘
```

## Why MCP is the primary entry point

MCP (Model Context Protocol) is currently the only protocol supported
**natively** by the major hosts users care about — Claude Desktop,
Cursor, VS Code Copilot Chat, OpenAI Codex CLI, Continue, Hermes
runtime. A single MCP stdio server reaches them all.

A skill manifest (Claude Skills `SKILL.md`, Hermes Skill, etc.) is just
a discovery/triggering hint that ultimately needs to *call something* —
either an MCP server or a CLI. Building MCP first means every skill
manifest can wrap the same backend without forking it.

CLI is kept as a third entry because:

- Not every workflow involves an LLM host (cron, CI, makefiles).
- It's the lowest-common-denominator interop format.
- It's how Hermes-style skills shell out today.

## Why a single shared `lib/`

The previous repo state had a duplicated `mcp-server/` tree containing a
*subset* of the `skill/lib/` files — the LLM-related libraries
(`llm-client`, `page-merge`, `ingest-cache`, etc.) had been ported to
`skill/lib/` only. That's a code-drift bomb: the next bug fix in
`graph-insights.ts` would have to be remembered twice. We collapsed it
to a single `skill/src/lib/` tree, then added a unified `dist/` with
both `cli.js` and `mcp-server.js` bin entries.

## Directory layout

```
skill/
├── src/
│   ├── lib/                  # core algorithms (Tauri-free)
│   │   ├── wiki-graph.ts
│   │   ├── graph-relevance.ts
│   │   ├── graph-insights.ts
│   │   ├── search.ts            (BM25 + RRF)
│   │   ├── path-utils.ts
│   │   ├── frontmatter.ts       (js-yaml)
│   │   ├── sources-merge.ts
│   │   ├── page-merge.ts        (LLM-assisted page merge)
│   │   ├── ingest-cache.ts      (SHA256 incremental cache)
│   │   ├── ingest-sanitize.ts
│   │   ├── ingest.ts            ★ two-stage LLM pipeline
│   │   ├── deep-research.ts     ★ web-search → synthesis → ingest
│   │   ├── llm-client.ts        (OpenAI-compatible SSE streaming)
│   │   ├── web-search.ts        (Tavily; configurable base URL)
│   │   ├── detect-language.ts
│   │   ├── output-language.ts
│   │   └── project-mutex.ts
│   ├── shims/                # Tauri → Node.js adapters
│   │   ├── fs-node.ts        (read/write/list)
│   │   ├── stores-node.ts    (env-driven config + activity logger)
│   │   └── embedding-stub.ts (no-op fallback)
│   ├── types/wiki.ts
│   ├── cli.ts                # entry point 1: command line
│   ├── mcp-server.ts         # entry point 2: MCP stdio server
│   └── test-server/
│       ├── fake-llm-server.ts # real local OpenAI-compatible HTTP server
│       └── e2e.ts             # end-to-end real test runner
├── dist/                     # tsc output (committed? no — built per install)
└── docs/                     # this file + usage-* + test-report.md
```

## Why the MCP server cannot use the host AI's model

A common question: *"I'm running this inside Cursor / Claude Desktop — can
`wiki_ingest` just use Cursor's model instead of requiring my own API key?"*

**No.** Here is why:

```
Host AI (Cursor / Claude)
  │
  │  MCP call: wiki_ingest(source_file="...")
  ▼
MCP server subprocess  ← isolated process, no model access
  │
  │  independent HTTP request to LLM API (OPENAI_API_KEY required)
  ▼
LLM provider (OpenAI / Ollama / OpenRouter / ...)
```

The MCP server is a plain Node.js process launched by the host. The MCP
protocol defines tool calls as function invocations with JSON
input/output — there is no mechanism for the subprocess to delegate a
generation request back to the host's model connection.

**Tools that require no LLM at all** (`wiki_status`, `wiki_search`,
`wiki_graph`, `wiki_insights`, `wiki_lint`) work with zero configuration.

**Tools that require LLM** (`wiki_ingest`, `wiki_deep_research`) must have
`OPENAI_API_KEY` (or `LLM_API_KEY`) set — or `LLM_BASE_URL` pointing to a
local Ollama server (no key required):

```json
"env": {
  "LLM_BASE_URL": "http://localhost:11434",
  "LLM_MODEL": "llama3.2"
}
```

## Configuration model

All runtime configuration is **env-var driven**. There are no config
files. This keeps the same code reachable from every host without
per-host config syntax.

| Variable | Purpose |
|---|---|
| `OPENAI_API_KEY` / `LLM_API_KEY` | LLM credentials |
| `LLM_BASE_URL` | OpenAI-compatible endpoint (Ollama, OpenRouter, ...) |
| `LLM_MODEL` | Model name (default `gpt-4o-mini`) |
| `LLM_PROVIDER` | `openai` / `anthropic` / `ollama` / `deepseek` |
| `TAVILY_API_KEY` | Tavily search (deep-research only) |
| `TAVILY_BASE_URL` | Override Tavily endpoint (test-only) |
| `EMBEDDING_*` | Optional vector search (gracefully degrades to BM25 only) |
| `WIKI_OUTPUT_LANGUAGE` | `auto` / `English` / `Chinese` / ... |
| `WIKI_PATH` | Default project path for MCP server |
| `SKILL_VERBOSE` | Mirror activity log to stderr |

## What is intentionally not ported

| Upstream feature | Reason |
|---|---|
| Image extraction (PDF/PPTX/DOCX) | Needs Rust pdfium binding |
| Vision-LLM caption pipeline | Needs multimodal endpoint config |
| Embedding generation | Optional; has a stub for graceful no-op |
| Sweep-reviews queue | Depends on long-lived Zustand React store |
| Chrome Web Clipper | Browser extension surface, not a skill |

REVIEW blocks emitted by the LLM are still **parsed** by `lib/ingest.ts`
and surfaced in the return value (CLI prints them as JSON; MCP renders
them in the tool reply), so a host can act on them.

## Test strategy

`skill/src/test-server/fake-llm-server.ts` is a real HTTP server
implementing the OpenAI-compatible Chat Completions SSE protocol and
the Tavily search REST protocol. The skill code is unchanged: real
`fetch`, real SSE parsing, real JSON decoding all execute. Only the
**upstream provider** is replaced with a deterministic local server,
which is the standard contract-test approach (vs. mocking out
`streamChat` in code, which would skip the SSE / fetch / network paths
entirely).

`skill/src/test-server/e2e.ts` drives every CLI command and every MCP
tool against a real on-disk wiki fixture and writes the raw transcript
to `skill/docs/test-report.md`.

Run with:
```bash
cd skill && npm run test:e2e
```
