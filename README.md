# Persistio Developer Documentation

Persistio is a self-hostable memory layer for AI agents and LLM applications. It ingests raw conversation chunks, asynchronously extracts durable memories, and surfaces them via semantic recall — giving your agents persistent, queryable context across sessions.

---

## Documentation

| Guide | Description |
|-------|-------------|
| [Getting Started](getting-started.md) | Run the server, create a vault, and make your first ingest and recall calls |
| [API Reference](api-reference.md) | Full reference for every endpoint with curl, Node.js, and Python examples |
| [Integration Guide](integration-guide.md) | Deep-dive patterns for building your own integration |

---

## Repositories

| Repo | Description |
|------|-------------|
| [Persistio/server](https://github.com/Persistio/server) | The Persistio server — Docker-ready, self-hostable |
| [Persistio/openclaw-persistio](https://github.com/Persistio/openclaw-persistio) | OpenClaw plugin for drop-in Persistio memory integration |

---

## Hosted Service

A managed instance is available at **[persistio.ai](https://persistio.ai)** — no infrastructure required.

---

## Quick Start

```bash
# 1. Run the server
git clone https://github.com/Persistio/server
cd server && docker compose up -d

# 2. Create a vault
curl -X POST https://your-persistio-instance/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-agent"}'

# 3. Ingest a conversation
curl -X POST https://your-persistio-instance/v1/ingest \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"session_id": "session-001", "chunks": [{"role": "user", "content": "My name is Alice"}]}'

# 4. Recall relevant memories
curl -X POST https://your-persistio-instance/v1/recall \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"query": "who am I", "top_k": 5}'
```

See [Getting Started](getting-started.md) for the full walkthrough.
