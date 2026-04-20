# 02 — Data Collection

## Source Material

The dataset is built from **14 Bengali YouTube podcast videos**. These were selected based on:

- Long-form conversations (30–90 minutes each)
- Clear speaker separation (typically 2 speakers per video — host + guest)
- High-quality audio (minimal background music, studio-like recording)
- Topic diversity (motivation, journalism, business, philosophy, health, society)
- Variety of speakers — different genders, ages, accents

The videos span a range of personas: motivational content creators, journalists, academics, doctors, businesspeople, and social activists — all in standard Bengali.

## Speaker Diarization & Segmentation

Each YouTube video was processed through a speaker diarization pipeline to:

1. **Detect speaker turns** — identify when one speaker stops and another begins
2. **Segment audio** — split continuous audio into individual utterances per speaker
3. **Organize by speaker** — save segments into `SPEAKER_00/`, `SPEAKER_01/` folders

Output structure per video:
```
VID_XX_<youtube_id>/
├── SPEAKER_00/
│   ├── main_data/
│   │   ├── VID_XX_SPEAKER_00_Seg_0001.wav
│   │   ├── VID_XX_SPEAKER_00_Seg_0002.wav
│   │   └── ...
│   └── extended_data/
│       └── (longer concatenated segments)
├── SPEAKER_01/
│   └── ...
└── metadata.json
```

Each segment is a `.wav` file at **24,000 Hz mono**, preserving the native sample rate of CosyVoice3.

## Dataset Scale

| Metric | Value |
|--------|-------|
| Videos processed | 14 |
| Total unique speakers | ~30 |
| Total WAV segments | 4,253 |
| Average segment duration | ~6.6 seconds |
| Total raw audio | ~7.8 hours |
| Storage (raw) | 1.5 GB |

## Naming Convention

All files follow a consistent naming convention:

```
VID_{video_number}_{youtube_id}_SPEAKER_{speaker_number}_Seg_{segment_number}.wav

Example:
VID_01_xpYF7pcyDfM_SPEAKER_00_Seg_0004.wav
│        │               │          │
│        │               │          └── Segment index (zero-padded)
│        │               └──────────── Speaker index within the video
│        └──────────────────────────── YouTube video ID
└───────────────────────────────────── Video number (01-14)
```

This naming convention is carried through the entire pipeline — from raw WAV to training text file — making every sample fully traceable back to its source.

## Cost Estimation Before Processing

Before running the Gemini API enrichment, a cost estimator calculated:

- Total audio duration across all 4,253 files
- Estimated input tokens (audio: 25 tokens/second)
- Estimated text tokens (prompt overhead per file)
- Projected API cost in USD and BDT

This step ensured budget control before committing to the full API run.

## Key Challenges

**Challenge 1: Background music in podcasts**  
Some videos had background music under the speech. These were handled by Gemini's `noise_confidence` scoring — high-noise segments were flagged and removed in the filtering stage.

**Challenge 2: Overlapping speech**  
In natural podcast conversations, speakers occasionally talk over each other. The `overlapping_confidence` and `interruption_score` metrics (0–100 each) were used to identify and reject these segments.

**Challenge 3: Multiple speakers in one segment**  
Diarization is not perfect — some segments contained brief sounds from the non-dominant speaker (a laugh, a "hm", an agreement sound). Gemini's `is_multi_speaker` flag was used to catch these.

**Challenge 4: Mixed-language speech**  
Bengali podcasters naturally mix in English words and numbers. These segments were identified, analyzed, and ultimately excluded from the training set to keep the text signal clean and purely Bengali.