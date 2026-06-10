---
name: headroom-quickstart
description: Minimal setup for headroom-ai proxy (compression) with uv.
version: 1.0.0
---

# Headroom Quickstart

## 1. Install (Python 3.11)
```bash
uv python install 3.11
uv pip install "headroom-ai[all]"
```

## 2. Start Proxy
```bash
headroom proxy --port 8787
```

## 3. Route Hermes Traffic
```bash
hermes config set model.base_url http://localhost:8787/v1
```

## 4. Verify
```bash
curl -s http://localhost:8787/livez
```