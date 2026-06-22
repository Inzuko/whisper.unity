# 8 — Setting Up a Game Scene

[01 — Getting Started](01-Getting-Started.md) covers installing the package and writing your first script. This guide is the hands-on companion: **how to build the actual scene** — which GameObjects to create, which components to add, and how to wire them together in the Inspector.

We go from the bare minimum to a complete microphone UI that mirrors the shipped sample scenes.

> **Fastest path:** if you just want a working scene to copy, open `Assets/Samples/2 - Microphone/2 - Microphone.unity`, then duplicate/strip it. The steps below explain how that scene is wired so you can build (or trim) your own. See [Reusing the sample scenes](#reusing-the-sample-scenes).

---

## A. Minimal scene — transcribe a clip (no UI)

The smallest possible setup: one manager, one driver script. Good for prototyping or non-interactive transcription.

### GameObject hierarchy
```
Scene
├── Whisper        → WhisperManager
└── Transcriber    → your script (e.g. TranscribeClip)
```

### Steps
1. **Create the manager.** GameObject → Create Empty, name it `Whisper`. Add Component → **Whisper Manager**.
2. **Configure the manager** in the Inspector:
   - **Model Path:** `Whisper/ggml-tiny.bin` (must exist under `Assets/StreamingAssets/Whisper/` — see [01 — Getting Started](01-Getting-Started.md)).
   - **Is Model Path In Streaming Assets:** ✔ on.
   - **Init On Awake:** ✔ on (loads the model automatically).
   - **Language:** `en` (or `auto`).
3. **Create the driver.** GameObject → Create Empty, name it `Transcriber`. Add a script with a `public WhisperManager whisper;` field (e.g. `TranscribeClip` from [04 — Integration Guide](04-Integration-Guide.md), Recipe 1).
4. **Wire it.** Drag the `Whisper` GameObject onto the script's **Whisper** field. Assign an **AudioClip**.
5. Press **Play**.

That's a complete, working transcription scene — no Canvas, no microphone.

---

## B. Full scene — microphone + on-screen transcript

This is the structure of `2 - Microphone`. It adds a microphone, a record button, and a text area. There are **three logic GameObjects** plus a standard uGUI Canvas.

### Logic GameObjects

| GameObject | Component | Role |
|------------|-----------|------|
| `Whisper` | **WhisperManager** | Loads the model, runs inference. |
| `VadDetector` | **MicrophoneRecord** | Captures the mic, chunks audio, does VAD/auto-stop. |
| `Demo` | **MicrophoneDemo** (your controller) | Wires the button, calls transcription, updates UI. |

> In the sample, the mic GameObject is named `VadDetector` because it also owns the Voice-Activity-Detection settings. You can name it anything.

### UI (under a Canvas)

A standard **Canvas** + **EventSystem** (Unity creates the EventSystem automatically when you add the first UI element). The sample's relevant UI objects:

| UI object | Type | Bound to (`MicrophoneDemo` field) |
|-----------|------|-----------------------------------|
| `RecordButton` | Button | `button` (its child `Text` → `buttonText`) |
| `Transcript` | Text | `outputText` |
| `Time` | Text | `timeText` (shows ms / progress) |
| `Scroll View` | ScrollRect | `scroll` (auto-scrolls long transcripts) |
| `LanguageDropdown` | Dropdown | `languageDropdown` |
| `EnglishTranslation` | Toggle | `translateToggle` |
| `VADStop` | Toggle | `vadToggle` |
| `MicrophoneDropdown` | Dropdown | (assigned to `MicrophoneRecord.microphoneDropdown`) |

### Build it step by step

1. **Whisper manager** — as in section A: GameObject `Whisper` + **Whisper Manager**, set Model Path + language.

2. **Microphone** — GameObject → Create Empty, name `VadDetector`, add **Microphone Record**. Key fields:
   - **Frequency:** `16000` (Whisper's native rate — avoids extra resampling).
   - **Max Length Sec:** how long a single recording can run.
   - **Use Vad / Vad Stop:** enable `Vad Stop` if you want recording to end automatically after silence (`Vad Stop Time` seconds).
   - **Echo:** plays back the recording when it stops (handy while testing).
   - (Optional) **Microphone Dropdown** / **Vad Indicator Image** — assign UI later.

3. **Canvas + UI** — GameObject → UI → Canvas (this also creates an EventSystem). Under it add:
   - a **Button** (`RecordButton`) with a child **Text** for its label,
   - a **Text** for the transcript (`Transcript`) — enable **Rich Text** if you'll show colored subtitles,
   - a **Text** for timing (`Time`),
   - optionally a **Scroll View** wrapping the transcript, **Dropdown**s, and **Toggle**s as in the table.

4. **Controller** — GameObject → Create Empty, name `Demo`, add your controller script (the sample's `MicrophoneDemo`, or your own from [04 — Integration Guide](04-Integration-Guide.md), Recipe 2). Its job: subscribe to events and update UI.

5. **Wire references** — select `Demo` and drag objects onto its fields:
   - **Whisper** ← the `Whisper` GameObject,
   - **Microphone Record** ← the `VadDetector` GameObject,
   - **Button / Button Text / Output Text / Time Text / Scroll / dropdowns / toggles** ← the matching UI objects.

6. Press **Play**, click **Record**, speak, click **Stop** (or let VAD auto-stop). The transcript appears in `Transcript`.

### How the wiring flows at runtime
```
RecordButton.onClick → MicrophoneRecord.StartRecord()/StopRecord()
MicrophoneRecord.OnRecordStop(AudioChunk) → controller
        → await WhisperManager.GetTextAsync(chunk.Data, chunk.Frequency, chunk.Channels)
WhisperManager.OnNewSegment / OnProgress (main thread) → update Text UI
```

---

## C. Streaming (live captions) scene

Same three logic GameObjects as section B, but the controller is a **streaming** controller (the sample `StreamingSampleMic`) and there's no separate "transcribe after stop" step.

Differences from section B:
- The controller creates a `WhisperStream` in `Start()`:
  ```csharp
  _stream = await whisper.CreateStream(microphoneRecord);
  ```
- On the record button it calls `_stream.StartStream()` **before** `microphoneRecord.StartRecord()`.
- It updates the transcript from `_stream.OnResultUpdated` instead of a single `GetTextAsync` call.
- On the `WhisperManager`, tune the **Streaming settings** (`stepSec`, `lengthSec`, `keepSec`, `singleSegment`, `useVad`) — see [04 — Integration Guide](04-Integration-Guide.md), Recipe 3.

Wire the controller's **Whisper**, **Microphone Record**, **Button**, **Button Text**, **Text**, and **Scroll** fields exactly as before. This mirrors `Assets/Samples/5 - Streaming/5 - Streaming.unity`.

---

## Reusing the sample scenes

Often the quickest start is to copy a sample and strip what you don't need:

1. Duplicate a sample scene file (e.g. `Assets/Samples/2 - Microphone/2 - Microphone.unity`) into your own folder, or copy the GameObjects via copy/paste between scenes.
2. Keep the `Whisper`, mic, and controller GameObjects; delete UI you don't want.
3. Replace the sample controller (`MicrophoneDemo` etc.) with your own script, or keep it as a reference.
4. Re-check the `WhisperManager` **Model Path** points at a model in **your** project's `StreamingAssets`.

> If you installed the package via the Package Manager (not by cloning the whole repo), the sample **scenes** under `Assets/Samples/` aren't imported into your project automatically — copy them from this repository, or build your scene from scratch using the steps above.

---

## Checklist before pressing Play

- [ ] `Whisper` GameObject has a **WhisperManager** with a valid **Model Path**.
- [ ] The model file actually exists under `Assets/StreamingAssets/...`.
- [ ] Mic scenes: a **MicrophoneRecord** GameObject exists and is assigned to the controller.
- [ ] Controller's `public` fields (Whisper, mic, UI) are all wired (no "None" / missing references).
- [ ] Streaming scenes: `StartStream()` is called before `StartRecord()`.
- [ ] Mobile: microphone permission is set up (see [06 — Platforms & Deployment](06-Platforms-and-Deployment.md)).

Next: [09 — Worked Example: Mic Streaming Game](09-Example-Microphone-Streaming-Game.md) for a complete, real microphone-streaming integration, [03 — API Reference](03-API-Reference.md) for every field these components expose, or [04 — Integration Guide](04-Integration-Guide.md) for the controller scripts.
