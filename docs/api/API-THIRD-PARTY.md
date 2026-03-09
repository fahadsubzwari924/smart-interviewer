# Third-Party API Method Reference

Self-contained reference for every third-party library method used in Smart Interviewer.
No external links. Cursor reads this file instead of browsing live documentation.

---

## 1. Socket.io Server — `@nestjs/websockets` + `socket.io`

### Decorators (NestJS Gateway)

| Decorator | Purpose |
|---|---|
| `@WebSocketGateway(options)` | Marks a class as a WebSocket gateway. Options: `{ cors: { origin: '*' }, namespace?: string }` |
| `@SubscribeMessage(event: string)` | Registers a method as a handler for the named WebSocket event |
| `@MessageBody()` | Injects the deserialized payload from the incoming event |
| `@ConnectedSocket()` | Injects the raw `Socket` instance for the connected client |

### `Socket` instance methods (server-side)

| Method | Signature | Description |
|---|---|---|
| `emit` | `socket.emit(event: string, data: any): boolean` | Emit an event to this specific client |
| `broadcast.emit` | `socket.broadcast.emit(event: string, data: any)` | Emit to all clients except the sender |
| `join` | `socket.join(room: string): Promise<void>` | Join a named room |
| `to` | `socket.to(room: string).emit(event, data)` | Emit to all sockets in a room |
| `id` | `string` | Unique socket connection ID |
| `disconnect` | `socket.disconnect(close?: boolean): void` | Disconnect the client |

### `Server` instance methods (server-side)

| Method | Signature | Description |
|---|---|---|
| `emit` | `server.emit(event: string, data: any)` | Broadcast to all connected clients |
| `to` | `server.to(room: string).emit(event, data)` | Emit to all clients in a room |

---

## 2. Socket.io Client — `socket.io-client`

### `io(url, options)` — factory function

| Parameter | Type | Description |
|---|---|---|
| `url` | `string` | Server URL, e.g. `http://localhost:3000` |
| `options.transports` | `string[]` | Transport list, use `['websocket']` to skip polling |
| `options.autoConnect` | `boolean` | Default `true`. Set `false` to connect manually |
| `options.reconnection` | `boolean` | Default `true` |
| `options.reconnectionAttempts` | `number` | Max reconnect attempts before giving up |

Returns a `Socket` instance.

### `Socket` instance methods (client-side)

| Method | Signature | Description |
|---|---|---|
| `connect` | `socket.connect(): Socket` | Manually connect (when `autoConnect: false`) |
| `disconnect` | `socket.disconnect(): Socket` | Disconnect from server |
| `emit` | `socket.emit(event: string, ...args: any[]): Socket` | Send an event to the server |
| `on` | `socket.on(event: string, listener: (...args) => void): Socket` | Listen for an event from the server |
| `off` | `socket.off(event: string, listener?: Function): Socket` | Remove an event listener |
| `once` | `socket.once(event: string, listener: Function): Socket` | Listen for an event exactly once |

### Built-in client events

| Event | Fires when |
|---|---|
| `connect` | Connection established; `socket.id` is set |
| `disconnect` | Connection lost; receives `reason: string` |
| `connect_error` | Connection attempt failed; receives `error: Error` |

---

## 3. Prisma Client (v5) — PostgreSQL

### `PrismaClient` constructor

`new PrismaClient({ log?: Array<'query'|'info'|'warn'|'error'> })`

Connect is implicit on first query. Call `$connect()` for explicit connection and `$disconnect()` on shutdown.

### Session model — methods used

| Method | Signature | Description |
|---|---|---|
| `findMany` | `prisma.session.findMany({ where?, orderBy?, skip?, take?, include? })` | Query multiple sessions with optional filters |
| `findUnique` | `prisma.session.findUnique({ where: { id }, include? })` | Fetch one session by primary key |
| `create` | `prisma.session.create({ data, include? })` | Insert a new session row |
| `update` | `prisma.session.update({ where: { id }, data })` | Update fields on an existing session |
| `delete` | `prisma.session.delete({ where: { id } })` | Delete a session (cascades to turns) |

### Turn model — methods used

