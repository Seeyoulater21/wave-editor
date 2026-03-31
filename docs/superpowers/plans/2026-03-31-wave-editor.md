# Wave Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file client-side WAV processor that trims silence from spoken-word audio and outputs each word repeated twice with configurable silence gaps.

**Architecture:** Single `index.html` with inline CSS and JS. Uses Web Audio API for decode/playback, `<canvas>` for waveform drawing, manual WAV encoding for output, and JSZip (CDN) for batch ZIP download. No build step, no server.

**Tech Stack:** HTML5, vanilla JS (ES6+), Web Audio API, Canvas 2D, JSZip 3.x (CDN)

---

## File Structure

All code lives in one file:

- **Create:** `index.html` — the entire application (HTML structure + `<style>` block + `<script>` block)

The JS inside `<script>` is organized into these logical sections (top to bottom):

1. **State & constants** — `slots[]` array (20 entries), `AudioContext` instance, default settings
2. **WAV encoding** — `encodeWAV(audioBuffer)` → `ArrayBuffer`
3. **Silence detection** — `detectSpeech(audioBuffer, thresholdPercent)` → `{start, end}` sample indices
4. **Audio processing** — `processSlot(slotIndex)` — trim + assemble + encode
5. **Waveform drawing** — `drawWaveform(canvas, audioBuffer)` — min/max per-pixel
6. **Playback** — `playBuffer(audioBuffer)` — one-shot Web Audio playback
7. **File I/O** — drag-drop handlers, file input handlers, download helpers, ZIP export
8. **UI rendering** — `renderSlot(index)` — update DOM for a slot based on its state
9. **Init** — generate 20 slot DOM elements, wire up global controls

---

