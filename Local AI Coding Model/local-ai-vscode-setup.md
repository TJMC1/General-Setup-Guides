# Local AI Coding Assistant: Ollama + Continue + VS Code

A complete guide to setting up a multi-model local AI coding assistant on an Apple Silicon Mac using Ollama and the Continue VS Code extension. Designed for privacy, zero subscription cost, and efficient use of unified memory.

---

## Overview

This setup uses three locally-hosted models with distinct roles:

| Role | Model | Purpose | Disk Size |
|---|---|---|---|
| **Chat / Agent** | `qwen3.6:35b-a3b` | Main reasoning, multi-file tasks, architecture | ~20GB |
| **Autocomplete / Edit** | `qwen2.5-coder:7b` | Inline tab completions, fast refactors | ~5.4GB |
| **Summarise / Embed** | `qwen2.5-coder:1.5b` | Context compression, codebase embeddings | ~1GB |
| **Embeddings** | `nomic-embed-text` | Codebase vector search | ~274MB |
| | | **Total disk required** | **~27GB** |

The main model (`qwen3.6:35b-a3b`) is a Mixture-of-Experts (MoE) architecture — it stores 35B parameters but only activates ~3B per token (the "A3B" suffix). This gives near-35B intelligence at the inference speed of a 3B dense model, running at ~45 tok/s on an M2 Max.

---

## Prerequisites

- Apple Silicon Mac (M1/M2/M3/M4 Pro or Max recommended, 32GB+ RAM)
- macOS 13 Ventura or later
- Homebrew installed (`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`)
- VS Code installed
- ~30GB free disk space

---

## Step 1 — Install Ollama

Ollama is the local inference backend. It manages model downloads, quantisation, and serves an OpenAI-compatible REST API on `http://localhost:11434`.

```bash
brew install ollama
```

Start Ollama as a background service so it launches automatically on every boot:

```bash
brew services start ollama
```

Verify it's running:

```bash
ollama --version
curl http://localhost:11434
# Should return: "Ollama is running"
```

> **Note:** The Ollama daemon itself uses ~50–100MB RAM when idle. No models are loaded until a request is made, so it has zero impact on other GPU workloads (video editing, stem separation, etc.) when VS Code is closed.

---

## Step 2 — Pull the Models

Download and quantise all three models. This only needs to be done once — models are cached at `~/.ollama/models/`.

```bash
# Main agent (~20GB download — Q4_K_M quantisation)
ollama pull qwen3.6:35b-a3b

# Autocomplete + edit sub-agent (~5.4GB)
ollama pull qwen2.5-coder:7b

# Summarise + embed sub-agent (~1GB)
ollama pull qwen2.5-coder:1.5b

# Embedding model for codebase search (~274MB)
ollama pull nomic-embed-text
```

Verify all models are present:

```bash
ollama list
```

Run a quick smoke test to confirm the main model loads correctly:

```bash
ollama run qwen3.6:35b-a3b "Say hello in one sentence"
```

---

## Step 3 — Install the Continue Extension

1. Open VS Code
2. Open the Extensions panel (`Cmd+Shift+X`)
3. Search for **Continue**
4. Install the extension by `Continue` (publisher: `Continue`)
5. Reload VS Code when prompted

---

## Step 4 — Configure Continue

Continue's config file lives at `~/.continue/config.yaml`. Open it:

```bash
code ~/.continue/config.yaml
```

Replace the contents with the following:

```yaml
models:
  - name: Qwen3.6 35B A3B (Main)
    provider: ollama
    model: qwen3.6:35b-a3b
    apiBase: http://localhost:11434
    roles:
      - chat
    capabilities:
      - tool_use

  - name: Qwen Coder 7B (Sub-agent)
    provider: ollama
    model: qwen2.5-coder:7b
    apiBase: http://localhost:11434
    roles:
      - autocomplete
      - edit

  - name: Qwen Coder 1.5B (Summarise)
    provider: ollama
    model: qwen2.5-coder:1.5b
    apiBase: http://localhost:11434
    roles:
      - summarize
      - embed

context:
  - provider: code
  - provider: diff
  - provider: terminal
  - provider: problems
  - provider: codebase

tabAutocompleteEnabled: true
```

Save the file. Continue reloads config automatically — no VS Code restart needed.

### How the roles work

| Role | Trigger | Model used |
|---|---|---|
| `chat` | Opening the Continue sidebar panel | Qwen3.6 35B A3B |
| `autocomplete` | Typing code (tab completion fires) | Qwen Coder 7B |
| `edit` | `Cmd+I` inline edit shortcut | Qwen Coder 7B |
| `summarize` | Context window compression on large files | Qwen Coder 1.5B |
| `embed` | Codebase indexing for `@codebase` search | Qwen Coder 1.5B |

The 35B model **only loads when you open the chat panel or use agent mode** — tab completions and quick edits exclusively use the faster, lighter models.

---

