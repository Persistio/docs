# Persistio API

## Base Behavior

- Server default listen port: `4827`
- Health route: `/health`
- Product API routes are registered only when `PERSISTIO_MODE !== worker`
- All request validation is done with Zod
- All successful JSON responses are unversioned and route-specific

## Authentication

### Vault authentication

Current implementation uses `Authorization: Bearer <vault_api_key>` for vault-scoped routes via `requireVaultAuth()`.

Important: the checked-in code does not currently read `x-api-key` for vault auth. If external docs mention `x-api-key`, that is not what `packages/server/src/middleware/auth.ts` enforces today.

### Admin authentication

Admin routes accept either:

- `x-admin-key: <ADMIN_API_KEY>`
- `Authorization: Bearer <ADMIN_API_KEY>`

### Health authentication

`GET /health` is unauthenticated unless `HEALTH_API_KEY` is configured. When configured, clients must send `x-health-key: <HEALTH_API_KEY>`.

## Rate Limiting

Quota-enforced routes emit rate-limit headers after successful quota consumption:

- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset`
- `Retry-After` on quota exhaustion

Routes that currently consume quota:

- `POST /v1/ingest` -> `ingest_events` after validation and embedding succeeds
- `POST /v1/recall` -> `searches`
- `POST /v1/memories` -> `memory_adds` plus memory-capacity enforcement

`GET` memory routes, admin routes, health, stats, and job polling do not currently consume quota.

## Error Shape

The common explicit error body is:

```json
{ "error": "Message" }
```

Common status codes:

- `400` invalid parent ownership or malformed inputs after validation/branch logic
- `401` missing or invalid auth
- `404` missing memory, job, or vault
- `429` quota exceeded
- `503` degraded health or open circuit breaker surfaced through dependency failures

Zod validation failures are handled by Fastify’s default error handling, not by a custom uniform error mapper.

## Routes

## `GET /health`

Purpose:

- liveness/readiness probe with DB ping and extraction/curation queue depths

Auth:

- none unless `HEALTH_API_KEY` is configured

Request:

- no body
- optional header `x-health-key`

Response `200`:

```json
{
  "status": "ok",
  "version": "0.1.17",
  "db": "ok",
  "db_latency_ms": 12,
  "extraction_queue_depth": 2,
  "curation_queue_depth": 1,
  "uptime_s": 345
}
```

Response `503`:

```json
{
  "status": "degraded",
  "version": "0.1.17",
  "db": "degraded",
  "db_latency_ms": 2001,
  "extraction_queue_depth": null,
  "curation_queue_depth": null,
  "uptime_s": 345
}
```

Errors:

- `401` invalid `x-health-key`

## `GET /v1/curation`

Purpose:

- return the authenticated vault's live Curator capacity state

Auth:

- vault Bearer token required

Response `200`:

```json
{
  "vault_id": "uuid",
  "plan": "unlimited",
  "period": "2026-05",
  "limits": {
    "curator_enabled": true,
    "curator_schedule_interval_minutes": 15,
    "curator_runs_per_month": 3000,
    "curator_jobs_per_run": 20,
    "curator_candidates_per_run": 400,
    "curator_candidates_per_call": 20,
    "curator_active_memories_per_call": 80,
    "curator_input_tokens_per_call": 12000,
    "curator_output_tokens_per_call": 2000,
    "curator_tokens_per_month": 25000000,
    "curator_requests_per_month": 6000,
    "curator_max_queue_age_hours": 24,
    "curator_backlog_limit": 10000,
    "curator_priority_weight": 3,
    "curator_overage_mode": "payg"
  },
  "usage": {
    "curator_runs": 2,
    "curator_requests": 4,
    "curator_input_tokens": 18000,
    "curator_output_tokens": 2100,
    "curator_candidates_processed": 65,
    "curator_candidates_deferred": 3
  },
  "backlog": {
    "pending_queue_rows": 7,
    "oldest_queue_age_seconds": 5400,
    "backlog_limit": 500,
    "pressure": "ok"
  },
  "schedule": {
    "last_run_at": "2026-05-23T08:00:00.000Z",
    "next_run_at": "2026-05-23T14:00:00.000Z",
    "eligible_now": false,
    "defer_reason": "waiting for scheduled curator interval"
  },
  "recent_runs": []
}
```

Errors:

- `401` unauthorized

## `GET /admin/vaults/:id/curation`

Purpose:

- return staff-readable live Curator queue, usage, plan-limit, schedule, defer-reason, and recent-run state for a vault

Auth:

- admin auth required

Response:

- same shape as `GET /v1/curation`

Errors:

- `401` unauthorized
- `404` vault not found

## `POST /v1/ingest`

Purpose:

- append raw conversation chunks, create segments, and enqueue extraction work

Auth:

- vault Bearer token required

Request body:

```json
{
  "session_id": "chat-123",
  "chunks": [
    {
      "role": "user",
      "content": "Need docs for Persistio.",
      "timestamp": "2026-05-12T16:00:00.000Z"
    }
  ]
}
```

Schema:

- `session_id`: non-empty string
- `chunks`: non-empty array
- `chunks[].role`: `user | assistant | tool`
- `chunks[].content`: non-empty string, up to `INGEST_CHUNK_MAX_CHARS` characters. For `EMBEDDER_PROVIDER=openai`, the API also preflights the OpenAI per-input estimated token limit. Default: 8000 characters.
- `chunks[].timestamp`: required ISO 8601 datetime string with UTC `Z` or an explicit offset. This must be the original conversation turn time, not the ingest request time.

Behavior:

- atomically reserves one `ingest_events` quota unit before embedding/provider work; validation failures do not reserve quota, and embedding/storage failures refund the reservation
- applies `INGEST_RATE_LIMIT_RPM` to normal ingest for non-premium vaults
- embeds every chunk before DB write; OpenAI embedding calls are internally split below provider batch/token limits
- encrypts chunk content when vault encryption is active
- stores raw chunk payloads in the configured raw chunk storage backend and inserts `raw_chunks` pointer rows with `created_at` set from `chunks[].timestamp`
- groups inserted chunks into segment drafts using cosine similarity and min/max segment size rules
- inserts `segments`
- inserts `extraction_queue` rows at `normal` priority

The live OpenClaw plugin populates `timestamp` automatically from the conversation message metadata. Custom clients and backfill importers must supply it explicitly so extraction can resolve relative dates against the source conversation time.

Response `202`:

```json
{
  "accepted": 1,
  "chunks": [
    {
      "id": "uuid",
      "created_at": "2026-05-12T16:00:00.000Z"
    }
  ]
}
```

Errors:

- `401` unauthorized
- `429` ingest quota exceeded

## `POST /v1/ingest/bulk`

Purpose:

- premium backfill/import path for sending up to `BULK_INGEST_MAX_CHUNKS` chunks in one request, 2048 by default
- request bodies may be up to `BULK_INGEST_BODY_LIMIT_BYTES` bytes, 25 MiB by default

Auth:

- vault Bearer token required
- vault plan must be `unlimited`

Request body:

- same schema as `POST /v1/ingest`

Behavior:

- atomically reserves one `ingest_events` quota unit before embedding/provider work; validation failures do not reserve quota, and embedding/storage failures refund the reservation
- embeds the chunk array with `embedBatch()`
- writes all raw chunks with one multi-row `INSERT`
- builds segments and enqueues them at `bulk` priority
- returns a `job_id` that can be polled with `GET /v1/jobs/:id`

Response `202`:

```json
{
  "accepted": 250,
  "chunks": [
    {
      "id": "uuid",
      "created_at": "2026-05-12T16:00:00.000Z"
    }
  ],
  "job_id": "uuid"
}
```

Errors:

- `401` unauthorized
- `403` vault plan is not premium
- `413` more than `BULK_INGEST_MAX_CHUNKS` chunks or request body exceeds `BULK_INGEST_BODY_LIMIT_BYTES`
- `413` any chunk exceeds `INGEST_CHUNK_MAX_CHARS`
- `429` ingest quota exceeded

## `POST /v1/recall`

Purpose:

- semantic memory recall, optional raw-chunk recall, or bundled memory context

Auth:

- vault Bearer token required

Query parameters:

- `format=bundle` optional

Request body:

```json
{
  "query": "How does Persistio extraction work?",
  "top_k": 10,
  "min_similarity": 0.30,
  "include_raw": false,
  "include_evidence": false,
  "include_pending": false,
  "mode": "agent"
}
```

Schema:

- `query`: non-empty string
- `top_k`: positive integer up to 100, optional
- `min_similarity`: number from 0 to 1, optional; defaults to `MIN_RECALL_SIMILARITY`
- `include_raw`: boolean, default `false`
- `include_evidence`: boolean, default `false`
- `include_pending`: boolean, default `false`; when true, direct recall may include fresh `candidate` memories that are awaiting curation
- `mode`: `agent | factual`, default `agent`

Behavior:

- consumes `searches` quota
- embeds the query
- in `agent` + `format=bundle` mode, returns up to 5 always-on global `user_rule` memories separately as `global_user_rules`
- query-relevant `memories` / `bundle` sections are selected from semantic recall within the `top_k` budget
- by default, semantic recall only searches active memories; when `include_pending=true`, it also searches fresh `candidate` memories whose `source_timestamp` (or `created_at` fallback) is within the pending recall freshness window
- overfetches semantic candidates from `memory_embeddings`, drops semantic and raw matches below `min_similarity`, applies mode-aware type ranking plus a small recency boost, then applies `top_k`
- in `agent` mode, close semantic matches with behavioral types such as `user_rule`, `user_preference`, and `task_pattern` rank ahead of factual/structural types
- in `factual` mode, close semantic matches with factual/structural types such as `system_fact`, `domain_knowledge`, `project`, `decision`, and `constraint` rank ahead of behavioral types
- close semantic matches from recent source conversations rank ahead of older memories; recency uses `source_timestamp` first, then `updated_at`, then `created_at`, and decays over a short rolling window
- `format=bundle` preserves the final recall ranking within each bundle section
- returns fewer than `top_k` direct semantic matches when insufficient memories clear the quality floor
- retrieves outgoing graph neighbors from `memory_edges` as supplemental `related_memories` / `related_bundle` context outside the direct `top_k` budget
- graph neighbors do not receive a synthetic similarity score and do not compete with direct semantic matches
- when `include_evidence=true`, retrieves bounded source-linked raw chunks for returned memories from `source_chunks` / `source_segment_id` (up to 6 per memory and 200 total)
- de-duplicates related graph results against direct semantic results
- updates `last_recalled` and `recall_count`
- decrypts returned memory data and optional raw chunk content

Response `200` default format:

```json
{
  "memories": [
    {
      "id": "uuid",
      "data": "Persistio runs extraction asynchronously.",
      "subject": "persistio",
      "categories": [],
      "confidence": 1,
      "score": 8,
      "salience": "0.80",
      "sensitivity": "low",
      "type": "system_fact",
      "scope": "global",
      "polarity": "neutral",
      "status": "active",
      "valid_from": null,
      "valid_until": null,
      "source_timestamp": "2026-05-12T16:00:00.000Z",
      "similarity": 0.91,
      "source": "semantic",
      "created_at": "2026-05-12T16:00:00.000Z",
      "updated_at": "2026-05-12T16:00:00.000Z",
      "recall_count": 4,
      "last_recalled": "2026-05-12T16:10:00.000Z"
    }
  ],
  "related_memories": [
    {
      "id": "uuid",
      "data": "Persistio extraction produces candidate memories.",
      "subject": "persistio",
      "categories": [],
      "confidence": 1,
      "score": 8,
      "salience": "0.80",
      "sensitivity": "low",
      "type": "system_fact",
      "scope": "global",
      "polarity": "neutral",
      "status": "active",
      "valid_from": null,
      "valid_until": null,
      "source_timestamp": "2026-05-12T16:00:00.000Z",
      "similarity": null,
      "source": "graph",
      "edge_type": "related",
      "created_at": "2026-05-12T16:00:00.000Z",
      "updated_at": "2026-05-12T16:00:00.000Z",
      "recall_count": 4,
      "last_recalled": "2026-05-12T16:10:00.000Z"
    }
  ],
  "evidence_chunks": [
    {
      "memory_id": "uuid",
      "id": "uuid",
      "session_id": "session-1",
      "role": "user",
      "content": "Persistio runs extraction asynchronously.",
      "created_at": "2026-05-16T12:34:56.000Z"
    }
  ],
  "raw_chunks": []
}
```

For memory objects, `source_timestamp` is the original conversation event time inferred from the source chunk timestamps. `created_at` is when the memory row was extracted or manually created. These can differ substantially for backfills.

Response `200` with `format=bundle`:

```json
{
  "bundle": {
    "global_user_rules": [],
    "user_rules": [],
    "user_preferences": [],
    "task_patterns": [],
    "workflows": [],
    "project": [],
    "constraints": [],
    "decisions": [],
    "system_facts": [],
    "domain_knowledge": []
  },
  "related_bundle": {
    "global_user_rules": [],
    "user_rules": [],
    "user_preferences": [],
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

Errors:

- `401` unauthorized
- `429` search quota exceeded

## `GET /v1/memories`

Purpose:

- list visible non-candidate memories, optionally including descendants

Auth:

- vault Bearer token required

Query parameters:

- `archived=true|false` default `false`
- `category=<string>` optional
- `include_children=<boolean>` default `false`
- `limit=<1..200>` default `50`
- `offset=<>=0` default `0`

Behavior:

- excludes `status='candidate'`
- when `include_children=true`, uses a recursive CTE up to depth 10 and then slices in application code

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "subject": "persistio",
      "data": "Persistio stores memories in Postgres.",
      "categories": [],
      "confidence": 1,
      "score": 5,
      "salience": "0.50",
      "sensitivity": "low",
      "type": "system_fact",
      "scope": "global",
      "evidence": null,
      "polarity": "neutral",
      "status": "active",
      "valid_from": null,
      "valid_until": null,
      "source_timestamp": "2026-05-12T16:00:00.000Z",
      "archived_at": null,
      "created_at": "2026-05-12T16:00:00.000Z",
      "updated_at": "2026-05-12T16:00:00.000Z",
      "parent_id": null,
      "volatility": "low",
      "edge_count": 0
    }
  ],
  "limit": 50,
  "offset": 0
}
```

`source_timestamp` is nullable for manually created memories and for legacy rows that predate source-date extraction.

Errors:

- `401` unauthorized

## `GET /v1/memories/graph`

Purpose:

- return a bounded live memory graph for vault review and App visualisation
- support seeded neighborhood expansion without loading the full vault graph

Auth:

- vault Bearer token required

Query parameters:

- `seed_memory_id=<uuid>` optional; when set, expansion starts from this visible memory
- `depth=<0..3>` default `1`
- `limit=<1..100>` default `50`
- `direction=out|in|both` default `both`
- `edge_types=<type,type>` optional comma-separated filter; repeated query params are also accepted

Behavior:

- excludes archived memories and `status='candidate'`
- with `seed_memory_id`, recursively expands through `memory_edges` up to `depth`
- without `seed_memory_id`, returns a salience/update ordered overview of visible memories
- returns only edges where both endpoints are present in the returned `nodes`
- applies `limit` to returned nodes and caps returned edges at 500
- returns `404` when a supplied seed is not visible in the authenticated vault

Response `200`:

```json
{
  "seed_memory_id": "uuid",
  "depth": 1,
  "limit": 50,
  "direction": "both",
  "edge_types": ["supports", "contradicts"],
  "nodes": [
    {
      "id": "uuid",
      "subject": "persistio",
      "data": "Persistio stores memories in Postgres.",
      "categories": [],
      "confidence": 1,
      "score": 5,
      "salience": "0.50",
      "sensitivity": "low",
      "type": "system_fact",
      "scope": "global",
      "evidence": null,
      "polarity": "neutral",
      "status": "active",
      "valid_from": null,
      "valid_until": null,
      "source_timestamp": "2026-05-12T16:00:00.000Z",
      "archived_at": null,
      "created_at": "2026-05-12T16:00:00.000Z",
      "updated_at": "2026-05-12T16:00:00.000Z",
      "parent_id": null,
      "volatility": "low",
      "edge_count": 2,
      "depth": 0
    }
  ],
  "edges": [
    {
      "id": "uuid",
      "from_memory_id": "uuid",
      "to_memory_id": "uuid",
      "type": "supports",
      "confidence": 0.9,
      "reason": "Related implementation details",
      "created_at": "2026-05-12T16:00:00.000Z",
      "updated_at": "2026-05-12T16:00:00.000Z"
    }
  ]
}
```

Errors:

- `400` invalid graph query
- `401` unauthorized
- `404` seed memory not found

## `POST /v1/memories`

Purpose:

- create a memory directly through the API

Auth:

- vault Bearer token required

Request body:

```json
{
  "data": "Use concise final responses.",
  "subject": "user",
  "categories": ["style"],
  "parent_id": null,
  "type": "user_preference",
  "scope": "global",
  "evidence": "User asked for concise answers.",
  "volatility": "low"
}
```

Schema:

- `data`: non-empty string
- `subject`: non-empty string
- `categories`: string array, default `[]`
- `parent_id`: nullable UUID, optional
- `type`: memory type enum, default `system_fact`
- `scope`: `global | project | task | session`, default `global`
- `evidence`: string, optional
- `volatility`: `very_low | low | medium | high`, default `low`

Behavior:

- enforces `memories_max`
- consumes `memory_adds` quota
- embeds `data`
- encrypts `data` and optionally `subject`
- inserts into `memories` and `memory_embeddings`

Response `201`:

- returns the full decrypted memory row

Errors:

- `400` `parent_id` does not belong to the same vault
- `401` unauthorized
- `429` memory creation or memory_adds quota exceeded

## `GET /v1/memories/:id`

Purpose:

- fetch a single non-candidate memory, or a fresh pending candidate when explicitly requested

Auth:

- vault Bearer token required

Path params:

- `id`: UUID

Query params:

- `include_pending`: boolean, default `false`; when true, the response may include a fresh `candidate` memory within the same pending recall freshness window used by `/v1/recall`

Response `200`:

- full decrypted memory row

Errors:

- `401` unauthorized
- `404` memory not found

## `PATCH /v1/memories/:id`

Purpose:

- update a single non-candidate memory

Auth:

- vault Bearer token required

Path params:

- `id`: UUID

Request body fields:

- `data`
- `subject`
- `categories`
- `confidence`
- `type`
- `scope`
- `evidence`
- `archived`

Behavior:

- reloads the current memory first
- re-encrypts and re-embeds when `data` changes
- re-encrypts subject when `subject` changes
- toggles `archived_at` based on `archived`
- updates `memory_embeddings` when content changes

Response `200`:

- full decrypted updated memory row

Errors:

- `401` unauthorized
- `404` memory not found

## `DELETE /v1/memories/:id`

Purpose:

- soft-archive a single non-candidate memory

Auth:

- vault Bearer token required

Path params:

- `id`: UUID

Response `200`:

```json
{
  "id": "uuid",
  "archived_at": "2026-05-12T16:00:00.000Z"
}
```

Errors:

- `401` unauthorized
- `404` memory not found

## `POST /v1/extract`

Purpose:

- trigger an immediate extraction worker pass for the authenticated vault

Auth:

- vault Bearer token required

Behavior:

- creates an in-memory job record
- posts a `run-once` message to the extraction worker thread
- if running in API-only mode, the job is marked failed because no worker thread exists

Response `202`:

```json
{ "job_id": "uuid" }
```

Errors:

- `401` unauthorized

## `GET /v1/jobs/:id`

Purpose:

- poll the in-memory status of an extraction trigger job

Auth:

- vault Bearer token required

Path params:

- `id`: UUID

Response `200`:

```json
{
  "id": "uuid",
  "vaultId": "uuid",
  "status": "completed",
  "createdAt": "2026-05-12T16:00:00.000Z",
  "updatedAt": "2026-05-12T16:00:05.000Z"
}
```

Status enum:

- `queued`
- `running`
- `completed`
- `failed`

Errors:

- `401` unauthorized
- `404` job not found or belongs to a different vault

## `GET /stats`

Purpose:

- return current plan, usage, memory counts, alias counts, and contradiction scan metadata
- report live/current-period state only; historical billing usage is App-owned and populated from platform period-close events

Auth:

- vault Bearer token required

Response `200`:

```json
{
  "vault_id": "uuid",
  "plan": "unlimited",
  "period": "2026-05",
  "memories": {
    "active": 10,
    "candidate": 2,
    "needs_review": 1,
    "contradicted": 0,
    "superseded": 1,
    "archived": 3,
    "limit": 50000
  },
  "entity_aliases": 6,
  "contradiction_scan": {
    "last_run": "2026-05-12T16:00:00.000Z",
    "arbitrations_this_week": 3
  },
  "usage": {
    "ingest_events": { "consumed": 12, "limit": 10000 },
    "memory_adds": { "consumed": 20, "limit": 50000 },
    "searches": { "consumed": 7, "limit": 100000 }
  }
}
```

Errors:

- `401` unauthorized

Historical usage:

- the platform does not expose long-term usage history from Postgres
- on billing-period rollover, the platform should durably emit `vault.usage_period.closed`
- downstream billing/reporting systems store closed-period usage history for billing views, charts, and upgrade prompts

## `POST /admin/vaults`

Purpose:

- create a vault and issue its initial API key

Auth:

- admin key required

Request body:

```json
{
  "name": "Acme",
  "purpose": "Persistent memory for Acme support agents",
  "plan": "unlimited",
  "type": "general"
}
```

Schema:

- `name`: non-empty string
- `purpose`: optional string, max 500
- `plan`: optional plan id, default `unlimited`
- `type`: optional vault prompt type: `general` or `custom`; unset falls back to general prompts
- `custom_extraction_prompt`: optional string, required when `type` is `custom`, write-only in admin responses
- `custom_curation_prompt`: optional string, required when `type` is `custom`, max 24KB, write-only in admin responses

Behavior:

- generates a random vault API key
- hashes the key before storage
- requires the referenced plan to already exist
- stores per-vault embedding dimensions in `settings`
- generates a wrapped DEK when encryption is enabled
- selects extraction and curation prompts from the vault `type`; unset or `general` vaults use the configured extractor and curator prompt files
- validates custom prompts before storage; custom prompts require an `unlimited` plan
- stores custom prompts through the same vault encryption path as customer memory data when vault encryption is active

Response `201`:

```json
{
  "id": "uuid",
  "name": "Acme",
  "purpose": "Persistent memory for Acme support agents",
  "plan": "unlimited",
  "type": "general",
  "api_key": "raw-generated-key"
}
```

Errors:

- `401` unauthorized
- `400` invalid custom prompt with actionable `feedback`
- `403` custom prompts require an Unlimited plan
- `404` plan not found

## `GET /admin/plans`

Purpose:

- list platform plans

Auth:

- admin key required

Response `200`:

```json
{
  "items": [
    {
      "id": "unlimited",
      "limits": {}
    }
  ]
}
```

## `GET /admin/plans/:id`

Purpose:

- return one platform plan

Auth:

- admin key required

Errors:

- `401` unauthorized
- `404` plan not found

## `POST /admin/plans`

Purpose:

- create or replace a platform plan's limits

Auth:

- admin key required

Request body:

```json
{
  "id": "unlimited",
  "limits": {}
}
```

## `PATCH /admin/plans/:id`

Purpose:

- replace an existing platform plan's limits

Auth:

- admin key required

Request body:

```json
{
  "limits": {}
}
```

Errors:

- `401` unauthorized
- `404` plan not found

## `DELETE /admin/plans/:id`

Purpose:

- delete an unused platform plan

Auth:

- admin key required

Errors:

- `401` unauthorized
- `404` plan not found
- `409` plan is referenced by at least one vault

## `POST /admin/vaults/:id/rotate-key`

Purpose:

- rotate a vault API key

Auth:

- admin key required

Path params:

- `id`: UUID

Response `200`:

```json
{
  "id": "uuid",
  "api_key": "new-raw-key"
}
```

Errors:

- `401` unauthorized
- `404` vault not found

## `PATCH /admin/vaults/:id`

Purpose:

- update vault metadata

Auth:

- admin key required

Path params:

- `id`: UUID

Request body:

- `name`: optional non-empty string
- `purpose`: optional string, nullable, max 500
- `plan`: optional plan id
- `status`: optional `active` or `disabled`
- `type`: optional vault prompt type, nullable to restore general fallback
- `custom_extraction_prompt`: optional string, nullable, write-only in admin responses
- `custom_curation_prompt`: optional string, nullable, write-only in admin responses

At least one field is required.

Plan updates only assign the vault to an existing plan. Plan limits are managed through `/admin/plans`, not through vault create/update requests.

When `type` is `custom`, both custom prompts must be present after the update and must pass validation. Setting a non-custom type clears the stored custom prompts.

Response `200`:

- updated vault row including `settings`, `plan_id`, `account_id`, and `vault_encryption_enabled`

Errors:

- `401` unauthorized
- `400` invalid request body or custom prompt validation failure
- `403` custom prompts require an Unlimited plan
- `404` vault or plan not found

## `DELETE /admin/vaults/:id`

Purpose:

- delete a vault, cascade its database data, and enqueue referenced raw chunk blobs for durable cleanup from the configured storage backend

If blob cleanup fails after the vault row is deleted, the blob keys remain in `raw_chunk_blob_deletion_queue`; retrying the same DELETE attempts cleanup again.

Auth:

- admin key required

Response `200`:

```json
{
  "id": "uuid",
  "deleted": true
}
```

Errors:

- `401` unauthorized
- `404` vault not found

## `GET /admin/vaults`

Purpose:

- list all vaults

Auth:

- admin key required

Response `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "name": "Acme",
      "purpose": null,
      "created_at": "2026-05-12T16:00:00.000Z",
      "settings": {},
      "plan_id": "free",
      "type": null,
      "has_custom_extraction_prompt": false,
      "has_custom_curation_prompt": false,
      "account_id": null,
      "vault_encryption_enabled": false
    }
  ]
}
```

Errors:

- `401` unauthorized
