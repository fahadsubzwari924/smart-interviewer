# Smart Interviewer — Architecture

**Stack:** NestJS (backend) · Next.js 15 (frontend) · Python FastAPI (voice) · Ollama (LLM) · SQLite/Prisma · Socket.io · Docker

---

## 1. Service Map

```
[Browser: Next.js]
  │  Socket.io (WS)          HTTP (REST)
  ▼                               ▼
[NestJS API :3000]────────────────────
  │  HTTP POST /transcribe    │  HTTP POST /synthesize
  ▼                           ▼
[Voice Service :8000]    [Voice Service :8000]
  (Faster-Whisper)          (Piper TTS)
  │
  ▼
[Ollama :11434]  ← NestJS calls via HTTP POST /api/chat
  (LLaMA 3 8B)

[SQLite] ← accessed only by NestJS via Prisma
```

All services run in the same Docker network. Frontend is deployed separately (Vercel).

---

## 2. Per-Turn Data Flow

One interview turn = candidate speaks → agent responds:

```
1. Candidate clicks mic  → browser records audio (Web Audio API)
2. Audio blob            → WS emit: candidate:audio  → NestJS Gateway
3. NestJS AudioService   → POST /transcribe           → Voice Service
4. Transcribed text      → NestJS LLMService          → POST /api/chat (Ollama)
5. LLM JSON response     → NestJS AudioService        → POST /synthesize (Voice Service)
6. Audio (WAV) + visemes → NestJS                     → WS emit: agent:response → Browser
7. Browser plays audio, drives avatar lip-sync via viseme timestamps
8. NestJS SessionService → persist turn to SQLite
```

**Target latency:** STT ≤2s · LLM ≤3s · TTS ≤2s · **Total ≤5s**

---

## 3. WebSocket Events

| Direction | Event | Payload |
|---|---|---|
| Client → Server | `session:start` | `{ candidateName, jobRole }` |
| Client → Server | `candidate:audio` | `ArrayBuffer` (raw audio) |
| Client → Server | `session:end` | `{ sessionId }` |
| Server → Client | `agent:response` | `{ audioUrl, visemes, questionText, score, feedback, isLast }` |
| Server → Client | `session:created` | `{ sessionId }` |
| Server → Client | `session:complete` | `{ sessionId, overallScore }` |
| Server → Client | `error` | `{ message, code }` |

---

## 4. REST Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Service health check |
| GET | `/admin/sessions` | List all sessions (API key required) |
| GET | `/admin/sessions/:id` | Session detail + full transcript |
| GET | `/admin/sessions/:id/export` | Download transcript as JSON |

Admin endpoints require `x-api-key` header.

---

## 5. Voice Service API (Python FastAPI :8000)

| Method | Path | Request | Response |
|---|---|---|---|
| POST | `/transcribe` | `multipart/form-data: audio (file)` | `{ text: string, duration_ms: number }` |
| POST | `/synthesize` | `{ text: string, voice?: string }` | `{ audioUrl: string, visemes: Viseme[] }` |
| GET | `/health` | — | `{ status: "ok" }` |

**Viseme object:**
```ts
{ time: number, value: string }  // e.g. { time: 0.12, value: "aa" }
```

---

## 6. Database Schema (Prisma / SQLite)

```prisma
model Session {
  id              String    @id @default(uuid())
  candidateName   String
  jobRole         String
  status          String    // pending | in_progress | completed | failed
  startedAt       DateTime  @default(now())
  endedAt         DateTime?
  totalDurationSec Int?
  overallScore    Float?
  turns           Turn[]
}

model Turn {
  id             String   @id @default(uuid())
  sessionId      String
  session        Session  @relation(fields: [sessionId], references: [id])
  turnIndex      Int
  questionText   String
  candidateAnswer String
  score          Float?
  feedback       String?
  sttLatencyMs   Int?
  llmLatencyMs   Int?
  ttsLatencyMs   Int?
  createdAt      DateTime @default(now())
}
```

---

## 7. NestJS Module Structure

```
AppModule
├── InterviewModule
│   ├── InterviewGateway   (WebSocket: handles all WS events)
│   └── InterviewService   (orchestrates STT → LLM → TTS → persist)
├── LLMModule
│   └── LLMService         (calls Ollama, builds/manages chat history)
├── AudioModule
│   └── AudioService       (calls Voice Service for STT + TTS)
├── SessionModule
│   ├── SessionService     (CRUD on Session + Turn via Prisma)
│   └── SessionController  (REST: admin endpoints)
└── PrismaModule
    └── PrismaService      (singleton DB client)
```

**Key pattern:** `InterviewGateway` receives WS events → delegates to `InterviewService` → calls `LLMService` + `AudioService` → calls `SessionService` to persist → emits result back via Gateway.

---

## 8. Frontend Structure (Next.js 15)

```
/app
  page.tsx              → Setup screen (name, role, mic test)
  /interview/page.tsx   → Main interview screen
  /results/page.tsx     → Transcript + scores

/components
  /avatar
    AvatarCanvas.tsx    → @react-three/fiber canvas
    LipSyncController.tsx → Drives morph targets from viseme data
  /interview
    MicButton.tsx       → Record / stop / visualize audio
    TranscriptPanel.tsx → Scrollable Q&A log
    StatusBar.tsx       → State indicator + turn counter

/hooks
  useSocket.ts          → Socket.io client (connect, emit, listen)
  useAudioRecorder.ts   → MediaRecorder API wrapper
  useInterview.ts       → Session state machine

/lib
  socket.ts             → Singleton socket.io-client instance
  visemeMap.ts          → Maps phoneme strings to avatar morph targets
```

**Session state machine (frontend):**
```
idle → setup → connecting → agent_speaking → candidate_turn → processing → agent_speaking → ... → complete
```

---

## 9. Avatar Lip-Sync Approach

1. **TTS returns** audio (WAV) + viseme array `[{ time, value }]`
2. **Frontend** loads WAV into `AudioContext`, starts playback
3. `LipSyncController` uses `requestAnimationFrame` loop:
   - Reads `audioContext.currentTime`
   - Finds current viseme from array
   - Sets corresponding morph target weight on avatar mesh (0→1)
4. Avatar mouth shape updates at 60fps

**Viseme → morph target map** (stored in `visemeMap.ts`):
```
aa → mouthOpen, E → mouthSmile, I → mouthSmileLeft,
O → mouthRound, U → mouthPucker, PP/FF/TH/... → mouthClose etc.
```

---

## 10. Docker Compose Services

```yaml
services:
  backend:    # NestJS :3000  — depends_on: voice-service
  voice:      # Python :8000  — GPU-enabled (Whisper + Piper)
  ollama:     # Ollama  :11434 — GPU-enabled, volume: ollama_models
```

Frontend deployed to Vercel (points to backend URL via env var `NEXT_PUBLIC_API_URL`).

**Env vars (backend):**
```
OLLAMA_URL=http://ollama:11434
VOICE_SERVICE_URL=http://voice:8000
DATABASE_URL=file:./dev.db
ADMIN_API_KEY=your_secret_key
LLM_MODEL=llama3:8b
MAX_QUESTIONS=8
```

---

## 11. Latency Optimization Notes

- **LLM streaming:** Use Ollama streaming API; emit partial text to frontend while TTS waits for full sentence
- **TTS chunking:** Send first sentence to TTS immediately, stream rest — reduces perceived latency
- **Audio pre-buffering:** Frontend buffers first 0.5s before playback to avoid stuttering
- **Same Docker network:** All inter-service calls are LAN speed (~1ms), not internet
