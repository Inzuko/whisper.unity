# 5 — Languages & Models

This document covers how Whisper handles languages, how to pick the right model, and — importantly — **how to "train" Whisper for a different or better-supported language**. Read the section on training carefully, because the word "train" means something specific here.

---

## How language support actually works

Whisper is a **pretrained** model. whisper.unity does **not** train anything at runtime — it runs inference with model weights you provide. So "supporting a language" comes down to three things:

1. **Using a multilingual model** (not an English-only `*.en` model).
2. **Selecting the language** (or letting Whisper auto-detect it).
3. **Optionally translating** the detected speech into English.

Whisper's multilingual models already understand **~60 languages** out of the box (English, Spanish, German, French, Italian, Portuguese, Dutch, Russian, Chinese, Japanese, Korean, Arabic, Hindi, Turkish, Polish, Ukrainian, and many more). For most games you never need to train anything — you just select a language and ship a model.

### Selecting a language

Set `WhisperManager.language` to an ISO 639-1 code:

```csharp
whisper.language = "de";    // German
whisper.language = "ja";    // Japanese
whisper.language = "auto";  // or "" / null → auto-detect
```

When auto-detecting, read the result back:

```csharp
var res = await whisper.GetTextAsync(clip);
Debug.Log($"Whisper thinks this is: {res.Language}");  // e.g. "fr"
```

### Translating to English

Set `translateToEnglish = true` to get English text regardless of the spoken language:

```csharp
whisper.language = "auto";        // detect any input language
whisper.translateToEnglish = true; // ...but output English
```

> Translation only goes **to English**. To "translate" into another target language, you'd transcribe in the source language and run a separate translation step (Whisper itself only does X→English).

### Enumerating supported languages

```csharp
if (whisper.IsMultilingual())
{
    foreach (var code in WhisperLanguage.GetAllLanguages())
        Debug.Log(code);   // "en", "de", "ru", ...
}

int id = WhisperLanguage.GetLanguageId("german"); // 2  (accepts name or code)
string code = WhisperLanguage.GetLanguageString(id); // "de"
```

`GetAllLanguages()` lists every language whisper.cpp knows — but a specific *model* may handle some better than others, and English-only models support only English. See `Assets/Samples/3 - Languages/LanguagesDemo.cs`.

---

## Choosing a model

