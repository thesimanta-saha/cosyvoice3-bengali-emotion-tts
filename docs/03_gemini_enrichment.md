# 03 — Gemini API Enrichment

## Why AI-Generated Metadata?

Traditional TTS dataset creation requires human annotators to:
- Transcribe speech manually
- Label emotions by listening
- Write speech style descriptions
- Score audio quality

This is expensive and slow. Instead, this pipeline uses **Gemini 2.5 Flash** — Google's multimodal AI — to process each audio file and extract all required metadata automatically.

**Model used:** `gemini-2.5-flash`  
**API:** Google GenAI Python SDK (`google-genai`)  
**Processing mode:** Inline audio bytes + structured JSON response

## What Gemini Extracts Per File

For each `.wav` segment, Gemini returns a structured JSON with 4 sections:

### 1. Metadata
| Field | Description |
|-------|-------------|
| `speech_transcript` | Bengali (Bangla) transcription of the audio |
| `bangla_speech_confidence` | Confidence score 0–100 for Bengali accuracy |
| `speech_instruction_text` | 10–30 word natural language description of delivery style |
| `speech_instruction_confidence` | Confidence 0–100 for the instruction accuracy |
| `emotional_intensity_score` | 0–100 score for how emotionally expressive the speech is |

### 2. Emotions and Character
| Field | Description |
|-------|-------------|
| `primary_emotion` | Main emotion from a list of 65 labels |
| `secondary_emotion` | Supporting/secondary emotion |
| `character_persona` | Speaker persona from a list of 37 labels |

### 3. Speaker Profile
| Field | Description |
|-------|-------------|
| `detected_gender` | Male / Female / Multiple |
| `is_multi_speaker` | true / false — even brief secondary sounds count |
| `accent_dialect_info` | Standard / Regional |

### 4. Quality and Interference
| Field | Description |
|-------|-------------|
| `noise_confidence` | 0–100 background noise level |
| `overlapping_confidence` | 0–100 simultaneous speech level |
| `interruption_score` | 0–100 one speaker cutting off another |
| `audio_quality_score` | 0–100 overall audio quality |

## Emotion Labels (65 categories)

The prompt constrains Gemini to select from:

```
adventurous, ambitious, ancient, angry, artistic, authoritative, bold, brave,
calm, charming, cheerful, clever, commanding, compassionate, confident,
conflicted, contempt, courageous, creative, cunning, curious, dark, deceptive,
dedicated, defiant, determined, disciplined, disgusted, empathetic, energetic,
fearful, fearless, happy, heroic, hopeful, humble, imaginative, indifferent,
insightful, intelligent, introspective, joyful, loyal, merciless, mysterious,
noble, objective, optimistic, passionate, patient, proud, relaxed, relentless,
responsible, sad, selfless, serious, shocked, stealthy, surprised, vengeful,
vigilant, wise, fast, slow, loud, soft
```

## Persona Labels (37 categories)

```
host, co-host, actor, actress, politician, sportsman, athlete, journalist,
reporter, doctor, physician, psychiatrist, psychologist, intellectual, scholar,
philosopher, businessman, entrepreneur, activist, social_worker, legal_expert,
lawyer, engineer, scientist, artist, musician, author, writer, filmmaker,
content_creator, influencer, religious_leader, teacher, professor, student, youth
```

## Speech Instruction Prompt Design

The core prompt instructs Gemini to describe speech **as a recording engineer would describe it to a voice actor**:

> *"Describe the tone, rhythm, and natural human delivery style. Focus on nuances like pauses, breathing, and prosody (e.g., 'A middle-aged man speaking with a deep, authoritative tone, featuring clear articulation and deliberate pauses for emphasis')."*

This produces instruction text that directly maps to CosyVoice3's `inference_instruct2()` format.

**Example output instruction:**
```
"A male speaker delivers with a calm, confident tone and steady rhythm,
using measured pauses and clear articulation throughout."
```

## Multi-Speaker Detection Logic

A critical part of the prompt is the strict multi-speaker detection instruction:

> *"Set `is_multi_speaker` to true if ANY second voice is present — even a tiny 'ji', 'hmmm', or laugh."*

This is more strict than typical diarization. The goal is to catch even micro-reactions from a second speaker that could corrupt the training signal by introducing voice-mixed audio.

Three separate signals are used:
- `is_multi_speaker` — boolean flag (catches any second voice)
- `overlapping_confidence` — how simultaneous the overlap is (0=none, 100=fully simultaneous)
- `interruption_score` — one speaker cutting off another mid-sentence

## Processing Details

| Parameter | Value |
|-----------|-------|
| API rate limit handling | 0.5 second sleep between requests |
| Retry logic | 3 attempts with exponential backoff |
| Resume safety | Skips files where `.json` already exists |
| Response format | `application/json` (structured output mode) |
| Audio encoding | Inline base64 bytes, `audio/wav` MIME type |

## Token Usage & Cost

Each audio file incurs:
- **Input tokens:** ~1,200–1,400 (audio at 25 tokens/sec + prompt text)
- **Output tokens:** ~250–350 (structured JSON response)

For 4,253 files, total approximate cost was calculated in advance using a separate cost estimator script before running the full pipeline.

## Output

For each `VID_XX_SPEAKER_YY_Seg_ZZZZ.wav`, a matching JSON sidecar is created:

```
VID_XX_SPEAKER_YY_Seg_ZZZZ.json
```

These JSON files live alongside the WAV files and carry all metadata through every downstream step of the pipeline.