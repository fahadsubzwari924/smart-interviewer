# NestJS (Backend) API Reference

**Base URL:** `http://localhost:3000`

All admin endpoints require the header: `x-api-key: <ADMIN_API_KEY>`

---

## Health

```
GET /health
→ 200 { "status": "ok", "timestamp": "2026-03-09T12:00:00Z" }
```

---

## WebSocket (Socket.io)

Connect to: `ws://localhost:3000` (namespace: `/`)

### Client → Server Events

| Event | Payload | Description |
|---|---|---|
| `session:start` | `{ candidateName: string, jobRole: string }` | Create a new session and begin the interview |
| `candidate:audio` | `ArrayBuffer` (raw audio blob, webm/wav) | Send recorded candidate audio for a turn |
| `session:end` | `{ sessionId: string }` | Explicitly end the session early |

### Server → Client Events

| Event | Payload | Description |
|---|---|---|
| `session:created` | `{ sessionId: string }` | Emitted after session is created; client stores sessionId |
| `agent:response` | `{ audioUrl: string, visemes: Viseme[], questionText: string, score: number \| null, feedback: string \| null, isLast: boolean }` | Agent's spoken response + lip-sync data for this turn |
| `session:complete` | `{ sessionId: string, overallScore: number }` | Interview finished; triggers results screen |
| `error` | `{ message: string, code: string }` | Any error during the session |

**Viseme type:**
```ts
{ time: number, value: string }
```

---

## REST — Admin Endpoints

All return `Content-Type: application/json`.

### List Sessions

```
GET /admin/sessions
Headers: x-api-key: <key>

Query params (all optional):
  jobRole   string   Filter by job role (e.g. "Software Engineer (Backend)")
  status    string   Filter by status: pending | in_progress | completed | failed
  from      string   ISO date start range (e.g. "2026-03-01")
  to        string   ISO date end range
  limit     number   Default 50, max 200
  offset    number   Pagination offset, default 0

→ 200 {
    "total": 23,
    "data": [
      {
        "id": "uuid",
        "candidateName": "John Doe",
        "jobRole": "Software Engineer (Backend)",
        "status": "completed",
        "startedAt": "2026-03-09T10:00:00Z",
        "endedAt": "2026-03-09T10:18:00Z",
        "totalDurationSec": 1080,
        "overallScore": 7.4
      }
    ]
  }
```

### Get Session Detail

```
GET /admin/sessions/:id
Headers: x-api-key: <key>

→ 200 {
    "id": "uuid",
    "candidateName": "John Doe",
    "jobRole": "Software Engineer (Backend)",
    "status": "completed",
    "startedAt": "2026-03-09T10:00:00Z",
    "endedAt": "2026-03-09T10:18:00Z",
    "totalDurationSec": 1080,
    "overallScore": 7.4,
    "turns": [
      {
        "id": "uuid",
        "turnIndex": 0,
        "questionText": "Tell me about your background.",
        "candidateAnswer": "I have 5 years of experience...",
        "score": 7,
        "feedback": "Good overview, could be more concise.",
        "sttLatencyMs": 1200,
        "llmLatencyMs": 2800,
        "ttsLatencyMs": 1500,
        "totalLatencyMs": 5500,
        "createdAt": "2026-03-09T10:00:12Z"
      }
    ]
  }
→ 404 { "message": "Session not found" }
```

### Export Session (JSON)

```
GET /admin/sessions/:id/export
Headers: x-api-key: <key>

→ 200  Content-Disposition: attachment; filename="session-<id>.json"
       (Full session JSON, same as detail endpoint — for fine-tuning dataset export)
```

### Analytics

```
GET /admin/analytics
Headers: x-api-key: <key>

Query params (all optional):
  from    string   ISO date start
  to      string   ISO date end

→ 200 {
    "totalSessions": 120,
    "completedSessions": 98,
    "dropOffRate": 0.18,
    "avgOverallScore": 6.9,
    "avgSttLatencyMs": 1150,
    "avgLlmLatencyMs": 2600,
    "avgTtsLatencyMs": 1400,
    "avgTotalLatencyMs": 5150,
    "scoresByRole": [
      { "jobRole": "Software Engineer (Backend)", "avgScore": 7.1, "count": 42 },
      { "jobRole": "Product Manager", "avgScore": 6.4, "count": 28 }
    ]
  }
```

---

## Error Codes

| Code | Meaning |
|---|---|
| `SESSION_NOT_FOUND` | No session with given ID |
| `SESSION_ALREADY_ACTIVE` | Session is already in_progress |
| `STT_FAILED` | Voice service transcription error |
| `LLM_FAILED` | Ollama LLM error |
| `TTS_FAILED` | Voice service synthesis error |
| `INVALID_JOB_ROLE` | Unknown job role |
| `UNAUTHORIZED` | Missing or invalid x-api-key |