## Step 5 — Index Your Codebase (Optional)

For `@codebase` context search (lets you query your entire repo), trigger indexing from the Continue panel:

1. Open the Continue sidebar (`Cmd+L`)
2. Click the settings icon → **Index codebase**
3. Wait for the progress bar to complete

Indexing uses `nomic-embed-text` via the 1.5B embed role and stores vectors locally in `~/.continue/`.

---

## Step 6 — Verify Everything Works

### Test tab autocomplete
Open any code file, start typing a function, and press `Tab` to accept a suggestion. The first suggestion may take 1–2 seconds (cold load), then becomes instant.

### Test chat
Press `Cmd+L` to open the Continue sidebar. Type a question about your code. The 35B model will load on first use (~5–10 seconds cold start), then respond.

### Test inline edit
Select a block of code, press `Cmd+I`, describe the change, and press `Enter`.

### Check model memory usage
```bash
ollama ps
```
This shows which models are currently loaded into GPU memory and their VRAM consumption.

---

## Memory & Resource Behaviour

Understanding how Ollama manages memory is important on a 32GB machine:

- **Ollama daemon**: ~50–100MB RAM, ~0% CPU when idle
- **Models load on demand**: a model enters GPU (Metal) memory only when a request arrives
- **Auto-unload after 5 minutes idle**: models free GPU memory automatically when not in use
- **Maximum concurrent models**: defaults to 3 — all three models can be resident simultaneously if needed

On a 32GB M2 Max, approximate memory usage when all models are loaded:

| Model | GPU RAM |
|---|---|
| qwen3.6:35b-a3b | ~20GB |
| qwen2.5-coder:7b | ~5.4GB |
| qwen2.5-coder:1.5b | ~1GB |
| **Total** | **~26.4GB** |

This leaves ~5.6GB for macOS and other apps. When running heavy GPU workloads (Final Cut, DaVinci Resolve, stem separation), close the Continue chat panel to allow the 35B to unload. The 7B and 1.5B are small enough to stay resident without noticeable impact.

---

## Optional: Tune Keep-Alive

By default, Ollama unloads idle models after 5 minutes. To extend this (reduces cold-start delays during long coding sessions):

```bash
# Keep models warm for 30 minutes of inactivity
launchctl setenv OLLAMA_KEEP_ALIVE 30m

# Restart the service to apply
brew services restart ollama
```

To pin the small sub-agent models in memory indefinitely (only ~6.4GB combined):

```bash
curl http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5-coder:7b", "keep_alive": -1}'

curl http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5-coder:1.5b", "keep_alive": -1}'
```

---

## Useful Ollama Commands

```bash
ollama list                        # List all downloaded models
ollama ps                          # Show currently loaded models + memory usage
ollama run qwen3.6:35b-a3b         # Interactive CLI chat with a model
ollama stop qwen3.6:35b-a3b        # Manually unload a model from memory
ollama rm qwen2.5-coder:7b         # Delete a model to reclaim disk space
du -sh ~/.ollama/models/           # Check total disk usage of all models
```

---

## Adding a Cloud Fallback (Optional)

For tasks requiring higher capability (large repo analysis, complex architecture decisions), add a cloud model alongside local ones. Continue supports mixing providers in the same config:

```yaml
models:
  # ... existing local models above ...

  - name: GLM-5.2 (Cloud Fallback)
    provider: openai
    model: glm-5.2
    apiBase: https://open.bigmodel.cn/api/paas/v4/
    apiKey: YOUR_API_KEY
    roles:
      - chat
```

Switch between models in the Continue sidebar using the model picker dropdown. GLM-5.2 is recommended as a cloud fallback — it performs within 1 point of Claude Opus 4.8 on coding benchmarks at ~5.7× lower cost ($4.40/1M output tokens vs $25/1M).

---

## Troubleshooting

**Tab completions not appearing**
- Check `tabAutocompleteEnabled: true` is in `config.yaml`
- Run `ollama ps` to confirm `qwen2.5-coder:7b` loads when you start typing
- In VS Code settings, search for `continue.enableTabAutocomplete` and ensure it is enabled

**35B model slow to first respond**
- This is normal on cold load (~5–10s). Once warm, responses are fast
- Increase `OLLAMA_KEEP_ALIVE` to reduce frequency of cold starts (see above)

**`@codebase` search returning no results**
- Trigger reindexing from the Continue settings panel
- Ensure `nomic-embed-text` was pulled (`ollama list`)

**Out of memory errors**
- Run `ollama ps` to see what is loaded
- Manually unload unused models: `ollama stop <model>`
- Reduce `OLLAMA_MAX_LOADED_MODELS` to 2 if needed: `launchctl setenv OLLAMA_MAX_LOADED_MODELS 2`

**Config changes not taking effect**
- Continue hot-reloads `config.yaml` on save — no restart needed
- If issues persist, reload VS Code window: `Cmd+Shift+P` → `Developer: Reload Window`
