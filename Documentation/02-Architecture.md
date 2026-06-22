# 2 — Architecture

This document explains how the package is structured and how audio flows through it to become text. Understanding the layers makes the API and the integration patterns much easier to reason about.

## Layered overview

```
┌──────────────────────────────────────────────────────────────┐
│  Your game code / Sample scripts                               │
│  (AudioClipDemo, MicrophoneDemo, StreamingSampleMic, ...)      │
└───────────────┬──────────────────────────────────────────────┘
                │ MonoBehaviour-friendly API, Unity events on main thread
┌───────────────▼──────────────────────────────────────────────┐
│  WhisperManager (MonoBehaviour)                                │
│  - Inspector config, model lifecycle, param mapping           │
│  - Marshals callbacks back to the Unity main thread            │
└───────────────┬──────────────────────────────────────────────┘
                │ plain C# (no MonoBehaviour)
┌───────────────▼──────────────────────────────────────────────┐
│  WhisperWrapper            WhisperStream                       │
│  - load model, run         - sliding window, VAD, prompt       │
│    inference, parse           handling for live audio          │
│    segments/tokens                                            │
│  WhisperParams / WhisperContextParams  (safe param wrappers)  │
│  WhisperResult / WhisperSegment / WhisperTokenData (results)   │
│  WhisperLanguage (language id <-> code helpers)               │
└───────────────┬──────────────────────────────────────────────┘
                │ P/Invoke (DllImport)
┌───────────────▼──────────────────────────────────────────────┐
│  Whisper.Native.WhisperNative + struct mirrors                 │
│  (WhisperNativeParams, WhisperNativeContextParams, ...)        │
└───────────────┬──────────────────────────────────────────────┘
                │ native shared/static library
┌───────────────▼──────────────────────────────────────────────┐
│  whisper.cpp v1.7.5 (libwhisper + ggml)                        │
│  Plugins/{Windows,MacOS,Linux,iOS,Android}                     │
└──────────────────────────────────────────────────────────────┘
```

## Folders

```
Packages/com.whisper.unity/
├── Runtime/
│   ├── WhisperManager.cs         MonoBehaviour entry point (Inspector + events)
│   ├── WhisperWrapper.cs         Loaded model: load + inference + result parsing
│   ├── WhisperStream.cs          Streaming logic (sliding window, VAD, prompts)
│   ├── WhisperParams.cs          WhisperParams + WhisperContextParams (safe wrappers)
│   ├── WhisperResult.cs          WhisperResult / WhisperSegment / WhisperTokenData
│   ├── WhisperLanguage.cs        language id <-> code helpers
│   ├── Native/
│   │   ├── WhisperNative.cs        DllImport bindings
│   │   └── WhisperNativeParams.cs  StructLayout mirrors of C++ structs
│   └── Utils/
│       ├── AudioUtils.cs           resample/downmix + simple VAD
│       ├── MicrophoneRecord.cs     mic capture, chunking, VAD
│       ├── FileUtils.cs            cross-platform model file reading
│       ├── MainThreadDispatcher.cs marshal callbacks to Unity main thread
│       └── ... (Text/Log/Ui helpers, demo helpers)
├── Editor/                        iOS/Android build post/pre-processors
├── Plugins/{Windows,MacOS,Linux,iOS,Android}   prebuilt native libs
└── Tests/Runtime/                 play-mode tests
```

## The two ways to use it

There are two entry points, and you pick based on whether you want Unity Inspector wiring or full programmatic control.

### 1. `WhisperManager` (recommended for most games)

A `MonoBehaviour` you drop in a scene. It exposes all common settings in the Inspector, owns the model's lifecycle, and re-raises whisper's worker-thread callbacks **on the Unity main thread** (so you can touch UI/Transforms safely). It internally holds one `WhisperWrapper` and one `WhisperParams` instance.

### 2. `WhisperWrapper` (no MonoBehaviour, full control)

