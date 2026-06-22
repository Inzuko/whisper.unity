# 9 — Worked Example: Microphone Streaming in a Game Scene

This is a **complete, real integration** — not snippets. We build a small voice-controlled game feature: the player holds a magic stick and **casts spells by speaking them aloud**. Speech is captured from the microphone and transcribed **live (streaming)** while the player talks; when a known spell phrase is recognized, the spell fires.

By the end you'll have a working scene with:

- continuous **microphone streaming** transcription (no "press to stop" needed),
- **Voice Activity Detection** so silence isn't transcribed,
- live caption text + a "casting…" indicator,
- a data-driven **spell book** that maps spoken phrases to in-game actions,
- correct **threading**, **lifecycle**, and **mobile permission** handling.

It builds on [08 — Scene Setup](08-Scene-Setup.md) (GameObject wiring) and [04 — Integration Guide](04-Integration-Guide.md) (the streaming API). Read those for the per-field reference; here we focus on a full, coherent example.

---

## 1. How it works (the big picture)

```
Player speaks ──▶ MicrophoneRecord ──chunks──▶ WhisperStream ──▶ WhisperManager model
   "fireball"      (mic + VAD)        0.5s        sliding window      (whisper.cpp)
                                                       │
                                          OnResultUpdated / OnSegmentFinished
                                                       │  (Unity main thread)
                                                       ▼
                                            SpellcastController
                                       (match phrase → cast spell, update UI)
```

Three package components do the heavy lifting:

| Component | Responsibility |
|-----------|----------------|
| `MicrophoneRecord` (`Whisper.Utils`) | Opens the mic, emits ~0.5s `AudioChunk`s, runs energy-based VAD. |
| `WhisperStream` (`Whisper`) | Maintains a sliding audio window, calls the model, raises live transcript events. |
| `WhisperManager` (`Whisper`) | Owns the loaded model; `CreateStream(mic)` produces a mic-driven `WhisperStream`. |

**Key fact that makes this easy:** when the stream is created from a `MicrophoneRecord`, it **auto-feeds** itself — you never call `AddToStream` manually. And because the mic drives chunks from Unity's `Update()` loop, the stream's events resume on the **main thread**, so you can touch UI/gameplay directly in the handlers. (This is why the shipped `StreamingSampleMic` updates `Text` straight from the events.)

---

## 2. Scene setup

Hierarchy (logic objects + a uGUI Canvas):

```
SpellcastScene
├── Whisper            → WhisperManager
├── Microphone         → MicrophoneRecord
├── SpellcastSystem    → SpellcastController  (the script below)
├── MagicStick         → your stick model / VFX origin
└── Canvas
    ├── CaptionText     (Text)   live transcript
    ├── StatusText      (Text)   "Listening…" / "Casting: Fireball!"
    └── VadIndicator    (Image)  green when speech detected
```

### Configure `WhisperManager` (on `Whisper`)
- **Model Path:** `Whisper/ggml-tiny.bin` (in `Assets/StreamingAssets/Whisper/`). For better spell-word accuracy, ship `ggml-base.bin` or `ggml-small.bin` instead — see [05 — Languages & Models](05-Languages-and-Models.md).
- **Init On Awake:** ✔ (loads the model at startup).
- **Use Gpu:** ✔ if targeting desktop/Apple GPUs — streaming runs many inferences, GPU helps a lot.
- **Language:** `en`.
- **Streaming settings** (these matter for responsiveness):
  - **Step Sec:** `1.0` — transcribe roughly once per second (lower = snappier, more CPU).
  - **Length Sec:** `5` — recurrent window length before context reset.
  - **Keep Sec:** `0.2` — overlap carried between windows.
  - **Single Segment:** ✔ — recommended for streaming.
  - **Use Vad:** ✔ — skip silent chunks.
- **Initial Prompt:** list your spell vocabulary so Whisper is biased toward it (see step 4) — this dramatically improves recognition of made-up spell names.

