# Obsidian Web Clipper + Ollama: Working Local AI Interpreter

## Why This Exists

The official Obsidian documentation says you can connect Ollama directly
as an interpreter provider. The connection works. The model loads. No
errors. But when you clip a page, the interpreter ignores your natural
language prompts entirely and prints them as raw text in your note:

> Summary: {{"a summary of the page"}}

Instead of summarizing anything, it just echoes the template syntax
back at you. Everything appears to work — until you look at the output.

This is a confirmed bug in the obsidian-clipper repository (Issue #516,
open as of mid-2025, affecting Ollama users). This repo documents the
workaround that actually fixes it.

> **Platform note:** This solution was developed and tested on Linux.
> The core concept — using LiteLLM as an OpenAI-compatible proxy for Ollama —
> should work on macOS and Windows too, but setup details like the systemd
> service file are Linux-specific. Adapt accordingly for your OS.

---

## The Root Cause

Obsidian Web Clipper's interpreter expects an OpenAI-compatible API
at `/v1/chat/completions`. Ollama's native API does not use that format.

Setting up CORS origins correctly (as the official docs instruct) gets
you a working connection — but the request still gets routed incorrectly
underneath. The model receives no real instruction, so nothing gets
interpreted. It just passes the raw prompt through.

> Note: 405 and 404 errors can appear if your Base URL is misconfigured.
> But even with a clean connection and no errors, the prompt-passthrough
> bug still occurs. A successful connection is not the same as a working
> interpreter.

---

## The Solution: LiteLLM as a Proxy

LiteLLM is a lightweight proxy that translates OpenAI-style API calls
into Ollama API calls. The Web Clipper talks to LiteLLM at the correct
endpoint format, LiteLLM talks to Ollama, and the interpreter actually
works.

```
Obsidian Web Clipper
        ↓
   LiteLLM Proxy        ← speaks OpenAI format the clipper expects
        ↓
      Ollama
        ↓
  your local model
```
## Python Compatibility

LiteLLM may not be compatible with the latest Python versions.
If you run into install or runtime errors, consider using pyenv to
manage a specific version — Python 3.11 or 3.12 is recommended.

pyenv: https://github.com/pyenv/pyenv

Note: pyenv is primarily a Linux/macOS tool. Windows users may need
a different approach — this has only been tested on Linux.
---

## Requirements

- [Ollama](https://ollama.ai) installed and running with at least one model
- [LiteLLM](https://github.com/BerriAI/litellm) installed (`pip install litellm`)
- Obsidian with the Web Clipper browser extension

---

## Setup

### 1. Verify your model is available

```bash
ollama list
```

### 2. Start the LiteLLM proxy

Replace `your-model-name` with whatever model you have installed in Ollama:

```bash
litellm --model ollama/your-model-name --port 4000 --host 127.0.0.1
```

The `ollama/` prefix is required — it tells LiteLLM which backend to route to.

> The author tested this successfully with `qwen2.5:7b-instruct`. Other
> Ollama models should work the same way.

Confirm it's running — you should see:

```
INFO: Uvicorn running on http://127.0.0.1:4000
```

### 3. Configure Web Clipper Interpreter settings

Do NOT use the built-in Ollama provider option. Use **OpenAI** as the
provider type and point it at your LiteLLM address instead.

| Field      | Value                                         |
|------------|-----------------------------------------------|
| Provider   | OpenAI                                        |
| Base URL   | `http://127.0.0.1:4000/v1`                    |
| API Key    | `anything` (required by UI, ignored locally)  |
| Model name | `ollama/your-model-name`                      |

### 4. Confirm it's working

In the LiteLLM terminal, a successful clip shows:

```
POST /v1/chat/completions   200 OK
```

Your note will now contain an actual summary instead of raw prompt text.

---

## Template Example

```
{{title}}
Source: {{url}}

## Summary
{{"Summarize the following article in 5 concise bullet points:\n\n" + content}}

---
{{content|strip_tags}}
```

> Important: your prompt must reference `content` — otherwise the model
> has no page text to work with and will respond generically.

---

## Autostart LiteLLM on Boot (Linux/systemd)

A `litellm.service` file is included in this repo.

First find your LiteLLM install path:

```bash
which litellm
```

Edit the `ExecStart` line in the service file to match your path and
model name, then run:

```bash
sudo cp litellm.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable litellm
sudo systemctl start litellm
```

---

## Tested On

- OS: Linux
- Browser: Vivaldi (any Chromium-based or Firefox browser should work)
- Model: qwen2.5:7b-instruct via Ollama (example — other models should work)
- Python: managed via pyenv

---

## Related

- [obsidian-clipper Issue #516](https://github.com/obsidianmd/obsidian-clipper/issues/516) — confirmed bug: interpreter outputs raw prompt text instead of processing it
- [LiteLLM](https://github.com/BerriAI/litellm)
- [Ollama](https://github.com/ollama/ollama)
The errors you'll see in a direct connection attempt:
