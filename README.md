# Rhetor

> *The StudyApp where you learn by teaching — on the go.*

Rhetor is built around the Feynman technique: upload your study material, let
the system generate a guide and questions, then record yourself "teaching" it.
The app listens, watches, and prompts you when you drift off-topic or skip a
concept. Recordings stay yours — edit, archive, share, or delete.

The name comes from the Greek **rhetor** — *the one who teaches by speaking*.
That's exactly what the app helps you become.

This is an in-progress project. Current phase: **v0.1 scaffolding**.

## Stack at a glance

- **Frontend**: Next.js 15 + TypeScript + Tailwind
- **Backend**: Python 3.11 + FastAPI
- **ML (all local, all open source)**:
  - `faster-whisper` (small) for speech-to-text
  - `Qwen2.5-3B-Instruct` via MLX for the "teacher brain"
  - `bge-small-en-v1.5` for embeddings / content alignment
  - `PyMuPDF` for PDF text extraction
- **Storage**: local filesystem + SQLite for v0.1; R2/S3-compatible later

## Layout

```
backend/    Python API + ML pipeline
frontend/   Next.js web app (becomes desktop + mobile via Tauri/Capacitor later)
ml/         Notebooks, experiments, model-download scripts
docs/       Architecture notes — start here if you're new
```

## Where your data lives

**Outside this repo.** Default: `~/Rhetor/data/`. Override with the
`RHETOR_DATA_DIR` environment variable. See `docs/00-architecture.md`.

## Getting started

Setup instructions land in `docs/01-setup.md` once the backend skeleton is in
(step 2 of the build plan). Hold tight.
