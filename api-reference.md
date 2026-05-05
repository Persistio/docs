# Persistio API Reference

Base URL: `https://your-persistio-instance`

**Auth:**
- Vault endpoints → `Authorization: Bearer pt_your_api_key_here`
- Admin endpoints → `X-Admin-Key: adm_your_admin_key_here`

---

## Health

### `GET /health`

Returns the server health status including a live database ping. No authentication required by default. If `HEALTH_API_KEY` is configured on the server, requests must include `X-Health-Key: <value>`.

**Response:** `200 OK`

```json
{
  "status": "ok",
  "version": "0.1.0",
  "db": "ok",
  "db_latency_ms": 12,
  "uptime_s": 34
}
```

Returns `503` with `"status": "degraded"` if the database check fails.

**curl**
```bash
curl https://your-persistio-instance/health
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/health');
const data = await res.json();
console.log(data.status); // 'ok'
```

**Python**
```python
import httpx
res = httpx.get('https://your-persistio-instance/health')
print(res.json()['status'])  # 'ok'
```

---

## Ingest

### `POST /v1/ingest`

Ingest a conversation session. Chunks are stored immediately; memory extraction happens asynchronously.

**Auth:** Bearer token

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | string | ✓ | Unique identifier for this conversation session |
| `chunks` | array | ✓ | Array of conversation chunks |
| `chunks[].role` | string | ✓ | `"user"`, `"assistant"`, or `"tool"` |
| `chunks[].content` | string | ✓ | The message content |

**Response:** `202 Accepted`

**curl**
```bash
curl -X POST https://your-persistio-instance/v1/ingest \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "session_id": "session-2024-001",
    "chunks": [
      { "role": "user", "content": "My name is Alice." },
      { "role": "assistant", "content": "Nice to meet you, Alice." }
    ]
  }'
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/ingest', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer pt_your_api_key_here',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    session_id: 'session-2024-001',
    chunks: [
      { role: 'user', content: 'My name is Alice.' },
      { role: 'assistant', content: 'Nice to meet you, Alice.' },
    ],
  }),
});
// 202 Accepted
```

**Python**
```python
import httpx

res = httpx.post(
    'https://your-persistio-instance/v1/ingest',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
    json={
        'session_id': 'session-2024-001',
        'chunks': [
            {'role': 'user', 'content': 'My name is Alice.'},
            {'role': 'assistant', 'content': 'Nice to meet you, Alice.'},
        ],
    },
)
# res.status_code == 202
```

---

## Recall

### `POST /v1/recall`

Retrieve memories semantically relevant to a query. Use this to prime your agent's context at the start of a session.

**Auth:** Bearer token

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | ✓ | Natural language query |
| `top_k` | integer | | Number of memories to return (default: 10) |
| `include_raw` | boolean | | If true, includes raw ingested chunks alongside extracted memories |

**Response:** `200 OK`

```json
{
  "memories": [
    {
      "id": "mem_abc123",
      "data": "User's name is Alice.",
      "score": 0.95,
      "categories": ["identity"]
    }
  ],
  "raw": []
}
```

**curl**
```bash
curl -X POST https://your-persistio-instance/v1/recall \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"query": "what is the user'\''s name?", "top_k": 5}'
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/recall', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer pt_your_api_key_here',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ query: "what is the user's name?", top_k: 5 }),
});
const { memories } = await res.json();
```

**Python**
```python
import httpx

res = httpx.post(
    'https://your-persistio-instance/v1/recall',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
    json={'query': "what is the user's name?", 'top_k': 5},
)
memories = res.json()['memories']
```

---

## Memories

### `GET /v1/memories`

List memories for the current vault with optional filters.

**Auth:** Bearer token

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `archived` | bool | Include archived memories (default: false) |
| `category` | string | Filter by category |
| `limit` | int | Page size (default: 50) |
| `offset` | int | Pagination offset (default: 0) |

**Response:** `200 OK` — paginated list of memories

**curl**
```bash
curl "https://your-persistio-instance/v1/memories?limit=20&category=preferences" \
  -H "Authorization: Bearer pt_your_api_key_here"
```

**Node.js**
```js
const res = await fetch(
  'https://your-persistio-instance/v1/memories?limit=20&category=preferences',
  { headers: { 'Authorization': 'Bearer pt_your_api_key_here' } }
);
const { items } = await res.json();
```

