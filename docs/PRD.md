# Smart Interviewer — Product Requirements Document (PRD)

**Version:** 1.0.0
**Date:** 2026-03-09
**Author:** fahadsubzwari924
**Status:** Draft

---

## 1. Product Overview

### 1.1 Product Name
**Smart Interviewer** — AI-Powered Voice & Avatar Interview Agent

### 1.2 Product Vision
Build an automated, voice-enabled interviewer agent that conducts structured job interviews using real-time speech-to-text, AI-driven conversational intelligence, text-to-speech responses, and an animated avatar with lip-sync — all self-hosted with minimal reliance on expensive third-party AI APIs.

### 1.3 Target Users
| User Type | Description |
|---|---|
| **Companies / Recruiters** | Use the platform to screen candidates automatically before human review |
| **Candidates** | Go through AI-driven interview sessions for real job roles |
| **Platform Admins** | Configure job roles, questions, scoring rubrics, and avatars |

### 1.4 Problem Statement
Traditional hiring screening is expensive, time-consuming, and inconsistent. Human screeners cost $50–$500+ per interview. Existing AI interview solutions rely heavily on expensive third-party APIs (OpenAI, ElevenLabs, HeyGen), making them cost-prohibitive for small businesses. Smart Interviewer solves this by delivering a premium interview experience with self-hosted AI, keeping costs under $200/month regardless of volume.

### 1.5 Business Model (Planned)
- SaaS subscription: $29–$99/month per company
- Pay-per-interview for smaller clients
- Premium tier with custom avatars, voice cloning, and advanced analytics

---

## 2. Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| **Backend** | NestJS (Node.js) | Modular, TypeScript-first, built-in WebSocket support |
| **Frontend** | Next.js 15 (React) | SSR/SSG, fast, great ecosystem for real-time + 3D |
| **Real-Time** | WebSocket (Socket.io) | Low-latency bidirectional communication |
| **LLM (AI Brain)** | Ollama + LLaMA 3 8B / Mistral 7B | Free, self-hosted, no API costs |
| **Speech-to-Text** | Faster-Whisper (Python microservice) | Open-source, fast, accurate |
| **Text-to-Speech** | Piper TTS / Coqui XTTS (Python microservice) | Open-source, natural-sounding voices |
| **Avatar / Lip-Sync** | Three.js + Viseme-based animation | Real-time, no GPU rendering needed |
| **Database** | SQLite (via Prisma ORM) | Simple, no-setup, easy to migrate later |
| **Containerization** | Docker + Docker Compose | Easy local and production deployment |
| **Hosting** | RunPod / Vast.ai (GPU) + Vercel (frontend) | Cost-effective GPU access |

---

## 3. System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Frontend (Next.js)                     │
│                                                              │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────┐  │
│  │  Interview View │  │  Avatar (Three.js│  │  Transcript│  │
│  │  (Main Screen)  │  │  + Lip Sync)     │  │  Panel     │  │
│  └────────┬────────┘  └──────────────────┘  └────────────┘  │
└───────────┼──────────────────────────────────────────────────┘
            │ WebSocket (Socket.io)
┌───────────▼──────────────────────────────────────────────────┐
│                     Backend (NestJS)                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │  Interview   │  │  Session     │  │  WebSocket         │  │
│  │  Module      │  │  Module      │  │  Gateway           │  │
│  └──────┬───────┘  └──────┬───────┘  └─────────┬──────────┘  │
│         │                 │                    │              │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌─────────▼──────────┐  │
│  │  LLM Service │  │  DB Service  │  │  Audio Service     │  │
│  │  (Ollama)    │  │  (Prisma +   │  │  (STT + TTS calls) │  │
│  │              │  │   SQLite)    │  │                    │  │
│  └──────────────┘  └──────────────┘  └────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
            │                              │
