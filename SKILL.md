---
name: token-cost-optimization
description: Use when you need to reduce LLM API token usage or costs — prompt compression, model tiering, caching, context trimming, and automated context compression via headroom-ai.
version: 1.5.0
author: protick-bjit2019
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [tokens, cost, llm, optimization, prompting, caching, context, headroom, compression]
    homepage: https://github.com/protick-bjit2019/token-cost-optimization
    related_skills: [andrej-karpathy, plan, hermes-agent]
---

# Token Cost Optimization

## Strategy 0 — headroom-ai: Automatic Compression
[headroom](https://github.com/chopratejas/headroom) compresses tool outputs, logs, and history (60–95% savings) with no accuracy loss.

### Installation & Proxy Mode
```bash
pip install "headroom-ai[all]"
headroom proxy --port 8787   # Start local proxy
```
Point any client at `http://localhost:8787/v1`. SDK/Library details in `references/library.md`.

## Strategy 1 — Terse Prompting
- **Remove Boilerplate:** Delete "You are a helpful assistant" and long preambles.
- **Terse Formatting:** "Analyze feedback. List issues." > "Please analyze this feedback and provide a list."
- **Few-Shot → Zero-Shot:** Test if zero-shot works before adding expensive examples.

## Strategy 2 — Model Tiering
Route to the cheapest tier that passes evals:
- **Fast:** GPT-4o-mini, Claude Haiku, Gemini Flash (Classification, Extraction).
- **Standard:** GPT-4o, Claude Sonnet, Gemini Pro (Coding, Reasoning).
- **Frontier:** o3, Claude Opus (Deep Research).

## Strategy 3 — Context Management
- **Trim History:** Keep only the last ~4k tokens + system prompt.
- **Prune Tools:** `hermes tools disable <name>` to remove schema overhead (~200 tokens/tool).
- **Prompt Caching:** Place static context (docs, system prompt) at the START.
  - Anthropic: Use `"cache_control": {"type": "ephemeral"}`.
  - OpenAI: Caches identical prefixes (1024+ tokens) automatically.

## Strategy 4 — Output Control
- **JSON Mode:** Forces concise, parseable output. 2-5x fewer tokens than prose.
- **Soft Limits:** "Respond in ≤3 sentences."
- **Hard Limits:** Set `max_tokens` per task.

## Common Pitfalls
1. **Tool Bloat:** Carrying 20+ enabled toolsets adds 4k+ tokens to every turn.
2. **Dynamic Prefix:** Putting dates/UUIDs at the start of the system prompt breaks provider caching.
3. **Implicit Over-Engineering:** Optimizing one-off chat where costs are <$0.01.

## Alignment Audit Protocol
If asked for an audit:
1. `skill_view(name='token-cost-optimization')`
2. Run live import tests from `references/library.md`.
3. Report: Verified/Total Imports % + Specific Fixes.
