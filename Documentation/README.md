# whisper.unity Documentation

`whisper.unity` is a set of Unity3D bindings for [whisper.cpp](https://github.com/ggerganov/whisper.cpp) — a high-performance C/C++ port of [OpenAI's Whisper](https://github.com/openai/whisper) automatic speech recognition (ASR) model. It lets your Unity game transcribe speech to text **entirely on the user's device, offline, with no server and no per-request cost.**

This documentation explains how the package is built, how to integrate it into a game, and how to support (and "train" for) different languages.

## What you get

- **Speech-to-text** from an `AudioClip`, a raw `float[]` audio buffer, or a live microphone.
- **Real-time streaming** transcription with a sliding window and Voice Activity Detection (VAD).
- **Multilingual** transcription (~60 languages) with automatic language detection.
- **Speech translation** — transcribe speech in one language directly into English text.
- **Token-level output** with per-word confidence and timestamps (useful for subtitles/karaoke effects).
- **Cross-platform** native libraries: Windows, macOS, Linux, iOS, Android, visionOS, with optional GPU acceleration (Vulkan / Metal).

## Documentation map

| Document | What it covers |
|----------|----------------|
| [01 — Getting Started](01-Getting-Started.md) | Requirements, installation (Package Manager / git URL), placing model weights, first transcription. |
| [08 — Scene Setup](08-Scene-Setup.md) | Hands-on companion to Getting Started: the GameObjects/components to create and how to wire them in the Inspector (minimal, microphone, and streaming scenes). |
| [02 — Architecture](02-Architecture.md) | How the package is layered, the native binding, threading, the data flow from audio to text. |
| [03 — API Reference](03-API-Reference.md) | Every public class, property, method, and event you'll use. |
| [04 — Integration Guide](04-Integration-Guide.md) | Practical, copy-pasteable examples: transcribe a clip, record from mic, stream live, build subtitles, push-to-talk voice commands. |
| [09 — Worked Example: Mic Streaming Game](09-Example-Microphone-Streaming-Game.md) | A full, real integration: voice-controlled spellcasting from live microphone streaming — scene, data, controller script, matching, tuning, mobile. |
| [05 — Languages & Models](05-Languages-and-Models.md) | Choosing models, multilingual support, translation, and **how to "train"/fine-tune Whisper for a new language** and convert it for use here. |
| [06 — Platforms & Deployment](06-Platforms-and-Deployment.md) | Platform notes, GPU acceleration, shipping model weights, building the C++ libraries from source. |
| [07 — Troubleshooting](07-Troubleshooting.md) | Common errors and how to fix them. |

## At a glance

```csharp
using UnityEngine;
using Whisper;

public class QuickStart : MonoBehaviour
{
    public WhisperManager whisper;   // drag a WhisperManager from the scene
    public AudioClip clip;           // any clip; will be resampled to 16 kHz mono

    private async void Start()
    {
        // WhisperManager loads the model on Awake by default.
        WhisperResult result = await whisper.GetTextAsync(clip);
        Debug.Log($"Detected language: {result.Language}");
        Debug.Log($"Transcript: {result.Result}");
    }
}
```

> **Note on versions.** This package wraps **whisper.cpp v1.7.5**. Model weights must be in the **GGML (`ggml-*.bin`)** format that matches that whisper.cpp version. The repo ships `ggml-tiny.bin` (multilingual, ~78 MB) under `Assets/StreamingAssets/Whisper/` so you can run immediately.
