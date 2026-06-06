# Integration Guide

Persistio exposes a vault-scoped HTTP API for durable agent memory.

## Memory Lifecycle

```text
conversation chunks
  -> segments
  -> async extraction worker
  -> durable memories and embeddings
  -> semantic recall
  -> retrieved context
```

Use `POST /v1/ingest` as the write path and `POST /v1/recall` as the read path. Ingest stores raw chunks and queues extraction; recall retrieves active memory and optional evidence.

## Ingest Shape

Send chunks for one logical conversation under one `session_id`. Each chunk requires a source `timestamp`.

```json
{
  "session_id": "user-alice-2026-05-19-session-1",
  "chunks": [
    {
      "role": "user",
      "content": "I prefer concise answers.",
      "timestamp": "2026-05-19T12:00:00.000Z"
    }
  ]
}
```

Batch enough adjacent messages to provide context for extraction, and filter noisy tool output before ingest.

## Recall Shape

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
    min_similarity: 0.3,
    include_pending: true,
    mode: 'agent'
  })
});

const { bundle, related_bundle } = await res.json();
```

Use bundle recall when assembling an agent prompt. Keep related context supplemental rather than mixing it into the direct semantic result budget.

## OpenClaw

For OpenClaw, use the [official Persistio plugin](https://github.com/Persistio/openclaw-persistio/blob/main/README.md). Configure the plugin `baseURL` to point at your self-host server and use a vault API key.
