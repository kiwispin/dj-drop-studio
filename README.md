# 🎙️ DJ Drop Studio

> A professional-grade, browser-based audio mixing tool for creating radio DJ drops — entirely in a single HTML file with no build step.

[![No Dependencies](https://img.shields.io/badge/build-none-brightgreen)](#) [![Browser](https://img.shields.io/badge/platform-browser-blue)](#) [![License](https://img.shields.io/badge/license-MIT-lightgrey)](#)

---

## Table of Contents

1. [Overview](#overview)
2. [Core Features](#core-features)
3. [Tech Stack](#tech-stack)
4. [Architecture & Data Flow](#architecture--data-flow)
5. [Track System](#track-system)
6. [Audio Engine & Signal Graph](#audio-engine--signal-graph)
7. [FX Processing Chain](#fx-processing-chain)
8. [Render & Export Pipeline](#render--export-pipeline)
9. [Local Persistence (IndexedDB)](#local-persistence-indexeddb)
10. [Cloud Library (Firebase)](#cloud-library-firebase)
11. [Settings Persistence](#settings-persistence)
12. [Installation & Setup](#installation--setup)
13. [Firebase Configuration](#firebase-configuration)
14. [File Structure](#file-structure)
15. [Future Roadmap](#future-roadmap)

---

## Overview

**DJ Drop Studio** is a zero-dependency, single-file web application that enables radio presenters, producers, and podcasters to compose professional-quality **DJ drops** — short audio idents that layer multiple audio elements together.

A typical drop combines:

| Layer | Description |
|---|---|
| **Intro SFX** | A stinger or jingle that plays before the voice |
| **Voice Recording** | The presenter's spoken "drop", drawn from a local library |
| **Outro SFX** | A hit or sting that fires after the voice ends |
| **Music Bed** | Background music that auto-ducks under the voice |

All mixing is computed entirely in the browser via the **Web Audio API's `OfflineAudioContext`** and exported as a broadcast-ready **16-bit stereo WAV file**. No server, no plugins, no installation.

**Target audience:** Radio producers, broadcasting students, podcast creators, and anyone needing quick professional voice-over idents.

---

## Core Features

### 🎚️ Mixing

- **4-track mixing engine** — Intro SFX, Voice, Outro SFX, and Music Bed tracks
- **Per-track volume sliders** with real-time value display
- **Auto-ducking** — automatic volume reduction of Intro, Outro, and Music Bed during voice playback (70% reduction, configurable per-track)
- **Waveform visualisation** on Intro, Outro, and Music Bed tracks (WaveSurfer.js)
- **Drag-to-trim** region handles on each waveform — crop any audio without destructive editing
- **Voice offset** — schedule voice -1s to +10s relative to the intro
- **Outro offset** — position the outro hit relative to voice end (-5s to +5s)

### 🎙️ Voice Library

- **Drop zone upload** — drag-and-drop or click to add voice recordings
- **Multi-voice library** — store an unlimited number of voice clips, persisted across sessions (IndexedDB)
- **One-click voice selection** — switch active voice without reloading
- **Waveform preview** — click within a region to audition a clip segment

### 🚀 FX Rack (Voice Core)

| Effect | Description |
|---|---|
| **Build-Up** | 4× stutter pre-roll with pitch sweep, HPF ramp, and flanger; fires 0.6s before the voice drop |
| **Pro Width** | Haas-effect stereo widener + Radio EQ (3kHz presence boost, 10kHz air shelf) |
| **Tail FX** | Configurable ping-pong delay (300ms/450ms) and programmatic reverb tail at voice end |
| **Mic Boost** | +8dB pre-gain with hard-knee compression for quiet recordings |

### 🌩️ Cloud SFX Library

- **Firebase-backed shared library** — browse and load community-uploaded SFX into any track
- **Live real-time sync** via Firestore `onSnapshot`
- **Batch upload** — upload multiple SFX files (max 5MB each) to the shared cloud
- **Pin-protected delete** for cloud entries
- **In-browser preview** of cloud files before loading
- **Graceful fallback** to local-only mode if Firebase is unreachable

### 💾 Export

- **Offline render** via `OfflineAudioContext` — full-mix computation at native sample rate
- **16-bit PCM WAV export** via custom encoder
- **File System Access API** save dialog (with `<a>` tag fallback for unsupported browsers)
- **Master compressor** on final mix (DynamicsCompressor, 4:1 ratio)
- **2-second fade-out tail** appended automatically

---

## Tech Stack

| Dependency | Version | Loaded From | Purpose |
|---|---|---|---|
| **WaveSurfer.js** | 7.x | `unpkg.com` | Waveform rendering |
| **WaveSurfer Regions Plugin** | 7.x | `unpkg.com` | Drag-to-trim selection regions |
| **Firebase App** | 11.6.1 | `gstatic.com` (ESM) | Firebase initialisation |
| **Firebase Auth** | 11.6.1 | `gstatic.com` (ESM) | Anonymous sign-in for cloud access |
| **Firebase Firestore** | 11.6.1 | `gstatic.com` (ESM) | Cloud SFX library metadata |
| **Firebase Storage** | 11.6.1 | `gstatic.com` (ESM) | Cloud SFX audio file hosting |
| **Google Fonts – Inter** | Latest | `fonts.googleapis.com` | UI typography |
| **Web Audio API** | Native | Browser | All audio processing & mixing |
| **IndexedDB** | Native | Browser | Local file persistence |
| **localStorage** | Native | Browser | Settings persistence |

**No build toolchain required.** The entire application is self-contained in a single `index.html` file.

---

## Architecture & Data Flow

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser (index.html)                     │
│                                                                  │
│  ┌────────────┐    ┌──────────────────────────────────────────┐  │
│  │  Sidebar   │    │              Main Mixer                   │  │
│  │            │    │  ┌──────────┐  ┌──────────────────────┐  │  │
│  │ My Crate   │    │  │ Track 1  │  │      Track 2          │  │  │
│  │ (IndexedDB)│    │  │ Intro SFX│  │   Voice Core + FX    │  │  │
│  │            │    │  └──────────┘  └──────────────────────┘  │  │
│  │ Cloud SFX  │    │  ┌──────────┐  ┌──────────────────────┐  │  │
│  │ (Firebase) │    │  │ Track 3  │  │       Track 4         │  │  │
│  │            │    │  │ Outro SFX│  │     Music Bed         │  │  │
│  └────────────┘    │  └──────────┘  └──────────────────────┘  │  │
│                    │         ▼ Render & Play ▼                  │  │
│                    │  ┌──────────────────────────────────────┐  │  │
│                    │  │    Transport Bar (Render + Save)      │  │  │
│                    │  └──────────────────────────────────────┘  │  │
│                    └──────────────────────────────────────────┘  │
│                                                                  │
│  Persistence Layer                                               │
│  ┌──────────────────────┐   ┌─────────────────────────────────┐ │
│  │  IndexedDB           │   │  localStorage                   │ │
│  │  (audio files)       │   │  (slider & checkbox settings)   │ │
│  └──────────────────────┘   └─────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
                │ Firebase SDK (ESM)
                ▼
┌──────────────────────────────────┐
│  Firebase Project: dji-drop-studio │
│  ├── Firestore (SFX metadata)    │
│  └── Storage (SFX audio files)   │
└──────────────────────────────────┘
```

### UI Layout

```
body
└── .studio-container  (CSS Grid: 260px sidebar | 1fr main)
    ├── aside.sidebar  (sticky, full-height; two tabs)
    │   ├── #tabLocal  — My Crate (voice drop zone + voice list)
    │   └── #tabCloud  — Cloud SFX (Firebase, hidden until auth)
    └── main.mixer  (scrollable column)
        ├── .top-row  (CSS Grid: 1fr 1fr)
        │   ├── Track 1 — Intro SFX
        │   └── Track 3 — Outro SFX
        ├── Track 2 — Voice Core + FX Rack
        ├── Track 4 — Music Bed
        └── .action-bar → .transport (sticky footer)
```

### Data Flow: Render

```
User clicks "Render & Play"
        │
        ▼
initAudio() → resumes AudioContext
        │
        ▼
Gather all AudioBuffers
(introBuffer, selectedVoice.buffer, outroBuffer, musicBuffer)
        │
        ▼
Compute timeline (voice offset, build-up, outro offset, fade)
        │
        ▼
Create OfflineAudioContext (stereo, native sample rate)
        │
        ▼
Wire signal graph:
  [sources] → [FX nodes] → masterComp → masterGain → destination
        │
        ▼
offlineCtx.startRendering()
        │
        ▼
renderedBuffer → playback via AudioBufferSourceNode
        │
        ▼
"⬇ Save" button enabled → audioBufferToWav() → File download
```

---

## Track System

### Track 1 — Intro SFX

| Control | ID | Range | Default |
|---|---|---|---|
| File input | `#introFile` | `audio/*` | — |
| Clear button | `#clearIntroBtn` | — | — |
| Status | `#introStatus` | — | — |
| Waveform | `#introWave` | — | — |
| Volume | `#introVol` | 0 – 1.5 | 0.8 |
| Auto-Duck | `#introDuck` | checkbox | off |

### Track 2 — Voice Core

Voice is selected from the sidebar voice library (`#voiceList`). The FX Rack provides four processing cards:

| Card | Toggle | Controls |
|---|---|---|
| 🚀 Build-Up | `#buildUpOn` | Fixed: 4 stutters × 150ms, pitch sweep –300→0 cents, HPF + Flanger |
| 💎 Pro Width | `#widthOn` | `#widthAmt` (0–1); Radio EQ presence + air |
| 🌌 Tail FX | `#tailOn` | `#tailStart` (0.1–0.9), `#tailMix` (0–1) |
| 🎚️ Voice Level | — | `#voiceOffset` (–1–+10s), `#voiceVol` (0–5), `#voicePitch` (–12–+12 semi), `#endTrim` (–10–+5s), `#micBoost` |

### Track 3 — Outro SFX

| Control | ID | Range | Default |
|---|---|---|---|
| File input | `#outroFile` | `audio/*` | — |
| Clear button | `#clearOutroBtn` | — | — |
| Status | `#outroStatus` | — | — |
| Waveform | `#outroWave` | — | — |
| Offset | `#outroOffset` | –5 – +5s | 0 |
| Volume | `#outroVol` | 0 – 1.5 | 0.8 |
| Auto-Duck | `#outroDuck` | checkbox | off |

### Track 4 — Music Bed

| Control | ID | Range | Default |
|---|---|---|---|
| File input | `#bedFile` | `audio/*` | — |
| Active toggle | `#bedOn` | checkbox | on |
| Clear button | `#clearBedBtn` | — | — |
| Status | `#bedStatus` | — | — |
| Waveform | `#bedWave` | — | — |
| Volume | `#bedVol` | 0 – 1 | 0.5 |
| Auto-Duck | `#autoDuck` | checkbox | on |

---

## Audio Engine & Signal Graph

### AudioContext Lifecycle

```js
const ctx = new (window.AudioContext || window.webkitAudioContext)();
// Resumed on first user gesture via initAudio()
```

### Key Audio Functions

| Function | Description |
|---|---|
| `loadAudioFile(file\|blob)` | Reads `ArrayBuffer`, decodes via `ctx.decodeAudioData()`. Returns `AudioBuffer \| null` |
| `findAudioStart(buffer)` | Scans channel 0 for first sample > 0.015 amplitude; skips leading silence |
| `createReverbImpulse(duration)` | Generates a 2s stereo exponential-decay noise IR for the Tail FX convolver |
| `audioBufferToWav(buffer)` | Custom 16-bit PCM WAV encoder; returns `Blob` |

### Offline Signal Graph

```
[Intro Src]          ──────────────────────► introGain ─┐
[Build-Up Stutters]  → HPF → Delay → Gain → (boost?) ──┤
[Main Voice Src]     → (EQ?) → (boostGain → comp?) → vg─┤
[Width L/R Sources]  → delays → merger → wg ────────────┤ → masterComp → masterGain → destination
[Tail Ping-Pong]     → delays ↔ feedback → merger ──────┤
[Tail Reverb]        → convolver ────────────────────────┤
[Outro Src]          ──────────────────────► outroGain ─┤
[Bed Src]            ──────────────────────► bedGain ───┘
```

**Master Compressor settings:**

| Parameter | Value |
|---|---|
| Threshold | –12 dB |
| Knee | 30 dB |
| Ratio | 4:1 |
| Attack | 3 ms |
| Release | 250 ms |

### Timeline Calculation

```
timeIntro      = 0
timeVoice      = max(0, voiceOffset)
buildUpStart   = timeVoice
absVoiceStart  = timeVoice + buildUpDur
voiceEnd       = absVoiceStart + (voiceDuration / pitchRate)
realOutroStart = max(0, voiceEnd + outroOffset)
naturalEnd     = max(endIntro, voiceEnd + tailBuffer, realOutroStart + outroDur)
fadeStart      = naturalEnd + endTrim
totalDuration  = fadeStart + 2.0s  ← master fade-out tail
```

- `pitchRate = 2^(semitones/12)`
- `buildUpDur = 0.6s` (4 × 150ms) if Build-Up on, else `0`
- `tailBuffer = 2.0s` if Tail FX on, else `0.5s`

---

## FX Processing Chain

### 🚀 Build-Up

Fires before the voice drop as a pre-roll stutter effect:

- **4 stutters**, each **150ms**, starting from the voice cue point
- Pitch sweeps from **–300 cents → 0 cents** (relative to `voicePitch`)
- Signal path: `BufferSource → HPF (200Hz→1kHz ramp) → Flanger (5ms, 3Hz LFO, 2-cent depth) → GainNode → masterComp`

### 💎 Pro Width

Applied when `#widthOn` is checked:

- **Radio EQ**: Peaking +4dB @ 3kHz (presence), High-shelf +5dB @ 10kHz (air)
- **Haas width**: Two extra mono voice sources (L: –25 cents / R: +25 cents), delayed (L: 18ms / R: 28ms), merged stereo, blended at `widthAmt × 0.85`

### 🔥 Mic Boost

Applied when `#micBoost` is checked:

- Pre-gain: **+2.5× (~+8dB)**
- DynamicsCompressor: –24dB threshold, 10 knee, **12:1 ratio**, 2ms attack, 100ms release
- Also applied to Build-Up path: 6.0× gain, –30dB threshold, 16:1 ratio

### 🌌 Tail FX

Applied when `#tailOn` is checked:

- Tap from voice gain output, fades in from `tailStart%` → voice end at `tailMix` level
- **Ping-Pong Delay**: L 300ms / R 450ms with **0.4 cross-feedback**
- **Convolver Reverb**: Uses the programmatic 2s exponential-decay IR

### 🦆 Auto-Ducking

| Track | Duck Depth | Duck-In Ramp | Duck-Out Ramp |
|---|---|---|---|
| Music Bed | 70% (→ vol × 0.3) | 0.3s linear | 1.0s linear |
| Intro SFX | 70% (if overlapping voice) | 0.3s linear | 1.0s linear |
| Outro SFX | 70% (if overlapping voice) | 0.3s linear | 1.0s linear |

---

## Render & Export Pipeline

### Render

1. User presses **▶ Render & Play**
2. `initAudio()` resumes the `AudioContext`
3. Timeline is computed from all slider values
4. A fresh `OfflineAudioContext` is created and the signal graph is wired
5. `offlineCtx.startRendering()` is awaited → produces `renderedBuffer`
6. `renderedBuffer` is played back immediately via `ctx.createBufferSource()`

### Export (Save as WAV)

1. User presses **⬇ Save** (enabled after a render)
2. `audioBufferToWav(renderedBuffer)` encodes a **16-bit PCM, stereo WAV** `Blob`
3. **File System Access API** (`showSaveFilePicker`) is attempted first
4. Falls back to `<a download>` click if API is unavailable
5. Suggested filename: `dj-drop-{timestamp}.wav`

---

## Local Persistence (IndexedDB)

**Database:** `DJDropStudioDB` | **Version:** 1

### Object Stores

| Store | Key Mode | Contents |
|---|---|---|
| `tracks` | Manual key | Audio `File` objects keyed by `"intro"`, `"outro"`, `"bed"` |
| `voices` | AutoIncrement | Objects `{ name: string, blob: File }` |

### Operations

| Function | Description |
|---|---|
| `initIDB()` | Opens/creates the database; on success calls `loadSavedAssets()` |
| `saveTrackToDB(key, file)` | Persists an audio file to the `tracks` store |
| `deleteTrackFromDB(key)` | Removes a track from the `tracks` store |
| `saveVoiceToDB(file)` | Appends a voice to the `voices` store |
| `loadSavedAssets()` | On startup: restores tracks and all voices; auto-selects first voice |

> **Note:** Voice files are append-only in local storage — there is no current UI to delete individual voices from IndexedDB once saved.

---

## Cloud Library (Firebase)

### Firebase Project

| Property | Value |
|---|---|
| Project ID | `dji-drop-studio` |
| Auth Domain | `dji-drop-studio.firebaseapp.com` |
| Storage Bucket | `dji-drop-studio.firebasestorage.app` |

### Authentication

Anonymous sign-in via `signInAnonymously(auth)`. On success:
- `cloudEnabled = true`
- Cloud tab becomes visible in the sidebar
- `loadCloudLibrary()` is invoked

If Firebase initialisation or auth fails, the app silently continues in **local-only mode**.

### Firestore Data Structure

```
/artifacts/{appId}/public/data/sfx_library/{docId}
  ├── name:        string   — original filename
  ├── type:        string   — "sfx"
  ├── storagePath: string   — "sfx_library/sfx/{timestamp}_{filename}"
  └── uploadedAt:  number   — Unix timestamp (ms)
```

- `appId` = `"dji-drop-studio-main"` (shared namespace — all users share the same library)
- Updated in real-time via `onSnapshot`

### Storage Path Convention

```
sfx_library/{type}/{timestamp}_{originalFilename}
```

### Cloud Operations

| Operation | Function | Notes |
|---|---|---|
| Load library | `loadCloudLibrary()` | Live `onSnapshot`, natural-sorted by name |
| Preview | `previewCloudItem(storagePath)` | `getBlob()` → decode → play; stops previous preview |
| Load to slot | `loadFromCloud(storagePath, name, slot)` | Downloads → saves to IndexedDB → initialises waveform |
| Upload | `uploadToCloud(file, type)` | Max **5MB** per file; writes to Storage + Firestore |
| Delete | `deleteCloudItem(docId, storagePath, name)` | PIN-protected (`1234`); removes from Storage then Firestore |

### CORS Configuration (`cors.json`)

```json
[
  {
    "origin": ["*"],
    "method": ["GET"],
    "maxAgeSeconds": 3600
  }
]
```

Apply with: `gsutil cors set cors.json gs://dji-drop-studio.firebasestorage.app`

---

## Settings Persistence

**localStorage key:** `dj-drop-settings`  
**Format:** JSON object mapping element `id` → value (`string` for ranges, `boolean` for checkboxes)

- **Saved:** on every `input` event (sliders) and `change` event (checkboxes)
- **Restored:** on page load (before IndexedDB initialisation)

Covers all mixer controls: track volumes, offsets, pitch, voice FX toggles, width amount, tail settings, mic boost, auto-duck, and bed active state.

---

## Installation & Setup

### Prerequisites

- A modern web browser with Web Audio API support (Chrome 89+, Firefox 88+, Edge 89+, Safari 15+)
- A Firebase project (optional — required only for the Cloud SFX Library)
- No Node.js, npm, or build tools required

### Local Usage

```bash
# 1. Clone the repository
git clone https://github.com/your-username/dj-drop-studio.git
cd dj-drop-studio

# 2. Open the app
# Option A: Double-click index.html in your file manager
# Option B: Serve via a simple HTTP server (recommended for Firebase)
npx serve .
# or
python -m http.server 8080
```

Open `http://localhost:8080` (or `http://localhost:3000` with `npx serve`) in your browser.

> **Why a server?** Firebase SDK requires a valid HTTP origin for anonymous authentication. Opening `index.html` directly via `file://` will disable the Cloud SFX Library tab.

### Basic Workflow

```
1. Upload voice recordings → "My Crate" sidebar (drag or click)
2. Load Intro SFX / Outro SFX → click "Choose File" on each track
3. Load a Music Bed → click "Choose File" on Track 4
4. Select a voice from the library → click a voice item in the sidebar
5. Adjust volumes, offsets, and FX settings
6. Click "▶ Render & Play" → hear the full rendered mix
7. Click "⬇ Save" → export as a 16-bit WAV file
```

---

## Firebase Configuration

To enable the Cloud SFX Library with your own Firebase project:

### 1. Create a Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Create a new project
3. Enable **Anonymous Authentication** under Authentication → Sign-in method
4. Create a **Firestore** database (production or test mode)
5. Enable **Firebase Storage**

### 2. Configure Firestore Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /artifacts/{appId}/public/data/{document=**} {
      allow read: if true;
      allow write: if request.auth != null;
    }
  }
}
```

### 3. Configure Storage Rules

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /sfx_library/{allPaths=**} {
      allow read: if true;
      allow write: if request.auth != null;
    }
  }
}
```

### 4. Apply CORS to Storage Bucket

```bash
gsutil cors set cors.json gs://YOUR_STORAGE_BUCKET
```

### 5. Update `index.html`

Replace the `firebaseConfig` object near the bottom of the `<script type="module">` block:

```js
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT.firebasestorage.app",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
```

---

## File Structure

```
dj-drop-studio/
├── index.html                        ← Entire application (HTML + CSS + JS, ~2270 lines)
├── cors.json                         ← Firebase Storage CORS configuration
├── SPEC.md                           ← Detailed technical specification
├── README.md                         ← This file
├── index_backup_post_trimming.html   ← Backup (post waveform-trim feature)
├── index_backup_pre_trimming.html    ← Backup (pre waveform-trim feature)
├── index_backup_sequencer.html       ← Backup (sequencer variant)
├── index_check.html                  ← Development/testing variant
└── index_check_b.html                ← Development/testing variant B
```

---

## Known Behaviours & Gotchas

| Issue | Detail |
|---|---|
| **Delete PIN** | Hardcoded as `"262626"` — visible in source. Provides minimal protection. |
| **`const` reassignment bug** | Line ~1987: `const realOutroStart` is reassigned in a dead `if` branch. Safe in practice as `outroOffset ≥ –5s` and `voiceEnd ≥ 0`. |
| **WaveSurfer 7 region end** | WaveSurfer 7 doesn't auto-stop at region end; an `audioprocess` listener is attached as a manual stop failsafe. |
| **Voices are append-only** | No UI to delete individual voices from IndexedDB. They accumulate across sessions. |
| **AudioContext at load** | `AudioContext` is created at top-level (may be suspended by browser policy); `initAudio()` resumes it on the first user gesture. |
| **Cloud tab hidden** | The Cloud SFX tab stays hidden until Firebase anonymous auth succeeds — no error is shown to the user on failure. |
| **Legacy cloud entries** | Entries without `storagePath` are shown with strikethrough and disabled load/preview buttons; delete still works. |

---

## Future Roadmap

Based on analysis of the codebase, the following enhancements would have the highest impact:

### 1. 🔒 Replace Hardcoded Delete PIN with Firebase Auth Roles
The current PIN (`1234`) is visible in plain source code and offers negligible security. Implementing Firebase Auth roles (e.g., an `admin` custom claim) would allow proper role-based access control for cloud library management without requiring structural changes to the existing Firebase setup.

### 2. 🗑️ Voice Library Management
The current voice library is append-only — users cannot delete, rename, or reorder voice recordings stored in IndexedDB. Adding a context menu or long-press interaction on voice list items to support rename and delete operations would significantly improve the day-to-day workflow for users managing large voice libraries.

### 3. 📦 Preset System (Scene Save/Load)
There is no way to save and recall a complete mix configuration (track files + all slider and FX settings). A preset system backed by IndexedDB or localStorage JSON export would allow users to maintain a library of drop templates (e.g., "Morning Show Ident", "News Bulletin Hit") and recall them instantly.

### 4. ⚖️ Fix the `const` Reassignment Bug
`realOutroStart` is declared with `const` but guarded by a conditional reassignment that is currently unreachable. While dormant, this will throw a `TypeError` in strict mode if the guard condition is ever triggered. Refactor to `let` or eliminate the guard entirely.

### 5. 📱 Responsive / Mobile Layout
The current layout uses a fixed 260px sidebar with sticky positioning that does not adapt well to viewports narrower than ~900px. Implementing a collapsible drawer-style sidebar and touch-optimised slider controls would make the app usable on tablets — valuable for remote presenters and on-location production.

---

*Last updated: April 2026*
