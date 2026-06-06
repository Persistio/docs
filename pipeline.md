# Persistio Pipeline

## End-to-End Flow

The implemented pipeline is:

1. client sends `POST /v1/ingest`
2. API validates, rate-limits, embeds, and stores raw chunks
3. API groups chunks into `segments`
4. API enqueues each segment in `extraction_queue`
5. extraction worker claims queue rows
6. worker reconstructs and decrypts the segment conversation
7. worker creates or reuses `session_contexts`
8. worker extracts aliases and facts with the extractor model
9. worker filters, canonicalizes subjects, embeds facts, and deduplicates into `memories`
10. worker runs contradiction scans on touched memories
11. after all extraction rows for the segment are settled, worker marks the segment ready and, if curation is enabled for the vault, enqueues `curation_queue`
12. curation worker turns candidate memories into active graph updates

## Ingest Stage

`POST /v1/ingest` is implemented in [`src/routes/ingest.ts`](https://github.com/Persistio/server/blob/main/src/routes/ingest.ts).

For each chunk the API:

- authenticates the vault
- consumes `ingest_events` quota
- requires a `timestamp` ISO 8601 datetime with UTC `Z` or an explicit offset from the client
- encrypts `content` if vault encryption is active
- writes the stored raw chunk payload to the configured raw chunk storage backend
- generates a chunk embedding with the configured embedder
- inserts the row into `raw_chunks` with `blob_store`, `blob_key`, and `created_at` set to that source timestamp

The timestamp should reflect when the original conversation turn occurred. It is not generated from request receipt time, because extraction uses source turn timestamps to resolve relative dates and to populate `memories.source_timestamp`.

After all chunks are inserted, the route builds segments and enqueues extraction work inside the same DB transaction.

## Segmentation Logic

Segmentation is local to the just-ingested batch, not the entire session history.

`buildSegments()` uses:

- minimum segment size: `3`
- maximum segment size: `40`
- split threshold: `SEGMENTATION_THRESHOLD`, default `0.75`

Algorithm:

- start with the first chunk
- compare each next chunk to the last chunk of the current segment using cosine similarity
- split when similarity drops below the threshold and both sides can still satisfy the minimum size
- also split when the segment already has `40` chunks
- if the final segment would be smaller than `3`, merge it back into the previous segment

Each segment stores:

- ordered `chunk_ids`
- a `context` preview equal to the first non-empty chunk content, truncated to 280 chars

That preview is later replaced in practice by richer session context during extraction.

## Extraction Queue Mechanics

Extraction work is stored in `extraction_queue`.

Claim loop behavior in `extraction-worker.ts`:

- rows with `claimed_at` older than 10 minutes are released first
- the worker claims rows with `FOR UPDATE SKIP LOCKED`
- rows are claimable only when every referenced raw chunk has a `blob_key`
- claim size is `EXTRACTION_BATCH_SIZE`
- each claimed row is stamped with `claimed_at` and `claimed_by`
- queue rows are ordered by `priority DESC, enqueued_at ASC`, so normal streaming work runs before bulk backfills
- per-row processing runs concurrently under `EXTRACTION_WORKER_CONCURRENCY`
- fact text embeddings are sent through `embedBatch()` once per segment instead of one API call per fact

The worker supports two queue shapes:

- legacy single-chunk rows via `chunk_id`
- segment-based rows via `segment_id`

The current ingest path enqueues by segment.

## Worker Polling Loop

The extraction worker:

- starts automatically whenever `PERSISTIO_MODE !== api`
- loops forever
- sleeps `EXTRACTION_INTERVAL_MS` between iterations
- also accepts `run-once` messages from the API process for manual triggering

On `run-once`, the worker optionally scopes processing to one vault and reports in-memory job status back to the main process.

## Session Context and Alias Extraction

Before fact extraction, the worker assembles the segment conversation as:

```text
[2026-05-16T12:34:56.000Z] role: decrypted content
[2026-05-16T12:35:10.000Z] role: decrypted content
...
```

It then:

- looks up an existing `session_contexts` row for `(vault_id, session_id)`
- if missing, calls `extractSessionContext()` to create a short noun-phrase summary
- stores that summary back in `session_contexts`

When the session context is newly created, the worker also calls `extractSessionAliases()` and upserts normalized alias pairs into `entity_aliases`.

## Prompt Header Construction

The extractor prompt header may include:

- a sanitized session summary
- the vault `purpose`
- known canonical subjects and aliases from `entity_aliases`

The header explicitly warns the model that those fields are untrusted data, not instructions.

## Extraction Model

Extraction is implemented by [`src/services/extractor.ts`](https://github.com/Persistio/server/blob/main/src/services/extractor.ts).

Default configuration:

- base URL: `https://api.openai.com/v1`
- model: `gpt-4o-mini`
- prompt file: `prompts/extractor.txt`

The service uses an OpenAI-compatible chat completion client, so other compatible providers can be used by changing the base URL and key.

The extractor has two model roles:

- `extraction`: routine fact extraction, session context, and session alias extraction.
- `escalation`: conflict and contradiction arbitration. Ambiguous subject arbitration uses the extraction role.

`EXTRACTION_*` and `ESCALATION_*` settings fall back to legacy `EXTRACTOR_*` settings when omitted. When switching either role to a different provider, set that role's base URL, API key, and model together. Provider-neutral AI budgets apply to extraction, escalation, and curation regardless of provider; role-specific circuit-breaker logs use `extractor.extraction` and `extractor.escalation`.

The extractor must produce JSON facts with:

- `fact`
- `subject`
- `score`
- `salience`
- `sensitivity`
- `type`
- `scope`
- `polarity`
- `status`
- `volatility`
- `evidence`
- `valid_from`
- `valid_until`

## Extraction Filtering

The worker applies multiple filters before writing memories:

### Score filter

Only facts with `score >= EXTRACTION_SCORE_THRESHOLD` survive. Default threshold: `5`.

### Secret-pattern filter

Facts matching secret-like patterns are dropped before later processing.

### Sensitivity filter

Facts with `sensitivity === 'restricted'` are dropped before embedding and storage.

Result:

- restricted or obviously secret material should not become stored memories

## Subject Resolution

Subject resolution is multi-tiered.

### Tier 1: text normalization and Levenshtein

`resolveSubjectTier1()` compares the extracted subject against known canonical subjects and aliases after normalization. This is free and fast.

### Tier 2: embedding similarity

If text matching fails, the subject itself is embedded and compared to cached canonical embeddings in `entity_aliases`.

Thresholds:

- high confidence: `SUBJECT_EMBED_HIGH_THRESHOLD`, default `0.92`
- ambiguous: `SUBJECT_EMBED_LOW_THRESHOLD`, default `0.80`

### Tier 3: LLM arbitration

If Tier 2 is ambiguous, the extractor model is asked whether the new subject refers to the existing canonical subject.

New canonical subjects store an embedding for future matches. Subjects resolved to an existing canonical through Tier 2 or Tier 3 are stored as aliases without replacing the canonical embedding, so later exact or text-normalized matches stay on the free path.

## Deduplication and Insert/Update Rules

Memory writes flow through [`src/services/dedup.ts`](https://github.com/Persistio/server/blob/main/src/services/dedup.ts).

Decision order:

1. if the memory is being inserted as a `candidate`, insert directly
2. if there is an exact `hash` match in the vault, update the existing memory’s evidence and metadata
3. otherwise find the best same-subject semantic match
4. if similarity `> 0.90`, update in place
5. if similarity `>= 0.80`, apply escalation routing:
   - routine moderate overlap is inserted as a separate memory without model arbitration
   - escalation arbitration is reserved for agent-context signal, low extraction confidence on the new candidate, strong semantic overlap, possible conflict, possible supersession, or existing memories that are candidates, low-confidence, or already marked `needs_review`
   - if escalation is required but no extractor is available, the safe fallback is `needs_review`
6. escalation arbitration requests are collected per extraction job and sent through `arbitrateConflictsBatch()` before memory writes; dedup reuses the precomputed decision only when the same existing memory is still the best match. If multiple candidates preflight to the same existing memory, only the earliest precomputed decision is reused and later candidates re-arbitrate live after prior writes.
7. when escalation arbitration runs:
   - `merge` -> update in place
   - `discard_new` -> skip insert
   - `supersede_old` -> mark existing memory `superseded`, then insert new
   - `needs_review` -> mark existing memory `needs_review`, then insert new
8. otherwise insert a new memory

The dedup path also:

- enforces `memories_max` and `memory_adds` quota when inserting
- writes `memory_embeddings`
- stores `source_segment_id` for pipeline provenance

## Plan-Specific Behavior

The extraction worker computes the current curation capability for the vault and changes behavior:

- vaults with curation enabled write extracted memories with `status='candidate'` only when `CURATOR_AUTO_RUN=true`
- if curation is disabled, vaults keep the extractor's status directly so memories remain recallable
- vaults without curation capacity keep the extractor's status directly, which is typically `active`

This is the entry point into the curation pipeline.

## Contradiction Scanning

After a batch finishes, the worker calls `scanForContradictions()` on affected memory ids.

Behavior:

- disabled unless `CONTRADICTION_SCAN_ENABLED=true`
- for each new memory, fetches similar memories above `CONTRADICTION_SCAN_MIN_SIMILARITY`
- decrypts both facts
- skips identical text
- asks the extractor model to arbitrate
- writes status updates:
  - `supersede_old` -> candidate memory becomes authoritative, old one marked `contradicted`
  - `discard_new` -> new one marked `contradicted`
  - `needs_review` -> both marked `needs_review`
  - `merge` -> new/current row `superseded`, candidate confidence boosted
- logs every decision in `contradiction_scan_log`

## Curation Queue Entry

When extraction settles for a segment:

- the extraction queue row is deleted
- the corresponding `raw_chunks.processed` values are set to `true`
- the worker checks that no other extraction queue rows remain for the segment
- every segment chunk must be processed or covered by a terminal extraction dead-letter row
- `segments.curation_ready_at` is set once when that invariant holds
- if `CURATOR_AUTO_RUN=true` and the plan enables curation, the worker inserts `(vault_id, segment_id)` into `curation_queue` idempotently and sets `segments.curation_enqueued_at`

The curator never starts from a partially extracted segment; the post-curation candidate check remains defense-in-depth telemetry rather than the primary queueing mechanism.

## Curation Worker

The curation worker polls `curation_queue` every `CURATION_INTERVAL_MS` and claims up to `CURATION_BATCH_SIZE` rows with `FOR UPDATE SKIP LOCKED`.

Curator model execution is gated by plan cadence, budget, and queue eligibility. Curation remains queue-polling based and closely tied to extraction output.

For each job it loads:

- the vault encryption context
- the segment’s stored context
- all `candidate` memories from that segment
- all active memories whose subjects match the candidate subjects

It then calls the curator model with:

- candidate memories
- existing active memories for matched subjects
- raw segment conversation

## Curator Output

The curator returns JSON actions:

- `nodes_to_create`
- `nodes_to_update`
- `edges_to_create`
- `nodes_to_archive`
- `discarded_candidates`

Aliases like `M1`, `C1` are used in the prompt and resolved back to real ids in the worker.

## Applying Curator Actions

The curation worker applies actions transactionally and logs each success or failure in `curation_action_log`.

Supported effects:

- create new active memories
- update existing candidate or active memories
- create `memory_edges`
- archive nodes
- discard candidate nodes
- archive untouched candidates that are near-duplicates of active same-subject memories, mark them `superseded`, log `archive_duplicate`, and merge their evidence into the active memory
- promote updated candidate nodes to `active`
- promote untouched remaining candidates from `candidate` to `active`

## Dead Letter Handling

### Extraction dead letter

Extraction has two failure paths:

- normal processing errors increment `retry_count`, clear the claim, and leave the row queued until `MAX_EXTRACTION_RETRIES`, then move the row to `extraction_dead_letter`
- repeated rate-limit failures use exponential backoff and, after `MAX_EXTRACTION_RATE_LIMIT_RETRIES`, move the row to `extraction_dead_letter`

Each dead-letter row stores:

- `vault_id`
- `chunk_id` or `segment_id`
- `retry_count`
- `last_error`

### Curation dead letter

Curation checks `retry_count` before running a job. If it reaches `MAX_CURATION_RETRIES` the worker:

- inserts a row into `curation_dead_letter`
- deletes the `curation_queue` row

Other curation failures increment `retry_count`, persist `last_error`, and release the claim.

## Circuit Breakers and Resilience

The extractor and curator services each have a process-wide circuit breaker keyed to their shared API credentials.

Important current behavior:

- the breaker opens on auth failures (`401`/`403`), not on `429`
- while open, jobs are released back to the queue with `last_error` updated
- provider-neutral AI request/token budgets are enforced in-process per vault and role for extraction, escalation, and curation
- when a background job exhausts local AI budget, the worker defers it by setting queue `available_at`; this is distinct from upstream provider `429` retry handling

## Staleness Maintenance

At the end of extraction batches, the worker runs `archiveStaleMemories()`:

- archives memories older than `MEMORY_ARCHIVE_TTL_DAYS` based on recall/update timestamps
- decays confidence every `CONFIDENCE_DECAY_INTERVAL_DAYS`
- auto-archives memories whose confidence drops to `<= 0` and salience is below `CONFIDENCE_DECAY_AUTO_ARCHIVE_SALIENCE_THRESHOLD`
