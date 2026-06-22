# 3 — API Reference

All public types live in the `Whisper` namespace; utilities live in `Whisper.Utils`; raw bindings in `Whisper.Native`. This reference covers everything you'll use day to day. Source files are linked at each section.

---

## `WhisperManager` (MonoBehaviour)

[Runtime/WhisperManager.cs](../Packages/com.whisper.unity/Runtime/WhisperManager.cs)

The Inspector-driven entry point. Add it to a GameObject and configure it, or set fields from code.

### Inspector fields

| Field | Type | Default | Meaning |
|-------|------|---------|---------|
| `logLevel` | `LogLevel` | `Log` | Verbosity of package logging. |
| `modelPath` (property `ModelPath`) | `string` | `Whisper/ggml-tiny.bin` | Path to weights. Relative to `StreamingAssets` if the flag below is on. |
| `isModelPathInStreamingAssets` (prop `IsModelPathInStreamingAssets`) | `bool` | `true` | Prepend `Application.streamingAssetsPath` to `modelPath`. |
| `initOnAwake` | `bool` | `true` | Load the model automatically in `Awake`. |
| `useGpu` | `bool` | `false` | Try GPU inference (Vulkan/Metal), falling back to CPU. |
| `flashAttention` | `bool` | `false` | Use the Flash Attention algorithm (faster on supported hardware). |
| `language` | `string` | `"en"` | Output language code. Use `""` or `"auto"` for auto-detection. |
| `translateToEnglish` | `bool` | `false` | Translate speech directly into English text. |
| `strategy` | `WhisperSamplingStrategy` | `GREEDY` | `GREEDY` (fast) or `BEAM_SEARCH` (slower, often better). |
| `noContext` | `bool` | `true` | Don't use previous transcription as decoder prompt. |
| `singleSegment` | `bool` | `false` | Force a single output segment (useful for streaming). |
| `enableTokens` | `bool` | `false` | Emit per-token data in each segment. |
| `initialPrompt` | `string` | `""` | Text prompt to bias/guide transcription (vocabulary, style). |
| `stepSec` | `float` | `3` | Streaming: min audio (s) processed per step. |
| `keepSec` | `float` | `0.2` | Streaming: seconds of previous chunk carried into the next. |
| `lengthSec` | `float` | `10` | Streaming: audio length transcribed recurrently before context update. |
| `updatePrompt` | `bool` | `true` | Streaming: feed prior text back as prompt for continuity. |
| `dropOldBuffer` | `bool` | `false` | Streaming: use ggml's bounded old-buffer behavior instead of keeping all. |
| `useVad` | `bool` | `true` | Streaming: skip chunks with no detected speech. |
| `tokensTimestamps` | `bool` | `false` | **[Experimental]** per-token timestamps (requires `enableTokens`). |
| `audioCtx` | `int` | `0` | **[Experimental]** override audio context size (0 = default). Lower = faster, worse quality. |

### Properties

- `bool IsLoaded` — model is loaded and ready.
- `bool IsLoading` — model is currently loading.

### Methods

- `Task InitModel()` — load the model and default params. Safe to await; no-ops if already loaded/loading. Called automatically when `initOnAwake` is true.
- `bool IsMultilingual()` — whether the loaded model supports multiple languages (false for `*.en` models).
- `Task<WhisperResult> GetTextAsync(AudioClip clip)` — transcribe a clip. Returns `null` on failure.
- `Task<WhisperResult> GetTextAsync(float[] samples, int frequency, int channels)` — transcribe a raw buffer.
- `Task<WhisperStream> CreateStream(int frequency, int channels)` — create a manual-feed streaming session.
- `Task<WhisperStream> CreateStream(MicrophoneRecord microphone)` — create a streaming session driven automatically by a microphone.

> If you call a transcription method while the model is still loading, it awaits loading first, then proceeds.

### Events (raised on the Unity main thread)

- `event OnNewSegmentDelegate OnNewSegment` — `void(WhisperSegment segment)`; a new text segment was produced during inference.
- `event OnProgressDelegate OnProgress` — `void(int progress)`; progress 0→100.

---

## `WhisperWrapper`

[Runtime/WhisperWrapper.cs](../Packages/com.whisper.unity/Runtime/WhisperWrapper.cs)

A loaded model, usable without a MonoBehaviour. **Events fire on a background thread.**

### Constants

- `const int WhisperSampleRate = 16000` — the rate whisper expects (preprocessing targets this).

### Static factory methods

- `WhisperWrapper InitFromFile(string modelPath)` / `InitFromFile(string, WhisperContextParams)`
- `Task<WhisperWrapper> InitFromFileAsync(string modelPath)` / `InitFromFileAsync(string, WhisperContextParams)`
- `WhisperWrapper InitFromBuffer(byte[] buffer)` / `InitFromBuffer(byte[], WhisperContextParams)`
- `Task<WhisperWrapper> InitFromBufferAsync(byte[] buffer)` / `InitFromBufferAsync(byte[], WhisperContextParams)`
- `static string GetSystemInfo()` — human-readable string of which CPU/GPU features the **library was compiled to expect** (e.g. `AVX=1`). Note: this reflects build expectations, not your hardware's actual capability.

