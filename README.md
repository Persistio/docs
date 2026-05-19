# Persistio Developer Documentation

Persistio is a self-hostable memory layer for AI agents and LLM applications. It stores raw conversation chunks, extracts durable memories asynchronously, supports semantic and graph-assisted recall, and exposes vault-scoped APIs for memory management.

Current source alignment:

- Server package: `@persistio/server` `0.1.9`
- OpenClaw plugin package: `@persistio/openclaw-plugin` `0.1.4`
- OpenClaw compatibility: `>=2026.3.24-beta.2`

The public API uses vault Bearer tokens for vault-scoped routes. Admin routes accept `X-Admin-Key` or a Bearer admin key.

---

## Documentation

| Guide | Description |
|-------|-------------|
| [Getting Started](getting-started.md) | Run the server, create a vault, ingest chunks, recall memory, and install the OpenClaw plugin |
| [API Reference](api-reference.md) | Endpoint reference with request fields, response shapes, and examples |
| [Integration Guide](integration-guide.md) | Integration patterns for ingest, recall, memory management, quotas, and plugin usage |
| [Architecture](architecture.md) | Runtime topology, API/worker split, extraction, curation, observability, and security |
| [Data Model](data-model.md) | Vault tenancy, core tables, memory metadata, graph edges, and queues |

---

## Repositories

| Repo | Description |
|------|-------------|
| [chriscoveyduck/persistio](https://github.com/chriscoveyduck/persistio) | Persistio server and package source |
| [Persistio/openclaw-persistio](https://github.com/Persistio/openclaw-persistio) | OpenClaw plugin for drop-in Persistio memory integration |

---

## Hosted Service

A managed instance is available at **[persistio.ai](https://persistio.ai)**.

---

## Quick Start

```bash
# 1. Run the server
git clone https://github.com/chriscoveyduck/persistio
cd persistio
docker compose up -d

# 2. Create a vault
curl -X POST http://localhost:4827/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-agent","purpose":"Personal assistant memory","plan":"free"}'

# 3. Ingest a conversation
curl -X POST http://localhost:4827/v1/ingest \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "session-001",
    "chunks": [
      {
        "role": "user",
        "content": "My name is Alice.",
        "timestamp": "2026-05-19T12:00:00.000Z"
      }
    ]
  }'

# 4. Recall relevant memories
curl -X POST http://localhost:4827/v1/recall \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"query":"who am I?","top_k":5}'
```

See [Getting Started](getting-started.md) for the full walkthrough.
