# Persistio Architecture

Persistio is a self-hosted memory service for AI agents. It stores raw conversation chunks, extracts durable memories, optionally curates those memories into a behavioral graph, and exposes retrieval and management APIs for client applications such as the OpenClaw plugin.

---

## Runtime Modes

`PERSISTIO_MODE` controls process behavior:

| Mode | Behavior |
|------|----------|
| `api` | Runs Fastify HTTP routes only |
| `worker` | Runs extraction and curation workers; product API routes are not registered |
| `combined` | Runs API and worker in one process |

All modes expose `/health`. Non-worker modes register the product API and run migrations on startup.

---

## API and Worker Split

The API path is synchronous and handles:

- vault and admin authentication
- `POST /v1/ingest`
- `raw_chunks`, `segments`, and `extraction_queue` writes
- recall, memory CRUD, stats, and admin vault routes
- manual extraction triggers through `POST /v1/extract`

The worker path is asynchronous and handles:

- polling `extraction_queue`
- decrypting and assembling conversation segments
- session context and alias extraction
- extractor model calls
- fact filtering, subject resolution, embedding, and deduplication
- contradiction scans
- optional Pro-plan curation

This split keeps embedding, extraction, and curation work off the request path. Ingest returns `202 Accepted` after queueing work.

---

## Shared State

API and worker containers coordinate through PostgreSQL.

Core tables include:

- `vaults`
- `raw_chunks`
- `segments`
- `extraction_queue`
- `memories`
- `memory_embeddings`
- `entity_aliases`
- `memory_edges`
- `curation_queue`
- `contradiction_scan_log`

The main handoff is:

1. API writes `raw_chunks`
2. API groups chunks into `segments`
3. API queues one extraction job per segment
4. Worker claims queue rows with `FOR UPDATE SKIP LOCKED`
5. Worker writes or updates `memories`
6. Worker optionally enqueues curation for Pro-plan vaults

---

## Extraction Pipeline

For each segment, the extraction worker:

1. reconstructs the conversation with timestamps and roles
2. creates or reuses a `session_contexts` summary
3. extracts aliases and stores canonical subject mappings
4. asks the extractor model for durable facts
5. filters low-score, secret-like, and restricted facts
6. resolves subjects with normalization, embedding similarity, and optional arbitration
7. deduplicates against existing memories
8. writes memory rows and embeddings
9. scans for contradictions when enabled

Extraction settings allow separate provider configuration for routine extraction and escalation arbitration.

---

## Recall

Recall embeds the query and returns active, non-archived memories from `memory_embeddings`.

It can also:

- expand through directed outgoing `memory_edges`
- return source-linked evidence chunks with `include_evidence`
- return raw chunk semantic matches with `include_raw`
- return a grouped prompt bundle with `?format=bundle`

Bundle mode groups by memory `type`, with global user rules separated for agent prompt construction.

---

## Curation

When `CURATOR_AUTO_RUN=true`, Pro-plan vaults can route extracted memories through curation.

The curation worker:

- claims `curation_queue` rows
- loads candidate memories, active same-subject memories, and raw segment context
- asks the curator model for node and edge actions
- creates, updates, promotes, archives, or links memories
- logs applied and failed actions

This produces an explicit memory graph through `memory_edges`.

---

## Security and Isolation

Vaults are the tenancy boundary. Each vault has:

- a unique API key hash
- plan metadata
- quota accounting
- optional per-vault encryption
- isolated raw chunks, memories, aliases, edges, and queues

When encryption is active, memory facts and session context are encrypted with a per-vault DEK. Subjects can be stored encrypted with an HMAC for exact-match lookup.

---

## Observability

Persistio emits request latency, recall latency, ingest chunk counts, extraction job counts, extraction lag, embedding duration, memory totals, and queue depth metrics. OpenTelemetry spans wrap ingest, recall, embedding, extraction, and deduplication. Azure Monitor export is enabled when `APPLICATIONINSIGHTS_CONNECTION_STRING` is configured.

---

## Deployment Shape

The simplest local shape is Docker Compose with one Persistio container and one `pgvector` PostgreSQL container. Split-role deployments run separate API and worker containers against the same PostgreSQL database and environment configuration.
