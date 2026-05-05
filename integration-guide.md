# Integration Guide

This guide is for developers building their own Persistio integration — without using the [openclaw-persistio](https://github.com/Persistio/openclaw-persistio) plugin. It covers the full memory lifecycle, practical ingest and recall patterns, memory management, multi-vault setups, and error handling.

---

## 1. The Memory Lifecycle

Persistio uses a **two-layer model**:

```
Raw conversation chunks
        ↓  (ingest)
   Stored chunks
        ↓  (async extraction daemon)
  Durable memories
        ↓  (semantic recall)
  Retrieved context
```

**Layer 1 — Raw chunks:** When you call `POST /v1/ingest`, the conversation chunks are stored immediately. This is fast and cheap — no LLM processing happens here.

**Layer 2 — Durable memories:** The extraction daemon picks up ingested chunks and runs an LLM-powered pass to identify facts, preferences, events, and relationships worth remembering. These become structured, searchable memories.

**Why ingest first and recall later?** Ingest is a write operation that happens *after* a conversation. Recall is a read operation that happens *before* the next one. The two-layer model means your agent can finish a session, safely hand off to the extraction daemon, and start the next session with enriched context — without blocking on LLM extraction during the live conversation.

---

## 2. Structuring Your Ingest Calls

### When to call ingest

**End-of-session (recommended):** Call `POST /v1/ingest` once when the conversation ends, sending all chunks in one request. This is the simplest pattern and gives the extraction daemon the full context to work with.

**Streaming during a session:** If sessions are very long, you can call ingest incrementally using the same `session_id`. Persistio deduplicates by session ID — repeated calls append to the existing session rather than creating duplicates.

### How to group chunks

Send all chunks for a single logical conversation in one ingest call. Don't split a single session across multiple `session_id` values — the extraction daemon needs the full thread to derive useful memories.

```json
{
  "session_id": "user-alice-2024-05-01-session-1",
  "chunks": [
    { "role": "user", "content": "..." },
    { "role": "assistant", "content": "..." },
    { "role": "tool", "content": "..." }
  ]
}
```

Include `tool` role chunks when they contain meaningful context (e.g. search results, function outputs). You can omit them if they're noisy.

### session_id best practices

- Make `session_id` **globally unique per session**, not per user. A good pattern: `{user_id}-{date}-{session_number}` or a UUID.
- **Don't reuse** session IDs across different conversations — it will merge unrelated chunks.
- Store session IDs in your own database if you need to append to in-progress sessions.

---

## 3. Recall Patterns

### Prime context at session start

The most important pattern: at the start of every new session, recall relevant memories and prepend them to your system prompt.

```js
// At session start
const { memories } = await recall(query, { top_k: 10 });

const systemPrompt = `
You are a helpful assistant. Here is what you know about this user:

${memories.map(m => `- ${m.data}`).join('\n')}

Use this context to personalise your responses.
`.trim();
```

### Choosing top_k

- Start with `top_k: 5–10` for most use cases
- Increase for agents that need broader context (research assistants, long-running projects)
- Decrease for tight, focused queries where precision matters more than breadth
- There's no penalty for requesting more than exist — Persistio returns however many are available up to `top_k`

### What `include_raw` does

Setting `include_raw: true` returns the raw ingested chunks (actual conversation excerpts) alongside extracted memories. Use this when:
- You want verbatim quotes, not summaries
- Debugging what was ingested
- Your application needs to cite the original conversation

For most production use cases, extracted memories are more useful — they're concise and already structured.

---

## 4. Managing Memories

### Listing

Use `GET /v1/memories` to browse what's been extracted. Useful for building admin UIs, debugging, or auditing what your agent knows.

```bash
# Get most recent memories
GET /v1/memories?limit=20&offset=0

# Filter by category
GET /v1/memories?category=preferences

# Include archived memories
GET /v1/memories?archived=true
```

### Updating

Use `PATCH /v1/memories/:id` to correct or enrich a memory. This is useful when you detect that the extraction daemon produced an inaccurate or outdated memory:

```json
PATCH /v1/memories/mem_abc123
{ "data": "User is based in London, not New York (updated 2024-05)." }
```

### Archiving

`DELETE /v1/memories/:id` performs a **soft delete** — the memory is archived, not permanently removed. Archived memories are excluded from recall by default but can be retrieved with `?archived=true`.

Use archiving to prune stale information without losing history.

### Categories

Categories are free-form string tags. Use them to organise memories and enable filtered recall. Suggested conventions:
- `identity` — name, location, role
- `preferences` — communication style, tool preferences, formats
- `projects` — ongoing work, goals
- `relationships` — people, teams, organisations
- `events` — meetings, deadlines, milestones

---

## 5. Multi-Vault Setup

Persistio is designed for multi-vault use. Each vault has its own isolated memory store, so you can safely run multiple users or agents on a single Persistio instance.

**One vault per user:**
```bash
# Create a vault for each user at registration time
POST /admin/vaults
{ "name": "user-alice" }
→ { "id": "...", "api_key": "pt_alice_key_here" }
```

Store each user's `api_key` in your own database. Use it as the Bearer token when ingesting or recalling for that user.

**One vault per agent:**
If you're running multiple AI agents (e.g. a support agent, a research agent, a personal assistant), give each its own vault to prevent memory bleed between contexts.

**Shared vault for a team:**
For cases where multiple users share a memory pool (e.g. a team knowledge base), use a single vault and differentiate via the `subject` field in memories or via categories.

**Rotating a vault's API key:**
If a key is compromised, rotate it immediately. The old key is invalidated as soon as you call:

```bash
POST /admin/vaults/:id/rotate-key
```

See the [API Reference](api-reference.md) for full details.

---

## 6. Error Handling

### Common status codes

| Code | Meaning |
|------|---------|
| `202` | Accepted — async operation queued (ingest, extract) |
| `400` | Bad request — check your request body |
| `401` | Unauthorised — invalid or missing API key |
| `404` | Not found — memory or job ID doesn't exist |
| `429` | Rate limited — back off and retry |
| `500` | Server error — retry with exponential backoff |

### Retry strategy

- `429` and `5xx` errors are safe to retry
- Use **exponential backoff**: start at 1s, double each attempt, cap at ~30s
- Add jitter to avoid thundering herd if you're running many agents concurrently
- `4xx` errors (except 429) indicate a problem with your request — don't retry without fixing the input

### Ingest is idempotent

If an ingest call fails mid-way and you retry with the same `session_id`, Persistio deduplicates the chunks. It's safe to retry ingest calls.

---

## 7. A Note on Extraction Timing

The extraction daemon runs **asynchronously** in the background. There is a delay — typically a few seconds — between calling `POST /v1/ingest` and those memories being available via `POST /v1/recall`.

For most applications this is fine: you ingest at the end of session N, and recall at the start of session N+1, by which time extraction has long since completed.

**For time-sensitive use cases** — e.g. you need memories available immediately within the same session — trigger extraction manually:

```bash
# Step 1: Ingest
POST /v1/ingest
→ 202 Accepted

# Step 2: Trigger extraction
POST /v1/extract
→ { "job_id": "job_xyz789" }

# Step 3: Poll until complete
GET /v1/jobs/job_xyz789
→ { "status": "running", ... }
→ { "status": "completed", "memories_created": 4 }

# Step 4: Recall
POST /v1/recall
→ { "memories": [...] }
```

Poll `GET /v1/jobs/:id` every 1–2 seconds. The job typically completes within 3–10 seconds depending on session length and LLM latency. Once `status` is `"completed"`, your memories are available.

---

## Further Reading

- [Getting Started](getting-started.md) — end-to-end walkthrough
- [API Reference](api-reference.md) — full endpoint specs
- [Persistio/openclaw-persistio](https://github.com/Persistio/openclaw-persistio) — drop-in plugin for OpenClaw users