┌───────────▼──────────────┐  ┌────────────▼───────────────────┐
│   LLM Microservice       │  │   Voice Microservice           │
│   (Ollama HTTP API)      │  │   (Python FastAPI)             │
│   LLaMA 3 / Mistral      │  │   Faster-Whisper + Piper TTS  │
└──────────────────────────┘  └────────────────────────────────┘
```

---

## 4. Features & Requirements

### 4.1 Feature List Overview

| ID | Feature | Priority | Phase |
|---|---|---|---|
| F-01 | Interview Session Management | 🔴 Must Have | MVP |
| F-02 | Speech-to-Text (Candidate Voice) | 🔴 Must Have | MVP |
| F-03 | Text-to-Speech (Agent Voice) | 🔴 Must Have | MVP |
| F-04 | AI Interview Logic (LLM) | 🔴 Must Have | MVP |
| F-05 | Avatar with Lip-Sync | 🔴 Must Have | MVP |
| F-06 | Interview UI (Main Screen) | 🔴 Must Have | MVP |
| F-07 | Setup / Config Screen | 🔴 Must Have | MVP |
| F-08 | Results / Transcript Screen | 🔴 Must Have | MVP |
| F-09 | Session Logging & Monitoring | 🟡 Should Have | MVP |
| F-10 | Admin Dashboard (Job Roles Config) | 🟡 Should Have | Phase 2 |
| F-11 | Custom Avatar Upload | 🟢 Nice to Have | Phase 2 |
| F-12 | Voice Cloning (Custom Voice) | 🟢 Nice to Have | Phase 2 |
| F-13 | Multi-Role Interview Templates | 🟡 Should Have | Phase 2 |
| F-14 | Analytics Dashboard | 🟢 Nice to Have | Phase 3 |
| F-15 | Payment / Subscription System | 🟢 Nice to Have | Phase 3 |

---

### 4.2 Detailed Feature Requirements

---

#### F-01: Interview Session Management

**Description:** Manage the full lifecycle of an interview session from start to finish.

**Functional Requirements:**
- [ ] Create a new interview session (linked to a job role)
- [ ] Track session state: `pending` → `in_progress` → `completed` → `failed`
- [ ] Conduct multi-turn interview (configurable 5–10 questions per role)
- [ ] Gracefully handle candidate drop-off (auto-save progress)
- [ ] End session and trigger report generation
- [ ] Store all session data persistently (SQLite)

**Non-Functional Requirements:**
- Session must handle up to 10 question turns without performance degradation
- Session data must be persisted within 1 second of each turn

**Data Model:**
```
Session {
  id: UUID
  candidateName: string
  jobRole: string
  status: enum (pending | in_progress | completed | failed)
  startedAt: DateTime
  endedAt: DateTime
  totalDurationSeconds: number
  overallScore: number (nullable)
  questions: Question[]
}

Question {
  id: UUID
  sessionId: UUID
  questionText: string
  candidateAnswer: string (transcribed)
  score: number (nullable)
  feedback: string (nullable)
  durationSeconds: number
  turnIndex: number
}
```

---

#### F-02: Speech-to-Text (STT)

**Description:** Convert candidate's spoken audio to text in real-time.

**Functional Requirements:**
- [ ] Capture audio from candidate's microphone (browser Web Audio API)
- [ ] Stream audio to backend WebSocket
- [ ] Process audio through Faster-Whisper Python microservice
- [ ] Return transcribed text to frontend within 2 seconds
- [ ] Display transcription live as candidate speaks (streaming if possible)
- [ ] Handle silence/timeout (stop recording after 3s of silence)
- [ ] Show clear recording state indicator (recording / processing / done)

**Non-Functional Requirements:**
- Transcription accuracy target: > 90% for clear audio
- Maximum latency from speech end to transcription: < 2 seconds
- Support English language (additional languages in Phase 2)

**Integration:**
- Python microservice endpoint: `POST /transcribe` (accepts audio blob, returns text)
- Called from NestJS AudioService via HTTP

---

#### F-03: Text-to-Speech (TTS)

**Description:** Convert interviewer agent's text responses to natural-sounding audio.

**Functional Requirements:**
- [ ] Accept interviewer question text from LLM
- [ ] Generate audio using Piper TTS or Coqui XTTS
- [ ] Stream or return audio file to frontend
- [ ] Frontend plays audio automatically before awaiting candidate response
- [ ] Return viseme/phoneme timing data alongside audio (for lip-sync)
- [ ] Support at least one high-quality default voice

**Non-Functional Requirements:**
- TTS generation latency: < 2 seconds for a typical question (< 30 words)
- Audio quality: Natural, clear, professional-sounding
- Audio format: WAV or MP3

**Integration:**
- Python microservice endpoint: `POST /synthesize` (accepts text, returns audio + viseme data)
- Called from NestJS AudioService via HTTP

---

#### F-04: AI Interview Logic (LLM)

**Description:** Drive intelligent, contextual interview conversations using a self-hosted LLM.

**Functional Requirements:**
- [ ] Initialize interview with role-specific system prompt
- [ ] Ask opening question based on job role
- [ ] Generate contextual follow-up questions based on candidate's previous answers
- [ ] Avoid repeating already-asked questions
- [ ] Evaluate candidate answer quality (score 1–10 with brief feedback)
- [ ] Detect when interview should wrap up (after N questions)
- [ ] Generate closing statement and thank-you message
- [ ] Return structured JSON responses (question, score, feedback, isLastQuestion)

**Non-Functional Requirements:**
- LLM response time: < 3 seconds for typical responses
- Must stay in character as professional interviewer throughout session
- Must not hallucinate or go off-topic

**LLM Integration:**
- Ollama HTTP API: `POST http://localhost:11434/api/chat`
- Model: `llama3:8b` (default) or `mistral:7b` (fallback)
- Called from NestJS LLMService