A plain C# class. Create it with `WhisperWrapper.InitFromFileAsync(absolutePath)` (or `InitFromBuffer*`), then call `GetText` / `GetTextAsync`. Its `OnNewSegment` / `OnProgress` events fire **on a background thread** — you must marshal to the main thread yourself (e.g. via `MainThreadDispatcher`) before touching Unity objects. Use this when you don't want a scene component, or in tests/tools.

## Data flow: from audio to text

1. **Input.** You pass an `AudioClip`, a `float[]` buffer (+ frequency + channels), or stream chunks.
2. **Extract samples.** For an `AudioClip`, `WhisperWrapper` calls `clip.GetData(...)` into a `float[]`.
3. **Preprocess.** `AudioUtils.Preprocess(samples, frequency, channels, 16000)` downmixes to mono and resamples to **16 kHz** (`WhisperWrapper.WhisperSampleRate`).
4. **Marshal params.** `WhisperParams` keeps a `WhisperNativeParams` struct in sync; language/prompt strings are pinned in unmanaged memory to survive GC.
5. **Register callbacks.** If you didn't supply custom ones, the wrapper installs static `new_segment_callback` and `progress_callback` (IL2CPP-safe `[MonoPInvokeCallback]` statics) routed through a `GCHandle` back to the instance.
6. **Inference.** `whisper_full(ctx, params, samplesPtr, length)` runs on a **background thread** via `Task.Factory.StartNew` (the `*Async` methods). The call is guarded by a `lock` so one model processes one request at a time.
7. **Read results.** It reads `whisper_full_n_segments`, then per segment the text and `t0/t1` timestamps; if `EnableTokens` is set, it also reads each token's id, probability, text, and (optionally) per-token timestamps.
8. **Detected language.** `whisper_full_lang_id` is mapped to a code via `WhisperLanguage`.
9. **Return.** A `WhisperResult` (full text + segments + language) is returned to your `await`.

## Threading model — the important part

- **Inference runs off the main thread.** `GetTextAsync` returns a `Task` you `await`; the heavy work happens on a thread-pool thread so your game doesn't freeze.
- **`WhisperManager` events are main-thread-safe.** Its `OnNewSegment` and `OnProgress` are routed through a `MainThreadDispatcher` (drained in `Update()`), so handlers can update UI directly.
- **`WhisperWrapper` events are NOT.** If you use the wrapper directly, its `OnNewSegment` / `OnProgress` fire on the worker thread. Marshal them yourself.
- **One request at a time per model.** The `lock` in `WhisperWrapper.GetText` serializes inference. For concurrent transcriptions you'd need multiple loaded models (multiple `WhisperWrapper` instances), which multiplies memory use.

## Native binding details

- `WhisperNative` declares the `DllImport`s. The library name resolves to `"__Internal"` on iOS/visionOS/Android (statically linked) and `"libwhisper"` elsewhere.
- `WhisperNativeParams` / `WhisperNativeContextParams` / `WhisperNativeTokenData` are `[StructLayout(LayoutKind.Sequential)]` **exact mirrors** of the C++ structs in whisper.cpp **v1.7.5**. They must not be edited unless the native library changes — a field mismatch corrupts memory.
- Callbacks use `CallingConvention.StdCall` and are passed as static methods (required for IL2CPP/AOT).

## Model file loading across platforms

`FileUtils` abstracts reading the weights file:

- On most platforms it's a direct `File.ReadAllBytes` / async equivalent.
- On **Android**, `StreamingAssets` lives compressed inside the APK, so files can't be read with `File` APIs. `FileUtils` instead reads them via `UnityWebRequest` (`ReadFileWebRequest`). This is handled automatically.

The bytes are then handed to `whisper_init_from_buffer_with_params`. whisper.cpp **copies** the buffer internally, so the managed array can be freed afterward.

Next: [03 — API Reference](03-API-Reference.md).
