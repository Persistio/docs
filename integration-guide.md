# Integration Guide

This guide is for developers integrating Persistio directly or through the OpenClaw plugin. It covers ingest, recall, memory management, quotas, multi-vault setups, and plugin behavior.

---

## 1. Memory Lifecycle

Persistio uses a two-layer model:

```text
Raw conversation chunks
        -> segments
        -> async extraction worker
        -> durable memories and embeddings
        -> semantic and graph-assisted recall
        -> retrieved context
```

`POST /v1/ingest` is the write path. It stores chunks, embeds them, groups them into segments, and queues extraction. Recall is the read path. It retrieves active memories from `memory_embeddings`, optionally expands through directed `memory_edges`, and can return raw evidence.

For Pro vaults with curation enabled, extraction can write candidate memories first. The curation worker promotes, updates, archives, or links them into a graph.

---

## 2. Structuring Ingest Calls

Send all chunks for one logical conversation under one `session_id`. Each chunk requires a source timestamp.

```json
{
  "session_id": "user-alice-2026-05-19-session-1",
  "chunks": [
    {
      "role": "user",
      "content": "I prefer concise answers.",
      "timestamp": "2026-05-19T12:00:00.000Z"
    },
    {
      "role": "assistant",
      "content": "Understood.",
      "timestamp": "2026-05-19T12:00:05.000Z"
    }
  ]
}
```

Use `tool` chunks only when tool output contains durable context. Noisy tool logs should be filtered before ingest.

Persistio segments only the chunks in the current ingest request. For best extraction quality, batch enough adjacent messages to provide context. Segments use a minimum size of 3, maximum size of 40, and semantic split threshold from `SEGMENTATION_THRESHOLD`.

---

## 3. Recall Patterns

### Default recall

Use default recall when you want ranked memory objects:

```js
const res = await fetch(`${baseURL}/v1/recall`, {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${apiKey}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    query: 'what does this user prefer?',
    top_k: 10,
    include_evidence: true,
    mode: 'agent'
  })
});

const { memories, evidence_chunks } = await res.json();
```

Use `include_evidence` when you need source-linked raw chunks for returned memories. Use `include_raw` when you want semantic raw chunk matches alongside memory matches.

### Bundle recall

Use `?format=bundle` when assembling an agent prompt:

```js
const res = await fetch(`${baseURL}/v1/recall?format=bundle`, {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${apiKey}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    query: 'current user and project context',
    top_k: 10,
    mode: 'agent'
  })
});

const { bundle } = await res.json();
```

Bundle keys map memory types into stable prompt sections:

- `global_user_rules`
- `user_rules`
- `user_preferences`
- `task_patterns`
- `workflows`
- `project`
- `constraints`
- `decisions`
- `system_facts`
- `domain_knowledge`

In `agent` bundle mode, active global `user_rule` memories are returned separately so important behavioral rules do not consume the query `top_k` budget.

---

## 4. Managing Memories

Create memories directly when an application has a verified durable fact:

```json
{
  "data": "User prefers concise answers.",
  "subject": "alice",
  "categories": ["preferences"],
  "type": "user_preference",
  "scope": "global",
  "evidence": "User stated this preference in onboarding.",
  "volatility": "low"
}
```

Use these metadata fields consistently:

| Field | Recommended use |
|-------|-----------------|
| `type` | Drives bundle grouping and downstream prompt structure |
| `scope` | Distinguishes global, project, task, and session-local memories |
| `categories` | Lightweight tags for filtering and UI organization |
| `parent_id` | Hierarchies and grouped memory trees |
| `evidence` | Short provenance summary |
| `volatility` | How often the fact is expected to change |

Archive stale memories with `DELETE /v1/memories/:id` or `PATCH /v1/memories/:id` with `"archived": true`. Archived memories are excluded from recall and list results unless requested with `archived=true`.

---

## 5. Quotas and Stats

Quota is enforced on:

- `POST /v1/ingest` as `ingest_events`
- `POST /v1/recall` as `searches`
- `POST /v1/memories` as `memory_adds`, plus memory capacity checks

Read plan, usage, limits, memory status counts, alias count, and contradiction-scan metadata from:

```bash
curl https://your-persistio-instance/stats \
  -H "Authorization: Bearer pt_your_api_key_here"
```

Back off on `429` and use the returned rate-limit headers when present.

---

## 6. Multi-Vault Setup

Use one vault per tenant, user, agent, or isolated memory boundary.

```bash
curl -X POST https://your-persistio-instance/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name":"user-alice","purpose":"Alice assistant memory","plan":"starter"}'
```

Store each vault API key in your own secrets store and use it only for that tenant's vault-scoped requests.

Rotate compromised keys immediately:

```bash
curl -X POST https://your-persistio-instance/admin/vaults/<vault-id>/rotate-key \
  -H "X-Admin-Key: adm_your_admin_key_here"
```

---

## 7. OpenClaw Plugin

The current package is `@persistio/openclaw-plugin` `0.1.4`.

It hooks into:

- `before_prompt_build` to recall memory and inject it into the prompt within `tokenBudget`
- `agent_end` to ingest transcript messages after each run
- OpenClaw memory capability registration so Persistio memories are searchable through OpenClaw's memory backend

It exposes these tools:

| Tool | Description |
|------|-------------|
| `memory_search` | Semantic recall over the vault |
| `memory_add` | Store a fact manually |
| `memory_delete` | Delete/archive a memory by ID |
| `memory_list` | List vault memories |

Configuration:

```json
{
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
```

The plugin deduplicates transcript messages per session in-process and expires deduplication keys after 24 hours of session inactivity. Tool messages are disabled by default because they are often noisy.

---

## 8. Error Handling

Common status codes:

| Code | Meaning |
|------|---------|
| `202` | Accepted; async operation queued |
| `400` | Invalid request body or parent ownership |
| `401` | Missing or invalid auth |
| `404` | Memory, job, or vault not found |
| `429` | Quota exceeded |
| `500` | Server error |
| `503` | Degraded health or dependency failure |

Retry `429` and transient `5xx` responses with exponential backoff and jitter. Do not retry validation failures without changing the request.

---

## Further Reading

- [Getting Started](getting-started.md)
- [API Reference](api-reference.md)
- [Architecture](architecture.md)
- [Data Model](data-model.md)
