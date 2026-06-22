# 4 — Integration Guide

Practical, copy-pasteable recipes for putting whisper.unity into a real game. Each example notes which sample scene it's based on so you can compare against a working scene.

> **Setup assumed for all examples:** a `WhisperManager` exists in the scene with a valid **Model Path** (e.g. `Whisper/ggml-tiny.bin`), and you've assigned references in the Inspector where the script declares `public` fields.

---

## Recipe 1 — Transcribe an existing AudioClip

The simplest case: you have a clip (recorded earlier, downloaded, an `AudioSource`'s clip, etc.) and want its text.

```csharp
using UnityEngine;
using Whisper;

public class TranscribeClip : MonoBehaviour
{
    public WhisperManager whisper;
    public AudioClip clip;

    public async void Transcribe()
    {
        var result = await whisper.GetTextAsync(clip);
        if (result == null)
        {
            Debug.LogError("Transcription failed (model not loaded / bad audio).");
            return;
        }

        Debug.Log($"Detected language: {result.Language}");
        Debug.Log($"Text: {result.Result}");

        foreach (var seg in result.Segments)
            Debug.Log($"{seg.TimestampToString()} {seg.Text}");
    }
}
```

Notes:
- `GetTextAsync` runs inference off the main thread, so the call won't hang your game.
- The clip can be any sample rate / channel count; it's resampled to 16 kHz mono internally.
- See `Assets/Samples/1 - Audio Clip/AudioClipDemo.cs` for a full UI version with a language dropdown, translate toggle, and live segment display.

### Showing progress and partial segments

`WhisperManager` raises main-thread-safe events during inference:

```csharp
private void OnEnable()
{
    whisper.OnProgress   += p => Debug.Log($"Progress: {p}%");
    whisper.OnNewSegment += s => Debug.Log($"New segment: {s.Text}");
}
```

You can build a "typing as it transcribes" effect by appending each `OnNewSegment` text to a buffer and showing `buffer + "..."` (exactly what `AudioClipDemo` does).

---

## Recipe 2 — Record from the microphone, then transcribe

Use the provided `MicrophoneRecord` component. Add it to a GameObject, then transcribe the recording when it stops. (Based on `MicrophoneDemo.cs`.)

```csharp
using UnityEngine;
using Whisper;
using Whisper.Utils;

public class PushToTalk : MonoBehaviour
{
    public WhisperManager whisper;
    public MicrophoneRecord mic;

    private void Awake()
    {
        mic.OnRecordStop += OnRecordStop;
    }

    // Call from a UI button / key press to toggle recording.
    public void Toggle()
    {
        if (!mic.IsRecording) mic.StartRecord();
        else                  mic.StopRecord();
    }

    private async void OnRecordStop(AudioChunk recorded)
    {
        var result = await whisper.GetTextAsync(
            recorded.Data, recorded.Frequency, recorded.Channels);

        if (result != null)
            Debug.Log($"You said: {result.Result}");
    }
}
```

Tips:
- Enable **`vadStop`** on `MicrophoneRecord` to auto-stop after `vadStopTime` seconds of silence — great for hands-free "speak a command" UX.
- `echo = true` plays back the recording when it stops (handy while testing).
- On mobile you must request microphone permission (see [06 — Platforms & Deployment](06-Platforms-and-Deployment.md)).

---

## Recipe 3 — Real-time streaming transcription

For live captions / continuous dictation, use a `WhisperStream` driven by the microphone. (Based on `StreamingSampleMic.cs`.)

```csharp
using UnityEngine;
using Whisper;
using Whisper.Utils;

public class LiveCaptions : MonoBehaviour
{
    public WhisperManager whisper;
    public MicrophoneRecord mic;

    private WhisperStream _stream;

    private async void Start()
    {
        // Build a stream bound to the microphone (auto-feeds chunks).
        _stream = await whisper.CreateStream(mic);

        _stream.OnResultUpdated   += full => Debug.Log($"Live: {full}");
        _stream.OnSegmentFinished += seg  => Debug.Log($"Final segment: {seg.Result}");
        _stream.OnStreamFinished  += final => Debug.Log($"Done: {final}");
    }

    public void Toggle()
    {
        if (!mic.IsRecording)
        {
            _stream.StartStream();   // start BEFORE recording
            mic.StartRecord();
        }
        else
        {
            mic.StopRecord();        // stream auto-stops via mic's OnRecordStop
        }
    }
}
```

How streaming behaves (tune via the `WhisperManager` "Streaming settings"):
- **`stepSec`** — how much new audio accumulates before each transcription pass (lower = more responsive, more CPU).
- **`lengthSec`** — total recurrent window length before the context resets.
- **`keepSec`** — overlap carried from the previous window for continuity.
- **`useVad`** — skip silent chunks so you don't transcribe nothing.
- **`updatePrompt`** — feed already-transcribed text back as a prompt so later words stay consistent.
- **`singleSegment = true`** is recommended for streaming.

> Streaming runs many small inferences in a row. For smooth real-time on CPU, use a small model (`tiny`/`base`) and consider enabling GPU. If a pass is still running when new audio arrives, the stream skips ahead — it won't queue up unbounded work.

### Manual feeding (no microphone)

If your audio comes from somewhere other than `MicrophoneRecord` (network voice chat, a custom capture), feed chunks yourself:

```csharp
_stream = await whisper.CreateStream(frequency: 16000, channels: 1);
_stream.StartStream();

// each time you have new audio samples:
_stream.AddToStream(new AudioChunk {
    Data = samples, Frequency = 16000, Channels = 1,
    Length = samples.Length / 16000f, IsVoiceDetected = true
});

// when done:
_stream.StopStream();
```

---

## Recipe 4 — Subtitles / karaoke with per-word confidence

Enable token data and per-token timestamps, then color each word by confidence while audio plays. (Based on `SubtitlesDemo.cs`.)

```csharp
// Must be set BEFORE transcribing (the demo forces them in Awake):
whisper.enableTokens     = true;
whisper.tokensTimestamps = true;

var res = await whisper.GetTextAsync(clip);

// While an AudioSource plays `clip`, build rich text up to source.time:
string BuildSubtitle(WhisperResult res, float timeSec)
{
    var sb = new System.Text.StringBuilder();
    var time = System.TimeSpan.FromSeconds(timeSec);

    foreach (var seg in res.Segments)
    foreach (var token in seg.Tokens)
    {
        if (token.IsSpecial) continue;
        if (time < token.Timestamp.Start) continue;

        // color by confidence: red < 0.33 < yellow < 0.66 < green
        string color = token.Prob <= 0.33f ? "red"
                     : token.Prob <= 0.66f ? "yellow" : "green";
        sb.Append($"<color={color}>{token.Text}</color>");
    }
    return sb.ToString();
}
```

Use this in an `Update`/coroutine loop while the `AudioSource` is playing and assign to a `Text` with rich text enabled. `WhisperSegment.TimestampToString()` is handy if you'd rather render plain timestamps.

---

## Recipe 5 — Voice commands (keyword matching)

Combine push-to-talk with simple string matching for a low-effort command system. For best accuracy, bias the model with an `initialPrompt` listing your expected vocabulary.

```csharp
private void Awake()
{
    // Nudge Whisper toward your game's vocabulary.
    whisper.initialPrompt = "Commands: attack, defend, heal, retreat, open map.";
    mic.OnRecordStop += OnRecordStop;
}

private async void OnRecordStop(AudioChunk recorded)
{
    var res = await whisper.GetTextAsync(recorded.Data, recorded.Frequency, recorded.Channels);
    if (res == null) return;

    var text = res.Result.ToLowerInvariant();
    if (text.Contains("attack"))      Attack();
    else if (text.Contains("defend")) Defend();
    else if (text.Contains("heal"))   Heal();
}
```

The `initialPrompt` doesn't restrict output, but it strongly biases recognition toward those words — very effective for proper nouns and game-specific terms.

---

## Recipe 6 — Using `WhisperWrapper` directly (no MonoBehaviour)

For editor tools, tests, or non-scene code. Remember: wrapper events fire on a **background thread**.

```csharp
using System.IO;
using UnityEngine;
using Whisper;

public static class OfflineTranscriber
{
    public static async System.Threading.Tasks.Task<string> Run(string absModelPath, AudioClip clip)
    {
        var ctxParams = WhisperContextParams.GetDefaultParams();
        ctxParams.UseGpu = false;

        var model = await WhisperWrapper.InitFromFileAsync(absModelPath, ctxParams);
        if (model == null) return null;

        var p = WhisperParams.GetDefaultParams();
        p.Language = "auto";          // auto-detect
        p.EnableTokens = false;

        var res = await model.GetTextAsync(clip, p);
        return res?.Result;
    }
}
```

To load weights from a custom location (e.g. downloaded at runtime):

```csharp
byte[] bytes = File.ReadAllBytes(Path.Combine(Application.persistentDataPath, "ggml-base.bin"));
var model = await WhisperWrapper.InitFromBufferAsync(bytes);
```

---

## Recipe 7 — Choosing or swapping the model at runtime

`WhisperManager` can only have its model path set **before** loading. To switch models, change the path while unloaded, or use separate managers.

```csharp
public WhisperManager whisper; // initOnAwake = false in Inspector

public async void LoadModel(string streamingAssetsRelativePath)
{
    if (whisper.IsLoaded || whisper.IsLoading) return; // can't change after load
    whisper.ModelPath = streamingAssetsRelativePath;   // e.g. "Whisper/ggml-base.bin"
    whisper.IsModelPathInStreamingAssets = true;
    await whisper.InitModel();
}
```

For downloaded models stored outside the project, set `IsModelPathInStreamingAssets = false` and pass an absolute path.

---

## Common gotchas

- **Don't touch UI from `WhisperWrapper` events** — they're on a worker thread. Use `WhisperManager` (events marshaled for you) or a `MainThreadDispatcher`.
- **`GetTextAsync` can return `null`** — always null-check (model not loaded, bad audio, inference error).
- **One inference at a time per model** — calls are serialized by a lock. For parallel work, load multiple models (more memory).
- **Set `enableTokens` / `tokensTimestamps` before transcribing** — they change what the inference produces.
- **First call is slow** — model load happens once; warm it up during a loading screen with `await whisper.InitModel()`.

Next: [05 — Languages & Models](05-Languages-and-Models.md).