**System Prompt Structure:**
```
You are Alex, a professional senior interviewer at a top tech company.
You are conducting a [JOB_ROLE] interview with [CANDIDATE_NAME].
Your behavior:
- Ask one focused question at a time
- Listen to answers and ask relevant follow-up questions
- Be professional, encouraging, but thorough
- Score each answer on a scale of 1-10
- After [MAX_QUESTIONS] questions, wrap up the interview
Always respond in valid JSON format:
{
  "question": "string",
  "score": number | null,
  "feedback": "string | null",
  "isLastQuestion": boolean
}
```

**Supported Job Roles (MVP):**
1. Software Engineer (Frontend)
2. Software Engineer (Backend)
3. Product Manager
4. Sales Representative
5. Data Analyst

---

#### F-05: Avatar with Lip-Sync

**Description:** Display an animated interviewer avatar whose lips sync to the TTS audio output.

**Functional Requirements:**
- [ ] Display 3D avatar of interviewer on screen
- [ ] Animate avatar lip-sync based on viseme data from TTS
- [ ] Play idle animation when agent is not speaking
- [ ] Play "thinking" animation while waiting for LLM response
- [ ] Play "listening" animation while candidate is speaking
- [ ] Support a default avatar (professional-looking character)
- [ ] Avatar must be responsive (scales to screen size)

**Non-Functional Requirements:**
- Lip-sync timing accuracy: < 100ms offset from audio
- Avatar rendering: 60fps minimum (Three.js GPU-accelerated)
- Avatar file size: < 5MB (GLTF/GLB format)

**Technical Approach:**
- 3D avatar model: Ready Player Me (free tier) or Mixamo rigged character
- Renderer: Three.js with @react-three/fiber in Next.js
- Lip-sync: Map TTS viseme data to avatar morph targets
- Animations: Mixamo idle/listen/think animations

---

#### F-06: Interview UI (Main Screen)

**Description:** The primary screen where the interview takes place.

**Functional Requirements:**
- [ ] Display avatar (left/center panel, ~60% width)
- [ ] Display current question text (below avatar)
- [ ] Display transcript panel (right panel, ~40% width, scrollable)
- [ ] Show microphone button (large, prominent — click to answer)
- [ ] Show visual audio waveform while recording
- [ ] Show processing indicator (LLM thinking, TTS generating)
- [ ] Show turn counter (e.g., "Question 3 of 8")
- [ ] Show elapsed time for session
- [ ] Disable microphone button while agent is speaking

**UI States:**
```
1. agent_speaking   → Avatar animating, audio playing, mic disabled
2. candidate_turn   → Avatar idle/listening, mic enabled, waiting for input
3. processing       → Avatar thinking animation, spinner, mic disabled
4. session_complete → Show completion message, trigger results screen
```

---

#### F-07: Setup / Config Screen

**Description:** Pre-interview configuration screen for the candidate.

**Functional Requirements:**
- [ ] Input field: Candidate name
- [ ] Dropdown: Select job role
- [ ] Microphone test button (record 3s, play back)
- [ ] Speaker test button (play sample audio)
- [ ] Microphone permission prompt/guide
- [ ] "Start Interview" button (disabled until mic permission granted)
- [ ] Display estimated interview duration

