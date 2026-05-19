# Getting Started with Persistio

This guide walks through running Persistio, creating a vault, ingesting conversation chunks, recalling memory, and using the current OpenClaw plugin.

---

## Prerequisites

- Docker and Docker Compose
- `curl` or another HTTP client
- An admin key configured for the server
- An embedding and extractor provider configured for extraction/recall

---

## 1. Run the Server

Clone the source repository and start the local stack:

```bash
git clone https://github.com/chriscoveyduck/persistio
cd persistio
docker compose up -d
```

The server listens on `http://localhost:4827` by default.

```bash
curl http://localhost:4827/health
```

`GET /health` is unauthenticated unless `HEALTH_API_KEY` is set. When configured, send `X-Health-Key: <HEALTH_API_KEY>`.

The response includes database status and queue depth fields:

```json
{
  "status": "ok",
  "version": "0.1.9",
  "db": "ok",
  "db_latency_ms": 12,
  "extraction_queue_depth": 0,
  "curation_queue_depth": 0,
  "queue_depth": 0,
  "uptime_s": 34
}
```

---

## 2. Create a Vault

Persistio is multi-vault. A vault is the tenant boundary for memories, raw chunks, quotas, graph data, and API keys.

Admin routes accept `X-Admin-Key: <ADMIN_API_KEY>` or `Authorization: Bearer <ADMIN_API_KEY>`.

```bash
curl -X POST http://localhost:4827/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-agent",
    "purpose": "Long-term memory for my assistant",
    "plan": "free"
  }'
```

Response:

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "my-agent",
  "purpose": "Long-term memory for my assistant",
  "plan": "free",
  "api_key": "pt_your_api_key_here"
}
```

Save the `api_key`. Use it as a Bearer token for vault-scoped routes.

---

## 3. Ingest a Conversation

Send one logical session at a time. Each chunk must include `role`, `content`, and an ISO datetime `timestamp`.

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
      },
      {
        "role": "assistant",
        "content": "Got it, Alice. I will keep things brief.",
        "timestamp": "2026-05-19T12:00:05.000Z"
      }
    ]
  }'
```

Response:

```json
{
  "accepted": 2,
  "chunks": [
    {
      "id": "2a2b46b4-5f8b-4ebf-9817-2dfaf0665ca4",
      "created_at": "2026-05-19T12:00:00.000Z"
    }
  ]
}
```

The API stores and embeds chunks, groups them into segments, and queues extraction work. Extraction happens asynchronously in the worker.

---

## 4. Recall Memories

At the start of a later session, query relevant memory:

```bash
curl -X POST http://localhost:4827/v1/recall \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "what do I know about this user?",
    "top_k": 5,
    "mode": "agent"
  }'
```

Default response:

```json
{
  "memories": [
    {
      "id": "b5f7b2bb-d0f1-44a9-b10d-8d9d8d346a11",
      "data": "User's name is Alice and prefers concise answers.",
      "subject": "alice",
      "categories": [],
      "type": "user_preference",
      "scope": "global",
      "status": "active",
      "similarity": 0.94
    }
  ],
  "evidence_chunks": [],
  "raw_chunks": []
}
```

Use `include_evidence: true` to return source-linked chunks for returned memories. Use `include_raw: true` to include raw chunk matches.

For agent prompt assembly, request a structured bundle:

```bash
curl -X POST "http://localhost:4827/v1/recall?format=bundle" \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"query":"current user context","top_k":10,"mode":"agent"}'
```

Bundle sections include `global_user_rules`, `user_rules`, `user_preferences`, `task_patterns`, `workflows`, `project`, `constraints`, `decisions`, `system_facts`, and `domain_knowledge`.

---

## 5. Install the OpenClaw Plugin

The current plugin package is `@persistio/openclaw-plugin` `0.1.4` and requires OpenClaw `>=2026.3.24-beta.2`.

```bash
npm install -g @persistio/openclaw-plugin
```

Register it in OpenClaw:

```json
{
  "plugins": {
    "entries": {
      "persistio": {
        "package": "@persistio/openclaw-plugin",
        "config": {
          "baseURL": "https://api.persistio.ai",
          "apiKey": "pt_your_api_key_here",
          "tokenBudget": 2000,
          "recallTopK": 10,
          "recallTimeout": 5000,
          "send": {
            "roles": {
              "user": "enabled",
              "agent": "enabled",
              "tool": "disabled"
            }
          }
        }
      }
    }
  }
}
```

The plugin hooks into `before_prompt_build` to recall memory and `agent_end` to ingest new transcript messages. It exposes `memory_search`, `memory_add`, `memory_delete`, and `memory_list`.

---

## Next Steps

- [API Reference](api-reference.md)
- [Integration Guide](integration-guide.md)
- [Architecture](architecture.md)
- [Data Model](data-model.md)