### Configure `MicrophoneRecord` (on `Microphone`)
- **Frequency:** `16000` (Whisper's native rate — avoids resampling).
- **Chunks Length Sec:** `0.5` — how often chunks are emitted to the stream.
- **Use Vad:** ✔, and assign **Vad Indicator Image** ← `VadIndicator`.
- **Vad Stop:** ✘ for *continuous* listening (we keep the mic open the whole time). Turn it ✔ only if you want push-to-talk that auto-ends on silence.
- **Echo:** ✘ (you don't want the game echoing the player's voice back).

Wire the controller's fields in step 3.

---

## 3. The spell data (data-driven, designer-friendly)

Define spells as a `ScriptableObject` so designers can add/edit them without touching code.

```csharp
// SpellDefinition.cs
using System;
using UnityEngine;

[CreateAssetMenu(menuName = "Spellcasting/Spell", fileName = "NewSpell")]
public class SpellDefinition : ScriptableObject
{
    [Tooltip("Display name, e.g. \"Fireball\".")]
    public string spellName;

    [Tooltip("Spoken phrases that trigger this spell. Lowercase; the matcher is case-insensitive. " +
             "Add variants the recognizer is likely to produce, e.g. \"fire ball\", \"fireball\".")]
    public string[] triggerPhrases;

    [Tooltip("Optional cooldown in seconds before this spell can fire again.")]
    public float cooldownSeconds = 1.5f;

    [NonSerialized] public float LastCastTime = -999f;
}
```

Create a few assets (right-click → Create → Spellcasting → Spell), e.g. `Fireball` with phrases `["fireball", "fire ball"]`, `Frost` with `["frost", "freeze", "frostbite"]`, `Heal` with `["heal", "healing", "mend"]`.

---

## 4. The controller — full, compilable script

This is the heart of the example. It opens the stream on `Start`, listens continuously, matches recognized text against the spell book, fires spells, and drives the UI. **Note the two `using` directives** — `WhisperManager` is in `Whisper`, but `MicrophoneRecord`/`AudioChunk` are in `Whisper.Utils` (this is the exact cause of the common `CS0246: MicrophoneRecord could not be found` error).

```csharp
// MagicStickSpellcast.cs
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using UnityEngine.UI;
using Whisper;          // WhisperManager, WhisperStream, WhisperResult
using Whisper.Utils;    // MicrophoneRecord, AudioChunk

[RequireComponent(typeof(AudioSource))]
public class MagicStickSpellcast : MonoBehaviour
{
    [Header("Whisper")]
    public WhisperManager whisper;
    public MicrophoneRecord microphone;

    [Header("Spells")]
    public SpellDefinition[] spellBook;

    [Header("UI (optional)")]
    public Text captionText;
    public Text statusText;

    [Header("Feedback (optional)")]
    public AudioSource sfx;
    public AudioClip castSound;

    private WhisperStream _stream;
    private bool _ready;

    // Phrases we've already acted on in the current utterance, so a spell
    // isn't fired repeatedly as the rolling transcript keeps containing it.
    private readonly HashSet<string> _firedThisUtterance = new HashSet<string>();

    private async void Start()
    {
        // 1) Bias the recognizer toward our spell vocabulary. This is the single
        //    most effective accuracy win for invented/proper-noun spell names.
        whisper.initialPrompt =
            "Spell incantations: " +
            string.Join(", ", spellBook.Select(s => s.spellName)) + ".";

        // 2) Mobile microphone permission (no-op on desktop/editor).
        await RequestMicPermission();

        // 3) Create a microphone-driven stream. This waits for the model to load.
        _stream = await whisper.CreateStream(microphone);
        if (_stream == null)
        {
            SetStatus("Failed to start speech recognition.");
            return;
        }

        // 4) Subscribe. These fire on the Unity main thread (mic-driven stream),
        //    so it's safe to update UI and gameplay directly.
        _stream.OnResultUpdated   += OnLiveTranscript;   // partial, every step
        _stream.OnSegmentFinished += OnSegmentFinished;  // a segment was finalized
        microphone.OnVadChanged   += OnVadChanged;

        // 5) Start listening continuously. StartStream() MUST come before StartRecord().
        _stream.StartStream();
        microphone.StartRecord();

        _ready = true;
        SetStatus("Listening… speak a spell!");
    }

    private void OnDestroy()
    {
        if (_stream != null)
        {
            _stream.OnResultUpdated   -= OnLiveTranscript;
            _stream.OnSegmentFinished -= OnSegmentFinished;
        }
        if (microphone != null)
        {
            microphone.OnVadChanged -= OnVadChanged;
            if (microphone.IsRecording)
                microphone.StopRecord();   // releases the mic device
        }
    }

    // ---- Speech callbacks -------------------------------------------------

    // Cumulative transcript of the whole stream so far — good for live captions
    // and for catching a spell the instant it's spoken.
    private void OnLiveTranscript(string fullTranscript)
    {
        if (captionText) captionText.text = fullTranscript;
        TryCastFrom(fullTranscript);
    }

    // A segment finalized: the player likely paused. Reset per-utterance dedup
    // so the same word can be cast again in the next sentence.
    private void OnSegmentFinished(WhisperResult segment)
    {
        _firedThisUtterance.Clear();
    }

    private void OnVadChanged(bool speaking)
    {
        if (_ready && statusText)
            statusText.text = speaking ? "Listening… (hearing you)" : "Listening…";
    }

    // ---- Spell matching ---------------------------------------------------

    private void TryCastFrom(string transcript)
    {
        var text = Normalize(transcript);

        foreach (var spell in spellBook)
        {
            foreach (var phrase in spell.triggerPhrases)
            {
                var key = Normalize(phrase);
                if (key.Length == 0 || !text.Contains(key))
                    continue;

                // already handled this phrase in the current utterance?
                if (_firedThisUtterance.Contains(key))
                    continue;

                // cooldown check
                if (Time.time - spell.LastCastTime < spell.cooldownSeconds)
                    continue;

                _firedThisUtterance.Add(key);
                spell.LastCastTime = Time.time;
                Cast(spell);
                return; // one spell per recognition pass
            }
        }
    }

    private void Cast(SpellDefinition spell)
    {
        SetStatus($"✨ Casting: {spell.spellName}!");
        if (sfx && castSound) sfx.PlayOneShot(castSound);

        // === Your gameplay hook ===
        // Spawn VFX at the magic stick, deal damage, play animation, etc.
        Debug.Log($"[Spellcast] {spell.spellName}");
        SpellEvents.Raise(spell);
    }

    // ---- Helpers ----------------------------------------------------------

    // Lowercase + strip punctuation so "Fireball!" matches "fireball".
    private static string Normalize(string s)
    {
        if (string.IsNullOrEmpty(s)) return "";
        var chars = s.ToLowerInvariant()
            .Where(c => char.IsLetterOrDigit(c) || char.IsWhiteSpace(c))
            .ToArray();
        return new string(chars).Trim();
    }

    private void SetStatus(string msg)
    {
        if (statusText) statusText.text = msg;
    }

    private static async System.Threading.Tasks.Task RequestMicPermission()
    {
#if UNITY_ANDROID && !UNITY_EDITOR
        if (!UnityEngine.Android.Permission.HasUserAuthorizedPermission(
                UnityEngine.Android.Permission.Microphone))
        {
            UnityEngine.Android.Permission.RequestUserPermission(
                UnityEngine.Android.Permission.Microphone);
            // give the OS dialog time; in production await a proper callback.
            await System.Threading.Tasks.Task.Delay(500);
        }
#elif UNITY_IOS && !UNITY_EDITOR
        Application.RequestUserAuthorization(UserAuthorization.Microphone);
        while (!Application.HasUserAuthorization(UserAuthorization.Microphone))
            await System.Threading.Tasks.Task.Yield();
#else
        await System.Threading.Tasks.Task.CompletedTask;
#endif
    }
}
```

A tiny event bus so other systems (VFX, animation, scoring) react without coupling to the controller:

```csharp
// SpellEvents.cs
using System;
public static class SpellEvents
{
    public static event Action<SpellDefinition> OnSpellCast;
    public static void Raise(SpellDefinition spell) => OnSpellCast?.Invoke(spell);
}
```

### Wire it in the Inspector
On `SpellcastSystem` (the `MagicStickSpellcast` component):
- **Whisper** ← `Whisper` GameObject.
- **Microphone** ← `Microphone` GameObject.
- **Spell Book** ← drag your `SpellDefinition` assets.
- **Caption Text / Status Text** ← the Canvas texts.
- **Sfx / Cast Sound** ← an `AudioSource` + a clip (optional).

Press **Play**, allow mic access, and say "fireball".

---

## 5. Why each design choice

- **Streaming, not record-then-transcribe.** Spells should fire *while* you speak, so we use `WhisperStream` + `OnResultUpdated` instead of `GetTextAsync` on a finished recording. The latency to a cast is roughly `Step Sec` (~1s here) rather than "after you stop talking".
- **`initialPrompt` from the spell list.** Whisper invents plausible spellings for unknown words; seeding it with your spell names biases output toward them. This is the cheapest large accuracy gain — bigger than most parameter tweaks.
- **Per-utterance dedup (`_firedThisUtterance`).** `OnResultUpdated` delivers the *cumulative* transcript, so "fireball" stays in the string for several passes. Without dedup the spell would re-fire every step. We clear it on `OnSegmentFinished` (a natural pause) so the player can cast it again.
- **Cooldowns.** A second guard against rapid re-fires and for game balance.
- **`Normalize`.** Whisper adds punctuation/casing ("Fireball!"); normalizing both sides makes matching robust.
- **Continuous listening.** `vadStop = false` keeps the mic open. VAD still prevents silence from being transcribed, so idle CPU stays low.

---

## 6. Variations

### Push-to-talk (hold the stick / a button to speak)
If you prefer explicit casting (less CPU, fewer false positives), don't auto-start. Toggle recording on input:

```csharp
public void OnCastButtonDown() { _stream.StartStream(); microphone.StartRecord(); }
public void OnCastButtonUp()   { microphone.StopRecord(); } // stream auto-stops via OnRecordStop
```

Here, transcribe on the finalized result via `_stream.OnStreamFinished += final => TryCastFrom(final);` (or use the simpler record-then-`GetTextAsync` flow from [04 — Integration Guide](04-Integration-Guide.md), Recipe 2). For push-to-talk you can also set `MicrophoneRecord.vadStop = true` to auto-end on silence.

### Confidence-gated casting
Require the spell word to be recognized confidently before firing. Enable tokens on the manager (`whisper.enableTokens = true`) and inspect `WhisperResult.Segments[].Tokens[].Prob` in `OnSegmentFinished`, only casting when the matching token's `Prob` exceeds a threshold. See [03 — API Reference](03-API-Reference.md) for token fields and `SubtitlesDemo` for a token-confidence example.

### Multilingual spells
Set `whisper.language = "auto"` and let players cast in their own language; give each `SpellDefinition` trigger phrases per language. See [05 — Languages & Models](05-Languages-and-Models.md).

---

## 7. Performance & platform notes

- **Model size vs. latency.** Streaming runs an inference every `Step Sec`. On CPU, stick to `tiny`/`base`. Enable **Use Gpu** on desktop / Apple Silicon for headroom to use `small`.
- **Tuning responsiveness.** Lower `Step Sec` → faster casts but more CPU; raise it on weak/mobile hardware. Keep `Single Segment` on.
- **One model, one stream.** Inference is serialized per loaded model (a lock); a single `WhisperManager` can't run two streams at once. For multiple simultaneous speakers you'd load multiple models (more memory).
- **Mobile permissions.** Android needs `RECORD_AUDIO` (the script requests it) and **IL2CPP + ARM64**; iOS needs `NSMicrophoneUsageDescription` in Player Settings. See [06 — Platforms & Deployment](06-Platforms-and-Deployment.md).
- **Always release the mic.** `OnDestroy` calls `StopRecord()`; without it the device can stay locked between scenes.
- **Warm up.** The first model load takes time — do it behind a loading screen (`Init On Awake`, or `await whisper.InitModel()`), so the first cast isn't delayed.

---

## 8. Troubleshooting this example

| Symptom | Likely cause / fix |
|--------|--------------------|
| `CS0246: MicrophoneRecord could not be found` | Missing `using Whisper.Utils;` (it's not in `Whisper`). |
| Nothing transcribes | `StartStream()` not called before `StartRecord()`; or mic permission denied; or `frequency` mismatch. |
| Spell fires many times per word | You removed the per-utterance dedup / cooldown — keep `_firedThisUtterance`. |
| Spell never matches | Add recognizer-friendly variants to `triggerPhrases` (e.g. `"fire ball"`), and seed `initialPrompt`; try a bigger model. |
| Laggy casts | Lower `Step Sec`, use a smaller model, or enable GPU. |
| Editor errors about main thread | You're using `WhisperWrapper` events directly — use `WhisperManager`/mic-driven `WhisperStream` (main-thread safe) as shown. |

See [07 — Troubleshooting](07-Troubleshooting.md) for the general list.

Back to the [documentation index](README.md).
