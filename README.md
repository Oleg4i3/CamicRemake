# CaMic

A minimalist Android camera app for landscape recording — clean UI, serious audio, and a seamless pre-buffer that captures the moment before you hit REC.

---

## Features

### Video
- **1280×720 @ 30 fps**, H.264 (AVC Baseline)
- Adjustable bitrate: **500 kbps → 12 Mbps** — from lecture slides to fast action
- Saved to `DCIM/CaMic/` via MediaStore (Android 10+) or direct file path

### Audio
- **48 kHz**, stereo AAC (320 kbps stereo / 192 kbps mono)
- Hardware AGC, noise suppressor and echo canceler **disabled** — raw capture
- Selectable audio source: built-in mic, raw/unprocessed, USB, wired headset, Bluetooth
- Software **gain control** (−20 dB … +20 dB) with vertical slider
- **Soft clip** limiter (tanh knee) prevents clipping without losing dynamics

### Pre-buffer (seamless)
- Continuously encodes video + audio into a ring buffer while the app is open
- Press REC → the last 1–5 seconds appear at the start of the file with **no cut, no re-encode, no artifacts**
- Implemented via always-on MediaCodec encoders; ring→file switch happens atomically inside the drain thread — no race conditions, no gap between buffered and live frames

### Pause / Resume
- Pause button (⏸) appears above STOP during recording
- Resumes exactly at the press moment, no pre-buffer applied
- PTS timeline is corrected for pause duration — no jump or drift in the output file

### Camera controls
- **Digital zoom** via spring-return lever (right edge), exponential speed curve
- **Manual focus** with a drum-style scrubber; optional **Focus Assist** (auto-zooms while focusing, then restores)
- **AE exposure compensation** (EV slider, range read from camera hardware)
- Tap-to-focus / continuous-video AF

### Audio visualization
- **Envelope** (10-second scrolling amplitude history) — on by default
- **Oscilloscope** (waveform, toggleable)
- **Spectrum analyzer** (60-bin FFT, log frequency scale, peak hold)
- **VU meter** (vertical, RMS + peak)

---

## UI layout

```
┌─────────────────────────────────────────────┐
│  [Gain │ VU]  Camera preview 16:9   [Focus │ Zoom]  │
│         Envelope / Oscilloscope overlay      │
│─────────────────────────────────────────────│
│ ⚙  Status line                               │
│ ┌──── Settings (collapsible) ─────────────┐ │
│ │ Src: [─── spinner ──]  │ EV [────────] │ │
│ │ [Soft clip][Man.focus] │ Bps:[spinner] │ │
│ │ [Oscillo]  [Spectrum]  │               │ │
│ │ [Pre-buf] [──] X s     │               │ │
│ └────────────────────────────────────────┘ │
│  Spectrum analyzer          [⏸]  [● REC]   │
└─────────────────────────────────────────────┘
```

- **Gain slider** — full-height vertical bar on the left
- **Zoom lever** — spring-return on the right; drag up to zoom in
- **REC / STOP** — fixed position bottom-right, never moves regardless of settings state
- **PAUSE** — small button above REC, visible only during recording

---

## Requirements

- Android **8.0 (API 26)** or later
- Camera2 API
- `CAMERA` and `RECORD_AUDIO` permissions (requested at launch)

Single-file build: the entire app is `MainActivity.java`. No external libraries, no Gradle plugins beyond the Android SDK.

---

## Building

```bash
# Standard Android Studio build
./gradlew assembleDebug
```

Or open in Android Studio and run directly on a device. No special configuration needed.

---

## Architecture notes

### Pre-buffer: how seamless works

Most pre-buffer implementations either re-encode the buffer (introduces quality loss and a re-encode gap) or take a snapshot of encoded frames from a separate thread (introduces a race condition and a visible cut at the splice point).

CaMic uses a different approach: the encoder drain thread owns the ring buffer and the muxer write path. When REC is pressed, `doStart()` only sets an atomic `FLUSH` flag. The **next time the drain thread wakes up** (within one frame interval, ~33 ms), it:

1. Flushes the ring starting from the first I-frame
2. Writes the current frame (the one that triggered the flush)
3. Switches to `LIVE` mode

Since steps 1–3 happen sequentially in the same thread, with no context switch between buffered and live frames, there is no gap and no missing packet.

### PTS alignment

Audio uses `System.nanoTime() / 1000` as the PTS base. The camera sensor timestamps (used by the video encoder via the encoder surface) are also `CLOCK_MONOTONIC`. Both clocks share the same epoch, so `mMuxBasePts` (the PTS of the first video I-frame) subtracted uniformly from both tracks gives correct A/V sync without any heuristic offset.

### Pause implementation

On pause, both write modes revert to `RING` and the ring buffers are cleared. On resume, `mMuxBasePts` is advanced by the pause duration, so the muxed PTS timeline is continuous and players see no discontinuity.

---

## Recommended workflow

1. Enable **Airplane Mode** (the app reminds you on launch) to prevent notification interruptions
2. Leave **Pre-buffer** on (default: 1 s) — you can extend to 5 s in settings
3. Set bitrate to match your use case:
   - Static/lecture: **1–2 Mbps**
   - Normal handheld: **6 Mbps** (default)
   - Fast action / high detail: **8–12 Mbps**

---

## License

MIT
