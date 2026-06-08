# headroom-ai Windows Installation Reference

Source: https://github.com/chopratejas/headroom
Docs structure: `docs/content/docs/*.mdx` (not `.md`)

## What headroom-ai does
Compress tool outputs, logs, files, and RAG chunks before they reach
the LLM. Real benchmarks from the README:

| Workload              | Before   | After   | Savings |
|-----------------------|----------|---------|---------|
| Code search 100 res.  | 17,765 t | 1,408 t | 92%     |
| SRE incident debug    | 65,694 t | 5,118 t | 92%     |
| GitHub issue triage   | 54,174 t | 14,761 t| 73%     |
| Codebase exploration  | 78,502 t | 41,254 t| 47%     |

Accuracy (unchanged after compression):
- GSM8K: ±0.000, TruthfulQA: +0.030, SQuAD v2: 97%, BFCL: 97%
  (at 19–32% compression levels)

## Three usage modes

### 1. Library (inline)
```python
from headroom import compress
result = compress(messages, model="gpt-4o")
# result.messages   → compressed message list
# result.savings    → dict with token/cost breakdown
# result.ccr_id     → ID for reversible retrieval (ccr_retrieve)
```

### 2. Proxy (zero code change)
```bash
headroom proxy --port 8787
export OPENAI_BASE_URL=http://localhost:8787/v1
# all LLM calls now transparently compressed
```

### 3. Agent wrap (CLI agents)
```bash
headroom wrap claude
headroom wrap codex
headroom wrap cursor
headroom wrap aider
headroom wrap copilot
```

## Internal pipeline stages (architecture.mdx)
1. **CacheAligner** — normalise + deduplicate repeated content (prompt caching boost)
2. **ContentRouter** — detect content type (JSON array, logs, code, markdown, prose)
3. **SmartCrusher** — statistical sampling for JSON arrays / log streams
   - 30% from start (schema / warm-up context)
   - 15% from end   (recency)
   - 55% by importance score (anomaly, error, keyword)
   - ERROR / CRITICAL / FATAL rows: **always kept, never dropped**
4. **CodeCompressor** — remove dead branches, keep comments, strip whitespace
5. **Kompress-base** — general prose compression (fine-tuned LM)
6. **ContextManager** — final rolling-window budget enforcement

## CCR — Content-Compressed Representation
Reversible compression: a `ccr_id` is returned and `ccr_retrieve(id)` fetches
the original. Useful when the downstream LLM needs to cite verbatim content.

## Content-type savings (from architecture.mdx)
| Content type | Typical savings |
|---|---|
| JSON arrays / search results | 85–95% |
| Log streams | 80–95% |
| Source code | 40–70% |
| Markdown docs | 30–60% |
| Prose / chat history | 10–25% |

**Never compressed:** user messages, system prompts, function/tool schema.

## Windows installation failure — MSVC C++ build tools required

headroom-ai uses Rust crates via Maturin/pyo3 (`quote`, `serde_core`,
`proc-macro2`). On Windows, `pip install headroom-ai` **fails** unless
Microsoft Visual C++ Build Tools are installed.

### Error symptom
```
error: linking with `link.exe` failed: exit code: 1
note: `link.exe` returned an unexpected error
note: in the Visual Studio installer, ensure the "C++ build tools" workload is selected
```
The error surfaces from `serde_core`, `proc-macro2`, or `quote` build steps.
`uv pip install` and `pip install --no-deps` both fail identically — the
build step is unavoidable.

### Fix
1. Install **Visual Studio Build Tools 2022** (free):
   https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022
2. In the installer, select the **"Desktop development with C++"** workload
   (this includes `link.exe` and the Windows SDK).
3. After install, open a new terminal and retry:
   ```bash
   pip install headroom-ai
   # or
   uv pip install headroom-ai
   ```

### Workaround (no MSVC available)
Use the inline SmartCrusher implementation in
`templates/smoke_test_token_opt.py` — it reproduces the core algorithm
using only stdlib + tiktoken.
The proxy mode (`headroom proxy`) may also work if headroom can be
installed on a separate Linux/Mac machine and exposed on the network.

## SDK integrations (from README)
- LangChain: `from headroom.integrations.langchain import HeadroomCallbackHandler`
- LlamaIndex: `from headroom.integrations.llama_index import HeadroomObserver`
- CrewAI: `from headroom.integrations.crewai import HeadroomMiddleware`
- Direct OpenAI: override `client.base_url = "http://localhost:8787/v1"`

## Provider prompt-cache alignment (CacheAligner detail)
Headroom pre-normalises content so that the prefix fed to the provider
matches the provider's caching hash. Concretely:
- Anthropic: guarantees ≥ 1024 tokens of stable prefix
- OpenAI: guarantees ≥ 128-token cacheable prefix
- This stacks with headroom compression — cache hits AND fewer tokens billed.
