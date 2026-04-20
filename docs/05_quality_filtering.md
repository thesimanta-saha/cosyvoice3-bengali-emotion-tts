# 05 — Quality Filtering & EDA

## Two-Stage Filtering

Quality filtering happens in two stages:

1. **`master_json.py`** — Full EDA across all 4,253 files. Categorizes everything, copies audio into organized folders for visual inspection. Produces 4 JSON reports.
2. **`final_data.py`** — Applies strict rejection thresholds. Produces the final 2,619-sample gold dataset.

---

## Stage 1: EDA and Categorization (`master_json.py`)

### Six Quality Dimensions

Every file is evaluated across 6 quality dimensions using the Gemini-generated scores:

| Dimension | Field | Threshold Used |
|-----------|-------|----------------|
| Audio quality | `audio_quality_score` | High ≥80, Medium 60–79, Low <60 |
| Noise level | `noise_confidence` | Clean=0, Low 1–20, High >20 |
| Overlapping speech | `overlapping_confidence` | Clean=0, Low 1–20, High >20 |
| Interruption | `interruption_score` | Clean=0, Low 1–20, High >20 |
| Multi-speaker | `is_multi_speaker` | true / false |
| Text purity | transcript content | Pure Bengali / digits / English |

### Text Analysis

Two regex checks are run on every transcript:

```
Contains digits:  [0-9০-৯]   ← Both ASCII and Bengali digit characters
Contains English: [a-zA-Z]   ← Any Latin character
```

Files are classified into:
- **Pure Bengali** — no digits, no English characters
- **Digit anomalies** — transcript contains numeral characters
- **English anomalies** — transcript contains Latin characters

### Manual Checking Organization

For human review, audio files are copied into an organized hierarchy:

```
Manual_Checking/
├── perfect_pure_bangla/         ← Clean audio, pure Bengali text
├── male/
│   ├── 1.audio_quality/low/
│   ├── 1.audio_quality/medium/
│   ├── 2.noise_confidence/high/
│   ├── 3.overlapping_confidence/low/
│   ├── 3.overlapping_confidence/high/
│   ├── 4.interruption_score/low/
│   ├── 4.interruption_score/high/
│   ├── 5.multi_speaker/true/
│   └── 6.text_analysis/
│       ├── anomalies_digits/
│       └── anomalies_english/
├── female/
│   └── (same structure)
└── unknown/
    └── (gender-undetected files)
```

This allows randomly sampling any quality category and listening to representative audio without running scripts.

### Reports Generated

| Report | Size | Contents |
|--------|------|----------|
| `Master_EDA_Report.json` | 5.6 MB | Full stats for all 4,252 files |
| `Perfect_Text_Original_Data.json` | 7.6 MB | Pure Bengali subset (complete JSON entries) |
| `Text_Anomalies_Digits_Original.json` | 965 KB | Files containing digits |
| `Text_Anomalies_English_Original.json` | 834 KB | Files containing English |

---

## Stage 2: Strict Filtering (`final_data.py`)

### Rejection Criteria

Files are **rejected** if any of the following are true:

| Criterion | Threshold | Reason |
|-----------|-----------|--------|
| `overlapping_confidence` | > 20 | Mixed voices corrupt speech tokens |
| `interruption_score` | > 20 | Incomplete utterances break prosody learning |
| `is_multi_speaker` | == true | Any secondary voice presence |
| Duration | < 3.0 seconds | Too short for stable attention alignment |
| Duration | > 15.0 seconds | Too long — risks attention drift in LLM |

Note: Digit and English files were already excluded by using `Perfect_Text_Original_Data.json` as the input to this stage.

### Rejection Summary

| Rejection Reason | Count |
|-----------------|-------|
| Overlapping / interruption / multi-speaker | 581 |
| Duration < 3 seconds or > 15 seconds | 288 |
| **Total rejected** | **869** |

Starting from 3,488 pure Bengali files → 2,619 final (81.5% retention after pure-Bengali filter, 61.5% overall retention from raw).

### What Happens to JSON Files

After filtering, all JSON files that pass are **cleaned**:
- The `speech_transcript_tugstugi` key is removed
- Only the Gemini primary transcript is kept
- Files are written to `/final_data/` alongside their WAV files

---

## Final Dataset Composition

### By Gender
| Gender | Count | % |
|--------|-------|---|
| Male | 1,771 | 67.6% |
| Female | 848 | 32.4% |

### Audio Quality of Final Set
| Quality | Count | % |
|---------|-------|---|
| High (≥80) | 2,609 | 99.6% |
| Medium (60–79) | 7 | 0.3% |
| Low (<60) | 3 | 0.1% |

### Interference Scores of Final Set
| Metric | Status |
|--------|--------|
| Overlapping speech | All ≤20 (zero high overlap) |
| Interruptions | All ≤20 (zero high interruption) |
| Multi-speaker | 0 files (all single speaker) |

### Duration Distribution
| Metric | Value |
|--------|-------|
| Minimum | 3.0 seconds |
| Maximum | 15.0 seconds |
| Total duration | **4.12 hours** |
| Average | ~5.7 seconds |
| Average instruction words | 24.5 words |

### Top 10 Emotions in Final Set
| Rank | Emotion | Count |
|------|---------|-------|
| 1 | calm | 716 |
| 2 | serious | 636 |
| 3 | objective | 499 |
| 4 | confident | 220 |
| 5 | curious | 180 |
| 6 | introspective | 142 |
| 7 | insightful | 87 |
| 8 | wise | 74 |
| 9 | determined | 58 |
| 10 | hopeful | 45 |

### Top 10 Personas in Final Set
| Rank | Persona | Count |
|------|---------|-------|
| 1 | content_creator | 929 |
| 2 | intellectual | 701 |
| 3 | host | 454 |
| 4 | journalist | 120 |
| 5 | businessman | 62 |
| 6 | teacher | 55 |
| 7 | entrepreneur | 48 |
| 8 | activist | 39 |
| 9 | philosopher | 34 |
| 10 | doctor | 28 |