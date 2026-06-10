# Headroom Installation Guide

This guide covers the complete setup of `headroom-ai` context compression on Windows using the `uv` package manager and Python 3.11.

## 1. Prerequisites (Install `uv`)

If you don't have `uv` installed yet, run this in PowerShell:
```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

## 2. Install Python 3.11
Ensure you are using Python 3.11 for compatibility:
```bash
uv python install 3.11
```

## 3. Install Headroom
Install Headroom with all optional dependencies (including AST-based code compression):
```bash
uv pip install "headroom-ai[all]"
```

## 4. Run the Proxy
Start the proxy server on the default port (8787). This will download the ModernBERT model on the first run.
```bash
headroom proxy --port 8787
```

*Note: On Windows, if the process hangs during the download, see the troubleshooting section.*

## 5. Configure Hermes Agent
Route your Hermes traffic through the local proxy to enable compression:
```bash
hermes config set model.base_url http://localhost:8787/v1
```

## 6. Verification
Check if the proxy is healthy and responding:
```bash
curl -s http://localhost:8787/livez
```

Expected response:
```json
{"service":"headroom-proxy","status":"healthy","alive":true,"version":"0.23.0",...}
```

## 7. Troubleshooting (Windows)
If port 8787 is occupied or the proxy fails to bind:
1. Check for existing processes: `netstat -ano | findstr :8787`
2. Kill the process: `taskkill /F /PID <PID>`
3. Restart the proxy.
