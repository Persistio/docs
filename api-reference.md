# Persistio API Reference

Base URL examples use `https://your-persistio-instance`.

**Authentication**

- Vault routes: `Authorization: Bearer pt_your_api_key_here`
- Admin routes: `X-Admin-Key: adm_your_admin_key_here` or `Authorization: Bearer adm_your_admin_key_here`
- Health route: no auth unless `HEALTH_API_KEY` is configured, then `X-Health-Key: <value>`

Quota-enforced routes may return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After`.

Common explicit error shape:

```json
{ "error": "Message" }
```

---

## Health

### `GET /health`

Returns service health, server version, database latency, and queue depths.

**Response:** `200 OK` when healthy, `503 Service Unavailable` when degraded.

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

**curl**

```bash
curl https://your-persistio-instance/health
```

---

## Ingest

### `POST /v1/ingest`

Append raw conversation chunks, embed them, group them into segments, and enqueue extraction work.

**Auth:** Bearer token

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | string | yes | Logical conversation/session identifier |
| `chunks` | array | yes | Non-empty array of conversation chunks |
| `chunks[].role` | string | yes | `user`, `assistant`, or `tool` |
| `chunks[].content` | string | yes | Non-empty message content |
| `chunks[].timestamp` | string | yes | ISO datetime for the source message |

**Response:** `202 Accepted`

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

**curl**

```bash
curl -X POST https://your-persistio-instance/v1/ingest \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "session-2026-001",
    "chunks": [
      {
        "role": "user",
        "content": "My name is Alice.",
        "timestamp": "2026-05-19T12:00:00.000Z"
      }
    ]
  }'
```

---

## Recall

### `POST /v1/recall`

Retrieve semantic memory matches, optional evidence chunks, optional raw chunks, or a structured agent bundle.

**Auth:** Bearer token

**Query parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `format` | string | Use `bundle` to return grouped prompt context instead of default arrays |

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | yes | Natural language query |
| `top_k` | integer | no | Positive integer up to 100; defaults to server `DEFAULT_RECALL_TOP_K` |
| `include_raw` | boolean | no | Return semantically matching raw chunks in `raw_chunks`; default `false` |
| `include_evidence` | boolean | no | Return source-linked chunks for returned memories; default `false` |
| `mode` | string | no | `agent` or `factual`; default `agent` |

**Default response:** `200 OK`

```json
{
  "memories": [
    {
      "id": "b5f7b2bb-d0f1-44a9-b10d-8d9d8d346a11",
      "data": "User prefers concise answers.",
      "subject": "alice",
      "categories": ["preferences"],
      "confidence": 1,
      "score": 8,
      "salience": "0.80",
      "sensitivity": "low",
      "type": "user_preference",
      "scope": "global",
      "polarity": "neutral",
      "status": "active",
      "valid_from": null,
      "valid_until": null,
      "similarity": 0.94,
      "source": "semantic",
      "created_at": "2026-05-19T12:00:00.000Z",
      "updated_at": "2026-05-19T12:00:00.000Z",
      "recall_count": 4,
      "last_recalled": "2026-05-19T12:10:00.000Z"
    }
  ],
  "evidence_chunks": [],
  "raw_chunks": []
}
```

When `include_evidence` is true, `evidence_chunks` contains source chunks linked to returned memories:

```json
{
  "memory_id": "b5f7b2bb-d0f1-44a9-b10d-8d9d8d346a11",
  "id": "2a2b46b4-5f8b-4ebf-9817-2dfaf0665ca4",
  "session_id": "session-2026-001",
  "role": "user",
  "content": "My name is Alice.",
  "created_at": "2026-05-19T12:00:00.000Z"
}
```

When `include_raw` is true, `raw_chunks` contains semantically matching raw chunks with `similarity`.

**Bundle response**

Request `POST /v1/recall?format=bundle` to receive grouped prompt context:

```json
{
  "bundle": {
    "global_user_rules": [],
    "user_rules": [],
    "user_preferences": ["User prefers concise answers."],
    "task_patterns": [],
    "workflows": [],
    "project": [],
    "constraints": [],
    "decisions": [],
    "system_facts": [],
    "domain_knowledge": []
  }
}
```

In `agent` bundle mode, up to 5 active global `user_rule` memories are included in `global_user_rules` without consuming the query `top_k` budget. Query-relevant sections come from semantic and graph recall.

**curl**

```bash
curl -X POST "https://your-persistio-instance/v1/recall?format=bundle" \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"query":"current user context","top_k":10,"mode":"agent"}'
```

---

## Memories

### `GET /v1/memories`

List memories for the current vault.

**Auth:** Bearer token

**Query parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `archived` | boolean string | `false` by default; `true` returns archived memories |
| `category` | string | Filter to memories containing the category |
| `include_children` | boolean | Include descendants through `parent_id` hierarchy |
| `limit` | integer | Page size, `1..200`, default `50` |
| `offset` | integer | Pagination offset, default `0` |

**Response:** `200 OK`

```json
{
  "items": [],
  "limit": 50,
  "offset": 0
}
```

### `POST /v1/memories`

Create a memory directly.

**Auth:** Bearer token

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data` | string | yes | Fact statement |
| `subject` | string | yes | Subject the fact is about |
| `categories` | string[] | no | Free-form tags |
| `parent_id` | uuid or null | no | Parent memory in a hierarchy |
| `type` | string | no | `user_preference`, `user_rule`, `task_pattern`, `workflow`, `project`, `constraint`, `decision`, `system_fact`, or `domain_knowledge`; default `system_fact` |
| `scope` | string | no | `global`, `project`, `task`, or `session`; default `global` |
| `evidence` | string | no | Evidence summary stored as structured provenance |
| `volatility` | string | no | `very_low`, `low`, `medium`, or `high`; default `low` |