All factories take an **absolute** path (the `WhisperManager` resolves `StreamingAssets` for you).

### Instance members

- `bool IsMultilingual` — model supports multiple languages.
- `WhisperResult GetText(AudioClip clip, WhisperParams param)` — **blocking** transcription.
- `WhisperResult GetText(float[] samples, int frequency, int channels, WhisperParams param)` — blocking.
- `Task<WhisperResult> GetTextAsync(AudioClip clip, WhisperParams param)` — async.
- `Task<WhisperResult> GetTextAsync(float[] samples, int frequency, int channels, WhisperParams param)` — async.
- `event OnNewSegmentDelegate OnNewSegment` / `event OnProgressDelegate OnProgress` — background-thread callbacks.

### Delegates

```csharp
public delegate void OnNewSegmentDelegate(WhisperSegment text);
public delegate void OnProgressDelegate(int progress);
```

---

## `WhisperParams`

[Runtime/WhisperParams.cs](../Packages/com.whisper.unity/Runtime/WhisperParams.cs)

A safe, managed wrapper over the native inference parameter struct. `WhisperManager` builds and updates one for you; you only touch this directly when using `WhisperWrapper`.

Create with `WhisperParams.GetDefaultParams(WhisperSamplingStrategy strategy = GREEDY)`.

Selected properties (see source for the full set):

| Property | Type | Notes |
|----------|------|------|
| `Strategy` | `WhisperSamplingStrategy` | Greedy or beam search. |
| `ThreadsCount` | `int` | CPU threads (≥1). |
| `Language` | `string` | ISO 639-1 code, or `null`/`""`/`"auto"` to auto-detect. Pinned in unmanaged memory. |
| `Translate` | `bool` | Translate to English. Overrides `Language`. |
| `NoContext` | `bool` | Ignore prior transcription as prompt. |
| `SingleSegment` | `bool` | One segment output. |
| `InitialPrompt` | `string` | Bias text prepended as tokens. Pinned in unmanaged memory. |
| `MaxTextContextCount` | `int` | Max past-text tokens used as prompt. |
| `OffsetMs` / `DurationMs` | `int` | Start offset / duration to process (ms). (`DurationMs` is noted as unreliable in source.) |
| `AudioCtx` | `int` | Override audio context size (experimental). |
| `TokenTimestamps` | `bool` | Per-token timestamps (experimental). |
| `EnableTokens` | `bool` | Unity-side flag: populate `WhisperSegment.Tokens`. |
| `PrintSpecial` / `PrintProgress` / `PrintRealtime` / `PrintTimestamps` | `bool` | Native C++ console logging (off by default; appears only in Unity log file). |
| `NewSegmentCallback` / `ProgressCallback` (+ `…UserData`) | delegate / `IntPtr` | Advanced: supply your own native callbacks. |

> String setters (`Language`, `InitialPrompt`) copy into unmanaged memory and free the previous allocation, so they're safe to change at runtime.

---

## `WhisperContextParams`

[Runtime/WhisperParams.cs](../Packages/com.whisper.unity/Runtime/WhisperParams.cs)

Parameters used at **model load** time. Create with `WhisperContextParams.GetDefaultParams()`.

- `bool UseGpu` — load model for GPU inference.
- `bool FlashAttn` — enable Flash Attention.

---

## Result types

[Runtime/WhisperResult.cs](../Packages/com.whisper.unity/Runtime/WhisperResult.cs)

### `WhisperResult`
- `List<WhisperSegment> Segments` — ordered segments.
- `string Result` — full transcript (all segment texts concatenated).
- `int LanguageId` — detected language id.
- `string Language` — detected language code (e.g. `"en"`).

### `WhisperSegment`
- `int Index` — segment index in the context.
- `string Text` — combined text of the segment.
- `TimeSpan Start` / `TimeSpan End` — segment timing relative to the audio.
- `WhisperTokenData[] Tokens` — per-token data; `null` unless `EnableTokens` was set.
- `string TimestampToString()` — `"[mm:ss:fff->mm:ss:fff]"`.

### `WhisperTokenData`
- `int Id` — token id in the vocabulary.
- `float Prob` — confidence in `[0,1]`.
- `float ProbLog` — log-probability.
- `string Text` — token text.
- `bool IsSpecial` — true for special tokens (`[EOT]`, `[BEG]`, …).
- `WhisperTokenTimestamp Timestamp` — per-token timing; `null` unless `TokenTimestamps` was set.

