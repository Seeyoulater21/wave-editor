# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Wave Editor** — a static web tool hosted on **GitHub Pages** for batch processing WAV audio files used in language-learning materials.

**Core workflow:** Each source file contains a single spoken word. The tool trims leading/trailing silence, then produces output as: `[silence] [word] [gap] [word] [silence]` — so learners hear the word twice with a pause in between.

Default timing: 0.5s lead silence → play → 1.0s gap → play → 0.5s trail silence. All values are user-configurable.

## Architecture

- **Pure client-side** — no backend, no build step. All audio processing runs in the browser via the Web Audio API.
- Hosted as static files on GitHub Pages (HTML + JS + CSS).
- Supports up to **20 files** per batch ("branch"), each processed independently.
- Files are drag-and-dropped or selected one at a time; output can be downloaded individually or as a ZIP.
- Input/output format: WAV (PCM 16-bit, stereo, 44100 Hz).

## Development

```bash
# Local dev — just serve the directory
npx serve .
# or
python3 -m http.server 8000
```

No build, lint, or test commands exist yet. If added, document them here.

## Deployment

Push to `main` branch → GitHub Pages serves from repo root (or `/docs` if configured).

## Sample Files

`3_2_*.wav` — sample spoken-word WAV files (stereo, 16-bit PCM, 44100 Hz, ~1.7s each) used for testing.

## Key Constraints

- All processing must be client-side (Web Audio API / AudioContext).
- WAV encoding/decoding in JS — no server-side ffmpeg.
- UI must clearly show all 20 file slots with per-file controls.
- Silence detection threshold and timing parameters must be adjustable.
- Output files must remain valid WAV format.
