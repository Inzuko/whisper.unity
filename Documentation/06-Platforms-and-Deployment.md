# 6 — Platforms & Deployment

Whisper inference is native C++ (whisper.cpp v1.7.5). The package ships prebuilt libraries for every supported platform under `Packages/com.whisper.unity/Plugins/`. This document covers per-platform requirements, GPU acceleration, shipping model weights, and rebuilding the native libraries.

## Supported platforms

| Platform | Arch | GPU option | Notes |
|----------|------|-----------|-------|
| Windows | x86_64 | Vulkan (optional) | `libwhisper` dynamic library. |
| macOS | Intel + Apple Silicon | Metal (optional) | Universal `.dylib`s. |
| Linux | x86_64 | Vulkan (optional) | `libwhisper`. |
| iOS | Device + Simulator | Metal (optional) | Statically linked (`__Internal`), Bitcode disabled automatically. |
| Android | ARM64 only | — | Statically linked; **IL2CPP + ARM64 required**. |
| visionOS | — | Metal (optional) | Statically linked. |
| **WebGL** | — | — | **Not supported** (no native code in the browser). |

## GPU acceleration

Whisper can use **Vulkan** (Windows, Linux) or **Metal** (macOS, iOS, visionOS). To enable: select your `WhisperManager` and turn on **Use GPU** (`useGpu`). Whisper attempts GPU inference and **falls back to CPU** if unsupported — so it's safe to leave on.

```csharp
// equivalent in code, before the model loads:
// (set the inspector toggle, or build a WhisperContextParams manually)
var ctx = WhisperContextParams.GetDefaultParams();
ctx.UseGpu = true;
ctx.FlashAttn = true; // optional, faster on supported hardware
```

Caveats:
- **Metal** requires Apple7-class GPUs or newer (Apple M1+). Older hardware falls back to CPU.
- **CUDA is no longer supported** — it was replaced by Vulkan. If you specifically need CUDA, use an earlier release of the package.
- `flashAttention` (`FlashAttn`) can further speed up inference on supported hardware.

You can check what the compiled library expects with `WhisperWrapper.GetSystemInfo()` (e.g. `AVX=1 NEON=1 METAL=1`). This reflects how the library was **built**, not necessarily what your CPU/GPU supports at runtime.

## Shipping model weights

Model files (`ggml-*.bin`) live in **`StreamingAssets`** and are read at runtime via `FileUtils`.

- **Desktop & iOS:** `StreamingAssets` files are copied verbatim and read with normal file APIs.
- **Android:** `StreamingAssets` is packed (compressed) inside the APK, so files can't be read with `System.IO.File`. The package automatically reads them via `UnityWebRequest` instead — no action needed from you.

### Reducing app size

Large models bloat your build. Options:
- Use a **smaller or quantized** model (see [05 — Languages & Models](05-Languages-and-Models.md)).
- **Download the model at runtime** into `Application.persistentDataPath`, then load it with `IsModelPathInStreamingAssets = false` and an absolute path (or `WhisperWrapper.InitFromBufferAsync(bytes)`). This keeps the model out of the initial download.

## Platform-specific setup

### Android
- **Scripting backend must be IL2CPP** and **target architecture must be ARM64**. The package enforces this at build time via `CheckAndroidPreprocessBuild` (the build throws a clear error otherwise). Set both in **Player Settings → Other Settings**.
- For microphone features, add the `RECORD_AUDIO` permission and request it at runtime (`Application.RequestUserAuthorization(UserAuthorization.Microphone)` or `Android.Permission`).

### iOS / visionOS
- Libraries are statically linked as `__Internal`.
- **Bitcode is disabled automatically** in the generated Xcode project by `DisableBitcodePostProcess` (a `[PostProcessBuild]` step) — no manual Xcode edits needed.
- Add a **microphone usage description** (`NSMicrophoneUsageDescription`) in Player Settings for mic features, or the app is rejected/crashes on mic access.

### Windows / macOS / Linux
- No special steps; the native libraries are imported as standard plugins. On macOS the build script sets an `@loader_path` rpath so Unity resolves the dependent `.dylib`s.

## Building the native C++ libraries from source

The package ships prebuilt binaries, so you only need this if you want to change the whisper.cpp version, build flags, or platform options.

### Easiest: GitHub Actions
Fork the repo and run **Actions → Build C++ → Run workflow**. Download the compiled libraries from the run's artifacts and drop them into `Plugins/`.

### Locally

1. Clone [whisper.cpp](https://github.com/ggerganov/whisper.cpp) and check out tag **v1.7.5** (other versions may not match the C# struct mirrors).
2. Open the whisper.unity folder in a terminal.
3. Run the script for your OS (each writes the built libs straight into the package `Plugins/` folders):

   **Windows** (produces the Windows library):
   ```bat
   .\build_cpp.bat path\to\whisper
   ```

   **macOS** (produces macOS, iOS, and Android libraries):
   ```bash
   sh build_cpp.sh path/to/whisper all path/to/ndk/android.toolchain.cmake
   ```
   You can pass `mac`, `ios`, or `android` instead of `all` to build a single target.

   **Linux** (produces the Linux library):
   ```bash
   sh build_cpp_linux.sh path/to/whisper
   ```

Build flags used by the scripts (for reference): Release build, examples/tests off, Metal on for Apple targets (`-DGGML_METAL=ON`, embedded Metal library), static libs for iOS/Android, OpenMP off for Android, universal `x86_64;arm64` for macOS/iOS.

> If you bump whisper.cpp to a newer version, you may also need to update the `[StructLayout]` mirrors in `Runtime/Native/WhisperNativeParams.cs` to match any changed C++ structs — a mismatch corrupts memory at runtime.

## Performance tips

- Warm up the model during a loading screen: `await whisper.InitModel();`.
- Pick the smallest model that meets your quality bar; `tiny`/`base` are realtime-capable on CPU, larger models usually want GPU.
- For streaming, prefer GPU + a small model + `singleSegment = true`; tune `stepSec`/`lengthSec`.
- Inference is serialized per model — don't expect two simultaneous transcriptions from one `WhisperManager`.
- `audioCtx` can speed up inference by shrinking the audio context, at a real quality cost (experimental).

Next: [07 — Troubleshooting](07-Troubleshooting.md).
