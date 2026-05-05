# Getting Started with Persistio

This guide walks you through integrating Persistio from scratch — from running the server to making your first ingest and recall calls.

---

## Prerequisites

- Docker and Docker Compose installed
- `curl` or an HTTP client

---

## 1. Run the Server

Clone the [Persistio/server](https://github.com/Persistio/server) repository and start the stack:

```bash
git clone https://github.com/Persistio/server
cd server
docker compose up -d
```

The server starts on `http://localhost:4827` by default. Confirm it's healthy:

```bash
curl http://localhost:4827/health
# → 200 OK
```

The Docker Compose stack includes:
- The Persistio API server
- A PostgreSQL database
- The async extraction daemon

See the [server README](https://github.com/Persistio/server) for environment variable configuration (LLM provider keys, admin key, port, etc.).

---

## 2. Create a Vault

Persistio is multi-vault. Each vault gets its own isolated memory store and API key. Admin calls use the `X-Admin-Key` header.

```bash
curl -X POST https://your-persistio-instance/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-agent"}'
```

Response:

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "api_key": "pt_your_api_key_here"
}
```

Save the `api_key` — you'll use it as a Bearer token for all vault API calls.

> **Auth summary:**
> - Vault API calls → `Authorization: Bearer <api_key>`
> - Admin calls → `X-Admin-Key: <ADMIN_API_KEY>`

---

## 3. Ingest Your First Conversation

Send a conversation session to Persistio. Each call represents one session (e.g. a single chat thread), made up of chunks.

```bash
curl -X POST https://your-persistio-instance/v1/ingest \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "session-2024-001",
    "chunks": [
      { "role": "user", "content": "My name is Alice and I prefer concise answers." },
      { "role": "assistant", "content": "Got it, Alice. I'\''ll keep things brief." },
      { "role": "user", "content": "I'\''m working on a Rust project targeting WebAssembly." }
    ]
  }'
```

Response: `202 Accepted`

The ingest call returns immediately. The extraction daemon picks up the chunks asynchronously and derives durable memories from them.

---

## 4. Recall Memories

At the start of your next session — or any time you need context — query for relevant memories:

```bash
curl -X POST https://your-persistio-instance/v1/recall \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "what do I know about this user?",
    "top_k": 5
  }'
```

Response:

```json
{
  "memories": [
    {
      "id": "mem_abc123",
      "data": "User's name is Alice and prefers concise answers.",
      "score": 0.94
    },
    {
      "id": "mem_def456",
      "data": "User is working on a Rust project targeting WebAssembly.",
      "score": 0.87
    }
  ]
}
```

Prepend these memories to your LLM system prompt to give your agent persistent context.

---

## 5. What Happens Next

After you call `POST /v1/ingest`, the extraction daemon processes your chunks asynchronously:

1. **Chunks are stored** immediately on ingest
2. **The extraction daemon** picks them up, runs an LLM-powered extraction pass, and identifies facts, preferences, and events worth remembering
3. **Durable memories** are written to the memory store, tagged, and indexed for semantic search
4. **Recall** queries these memories using vector similarity search

Because extraction is async, there's a short delay (typically seconds) between ingesting a session and those memories being available via recall. For time-sensitive cases, trigger extraction manually:

```bash
# Trigger extraction immediately
curl -X POST https://your-persistio-instance/v1/extract \
  -H "Authorization: Bearer pt_your_api_key_here"

# Poll for completion
curl https://your-persistio-instance/v1/jobs/<job_id> \
  -H "Authorization: Bearer pt_your_api_key_here"
```

See the [Integration Guide](integration-guide.md) for patterns on structuring ingest calls, recall strategies, and multi-vault setups.

---

## Next Steps

- [API Reference](api-reference.md) — full endpoint docs with code samples
- [Integration Guide](integration-guide.md) — patterns for production integrations
- [Persistio/openclaw-persistio](https://github.com/Persistio/openclaw-persistio) — drop-in plugin if you're using OpenClaw