| Method | Signature | Description |
|---|---|---|
| `create` | `prisma.turn.create({ data, include? })` | Insert a new turn row |
| `findMany` | `prisma.turn.findMany({ where: { sessionId }, orderBy: { turnIndex: 'asc' } })` | Fetch all turns for a session |

### Aggregation — methods used

| Method | Signature | Description |
|---|---|---|
| `aggregate` | `prisma.session.aggregate({ _avg: { overallScore: true }, where? })` | Compute avg/min/max/count on numeric fields |
| `groupBy` | `prisma.session.groupBy({ by: ['jobRole'], _avg: { overallScore: true }, where? })` | Group and aggregate — used for per-role analytics |

### `where` filter operators

| Operator | Type | Description |
|---|---|---|
| `equals` | `any` | Exact match (default when passing plain value) |
| `in` | `any[]` | Match any value in array |
| `gte` / `lte` | `number \| Date` | Greater/less than or equal |
| `contains` | `string` | SQL LIKE %value% |
| `not` | `any` | Negation |

---

## 4. Ollama HTTP API

Ollama runs locally at `http://ollama:11434` (Docker) or `http://localhost:11434` (dev).

### `POST /api/chat` — Chat completion (used by LLMService)

**Request body:**

| Field | Type | Description |
|---|---|---|
| `model` | `string` | Model name, e.g. `llama3:8b` |
| `messages` | `Message[]` | Conversation history (see below) |
| `stream` | `boolean` | `false` = wait for full response; `true` = stream NDJSON chunks |
| `options.temperature` | `number` | 0–1. Higher = more creative. Use `0.4` for structured interview responses |
| `options.num_predict` | `number` | Max tokens to generate |

**Message object:**

| Field | Type | Values |
|---|---|---|
| `role` | `string` | `system` or `user` or `assistant` |
| `content` | `string` | Message text |

**Response (stream: false):**

| Field | Type | Description |
|---|---|---|
| `message.role` | `string` | Always `assistant` |
| `message.content` | `string` | The LLM's reply text |
| `done` | `boolean` | Always `true` when not streaming |
| `eval_duration` | `number` | Nanoseconds spent on generation |

**Response (stream: true)** — Each line is a JSON object:

| Field | Type | Description |
|---|---|---|
| `message.content` | `string` | Partial text chunk |
| `done` | `boolean` | `true` on the final chunk only |

### `GET /api/tags` — List available models

Returns `{ models: [{ name, size, modified_at }] }`. Used on startup to verify model is loaded.

### `POST /api/pull` — Pull a model

Request: `{ name: string, stream?: boolean }`. Pulls `llama3:8b` if not present. Streams progress lines.

---

## 5. Faster-Whisper (Python — via Voice Service)

Faster-Whisper is consumed through the internal Voice Service (`POST /transcribe`). The Python-side API used in `voice-service/main.py`:

### `WhisperModel` constructor

`WhisperModel(model_size, device, compute_type)`

| Parameter | Value used | Description |
|---|---|---|
| `model_size` | `"base"` | Model size: tiny / base / small / medium / large-v3 |
| `device` | `"cuda"` or `"cpu"` | Inference device |
| `compute_type` | `"float16"` (GPU) / `"int8"` (CPU) | Quantization type |

### `model.transcribe(audio, beam_size, language, vad_filter)`

| Parameter | Value used | Description |
|---|---|---|
| `audio` | `str` (file path) or `np.ndarray` | Input audio |
| `beam_size` | `5` | Beam search width; higher = more accurate, slower |
| `language` | `"en"` | Force language detection to English |
| `vad_filter` | `True` | Skip silent segments using Silero VAD |

Returns `(segments, info)`. Iterate `segments` to collect `.text` and `.start`/`.end` timestamps.

---

## 6. Piper TTS (Python — via Voice Service)

Piper is a binary called via subprocess in `voice-service/main.py`. It is not a Python SDK.

### CLI invocation

```
piper --model <model_path> --output_file <output.wav>
```

Input text is passed via **stdin**. The binary writes WAV audio to `--output_file`.

### Arguments used

| Argument | Description |
|---|---|
| `--model` | Path to `.onnx` model file, e.g. `models/en_US-amy-medium.onnx` |
| `--output_file` | Output WAV file path |
| `--sentence_silence` | Seconds of silence between sentences (default `0.2`) |
| `--length_scale` | Speech speed. `1.0` = normal, `0.9` = slightly faster |

