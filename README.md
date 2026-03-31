# Wave Editor

Client-side web tool for batch processing spoken-word WAV audio files. Designed for language-learning materials where each source file contains a single word.

**Live:** [https://seeyoulater21.github.io/wave-editor/](https://seeyoulater21.github.io/wave-editor/)

## What it does

1. Trims leading/trailing silence from each WAV file
2. Outputs the word repeated twice with configurable gaps:

```
[lead silence] [word] [gap] [word] [trail silence]
```

Default timing: 0.5s lead → word → 1.0s gap → word → 0.5s trail

## Features

- Drag & drop single or multiple WAV files (up to 20 slots)
- Automatic silence detection and trimming
- Waveform visualization (original + processed)
- Preview playback for both original and processed audio
- Download individually or as a ZIP
- All processing runs in the browser — no server needed

## Usage

1. Open the page (or `index.html` directly)
2. Drag WAV files onto the batch drop zone or into individual slots
3. Files are auto-processed with current settings
4. Adjust Lead / Gap / Trail / Threshold as needed, then click "Process All"
5. Download individual files or "Download All ZIP"

## Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| Lead | 0.5s | Silence before first word |
| Gap | 1.0s | Silence between two plays |
| Trail | 0.5s | Silence after second word |
| Threshold | 2% | Silence detection sensitivity (% of peak RMS) |

## Input format

WAV files: PCM 16-bit, stereo, 44100 Hz

## Development

```bash
# Serve locally
python3 -m http.server 8000
# Open http://localhost:8000
```

Single `index.html` file — no build step, no dependencies (JSZip loaded from CDN).
