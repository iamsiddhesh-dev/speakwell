# SpeakWell — AI English Speaking Coach

<div align="center">

**Record yourself speaking. Get instant AI feedback. Improve faster.**

🌐 [speakwell-live.vercel.app](https://speakwell-live.vercel.app) · [Report a Bug](https://github.com/iamsiddhesh-dev/speakwell/issues) · [Request a Feature](https://github.com/iamsiddhesh-dev/speakwell/issues)

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-000000?style=flat&logo=nextdotjs&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)

</div>

---

## What Is SpeakWell?

SpeakWell is a full-stack AI-powered English speaking coach. You record yourself speaking — even with mistakes — and within seconds you get:

- **Grammar corrections** with clear explanations for each mistake
- **Filler word detection** — catches overused words like "so", "like", "basically"
- **Fluency, clarity, and confidence scores** out of 100
- **A corrected version** of what you said, read back to you in audio
- **A natural native-speaker version** — how a fluent speaker would phrase it
- **Waveform audio players** for your original recording, corrected version, and natural version side by side
- **Session history** — every practice session is saved so you can track improvement over time

---

## Live Demo

> 🌐 [https://speakwell-live.vercel.app](https://speakwell-live.vercel.app)

Sign up with any email → click SPEAK → record yourself → get feedback in 10-15 seconds.

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        USER BROWSER                          │
│              https://speakwell-live.vercel.app               │
└───────────────────────────┬──────────────────────────────────┘
                            │ HTTPS
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     VERCEL (Frontend)                        │
│                        Next.js 16                            │
│    Voice Recorder │ Dashboard UI │ Auth UI │ History Page    │
└───────────────────────────┬──────────────────────────────────┘
                            │ REST API + polling
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                    RAILWAY (Backend)                         │
│                                                              │
│  ┌──────────────────┐         ┌───────────────────────────┐  │
│  │   FastAPI        │         │     Celery Worker         │  │
│  │   API Gateway    │──Redis─▶│   AI Processing Pipeline  │  │
│  │   Port 8000      │         │                           │  │
│  └──────────────────┘         │  1. Groq Whisper (STT)    │  │
│                               │  2. RMS energy check      │  │
│                               │  3. Groq Llama (analysis) │  │
│                               │  4. gTTS (text-to-speech) │  │
│                               │  5. Supabase session save │  │
│                               └───────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                            │
           ┌────────────────┼─────────────────┐
           ▼                ▼                  ▼
┌───────────────┐  ┌────────────────┐  ┌──────────────────────┐
│   Supabase    │  │   Supabase     │  │      Groq API        │
│  PostgreSQL   │  │   Storage      │  │                      │
│               │  │                │  │  whisper-large-v3    │
│  · Users      │  │  · Uploads     │  │  llama-3.3-70b       │
│  · Sessions   │  │  · Generated   │  │                      │
│  · Profiles   │  │    audio       │  │                      │
└───────────────┘  └────────────────┘  └──────────────────────┘
```

### Request Flow

1. User records audio in browser via `MediaRecorder` API
2. Frontend uploads audio blob to FastAPI → audio stored in Supabase Storage
3. FastAPI queues a Celery task and returns `task_id` immediately (< 200ms)
4. Celery worker picks up task from Redis queue
5. Worker downloads audio from Supabase Storage to `/tmp/`
6. RMS energy check rejects silence before calling any AI
7. Groq Whisper transcribes audio (~0.2s on Groq's GPU)
8. Groq Llama analyzes transcript — grammar, scores, corrections
9. Filler words counted deterministically from Whisper's word segments — no LLM
10. gTTS generates two audio files → uploaded to Supabase Storage
11. Session saved to Supabase PostgreSQL
12. Frontend polls `GET /api/audio/task/{id}` every 2 seconds until done
13. Results rendered — transcript, scores, corrections, three audio players

---

## Tech Stack

### Frontend
| Technology | Purpose |
|---|---|
| Next.js 16 + TypeScript | React framework with App Router |
| Tailwind CSS | Utility-first styling |
| WaveSurfer.js | Audio waveform visualization |
| Supabase JS | Auth client + database queries |
| Axios | HTTP client with JWT interceptor |

### Backend
| Technology | Purpose |
|---|---|
| FastAPI | Async REST API gateway |
| Celery | Distributed task queue |
| Redis | Message broker + result backend |
| Pydantic Settings | Centralized config management |
| gTTS | Text-to-speech generation |
| Librosa | RMS energy detection (silence guard) |

### AI Services
| Service | Model | Purpose |
|---|---|---|
| Groq | whisper-large-v3-turbo | Speech to text (< 1s) |
| Groq | llama-3.3-70b-versatile | Grammar analysis + corrections |

### Infrastructure
| Technology | Purpose |
|---|---|
| Vercel | Frontend hosting + CDN |
| Railway | Backend + Celery worker + Redis |
| Supabase | PostgreSQL + Storage + Auth |
| Docker + Docker Compose | Local development orchestration |

---

## Why It's Built This Way

Five decisions that shaped the architecture, and the reasoning behind each:

**The API returns in under 200ms and never waits for the AI.** Transcription,
analysis and speech synthesis together take 10–15 seconds — far past any sensible
HTTP timeout, and long enough that a dropped connection would lose the work
entirely. So the upload endpoint does exactly two things: store the audio and
queue a Celery task. The client gets a `task_id` immediately and polls. The cost
is a polling loop and a Redis dependency; the benefit is that a request can't time
out, and a worker crash loses one task rather than the user's recording.

**A silence check runs before any AI call.** An RMS energy threshold rejects empty
or near-silent recordings up front. Whisper will happily hallucinate confident
text out of near-silence, so this isn't a cost optimization — it's what stops the
product confidently grading a recording that contains nothing.

**Speech-to-text and analysis are separate models, both on Groq.** Whisper
transcribes, Llama analyzes. Keeping them separate means the analysis prompt
receives clean text and can be iterated on without touching transcription, and
Groq's inference speed is what makes a 10–15 second round trip feasible at all —
the transcription step alone is ~0.2s on their hardware.

**Filler-word detection is deterministic, not a model call.** Counting how often
someone says "basically" is arithmetic over Whisper's word segments, and a
`Counter` cannot miscount or invent a word the speaker never said. Two strategies
run over that list: words that are always fillers (`um`, `uh`, `basically`), and
words that only count as fillers once repeated past a threshold (`so`, `and`,
`like` — normal English until they aren't). Leaving this to the LLM would trade a
guaranteed-correct count for a plausible one.

**Config is centralized and type-safe.** Every secret and setting goes through
Pydantic `BaseSettings` rather than scattered `os.environ` reads, so a missing
variable fails loudly at startup instead of surfacing as a confusing runtime error
inside a Celery worker three containers away — which is exactly how it failed the
first time.

---

## Known Limitations

- Short disfluencies like "um" and "uh" are frequently dropped by Whisper during transcription, limiting filler detection accuracy for those specific words
- Text-to-speech uses gTTS which produces a synthetic voice rather than natural or cloned speech
- No GPU acceleration — processing runs on CPU-only Railway containers
- No email verification or password reset flow currently implemented
- Mobile app (React Native) not yet built
- History page loads all sessions at once — no pagination for large histories

---

## What I Learned Building This

This project was built as a learning exercise covering the full stack from scratch:

- Designing async architectures with Celery + Redis for long-running AI tasks
- Debugging distributed systems where backend and worker run in separate containers
- Managing secrets and environment variables across local, Docker, and production environments
- Docker multi-stage builds for minimal production images
- Implementing JWT-based auth with Supabase and protecting routes in Next.js App Router
- Using Pydantic BaseSettings for centralized, type-safe configuration
- Git Flow branching strategy with feature branches, PRs, and semantic commits
