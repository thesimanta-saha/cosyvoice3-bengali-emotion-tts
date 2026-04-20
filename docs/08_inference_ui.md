# 08 — Inference UI (Gradio App)

## Overview

![Gradio App](../results/Gradio_App.png)

A Gradio web application provides an interactive evaluation interface for all trained checkpoints. It runs on `port` and requires no command-line arguments.

**Features:**
- Dynamic checkpoint discovery (all `.pt` files in `exp/emotion_sft/`)
- In-place LLM weight swapping (base model is loaded once, only the LLM head is hot-swapped)
- Two evaluation tabs: **Generate** and **A/B Compare**
- Side-by-side emotional vs. neutral output comparison
- Reference audio support for speaker voice cloning
- Speed control (0.5× – 2.0×)

---

## Architecture

The Gradio app uses an important optimization: **the base model is loaded once** and the LLM state dict is snapshotted. When you switch checkpoints, only the LLM weights are swapped in-place — the flow matching decoder and other heavy components are never reloaded. This makes checkpoint switching fast.

```python
# On startup:
BASE_LLM_STATE = {k: v.clone() for k, v in MODEL.model.llm.state_dict().items()}

# On checkpoint switch:
MODEL.model.llm.load_state_dict(new_weights, strict=False)
```

---

## Tab 1: Generate

**Purpose:** Evaluate how well a checkpoint follows an emotion instruction.

### Inputs

| Input | Description |
|-------|-------------|
| Checkpoint | Dropdown — select any `.pt` file or `[Base pretrained]` |
| Text | Bengali text to synthesize |
| Instruction | Emotion/style instruction (editable) |
| Reference audio | 3–10 second audio clip of the target speaker |
| Speed | Slider from 0.5× to 2.0× |

### Outputs

Two audio outputs are generated **back-to-back** from the same checkpoint:

1. **With Emotion Instruction** — Full instruction passed to the LLM
2. **Plain (no instruction)** — Empty instruction suffix (`<|endofprompt|>` only)

This side-by-side comparison reveals how strongly the checkpoint responds to the instruction vs. what it would produce without any style guidance.


## Tab 2: A/B Compare

**Purpose:** Compare two different checkpoints on the same instruction.

### Recommended Comparison

> **Checkpoint A:** `[Base pretrained]`  
> **Checkpoint B:** `epoch_29_whole`

If fine-tuning worked correctly, Checkpoint B should produce clearly more expressive and instruction-aligned speech than Checkpoint A for the same input.

### Inputs

- Checkpoint A (dropdown)
- Checkpoint B (dropdown)
- Text, Instruction, Reference audio, Speed (same as Tab 1)

### Outputs

Two audio players side-by-side — one per checkpoint — with status messages showing which checkpoint and prompt was used.

---

## Checkpoint Sorting

Checkpoints are automatically discovered and sorted in training order:

```
[Base pretrained]     ← Always first
init.pt               ← Pre-training snapshot
epoch_0_step_500.pt
epoch_0_whole.pt
epoch_1_step_1000.pt
...
epoch_29_step_26000.pt
epoch_29_whole.pt     ← Always last (final)
```

---

## CosyVoice3 Inference API

The app calls `MODEL.inference_instruct2()`:

```python
gen = MODEL.inference_instruct2(
    tts_text,          # Bengali text to synthesize
    instruct_text,     # 
    ref_audio,         # filepath to reference WAV (speaker voice)
    stream=False,      # Return all at once
    speed=1.0,         # Speed multiplier
)
```

The generator yields chunks with `{"tts_speech": tensor}` — these are concatenated into a single numpy array and returned as `(sample_rate, audio_array)` for Gradio's audio player.

---

## Sample Audio

| File | Description |
|------|-------------|
| [Reference_Audio.wav](../reference_audio/Reference_Audio.wav) | Reference speaker input for voice cloning |
| [Output_Emotional.wav](../reference_audio/Output_Emotional.wav) | Emotion-controlled output sample |

---

## Running the App

Prerequisites:
- CosyVoice3 installed at `path`
- `Fun-CosyVoice3-0.5B` pretrained model at `path/pretrained_models/`
- Fine-tuned checkpoints in `exp/emotion_sft/`
- Python environment with: `gradio`, `torch`, `numpy`, `cosyvoice`

```bash
python inference/gradio_app.py
# → Loads model (~30 seconds)
# → Discovers 160+ checkpoints
# → Starts at <port>
```