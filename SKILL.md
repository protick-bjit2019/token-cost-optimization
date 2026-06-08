---
name: token-cost-optimization
description: Use when you need to reduce LLM API token usage or costs — prompt compression, model tiering, caching, context trimming, tool call minimization, output length control, and automated context compression via headroom-ai.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [tokens, cost, llm, optimization, prompting, caching, context, headroom, compression, context-window]
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
pip install "headroom-ai[all]"    # Python — everything included
npm install headroom-ai           # TypeScript / Node (requires proxy)
docker pull ghcr.io/chopratejas/headroom:latest
```

Granular extras: `[proxy]`, `[mcp]`, `[ml]`, `[code]`, `[memory]`, `[image]`, `[langchain]`, `[agno]`, `[evals]`. Requires Python 3.10+.

### Three usage modes

**Mode 1 — Library (minimal code change):**

```python
from headroom import compress

result = compress(messages, model="gpt-4o")

# result.messages is drop-in replacement for original messages
response = client.chat.completions.create(
    model="gpt-4o",
    messages=result.messages,  # compressed
)

# Inspect savings
print(f"Tokens: {result.tokens_before} → {result.tokens_after}  "
      f"({result.compression_ratio:.0%} saved)")
print(f"Transforms applied: {result.transforms_applied}")
```

**Mode 2 — Proxy (zero code changes):**

```bash
# Start local proxy on port 8787
headroom proxy --port 8787

# Point any OpenAI-compatible client at it
OPENAI_BASE_URL=http://localhost:8787/v1 your-app
ANTHROPIC_BASE_URL=http://localhost:8787 claude

# Stats endpoint
curl http://localhost:8787/stats
# {"requests_total": 42, "tokens_saved_total": 125000, ...}
```

**Mode 3 — Agent wrap (one command):**

```bash
headroom wrap claude     # wraps Claude Code with --memory and --code-graph
headroom wrap codex      # shares memory with Claude
headroom wrap cursor     # prints config, paste once
headroom wrap aider      # starts proxy + launches
headroom wrap copilot    # starts proxy + launches
```

### What headroom compresses (and what it doesn't touch)

| Content type | Compressor | Typical savings |
|---|---|---|
| JSON arrays of dicts | SmartCrusher (statistical sampling) | 83–95% |
| JSON arrays of strings | Dedup + adaptive sampling | 60–90% |
| Build/test logs | Pattern clustering | 85–94% |
| Source code | CodeCompressor (AST-aware) | 40–70% |
| Plain text | Kompress-base (HuggingFace model) | 30–50% |
| HTML | Article extraction (trafilatura) | ~95% |
| Short content (<200 tokens) | **Not compressed** (overhead > savings) | — |
| User messages | **Never compressed** (intent preserved) | — |
| System prompt content | **Preserved**; only dynamic parts relocated | — |

### How the pipeline works internally

Three-stage compression pipeline:

1. **CacheAligner** — extracts dynamic content (dates, UUIDs, tokens) from system prompt prefix and moves it to the tail. Stabilizes prefixes so provider KV caches (Anthropic `cache_control`, OpenAI prefix cache) actually hit. Sub-millisecond overhead.

2. **ContentRouter → SmartCrusher / CodeCompressor / Kompress-base** — detects content type, selects best compressor. SmartCrusher applies field-level statistical analysis (variance, uniqueness, Kneedle algorithm on bigram coverage). Errors and anomalies are always kept regardless of budget. Overhead: 1–50ms.

3. **Context Manager** — ensures final message array fits within the model's context window.
   - *Rolling Window* (default): drops oldest messages first, preserving system prompt and recent turns. Tool call + response pairs dropped atomically.
   - *Intelligent Context* (advanced): scores messages on 6 dimensions (recency, semantic similarity, TOIN importance, error indicators, forward references, token density).

**CCR (Compress-Cache-Retrieve) — reversible compression:** originals are never deleted; they're stored in a local SQLite cache. If the LLM needs full data it calls `ccr_retrieve("<hash>")` to get it back. Headroom is the only major compressor that is fully reversible.

### SDK integrations

```python
# Wrap existing client — Anthropic or OpenAI
from headroom.integrations import withHeadroom
from anthropic import Anthropic

