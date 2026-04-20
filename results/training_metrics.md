# Training Metrics

## Final Results

| Metric | Value |
|--------|-------|
| Final Loss | **0.1067** |
| Final Accuracy | **97.23%** |
| Total Steps | 26,032 |
| Total Epochs | 30 |

---

## Checkpoint Inventory

160+ checkpoints were saved during training. The complete list follows the pattern:

```
init.pt                      ← Pre-training snapshot (epoch 0, step 0)
epoch_0_step_500.pt          ← Step 500
epoch_0_whole.pt             ← End of epoch 0
epoch_1_step_1000.pt         ← Step 1000
epoch_1_step_1500.pt         ← Step 1500
epoch_1_whole.pt             ← End of epoch 1
epoch_2_step_2000.pt
epoch_2_step_2500.pt
epoch_2_whole.pt
...
epoch_29_step_25500.pt
epoch_29_step_26000.pt
epoch_29_whole.pt            ← FINAL (step 26032)
```

Each checkpoint is ~1.9 GB (LLM weights only).

---

## Steps Per Epoch

With 2,619 training samples and batch processing:

| Epoch | Approx Steps |
|-------|-------------|
| Each epoch | ~867 steps |
| 30 epochs | 26,032 total |

The `batch_idx: 866` in the final YAML confirms the last batch index was 866 (0-indexed = 867 batches/epoch).

---

## Training Configuration Reference

| Parameter | Value | Impact |
|-----------|-------|--------|
| `lr: 0.0001` | 1e-4 | Conservative, stable |
| `warmup_steps: 100` | ~11.5% of epoch 0 | Prevents early instability |
| `grad_clip: 5` | Norm clipping | Prevents gradient explosion |
| `dtype: fp32` | Full precision | No numerical issues |
| `save_per_step: 500` | ~0.58 epochs | Dense checkpointing for recovery |
| `accum_grad: 1` | No accumulation | Effective batch = 1 sample |

---

## Observations on Convergence

### Loss = 0.1067

The cross-entropy loss on speech token next-token prediction converged to 0.1067 by epoch 29. In context:

- **Pre-training baseline** (on this fine-tuning data): Would be significantly higher since the base model hasn't seen Bengali emotional instructions
- **0.1067**: Indicates the LLM has learned strong associations between instruction tokens and the corresponding speech token distributions
- The loss did not diverge or show instability, which validates the conservative `lr=1e-4` and `fp32` choice

### Accuracy = 97.23%

Token-level prediction accuracy of 97.23% means the model correctly predicts almost all next speech tokens when given the instruction context. This high number at epoch 29 (not epoch 5 or 10) suggests genuine convergence, not just early memorization.

---

## Recommended Checkpoints for Evaluation

| Checkpoint | Step | When to Use |
|------------|------|-------------|
| `[Base pretrained]` | — | Baseline comparison |
| `epoch_5_whole.pt` | ~4,335 | Early-stage SFT — subtle emotion effect |
| `epoch_10_whole.pt` | ~8,670 | Mid-stage — noticeable emotion |
| `epoch_20_whole.pt` | ~17,340 | Later-stage — strong emotion |
| `epoch_29_whole.pt` | 26,032 | **Final model — maximum instruction following** |

The A/B comparison tab in the Gradio UI makes it easy to compare any two of these to observe how the model's behavior evolves over training.

---

## Storage Usage

| Path | Size |
|------|------|
| `exp/emotion_sft/` (all checkpoints) | ~157 GB |
| `data/train/parquet/` (training data) | 683 MB |
| `final_data/` (gold dataset) | 695 MB |
| `dataset_speaker_only/` (raw audio) | 1.5 GB |
| **Total project** | **~160 GB** |

The 157 GB checkpoint directory is the dominant storage cost. For production use, keeping only `epoch_29_whole.pt` (1.9 GB) is sufficient.