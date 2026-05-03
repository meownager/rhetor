# Architecture — the big picture

This is the map. Read it once, come back when you're lost.

## What we're building

**Rhetor** — the StudyApp where you learn by teaching. The user uploads a PDF,
the system turns it into a study guide with comprehension questions, the user
studies it, then records themselves "teaching" the material. The system listens
to the teaching session in real time, compares what's being said against the
source PDF, and injects cues when the user drifts, gets something wrong, or
skips a concept.

The name comes from the Greek *rhetor* — the one who teaches by speaking.

## The flow, end to end

```
┌──────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────┐
│  UPLOAD  │ → │  INGEST  │ → │  GUIDE+QUIZ  │ → │  TEACH MODE  │ → │ ARCHIVE  │
└──────────┘   └──────────┘   └──────────────┘   └──────────────┘   └──────────┘
   PDF in      Extract text     LLM produces       Record video,      Compress,
   (≤25 MB,    chunks, embed    study guide +      live transcript,   store,
   ≤100 pp)    each chunk       comprehension      content-mapping,   list/play/
               into vectors     questions          Socratic cues      delete
```

## Why each piece exists

### Frontend (Next.js)
The browser already knows how to capture camera, mic, and screen via
`getUserMedia` + `MediaRecorder`. We don't have to write any of that. Next.js
gives us routing, server components for SEO later, and one codebase that wraps
into desktop (Tauri) and mobile (Capacitor) when the time comes.

### Backend (FastAPI)
Python is where ML lives. Every model we use has Python bindings. FastAPI is
async, so a slow LLM call doesn't block transcription. It also auto-generates
API docs at `/docs` — useful when you're learning what calls exist.

### The four models, and why each was chosen

| Model | Job | Size | Why |
|---|---|---|---|
| **faster-whisper (small)** | speech → text | ~500 MB | Fastest open Whisper. CTranslate2 backend. Real-time on M4. |
| **Qwen2.5-3B via MLX** | teacher brain | ~2 GB | MLX uses Apple's unified memory — no copying between CPU/GPU. 3B is the sweet spot for instruction quality at low RAM. |
| **bge-small-en-v1.5** | text embeddings | 130 MB | Top of the MTEB leaderboard for its size. Used to score whether the user is covering the material. |
| **PyMuPDF** | PDF → text | tiny | Not ML. Fast, accurate text extraction. |

Total RAM ceiling during a session: ~3–4 GB. Leaves room on a 16 GB Mac.

## Where data lives

**Outside the repo.** Three reasons:
1. Recordings are private user data — must never be in git.
2. Models are huge (~2.6 GB total) — git hates big binaries.
3. Separating data from code means you can `rm -rf` the repo and your
   recordings survive.

Default location: `~/Rhetor/data/` on macOS/Linux. Configurable via the
`RHETOR_DATA_DIR` environment variable.

```
~/Rhetor/data/
├── uploads/        Original PDFs (kept for re-processing)
├── recordings/     Compressed video/audio files
├── models/         Downloaded ML model weights (one-time download)
└── rhetor.db       SQLite — metadata, transcripts, study guides
```

### The Storage abstraction

Today we write to disk. Tomorrow we might write to Cloudflare R2 (free
egress) or Google Drive. To make that swap painless, all file I/O goes
through a `Storage` interface (`backend/app/core/storage.py`). The rest of
the app never sees a raw `open()` call. This is the **Repository pattern** —
one of the most useful design patterns you'll learn.

## Compression strategy

### PDFs (input)
- Hard limit: 25 MB, 100 pages
- Reject scanned/image-only PDFs in v0.1 (no extractable text)
- Store original; cache extracted text in DB to avoid re-parsing

### Recordings (output)
- Browser records: WebM, VP9 video, Opus audio, 720p / 24 fps
- Backend re-encodes to AV1 in the background (FFmpeg) → 60–70% smaller
- Audio-only mode: Opus mono 32 kbps — ~0.5 MB/min
- Video mode: ~2–3 MB/min after AV1 re-encode

A 30-minute session lands at 60–90 MB video, ~15 MB audio-only.

## Privacy defaults

- Camera: **off** by default. User must opt in.
- Mic: required (it's the whole point).
- Screen share: off by default, opt-in.
- All processing local. No data leaves your machine in v0.1.

## What's NOT in v0.1 (intentionally)

- User accounts / auth
- Sharing / public links
- Video editor (just record, save, delete)
- Vision model (Moondream / VLM) — added in v0.2 once core works
- Mobile / desktop wrappers — web only first
- OCR for scanned PDFs

We add these once the core loop is rock solid. Premature features make
projects collapse under their own weight.

## Build order

Each step ends with something runnable. Don't skip ahead.

1. **Scaffolding + git hygiene** ← you are here
2. Backend skeleton — FastAPI runs, hello-world endpoint
3. Frontend skeleton — Next.js renders, calls backend
4. Recording UI — camera/mic/screen toggles, save locally
5. Whisper integration — transcribe a saved clip
6. LLM integration via MLX — first generated study guide
7. PDF upload + chunking + embedding
8. Wire it all: live transcript + content-mapping + Socratic cues

## A note on "heavy engineering"

You asked for a project that teaches you serious AI/ML engineering. That
means we'll deliberately practice patterns even when a shortcut would work:

- **Typed everywhere** — Python type hints + TypeScript. Catches bugs early.
- **Tested** — pytest for backend, Playwright for frontend later.
- **Configured, not hardcoded** — pydantic-settings for env vars.
- **Logged, not printed** — structlog from day one.
- **Async where it matters** — transcription shouldn't block the API.
- **Abstractions at boundaries** — Storage interface, ML model interface.

When a pattern feels like overkill for a tiny app, that's the point. You're
training the muscle.
