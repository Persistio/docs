# Persistio Architecture

## What Persistio Is

Persistio is a self-hosted memory service for AI agents. It stores raw conversational chunks, extracts durable memories from those chunks, optionally curates those memories into a behavioral graph, and exposes retrieval and management APIs for client applications such as the OpenClaw plugin in `packages/plugin`.

At a high level Persistio provides:

- multi-tenant vaults with per-vault API keys
- semantic ingest and recall over pgvector-backed embeddings
- an asynchronous extraction pipeline that turns conversation segments into memories
- optional curation for curator-enabled vaults that promotes extracted candidate memories into an explicit memory graph
- quota tracking, rate limiting, encryption-at-rest for vault content, and telemetry instrumentation

## Runtime Topology

Persistio has three runtime modes controlled by `PERSISTIO_MODE`:

- `api`: runs Fastify HTTP routes only
- `worker`: runs the extraction worker loop and optional curation worker, but does not register the product API
- `combined`: runs both in one process

Mode detection happens in [`src/index.ts`](https://github.com/Persistio/server/blob/main/src/index.ts):

- `shouldStartWorker = PERSISTIO_MODE !== 'api'`
- `shouldRegisterFullApi = PERSISTIO_MODE !== 'worker'`
- `shouldStartCurationWorker = shouldStartWorker && CURATOR_AUTO_RUN`

All modes expose `/health`. Only non-`worker` modes run database migrations on startup and register the application routes.

## API and Worker Split

The main architectural split is between the synchronous API path and the asynchronous extraction/curation path.

### API container responsibilities

- authenticate vault and admin requests
- accept raw chunk ingest through `POST /v1/ingest`
- write `raw_chunks`, `segments`, and `extraction_queue`
- serve recall, memory CRUD, stats, and admin vault routes
- optionally trigger an immediate worker pass through `POST /v1/extract`
- keep current-period vault usage counters for quota enforcement and live stats
- durably emit closed-period usage events for the App to store as billing history

### Worker container responsibilities

- poll `extraction_queue`
- decrypt and assemble conversation segments
- call the extractor LLM and embedding provider
- resolve canonical subjects
- deduplicate or insert memories
- scan for contradictions
- enqueue curation for curator-enabled candidate memories when enabled
- poll `curation_queue` and apply curator actions

The split keeps CPU-bound and network-heavy extraction work off the request path. The API returns `202 Accepted` quickly, while the worker handles embedding, extraction, and curation in the background.

## Shared State Between Containers

API and worker containers coordinate exclusively through PostgreSQL.

Shared tables include:

- `vaults` for tenant metadata, plan, keys, and encryption settings
- `raw_chunks` for ingested conversation records
- `segments` for grouped chunk batches
- `extraction_queue` for extraction work dispatch
- `memories` and `memory_embeddings` for durable memory storage
- `curation_queue` and `curation_action_log` for the curation pipeline

The critical handoff is:

1. API writes `raw_chunks`
2. API groups those chunks into `segments`
3. API inserts one queue row per segment into `extraction_queue`
4. worker claims queue rows with `FOR UPDATE SKIP LOCKED`
5. worker writes or updates `memories`
6. worker optionally enqueues `curation_queue` entries for curator-enabled vaults

## Core Dependencies

Persistio depends on:

- Fastify for the HTTP server
- PostgreSQL plus `pgvector` for relational storage and vector similarity
- `pg` and `pgvector/pg` for database access
- Zod for request and environment validation
- telemetry hooks for tracing and metrics
- OpenAI-compatible chat completions for extraction and curation
- configurable embedding providers for vector generation
- configurable key wrapping when encryption is enabled

## Storage Model

Persistio stores two different kinds of data:

- raw conversational evidence in `raw_chunks`, `segments`, and `session_contexts`
- distilled durable knowledge in `memories`, `memory_embeddings`, `entity_aliases`, `memory_edges`, and queue/log tables

Embeddings are normalized to the 1536-dimension storage schema. OpenAI natively matches that shape; Ollama embeddings are zero-padded to fit.

## Extraction and Curation Architecture

Extraction is implemented by [`src/daemon/extraction-worker.ts`](https://github.com/Persistio/server/blob/main/src/daemon/extraction-worker.ts).

Key behaviors:

- stale queue claims older than 10 minutes are released
- work is claimed in batches with `FOR UPDATE SKIP LOCKED`
- per-batch concurrency is limited by `EXTRACTION_WORKER_CONCURRENCY`
- session context and entity aliases are derived once per session
- facts are filtered by score, secret patterns, and sensitivity before insertion
- curator-enabled vaults write `candidate` memories first, then the curation worker promotes or mutates them

Curation is implemented by [`src/daemon/curation-worker.ts`](https://github.com/Persistio/server/blob/main/src/daemon/curation-worker.ts).

Key behaviors:

- only runs when `CURATOR_AUTO_RUN=true`
- claims `curation_queue` rows in batches
- loads candidate memories for a segment plus active memories for matching subjects
- asks the curator model for node/edge actions
- records every applied or failed action in `curation_action_log`
- promotes untouched candidates from `candidate` to `active`

The current curation worker is event-adjacent: extraction enqueues eligible curation work and the worker drains the queue behind plan cadence, budget, and queue eligibility gates.

## Observability

Persistio emits:

- request latency histogram
- recall latency histogram
- ingest chunk counter
- extraction job counter
- extraction lag histogram
- embedding duration histogram
- gauges for memory totals and extraction queue depth

Tracing helpers in `telemetry.ts` wrap ingest, recall, embedding, extraction, and deduplication spans. Telemetry export is optional and provider-selectable at runtime.

## Security and Isolation

Vaults are the tenancy boundary. Each vault has:

- a unique API key hash
- a plan id (`unlimited` by default on self-host installs)
- optional per-vault wrapped DEK
- separate current-period quota accounting in `vault_usage`

Closed-period usage history is not stored long-term in platform Postgres. The platform can export durable period-close events for downstream billing/reporting systems.

When encryption is enabled:

- facts and session context are AES-256-GCM encrypted with a per-vault DEK
- the DEK is wrapped by the configured key provider
- subjects are stored encrypted plus HMACed for exact-match lookup

## Self-Host Topology

The checked-in [`docker-compose.yml`](https://github.com/Persistio/server/blob/main/docker-compose.yml) is the simplest self-host shape: one `persistio` container plus one `pgvector/pgvector:pg17` PostgreSQL container. For split-role self-host runs, run separate API and worker containers against the same Postgres instance and the same environment configuration.
