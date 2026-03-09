# Voice Service API Reference

**Base URL (Docker):** `http://voice:8000`  
**Base URL (local dev):** `http://localhost:8000`

---

## Transcribe (STT)

```
POST /transcribe
Content-Type: multipart/form-data

Body: audio=<file>   (webm or wav, max 10MB)

→ 200 { "text": "I have 5 years of experience...", "duration_ms": 1340 }
→ 422 { "detail": "Audio file required" }
→ 500 { "detail": "STT processing failed" }
```

**NestJS call:**
```ts
const form = new FormData()
form.append('audio', audioBuffer, { filename: 'audio.webm', contentType: 'audio/webm' })
const res = await axios.post(`${this.voiceUrl}/transcribe`, form, { headers: form.getHeaders() })
const { text, duration_ms } = res.data
```

---

## Synthesize (TTS)

```
POST /synthesize
Content-Type: application/json

{ "text": "Tell me about your experience.", "voice": "en_US-amy-medium" }

→ 200 {
    "audioUrl": "/audio/abc123.wav",
    "visemes": [
      { "time": 0.00, "value": "sil" },
      { "time": 0.08, "value": "t" },
      { "time": 0.15, "value": "E" }
    ]
  }
→ 500 { "detail": "TTS processing failed" }
```

**NestJS call:**
```ts
const res = await axios.post(`${this.voiceUrl}/synthesize`, { text, voice: 'en_US-amy-medium' })
const { audioUrl, visemes } = res.data
```

---

## Health

```
GET /health
→ 200 { "status": "ok", "stt": "ready", "tts": "ready" }
```

---

## Available Voices

| Voice ID | Gender | Style |
|---|---|---|
| `en_US-amy-medium` | Female | Professional (default) |
| `en_US-ryan-high` | Male | Professional |
| `en_GB-alba-medium` | Female | British |

---

## Viseme → Morph Target Map

Used in `visemeMap.ts` on the frontend to drive avatar mouth shapes.

| Viseme | Morph Target | Sound example |
|---|---|---|
| `sil` | (reset all to 0) | Silence |
| `aa` | `mouthOpen` | "father" |
| `E` | `mouthSmile` | "bed" |
| `I` | `mouthSmileLeft` | "bit" |
| `O` | `mouthRound` | "story" |
| `U` | `mouthPucker` | "food" |
| `PP` | `mouthClose` | "p", "b", "m" |
| `FF` | `mouthLowerDownLeft` | "f", "v" |
| `TH` | `mouthOpen` (small) | "th" |
| `DD` | `mouthClose` | "d", "t", "n" |
| `kk` | `mouthUpperUpLeft` | "k", "g" |
| `CH` | `mouthSmile` | "ch", "sh" |
| `SS` | `mouthSmileLeft` | "s", "z" |
| `nn` | `mouthClose` | "n", "ng" |
| `RR` | `mouthRound` | "r" |

> Timing: each viseme applies from its `time` until the next viseme's `time`. Reset all morph targets to 0 on `sil`.