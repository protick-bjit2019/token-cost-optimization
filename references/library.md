# Headroom Library & SDK Reference (v0.23.0)

This document contains deep-dive code examples for integrating the Headroom library directly into Python and TypeScript applications.

## Mode 1 — Python Library (SmartCrusher)

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
```

## Mode 1b — HeadroomClient (Drop-in)

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
```

## Mode 1c — Persistent Memory Wrapper

```python
from headroom import with_memory
from openai import OpenAI

client = with_memory(OpenAI(), user_id="alice", session_id="session-42")
# All calls now auto-extract and inject relevant memories
response = client.chat.completions.create(model="gpt-4o", messages=messages)
```

## Mode 1d — Full Message List Compression

```python
from headroom import compress, CompressConfig

result = compress(
    messages,
    model="claude-sonnet-4-5",
    optimize=True,
)
# result.messages — compressed messages list
```

## Mode 1e — Batch Compression (UniversalCompressor)

```python
from headroom.compression import UniversalCompressor

uc = UniversalCompressor()
results = uc.compress_batch([tool_output_1, tool_output_2, log_text])
```

## TransformPipeline

```python
from headroom import TransformPipeline, SmartCrusher
from headroom.transforms import SearchCompressor, LogCompressor

pipeline = TransformPipeline([
    SearchCompressor(),
    LogCompressor(),
    SmartCrusher(),
])
result = pipeline.compress(raw_text, query="deployment error")
```

## Error Handling

```python
from headroom.exceptions import (
    HeadroomError,        # base exception
    CompressionError,     # compressor failure
    ConfigurationError,   # bad config / env var
    CacheError,           # CCR cache / storage failure
)

try:
    result = crusher.crush(tool_output, query=query)
    compressed = result.compressed
except HeadroomError:
    compressed = tool_output   # graceful fallback
```
