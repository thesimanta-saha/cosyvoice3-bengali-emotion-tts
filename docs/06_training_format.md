# 06 — Training Format Preparation

## Overview

CosyVoice3's training pipeline expects data in **Kaldi-style text files**, which are then converted to **WebDataset Parquet shards** for distributed training. This stage bridges the filtered gold dataset to the format the trainer understands.

---

## Step 1: Kaldi Format (`kaldi_style.py`)

### Three Core Files

**`wav.scp`** — Maps utterance ID to absolute audio path:
```
VID_01_xpYF7pcyDfM_SPEAKER_00_Seg_0004  /home/.../final_data/VID_01_...Seg_0004.wav
VID_01_xpYF7pcyDfM_SPEAKER_01_Seg_0024  /home/.../final_data/VID_01_...Seg_0024.wav
...
```

**`text`** — Maps utterance ID to the full instruction + transcript string.

**`utt2spk`** — Maps utterance ID to speaker ID.

### Instruction Text Construction

### Speaker ID Extraction



## Step 2: Speaker Embeddings

CosyVoice3 requires speaker embeddings for each utterance. These are generated using the CosyVoice tools from the Kaldi files:

**`spk2embedding.pt`** (53 KB) — Average speaker embedding per speaker ID  
**`utt2embedding.pt`** (4.5 MB) — Per-utterance speaker embedding  
**`utt2speech_token.pt`** (1.2 MB) — Pre-computed speech tokens (flow matching targets)

These `.pt` files are computed once and cached, significantly speeding up training by avoiding on-the-fly acoustic feature computation.

---

## Step 3: Parquet Sharding (WebDataset)

For efficient distributed training with PyTorch DDP, the dataset is converted to **WebDataset Parquet tar shards**:

```
data/train/parquet/
├── parquet_000000000.tar   (259 MB)  ← Shard 0
├── parquet_000000001.tar   (262 MB)  ← Shard 1
├── parquet_000000002.tar   (163 MB)  ← Shard 2
├── data.list               ← Shard manifest
├── spk2data.list           ← Speaker → shard mapping
├── utt2data.list           ← Utterance → shard mapping
├── utt2parquet_0.json      ← Detailed index shard 0
├── utt2parquet_1.json      ← Detailed index shard 1
└── utt2parquet_2.json      ← Detailed index shard 2
```

Each tar shard contains:
- Raw audio bytes (WAV)
- Transcript + instruction text
- Pre-computed speaker embedding
- Pre-computed speech tokens

**Total parquet size:** 683 MB (3 shards for 2,619 samples)

---

## Final Data Directory Layout

```
data/train/
├── wav.scp              (320 KB — 2,619 lines)
├── text                 (990 KB — 2,619 lines, with Bengali Unicode)
├── utt2spk              (134 KB — 2,619 lines)
├── spk2embedding.pt     (53 KB)
├── utt2embedding.pt     (4.5 MB)
├── utt2speech_token.pt  (1.2 MB)
└── parquet/             (683 MB total)
    ├── *.tar (3 shards)
    └── *.json (index files)
```

---

## The `<|endofprompt|>` Token

This special token is the key to CosyVoice3's instruction-following architecture. It separates:

- **Everything before** `<|endofprompt|>` → Instruction context (processed by LLM)
- **Everything after** `<|endofprompt|>` → Text to synthesize (speech generation target)


The text to synthesize is passed separately to the generation function.

This design means the LLM learns to condition its hidden states on the instruction before producing the speech token sequence — which is exactly what enables emotion and persona control.