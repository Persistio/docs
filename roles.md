# API and Worker Roles

## Modes

`PERSISTIO_MODE` accepts:

- `api`
- `worker`
- `combined`

Default: `combined`

## `PERSISTIO_MODE=api`

What runs:

- Fastify server
- `/health`
- all product API routes
- admin routes
- in-memory extraction job store

What does not run:

- extraction worker thread
- curation worker thread

Implication:

- `POST /v1/ingest` still queues work in Postgres
- `POST /v1/extract` creates an in-memory job record but cannot complete it successfully because no extraction worker is attached

Use this mode when:

- you deploy a dedicated HTTP/API container
- background extraction is handled by separate worker containers

## `PERSISTIO_MODE=worker`

What runs:

- extraction worker thread and loop
- optional curation worker thread and loop when `CURATOR_AUTO_RUN=true`
- `/health`

What does not run:

- ingest, recall, memory CRUD, stats, jobs, and admin routes
- startup migrations

Use this mode when:

- you deploy dedicated background workers
- you want to scale API and extraction separately

Important:

- worker mode still needs full DB and model credentials because it performs extraction, embedding, deduplication, contradiction scanning, and optional curation

## `PERSISTIO_MODE=combined`

What runs:

- full API
- extraction worker
- optional curation worker

Use this mode when:

- running locally
- deploying a single small instance
- debugging end-to-end behavior without separate containers

Tradeoff:

- the API and background work share one process and one Node runtime, so CPU/network contention is higher than in split deployments

## Route Ownership

Routes registered only when mode is not `worker`:

- `GET /health`
- `POST /v1/ingest`
- `POST /v1/recall`
- `GET /v1/memories`
- `POST /v1/memories`
- `GET /v1/memories/:id`
- `PATCH /v1/memories/:id`
- `DELETE /v1/memories/:id`
- `POST /v1/extract`
- `GET /v1/jobs/:id`
- `GET /stats`
- `/admin/vaults` CRUD routes

Route registered in every mode:

- `GET /health`

## Worker-Owned Services

The worker side owns:

- `ExtractorService`
- `CuratorService`
- subject resolution and alias maintenance
- deduplication and conflict arbitration
- contradiction scanning
- stale-memory archiving
- queue claim/retry/dead-letter logic

## Why the Split Exists

The split is not cosmetic. It isolates expensive and failure-prone work from latency-sensitive HTTP handling.

Reasons:

- embedding calls are network-bound and can be slow
- extractor/curator LLM calls are slower and less predictable than CRUD routes
- queue workers need retry and backoff behavior that should not block request threads
- scaling API replicas and worker replicas independently is simpler than scaling one combined service
- operationally, a bad extractor dependency should not make the API path unusable

## Local Development

### Single-process local run

Use `combined` to test everything in one process.

Typical requirements:

- PostgreSQL with pgvector
- `DATABASE_URL`
- `ADMIN_API_KEY`
- embedder configuration
- extractor credentials
- curator credentials if `CURATOR_AUTO_RUN=true`

### Split local run

Run two server processes against the same Postgres database:

1. API process with `PERSISTIO_MODE=api`
2. Worker process with `PERSISTIO_MODE=worker`

Both processes must share:

- `DATABASE_URL`
- encryption config
- embedder config
- extractor config
- curator config when enabled

## Docker Compose Note

The checked-in [`docker-compose.yml`](https://github.com/Persistio/server/blob/main/docker-compose.yml) runs a single `persistio` service plus Postgres, which implies `combined` mode unless overridden in `.env`.

For a split-role container deployment, define separate API and worker services using the same image but different `PERSISTIO_MODE` values.