### `WhisperTokenTimestamp`
- `int Id`, `float Prob`, `float ProbSum`, `TimeSpan Start`, `TimeSpan End`, `float VoiceLength`.

---

## `WhisperStream` & `WhisperStreamParams`

[Runtime/WhisperStream.cs](../Packages/com.whisper.unity/Runtime/WhisperStream.cs)

Real-time transcription with a sliding window + optional VAD. Create via `WhisperManager.CreateStream(...)`.

### `WhisperStream` methods
- `void StartStream()` — begin a session. If created from a `MicrophoneRecord`, it auto-subscribes to mic chunks (no manual feeding needed).
- `void AddToStream(AudioChunk chunk)` — manually feed audio (when not using a mic).
- `void StopStream()` — finish: processes the last buffered audio, raises `OnStreamFinished`, then resets.

### `WhisperStream` events
- `event OnStreamResultUpdatedDelegate OnResultUpdated` — `void(string updatedResult)`; full transcript so far.
- `event OnStreamSegmentUpdatedDelegate OnSegmentUpdated` — `void(WhisperResult segment)`; current (in-progress) segment.
- `event OnStreamSegmentFinishedDelegate OnSegmentFinished` — `void(WhisperResult segment)`; a segment was finalized.
- `event OnStreamFinishedDelegate OnStreamFinished` — `void(string finalResult)`; the stream ended.

> When the stream is created from a `MicrophoneRecord`, it also auto-calls `StopStream()` on the mic's `OnRecordStop`. Stream callbacks come via `WhisperWrapper` (background thread) but the mic-driven path runs through Unity's event loop; if you touch UI from stream events created without `WhisperManager`'s dispatcher, marshal to the main thread.

### `WhisperStreamParams`
Built internally from `WhisperManager`'s streaming fields. Key readonly values: `InferenceParam`, `Frequency`, `Channels`, `StepSec`/`StepSamples`, `KeepSec`/`KeepSamples`, `LengthSec`/`LengthSamples`, `StepsCount`, `UpdatePrompt`, `DropOldBuffer`, `UseVad`.

---

## `WhisperLanguage` (static)

[Runtime/WhisperLanguage.cs](../Packages/com.whisper.unity/Runtime/WhisperLanguage.cs)

- `int GetLanguageMaxId()` — largest valid language id.
- `int GetLanguageId(string lang)` — id for `"de"` or `"german"`; `-1` if unknown.
- `string GetLanguageString(int id)` — code for an id (e.g. `2 -> "de"`); `null` if invalid.
- `string[] GetAllLanguages()` — every language code whisper.cpp knows (cached). Note: not every model supports every language.

---

## `MicrophoneRecord` (MonoBehaviour, `Whisper.Utils`)

[Runtime/Utils/MicrophoneRecord.cs](../Packages/com.whisper.unity/Runtime/Utils/MicrophoneRecord.cs)

Microphone capture with chunking and energy-based VAD.

### Key Inspector fields
- `maxLengthSec` (default 60), `loop`, `frequency` (default 16000), `chunksLengthSec` (0.5), `echo`.
- VAD: `useVad`, `vadUpdateRateSec`, `vadContextSec`, `vadLastSec`, `vadThd`, `vadFreqThd`, `vadIndicatorImage`.
- VAD-stop: `vadStop`, `dropVadPart`, `vadStopTime`.
- Selection: `microphoneDropdown`, `microphoneDefaultLabel`, `SelectedMicDevice`, `AvailableMicDevices`.

### Methods / state
- `void StartRecord()` / `void StopRecord(float dropTimeSec = 0f)`.
- `bool IsRecording`, `bool IsVoiceDetected`, `int ClipSamples`.

### Events
- `event OnChunkReadyDelegate OnChunkReady` — `void(AudioChunk)`; a new chunk (streaming).
- `event OnRecordStopDelegate OnRecordStop` — `void(AudioChunk)`; recording stopped, returns the full recorded audio.
- `event OnVadChangedDelegate OnVadChanged` — `void(bool isSpeechDetected)`.

### `AudioChunk` (struct)
- `float[] Data`, `int Frequency`, `int Channels`, `float Length`, `bool IsVoiceDetected`.

---

## `Whisper.Native` (advanced)

[Runtime/Native/WhisperNative.cs](../Packages/com.whisper.unity/Runtime/Native/WhisperNative.cs), [Runtime/Native/WhisperNativeParams.cs](../Packages/com.whisper.unity/Runtime/Native/WhisperNativeParams.cs)

Raw `DllImport` bindings and `StructLayout` mirrors of whisper.cpp v1.7.5 structs. You normally never call these directly. `WhisperSamplingStrategy` lives here:

```csharp
public enum WhisperSamplingStrategy { WHISPER_SAMPLING_GREEDY = 0, WHISPER_SAMPLING_BEAM_SEARCH = 1 }
```

Next: [04 — Integration Guide](04-Integration-Guide.md).