client = withHeadroom(Anthropic())   # all calls auto-compressed

# LangChain
from headroom.integrations.langchain import HeadroomChatModel
llm = HeadroomChatModel(your_llm)

# Agno
from headroom.integrations.agno import HeadroomAgnoModel
model = HeadroomAgnoModel(your_model)

# ASGI middleware
from headroom.integrations.asgi import CompressionMiddleware
app.add_middleware(CompressionMiddleware)

# MCP server (for any MCP client)
headroom mcp install
# Exposes: headroom_compress, headroom_retrieve, headroom_stats
```

### Provider cache optimization (built-in)

After compression, headroom automatically applies provider-specific cache hints:

| Provider | Mechanism | Cache savings |
|---|---|---|
| Anthropic | `cache_control` blocks on stable prefix | Up to 90% on repeated tokens |
| OpenAI | Prefix alignment for automatic caching | Up to 50% on repeated tokens |
| Google | `CachedContent` API | Up to 75% on repeated tokens |

### Cross-agent shared memory

```python
from headroom.memory import SharedContext

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

### Performance check

```bash
headroom perf    # benchmark compression on your actual workloads
```

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

## Support Files

| File | Purpose |
|------|---------|
| `templates/smoke_test_token_opt.py` | Runnable BEFORE/AFTER smoke test (tiktoken only, no headroom install needed). Copy and run: `python smoke_test_token_opt.py` |
| `references/headroom-ai.md` | headroom-ai architecture notes, benchmarks, all 3 usage modes, Windows MSVC install fix, SDK integrations, CCR detail |

---

## Common Pitfalls

1. **Not trying headroom first.** For any app that has tool calls, RAG results, or log output in context, headroom is almost always the highest-leverage first move. Install it and wrap your client before hand-tuning anything else.

2. **headroom-ai fails to install on Windows (Rust/MSVC blocker).** The package uses Rust crates (Maturin/pyo3: `quote`, `serde_core`, `proc-macro2`). `pip install headroom-ai` and `uv pip install headroom-ai` both fail with:
   ```
   error: linking with `link.exe` failed: exit code: 1
   note: ensure the "C++ build tools" workload is selected in Visual Studio
   ```
   **Fix:** Install [VS Build Tools 2022](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022) → select **"Desktop development with C++"** workload → re-run install in a new terminal.
   **Workaround (no MSVC):** Use the inline SmartCrusher in `templates/smoke_test_token_opt.py` (stdlib + tiktoken only), or run the headroom proxy on a separate Linux/Mac machine. Full details: `references/headroom-ai.md`.

3. **Compressing user messages.** headroom never touches user messages for a reason — user intent must be preserved exactly. Don't manually truncate or rewrite user input.

3. **Optimizing before measuring.** You often don't know where tokens actually go until you log `response.usage` or run `headroom perf`. Profile first.

4. **Caching dynamic content.** Prompt caching only helps if the prefix is truly static. If user ID / session info is in the system prompt prefix, the cache never hits. headroom's CacheAligner automatically moves dynamic content to the tail to fix this.

5. **Setting `max_tokens` too low.** If the model hits the limit mid-sentence the output is truncated and often useless. Set it generously for tasks that need complete responses.

6. **Ignoring completion-token cost.** For some providers, completion tokens cost 3-5× more per token than input tokens. A 100-token completion can cost more than a 400-token prompt. Control output length aggressively.

7. **Stripping context that's actually needed.** Over-aggressive trimming causes the model to lose track of key facts, leading to retries that cost more than you saved. headroom's CCR (Compress-Cache-Retrieve) avoids this by storing originals and letting the LLM retrieve them on demand.

8. **Forgetting tool schema tokens.** Each tool in the `tools=` array is tokenized into the prompt. 10 tools × 200 tokens = 2,000 tokens added to every request silently.

9. **Not testing after compression.** Always re-run evals after changing prompts. headroom's `python -m headroom.evals suite` and benchmark table can validate accuracy preservation automatically.

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
