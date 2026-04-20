# 04 — ASR Verification with Tugstugi

## Why a Second Transcription?

Gemini 2.5 Flash is a powerful multimodal model, but it is not a specialized Bengali ASR system. For the transcript field — which becomes the ground truth text in TTS training — a single source of truth is risky. Transcription errors would directly corrupt the audio-text alignment learned by the model.

To address this, a second transcription is generated independently using **bengaliAI/Tugstugi**, a dedicated Bengali ASR model built on Whisper.

**Model:** `bengaliAI/tugstugi_bengaliai-regional-asr_whisper-medium`  
**Source:** HuggingFace — https://huggingface.co/bengaliAI/tugstugi_bengaliai-regional-asr_whisper-medium  
**Architecture:** Whisper-medium fine-tuned specifically on Bengali speech

## What Tugstugi Adds

For each WAV file, Tugstugi generates a second Bengali transcript. This is stored in the JSON under a separate key:

```json
{
  "metadata": {
    "speech_transcript": "...",              ← Gemini transcript (primary)
    "speech_transcript_tugstugi": "..."      ← Tugstugi transcript (verification)
  }
}
```

## Mismatch Detection

In the EDA stage (`master_json.py`), both transcripts are compared after text normalization:

1. **Text cleaning:** Remove punctuation, normalize Unicode (NFC), collapse whitespace
2. **Word-level diff:** Symmetric difference between the two word sets
3. **Classification:**

| Category | Condition |
|----------|-----------|
| `perfect_align` | Zero word differences |
| `minor_mismatch` | 1–3 word differences |
| `major_mismatch` | 4+ word differences |
| `missing_data` | One transcript is empty |

## Dataset-wide Mismatch Statistics

| Match Type | Count | % |
|------------|-------|---|
| Perfect alignment | 2,383 | 56.0% |
| Minor mismatch (1–3 words) | ~1,200 | ~28.2% |
| Major mismatch (4+ words) | ~669 | ~15.7% |

**56% perfect match** between two independently trained AI systems on the same audio is a strong signal of transcript quality for the pure Bengali subset.

## Key Insight: Digits and Bengali Numerals

One of the most revealing findings from the dual-transcription comparison was **digit representation**:

- Gemini would often transcribe: `"২০২৪ সালে"` (using Bengali digit characters)
- Tugstugi would sometimes transcribe: `"দুই হাজার চব্বিশ সালে"` (spelled out in words)
- In other cases, the reverse

This `digit_vs_word_mismatch` flag was tracked separately because it represents a real text normalization ambiguity — not a transcription error. These files were categorized under `Text_Anomalies_Digits_Original.json` for separate handling.

## Processing Details

| Parameter | Value |
|-----------|-------|
| Model loading | GPU (CUDA) if available, fallback to CPU |
| Resume safety | Skips files where `speech_transcript_tugstugi` key already exists in JSON |
| Batch processing | Sequential (one file at a time) |
| In-place update | JSON files updated in-place with the new key |

## Decision: Which Transcript is Used for Training?

**Gemini transcript is used as the primary (and only) training transcript.**

Reasoning:
- Gemini is given an explicit structured prompt with Bengali language instruction
- Gemini's multimodal understanding handles code-switching and context better
- The Tugstugi transcript is used **only for mismatch detection and quality flagging**
- In the `final_data.py` step, the `speech_transcript_tugstugi` key is removed from all JSON files before training

The dual-ASR approach is a **data quality gate**, not a label ensemble. Its job is to surface suspicious files — not to replace or average the transcription.