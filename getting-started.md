# Getting Started

## Prerequisites

- Docker and Docker Compose
- `curl` or another HTTP client
- A configured `ADMIN_API_KEY`
- Embedding and extraction providers configured in `.env`

## Run The Server

```bash
git clone https://github.com/Persistio/server.git persistio-server
cd persistio-server
cp .env.example .env
docker compose up -d
```

The server listens on `http://localhost:4827` by default.

```bash
curl http://localhost:4827/health
```

## Create A Vault

```bash
curl -X POST http://localhost:4827/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-agent","purpose":"Long-term memory for my assistant"}'
```

Save the returned `api_key`. Use it as a Bearer token for vault-scoped routes.

## Ingest A Conversation

```bash
curl -X POST http://localhost:4827/v1/ingest \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "session-2026-001",
    "chunks": [
      {
        "role": "user",
        "content": "My name is Alice and I prefer concise answers.",
        "timestamp": "2026-05-19T12:00:00.000Z"
      }
    ]
  }'
```

Extraction happens asynchronously after ingest.

## Recall Memories

```bash
curl -X POST http://localhost:4827/v1/recall \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"query":"what do I know about this user?","top_k":5,"min_similarity":0.3,"include_pending":true}'
```

## OpenClaw

Use the [official Persistio OpenClaw plugin](https://github.com/Persistio/openclaw-persistio/blob/main/README.md) to connect OpenClaw agents to the self-host server.
