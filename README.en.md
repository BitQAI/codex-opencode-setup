# Codex × opencode go (deepseek-v4-flash) + mimo-v2.5 Image Recognition — Deployment Guide

[中文](/README.md) | [English](/README.en.md)

Quickly configure Codex on a new machine to use **deepseek-v4-flash via the opencode go subscription** as the primary model, with **mimo-v2.5 multimodal image recognition** (image tasks are automatically described by mimo-v2.5, covering the primary model's vision blind spot).

---

## 1. Architecture Overview

```
Codex Desktop / CLI
  │  config.toml: model_provider="opencode" → https://opencode.ai/zen/go/v1
  │  wire_api = "responses" (native protocol, no middleware)
  ▼
opencode.ai/zen/go/v1  (opencode go subscription, Responses API)
  ├── deepseek-v4-flash  primary model (text only, no image support)
  └── mimo-v2.5          vision model (multimodal text+image)

Image task flow:
User sends image → codex detects primary model lacks image support → replaces with placeholder
→ primary model follows developer_instructions + describe-image skill
→ runs describe_image.py → mimo-v2.5 describes image → text description → answer
```

**Design highlights**:
- **Zero middleware**: opencode.ai natively supports the Responses API (apply_patch / web_search / reasoning all native), no bridge or proxy needed
- **Vision fallback**: primary model explicitly `['text']` modality (cannot see images) → image tasks automatically go through mimo-v2.5
- **Subagents**: `[agents] default_subagent_model = "mimo-v2.5"` — complex image tasks can spawn a mimo-v2.5 subagent
- **Anti-OCR misuse**: the describe-image skill steers the model to mimo-v2.5; layout-ocr/smart-ocr skills are moved away to prevent the model from hacking OCR itself
- **No free models**: Codex only speaks the Responses protocol, but opencode serves free models over Chat Completions only → protocol mismatch, see below

---

## 2. Quick Start (One-Click Script, Recommended)

### Prerequisites

1. Codex installed (CLI or ChatGPT desktop app), and **launched at least once** (creates the `~/.codex` directory)
2. An **API Key** for the opencode go subscription (starts with `sk-`)

### Prerequisites (all platforms)

1. Codex installed (CLI or ChatGPT desktop app), and **launched at least once** (creates the `~/.codex` directory)
2. An **API Key** for the opencode go subscription (starts with `sk-`)
3. **Python 3.11+** installed (the scripts use built-in `tomllib` for validation)

### One-click commands by platform

| Platform | One-click command | Extra requirements |
|---|---|---|
| macOS / Linux | `bash <(curl -fsSL ...)` | none |
| **Windows (recommended)** | `irm https://raw.githubusercontent.com/BitQAI/codex-opencode-setup/main/setup-codex-opencode.ps1 \| iex` (PowerShell) | PowerShell 5.1+ (built into Windows); no Git Bash needed |
| Windows (fallback) | run the bash commands inside **Git Bash** | install [Git for Windows](https://git-scm.com/download/win) |

> The two scripts (`setup-codex-opencode.sh` / `setup-codex-opencode.ps1`) are functionally identical (backup / write / restore) and share the same inlined templates; pick whichever fits your platform.

### macOS / Linux

```bash
# Option 1: curl one-click install (recommended, from GitHub)
bash <(curl -fsSL https://raw.githubusercontent.com/BitQAI/codex-opencode-setup/main/setup-codex-opencode.sh)

# Option 2: local script
bash ~/.codex/opencode-codex-setup/setup-codex-opencode.sh
# or run from this directory
cd ~/.codex/opencode-codex-setup && bash setup-codex-opencode.sh
```

### Windows (PowerShell, recommended -- no Git Bash required)

Open **PowerShell** (search "PowerShell" in Start Menu; built into Windows 10/11) and paste:

```powershell
irm https://raw.githubusercontent.com/BitQAI/codex-opencode-setup/main/setup-codex-opencode.ps1 | iex
```

On first run it asks for your API Key; if an installation record is detected it shows a menu: `1` reinstall/update, `2` restore to the pre-install state.

You can also download it and run locally (handy for reviewing the source / offline):

```powershell
powershell -ExecutionPolicy Bypass -File setup-codex-opencode.ps1
```

### Windows (Git Bash fallback)

If you prefer bash (or cannot use PowerShell), run the macOS/Linux commands inside **Git Bash**. The script auto-detects Windows:
- Uses `%USERPROFILE%\.codex` (i.e. `C:\Users\<you>\.codex`) as the Codex config dir
- Picks the `python` command automatically (skipping the Microsoft Store python stub)
- Generates Windows-native paths for the vision command / skill

### What the script does (identical on all platforms)

1. Back up existing `config.toml` / `models.json` / OCR skills (to `~/.codex/backup-opencode-codex/`)
2. Write `models.json` (full entries for deepseek-v4-flash + mimo-v2.5)
3. Modify `config.toml` (provider / agents / image-handling instructions, **preserving existing MCP, project trust, etc.**)
4. Create the vision script `~/.codex/opencode-bridge-go/describe_image.py`
5. Create the `describe-image` skill (`~/.agents/skills/` + `~/.codex/skills/`)
6. Move away `layout-ocr` / `smart-ocr` skills (backed up, to prevent OCR misuse)
7. Validate TOML/JSON syntax

### Done

**Restart Codex**, then test:
- Normal conversation (deepseek-v4-flash direct)
- Send an image → should be automatically described by mimo-v2.5
- Ask the model to edit files with `apply_patch` → should work natively

---

## 2.5. Why Free Models Are Not Used (Codex protocol limit)

This setup **only uses paid models** (go subscription, `zen/go/v1` endpoint) and provides no free-model option, because:

1. **Codex only speaks the Responses protocol**: Codex removed `wire_api = "chat"` support (see [openai/codex discussion #7782](https://github.com/openai/codex/discussions/7782)); only `"responses"` is accepted
2. **opencode serves free models over Chat Completions only**: `zen/v1/chat/completions` works for free models (hy3-free, nemotron-3-ultra-free, etc.), while `zen/v1/responses` returns 429 or server_error for them — free quota is not on the Responses API
3. → The two protocols are incompatible, so **free models cannot be used inside Codex** (short of a local protocol-translation proxy, which would break this project's zero-middleware design)

**Alternative**: to use free models, connect OpenCode Zen in [opencode](https://opencode.ai)'s own TUI/CLI (it natively supports Chat Completions). Codex continues to use the paid go-subscription models.

---

## 3. Manual Configuration (Alternative)

### 3.1 `~/.codex/models.json`

Write full metadata for the two models (identical to the inlined template in `setup-codex-opencode.sh`, see the `MODELS_JSON_EOF` section):
- `deepseek-v4-flash`: `input_modalities: ["text"]` (**critical**: explicitly cannot see images, triggers vision fallback)
- `mimo-v2.5`: `input_modalities: ["text", "image"]` (vision model)

### 3.2 `~/.codex/config.toml`

```toml
model = "deepseek-v4-flash"
model_provider = "opencode"
model_reasoning_effort = "high"
model_catalog_json = "~/.codex/models.json"

developer_instructions = "IMAGE HANDLING (MANDATORY): when the user provides an image, run python3 ~/.codex/opencode-bridge-go/describe_image.py <path> [question] to get a text description from mimo-v2.5; do not OCR yourself..."

[model_providers.opencode]
name = "opencode go"
base_url = "https://opencode.ai/zen/go/v1"
wire_api = "responses"
experimental_bearer_token = "<your opencode API Key>"

[agents]
default_subagent_model = "mimo-v2.5"
default_subagent_reasoning_effort = "medium"
```

### 3.3 Vision script `~/.codex/opencode-bridge-go/describe_image.py`

See the `DESCRIBE_EOF` section in `setup-codex-opencode.sh` (or copy from an existing machine).

```bash
python3 ~/.codex/opencode-bridge-go/describe_image.py <image_path_or_url> [question]
# output: {"description": "..."}
```

> On Windows (Git Bash) the script lives at `C:\Users\<you>\.codex\opencode-bridge-go\describe_image.py`; use `python` instead of `python3`.

### 3.4 Vision skill `~/.agents/skills/describe-image/SKILL.md` (and `~/.codex/skills/describe-image/`)

See the `SKILL_EOF` section in the script. It makes the model see the "use mimo-v2.5 for images" approach first on image tasks.

### 3.5 Move away OCR skills (optional but recommended)

```bash
mkdir -p ~/.codex/backup-opencode-codex/disabled-skills-ocr
mv ~/.agents/skills/layout-ocr ~/.codex/backup-opencode-codex/disabled-skills-ocr/
mv ~/.agents/skills/smart-ocr ~/.codex/backup-opencode-codex/disabled-skills-ocr/
```

Otherwise the model tends to OCR images itself (poor results) instead of using mimo-v2.5.

---

## 4. How Image Recognition Works

| Stage | Mechanism |
|---|---|
| Primary model can't see images | `deepseek-v4-flash` `input_modalities: ["text"]` → codex replaces images with a placeholder |
| Model knows what to do | `developer_instructions` (developer message) + `describe-image` skill (system injection) |
| Actual recognition | `describe_image.py` calls **mimo-v2.5** on opencode → returns text description |
| Complex image tasks | spawn a **mimo-v2.5 subagent** (`[agents] default_subagent_model`) |
| Search | native `web_search` (executed on the opencode server) |
| File editing | native `apply_patch` (custom_tool_call) |

---

## 5. Verification

```bash
# 1. Config syntax
python3 -c "import tomllib; tomllib.load(open('$HOME/.codex/config.toml','rb')); print('config OK')"
python3 -c "import json; json.load(open('$HOME/.codex/models.json')); print('models OK')"

# 2. Vision script
python3 ~/.codex/opencode-bridge-go/describe_image.py /path/to/image.png

# 3. Direct opencode connection (optional, confirm the key works)
curl -s -X POST "https://opencode.ai/zen/go/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <key>" \
  -d '{"model":"deepseek-v4-flash","input":"hi","stream":false}'
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Model still OCRs after restart | skill not injected / codex not restarted | Confirm Codex restarted; check `~/.agents/skills/describe-image/` exists |
| Image path not found | clipboard images live under `/var/folders` temp | developer_instructions covers this; model uses `find` to locate |
| apply_patch errors | should be native when direct | Confirm `wire_api="responses"` (not a bridge address) |
| reasoning 400 after multi-turn | very long session (300k+ tokens) | Start a new session; or lower `auto_compact_token_limit` |
| 403 (Cloudflare) | request User-Agent blocked | Use a browser UA (script already does); avoid bare urllib default UA |

---

## 7. Restore / Roll Back to Native Codex

**One-click restore** (restores to the pre-install state, including config.toml / models.json / OCR skills):

```bash
# macOS / Linux (or Windows Git Bash)
bash ~/.codex/opencode-codex-setup/setup-codex-opencode.sh --restore
# or remote: bash <(curl -fsSL https://raw.githubusercontent.com/BitQAI/codex-opencode-setup/main/setup-codex-opencode.sh) --restore
```

```powershell
# Windows (PowerShell): rerun the script and pick menu item 2, or pass -Restore
irm https://raw.githubusercontent.com/BitQAI/codex-opencode-setup/main/setup-codex-opencode.ps1 | iex   # then pick 2
powershell -ExecutionPolicy Bypass -File setup-codex-opencode.ps1 -Restore
```

The script uses the `manifest.txt` recorded at install time to decide intelligently:
- `config.toml` existed before → restore from backup; didn't exist → delete
- `models.json` existed before → restore from backup; didn't exist → delete
- Restore `layout-ocr` / `smart-ocr` skills
- Delete the `describe-image` skill and `describe_image.py`

**Manual restore** (backups in `~/.codex/backup-opencode-codex/`):
```bash
cp ~/.codex/backup-opencode-codex/config.toml ~/.codex/config.toml
cp ~/.codex/backup-opencode-codex/models.json ~/.codex/models.json
mv ~/.codex/backup-opencode-codex/disabled-skills-ocr/layout-ocr ~/.agents/skills/
mv ~/.codex/backup-opencode-codex/disabled-skills-ocr/smart-ocr ~/.agents/skills/
rm -rf ~/.agents/skills/describe-image ~/.codex/skills/describe-image
```

> Restart Codex after restoring. If you only want to switch the primary model (without changing the architecture), just edit the `model` field in `config.toml`.

---

## 8. Security Notes

- **API Key**: written into `experimental_bearer_token` in `config.toml`; consider `chmod 600 ~/.codex/config.toml`
- **Recognized content**: `describe_image.py` sends the image (base64) to opencode.ai — be aware of sensitive images
- **No hardcoded keys in the script**: the API key is entered at runtime (or read from `.env`), never baked into the script