**Python**
```python
import httpx

res = httpx.get(
    'https://your-persistio-instance/v1/memories',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
    params={'limit': 20, 'category': 'preferences'},
)
data = res.json()
```

---

### `POST /v1/memories`

Create a memory directly, without going through the ingest/extract pipeline.

**Auth:** Bearer token

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `data` | string | ✓ | The memory content |
| `subject` | string | ✓ | Who or what the memory is about |
| `categories` | string[] | | Optional category tags |

**Response:** `201 Created`

**curl**
```bash
curl -X POST https://your-persistio-instance/v1/memories \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"data": "User prefers dark mode.", "subject": "alice", "categories": ["preferences"]}'
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/memories', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer pt_your_api_key_here',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    data: 'User prefers dark mode.',
    subject: 'alice',
    categories: ['preferences'],
  }),
});
// 201 Created
```

**Python**
```python
import httpx

res = httpx.post(
    'https://your-persistio-instance/v1/memories',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
    json={'data': 'User prefers dark mode.', 'subject': 'alice', 'categories': ['preferences']},
)
# res.status_code == 201
```

---

### `GET /v1/memories/:id`

Retrieve a single memory by ID.

**Auth:** Bearer token

**Response:** `200 OK` or `404 Not Found`

**curl**
```bash
curl https://your-persistio-instance/v1/memories/mem_abc123 \
  -H "Authorization: Bearer pt_your_api_key_here"
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/memories/mem_abc123', {
  headers: { 'Authorization': 'Bearer pt_your_api_key_here' },
});
const memory = await res.json();
```

**Python**
```python
import httpx

res = httpx.get(
    'https://your-persistio-instance/v1/memories/mem_abc123',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
)
memory = res.json()
```

---

### `PATCH /v1/memories/:id`

Update a memory's content or metadata.

**Auth:** Bearer token

**Response:** `200 OK` or `404 Not Found`

**curl**
```bash
curl -X PATCH https://your-persistio-instance/v1/memories/mem_abc123 \
  -H "Authorization: Bearer pt_your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"data": "User prefers dark mode and high contrast."}'
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/memories/mem_abc123', {
  method: 'PATCH',
  headers: {
    'Authorization': 'Bearer pt_your_api_key_here',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ data: 'User prefers dark mode and high contrast.' }),
});
```

**Python**
```python
import httpx

res = httpx.patch(
    'https://your-persistio-instance/v1/memories/mem_abc123',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
    json={'data': 'User prefers dark mode and high contrast.'},
)
```

---

### `DELETE /v1/memories/:id`

Archive a memory (soft delete). Archived memories are excluded from recall by default but can be retrieved with `?archived=true`.

**Auth:** Bearer token

**Response:** `200 OK` or `404 Not Found`

**curl**
```bash
curl -X DELETE https://your-persistio-instance/v1/memories/mem_abc123 \
  -H "Authorization: Bearer pt_your_api_key_here"
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/memories/mem_abc123', {
  method: 'DELETE',
  headers: { 'Authorization': 'Bearer pt_your_api_key_here' },
});
```

**Python**
```python
import httpx

res = httpx.delete(
    'https://your-persistio-instance/v1/memories/mem_abc123',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
)
```

---

## Extraction

### `POST /v1/extract`

Trigger an extraction job immediately. Normally extraction runs automatically via the daemon; call this when you need memories available without delay.

**Auth:** Bearer token

**Response:** `202 Accepted`

```json
{ "job_id": "job_xyz789" }
```

**curl**
```bash
curl -X POST https://your-persistio-instance/v1/extract \
  -H "Authorization: Bearer pt_your_api_key_here"
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/extract', {
  method: 'POST',
  headers: { 'Authorization': 'Bearer pt_your_api_key_here' },
});
const { job_id } = await res.json();
```

**Python**
```python
import httpx

res = httpx.post(
    'https://your-persistio-instance/v1/extract',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
)
job_id = res.json()['job_id']
```

---

### `GET /v1/jobs/:id`

Check the status of an extraction job.

**Auth:** Bearer token

**Response:** `200 OK` or `404 Not Found`

```json
{
  "id": "job_xyz789",
  "status": "completed",
  "memories_created": 3,
  "created_at": "2024-05-01T12:00:00Z",
  "completed_at": "2024-05-01T12:00:04Z"
}
```