### Model config sidecar (`.onnx.json`)

Every `.onnx` model ships with a `.onnx.json` file. The relevant field for viseme generation:

| Field | Description |
|---|---|
| `phoneme_map` | Maps IPA phonemes to integer IDs |
| `audio.sample_rate` | Output sample rate (typically `22050`) |

Visemes are derived by aligning Piper's phoneme sequence with audio duration using the segment timestamps from Faster-Whisper or a forced aligner.

---

## 7. Three.js / @react-three/fiber (Frontend Avatar)

### `@react-three/fiber` — used APIs

| API | Description |
|---|---|
| `<Canvas>` | Root component; creates WebGL renderer and scene |
| `useFrame(callback)` | Registers a per-frame callback (60fps). Receives `(state, delta)`. Used in `LipSyncController` to update morph targets |
| `useLoader(GLTFLoader, url)` | Load a `.glb` / `.gltf` avatar model. Returns `{ scene, nodes, animations }` |
| `useThree()` | Access renderer, camera, scene from any child component |

### Morph target (blend shape) manipulation

Avatar mouth shapes are driven by setting weights on `SkinnedMesh.morphTargetInfluences`:

| Property / Method | Description |
|---|---|
| `mesh.morphTargetDictionary` | `Record<string, number>` — maps morph target name to its index |
| `mesh.morphTargetInfluences` | `number[]` — weight array (0–1) indexed by morph target index |

To set a morph target weight:
```ts
mesh.morphTargetInfluences[mesh.morphTargetDictionary['mouthOpen']] = 0.8
``` 
### `AudioContext` (Web Audio API — browser built-in)

| Method / Property | Description |
|---|---|
| `new AudioContext()` | Create audio context |
| `ctx.decodeAudioData(arrayBuffer)` | Decode WAV/MP3 bytes into `AudioBuffer`. Returns `Promise<AudioBuffer>` |
| `ctx.createBufferSource()` | Create a playback node |
| `source.buffer = audioBuffer` | Assign decoded audio |
| `source.connect(ctx.destination)` | Route audio to speakers |
| `source.start(when?)` | Start playback. `when` defaults to `ctx.currentTime` |
| `ctx.currentTime` | Read-only. Current playback position in seconds. Used to sync viseme timing |

---

## 8. MediaRecorder API (Browser — Audio Capture)

| Method / Property | Description |
|---|---|
| `navigator.mediaDevices.getUserMedia({ audio: true })` | Request mic access. Returns `Promise<MediaStream>` |
| `new MediaRecorder(stream, { mimeType })` | Create recorder. Use `mimeType: 'audio/webm'` (widely supported) |
| `recorder.start(timeslice?)` | Begin recording. Optional `timeslice` triggers `ondataavailable` every N ms |
| `recorder.stop()` | Stop recording. Fires `onstop` after final `ondataavailable` |
| `recorder.ondataavailable` | Event handler: receives `BlobEvent` with `.data: Blob` chunk |
| `recorder.onstop` | Event handler: fires when recording fully stopped |
| `recorder.state` | `"inactive"` / `"recording"` / `"paused"` |

---

## 9. FastAPI (Python Voice Service)

### App setup

| API | Description |
|---|---|
| `FastAPI()` | Create app instance |
| `@app.post(path)` | Register a POST endpoint |
| `@app.get(path)` | Register a GET endpoint |
| `uvicorn.run(app, host, port)` | Start ASGI server |

### Request types used

| Type | Import | Usage |
|---|---|---|
| `UploadFile` | `fastapi` | Receive audio file upload at `/transcribe` |
| `File(...)` | `fastapi` | Mark parameter as file upload |
| `BaseModel` | `pydantic` | Define JSON request body (used for `/synthesize`) |

### Response helpers

| Helper | Description |
|---|---|
| `FileResponse(path, media_type)` | Stream a file back. Used to return WAV audio |
| `JSONResponse(content, status_code)` | Return JSON with custom status |
| `HTTPException(status_code, detail)` | Raise an HTTP error |
