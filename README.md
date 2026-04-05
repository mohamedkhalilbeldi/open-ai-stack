# Build Your Own Local AI Chat Stack

A step-by-step tutorial for building a self-hosted AI chat stack using **LiteLLM** as a unified model proxy and **Open WebUI** as the chat interface. By the end you will have a running system that can talk to any AI provider — GitHub Copilot, OpenAI, Anthropic, Groq, Ollama, and more — through a single endpoint.

---

## Table of Contents

1. [How It Works](#1-how-it-works)
2. [Prerequisites](#2-prerequisites)
3. [Project Structure](#3-project-structure)
4. [Step 1 — Write the Docker Compose file](#step-1--write-the-docker-compose-file)
5. [Step 2 — Configure LiteLLM](#step-2--configure-litellm)
6. [Step 3 — Set environment variables](#step-3--set-environment-variables)
7. [Step 4 — Start the stack](#step-4--start-the-stack)
8. [Step 5 — Add your first provider](#step-5--add-your-first-provider)
9. [Provider Cookbook](#provider-cookbook)
10. [Managing Models at Runtime](#managing-models-at-runtime)
11. [Useful Commands](#useful-commands)
12. [Troubleshooting](#troubleshooting)

---

## 1. How It Works

```
Browser
  └── Open WebUI  :3000          ← chat interface (users interact here)
        └── LiteLLM Proxy  :4000 ← unified OpenAI-compatible API
              ├── GitHub Copilot  (claude-sonnet-4.6, gpt-5.4, grok …)
              ├── OpenAI          (gpt-4o, o3 …)
              ├── Anthropic       (claude-opus-4 …)
              ├── Groq            (llama-3.3-70b … free tier)
              ├── Ollama          (local models, no API key)
              └── PostgreSQL      (spend tracking, keys, logs)
```

**LiteLLM** is the key piece. It acts as a router: you point Open WebUI (or any OpenAI-compatible client) at LiteLLM, and LiteLLM forwards the request to whichever real provider you configured. You configure providers once in `litellm-config.yaml`. Everything else — retries, load balancing, spend limits — is handled automatically.

**Open WebUI** is the chat front-end. It only knows it is talking to an OpenAI-compatible API. Swap providers or add models without touching the UI.

---

## 2. Prerequisites

- **Docker** and **Docker Compose** installed
- At least one AI provider account (see [Provider Cookbook](#provider-cookbook))

---

## 3. Project Structure

```
.
├── docker-compose.yml      # Runs the three services
├── litellm-config.yaml     # All model/provider definitions live here
├── .env                    # Your secrets — never commit this
└── .env.example            # Safe template to copy from
```

---

## Step 1 — Write the Docker Compose file

The stack has three services: **PostgreSQL** (LiteLLM's database), **LiteLLM** (the proxy), and **Open WebUI** (the chat UI).

```yaml
# docker-compose.yml
services:

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: litellm
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 10

  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    restart: unless-stopped
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/litellm
      STORE_MODEL_IN_DB: "True"
    env_file:
      - .env
    volumes:
      - ./litellm-config.yaml:/app/config.yaml:ro
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    depends_on:
      postgres:
        condition: service_healthy

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      OPENAI_API_BASE_URL: http://litellm:4000/v1
      OPENAI_API_KEY: ${LITELLM_MASTER_KEY}
      ENABLE_OLLAMA_API: "false"
      DEFAULT_MODELS: "gpt-5-mini"   # model selected on first load
    volumes:
      - openwebui_data:/app/backend/data
    depends_on:
      - litellm

volumes:
  postgres_data:
  openwebui_data:
```

Key points:
- `env_file: .env` injects all your API keys into the LiteLLM container.
- `OPENAI_API_BASE_URL` points Open WebUI at LiteLLM, not at OpenAI directly.
- `DEFAULT_MODELS` sets which model is pre-selected in new chats.

---

## Step 2 — Configure LiteLLM

`litellm-config.yaml` is where all model definitions live. Each entry has two parts:

- `model_name` — the name shown in Open WebUI and used in API calls
- `litellm_params` — how LiteLLM actually calls the provider

```yaml
# litellm-config.yaml
model_list:

  - model_name: my-model-alias       # what you call it
    litellm_params:
      model: provider/model-id       # how LiteLLM calls it
      api_key: os.environ/MY_API_KEY # read from .env

router_settings:
  num_retries: 3
  retry_after: 2

general_settings:
  database_url: "os.environ/DATABASE_URL"
  store_model_in_db: true
  master_key: "os.environ/LITELLM_MASTER_KEY"
```

The `model` field follows the pattern `provider/model-id`. The full list of supported providers is at https://docs.litellm.ai/docs/providers.

---

## Step 3 — Set environment variables

```bash
cp .env.example .env
```

At minimum you need:

```env
# .env

# Postgres — any strong password
POSTGRES_PASSWORD=change_me

# LiteLLM master key — used as the API key by Open WebUI
# Must start with sk-
LITELLM_MASTER_KEY=sk-change_me

# Your provider API keys (add only the ones you use)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GROQ_API_KEY=gsk_...
GITHUB_COPILOT_TOKEN=ghu_...
```

Environment variables are referenced in `litellm-config.yaml` as `os.environ/VAR_NAME`. LiteLLM reads them at startup — no hardcoded secrets in config files.

---

## Step 4 — Start the stack

```bash
docker compose up -d
```

Check everything is running:

```bash
docker compose ps
```

Open **http://localhost:3000** — the Open WebUI login page should appear within ~30 seconds.

Verify LiteLLM is healthy:

```bash
curl http://localhost:4000/health/liveliness
# → {"status": "healthy"}
```

---

## Step 5 — Add your first provider

Adding any provider follows the same three-step pattern:

### Pattern

**1. Get an API key** from the provider's website.

**2. Add the key to `.env`:**
```env
MY_PROVIDER_API_KEY=your_key_here
```

**3. Add the model to `litellm-config.yaml`:**
```yaml
- model_name: my-model
  litellm_params:
    model: provider/model-id
    api_key: os.environ/MY_PROVIDER_API_KEY
```

**4. Restart LiteLLM:**
```bash
docker compose restart litellm
```

That's it. The model will appear in Open WebUI immediately.

---

## Provider Cookbook

Ready-to-paste blocks for the most common providers. Copy the relevant section into your `litellm-config.yaml` and add the API key to `.env`.

---

### GitHub Copilot

Requires an active Copilot Individual, Business, or Enterprise subscription. No extra cost per token — usage is included in the subscription.

**Get the token** (must use the device flow — a regular PAT will not work):

```bash
python3 -c "
import urllib.request, json, time
CLIENT_ID = 'Iv1.b507a08c87ecfe98'
req = urllib.request.Request('https://github.com/login/device/code',
    data=json.dumps({'client_id': CLIENT_ID, 'scope': 'read:user'}).encode(),
    headers={'Accept': 'application/json', 'Content-Type': 'application/json'})
resp = json.loads(urllib.request.urlopen(req).read())
print(f'Go to:  {resp[\"verification_uri\"]}')
print(f'Code:   {resp[\"user_code\"]}')
for _ in range(24):
    time.sleep(5)
    r = json.loads(urllib.request.urlopen(urllib.request.Request(
        'https://github.com/login/oauth/access_token',
        data=json.dumps({'client_id': CLIENT_ID, 'device_code': resp['device_code'],
            'grant_type': 'urn:ietf:params:oauth:grant-type:device_code'}).encode(),
        headers={'Accept': 'application/json', 'Content-Type': 'application/json'})).read())
    if 'access_token' in r: print('Token:', r['access_token']); break
"
```

**`.env`:**
```env
GITHUB_COPILOT_TOKEN=ghu_your_token_here
```

**`litellm-config.yaml`:**
```yaml
- model_name: claude-sonnet-4.6
  litellm_params:
    model: github_copilot/claude-sonnet-4.6
    api_key: os.environ/GITHUB_COPILOT_TOKEN

- model_name: gpt-5-mini
  litellm_params:
    model: github_copilot/gpt-5-mini
    api_key: os.environ/GITHUB_COPILOT_TOKEN

- model_name: gpt-5.4
  litellm_params:
    model: github_copilot/gpt-5.4
    api_key: os.environ/GITHUB_COPILOT_TOKEN

- model_name: gpt-5.3-codex
  litellm_params:
    model: github_copilot/gpt-5.3-codex
    api_key: os.environ/GITHUB_COPILOT_TOKEN

- model_name: grok-code-fast-1
  litellm_params:
    model: github_copilot/grok-code-fast-1
    api_key: os.environ/GITHUB_COPILOT_TOKEN
```

List all model IDs available on your specific account:

```bash
docker compose exec litellm python3 -c "
import json
from litellm.llms.github_copilot.authenticator import Authenticator
import urllib.request
a = Authenticator()
token = a.get_api_key()
api_base = a.get_api_base() or 'https://api.githubcopilot.com'
req = urllib.request.Request(f'{api_base}/models', headers={
    'Authorization': f'Bearer {token}',
    'editor-version': 'vscode/1.85.1',
    'editor-plugin-version': 'copilot/1.155.0',
    'user-agent': 'GithubCopilot/1.155.0',
    'Copilot-Integration-Id': 'vscode-chat'})
resp = json.loads(urllib.request.urlopen(req).read())
for m in resp.get('data', []): print(m['id'], '-', m.get('name',''))
"
```

> **Token expiry:** `ghu_` tokens can expire. If LiteLLM logs a device auth prompt on startup, authorize the code then run:
> ```bash
> NEW=$(docker compose exec litellm cat /root/.config/litellm/github_copilot/access-token | tr -d '[:space:]') \
>   && sed -i '' "s|^GITHUB_COPILOT_TOKEN=.*|GITHUB_COPILOT_TOKEN=${NEW}|" .env
> ```

---

### OpenAI

API keys at https://platform.openai.com/api-keys

**`.env`:**
```env
OPENAI_API_KEY=sk-...
```

**`litellm-config.yaml`:**
```yaml
- model_name: gpt-4o
  litellm_params:
    model: openai/gpt-4o
    api_key: os.environ/OPENAI_API_KEY

- model_name: gpt-4o-mini
  litellm_params:
    model: openai/gpt-4o-mini
    api_key: os.environ/OPENAI_API_KEY

- model_name: o3
  litellm_params:
    model: openai/o3
    api_key: os.environ/OPENAI_API_KEY
```

---

### Anthropic

API keys at https://console.anthropic.com

**`.env`:**
```env
ANTHROPIC_API_KEY=sk-ant-...
```

**`litellm-config.yaml`:**
```yaml
- model_name: claude-opus-4
  litellm_params:
    model: anthropic/claude-opus-4-5
    api_key: os.environ/ANTHROPIC_API_KEY

- model_name: claude-sonnet-4
  litellm_params:
    model: anthropic/claude-sonnet-4-5
    api_key: os.environ/ANTHROPIC_API_KEY
```

---

### Groq — free tier

Fast inference with a generous free tier. API keys at https://console.groq.com

**`.env`:**
```env
GROQ_API_KEY=gsk_...
```

**`litellm-config.yaml`:**
```yaml
- model_name: llama-3.3-70b
  litellm_params:
    model: groq/llama-3.3-70b-versatile
    api_key: os.environ/GROQ_API_KEY

- model_name: gemma-2-9b
  litellm_params:
    model: groq/gemma2-9b-it
    api_key: os.environ/GROQ_API_KEY
```

---

### Google Gemini — free tier

Free tier available. API keys at https://aistudio.google.com/app/apikey

**`.env`:**
```env
GEMINI_API_KEY=AIza...
```

**`litellm-config.yaml`:**
```yaml
- model_name: gemini-2.0-flash
  litellm_params:
    model: gemini/gemini-2.0-flash
    api_key: os.environ/GEMINI_API_KEY

- model_name: gemini-2.5-pro
  litellm_params:
    model: gemini/gemini-2.5-pro
    api_key: os.environ/GEMINI_API_KEY
```

---

### Ollama — local models, no API key

Run open-source models locally. Install Ollama from https://ollama.com, then pull a model:

```bash
ollama pull llama3.2
ollama pull qwen2.5-coder
```

Add the `ollama` service to `docker-compose.yml` or point at the host machine:

**`litellm-config.yaml`:**
```yaml
- model_name: llama3.2
  litellm_params:
    model: ollama/llama3.2
    api_base: http://host.docker.internal:11434  # macOS/Windows
    # api_base: http://172.17.0.1:11434          # Linux

- model_name: qwen2.5-coder
  litellm_params:
    model: ollama/qwen2.5-coder
    api_base: http://host.docker.internal:11434
```

No API key needed — remove `ENABLE_OLLAMA_API: "false"` from the Open WebUI service in `docker-compose.yml` if you want Ollama to also show in the Ollama section of the UI.

---

### OpenRouter — hundreds of models, many free

OpenRouter aggregates 200+ models. Free models are marked `:free`. API keys at https://openrouter.ai/keys

**`.env`:**
```env
OPENROUTER_API_KEY=sk-or-...
```

**`litellm-config.yaml`:**
```yaml
# Free models
- model_name: deepseek-r1-free
  litellm_params:
    model: openrouter/deepseek/deepseek-r1:free
    api_key: os.environ/OPENROUTER_API_KEY

- model_name: llama-4-scout-free
  litellm_params:
    model: openrouter/meta-llama/llama-4-scout:free
    api_key: os.environ/OPENROUTER_API_KEY

# Paid models
- model_name: deepseek-r1
  litellm_params:
    model: openrouter/deepseek/deepseek-r1
    api_key: os.environ/OPENROUTER_API_KEY
```

---

### Azure OpenAI

**`.env`:**
```env
AZURE_API_KEY=...
AZURE_API_BASE=https://your-resource.openai.azure.com
AZURE_API_VERSION=2024-02-15-preview
```

**`litellm-config.yaml`:**
```yaml
- model_name: azure-gpt-4o
  litellm_params:
    model: azure/your-deployment-name
    api_key: os.environ/AZURE_API_KEY
    api_base: os.environ/AZURE_API_BASE
    api_version: os.environ/AZURE_API_VERSION
```

---

## Managing Models at Runtime

### Add a model without restarting

Use the LiteLLM Admin UI at **http://localhost:4000/ui** (log in with `LITELLM_MASTER_KEY`) to add models live. Changes are saved to PostgreSQL.

Or via the API:

```bash
source .env
curl -X POST http://localhost:4000/model/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model_name": "my-new-model",
    "litellm_params": {
      "model": "groq/llama-3.3-70b-versatile",
      "api_key": "gsk_..."
    }
  }'
```

### List active models

```bash
source .env
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" | python3 -m json.tool
```

### Test a model

```bash
source .env
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-5-mini","messages":[{"role":"user","content":"hello"}],"max_tokens":20}'
```

### Set the default model in Open WebUI

Edit `docker-compose.yml`:

```yaml
environment:
  DEFAULT_MODELS: "gpt-5-mini"   # change to any model_name from your config
```

Then: `docker compose restart open-webui`

---

## Useful Commands

```bash
# Start the stack
docker compose up -d

# Stop the stack (keeps volumes)
docker compose down

# Stop and delete all data
docker compose down -v

# Restart a single service after config change
docker compose restart litellm

# Stream logs
docker compose logs litellm -f
docker compose logs open-webui -f

# Check service health
docker compose ps
```

---

## Troubleshooting

### Model returns "not supported" error

The model ID sent to the provider is wrong. Check the exact ID the provider expects:

```bash
# For GitHub Copilot — list your account's available models
docker compose exec litellm python3 -c "
import json
from litellm.llms.github_copilot.authenticator import Authenticator
import urllib.request
a = Authenticator()
token = a.get_api_key()
api_base = a.get_api_base() or 'https://api.githubcopilot.com'
req = urllib.request.Request(f'{api_base}/models', headers={
    'Authorization': f'Bearer {token}',
    'editor-version': 'vscode/1.85.1', 'editor-plugin-version': 'copilot/1.155.0',
    'user-agent': 'GithubCopilot/1.155.0', 'Copilot-Integration-Id': 'vscode-chat'})
resp = json.loads(urllib.request.urlopen(req).read())
for m in resp.get('data', []): print(m['id'])
"
```

A common mistake: GitHub Copilot model IDs use **dots** (`claude-sonnet-4.6`), not dashes (`claude-sonnet-4-6`).

### Models list is empty after restart

Models added via the Admin UI are stored in PostgreSQL. Models in `litellm-config.yaml` are loaded on startup. If both exist, both are served. Check logs for errors:

```bash
docker compose logs litellm 2>&1 | grep -E "ERROR|WARNING|initialized"
```

### Open WebUI shows no models

Verify LiteLLM is reachable from the Open WebUI container:

```bash
docker compose exec open-webui wget -qO- http://litellm:4000/health/liveliness
```

### Authentication error on startup

Your API key is invalid or the provider rejected it. Check the key is correctly set in `.env` and that the variable name in `.env` matches what `litellm-config.yaml` references via `os.environ/VAR_NAME`.