---

#### F-08: Results / Transcript Screen

**Description:** Post-interview summary shown after session completion.

**Functional Requirements:**
- [ ] Display full interview transcript (Q&A pairs)
- [ ] Display per-question score (1–10) and AI feedback
- [ ] Display overall session score (average)
- [ ] Display total interview duration
- [ ] Download transcript as PDF or TXT
- [ ] Option to start a new interview

---

#### F-09: Session Logging & Monitoring

**Description:** Log all session data for quality monitoring and future fine-tuning.

**Functional Requirements:**
- [ ] Log every question asked (text + timestamp)
- [ ] Log every candidate answer (transcribed text + timestamp)
- [ ] Log LLM response (question, score, feedback, raw response)
- [ ] Log latency per turn (STT time, LLM time, TTS time)
- [ ] Log session outcome (completed, dropped, error)
- [ ] Expose basic admin endpoint to list sessions: `GET /admin/sessions`
- [ ] Expose session detail endpoint: `GET /admin/sessions/:id`

**Purpose:** Used to evaluate LLM quality, STT accuracy, and TTS performance after MVP launch to decide on fine-tuning.

---

## 5. Non-Functional Requirements

| Category | Requirement |
|---|---|
| **Performance** | End-to-end turn latency < 5 seconds (STT + LLM + TTS combined) |
| **Scalability (MVP)** | Support 1 concurrent session (single GPU server) |
| **Availability** | Best-effort uptime for MVP (no SLA required) |
| **Security** | Basic API key auth for admin endpoints (MVP) |
| **Browser Support** | Chrome and Edge (latest 2 versions) |
| **Responsive Design** | Desktop-first (1280px+), basic tablet support |
| **Accessibility** | Basic ARIA labels on interactive elements |
| **Cost** | Total infrastructure cost < $200/month |

---

## 6. Out of Scope (MVP)

The following features are explicitly **NOT** included in the MVP:

- ❌ Multi-candidate concurrent sessions
- ❌ User authentication / accounts (login, register)
- ❌ Payment / subscription system
- ❌ Mobile app or mobile-optimized web
- ❌ Video recording of interview sessions
- ❌ Human interviewer review workflow
- ❌ Advanced analytics dashboard
- ❌ Multi-language support
- ❌ Custom avatar upload by end-user
- ❌ Voice cloning / custom interviewer voice
- ❌ Email notifications
- ❌ ATS (Applicant Tracking System) integrations

---

## 7. Project Folder Structure

```
xmart-interviewer/
├── backend/                        # NestJS Backend
│   ├── src/
│   │   ├── modules/
│   │   │   ├── interview/          # Interview session logic
│   │   │   │   ├── interview.module.ts
│   │   │   │   ├── interview.service.ts
│   │   │   │   ├── interview.controller.ts
│   │   │   │   └── interview.gateway.ts   # WebSocket
│   │   │   ├── llm/                # LLM (Ollama) integration
│   │   │   │   ├── llm.module.ts
│   │   │   │   └── llm.service.ts
│   │   │   ├── audio/              # STT + TTS integration
│   │   │   │   ├── audio.module.ts
│   │   │   │   └── audio.service.ts
│   │   │   └── session/            # Session management + logging
│   │   │       ├── session.module.ts
│   │   │       ├── session.service.ts
│   │   │       └── session.controller.ts
│   │   ├── prisma/                 # Prisma ORM
│   │   │   └── schema.prisma
│   │   ├── config/                 # App configuration
│   │   └── main.ts
│   ├── Dockerfile
│   └── package.json
│
├── frontend/                       # Next.js 15 Frontend
│   ├── src/
│   │   ├── app/
│   │   │   ├── page.tsx            # Setup / Config Screen
│   │   │   ├── interview/
│   │   │   │   └── page.tsx        # Interview Main Screen
│   │   │   └── results/
│   │   │       └── page.tsx        # Results / Transcript Screen
│   │   ├── components/
│   │   │   ├── avatar/             # Three.js Avatar + Lip-Sync
│   │   │   │   ├── AvatarCanvas.tsx
│   │   │   │   └── LipSyncController.tsx
│   │   │   ├── interview/
│   │   │   │   ├── MicrophoneButton.tsx
│   │   │   │   ├── TranscriptPanel.tsx
│   │   │   │   ├── QuestionDisplay.tsx
│   │   │   │   └── ProgressBar.tsx
│   │   │   └── ui/                 # Shared UI components
│   │   ├── hooks/
│   │   │   ├── useSocket.ts        # WebSocket connection
│   │   │   ├── useAudioRecorder.ts # Mic capture
│   │   │   └── useInterviewSession.ts
│   │   ├── lib/
│   │   │   └── socket.ts
│   │   └── types/
│   │       └── interview.types.ts
│   ├── public/
│   │   └── avatars/                # Default avatar GLB files
│   ├── Dockerfile
│   └── package.json
│
├── voice-service/                  # Python Microservice (STT + TTS)
│   ├── main.py                     # FastAPI app
│   ├── stt/
│   │   └── whisper_service.py      # Faster-Whisper STT
│   ├── tts/
│   │   └── piper_service.py        # Piper TTS
│   ├── requirements.txt
│   └── Dockerfile
│
├── docs/
│   ├── PRD.md                      # This document
│   ├── ARCHITECTURE.md             # Detailed architecture (coming next)
│   ├── API.md                      # API reference (coming next)
│   └── SETUP.md                    # Local dev setup guide (coming next)
│
├── docker-compose.yml              # Full stack orchestration
└── README.md
```

