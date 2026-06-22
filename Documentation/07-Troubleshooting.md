# 7 — Troubleshooting

Common problems and fixes. Most issues come down to the model path, the model format, threading, or platform settings.

## Model loading

### "Path … doesn't exist!" / model fails to load
- Confirm the file actually exists at `Assets/StreamingAssets/<your path>`.
- `WhisperManager.ModelPath` is **relative to `StreamingAssets`** when `IsModelPathInStreamingAssets` is on. So `Whisper/ggml-tiny.bin` resolves to `StreamingAssets/Whisper/ggml-tiny.bin`.
- If you keep the model elsewhere, turn **off** `IsModelPathInStreamingAssets` and provide an **absolute** path.
- You can't change the path after the model is loaded/loading — it throws `InvalidOperationException`. Set it before `InitModel()`.

### "Failed to load Whisper model!" with a valid path
- The file is probably the **wrong format or version**. whisper.unity needs **GGML `ggml-*.bin`** weights compatible with **whisper.cpp v1.7.5**.
- PyTorch (`.pt`), Hugging Face (`safetensors`), CoreML, or ONNX files will **not** load — convert them to GGML first (see [05 — Languages & Models](05-Languages-and-Models.md)).
- A truncated/corrupted download fails the same way — re-download and check the file size.

### Android: model won't load even though the file is in StreamingAssets
- This is expected if something reads it with `System.IO.File` — on Android `StreamingAssets` is inside the (compressed) APK. The package handles this via `UnityWebRequest` automatically; make sure you're loading through `WhisperManager`/`WhisperWrapper` and not your own `File.ReadAllBytes`.

## Build errors

### Android build throws "Unsupported scripting backend" or "Unsupported architecture"
- whisper.unity for Android requires **IL2CPP** and **ARM64**. Set both in **Player Settings → Other Settings**. (Enforced by `CheckAndroidPreprocessBuild`.)

### iOS build / Bitcode issues
- Bitcode is disabled automatically by the package's `[PostProcessBuild]` step. If you re-enable it manually, the native library may not link — leave it off.

### DllNotFoundException / EntryPointNotFound at runtime
- The native plugin for the current platform isn't being included or matched. Verify the `Plugins/<Platform>` folder has the library and its import settings target that platform.
- On a platform you rebuilt yourself, ensure the libraries were copied into the package `Plugins/` folders (the build scripts do this).
- A struct/signature mismatch (from mixing a different whisper.cpp version with the bundled C# bindings) can also surface here — keep the native libs and `Runtime/Native` in sync at v1.7.5.

## Runtime behavior

### `GetTextAsync` returns `null`
Always null-check. Causes:
- Model not loaded yet (call/await `InitModel()` first, or rely on `initOnAwake`).
- `AudioClip.GetData` failed (bad/empty clip).
- Native inference returned an error.

### App freezes during transcription
- Use the **`*Async`** methods (`GetTextAsync`), not the blocking `WhisperWrapper.GetText`. The async path runs inference on a background thread.

### UI/Transform errors like "can only be called from the main thread"
- You're handling **`WhisperWrapper`** events directly — they fire on a background thread.
- Use **`WhisperManager`** instead (its `OnNewSegment`/`OnProgress` are marshaled to the main thread), or route wrapper events through a `MainThreadDispatcher`.

### Poor accuracy
- Use a **bigger model** (`small`/`medium` instead of `tiny`).
- Set the correct **`language`** instead of relying on auto-detect for short/noisy clips.
- Provide an **`initialPrompt`** with expected vocabulary/names.
- Make sure input audio is clean; very noisy or far-field audio degrades results.
- Don't set `audioCtx` to a low non-zero value unless you accept the quality hit.

### Wrong language detected
- Set `language` explicitly. Auto-detection is unreliable on very short clips or heavy background noise.
- English-only (`*.en`) models can't detect or transcribe other languages — use a multilingual model (`whisper.IsMultilingual()` to verify).

### Translation didn't produce my target language
- `translateToEnglish` only translates **into English**. There is no built-in translation to other languages — transcribe in the source language, then translate the text with a separate system.

## Streaming

### Streaming output is laggy or choppy
- Lower `stepSec` for responsiveness (costs more CPU), or use a smaller model / enable GPU.
- Set `singleSegment = true` and keep `useVad = true` so silence is skipped.

### Stream produces nothing
- Call `StartStream()` **before** starting the microphone.
- If feeding manually, ensure chunks have `IsVoiceDetected = true` when `useVad` is on (otherwise silent chunks are dropped).
- Verify the `MicrophoneRecord.frequency`/channels match what you passed to `CreateStream`.

## Microphone

### No audio captured / empty recording
- Request microphone permission on mobile (`NSMicrophoneUsageDescription` on iOS; `RECORD_AUDIO` + runtime request on Android).
- Check `MicrophoneRecord.SelectedMicDevice` is valid; `AvailableMicDevices` lists options.
- Recording shorter than a chunk, or stopping immediately, can yield zero samples.

## Performance

### First transcription is very slow
- That's the one-time model load. Warm up with `await whisper.InitModel()` during a loading screen; later calls reuse the loaded model.

### High memory usage
- Larger models use a lot of RAM/VRAM (`medium` ~1.5 GB, `large` ~3 GB). Use a smaller or quantized model, especially on mobile. Each loaded `WhisperWrapper` holds its own copy of the weights.

---

If something here doesn't cover your case, check the upstream project issues: https://github.com/Macoron/whisper.unity/issues and whisper.cpp: https://github.com/ggerganov/whisper.cpp.