**Response:** `201 Created`

```bash
curl -X POST https://your-persistio-instance/v1/memories \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "data": "User prefers concise answers.",
    "subject": "alice",
    "categories": ["preferences"],
    "type": "user_preference",
    "scope": "global"
  }'
```

### `GET /v1/memories/:id`

Fetch one memory by UUID.

**Auth:** Bearer token

**Response:** `200 OK` or `404 Not Found`

### `PATCH /v1/memories/:id`

Update memory content or metadata.

**Auth:** Bearer token

Mutable fields: `data`, `subject`, `categories`, `confidence`, `type`, `scope`, `evidence`, and `archived`.

```bash
curl -X PATCH https://your-persistio-instance/v1/memories/b5f7b2bb-d0f1-44a9-b10d-8d9d8d346a11 \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"data":"User prefers concise answers and bullet summaries.","archived":false}'
```

### `DELETE /v1/memories/:id`

Archive a memory. This is a soft delete; archived memories are excluded from recall.

**Auth:** Bearer token

**Response:** `200 OK` or `404 Not Found`

---

## Extraction Jobs

### `POST /v1/extract`

Trigger a worker pass for the current vault.

**Auth:** Bearer token

**Response:** `202 Accepted`

```json
{ "job_id": "1d5f84d3-52a2-47c1-90a5-37889f8f71be" }
```

### `GET /v1/jobs/:id`

Check a manually triggered extraction job.

**Auth:** Bearer token

Status values: `queued`, `running`, `completed`, `failed`.

```json
{
  "id": "1d5f84d3-52a2-47c1-90a5-37889f8f71be",
  "vaultId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "completed",
  "createdAt": "2026-05-19T12:00:00.000Z",
  "updatedAt": "2026-05-19T12:00:04.000Z"
}
```

---

## Stats

### `GET /stats`

Fetch vault plan, usage, limits, memory counts, entity alias count, and contradiction scan metadata.

**Auth:** Bearer token

**Response:** `200 OK`

```json
{
  "vault_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "plan": "free",
  "period": "2026-05",
  "memories": {
    "active": 12,
    "candidate": 0,
    "needs_review": 0,
    "contradicted": 0,
    "superseded": 1,
    "archived": 2,
    "limit": 1000
  },
  "entity_aliases": 3,
  "contradiction_scan": {
    "last_run": null,
    "arbitrations_this_week": 0
  },
  "usage": {
    "ingest_events": { "consumed": 20, "limit": 1000 },
    "memory_adds": { "consumed": 4, "limit": 100 },
    "searches": { "consumed": 37, "limit": 5000 }
  }
}
```

---

## Admin

### `POST /admin/vaults`

Create a vault and receive its API key.

**Auth:** `X-Admin-Key` or Bearer admin key

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Vault name |
| `purpose` | string | no | Vault-specific context used by extraction |
| `plan` | string | no | `free`, `starter`, or `pro`; default `free` |

**Response:** `201 Created`

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "my-agent",
  "purpose": "Personal assistant memory",
  "plan": "free",
  "api_key": "pt_your_api_key_here"
}
```

### `GET /admin/vaults`

List vaults.

**Auth:** `X-Admin-Key` or Bearer admin key

**Response:** `200 OK`

```json
{
  "items": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "my-agent",
      "purpose": "Personal assistant memory",
      "created_at": "2026-05-19T12:00:00.000Z",
      "settings": { "embedding_dimensions": 1536 },
      "plan_id": "free",
      "account_id": null,
      "vault_encryption_enabled": false
    }
  ]
}
```

### `PATCH /admin/vaults/:id`

Update a vault's `name`, `purpose`, or `plan`.

**Auth:** `X-Admin-Key` or Bearer admin key

```bash
curl -X PATCH https://your-persistio-instance/admin/vaults/a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{"purpose":"Memory for support assistant","plan":"starter"}'
```

### `POST /admin/vaults/:id/rotate-key`

Rotate a vault API key. The old key is invalidated immediately.

**Auth:** `X-Admin-Key` or Bearer admin key

**Response:** `200 OK`

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "api_key": "pt_new_api_key_here"
}
```

### `DELETE /admin/vaults/:id`

Delete a vault and its associated data.

**Auth:** `X-Admin-Key` or Bearer admin key

**Response:** `200 OK` or `404 Not Found`
