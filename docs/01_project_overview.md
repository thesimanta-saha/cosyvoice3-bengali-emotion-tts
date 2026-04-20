# 01 — Project Overview

## Why Bengali TTS?

Bengali (Bangla) is spoken by over 230 million people worldwide — the 5th most spoken language on Earth. Yet high-quality, expressive Bengali TTS systems barely exist in open research. Most available tools produce robotic, monotone output with no emotional nuance. There is no publicly available Bengali TTS dataset with emotion labels, persona annotations, or speech instruction metadata.

This project was started to fill that gap — at least partially — by building a real, trainable pipeline from scratch, using only publicly available tools and personal engineering effort.

## The Goal

Fine-tune **CosyVoice3** — a state-of-the-art multilingual TTS model from Alibaba's FunAudioLLM team — to:

1. Synthesize natural Bengali speech
2. Follow **emotional instructions** in natural language (e.g., *"speak with calm confidence"*, *"speak like a journalist"*)
3. Support **speaker voice cloning** (reference audio)
4. Distinguish between 43 distinct emotional styles and 37 personas

## Why CosyVoice3?

CosyVoice3 (specifically `Fun-CosyVoice3-0.5B`) was chosen because:

- It natively supports **instruction-controlled speech synthesis** via `inference_instruct2()`
- It uses a language model (LLM) head that can be fine-tuned with standard SFT
- Its `<|endofprompt|>` special token creates a clean boundary between instruction and text
- The 0.5B parameter size is trainable on a single GPU without DeepSpeed
- The FunAudioLLM team actively maintains the repo with Bengali/multilingual support potential

**Official repo:** https://github.com/FunAudioLLM/CosyVoice  
**Model card:** https://huggingface.co/FunAudioLLM/CosyVoice3-0.5B

## What Makes This Different

Most TTS fine-tuning projects:
- Use studio-recorded clean speech data
- Operate in English or Chinese
- Have human-annotated metadata

This project uses:
- **Podcast audio** — real conversational Bengali speech from YouTube
- **AI-generated metadata** — Gemini 2.5 Flash for emotion/persona/quality labeling
- **Dual ASR verification** — cross-checking with Tugstugi Bengali ASR
- **Fully automated quality filtering** — 6-dimensional quality scoring without manual labeling

The entire pipeline, from raw YouTube audio to trained model, was built and completed in approximately **5 days**.

## Project Timeline

| Date | Milestone |
|------|-----------|
| Day 1 | Identified CosyVoice3, set up environment, selected 14 YouTube podcasts |
| Day 2 | Built `pipeline.py` (Gemini API enrichment), ran on all 4,253 segments |
| Day 3 | Built `tugstugi.py` (ASR verification), ran `master_json.py` (EDA) |
| Day 4 | Built `final_data.py` (filtering), `kaldi_style.py` (format conversion), generated parquet shards |
| Day 5 | Trained 30 epochs, built Gradio inference UI, evaluated checkpoints |

## Key Technical Decisions

| Decision | Reasoning |
|----------|-----------|
| Fine-tune LLM only (not full model) | Preserves acoustic quality of pretrained Flow Matching decoder; LLM is the instruction-understanding component |
| fp32 training (no AMP) | Stability over speed for small dataset; avoids NaN loss in early epochs |
| 3–15s duration filter | Optimal range for TTS training — short enough for stable attention, long enough for prosody learning |
| Pure Bengali only (no digits/English) | Avoids text normalization complexity; ensures clean training signal |
| Constant LR with warmup | Simple, reliable for small datasets — cosine annealing risks under-fitting at small scale |
| Dual ASR transcription | Gemini is primary but expensive; Tugstugi catches hallucinations and transcript errors |