Status values: `pending`, `running`, `completed`, `failed`

**curl**
```bash
curl https://your-persistio-instance/v1/jobs/job_xyz789 \
  -H "Authorization: Bearer pt_your_api_key_here"
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/v1/jobs/job_xyz789', {
  headers: { 'Authorization': 'Bearer pt_your_api_key_here' },
});
const job = await res.json();
```

**Python**
```python
import httpx

res = httpx.get(
    'https://your-persistio-instance/v1/jobs/job_xyz789',
    headers={'Authorization': 'Bearer pt_your_api_key_here'},
)
job = res.json()
```

---

## Admin

### `POST /admin/vaults`

Create a new vault and receive its API key.

**Auth:** X-Admin-Key header

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✓ | Human-readable vault name |

**Response:** `201 Created`

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "api_key": "pt_your_api_key_here"
}
```

**curl**
```bash
curl -X POST https://your-persistio-instance/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name": "my-agent"}'
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/admin/vaults', {
  method: 'POST',
  headers: {
    'X-Admin-Key': 'adm_your_admin_key_here',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ name: 'my-agent' }),
});
const { id, api_key } = await res.json();
```

**Python**
```python
import httpx

res = httpx.post(
    'https://your-persistio-instance/admin/vaults',
    headers={'X-Admin-Key': 'adm_your_admin_key_here'},
    json={'name': 'my-agent'},
)
vault = res.json()  # { 'id': '...', 'api_key': '...' }
```

---

### `GET /admin/vaults`

List all vaults.

**Auth:** X-Admin-Key header

**Response:** `200 OK` — array of vault objects

**curl**
```bash
curl https://your-persistio-instance/admin/vaults \
  -H "X-Admin-Key: adm_your_admin_key_here"
```

**Node.js**
```js
const res = await fetch('https://your-persistio-instance/admin/vaults', {
  headers: { 'X-Admin-Key': 'adm_your_admin_key_here' },
});
const { items } = await res.json();
```

**Python**
```python
import httpx

res = httpx.get(
    'https://your-persistio-instance/admin/vaults',
    headers={'X-Admin-Key': 'adm_your_admin_key_here'},
)
vaults = res.json()
```

---

### `POST /admin/vaults/:id/rotate-key`

Rotate the API key for a vault. The old key is immediately invalidated.

**Auth:** X-Admin-Key header

**Response:** `200 OK` or `404 Not Found`

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "api_key": "pt_new_api_key_here"
}
```

**curl**
```bash
curl -X POST \
  https://your-persistio-instance/admin/vaults/a1b2c3d4-e5f6-7890-abcd-ef1234567890/rotate-key \
  -H "X-Admin-Key: adm_your_admin_key_here"
```

**Node.js**
```js
const res = await fetch(
  'https://your-persistio-instance/admin/vaults/a1b2c3d4-e5f6-7890-abcd-ef1234567890/rotate-key',
  {
    method: 'POST',
    headers: { 'X-Admin-Key': 'adm_your_admin_key_here' },
  }
);
const { api_key } = await res.json();
```

**Python**
```python
import httpx

res = httpx.post(
    'https://your-persistio-instance/admin/vaults/a1b2c3d4-e5f6-7890-abcd-ef1234567890/rotate-key',
    headers={'X-Admin-Key': 'adm_your_admin_key_here'},
)
new_key = res.json()['api_key']
```

---

### `DELETE /admin/vaults/:id`

Delete a vault and all associated data.

**Auth:** X-Admin-Key header

**Response:** `200 OK` or `404 Not Found`

**curl**
```bash
curl -X DELETE \
  https://your-persistio-instance/admin/vaults/a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  -H "X-Admin-Key: adm_your_admin_key_here"
```

**Node.js**
```js
const res = await fetch(
  'https://your-persistio-instance/admin/vaults/a1b2c3d4-e5f6-7890-abcd-ef1234567890',
  {
    method: 'DELETE',
    headers: { 'X-Admin-Key': 'adm_your_admin_key_here' },
  }
);
```

**Python**
```python
import httpx

res = httpx.delete(
    'https://your-persistio-instance/admin/vaults/a1b2c3d4-e5f6-7890-abcd-ef1234567890',
    headers={'X-Admin-Key': 'adm_your_admin_key_here'},
)
```
