# Wave Editor — Design Spec

## Purpose

Client-side web tool for batch processing spoken-word WAV files used in language-learning materials. Each source file contains one word. The tool trims silence, then outputs: `[lead silence] [word] [gap] [word] [trail silence]` — so learners hear the word twice.

## Architecture

- Single `index.html` file — all HTML, CSS, and JS inline. No build step.
- Only external dependency: JSZip (from CDN) for ZIP download.
- Hosted on GitHub Pages. Also works opened directly from filesystem.

## Input Format

- WAV: PCM 16-bit, stereo, 44100 Hz
- Each file contains a single spoken word (~0.5–1.0s of speech with surrounding silence)

## Output Format

- WAV: PCM 16-bit, stereo, 44100 Hz
- Filename: `{original}_processed.wav`
- Structure: `[0.5s silence] [trimmed word] [1.0s gap] [trimmed word] [0.5s silence]`

## Global Settings (applies to all files)

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| Lead silence | 0.5s | 0–3s | Silence before first word |
| Gap | 1.0s | 0–5s | Silence between two plays |
| Trail silence | 0.5s | 0–3s | Silence after second word |
| Silence threshold | 2% | 0.5–10% | RMS threshold (% of peak) for silence detection |

## UI Layout

```
┌─────────────────────────────────────────────┐
│  Wave Editor                                │
│                                             │
│  ┌─ Global Settings ──────────────────────┐ │
│  │ Lead: [0.5s]  Gap: [1.0s]  Trail:[0.5s]│ │
│  │ Threshold: [====o========] 2%           │ │
│  │ [Process All]  [Download All ZIP]       │ │
│  └─────────────────────────────────────────┘ │
│                                             │
│  ┌─ Slot 1 ───────────────────────────────┐ │
│  │ [Drop file here / Browse]  3_2_1.wav   │ │
│  │ Original: ▁▂▅█▇▅▂▁▁▁▁▁▁▁▁            │ │
│  │ Output:   ▁▁▁▂▅█▇▅▂▁▁▁▁▂▅█▇▅▂▁▁▁    │ │
│  │ [▶ Original] [▶ Processed] [Download]  │ │
│  └─────────────────────────────────────────┘ │
│  ┌─ Slot 2 ───────────────────────────────┐ │
│  │ [Drop file here / Browse]              │ │
│  └─────────────────────────────────────────┘ │
│  ... (20 slots) ...                         │
└─────────────────────────────────────────────┘
```

- Simple, functional UI — native HTML elements, minimal CSS
- Each slot shows: filename, original waveform, processed waveform, play buttons, download button
- Empty slots show drop zone / browse button only

## Core Processing Pipeline

1. **Load**: User drops/selects WAV → `FileReader` → `AudioContext.decodeAudioData()` → `AudioBuffer`
2. **Waveform (original)**: Draw min/max per-pixel on `<canvas>` (~400x60px)
3. **Silence detection**: Scan mono-mixed samples in 10ms chunks, compute RMS per chunk, mark chunks above threshold as "speech"
4. **Trim**: Extract sample range from first speech chunk to last speech chunk
5. **Assemble**: Create new buffer = lead silence + trimmed word + gap + trimmed word + trail silence
6. **Waveform (processed)**: Draw assembled buffer on second `<canvas>`
7. **Encode WAV**: Write 44-byte RIFF/WAV header + interleaved 16-bit PCM samples
8. **Download**: Individual file via `<a download>` blob URL, or ZIP all via JSZip

## WAV Encoding (JS)

44-byte header structure:
- RIFF chunk descriptor (12 bytes)
- fmt sub-chunk (24 bytes): PCM format, 2 channels, 44100 Hz, 16-bit
- data sub-chunk header (8 bytes) + raw PCM samples

Samples clamped to [-1, 1] then scaled to int16 range [-32768, 32767].

## Silence Detection Algorithm

1. Mix stereo to mono: `(left + right) / 2`
2. Divide into 10ms chunks (441 samples at 44100Hz)
3. Compute RMS for each chunk: `sqrt(mean(samples²))`
4. Find peak RMS across all chunks
5. Threshold = `peak_rms * (threshold_percent / 100)`
6. First chunk above threshold = speech start
7. Last chunk above threshold = speech end
8. Trim to speech start → speech end

## File Slot States

1. **Empty**: Shows drop zone + browse button
2. **Loaded**: Shows filename + original waveform. Ready to process.
3. **Processed**: Shows both waveforms + play buttons + download button

## Actions

- **Process All**: Process every loaded slot with current global settings
- **Download All ZIP**: ZIP all processed files, download as `wave_editor_output.zip`
- **Per-slot Download**: Download individual `_processed.wav`
- **Per-slot Play Original**: Play original audio via Web Audio API
- **Per-slot Play Processed**: Play processed audio via Web Audio API

## Auto-process Behavior

When a file is dropped/selected, it auto-processes immediately with current settings. "Process All" re-processes everything (useful after changing settings).
