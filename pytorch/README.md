# Raw PyTorch checkpoints

Trained by [imagekid-training](https://github.com/silvandiepen/imagekid-training).
These are **not** Core ML — they are BasicSR checkpoints, loaded through that
repo's `configs/`. The `.mlpackage` files in `../models/` are what the app
consumes on device.

| file | model | notes |
|---|---|---|
| `model_a_cleanup.pth` | Model A — cleanup, scale 1 | Pre-vectorisation cleanup for FekthorKit |
| `model_b_upscale_x4.pth` | Model B — 4x upscale | Graphics-specific; replaces RealESRGAN for non-photo input |
| `model_c_combined.pth` | Model C — cleanup + upscale | A and B fused, fine-tuned end to end |

Scores and known weaknesses: `RESULTS.md` in the training repo. Read it before
shipping — Model A beats every baseline overall but is *not* a clean win on the
logos category specifically.

The icon-generation LM is not here: at ~3GB it belongs in a Release or R2
rather than LFS, where every clone would spend bandwidth quota.

Source commit: `8c33c3947b5c1edffd7e6a3ce4c380b2c293c13a`
Published: 2026-07-27 02:28:31 UTC
