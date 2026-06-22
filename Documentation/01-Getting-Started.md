# 1 — Getting Started

## Requirements

- **Unity 2020.1 or newer** (the package targets `.NET Standard 2.0` / 2020.1 per `package.json`).
- One of the supported build targets:
  - Windows (x86_64), macOS (Intel & Apple Silicon), Linux (x86_64)
  - iOS (device & simulator), Android (ARM64), visionOS
  - WebGL is **not** supported (native code can't run in the browser sandbox — see whisper.unity issue #20).
- A microphone is only required for the live recording / streaming examples.

## Installation

There are two ways to use the package.

### Option A — Clone the whole project (easiest to explore)

Clone the repository and open it as a normal Unity project. It already contains:

- the package under `Packages/com.whisper.unity/`,
- five runnable example scenes under `Assets/Samples/`,
- a ready-to-use `ggml-tiny.bin` model in `Assets/StreamingAssets/Whisper/`.

Open any scene under `Assets/Samples/` and press Play.

### Option B — Add as a Unity Package (for your own project)

Open **Window → Package Manager → + → Add package from git URL…** and paste:

```
https://github.com/inzuko-zero/whisper.unity.git?path=/Packages/com.whisper.unity
```

(or the upstream `https://github.com/Macoron/whisper.unity.git?path=/Packages/com.whisper.unity`).

The package brings the runtime scripts and all prebuilt native plugins. It does **not** copy model weights or example scenes into your project — you add those yourself (see below).

## Installing model weights

Whisper needs a model weights file (`ggml-*.bin`). The package loads it from your **`StreamingAssets`** folder by default.

1. Create the folder `Assets/StreamingAssets/Whisper/` in your project (any subfolder works).
2. Put a model file there, e.g. `ggml-tiny.bin`.
   - The repo already ships `ggml-tiny.bin`. To copy it into your own project, grab it from this repo's `Assets/StreamingAssets/Whisper/`.
   - For other sizes/qualities, download GGML models from [Hugging Face — ggerganov/whisper.cpp](https://huggingface.co/ggerganov/whisper.cpp). See [05 — Languages & Models](05-Languages-and-Models.md).
3. In the `WhisperManager` component, set **Model Path** to the path *relative to `StreamingAssets`*, e.g. `Whisper/ggml-tiny.bin`, and keep **Is Model Path In Streaming Assets** checked.

> If you store weights elsewhere (e.g. downloaded at runtime into `Application.persistentDataPath`), uncheck **Is Model Path In Streaming Assets** and set **Model Path** to an absolute path.

## Your first transcription (Editor)

1. Create an empty GameObject and add the **`WhisperManager`** component.
2. Set its **Model Path** to `Whisper/ggml-tiny.bin`.
3. Add this script to another GameObject and assign the `WhisperManager` and an `AudioClip` in the Inspector:

```csharp
using UnityEngine;
using Whisper;

public class HelloWhisper : MonoBehaviour
{
    public WhisperManager whisper;
    public AudioClip clip;

    private async void Start()
    {
        // The model is loaded automatically on Awake (initOnAwake = true).
        var result = await whisper.GetTextAsync(clip);
        if (result == null)
        {
            Debug.LogError("Transcription failed.");
            return;
        }

        Debug.Log($"[{result.Language}] {result.Result}");
    }
}
```

4. Press Play. The first run loads the model into memory (a few hundred ms to seconds depending on size); subsequent calls reuse it.

## Audio requirements (handled for you)

Whisper internally operates on **16 kHz mono** audio. You do **not** need to resample yourself — `WhisperWrapper` calls `AudioUtils.Preprocess(...)` to downmix to mono and resample to 16 kHz before inference. You can feed clips/buffers at any sample rate and channel count.

## The example scenes

Five sample scenes under `Assets/Samples/` demonstrate the main use cases. Each has a matching script you can read alongside [04 — Integration Guide](04-Integration-Guide.md):

| Scene | Script | Demonstrates |
|-------|--------|-------------|
| `1 - Audio Clip` | `AudioClipDemo.cs` | Transcribe a static `AudioClip`; live segment streaming; progress; language & translate toggles. |
| `2 - Microphone` | `MicrophoneDemo.cs` | Record from mic, then transcribe the whole recording. VAD auto-stop. |
| `3 - Languages` | `LanguagesDemo.cs` | Check if a model is multilingual and list all supported languages. |
| `4 - Subtitles` | `SubtitlesDemo.cs` | Token timestamps + per-word confidence rendered as colored subtitles synced to playback. |
| `5 - Streaming` | `StreamingSampleMic.cs` | Real-time streaming transcription from the microphone with a sliding window. |

Next: [08 — Scene Setup](08-Scene-Setup.md) to build a scene step by step, or [02 — Architecture](02-Architecture.md) for how the package works internally.