---

## 8. Development Phases & Timeline

| Phase | Scope | Estimated Duration |
|---|---|---|
| **Phase 1** | Backend Foundation (NestJS setup, LLM integration, basic API) | Week 1–2 |
| **Phase 2** | Voice Integration (STT + TTS Python microservice + NestJS integration) | Week 2–3 |
| **Phase 3** | Frontend UI (Next.js setup, Interview UI, Setup & Results screens) | Week 3–4 |
| **Phase 4** | Avatar & Lip-Sync (Three.js avatar + viseme-based animation) | Week 4–5 |
| **Phase 5** | Session Logging & Monitoring (DB logging, admin endpoints) | Week 5 |
| **Phase 6** | Integration Testing & Deployment (Docker, GPU server, end-to-end QA) | Week 6 |

---

## 9. Success Metrics (Post-MVP)

These metrics will be tracked after launch to evaluate quality and guide fine-tuning decisions:

| Metric | Target | Measurement Method |
|---|---|---|
| End-to-end turn latency | < 5 seconds | Automatic logging per session |
| STT accuracy | > 90% | Manual review of 20 sample sessions |
| LLM question relevance | > 80% rated "relevant" | Manual review + user feedback |
| Session completion rate | > 85% | Session status tracking |
| TTS naturalness score | > 4/5 | Post-interview user survey |
| Avatar lip-sync accuracy | < 100ms offset | Manual observation |

---

## 10. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LLM response latency too high | Medium | High | Use streaming responses; fallback to smaller model (Phi-3) |
| Real-time lip-sync complexity | High | Medium | Start with static avatar; add lip-sync in Phase 4 |
| GPU server cost overrun | Low | Medium | Set billing alerts; use serverless GPU for MVP |
| Faster-Whisper STT inaccuracy | Low | High | Test with multiple accents; prompt user to speak clearly |
| Browser mic permission issues | Medium | Medium | Provide clear permission guide on setup screen |
| Python/Node service communication latency | Low | Medium | Use local network (same Docker network) |

---

## 11. Glossary

| Term | Definition |
|---|---|
| **STT** | Speech-to-Text: Converting spoken audio to written text |
| **TTS** | Text-to-Speech: Converting written text to spoken audio |
| **LLM** | Large Language Model: AI model that generates conversational responses |
| **Viseme** | Visual phoneme: A mouth shape corresponding to a sound, used for lip-sync |
| **Ollama** | A tool for running LLMs locally on your machine |
| **Faster-Whisper** | An optimized version of OpenAI's Whisper STT model |
| **Piper TTS** | A fast, open-source text-to-speech engine |
| **Three.js** | A JavaScript 3D library used for rendering the avatar |
| **GLB/GLTF** | 3D model file formats used for the avatar |
| **Morph Target** | A 3D animation technique used to animate mouth shapes for lip-sync |
| **MVP** | Minimum Viable Product: The first working version of the product |

---

*This document will be updated as the project evolves. Next document: `ARCHITECTURE.md`*