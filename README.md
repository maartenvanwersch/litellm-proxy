
Here’s a **clean, copy-paste ready README.md** for your setup.
It’s written as if this repo is the source of truth for running **Claude Code → LiteLLM → local Ollama**.

---

# Claude Code with LiteLLM + Local Ollama (Docker Compose)

Run **Claude Code** against a **local LLM** using **LiteLLM** as an Anthropic-compatible gateway and **Ollama** as the model runtime.

This setup uses **Docker Compose** (no custom Dockerfiles, no supervisord).

---

## Architecture

```
Claude Code (host)
        │
        ▼
LiteLLM Proxy (Anthropic /v1/messages)
        │
        ▼
Ollama (local LLMs)
```

* **Claude Code** runs on your host machine
* **LiteLLM** exposes an Anthropic-compatible API
* **Ollama** runs the local models

---

## Requirements

* Docker + Docker Compose (v2)
* Claude Code CLI installed on your host
* macOS, Linux, or Windows (Docker Desktop)

---

## Repository Structure

```
.
├── docker-compose.yml
├── config.yaml
└── .env
```

---

## Configuration

### 1. `.env`

Create a `.env` file:

```bash
LITELLM_MASTER_KEY=sk-change-this-to-a-long-random-value
```

This key is used by LiteLLM and passed to Claude Code.

---

### 2. `config.yaml` (LiteLLM)

```yaml
model_list:
  - model_name: local-main
    litellm_params:
      model: ollama_chat/llama3.1
      api_base: http://ollama:11434

litellm_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

You can swap `llama3.1` for any Ollama-supported model:

* `qwen2.5-coder:7b`
* `mistral`
* `deepseek-coder`
* etc.

---

### 3. `docker-compose.yml`

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    restart: unless-stopped

  litellm:
    image: ghcr.io/berriai/litellm:main
    container_name: litellm
    depends_on:
      - ollama
    ports:
      - "4000:4000"
    environment:
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
    volumes:
      - ./config.yaml:/app/config.yaml:ro
    command: ["--config", "/app/config.yaml", "--host", "0.0.0.0", "--port", "4000"]
    restart: unless-stopped

volumes:
  ollama_data:
```

---

## Startup

Start everything:

```bash
docker compose up -d
```

Pull the model (first run only):

```bash
docker exec -it ollama ollama pull llama3.1
```

Check containers:

```bash
docker compose ps
```

---

## Test the Gateway

LiteLLM exposes an **Anthropic-compatible endpoint**:

```bash
curl http://localhost:4000/v1/messages \
  -H "Authorization: Bearer sk-change-this-to-a-long-random-value" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "local-main",
    "max_tokens": 80,
    "messages": [
      { "role": "user", "content": "Say hi in one sentence." }
    ]
  }'
```

You should get a valid Anthropic-style response.

---

## Use with Claude Code

Run Claude Code **on your host**:

```bash
export ANTHROPIC_BASE_URL="http://localhost:4000"
export ANTHROPIC_AUTH_TOKEN="sk-change-this-to-a-long-random-value"

claude --model local-main
```

Claude Code now talks to your **local LLM** via LiteLLM.

---

## GPU Support (Optional)

### NVIDIA (Linux)

Add this to the `ollama` service:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - capabilities: [gpu]
```

Ensure:

* NVIDIA drivers installed
* `nvidia-container-toolkit` configured

---

## Common Issues

### Model not found

* Ensure the model is pulled:

  ```bash
  docker exec -it ollama ollama list
  ```

### Authentication errors

* `ANTHROPIC_AUTH_TOKEN` must match `LITELLM_MASTER_KEY`

### Slow responses

* Use a smaller model (`qwen2.5-coder:7b`)
* Ensure GPU acceleration is enabled (if available)

---

## Why Docker Compose?

* One process per container
* No custom Dockerfiles
* No supervisors
* Easy to extend (multiple models, logging, auth)

---

## Next Steps

* Add **multiple models** (fast vs smart)
* Enable **request logging**
* Add **rate limiting**
* Share LiteLLM as a **team gateway**

If you want any of those, just ask.