### Task 1: HTML skeleton + global settings UI

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create `index.html` with HTML structure and CSS**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Wave Editor</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body { font-family: system-ui, sans-serif; padding: 20px; max-width: 900px; margin: 0 auto; background: #fafafa; }
  h1 { margin-bottom: 16px; font-size: 1.5rem; }

  .settings {
    background: #fff; border: 1px solid #ddd; border-radius: 6px;
    padding: 16px; margin-bottom: 20px;
  }
  .settings-row { display: flex; gap: 16px; align-items: center; flex-wrap: wrap; margin-bottom: 8px; }
  .settings-row:last-child { margin-bottom: 0; }
  .settings label { font-size: 0.875rem; }
  .settings input[type="number"] { width: 70px; padding: 4px; border: 1px solid #ccc; border-radius: 3px; }
  .settings input[type="range"] { width: 150px; }
  .settings button { padding: 6px 16px; border: 1px solid #333; background: #333; color: #fff; border-radius: 3px; cursor: pointer; }
  .settings button:hover { background: #555; }

  .slot {
    background: #fff; border: 1px solid #ddd; border-radius: 6px;
    padding: 12px; margin-bottom: 8px;
  }
  .slot-header { display: flex; align-items: center; gap: 8px; margin-bottom: 8px; font-size: 0.875rem; }
  .slot-number { font-weight: bold; color: #666; min-width: 24px; }
  .slot-filename { color: #333; }

  .drop-zone {
    border: 2px dashed #ccc; border-radius: 4px; padding: 16px;
    text-align: center; color: #999; cursor: pointer; font-size: 0.875rem;
  }
  .drop-zone.dragover { border-color: #666; background: #f0f0f0; }

  .waveform-row { display: flex; align-items: center; gap: 8px; margin-bottom: 4px; font-size: 0.75rem; color: #666; }
  .waveform-row canvas { border: 1px solid #eee; border-radius: 2px; }

  .slot-actions { display: flex; gap: 8px; margin-top: 8px; }
  .slot-actions button { padding: 4px 12px; font-size: 0.8rem; border: 1px solid #ccc; background: #fff; border-radius: 3px; cursor: pointer; }
  .slot-actions button:hover { background: #f0f0f0; }

  .hidden { display: none; }
</style>
</head>
<body>

<h1>Wave Editor</h1>

<div class="settings">
  <div class="settings-row">
    <label>Lead: <input type="number" id="leadSilence" value="0.5" min="0" max="3" step="0.1">s</label>
    <label>Gap: <input type="number" id="gapDuration" value="1.0" min="0" max="5" step="0.1">s</label>
    <label>Trail: <input type="number" id="trailSilence" value="0.5" min="0" max="3" step="0.1">s</label>
  </div>
  <div class="settings-row">
    <label>Threshold: <input type="range" id="threshold" min="0.5" max="10" step="0.5" value="2">
    <span id="thresholdValue">2</span>%</label>
  </div>
  <div class="settings-row">
    <button id="processAllBtn">Process All</button>
    <button id="downloadAllBtn">Download All ZIP</button>
  </div>
</div>

<div id="slotsContainer"></div>

<script>
// === All JS goes here in subsequent tasks ===
</script>
</body>
</html>
```

- [ ] **Step 2: Open in browser to verify layout**

Run: `cd "/Users/pakdaesedmetapiphat/Mac/Claude Code University/wave-editor" && python3 -m http.server 8000`

Open `http://localhost:8000` — should see "Wave Editor" heading, settings panel with inputs, and an empty slots area. No console errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add HTML skeleton with global settings UI"
```

---

### Task 2: Generate 20 file slots with drag-and-drop

**Files:**
- Modify: `index.html` (inside `<script>`)

- [ ] **Step 1: Add state management and slot generation**

Replace the `// === All JS goes here ===` comment inside `<script>` with:

```javascript
'use strict';

const SLOT_COUNT = 20;
const SAMPLE_RATE = 44100;
const NUM_CHANNELS = 2;
const BIT_DEPTH = 16;

let audioCtx = null;
function getAudioCtx() {
  if (!audioCtx) audioCtx = new AudioContext({ sampleRate: SAMPLE_RATE });
  return audioCtx;
}

// Each slot: { file, fileName, originalBuffer, processedBuffer, processedWavBlob }
const slots = Array.from({ length: SLOT_COUNT }, () => ({
  file: null, fileName: '', originalBuffer: null, processedBuffer: null, processedWavBlob: null
}));

function getSettings() {
  return {
    lead: parseFloat(document.getElementById('leadSilence').value) || 0.5,
    gap: parseFloat(document.getElementById('gapDuration').value) || 1.0,
    trail: parseFloat(document.getElementById('trailSilence').value) || 0.5,
    threshold: parseFloat(document.getElementById('threshold').value) || 2,
  };
}

// Threshold slider live value display
document.getElementById('threshold').addEventListener('input', e => {
  document.getElementById('thresholdValue').textContent = e.target.value;
});

function createSlotElement(index) {
  const div = document.createElement('div');
  div.className = 'slot';
  div.id = `slot-${index}`;
  div.innerHTML = `
    <div class="slot-header">
      <span class="slot-number">#${index + 1}</span>
      <span class="slot-filename" id="filename-${index}"></span>
    </div>
    <div class="drop-zone" id="dropzone-${index}">
      Drop WAV file here or <label style="color:#337ab7;cursor:pointer;text-decoration:underline;">browse<input type="file" accept=".wav" class="hidden" id="fileinput-${index}"></label>
    </div>
    <div class="hidden" id="waveforms-${index}">
      <div class="waveform-row"><span>Original:</span> <canvas id="canvas-orig-${index}" width="400" height="60"></canvas></div>
      <div class="waveform-row"><span>Output:</span> <canvas id="canvas-proc-${index}" width="400" height="60"></canvas></div>
    </div>
    <div class="slot-actions hidden" id="actions-${index}">
      <button onclick="playOriginal(${index})">▶ Original</button>
      <button onclick="playProcessed(${index})">▶ Processed</button>
      <button onclick="downloadOne(${index})">Download</button>
    </div>
  `;

  // Drag and drop
  const dropzone = div.querySelector(`#dropzone-${index}`);
  dropzone.addEventListener('dragover', e => { e.preventDefault(); dropzone.classList.add('dragover'); });
  dropzone.addEventListener('dragleave', () => dropzone.classList.remove('dragover'));
  dropzone.addEventListener('drop', e => {
    e.preventDefault();
    dropzone.classList.remove('dragover');
    const file = e.dataTransfer.files[0];
    if (file && file.name.endsWith('.wav')) loadFile(index, file);
  });

  // File input
  div.querySelector(`#fileinput-${index}`).addEventListener('change', e => {
    const file = e.target.files[0];
    if (file) loadFile(index, file);
  });

  return div;
}

async function loadFile(index, file) {
  const slot = slots[index];
  slot.file = file;
  slot.fileName = file.name;
  slot.processedBuffer = null;
  slot.processedWavBlob = null;

  const arrayBuffer = await file.arrayBuffer();
  const ctx = getAudioCtx();
  slot.originalBuffer = await ctx.decodeAudioData(arrayBuffer);

  document.getElementById(`filename-${index}`).textContent = file.name;
  document.getElementById(`dropzone-${index}`).classList.add('hidden');
  document.getElementById(`waveforms-${index}`).classList.remove('hidden');

  drawWaveform(document.getElementById(`canvas-orig-${index}`), slot.originalBuffer);

  // Auto-process
  await processSlot(index);
}

// Generate all slots
const container = document.getElementById('slotsContainer');
for (let i = 0; i < SLOT_COUNT; i++) {
  container.appendChild(createSlotElement(i));
}
```

- [ ] **Step 2: Add placeholder functions so no errors on load**

Append after the slot generation code (still inside `<script>`):

```javascript
// Placeholder functions — implemented in later tasks
function drawWaveform(canvas, buffer) { /* Task 3 */ }
async function processSlot(index) { /* Task 4+5 */ }
function playOriginal(index) { /* Task 6 */ }
function playProcessed(index) { /* Task 6 */ }
function downloadOne(index) { /* Task 7 */ }
```

- [ ] **Step 3: Verify in browser**

Reload `http://localhost:8000` — should see 20 slots, each with "Drop WAV file here or browse". Drag a sample WAV onto slot 1 — filename should appear, drop zone should hide, waveform canvases should show (blank is ok). No console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add 20 file slots with drag-drop and file input"
```

---

### Task 3: Waveform drawing

**Files:**
- Modify: `index.html` (replace `drawWaveform` placeholder)

- [ ] **Step 1: Implement `drawWaveform`**

Replace the `drawWaveform` placeholder with:

```javascript
function drawWaveform(canvas, buffer) {
  const ctx = canvas.getContext('2d');
  const width = canvas.width;
  const height = canvas.height;
  const mid = height / 2;

  ctx.clearRect(0, 0, width, height);

  // Mix to mono for visualization
  const ch0 = buffer.getChannelData(0);
  const ch1 = buffer.numberOfChannels > 1 ? buffer.getChannelData(1) : ch0;
  const len = ch0.length;
  const samplesPerPixel = len / width;

  ctx.fillStyle = '#f8f8f8';
  ctx.fillRect(0, 0, width, height);

  ctx.strokeStyle = '#4a90d9';
  ctx.lineWidth = 1;
  ctx.beginPath();

  for (let x = 0; x < width; x++) {
    const start = Math.floor(x * samplesPerPixel);
    const end = Math.min(Math.floor((x + 1) * samplesPerPixel), len);
    let min = 1, max = -1;
    for (let i = start; i < end; i++) {
      const sample = (ch0[i] + ch1[i]) / 2;
      if (sample < min) min = sample;
      if (sample > max) max = sample;
    }
    const yMin = mid - max * mid;
    const yMax = mid - min * mid;
    ctx.moveTo(x, yMin);
    ctx.lineTo(x, yMax);
  }
  ctx.stroke();

  // Center line
  ctx.strokeStyle = '#ddd';
  ctx.lineWidth = 0.5;
  ctx.beginPath();
  ctx.moveTo(0, mid);
  ctx.lineTo(width, mid);
  ctx.stroke();
}
```

- [ ] **Step 2: Verify in browser**

Reload, drop `3_2_1.wav` onto slot 1. The "Original" canvas should show a waveform — a short speech burst followed by silence. The "Output" canvas stays blank (processing not implemented yet).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add waveform drawing on canvas"
```

---

### Task 4: Silence detection + audio assembly

**Files:**
- Modify: `index.html` (add `detectSpeech` and `assembleBuffer`, replace `processSlot` placeholder)

- [ ] **Step 1: Implement `detectSpeech`**

Add above the placeholder functions section:

```javascript
function detectSpeech(buffer, thresholdPercent) {
  const ch0 = buffer.getChannelData(0);
  const ch1 = buffer.numberOfChannels > 1 ? buffer.getChannelData(1) : ch0;
  const len = ch0.length;
  const chunkSize = Math.floor(SAMPLE_RATE * 0.01); // 10ms = 441 samples

  // Compute RMS per chunk, find peak
  const chunkCount = Math.floor(len / chunkSize);
  const rmsValues = new Float32Array(chunkCount);
  let peakRms = 0;

  for (let c = 0; c < chunkCount; c++) {
    let sum = 0;
    const offset = c * chunkSize;
    for (let i = 0; i < chunkSize; i++) {
      const sample = (ch0[offset + i] + ch1[offset + i]) / 2;
      sum += sample * sample;
    }
    rmsValues[c] = Math.sqrt(sum / chunkSize);
    if (rmsValues[c] > peakRms) peakRms = rmsValues[c];
  }

  const threshold = peakRms * (thresholdPercent / 100);
  let firstChunk = -1, lastChunk = -1;

  for (let c = 0; c < chunkCount; c++) {
    if (rmsValues[c] > threshold) {
      if (firstChunk === -1) firstChunk = c;
      lastChunk = c;
    }
  }

  if (firstChunk === -1) {
    // No speech found — return entire buffer
    return { startSample: 0, endSample: len };
  }

  return {
    startSample: firstChunk * chunkSize,
    endSample: Math.min((lastChunk + 1) * chunkSize, len)
  };
}
```

- [ ] **Step 2: Implement `assembleBuffer`**

Add right after `detectSpeech`:

```javascript
function assembleBuffer(originalBuffer, startSample, endSample, settings) {
  const ctx = getAudioCtx();
  const speechLength = endSample - startSample;
  const leadSamples = Math.floor(settings.lead * SAMPLE_RATE);
  const gapSamples = Math.floor(settings.gap * SAMPLE_RATE);
  const trailSamples = Math.floor(settings.trail * SAMPLE_RATE);
  const totalLength = leadSamples + speechLength + gapSamples + speechLength + trailSamples;

  const output = ctx.createBuffer(NUM_CHANNELS, totalLength, SAMPLE_RATE);

  for (let ch = 0; ch < NUM_CHANNELS; ch++) {
    const src = originalBuffer.getChannelData(ch);
    const dst = output.getChannelData(ch);
    // dst is already zeroed (silence)

    // First play
    let offset = leadSamples;
    for (let i = 0; i < speechLength; i++) {
      dst[offset + i] = src[startSample + i];
    }

    // Second play
    offset = leadSamples + speechLength + gapSamples;
    for (let i = 0; i < speechLength; i++) {
      dst[offset + i] = src[startSample + i];
    }
  }

  return output;
}
```

- [ ] **Step 3: Implement `processSlot`**

Replace the `processSlot` placeholder with:

```javascript
async function processSlot(index) {
  const slot = slots[index];
  if (!slot.originalBuffer) return;

  const settings = getSettings();
  const { startSample, endSample } = detectSpeech(slot.originalBuffer, settings.threshold);
  slot.processedBuffer = assembleBuffer(slot.originalBuffer, startSample, endSample, settings);
  slot.processedWavBlob = encodeWAV(slot.processedBuffer);

  drawWaveform(document.getElementById(`canvas-proc-${index}`), slot.processedBuffer);
  document.getElementById(`actions-${index}`).classList.remove('hidden');
}
```

- [ ] **Step 4: Add `encodeWAV` placeholder**

Add above `detectSpeech` (will be implemented in Task 5):

```javascript
function encodeWAV(buffer) { /* Task 5 */ return null; }
```

- [ ] **Step 5: Verify in browser**

Reload, drop `3_2_1.wav`. Both waveforms should render — the output waveform should show: silence → speech → silence → speech → silence. Actions buttons should appear. Console should show no errors (download won't work yet since `encodeWAV` returns null).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add silence detection and audio assembly"
```

---

### Task 5: WAV encoding

**Files:**
- Modify: `index.html` (replace `encodeWAV` placeholder)

- [ ] **Step 1: Implement `encodeWAV`**

Replace the `encodeWAV` placeholder with:

```javascript
function encodeWAV(audioBuffer) {
  const numChannels = audioBuffer.numberOfChannels;
  const sampleRate = audioBuffer.sampleRate;
  const numFrames = audioBuffer.length;
  const bytesPerSample = BIT_DEPTH / 8;
  const blockAlign = numChannels * bytesPerSample;
  const dataSize = numFrames * blockAlign;
  const headerSize = 44;
  const buffer = new ArrayBuffer(headerSize + dataSize);
  const view = new DataView(buffer);

  // Helper: write string
  function writeString(offset, str) {
    for (let i = 0; i < str.length; i++) view.setUint8(offset + i, str.charCodeAt(i));
  }

  // RIFF header
  writeString(0, 'RIFF');
  view.setUint32(4, 36 + dataSize, true);
  writeString(8, 'WAVE');

  // fmt chunk
  writeString(12, 'fmt ');
  view.setUint32(16, 16, true);              // chunk size
  view.setUint16(20, 1, true);               // PCM format
  view.setUint16(22, numChannels, true);
  view.setUint32(24, sampleRate, true);
  view.setUint32(28, sampleRate * blockAlign, true); // byte rate
  view.setUint16(32, blockAlign, true);
  view.setUint16(34, BIT_DEPTH, true);

  // data chunk
  writeString(36, 'data');
  view.setUint32(40, dataSize, true);

  // Interleave channels and write samples
  const channels = [];
  for (let ch = 0; ch < numChannels; ch++) {
    channels.push(audioBuffer.getChannelData(ch));
  }

  let offset = headerSize;
  for (let i = 0; i < numFrames; i++) {
    for (let ch = 0; ch < numChannels; ch++) {
      let sample = channels[ch][i];
      sample = Math.max(-1, Math.min(1, sample));
      const intSample = sample < 0 ? sample * 32768 : sample * 32767;
      view.setInt16(offset, intSample, true);
      offset += 2;
    }
  }

  return new Blob([buffer], { type: 'audio/wav' });
}
```

- [ ] **Step 2: Verify in browser**

Reload, drop `3_2_1.wav`. Processing should complete with no errors. The download button is wired up in Task 7, but we can verify encoding by checking that `slots[0].processedWavBlob` is a Blob with reasonable size. In the browser console:

```javascript
slots[0].processedWavBlob.size
// Should be > 100000 (roughly: 44 + (0.5+0.66+1.0+0.66+0.5)*44100*2*2 ≈ 585k bytes)
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add WAV encoding (PCM 16-bit stereo)"
```

---

### Task 6: Playback

**Files:**
- Modify: `index.html` (replace `playOriginal` and `playProcessed` placeholders)

- [ ] **Step 1: Implement playback functions**

Replace both playback placeholders with:

```javascript
let currentSource = null;

function stopPlayback() {
  if (currentSource) {
    try { currentSource.stop(); } catch (e) {}
    currentSource = null;
  }
}

function playBuffer(audioBuffer) {
  stopPlayback();
  const ctx = getAudioCtx();
  const source = ctx.createBufferSource();
  source.buffer = audioBuffer;
  source.connect(ctx.destination);
  source.onended = () => { currentSource = null; };
  source.start();
  currentSource = source;
}

function playOriginal(index) {
  const slot = slots[index];
  if (slot.originalBuffer) playBuffer(slot.originalBuffer);
}

function playProcessed(index) {
  const slot = slots[index];
  if (slot.processedBuffer) playBuffer(slot.processedBuffer);
}
```

- [ ] **Step 2: Verify in browser**

Reload, drop `3_2_1.wav`. Click "▶ Original" — should hear the original word once. Click "▶ Processed" — should hear: 0.5s silence → word → 1s silence → word → 0.5s silence. Clicking a play button while another is playing should stop the first.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add audio playback for original and processed"
```

---

### Task 7: Download individual + ZIP all + Process All

**Files:**
- Modify: `index.html` (replace `downloadOne` placeholder, wire up global buttons)

- [ ] **Step 1: Implement download and global actions**

Replace the `downloadOne` placeholder and add the global button handlers. Place after the playback functions:

```javascript
function downloadOne(index) {
  const slot = slots[index];
  if (!slot.processedWavBlob) return;
  const baseName = slot.fileName.replace(/\.wav$/i, '');
  const url = URL.createObjectURL(slot.processedWavBlob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `${baseName}_processed.wav`;
  a.click();
  URL.revokeObjectURL(url);
}

document.getElementById('processAllBtn').addEventListener('click', async () => {
  for (let i = 0; i < SLOT_COUNT; i++) {
    if (slots[i].originalBuffer) await processSlot(i);
  }
});

document.getElementById('downloadAllBtn').addEventListener('click', async () => {
  const zip = new JSZip();
  let count = 0;
  for (let i = 0; i < SLOT_COUNT; i++) {
    const slot = slots[i];
    if (slot.processedWavBlob) {
      const baseName = slot.fileName.replace(/\.wav$/i, '');
      zip.file(`${baseName}_processed.wav`, slot.processedWavBlob);
      count++;
    }
  }
  if (count === 0) { alert('No processed files to download.'); return; }
  const blob = await zip.generateAsync({ type: 'blob' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'wave_editor_output.zip';
  a.click();
  URL.revokeObjectURL(url);
});
```

- [ ] **Step 2: Verify in browser**

1. Reload, drop `3_2_1.wav` on slot 1 and `3_2_2.wav` on slot 2.
2. Click "Download" on slot 1 — should download `3_2_1_processed.wav`.
3. Open the downloaded file in any audio player — should hear word twice with gap.
4. Click "Download All ZIP" — should download `wave_editor_output.zip` containing both processed files.
5. Change "Gap" to 2.0, click "Process All" — output waveforms should update with a wider gap. Download and verify.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add individual download, ZIP export, and process-all"
```

---

### Task 8: Final cleanup and remove placeholders

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Remove any remaining placeholder comments**

Scan the `<script>` section for any leftover `/* Task N */` comments and remove them. Ensure all functions are defined before they're called (reorder if needed — the final order top-to-bottom should be):

1. Constants + state + `getSettings` + threshold listener
2. `getAudioCtx`
3. `encodeWAV`
4. `detectSpeech`
5. `assembleBuffer`
6. `drawWaveform`
7. `stopPlayback` + `playBuffer` + `playOriginal` + `playProcessed`
8. `processSlot`
9. `downloadOne`
10. `loadFile`
11. `createSlotElement`
12. Slot generation loop
13. Global button event listeners

- [ ] **Step 2: End-to-end test with all 3 sample files**

1. Open `http://localhost:8000`
2. Drop `3_2_1.wav` on slot 1, `3_2_2.wav` on slot 2, `3_2_3.wav` on slot 3
3. Verify all three auto-process (both waveforms drawn, action buttons visible)
4. Play each original and processed — verify they sound correct
5. Change threshold to 5%, click "Process All" — waveforms should update
6. Change lead to 1.0s, gap to 2.0s — "Process All" — verify longer silences
7. "Download All ZIP" — unzip and verify 3 files, each plays correctly
8. Verify the downloaded WAV files are valid: `ffprobe 3_2_1_processed.wav` should show PCM 16-bit stereo 44100Hz

- [ ] **Step 3: Test opening directly from filesystem**

Double-click `index.html` (or open via `file://` URL). Drop a WAV file. Processing and playback should work. ZIP download should work (JSZip loads from CDN, so requires internet but not a local server).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: complete wave editor — cleanup and final ordering"
```
