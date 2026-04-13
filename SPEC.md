# DJ Drop Studio — Technical Specification

> A browser-based, single-file application for producing professional-quality radio DJ drops.  
> Stack: Vanilla HTML/CSS/JS · Web Audio API · WaveSurfer.js · Firebase (Firestore + Storage)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Tech Stack & Dependencies](#2-tech-stack--dependencies)
3. [Layout & UI Architecture](#3-layout--ui-architecture)
4. [Track System](#4-track-system)
5. [Audio Engine](#5-audio-engine)
6. [FX Processing Chain](#6-fx-processing-chain)
7. [Waveform Viewer & Region Trimming](#7-waveform-viewer--region-trimming)
8. [Local Persistence (IndexedDB)](#8-local-persistence-indexeddb)
9. [Cloud Library (Firebase)](#9-cloud-library-firebase)
10. [Settings Persistence (localStorage)](#10-settings-persistence-localstorage)
11. [Render & Export Pipeline](#11-render--export-pipeline)
12. [Design System & CSS Tokens](#12-design-system--css-tokens)
13. [State Variables Reference](#13-state-variables-reference)
14. [Key Functions Reference](#14-key-functions-reference)
15. [Known Behaviours & Gotchas](#15-known-behaviours--gotchas)
16. [File Structure](#16-file-structure)

---

## 1. Overview

**DJ Drop Studio** is a tool for radio presenters and producers to create fully mixed "DJ drops" — short audio idents that combine:

- An **Intro SFX** (stinger/jingle before the voice)
- A **Voice recording** (the "drop" itself, selected from a local library)
- An **Outro SFX** (hit after the voice)
- A **Music Bed** (background music that ducks under the voice)

All audio is mixed entirely in the browser using the **Web Audio API's OfflineAudioContext**, then played back and exported as a **16-bit PCM WAV file**. No server-side processing is required.

---

## 2. Tech Stack & Dependencies

| Dependency | Version | Source | Purpose |
|---|---|---|---|
| WaveSurfer.js | 7.x | `unpkg.com` | Waveform visualisation & region-based trimming |
| WaveSurfer Regions Plugin | 7.x | `unpkg.com` | Drag-to-trim selection regions |
| Firebase App | 11.6.1 | `gstatic.com` (ESM) | Firebase initialisation |
| Firebase Auth | 11.6.1 | `gstatic.com` (ESM) | Anonymous sign-in for cloud access |
| Firebase Firestore | 11.6.1 | `gstatic.com` (ESM) | Cloud SFX library metadata |
| Firebase Storage | 11.6.1 | `gstatic.com` (ESM) | Cloud SFX audio file storage |
| Google Fonts – Inter | Latest | `fonts.googleapis.com` | UI typography |

**No build step required.** The entire application is a single `index.html` file.

---

## 3. Layout & UI Architecture

### 3.1 Top-Level Layout

```
body
└── .studio-container  (CSS Grid: 260px sidebar | 1fr main)
    ├── aside.sidebar  (sticky, full-height)
    └── main.mixer     (scrollable column)
```

- Container is capped at `max-width: 1400px`, centred in the viewport.
- The sidebar is `position: sticky; top: 16px; height: calc(100vh - 32px)`.

### 3.2 Sidebar

Contains two tabs:

| Tab ID | Label | Visibility |
|---|---|---|
| `#tabLocal` | My Crate | Always visible |
| `#tabCloud` | Cloud SFX | Hidden until Firebase auth succeeds |

**My Crate tab:**
- Drop zone / click-to-upload area (`#localDropZone`, `#voiceUploader`)
- Voice list (`#voiceList`) — populated from IndexedDB on load
- Footer note: "Files saved to Browser Memory (IndexedDB)"

**Cloud SFX tab:**
- Community Library description
- Batch Upload button (`#batchUploadBtn` → `#batchUploadInput`)
- Cloud list (`#cloudList`) — real-time Firestore snapshot

### 3.3 Main Mixer

Vertical stack of **Track Cards** inside `main.mixer`:

```
.top-row (CSS Grid 1fr 1fr)
├── Track 1 — Intro SFX
└── Track 3 — Outro SFX
Track 2 — Voice Core
Track 4 — Music Bed
.transport (sticky bottom bar)
```

### 3.4 Transport Bar

Sticky footer containing:
- **Visualiser** (`#progressBar` inside `.viz-container`) — progress indicator
- **Status text** (`#statusText`) — "Ready to Render" / "Rendering..." / "Playing..." / "Done!" / "Saving..."
- **▶ Render & Play** button (`#generateBtn`) — orange, pulsing glow
- **⬇ Save** button (`#downloadBtn`) — disabled until a render exists

---

## 4. Track System

### Track 1 — Intro SFX

| Property | Value |
|---|---|
| CSS class | `.track-intro` (green left border) |
| File input | `#introFile` (audio/*) |
| Clear button | `#clearIntroBtn` |
| Status display | `#introStatus` |
| Waveform container | `#introWave` |
| Volume slider | `#introVol` (0–1.5, default 0.8) |
| Auto-Duck toggle | `#introDuck` checkbox |

### Track 2 — Voice Core

| Property | Value |
|---|---|
| CSS class | `.track-voice` (purple left border) |
| Voice picker | Clicking items in `#voiceList` |
| Selected name display | `#selectedVoiceName` |

Contains the **FX Rack** (4 FX cards in a 4-column grid):

| Card | Toggle | Controls |
|---|---|---|
| 🚀 Build-Up | `#buildUpOn` | Fixed: 4 stutters × 150ms, Pitch+Filter+Flange chain |
| 💎 Pro Width | `#widthOn` | `#widthAmt` slider (0–1, Stereo Width); fixed Radio EQ (presence + air) |
| 🌌 Tail FX | `#tailOn` | `#tailStart` (0.1–0.9), `#tailMix` (0–1) |
| 🎚️ Voice Level | — | Offset `#voiceOffset` (-1–10s), Volume `#voiceVol` (0–5), Pitch `#voicePitch` (-12–+12 semitones), End Trim `#endTrim` (-10–+5s); 🔥 Boost toggle `#micBoost` |

### Track 3 — Outro SFX

| Property | Value |
|---|---|
| CSS class | `.track-outro` (purple/violet left border) |
| File input | `#outroFile` (audio/*) |
| Clear button | `#clearOutroBtn` |
| Status display | `#outroStatus` |
| Waveform container | `#outroWave` |
| Offset slider | `#outroOffset` (-5–+5s, default 0) |
| Volume slider | `#outroVol` (0–1.5, default 0.8) |
| Auto-Duck toggle | `#outroDuck` checkbox |

### Track 4 — Music Bed

| Property | Value |
|---|---|
| CSS class | `.track-master` (cyan left border) |
| File input | `#bedFile` (audio/*) |
| Clear button | `#clearBedBtn` |
| Status display | `#bedStatus` |
| Waveform container | `#bedWave` |
| Volume slider | `#bedVol` (0–1, default 0.5) |
| Auto-Duck toggle | `#autoDuck` checkbox |
| Active toggle | `#bedOn` checkbox (enables/disables bed entirely) |

---

## 5. Audio Engine

### 5.1 Audio Context

```js
const ctx = new (window.AudioContext || window.webkitAudioContext)();
```

Created at page load. Resumed on first user gesture (button click) to satisfy browser autoplay policies.

### 5.2 Audio Loading

`loadAudioFile(file | blob)` — reads the file as an `ArrayBuffer`, then calls `ctx.decodeAudioData()`. Returns a decoded `AudioBuffer` or `null`.

### 5.3 Auto-Start Detection

`findAudioStart(buffer)` — scans channel 0 sample-by-sample for the first sample exceeding amplitude `0.015`. Returns the timestamp in seconds. Used to skip leading silence in the voice track.

### 5.4 Reverb Impulse Response

`createReverbImpulse(duration)` — programmatically generates a stereo exponential-decay noise IR (`2s` by default). Used by the Tail FX convolver node. Created once at startup and stored in `reverbBuffer`.

### 5.5 Render Architecture

All mixing happens in an **OfflineAudioContext** (2ch, 44100 Hz implied by `ctx.sampleRate`).

The signal graph inside `OfflineAudioContext`:

```
[Intro Src] ──────────────────────────────────► introGain ─┐
[Build-Up Stutter Sources] → hpf → dly → bg → (boost?) ───┤
[Main Voice Src] → (EQ chain?) → (boostGain → vocalComp?) → vg ─┤
[Width L/R Sources] → delays → merger → wg ────────────────┤
[Tail FX tap] → delays ↔ feedback → merger ────────────────┤  masterComp → masterGain → destination
[Tail FX tap] → convolver (reverb) ────────────────────────┤
[Outro Src] ──────────────────────────────────► outroGain ─┤
[Bed Src] ────────────────────────────────────► bedGain ───┘
```

- **masterComp**: DynamicsCompressor (-12 dB threshold, 30 knee, 4:1 ratio, 3ms attack, 250ms release)
- **masterGain**: Linear fade-out from `fadeStart` → `totalDuration`

---

## 6. FX Processing Chain

### 6.1 Build-Up (Pre-Voice Stutter)

- Triggered if `#buildUpOn` is checked
- Fires `stutterCount = 4` stutter segments, each `stutterLen = 150ms` long
- Each stutter starts from `cuePoint` of the voice buffer
- Pitch sweeps from `-300 cents` to `0 cents` (relative to voice pitch)
- Signal path: `BufferSource → HPF (200Hz→1kHz ramp) → Flanger delay (5ms, 3Hz LFO, 2-cent depth) → gain → [optional boost] → masterComp`
- Total build-up duration: `4 × 0.150 = 0.600s`

### 6.2 Pro Width (Stereo Widening + Radio EQ)

- Triggered if `#widthOn` is checked
- **Radio EQ** on main voice: Peaking +4dB @ 3kHz (presence), High-shelf +5dB @ 10kHz (air)
- **Width Effect**: Two additional mono sources (L: -25 cents, R: +25 cents), delayed (L: 18ms, R: 28ms), merged stereo, blended at `widthAmt × 0.85`

### 6.3 Mic Boost

- Triggered if `#micBoost` is checked
- Pre-gain: `+2.5×` (≈ +8dB)
- Followed by hard-knee DynamicsCompressor: -24dB threshold, 10 knee, 12:1 ratio, 2ms attack, 100ms release
- **Also** applied to Build-Up path: gain `6.0×`, compressor (-30dB threshold, 10 knee, 16:1 ratio, 2ms/100ms)

### 6.4 Tail FX (Ping-Pong Delay + Reverb)

- Triggered if `#tailOn` is checked
- Tap taken from voice gain output (`vg`) into a muted gain `ts`
- `ts` fades from `0` → `tailMix` over the last portion of the voice (starting at `tailStart%` through to voice end)
- **Ping-Pong Delay**: L 300ms / R 450ms, cross-feedback 0.4 each
- **Convolver**: Uses the programmatic reverb IR

### 6.5 Auto-Ducking

Applied during offline render if the corresponding checkbox is checked. Three independent ducking envelopes:

| Target | When | Depth |
|---|---|---|
| Music Bed | During entire voice segment | 70% reduction (to `vol × 0.3`) |
| Intro SFX | If intro overlaps voice start | 70% reduction |
| Outro SFX | If outro starts before voice ends | 70% reduction |

All duck-in transitions: `0.3s linear ramp`. All duck-out (restore): `1.0s linear ramp`.

---

## 7. Waveform Viewer & Region Trimming

### 7.1 WaveSurfer Config

| Option | Intro | Outro | Bed |
|---|---|---|---|
| `waveColor` | `#0ea5e9` (blue) | `#f43f5e` (red) | `#06b6d4` (cyan) |
| `height` | 60px | 60px | 60px |
| `barWidth` / `barGap` | 2 / 1 | 2 / 1 | 2 / 1 |
| `normalize` | true | true | true |
| `interact` | false | false | false |

Regions plugin is registered on each instance. A single full-length region is created on `ready`.

### 7.2 Trim State

```js
let introTrim = { start: 0, end: 0 }; // seconds
let outroTrim = { start: 0, end: 0 };
let bedTrim   = { start: 0, end: 0 };
```

Updated live on `region.on('update')` and `region.on('update-end')`. The render pipeline reads these values to determine the `offset` and `duration` arguments passed to `bufferSource.start(when, offset, duration)`.

### 7.3 Waveform Preview Playback

Clicking inside a region plays that segment via WaveSurfer's own `region.play()`. An `audioprocess` listener manually pauses when `currentTime >= region.end` (failsafe for WaveSurfer 7 region end behaviour).

---

## 8. Local Persistence (IndexedDB)

### 8.1 Database Schema

**Database:** `DJDropStudioDB` (version 1)

| Object Store | Key Mode | Contents |
|---|---|---|
| `tracks` | Manual key | Audio Files keyed by `"intro"`, `"outro"`, `"bed"` |
| `voices` | AutoIncrement | Objects `{ name: string, blob: File }` |

### 8.2 Operations

| Function | Description |
|---|---|
| `initIDB()` | Opens/creates the database; on success calls `loadSavedAssets()` |
| `saveTrackToDB(key, file)` | `put(file, key)` into `tracks` store |
| `deleteTrackFromDB(key)` | `delete(key)` from `tracks` store |
| `saveVoiceToDB(file)` | `add({ name, blob })` into `voices` store |
| `loadSavedAssets()` | On startup: fetches `intro`, `outro`, `bed` from `tracks`; fetches all from `voices` |

### 8.3 Behaviour on Load

1. All three track slots restored from `tracks` store → audio decoded, waveform initialised
2. All voice files restored from `voices` store → decoded, added to `voiceLibrary[]`, first voice auto-selected

---

## 9. Cloud Library (Firebase)

### 9.1 Firebase Project

| Property | Value |
|---|---|
| Project ID | `dji-drop-studio` |
| Auth Domain | `dji-drop-studio.firebaseapp.com` |
| Storage Bucket | `dji-drop-studio.firebasestorage.app` |

### 9.2 Authentication

Anonymous sign-in via `signInAnonymously(auth)`. On success:
- `cloudEnabled = true`
- Cloud tab becomes visible
- `loadCloudLibrary()` is called

If Firebase init or auth fails, the app silently falls back to local-only mode.

### 9.3 Firestore Data Structure

```
/artifacts/{appId}/public/data/sfx_library/{docId}
  name:        string   (original filename)
  type:        string   ("sfx")
  storagePath: string   (e.g. "sfx_library/sfx/1712345678_drop.wav")
  uploadedAt:  number   (Unix timestamp ms)
```

- `appId` = `"dji-drop-studio-main"` (shared namespace — all users see same library)
- Real-time sync via `onSnapshot`

### 9.4 Storage Path Convention

```
sfx_library/{type}/{timestamp}_{originalFilename}
```

### 9.5 Cloud Operations

| Operation | Function | Notes |
|---|---|---|
| Load library | `loadCloudLibrary()` | Live snapshot, natural-sorted by name |
| Preview | `previewCloudItem(storagePath)` | `getBlob()` → decode → play directly; stops previous preview |
| Load to slot | `loadFromCloud(storagePath, name, slot)` | `getBlob()` → `File` → saves to IndexedDB + initialises waveform |
| Upload (batch) | `uploadToCloud(file, type)` | Max 5MB per file; uploads to Storage, writes metadata to Firestore |
| Delete | `deleteCloudItem(docId, storagePath, name)` | PIN-protected (`1234`); deletes from Storage then Firestore |

### 9.6 Legacy Entry Handling

Entries without a `storagePath` field are flagged as "old format". Their Preview and Load buttons are disabled, but Delete still works.

---

## 10. Settings Persistence (localStorage)

**Key:** `dj-drop-settings`  
**Format:** JSON object mapping input element `id` → value (string for ranges, boolean for checkboxes)

- **Saved on:** every `input` event on range sliders, every `change` event on checkboxes
- **Loaded on:** page load (before `initIDB`)

Covers all sliders and checkboxes in the UI (intro/outro/bed volume, offsets, pitch, voice FX toggles, width amount, tail settings, mic boost, auto-duck, bed active).

---

## 11. Render & Export Pipeline

### 11.1 Timeline Calculation

All times are computed relative to `t0 = 0` (start of the OfflineAudioContext).

```
timeIntro      = 0
timeVoice      = max(0, voiceOffset)
buildUpStart   = timeVoice
absVoiceStart  = timeVoice + buildUpDur
voiceEnd       = absVoiceStart + (voiceDuration / pitchRate)
realOutroStart = max(0, voiceEnd + outroOffset)

naturalEnd     = max(endIntro, voiceEnd + tailBuffer, realOutroStart + outroDur)
fadeStart      = naturalEnd + endTrim
totalDuration  = fadeStart + 2.0   ← 2s fade-out tail
```

- `pitchRate` = `Math.pow(2, semitones / 12)`
- `buildUpDur` = `4 × 0.15 = 0.6s` if build-up is on, else `0`
- `tailBuffer` = `2.0s` if tail FX on, else `0.5s`

### 11.2 OfflineAudioContext Render

- Channel count: **2 (stereo)**
- Sample rate: inherited from `ctx.sampleRate`
- Buffer length: `totalDuration × sampleRate`

`offlineCtx.startRendering()` is awaited; the resulting `AudioBuffer` is stored in `renderedBuffer`.

### 11.3 Playback Preview

The rendered buffer is played back via `ctx.createBufferSource()` immediately after render. Previous playback is stopped before a new render begins.

### 11.4 WAV Export

`audioBufferToWav(renderedBuffer)` — custom encoder:
- Format: **PCM** (format = 1)
- Bit depth: **16-bit**
- Channels: **2**
- Sample rate: inherited from buffer
- Output: `Blob` with `type: 'audio/wav'`

The download flow prefers the **File System Access API** (`showSaveFilePicker`) with a fallback to a programmatic `<a>` click. Suggested filename: `dj-drop-{timestamp}.wav`.

---

## 12. Design System & CSS Tokens

### 12.1 Color Tokens

| Token | Value | Used For |
|---|---|---|
| `--bg` | `#0a0a0a` | Page background |
| `--panel` | `rgba(22,22,22,0.9)` | Sidebar, transport |
| `--surface` | `#1a1a1a` | — |
| `--card` | `rgba(26,26,26,0.8)` | Track cards |
| `--border` | `rgba(255,255,255,0.08)` | Default borders |
| `--border-hover` | `rgba(255,255,255,0.15)` | Hover borders |
| `--intro` | `#22c55e` | Intro track accent (green) |
| `--voice` | `#d946ef` | Voice track accent (fuchsia) |
| `--outro` | `#a855f7` | Outro track accent (purple) |
| `--master` | `#06b6d4` | Music bed + primary accent (cyan) |
| `--action` | `#f97316` | Generate button (orange) |
| `--text` | `#f3f4f6` | Primary text |
| `--text-dim` | `#888888` | Secondary text |
| `--text-muted` | `#555555` | Tertiary/placeholder text |
| `--radius` | `12px` | Cards, sidebar, transport |
| `--radius-sm` | `8px` | FX cards, buttons |

### 12.2 Typography

- **Font**: Inter (Google Fonts), with `-apple-system, BlinkMacSystemFont` fallback
- Track titles: `0.8rem`, uppercase, `letter-spacing: 0.08em`
- FX card labels: `0.7rem`, uppercase
- Control values: `0.7rem`, cyan (`--master`)

### 12.3 Micro-Animations

| Name | Trigger | Effect |
|---|---|---|
| Tactile press | `button:active` | `scale(0.96) translateY(1px)`, `brightness(0.9)` |
| Pulse glow | `.btn-gen:not(:disabled)` | Orange box-shadow ring pulse, 2s infinite |
| Slider thumb hover | `::-webkit-slider-thumb:hover` | `scale(1.15)`, intensified glow |
| Slider thumb drag | `::-webkit-slider-thumb:active` | `scale(1.3)`, white glow + grabbing cursor |
| List item entry | `.voice-item`, `.cloud-item` | `slideIn` — 200ms opacity + 5px translateX |

### 12.4 Responsive Breakpoints

| Breakpoint | Change |
|---|---|
| `≤ 1200px` | FX section: 4-col → 2-col grid |
| `≤ 900px` | Sidebar: sticky → relative; sidebar max-height 300px; Intro/Outro: side-by-side → stacked |

---

## 13. State Variables Reference

| Variable | Type | Description |
|---|---|---|
| `ctx` | `AudioContext` | Main audio context |
| `voiceLibrary` | `Array<{name, buffer}>` | In-memory voice library |
| `selectedVoiceIndex` | `number` | Index into `voiceLibrary`, -1 = none |
| `introBuffer` | `AudioBuffer \| null` | Decoded intro SFX |
| `outroBuffer` | `AudioBuffer \| null` | Decoded outro SFX |
| `musicBuffer` | `AudioBuffer \| null` | Decoded music bed |
| `reverbBuffer` | `AudioBuffer` | Pre-generated reverb IR |
| `renderedBuffer` | `AudioBuffer \| null` | Last render output |
| `playbackSource` | `AudioBufferSourceNode \| null` | Current playback node |
| `currentIntroFile` | `File \| null` | Intro File object (for cloud upload) |
| `currentOutroFile` | `File \| null` | Outro File object (for cloud upload) |
| `introTrim` / `outroTrim` / `bedTrim` | `{start, end}` | Active trim region in seconds |
| `wsIntro` / `wsOutro` / `wsBed` | `WaveSurfer \| null` | WaveSurfer instances |
| `wsIntroRegion` etc. | `Region \| null` | Active WaveSurfer region handles |
| `idb` | `IDBDatabase` | IndexedDB handle |
| `cloudEnabled` | `boolean` | True when Firebase auth succeeded |
| `db` | `Firestore \| null` | Firestore reference |
| `storage` | `FirebaseStorage \| null` | Storage reference |
| `previewSource` | `AudioBufferSourceNode \| null` | Current cloud preview node |

---

## 14. Key Functions Reference

| Function | Location | Description |
|---|---|---|
| `initAudio()` | ~L1192 | Ensures AudioContext is running |
| `saveSettings()` | ~L1202 | Serialises all slider/checkbox values to localStorage |
| `loadSettings()` | ~L1215 | Restores slider/checkbox values from localStorage |
| `switchTab(tab)` | ~L1256 | Toggles sidebar tab visibility |
| `initIDB()` | ~L1531 | Opens IndexedDB, triggers `loadSavedAssets()` |
| `saveTrackToDB(key, file)` | ~L1545 | Persists a track file to IDB |
| `deleteTrackFromDB(key)` | ~L1551 | Removes a track from IDB |
| `saveVoiceToDB(file)` | ~L1557 | Adds a voice file to IDB voices store |
| `loadSavedAssets()` | ~L1563 | Restores all saved tracks and voices from IDB on startup |
| `loadAudioFile(file)` | ~L1620 | Decodes a File/Blob to AudioBuffer |
| `findAudioStart(buffer)` | ~L1626 | Finds first sample above 0.015 amplitude |
| `createReverbImpulse(dur)` | ~L1635 | Generates a stereo exponential-decay noise IR |
| `audioBufferToWav(buffer)` | ~L1649 | Encodes AudioBuffer → 16-bit PCM WAV Blob |
| `addVoiceToUI(name, id, dur)` | ~L1706 | Appends a voice item to the sidebar list |
| `selectVoice(idx)` | ~L1716 | Activates a voice by index |
| `initWaveform(containerId, file, type)` | ~L1737 | Creates/replaces a WaveSurfer instance with regions |
| `loadCloudLibrary()` | ~L1328 | Subscribes to Firestore snapshot, renders cloud list |
| `uploadToCloud(file, type)` | ~L1390 | Uploads file to Firebase Storage + adds Firestore doc |
| `loadFromCloud(path, name, slot)` | ~L1416 | Downloads cloud file → loads into intro/outro slot |
| `previewCloudItem(path)` | ~L1450 | Plays a cloud file without loading it to a slot |
| `deleteCloudItem(docId, path, name)` | ~L1477 | PIN-protected delete from Storage + Firestore |
| `generateBtn` click handler | ~L1936 | Master render + playback function |
| `downloadBtn` click handler | ~L2230 | WAV export with File System Access API + fallback |

---

## 15. Known Behaviours & Gotchas

- **Delete PIN** is hardcoded as `"1234"` (line ~1476). This is visible in source code and provides only light protection.
- **Cloud tab** is hidden by default (`display:none`) and shown only after successful Firebase anonymous auth. If Firebase fails, the app works fully in local-only mode.
- **Legacy cloud entries** (uploaded before `storagePath` was added) are shown with strikethrough text and disabled load/preview buttons; delete still works.
- **Voice offset** allows values from -1s to +10s, meaning voice can be scheduled up to 1 second before or 10 seconds after the intro starts. The timeline clamp ensures `timeVoice >= t0`.
- **`realOutroStart` const assignment bug**: Line ~1987 attempts a `const realOutroStart = ...; if (realOutroStart < t0) realOutroStart = t0;` which would throw in strict mode (reassigning a const). In practice the `if` branch is unreachable because `outroOffset` minimum is `-5s` and `voiceEnd` is always ≥ 0, so the bug is dormant.
- **WaveSurfer 7 region end stop**: WaveSurfer 7 does not automatically stop at region end, so an `audioprocess` event listener is attached as a manual failsafe.
- **IndexedDB voices are append-only**: There is no UI to delete individual voices from the local library (only cloud items can be deleted). Voices accumulate across sessions.
- **Waveform on Cloud Load**: When a cloud audio file is loaded into intro/outro, IndexedDB is updated (`saveTrackToDB`) so the file persists across page reloads.
- **Audio context creation at top level**: `const ctx = new AudioContext()` is called at module evaluation time, which may cause browsers to create a suspended context. `initAudio()` is called on user gesture to resume it.

---

## 16. File Structure

```
dj-drop-studio/
├── index.html                        ← Entire application (HTML + CSS + JS)
├── index_backup_post_trimming.html   ← Old version backup
├── index_backup_pre_trimming.html    ← Old version backup
├── index_backup_sequencer.html       ← Sequencer variant backup
├── index_check.html                  ← Dev/testing variant
├── index_check_b.html                ← Dev/testing variant B
├── cors.json                         ← Firebase Storage CORS configuration
├── README.md                         ← Brief project description
└── SPEC.md                           ← This document
```

### cors.json

Grants cross-origin access to the Firebase Storage bucket for browser-based `getBlob()` calls:

```json
[
  {
    "origin": ["*"],
    "method": ["GET"],
    "maxAgeSeconds": 3600
  }
]
```

---

*Last updated: 2026-04-09*
