---
name: token-cost-optimization
description: Use when you need to reduce LLM API token usage or costs — prompt compression, model tiering, caching, context trimming, tool call minimization, output length control, and automated context compression via headroom-ai.
version: 1.2.0
author: protick-bjit2019
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [tokens, cost, llm, optimization, prompting, caching, context, headroom, compression, context-window]
    homepage: https://github.com/protick-bjit2019/token-cost-optimization
    related_skills: [andrej-karpathy, plan, spike]
---

# Token Cost Optimization

## Overview

Every token costs money and latency. This skill encodes battle-tested techniques for cutting LLM API spend without degrading output quality. Covers both **design-time** strategies (how you structure prompts and tools) and **runtime** strategies (caching, batching, model routing).

Applies to any provider (OpenAI, Anthropic, Mistral, Gemini, etc.) and also to local inference where you pay in GPU time instead of dollars.

---

## When to Use

- You are building an LLM-powered feature and want to keep costs low from day one.
- An existing app's API bill is unexpectedly high and you're diagnosing why.
- You want to set a per-request token budget and enforce it programmatically.
- You are selecting which model tier to use for a given task.
- You want to add semantic caching or prompt caching to an existing pipeline.

**Don't use for:**
- One-off interactive chat (costs are negligible; don't over-engineer).
- Tasks where output quality is the only constraint (compression may hurt).

---

## Strategy 0 — headroom-ai: Drop-In Context Compression (60–95% savings)

[headroom](https://github.com/chopratejas/headroom) is an open-source library (17k+ ⭐, Apache 2.0) that automatically compresses tool outputs, logs, RAG chunks, files, and conversation history before they reach the LLM — with no accuracy loss.

**Benchmarked savings on real agent workloads:**

| Workload | Before | After | Savings |
|---|---:|---:|---:|
| Code search (100 results) | 17,765 | 1,408 | **92%** |
| SRE incident debugging | 65,694 | 5,118 | **92%** |
| GitHub issue triage | 54,174 | 14,761 | **73%** |
| Codebase exploration | 78,502 | 41,254 | **47%** |

**Accuracy on benchmarks: unchanged.** (GSM8K ±0.000, TruthfulQA +0.030, SQuAD v2 97%, BFCL 97% with 19–32% compression.)

### Installation

```bash
# Python (requires 3.10+)
pip install "headroom-ai[all]"    # everything included

# Granular extras:
# [proxy]     — proxy server + HTTP API
# [ml]        — Kompress ModernBERT (requires PyTorch)
# [code]      — CodeCompressor tree-sitter AST
# [mcp]       — MCP server tools (lightweight)
# [memory]    — persistent memory (SQLite + HNSW)
# [relevance] — embedding-based relevance scoring
# [image]     — image compression via ML router
# [langchain] — HeadroomChatModel wrapper
# [agno]      — HeadroomAgnoModel wrapper
# [evals]     — GSM8K, SQuAD, BFCL eval suite
# [llmlingua] — LLMLingua-2 ML compression (~2GB, 10-30s cold start)

# TypeScript / Node.js (requires Node 18+)
npm install headroom-ai       # communicates with local Python proxy

# Docker
docker pull ghcr.io/chopratejas/headroom:latest
docker run -p 8787:8787 ghcr.io/chopratejas/headroom:latest
# Tags: latest, <version>, nonroot, code, slim, code-slim (and -nonroot variants)
```

### Three usage modes

**Mode 1 — Library (minimal code change):**

```python
from headroom import SmartCrusher, SmartCrusherConfig

crusher = SmartCrusher(SmartCrusherConfig(
    max_items_after_crush=20,   # keep top-N rows by importance
    first_fraction=0.30,        # always keep first 30% of remaining budget
    last_fraction=0.15,         # always keep last 15%
    factor_out_constants=True,  # hoist repeated fields to save tokens
))

# ✅ Use .crush() — returns CrushResult
result     = crusher.crush(tool_output_str, query="error root cause")
compressed = result.compressed   # str — drop-in replacement for tool output

# CrushResult attributes (confirmed v0.23.0):
#   .compressed   — str, the compressed content
#   .original     — str, the original content unchanged
#   .strategy     — str, e.g. "lossless:table(200->len=31363)"
#   .was_modified — bool, False if content was too small to compress

# ⚠️  DO NOT use crush_array_json() — it returns a metadata dict that
#     wraps the content in extra keys, increasing token count by 50%+.
#     Always use crush() for tool outputs.

# Inject compressed result into messages
msgs_after = [
    *msgs_before[:-1],   # keep all but the last tool-result message
    {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": tool_use_id,
         "content": compressed}
    ]},
]
response = client.messages.create(model="claude-sonnet-4-5", messages=msgs_after, ...)

# Check what happened
print(f"Strategy  : {result.strategy}")
print(f"Modified  : {result.was_modified}")
```

**Mode 1b — HeadroomClient (full-featured drop-in):**

```python
from headroom import HeadroomClient, SmartCrusherConfig

client = HeadroomClient(
    default_mode="optimize",          # "audit" | "optimize" | "simulate"
    smart_crusher=SmartCrusherConfig(max_items_after_crush=20),
)

# Drop-in for any OpenAI-compatible client
response = client.chat.completions.create(
    model="claude-sonnet-4-5",
    messages=messages,
)

# Preview savings without making an LLM call
sim = client.chat.completions.simulate("claude-sonnet-4-5", messages)
print(sim.tokens_before, sim.tokens_after, sim.compression_ratio)

# Stats
print(client.get_summary())   # total_tokens_saved, avg_compression_ratio, total_cost_saved_usd
print(client.get_stats())     # session-level breakdown

# Per-tool compression config (override per request)
response = client.chat.completions.create(
    ...,
    headroom_tool_profiles={
        "important_tool": {"skip_compression": True},
        "search_tool":    {"max_items_after_crush": 25},
    }
)
```

**Mode 1c — Persistent Memory wrapper:**

```python
from headroom import with_memory
from openai import OpenAI   # or Anthropic, Azure, Groq — any OpenAI-compatible client

client = with_memory(OpenAI(), user_id="alice", session_id="session-42")

# All calls now auto-extract and inject relevant memories
response = client.chat.completions.create(model="gpt-4o", messages=messages)

# Manual memory ops
client.memory.add("User prefers concise answers", category="PREFERENCE", importance=0.9)
client.memory.search("preferred coding style", top_k=5)
client.memory.stats()
client.memory.clear()

# Memory scopes: User (all sessions) > Session > Agent > Turn
# Categories: PREFERENCE, FACT, CONTEXT, ENTITY, DECISION, INSIGHT
# Backends:   LOCAL (all-MiniLM-L6-v2) | OPENAI (text-embedding-3-small) | OLLAMA
# Storage:    SQLite + HNSW (vector) + FTS5 (full-text) — fully embedded, no external services
# Temporal versioning:
client.memory.supersede(old_memory_id, "updated content")   # creates audit chain
client.memory.get_history(memory_id)                         # full audit trail
```

**Mode 1d — Wrap existing SDK client (one line):**

```python
from headroom import withHeadroom
from anthropic import Anthropic
from openai import OpenAI

anthropic_client = withHeadroom(Anthropic())   # all Anthropic calls auto-compressed
openai_client    = withHeadroom(OpenAI())      # all OpenAI calls auto-compressed
```

**Mode 1e — Batch compression:**

```python
from headroom import compress_batch, count_tokens_text, count_tokens_messages

# Compress multiple content strings at once
results = compress_batch([tool_output_1, tool_output_2, log_text])

# Built-in token counting (no tiktoken import needed)
n = count_tokens_text("your text here", model="claude-sonnet-4-5")
n = count_tokens_messages(messages, model="claude-sonnet-4-5")
```

**Mode 2 — Proxy (zero code changes):**

```bash
# Start local proxy on port 8787
headroom proxy --port 8787

# All proxy options:
# --host 0.0.0.0              Bind address
# --budget 5.00               Daily USD spend cap
# --log-file /tmp/hr.jsonl    JSONL request log
# --no-intelligent-context    Fall back to RollingWindow
# --llmlingua                 Enable LLMLingua-2 ML compression
# --llmlingua-device auto     auto | cuda | cpu | mps
# --llmlingua-rate 0.3        Target compression ratio
# --backend bedrock           bedrock | vertex_ai | azure | openrouter
# --no-telemetry              Disable anonymous telemetry
# --no-optimize               Disable all compression (pass-through only)
# --no-cache                  Disable CCR cache
# --no-rate-limit             Disable rate limiting
# --no-compress-first         Skip compression; apply caching only
# --openai-api-url URL        Upstream OpenAI-compatible base URL

# Point any OpenAI-compatible client at it
OPENAI_BASE_URL=http://localhost:8787/v1 your-app
ANTHROPIC_BASE_URL=http://localhost:8787 claude

# Bypass header for a specific request (skip compression)
curl ... -H "x-headroom-bypass: true"

# HTTP API endpoints
GET  /health            # health check + session stats
GET  /stats             # live savings (JSON)
GET  /stats-history     # hourly/daily/weekly/monthly rollups (?format=csv&series=weekly)
GET  /metrics           # Prometheus metrics
GET  /dashboard         # savings dashboard
POST /v1/compress       # compression-only, no LLM call (used by TS SDK)
```

**Mode 3 — Agent wrap (one command):**

```bash
headroom wrap claude     # Claude Code (--memory and --code-graph flags)
headroom wrap codex      # OpenAI Codex (shares memory with Claude)
headroom wrap cursor     # Cursor (prints config, paste once)
headroom wrap aider      # starts proxy + launches
headroom wrap copilot    # starts proxy + launches
headroom wrap gemini     # Gemini
headroom wrap openclaw   # installs as ContextEngine plugin
```

**Mode 4 — MCP server (Claude Code, Cursor, any MCP client):**

```bash
headroom mcp install                             # register with Claude Code (stdio)
headroom mcp install --remote <url>              # configure remote HTTP MCP
headroom mcp install --force                     # overwrite existing config
headroom mcp serve                               # run MCP server (stdio)
headroom mcp serve --transport http --port 8080  # HTTP transport for remote agents
headroom mcp serve --debug                       # debug mode
headroom mcp status                              # check server status
headroom mcp uninstall                           # remove MCP config

# MCP tools exposed to the LLM:
# headroom_compress   — compress any content; returns hash + savings stats
# headroom_retrieve   — retrieve original by hash (query="" for full, or filtered)
# headroom_stats      — session stats: compressions, tokens_saved, cost_saved_usd

# Transports:
# stdio (local)              →  headroom mcp install → claude
# Streamable HTTP (remote)   →  headroom mcp serve --transport http --port 8080
# Via proxy (auto)           →  POST/GET/DELETE  http://host:8787/mcp
```

### What headroom compresses (and what it doesn't touch)

| Content type | Compressor | Typical savings |
|---|---|---|
| JSON arrays of dicts | SmartCrusher `.crush()` — lossless table reformat + statistical sampling | 42–95% |
| JSON arrays of strings | Dedup + adaptive sampling | 60–90% |
| Search results (`file:line:content`) | SearchCompressor | 80–95% |
| Build / test logs | LogCompressor — pattern clustering | 85–95% |
| Diffs (unified format) | DiffCompressor | 60–80% |
| Source code | CodeCompressor — AST-aware via tree-sitter | 40–70% |
| Plain text | TextCompressor / Kompress-base (ModernBERT ML) | 30–80% |
| HTML | HTMLCompressor (trafilatura-based) | ~95% |
| Images | Image ML Router | 40–90% |
| Short content (<200 tokens) | **Not compressed** (overhead > savings) | — |
| User messages | **Never compressed** (intent preserved) | — |
| System prompt content | **Preserved**; CacheAligner moves dynamic parts to tail | — |

> ⚠️ **crush_array_json() pitfall:** Returns a metadata dict wrapping the original JSON — increases tokens by 50%+. Always use **`crush()`** for tool outputs.

### How the pipeline works internally

Three-stage compression pipeline:

1. **CacheAligner** — extracts dynamic content (dates, UUIDs, tokens) from system prompt prefix and moves it to the tail. Stabilizes prefixes so provider KV caches (Anthropic `cache_control`, OpenAI prefix cache) actually hit. Sub-millisecond overhead.

### Stage 3: Context Management

Two strategies — both configurable via `HeadroomClient` or env vars:

**IntelligentContextManager** (default) — Scores messages on 6 dimensions before dropping:

| Dimension | Weight |
|---|---|
| `recency` | 0.20 |
| `semantic_similarity` | 0.20 |
| `toin_importance` | 0.25 |
| `error_indicator` | 0.15 |
| `forward_reference` | 0.15 |
| `token_density` | 0.05 |

Drops lowest-scored messages; originals stored in CCR for retrieval. Disable with `--no-intelligent-context`.

**RollingWindow** (fallback) — drops oldest messages first; preserves system prompt + recent turns.

**TOIN (Tool Output Intelligence Network)** — learns compression patterns across sessions. Feeds learned field-importance stats into SmartCrusher and IntelligentContext scoring. Falls back to statistical heuristics on cold start.

**CCR (Compress-Cache-Retrieve) — reversible compression:** originals are never deleted; they're stored in a local SQLite cache. If the LLM needs full data it calls `ccr_retrieve("<hash>")` to get it back. Headroom is the only major compressor that is fully reversible.

### SDK integrations

```python
# Wrap existing client — Anthropic or OpenAI
from headroom import withHeadroom
from anthropic import Anthropic
from openai import OpenAI

client = withHeadroom(Anthropic())   # all calls auto-compressed
client = withHeadroom(OpenAI())

# Persistent memory
from headroom import with_memory
client = with_memory(OpenAI(), user_id="alice")

# LangChain
from headroom.integrations.langchain import HeadroomChatModel
llm = HeadroomChatModel(your_llm)   # supports memory, retrievers, tools, streaming, async

# Agno
from headroom.integrations.agno import HeadroomAgnoModel
model = HeadroomAgnoModel(your_model)

# Strands
from headroom.integrations.strands import HeadroomStrandsModel
model = HeadroomStrandsModel(...)

# LiteLLM (single callback — covers all 100+ providers)
import litellm
litellm.callbacks = [HeadroomCallback()]

# Vercel AI SDK (TypeScript)
import { withHeadroom, headroomMiddleware } from 'headroom-ai/vercel-ai';
const model = withHeadroom(openai('gpt-4o'));
// Or: wrapLanguageModel({ model, middleware: headroomMiddleware() })

# ASGI middleware
from headroom.integrations.asgi import CompressionMiddleware
app.add_middleware(CompressionMiddleware)

# Multi-agent / CrewAI / LangGraph / OpenAI Agents SDK — SharedContext
from headroom import SharedContext
ctx = SharedContext()
ctx.put("key", big_content, agent="claude")
ctx.get("key")          # compressed view
ctx.get("key", full=True)  # original
ctx.stats()             # entries, totalOriginalTokens, totalCompressedTokens, savingsPercent
ctx.keys(); ctx.clear()

# Cloud backends (via proxy --backend flag)
# bedrock   — AWS Bedrock   (--region us-east-1)
# vertex_ai — Google Vertex (--region us-central1)
# azure     — Azure OpenAI
# openrouter — 400+ models
```

### Relevance Scoring API

Score messages/chunks by relevance before compression to keep the most important content.

```python
from headroom.relevance import BM25Scorer, EmbeddingScorer, HybridScorer, create_scorer

# BM25 — keyword-based, no GPU, fast
scorer = BM25Scorer()
scores = scorer.score(query="error root cause", documents=chunks)

# Embedding — semantic, requires [relevance] extra
scorer = EmbeddingScorer(model="all-MiniLM-L6-v2")
scores = scorer.score(query="error root cause", documents=chunks)

# Hybrid — BM25 + embedding blend (best accuracy, slightly slower)
scorer = HybridScorer(bm25_weight=0.4, embedding_weight=0.6)
scores = scorer.score(query="error root cause", documents=chunks)

# Factory helper — infers best scorer from available deps
scorer = create_scorer("hybrid")   # "bm25" | "embedding" | "hybrid"
ranked = sorted(zip(scores, chunks), reverse=True)
top_k  = [c for _, c in ranked[:10]]
```

Install extras for richer scoring:
```bash
pip install "headroom-ai[relevance]"   # embedding scorer (all-MiniLM-L6-v2)
```

---

### TransformPipeline — custom compression chains

Chain multiple compressors into a single pipeline with fallback logic.

```python
from headroom import TransformPipeline, SmartCrusher, LogCompressor, SearchCompressor

pipeline = TransformPipeline([
    SearchCompressor(),   # try search-result format first
    LogCompressor(),      # fall back to log clustering
    SmartCrusher(),       # final fallback for generic content
])

result = pipeline.compress(raw_text, query="deployment error")
print(result.compressed)   # uses whichever stage matched
print(result.strategy)     # e.g. "search_compressor" | "log_compressor" | "smart_crusher"
```

---

### UniversalCompressor — multi-type batch

Automatically detect content type and apply the right compressor per item.

```python
from headroom import UniversalCompressor

uc = UniversalCompressor()

# Single item — auto-detects type (JSON, log, diff, code, HTML, text)
result = uc.compress(raw_content, query="why did the build fail?")

# Batch — mixed types in one call
results = uc.compress_batch([
    tool_output_json,
    build_log_text,
    git_diff_text,
    html_page,
], query="deployment failure")

# Each result has .compressed, .strategy, .was_modified, .original
```

---

### Provider cache optimization (built-in)

After compression, headroom automatically applies provider-specific cache hints:

| Provider | Mechanism | Cache savings |
|---|---|---|
| Anthropic | `cache_control` blocks on stable prefix | Up to 90% on repeated tokens |
| OpenAI | Prefix alignment for automatic caching | Up to 50% on repeated tokens |
| Google | `CachedContent` API | Up to 75% on repeated tokens |

### Cross-agent shared memory

```python
from headroom import SharedContext

ctx = SharedContext()
ctx.put("key", value, agent="claude")      # store from any agent
ctx.get("key")                              # retrieve from any agent (Claude, Codex, Gemini)
```

`headroom learn` mines failed agent sessions and writes corrections to `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` automatically.

### headroom vs. alternatives

| Tool | Scope | Local | Reversible |
|---|---|:---:|:---:|
| **headroom** | All context — tools, RAG, logs, files, history | ✅ | ✅ |
| RTK | CLI command outputs only | ✅ | ❌ |
| lean-ctx | CLI commands, MCP tools, editor rules | ✅ | ❌ |
| Compresr / Token Co. | Text sent to hosted API | ❌ | ❌ |
| OpenAI Compaction | Conversation history only | ❌ | ❌ |

> headroom ships RTK internally and can also use lean-ctx as the context tool (`HEADROOM_CONTEXT_TOOL=lean-ctx`).

### headroom learn (failure mining)

```bash
headroom learn --project ./my-repo   # dry-run: show recommendations
headroom learn --apply               # write to CLAUDE.md / AGENTS.md / GEMINI.md
headroom learn --all                 # analyze all discovered projects
headroom learn --claude-dir PATH     # override path to write/read CLAUDE.md
```

- Mines failed sessions from `~/.claude/projects/*.jsonl`
- Learns: environment facts, file path corrections, search scope, command patterns, known large files
- Updates files with marker blocks: `<!-- headroom:learn:start -->` … `<!-- headroom:learn:end -->`

### headroom perf & evals

```bash
headroom perf                                    # benchmark compression on current session
python -m headroom.evals suite --tier 1          # run GSM8K, TruthfulQA, SQuAD v2, BFCL
pip install "headroom-ai[evals]"                 # required for evals
```

---

### Configuration & Environment Variables

| Variable | Default | Description |
|---|---|---|
| `HEADROOM_DEFAULT_MODE` | `optimize` | `audit` \| `optimize` \| `simulate` |
| `HEADROOM_STORE_URL` | temp dir | SQLite DB path for CCR + metrics |
| `HEADROOM_API_KEY` | — | API key for Headroom Cloud |
| `HEADROOM_LOG_LEVEL` | `INFO` | Logging level |
| `HEADROOM_SAVINGS_PATH` | `~/.headroom/proxy_savings.json` | Persistent savings file |
| `HEADROOM_TELEMETRY` | `on` | Set `off` to disable anonymous telemetry |
| `HEADROOM_PORT` | `8787` | Proxy port |
| `HEADROOM_HOST` | `0.0.0.0` | Proxy bind address |
| `HEADROOM_CONTEXT_TOOL` | — | Set to `lean-ctx` to use lean-ctx CLI tool |
| `OPENAI_API_KEY` | — | OpenAI key |
| `ANTHROPIC_API_KEY` | — | Anthropic key |

**Three modes explained:**
| Mode | Behavior |
|---|---|
| `audit` | Observes and logs only — no modifications made |
| `optimize` | Applies safe, deterministic transforms (default) |
| `simulate` | Returns compression plan without making an LLM API call |

**Custom model config** — `~/.headroom/models.json` or `HEADROOM_MODEL_LIMITS=<json>`:
Pattern-based inference for unknown models (`*opus*`, `*sonnet*`, `gpt-4o*`, `o1*`, etc.)

### Observability

```python
# Simulation mode — preview savings before committing
sim = client.chat.completions.simulate("claude-sonnet-4-5", messages)
print(f"Would save: {sim.tokens_before - sim.tokens_after} tokens")
print(f"Waste signals: {sim.waste_signals}")
# Waste signals: json_bloat_tokens, html_noise_tokens, whitespace_tokens,
#                dynamic_date_tokens, repetition_tokens

# Prometheus metrics (proxy)
GET /metrics
# headroom_requests_total, headroom_tokens_saved_total,
# headroom_compression_ratio_bucket, headroom_latency_seconds_bucket,
# headroom_cache_hits_total

# Report generation
from headroom import generate_report
generate_report(format="html", period="day")   # or "markdown"
```

---

### Error Handling

Always catch headroom-specific exceptions to degrade gracefully (pass-through if compression fails).

```python
from headroom.exceptions import (
    HeadroomError,              # base exception for all headroom errors
    HeadroomConnectionError,    # proxy / network unreachable
    HeadroomConfigError,        # invalid configuration (bad env var, missing field)
    HeadroomCompressionError,   # compressor raised an error (bad input, type mismatch)
    HeadroomMemoryError,        # memory backend unavailable or read/write failure
    HeadroomRateLimitError,     # proxy rate limit exceeded (--no-rate-limit to disable)
    HeadroomBudgetError,        # daily spend cap reached (--budget flag)
)

try:
    result = crusher.crush(tool_output, query=query)
    compressed = result.compressed
except HeadroomCompressionError:
    compressed = tool_output   # graceful fallback — pass raw content through
except HeadroomConnectionError:
    # proxy is down — switch to direct client
    compressed = tool_output
except HeadroomError as e:
    # catch-all for any headroom issue
    compressed = tool_output
```

| Exception | When it fires | Graceful fallback |
|---|---|---|
| `HeadroomError` | Base — any headroom issue | Pass raw content through |
| `HeadroomConnectionError` | Proxy unreachable / network down | Direct LLM call |
| `HeadroomConfigError` | Bad env var, missing required field | Log + skip compression |
| `HeadroomCompressionError` | Bad input type, compressor failure | Pass raw content through |
| `HeadroomMemoryError` | SQLite/HNSW backend failure | Skip memory ops, continue |
| `HeadroomRateLimitError` | Proxy rate limit exceeded | Retry with backoff |
| `HeadroomBudgetError` | Daily `--budget` cap reached | Switch to cheaper tier |

---

## Strategy 1 — Measure First

Never optimize blind. Instrument token usage before changing anything.

```python
# openai SDK: response.usage is always present
response = client.chat.completions.create(...)
print(response.usage)  # CompletionUsage(prompt_tokens=..., completion_tokens=..., total_tokens=...)

# anthropic SDK
response = client.messages.create(...)
print(response.usage)  # Usage(input_tokens=..., output_tokens=...)
```

Key metrics to track per request:
| Metric | Formula |
|---|---|
| Prompt tokens | tokens used by the input |
| Completion tokens | tokens in the output |
| Cost | prompt_tokens × price_in + completion_tokens × price_out |
| Prompt-to-completion ratio | prompt / completion — high ratios mean you're paying for context |

Use **tiktoken** (OpenAI) or `anthropic.count_tokens()` to estimate costs before sending:

```python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4o")
n = len(enc.encode(your_prompt_text))
print(f"{n} tokens ≈ ${n / 1_000_000 * 5:.4f} at $5/1M input")
```

---

## Strategy 2 — Model Tiering

Route tasks to the cheapest model that can do the job. Typical tiers:

| Tier | Examples | Use for |
|---|---|---|
| Micro / fast | GPT-4o-mini, Claude Haiku, Gemini Flash | Classification, extraction, summarization, short Q&A |
| Standard | GPT-4o, Claude Sonnet, Gemini Pro | Multi-step reasoning, coding, analysis |
| Frontier | o3, Claude Opus, Gemini Ultra | Research, long-horizon agents, highest accuracy |

**Rule of thumb:** Start with the cheapest tier. Only escalate if evals fail.

```python
# Simple routing: route by estimated complexity
def pick_model(prompt: str) -> str:
    n_tokens = len(enc.encode(prompt))
    if n_tokens < 500 and "summarize" in prompt.lower():
        return "gpt-4o-mini"    # 15x cheaper than gpt-4o
    return "gpt-4o"
```

---

## Strategy 3 — Prompt Compression

### 3a. Remove Boilerplate

Every word in the system prompt costs tokens on every call. Ruthlessly remove:
- "You are a helpful AI assistant" — LLMs know they're LLMs.
- Long preambles explaining what the AI should generally do.
- Repeated instructions that are already default behavior.

### 3b. Use Terse Formatting

```
BEFORE (47 tokens):
"Please carefully analyze the following customer feedback and provide
a detailed response identifying the main issues."

AFTER (12 tokens):
"Analyze feedback. List issues."
```

### 3c. Few-Shot → Zero-Shot

Each example in a few-shot prompt costs its full token count. Test if zero-shot works first. If accuracy suffers, add one example, not five.

### 3d. Compress Reference Documents

When inserting long documents into context, summarize or extract only the relevant sections:

```python
# Instead of stuffing 10k-token PDF into context:
# 1. Chunk it
# 2. Embed chunks
# 3. Retrieve top-k by similarity
# 4. Inject only top-k (typically 3-5 chunks × ~300 tokens = ~1k tokens total)
```

---

## Strategy 4 — Context Window Management

### Sliding Window (for long conversations)

Keep only recent messages when history grows too large:

```python
MAX_HISTORY_TOKENS = 4000

def trim_history(messages: list[dict], enc) -> list[dict]:
    # Always keep system prompt (index 0)
    system = messages[:1]
    history = messages[1:]
    
    # Count from most recent until budget exhausted
    kept, budget = [], MAX_HISTORY_TOKENS
    for msg in reversed(history):
        cost = len(enc.encode(msg["content"]))
        if cost > budget:
            break
        kept.append(msg)
        budget -= cost
    
    return system + list(reversed(kept))
```

### Summarize Instead of Truncate

When truncating loses important context, summarize old history with a cheap model first:

```python
def summarize_old_history(old_msgs: list[dict]) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",  # cheap model for summarization
        messages=[
            {"role": "user", "content": f"Summarize this conversation in ≤100 words:\n{old_msgs}"}
        ],
        max_tokens=150,
    )
    return response.choices[0].message.content
```

---

## Strategy 5 — Prompt Caching

**Anthropic** and **OpenAI** offer explicit prompt caching — identical prefix tokens are cached at a 90% discount on repeat calls.

### Anthropic `cache_control`

```python
response = anthropic.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": LARGE_SYSTEM_PROMPT,        # long static prefix
            "cache_control": {"type": "ephemeral"}  # mark for caching
        }
    ],
    messages=[{"role": "user", "content": user_question}]
)
# cache_read_input_tokens shows how many were served from cache
print(response.usage.cache_read_input_tokens)
```

**Rules for Anthropic prompt caching:**
- Minimum cacheable prefix: **1024 tokens** (Sonnet/Haiku) or **2048 tokens** (Opus).
- Cache TTL: **5 minutes** (refreshed on each cache hit).
- Cost: cache write = 1.25× normal; cache read = 0.1× normal.
- Structure your prompt so the STATIC part comes first (system prompt, documents) and the DYNAMIC part (user input) comes last.

### OpenAI Automatic Caching

OpenAI caches automatically — no code changes needed. The first 1024+ token prefix of any request is cached. You get a discount automatically if the same prefix is reused within ~1 hour.

Check with: `response.usage.prompt_tokens_details.cached_tokens`

---

## Strategy 6 — Semantic Caching

Cache at the semantic level — if a new query is very similar to a past query, return the cached answer.

```python
# Minimal semantic cache with sentence-transformers + numpy
# pip install sentence-transformers numpy
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("all-MiniLM-L6-v2")  # 80MB, fast
cache: list[tuple[np.ndarray, str]] = []          # (embedding, answer)

SIMILARITY_THRESHOLD = 0.92

def cached_query(question: str) -> str | None:
    q_emb = model.encode(question)
    for emb, answer in cache:
        similarity = np.dot(q_emb, emb) / (np.linalg.norm(q_emb) * np.linalg.norm(emb))
        if similarity > SIMILARITY_THRESHOLD:
            return answer
    return None

def store_in_cache(question: str, answer: str):
    cache.append((model.encode(question), answer))
```

For production: use **Redis** with vector search or **Qdrant/Weaviate** as the cache backend.

---

## Strategy 7 — Output Length Control

Completion tokens are often more expensive per-token than input tokens. Control output length explicitly:

```python
# Hard limit
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    max_tokens=256,    # hard cutoff — set per task
)

# Soft limit via prompt instruction
system = "Respond in ≤3 sentences. Be direct."

# Structured output reduces verbose prose
# Ask for JSON instead of paragraphs — usually 2-5× fewer tokens
system = "Respond as JSON: {\"issue\": str, \"severity\": 1-5, \"fix\": str}"
```

Use JSON/structured output mode for extraction tasks — it forces concise, parseable answers.

---

## Strategy 8 — Tool Call Minimization

Each tool call round-trip adds tokens (tool schema + tool result injected back into context). Minimize by:

1. **Batch tool calls** — request multiple results in one call instead of chaining.
2. **Filter tool schemas** — only include tools relevant to the current task. Every tool definition in the schema costs ~100-300 tokens per call.
3. **Inline small lookups** — if a tool just returns a short string, compute it before the LLM call and inject it directly into the prompt.
4. **Return minimal tool results** — truncate or summarize large tool outputs before injecting them:

```python
def truncate_tool_result(result: str, max_chars: int = 2000) -> str:
    if len(result) <= max_chars:
        return result
    return result[:max_chars] + f"\n[...truncated {len(result) - max_chars} chars]"
```

---

## Strategy 9 — Batching

If you're making many independent LLM calls, batch them:

```python
# OpenAI Batch API — 50% discount, 24h turnaround
# Good for: eval pipelines, bulk classification, offline enrichment
import json

requests = [
    {"custom_id": f"req-{i}", "method": "POST", "url": "/v1/chat/completions",
     "body": {"model": "gpt-4o-mini", "messages": [{"role": "user", "content": q}]}}
    for i, q in enumerate(questions)
]

# Write JSONL file, upload, then call client.batches.create(...)
```

For Anthropic, use the **Message Batches API** (`client.beta.message_batches.create(...)`).

---

## Strategy 10 — Token Budget Enforcement

Fail fast rather than blowing the budget silently:

```python
TOKEN_BUDGET = 8_000  # per request hard limit

def safe_complete(messages: list[dict]) -> str:
    total = sum(len(enc.encode(m["content"])) for m in messages)
    if total > TOKEN_BUDGET:
        raise ValueError(f"Prompt exceeds budget: {total} > {TOKEN_BUDGET} tokens")
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        max_tokens=min(1024, TOKEN_BUDGET - total),
    )
    return response.choices[0].message.content
```

---

## Publishing a Skill to GitHub for Peer Sharing

When sharing a skill via `hermes skills install <url>`, the SKILL.md **must** have complete frontmatter or peers get a warning:
> "The file lacks proper Hermes SKILL.md frontmatter."

Required frontmatter fields for clean peer install:

```yaml
---
name: skill-name
description: Use when ...   # ≤1024 chars, starts with "Use when"
version: 1.0.0
author: your-github-username
license: MIT
platforms: [linux, macos, windows]   # ← REQUIRED — missing = warning
metadata:
  hermes:
    tags: [tag1, tag2]
    homepage: https://github.com/you/skill-name
    related_skills: [other-skill]
---
```

**Publish workflow:**
```bash
# 1. Create repo with the skill files
mkdir my-skill && cd my-skill
cp ~/.hermes/skills/<category>/<name>/SKILL.md .
cp -r references/ templates/ .        # include support files

# 2. Write README.md with install command, smoke test results, file list
# 3. Init, create remote, push
git init && git add .
git commit -m "feat: initial release"
gh repo create my-skill --public --source . --push

# 4. Peers install with:
hermes skills install https://raw.githubusercontent.com/<user>/<repo>/master/SKILL.md
```

**Validate frontmatter before pushing:**
```python
import yaml, re, pathlib
content = pathlib.Path("SKILL.md").read_text(encoding="utf-8")
assert content.startswith("---")
m = re.search(r'\n---\s*\n', content[3:])
fm = yaml.safe_load(content[3:m.start()+3])
assert "name" in fm and "description" in fm and "platforms" in fm
assert len(fm["description"]) <= 1024
print("✅ Frontmatter valid")
```

---

## Smoke Test Output Format

When reporting smoke test results, use this format (emoji headers, "Prompt tokens" label, system counted as a message):

```
==============================================================
🧪 TOKEN COST OPTIMIZATION — SMOKE TEST
   Model  : <model-name>
   Pricing: $X/1M in · $Y/1M out · $Z/1M cache-read
   Scenario: <description>
==============================================================

──────────────────────────────────────────────────────────────
📥 BEFORE  (raw, no optimization)
──────────────────────────────────────────────────────────────
   Messages          : N  (incl. system)
   Prompt tokens     : XX,XXX
   ↳ tool output     : XX,XXX  (XX% of total)
   ↳ system prompt   : XX
   Est. cost / call  : $X.XXXXX

──────────────────────────────────────────────────────────────
📤 AFTER   (headroom SmartCrusher — Strategy 0)
──────────────────────────────────────────────────────────────
   Messages          : N  (incl. system)
   Prompt tokens     : X,XXX
   ↳ tool output     : X,XXX  (XX% of total)
   Est. cost / call  : $X.XXXXX
   Compression time  : X.XX ms
   Rows kept         : XX/XXX  (XX% removed)  |  errors always kept: XX

==============================================================
📊 DELTA
==============================================================
   Tokens saved            : XX,XXX  (XX.X%)
   Cost saved / call       : $X.XXXXX
   Cost saved / 1,000 calls: $XX.XX
   Cost saved /10,000 calls: $XXX.XX
```

Key formatting rules:
- Use **model's actual pricing** — not GPT-4o pricing if user is on Claude
- Count system prompt as **message 1** → `Messages: 5 (incl. system)`
- Label is `Prompt tokens` not `Total tokens`
- Rows format: `kept/original` not `original → kept`

---

## Quick-Reference Checklist

Run this mental checklist before deploying any LLM feature:

- [ ] **headroom installed?** For any app with tool calls, RAG, or log output, `pip install "headroom-ai[all]"` and wrap the client or start the proxy — typically yields 60–95% savings with zero accuracy loss.
- [ ] Measured baseline token usage per request type (`response.usage`, `headroom perf`)
- [ ] Chose the cheapest model tier that passes evals
- [ ] System prompt is ≤ 500 tokens (unless prompt caching covers the excess)
- [ ] Long static context is marked for prompt caching (or headroom CacheAligner handles it)
- [ ] History trimming / summarization in place for multi-turn chat (or headroom Rolling Window)
- [ ] `max_tokens` set per task (not left at API default)
- [ ] Tool schemas pruned to only relevant tools
- [ ] Extraction tasks use JSON/structured output
- [ ] Repeated queries go through semantic cache
- [ ] Bulk offline tasks use the Batch API

---

## Smoke Test Report Format

When producing a token cost smoke test report, always use this exact format:

- **Header emoji:** `🧪` for test header, `📥` BEFORE, `📤` AFTER, `📊` DELTA, `🗄️` caching strategy, `🏷️` model tiering, `✅` checklist
- **Label:** `Prompt tokens` (NOT `Total tokens`)
- **Messages count:** include system as message 1 → `Messages: N  (incl. system)`
- **Compression rows:** `Rows kept: N/M  (X% removed)  |  errors always kept: N`
- **Model/pricing:** ALWAYS use the user's actual active model and Anthropic pricing — never default to GPT-4o or OpenAI pricing
- **Indent:** 3 spaces inside each section block

Example BEFORE block:
```
📥 BEFORE  (raw, no optimization)
──────────────────────────────────────────────────────────────
   Messages          : 5  (incl. system)
   Prompt tokens     : 24,186
   ↳ tool output     : 23,665  (98% of total)
   ↳ system prompt   : 24
   Est. cost / call  : $0.07256  (@ $3.0/1M in)
```

---

## Strategy 11 — Measure Actual Spend from the Hermes Session DB

When running through Hermes, every session's token counts are written to a local
SQLite DB. Query it directly to see exactly how much you spent today, last 7
days, or last 30 days — no API key or dashboard required.

**DB location:**
- Windows: `%LOCALAPPDATA%\hermes\state.db`
- Linux/Mac: `~/.local/share/hermes/state.db`

**Key columns in `sessions`:** `input_tokens`, `output_tokens`,
`cache_read_tokens`, `cache_write_tokens`, `model`, `started_at`.

**⚠️ Copilot billing caveat:** when `billing_provider = 'copilot'`,
`estimated_cost_usd` is always `0.0`. Compute cost yourself from token counts
using Anthropic API-equivalent pricing — this shows what it *would* cost on the
direct API, not the Copilot subscription fee.

**Cost formula:**
```python
cost = (
    input_tokens        / 1e6 * price_in   +
    output_tokens       / 1e6 * price_out  +
    cache_write_tokens  / 1e6 * price_cw   +
    cache_read_tokens   / 1e6 * price_cr
)
```

Cache-read tokens can reach millions per session (every KV-cache hit is counted)
but are cheap ($0.30/1M). Always include them — omitting understates cost by ~30%.

See `templates/spend_report.py` for a full CLI tool with `--today / --7d / --30d`
flags and a per-model daily bar-chart breakdown.
See `references/hermes-session-db.md` for the full DB schema and query patterns.

---

## Support Files

| File | Purpose |
|------|---------| 
| `templates/smoke_test_token_opt.py` | Runnable BEFORE/AFTER smoke test (tiktoken only, no headroom install needed). Uses emoji headers and Anthropic claude-sonnet-4-5 pricing by default. Copy and run: `python smoke_test_token_opt.py` |
| `templates/cost_count.py` | CLI token counter — counts tokens and estimates cost for any text/JSON conversation. Accepts inline text, file, or stdin. Flags: `-f FILE`, `-m MODEL`, `--system TEXT`, `--list-models`. Requires `tiktoken`. |
| `templates/spend_report.py` | CLI spend reporter — queries Hermes `state.db` for actual session token counts and computes $ cost. Flags: `--today`, `--7d`, `--30d`, `--all`, `--from`/`--to`. No extra deps beyond stdlib + sqlite3. |
| `references/headroom-ai.md` | headroom-ai architecture, benchmarks, **confirmed Windows uv-venv install (no MSVC)**, SDK integrations |
| `references/hermes-session-db.md` | Hermes state.db schema reference — cost-relevant columns, copilot billing caveat, cost computation pattern, model-name variations, cache-read token note. |

---

## Common Pitfalls

1. **Not trying headroom first.** For any app that has tool calls, RAG results, or log output in context, headroom is almost always the highest-leverage first move. Install it and wrap your client before hand-tuning anything else.

2. **headroom-ai on Windows — use `uv venv`, NOT bare `pip install`.** Bare `pip install headroom-ai` / `uv pip install headroom-ai --system` tries to compile Rust crates and fails without MSVC. But creating an isolated venv first pulls **pre-built wheels** and succeeds without any build tools:
   ```bash
   uv venv headroom-env --python 3.11
   source headroom-env/Scripts/activate   # bash/git-bash
   # OR: headroom-env\Scripts\activate.bat  (cmd)
   uv pip install "headroom-ai[all]"
   python -c "import headroom; print(headroom.__version__)"  # ✅ 0.23.0
   ```
   Confirmed working on Windows 10 with Python 3.11.15, uv, no MSVC installed.
   Full details: `references/headroom-ai.md`.

6. **`crush_array_json()` inflates tokens — use `crush()` instead.** `crush_array_json()` returns a `dict` with `items`, `ccr_hash`, `dropped_summary`, `strategy_info`, `compacted`, and `compaction_kind` keys. Serialising this dict adds ~50% overhead. Confirmed with real data: 17,062 raw tokens → 25,589 after `crush_array_json()` vs → 10,688 after `crush()`. Always call `.crush(text, query="...")` and use `.compressed` from the result.

7. **LLMLingua-2 cold start (~10–30s, ~2GB RAM).** Only enable `--llmlingua` on the proxy when running persistent long sessions. Don't use it for one-off scripts — the startup overhead negates savings.

8. **Compressing user messages.** headroom never touches user messages for a reason — user intent must be preserved exactly. Don't manually truncate or rewrite user input.

9. **Optimizing before measuring.** You often don't know where tokens actually go until you log `response.usage` or run `headroom perf`. Profile first.

10. **Caching dynamic content.** Prompt caching only helps if the prefix is truly static. If user ID / session info is in the system prompt prefix, the cache never hits. headroom's CacheAligner automatically moves dynamic content to the tail to fix this.

11. **Setting `max_tokens` too low.** If the model hits the limit mid-sentence the output is truncated and often useless. Set it generously for tasks that need complete responses.

12. **Ignoring completion-token cost.** For some providers, completion tokens cost 3-5× more per token than input tokens. A 100-token completion can cost more than a 400-token prompt. Control output length aggressively.

13. **Stripping context that's actually needed.** Over-aggressive trimming causes the model to lose track of key facts, leading to retries that cost more than you saved. headroom's CCR (Compress-Cache-Retrieve) avoids this by storing originals and letting the LLM retrieve them on demand.

14. **Forgetting tool schema tokens.** Each tool in the `tools=` array is tokenized into the prompt. 10 tools × 200 tokens = 2,000 tokens added to every request silently.

15. **Not testing after compression.** Always re-run evals after changing prompts. headroom's `python -m headroom.evals suite` and benchmark table can validate accuracy preservation automatically.

---

## Verification Checklist

- [ ] headroom installed and wrapping LLM client (or proxy running) for any tool/RAG/log-heavy app
- [ ] `headroom perf` run to measure actual savings on real workloads
- [ ] Baseline cost/token logged before optimization
- [ ] Model tier justified by task complexity
- [ ] Prompt caching applied to static prefix (≥1024 tokens for Anthropic; headroom CacheAligner handles alignment automatically)
- [ ] `max_tokens` explicitly set per task
- [ ] Tool schemas filtered to only relevant tools
- [ ] History trimming strategy in place for chat pipelines (headroom Rolling Window or manual)
- [ ] Semantic cache hit rate measured after deployment
- [ ] Evals re-run after prompt compression to confirm no quality regression (`python -m headroom.evals suite --tier 1`)
