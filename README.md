# 🪙 token-cost-optimization

A [Hermes Agent](https://github.com/NousResearch/hermes-agent) skill that encodes battle-tested strategies for cutting LLM API token costs — without degrading output quality.

## 📊 What it does

Automatically applied on every task. Covers:

| Strategy | Typical Savings |
|---|---|
| **Strategy 0** — headroom-ai context compression | **60–95%** |
| **Strategy 1** — Measure first (tiktoken / response.usage) | Baseline |
| **Strategy 2** — Model tiering (haiku vs sonnet vs opus) | Up to 97% combined |
| **Strategy 3** — Prompt compression & boilerplate removal | 10–50% |
| **Strategy 4** — Context window management (sliding window) | Varies |
| **Strategy 5** — Anthropic prompt caching (cache_control) | Up to 90% |
| **Strategy 6** — Semantic caching | Varies |
| **Strategy 7** — Output length control (max_tokens, JSON) | 20–60% |
| **Strategy 8** — Tool call minimization | 10–30% |
| **Strategy 9** — Batch API (50% discount) | 50% |
| **Strategy 10** — Token budget enforcement | Safety net |

## 🧪 Smoke Test Results (claude-sonnet-4.5, SRE scenario)

```
📥 BEFORE  (raw 200-item JSON tool output)
   Prompt tokens : 24,186  |  Est. cost: $0.07256/call

📤 AFTER   (headroom SmartCrusher)
   Prompt tokens :  2,592  |  Est. cost: $0.00778/call
   Compression   :  8.7 ms

📊 DELTA
   Tokens saved  : 21,594  (89.3%)
   Cost saved    : $64.78 per 1,000 calls
                   $647.82 per 10,000 calls

🏷️ Best combo: haiku-3 + headroom = $0.002074/call  (97.1% vs baseline)
🗄️ Prompt caching (2k-token system): $6.00 → $0.61  (90% off, 1000 calls)
```

## 📦 Install

```bash
hermes skills install https://raw.githubusercontent.com/protick-bjit2019/token-cost-optimization/main/SKILL.md
```

Or clone manually:
```bash
mkdir -p ~/.hermes/skills/software-development/token-cost-optimization
curl -o ~/.hermes/skills/software-development/token-cost-optimization/SKILL.md \
  https://raw.githubusercontent.com/protick-bjit2019/token-cost-optimization/main/SKILL.md
```

Then in a Hermes session: `/reload-skills`

## 📁 Files

| File | Purpose |
|---|---|
| `SKILL.md` | Main skill — all 10 strategies, checklists, pitfalls |
| `templates/smoke_test_token_opt.py` | Runnable before/after smoke test (tiktoken only, no headroom install needed) |
| `references/headroom-ai.md` | headroom-ai architecture, benchmarks, Windows install fix |

## ⚠️ headroom-ai on Windows

`pip install headroom-ai` fails on Windows without MSVC build tools (Rust crates).

**Fix:** Install [VS Build Tools 2022](https://visualstudio.microsoft.com/downloads/#build-tools-for-visual-studio-2022) → **"Desktop development with C++"** workload → re-run install.

**Workaround:** The smoke test script implements SmartCrusher inline (stdlib + tiktoken only) — run it without headroom installed.

## 📄 License

MIT — by [protick-bjit2019](https://github.com/protick-bjit2019)

Built with [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research.
