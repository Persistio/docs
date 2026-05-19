# Persistio Data Model

Persistio is vault-scoped. Almost every application table keys data by `vault_id`, making the vault the tenant boundary for raw data, durable memories, aliases, graph relationships, queues, and usage.

---

## Core Entities

| Entity | Purpose |
|--------|---------|
| `plans` | Plan catalog and quota limits |
| `vaults` | Tenant root, API key hash, settings, plan, purpose, encryption state |
| `vault_usage` | Current-period usage counters |
| `raw_chunks` | Ingested conversation events |
| `segments` | Ordered groups of raw chunks used as extraction units |
| `session_contexts` | One extracted summary per vault/session |
| `memories` | Durable facts and behavioral memories |
| `memory_embeddings` | Normalized embeddings used for recall |
| `entity_aliases` | Canonical subject and alias mappings |
| `memory_edges` | Directed relationships between memories |
| `extraction_queue` | Work queue for extraction |
| `curation_queue` | Work queue for candidate-memory curation |
| `contradiction_scan_log` | Audit log for contradiction arbitration |

---

## Vaults

Vaults isolate tenant data and configuration.

Important fields:

| Field | Description |
|-------|-------------|
| `id` | Vault UUID |
| `name` | Human-readable name |
| `api_key_hash` | Hash of the vault API key |
| `settings` | JSON settings, including embedding dimensions |
| `plan_id` | `free`, `starter`, or `pro` |
| `purpose` | Optional context injected into extraction |
| `encrypted_dek` | Wrapped data encryption key when encryption is enabled |
| `vault_encryption_enabled` | Per-vault encryption flag |

---

## Raw Chunks and Segments

`raw_chunks` stores conversation messages:

| Field | Description |
|-------|-------------|
| `vault_id` | Tenant boundary |
| `session_id` | Client session key |
| `role` | `user`, `assistant`, or `tool` |
| `content` | Raw or encrypted text |
| `embedding` | Chunk embedding |
| `created_at` | Source timestamp from ingest |
| `processed` | Extraction completion marker |

`segments` groups ordered chunk IDs into extraction units. The ingest route creates segments within the same database transaction as chunk insertion and extraction queueing.

---

## Memories

`memories` is the primary durable knowledge table.

Important fields:

| Field | Description |
|-------|-------------|
| `data` | Fact statement, encrypted when enabled |
| `subject` | Plain subject, or blank when encrypted subject storage is active |
| `subject_encrypted` / `subject_hmac` | Encrypted subject plus lookup HMAC |
| `categories` | Free-form tags |
| `confidence` | Confidence score |
| `score` | Extractor usefulness score |
| `salience` | Importance score |
| `sensitivity` | `low`, `medium`, `high`, or `restricted` |
| `type` | `user_preference`, `user_rule`, `task_pattern`, `workflow`, `project`, `constraint`, `decision`, `system_fact`, or `domain_knowledge` |
| `scope` | `global`, `project`, `task`, or `session` |
| `polarity` | `positive`, `negative`, or `neutral` |
| `status` | `active`, `candidate`, `superseded`, `contradicted`, or `needs_review` |
| `valid_from` / `valid_until` | Optional validity window |
| `parent_id` | Optional parent memory |
| `volatility` | `very_low`, `low`, `medium`, or `high` |
| `source_segment_id` | Segment provenance |
| `evidence` | Structured provenance summary |
| `last_recalled` / `recall_count` | Recall tracking |
| `archived_at` | Soft-delete marker |

Active, non-archived memories are eligible for recall. Candidate memories are excluded from normal memory list and recall paths until promoted.

---

## Embeddings and Recall

`memory_embeddings` stores one normalized embedding per memory. Recall uses this table for vector search and joins back to active memories.

OpenAI embeddings match the storage dimension directly. Other providers can be normalized to the 1536-dimension storage schema.

---

## Alias and Graph Tables

`entity_aliases` maps aliases to canonical subjects. Subject resolution uses normalization, alias lookup, embedding similarity, and optional arbitration.

`memory_edges` stores directed memory relationships:

- `applies_to`
- `part_of`
- `depends_on`
- `supports`
- `contradicts`
- `supersedes`
- `refines`
- `relevant_when`

Recall can include outgoing graph neighbors from semantic matches.

---

## Queues

`extraction_queue` dispatches extraction work. Current ingest enqueues by `segment_id`; legacy single-chunk rows may still exist through `chunk_id`.

`curation_queue` dispatches Pro-plan curation work when curation is enabled.

Both queues use claim timestamps and worker IDs so multiple workers can claim rows safely.

---

## Usage and Limits

`vault_usage` tracks the current billing/usage period:

- `ingest_events`
- `memory_adds`
- `searches`

Plan limits come from `plans.limits` and are surfaced through `GET /stats`.