Models trade **speed**, **memory**, and **accuracy**. All are in GGML format (`ggml-*.bin`) and go in `Assets/StreamingAssets/Whisper/`. Download from [Hugging Face — ggerganov/whisper.cpp](https://huggingface.co/ggerganov/whisper.cpp).

| Model | Approx. size | Relative speed | Quality | Multilingual? |
|-------|-------------:|:--------------:|:-------:|:-------------:|
| `tiny` / `tiny.en` | ~75 MB | Fastest | Lowest | tiny = yes, tiny.en = English only |
| `base` / `base.en` | ~142 MB | Very fast | Low-mid | base = yes |
| `small` / `small.en` | ~466 MB | Moderate | Good | small = yes |
| `medium` / `medium.en` | ~1.5 GB | Slow | Very good | medium = yes |
| `large-v3` | ~3 GB | Slowest | Best | yes |
| `large-v3-turbo` | ~1.6 GB | Fast-ish | Near-large | yes |

Guidance:
- **Real-time / streaming or mobile:** `tiny` or `base` (optionally `*.en` for English-only games — they're smaller and more accurate for English).
- **Offline batch transcription (subtitles, etc.):** `small` or `medium` for much better accuracy.
- **Highest quality, desktop only:** `large-v3` / `large-v3-turbo` (heavy RAM/VRAM).

This repo ships **`ggml-tiny.bin`** (multilingual) so it runs anywhere immediately.

### Quantized models (smaller downloads)

whisper.cpp supports quantized weights (e.g. `ggml-base-q5_0.bin`, `ggml-small-q5_1.bin`). They're substantially smaller with a small accuracy cost — useful for shipping on mobile or reducing download size. As long as the file is GGML format compatible with **whisper.cpp v1.7.5**, it loads like any other model.

> **Compatibility rule:** the model must be in the GGML format expected by whisper.cpp v1.7.5. Models converted for very different whisper.cpp versions, or non-GGML formats (PyTorch `.pt`, Hugging Face `safetensors`, CoreML/ONNX), will **not** load directly — they must be converted (see below).

---

## "Training" Whisper for a different language

There are three escalating levels. Pick the lowest one that solves your problem — full training is rarely necessary.

### Level 1 — Just select the language (no training)

If your language is among Whisper's ~60 supported languages, you're done: set `whisper.language` and ship a multilingual model. **This handles the vast majority of cases.** Try this first.

### Level 2 — Bias the model with prompts and a better model size (no training)

If recognition is *mostly* right but struggles with specific words (game terms, names, jargon), you don't need training — you need **prompting** and possibly a **bigger model**:

```csharp
// Bias toward your game's vocabulary and spelling/style for the language.
whisper.initialPrompt = "Spielbegriffe: Drache, Zauberspruch, Heiltrank, Festung.";
whisper.language = "de";
```

- `initialPrompt` is prepended as tokens and strongly biases recognition toward those words and writing style (including accents/diacritics and casing).
- Moving from `tiny` → `small`/`medium` often improves a struggling language dramatically — frequently more cost-effective than fine-tuning.

### Level 3 — Fine-tune Whisper, then convert to GGML (actual training)

Only needed when: the language **isn't** in Whisper's set, you need a niche dialect/accent, or you require domain-specific accuracy that prompting can't reach. **Training happens entirely outside Unity** (in Python, on a machine with a GPU). whisper.unity then loads the converted result.

The end-to-end pipeline:

```
 1. Collect a labeled dataset       (audio clips + correct transcripts)
        │
 2. Fine-tune Whisper in Python     (Hugging Face Transformers / OpenAI Whisper)
        │   → produces PyTorch / safetensors weights
        │
 3. Convert weights to GGML         (whisper.cpp conversion scripts)
        │   → produces ggml-<yourmodel>.bin
        │
 4. (optional) Quantize             (whisper.cpp quantize tool)
        │
 5. Drop ggml-<yourmodel>.bin into  Assets/StreamingAssets/Whisper/
        │
 6. Point WhisperManager.modelPath  at it and ship
```

#### Step 1 — Dataset

You need many hours of audio paired with accurate transcripts in your target language. Common sources:
- [Mozilla Common Voice](https://commonvoice.mozilla.org/) (many languages, permissively licensed).
- [FLEURS](https://huggingface.co/datasets/google/fleurs), VoxPopuli, or your own recorded/annotated data.

Audio should be resampled to **16 kHz mono** (the format Whisper uses).

#### Step 2 — Fine-tune (Python, outside Unity)

The most-traveled path is Hugging Face Transformers. The canonical walkthrough is the official guide:

- **Hugging Face: "Fine-Tune Whisper For Multilingual ASR"** — https://huggingface.co/blog/fine-tune-whisper

In outline:
1. `pip install transformers datasets accelerate evaluate jiwer`.
2. Load a base checkpoint (`openai/whisper-small`, etc.) with `WhisperForConditionalGeneration`, plus its `WhisperFeatureExtractor`, `WhisperTokenizer`, `WhisperProcessor`.
3. Preprocess: extract log-Mel features from audio, tokenize transcripts.
4. Train with `Seq2SeqTrainer` against your dataset (set the language/task in the processor).
5. Evaluate with Word Error Rate (WER) and save the fine-tuned checkpoint.

Start from `small` or `medium` for serious quality; `tiny`/`base` train faster but cap out lower.

#### Step 3 — Convert to GGML (required for whisper.unity)

whisper.unity only loads GGML `ggml-*.bin` files, so convert your fine-tuned Hugging Face/PyTorch checkpoint using **whisper.cpp's** converter:

```bash
# in a clone of whisper.cpp (use a version compatible with v1.7.5)
python models/convert-h5-to-ggml.py  /path/to/your-finetuned-model  /path/to/whisper  ./out
# → produces ggml-model.bin
```

whisper.cpp ships a few conversion scripts depending on your source format:
- `models/convert-h5-to-ggml.py` — from a Hugging Face Transformers checkpoint.
- `models/convert-pt-to-ggml.py` — from an OpenAI `whisper` (`.pt`) checkpoint.

See the [whisper.cpp models README](https://github.com/ggerganov/whisper.cpp/tree/master/models) for current script names and usage.

> **Match the whisper.cpp version.** This package targets **v1.7.5**. Convert with a whisper.cpp version whose GGML format that release accepts, or the model won't load. If you also rebuild the native libraries you can move to a newer whisper.cpp version (see [06 — Platforms & Deployment](06-Platforms-and-Deployment.md)).

#### Step 4 — (optional) Quantize for size

```bash
# inside a built whisper.cpp
./quantize ggml-model.bin ggml-model-q5_0.bin q5_0
```

This shrinks the file for distribution at a small accuracy cost — valuable on mobile.

#### Step 5–6 — Use it in Unity

1. Put `ggml-model.bin` (or the quantized version) in `Assets/StreamingAssets/Whisper/`.
2. Set `WhisperManager.ModelPath = "Whisper/ggml-model.bin"`.
3. Set `whisper.language` to your target language code (your fine-tune was trained for it).
4. Verify with `whisper.IsMultilingual()` and a test clip.

### Which level do I need?

| Situation | Do this |
|-----------|---------|
| Language is one of Whisper's ~60 | **Level 1** — set `language`. |
| Right language, struggles on game terms/names | **Level 2** — `initialPrompt` + bigger model. |
| Unsupported language / niche dialect / domain accuracy | **Level 3** — fine-tune in Python, convert to GGML. |

Most teams never go past Level 2.

Next: [06 — Platforms & Deployment](06-Platforms-and-Deployment.md).
