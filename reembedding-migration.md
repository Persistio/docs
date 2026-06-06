# Qwen3 Re-embedding Migration

Use this only when moving an existing database from a different embedding model or dimension to Ollama `qwen3-embedding:0.6b` with 1024-dimensional pgvector columns.

Fresh installs do not need this script. Set:

```bash
EMBEDDER_PROVIDER=ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_EMBEDDING_MODEL=qwen3-embedding:0.6b
STORAGE_EMBEDDING_DIMENSIONS=1024
```

For TEI-hosted Qwen embeddings, use the same storage dimensions with:

```bash
EMBEDDER_PROVIDER=tei
TEI_BASE_URL=http://tei:80
TEI_EMBEDDING_MODEL=text-embeddings-inference
STORAGE_EMBEDDING_DIMENSIONS=1024
```

## Required Preflight

1. Take a PostgreSQL backup or provider snapshot.
2. Stop API ingress, recall traffic, extraction workers, and curation workers.
3. Confirm `extraction_queue`, `curation_queue`, and active jobs are empty.
4. Confirm Ollama is private/internal and has `qwen3-embedding:0.6b` pulled before the maintenance window.
5. Keep the old database snapshot until row counts, indexes, vector dimensions, and sample recall are validated.

The normal schema migration refuses to change vector dimensions while stored embeddings exist. That is intentional: changing `raw_chunks.embedding`, `memories.embedding`, `memory_embeddings.embedding`, and `entity_aliases.embedding` requires re-embedding the stored data.

## Scripted Staging-table Path

Run from the repository root:

```bash
export DATABASE_URL='postgresql://...'
export OLLAMA_BASE_URL='http://ollama:11434'
export OLLAMA_EMBEDDING_MODEL='qwen3-embedding:0.6b'
export STORAGE_EMBEDDING_DIMENSIONS=1024

npm run migrate:qwen3-embeddings -- --dry-run
npm run migrate:qwen3-embeddings -- --confirm-maintenance-window
```

The script:

- acquires an advisory migration lock
- refuses encrypted vaults because it cannot decrypt memory data outside the app runtime
- refuses active extraction/curation queues and active jobs
- computes Qwen embeddings into staging tables first
- validates staging row counts
- drops vector indexes, resizes vector columns to 1024, swaps staged embeddings, and recreates indexes inside a transaction where possible
- preserves memory metadata timestamps; only embedding columns and `memory_embeddings.embedded_at` change

On the next application startup, migration `034_configurable_embedding_dimensions.sql` detects that vector columns already match `STORAGE_EMBEDDING_DIMENSIONS=1024`, returns without rewriting stored vectors, and records the migration as applied.

For raw chunks, the script can read `raw_chunks.content` when the legacy column still exists, or blob-backed chunks after `scripts/finalize-raw-chunk-blob-migration.sql` has dropped that column. Set `RAW_CHUNK_LOCAL_DIR` for `local` blobs, or set `RAW_CHUNK_BLOB_CONTAINER` with either `AZURE_STORAGE_CONNECTION_STRING` or `AZURE_STORAGE_ACCOUNT_NAME` for `azure_blob` chunks.

## Fresh-database Path

For encrypted vaults or very large datasets, prefer a fresh database:

1. Provision a new database with `STORAGE_EMBEDDING_DIMENSIONS=1024`.
2. Start Persistio with private Ollama/Qwen embeddings.
3. Replay or import raw data through the application so decryption, blob reads, and embedding writes happen through normal runtime code.
4. Validate recall and usage telemetry.
5. Cut over `DATABASE_URL`.

## Validation

Run SQL checks after the migration:

```sql
SELECT COUNT(*) FROM raw_chunks WHERE embedding IS NULL;
SELECT COUNT(*) FROM memories WHERE embedding IS NULL;
SELECT COUNT(*) FROM memory_embeddings WHERE embedding IS NULL;
SELECT COUNT(*) FROM entity_aliases WHERE embedding IS NULL;

SELECT vector_dims(embedding), COUNT(*) FROM raw_chunks WHERE embedding IS NOT NULL GROUP BY 1;
SELECT vector_dims(embedding), COUNT(*) FROM memories WHERE embedding IS NOT NULL GROUP BY 1;
SELECT vector_dims(embedding), COUNT(*) FROM memory_embeddings WHERE embedding IS NOT NULL GROUP BY 1;
SELECT vector_dims(embedding), COUNT(*) FROM entity_aliases WHERE embedding IS NOT NULL GROUP BY 1;

SELECT indexname FROM pg_indexes WHERE tablename IN ('raw_chunks', 'memory_embeddings', 'entity_aliases');
```

Then run representative recall requests for several vaults and compare results against the backup or known-good baseline.

## Rollback

If the script fails before commit, PostgreSQL rolls back the column swap. If it commits and validation fails, restore the preflight backup/snapshot or cut back to the previous database. Do not try to cast 1024-dimensional vectors back to the old dimension in place.
