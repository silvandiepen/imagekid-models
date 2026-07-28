# Custom model training — postmortem (July 2026)

**Do not ship any of the models described here.** Five were trained on rented
GPUs over two days at a cost of roughly **$38**. All five were verified against
real inputs afterwards and none is good enough to use. This document records
what was built, why it failed, and what would have to change for another
attempt to be worth funding.

The models themselves are archived (see [Where the weights are](#where-the-weights-are))
so the effort is reproducible, not so it is deployable.

---

## TL;DR

| model | intent | verdict |
|---|---|---|
| Model A (cleanup) | pre-vectorisation cleanup for FekthorKit | **destroys text and thin lines** |
| Model A v2 (full-res) | fixes A's architecture | preserves text, but **fades and tints line art** — worse than doing nothing |
| Model B (4x upscale) | graphics-specific upscaler | same detail destruction as A |
| Model C (combined) | A+B fine-tuned end to end | same |
| Icon LM 0.5B / 1.5B | generate SVG icons | **~50% of output is invalid**; a general-purpose LLM does it better |

The single most useful sentence in this document: **test against
`packages/FekthorKit/fixtures/inputs/` before training anything.** Those files
existed throughout and were not used until after all the money was spent. They
would have exposed the fatal problem on day one for free.

---

## What was attempted

Two tracks in [`imagekid-training`](https://github.com/silvandiepen/imagekid-training):

1. **Upscale / restoration** — RezFixer-style two-stage graphics restoration
   (BasicSR / RRDBNet). Model A cleans at original resolution, Model B upscales
   4x, Model C fuses and fine-tunes both.
2. **Icon generation** — a code LM fine-tuned to emit SVG path data directly,
   conditioned on style family, grid, stroke width and concept name.

Training data was **synthetic**: ~42,000 generated images of flat-colour
shapes, wordmarks and brand marks, degraded with a randomised pipeline (JPEG
and WebP recompression, blur, ringing, posterisation, mismatched resampling).

---

## Why the restoration models fail

### 1. The architecture discards the detail before training begins

BasicSR's `RRDBNet` handles `scale=1` by pixel-unshuffling the input **4x**
before the trunk:

```python
elif self.scale == 1:
    feat = pixel_unshuffle(x, scale=4)     # rrdbnet_arch.py
```

A 256px image is therefore processed at 64px internally, and a 1–2px letter
stroke is sub-pixel before the network sees it. No loss weighting can recover
detail the architecture threw away at its first layer.

This was established by experiment. An earlier hypothesis — that
`FlatRegionLoss` was over-weighted — was tested with two matched 8k-iteration
runs and **made no measurable difference** (text error 21.39 vs 21.22).
Replacing the architecture with a full-resolution variant did fix text
survival.

### 2. Every metric used was blind to the failure

SSIM, PSNR, mean absolute error, text-region error, and a purpose-built
detail-retention measure **all pointed the wrong way**. Model A scored 0.9670
SSIM and beat every baseline on the project's own torture set while producing
illegible text.

The reason is simple and applies to any graphics restoration work: these
metrics are pixel-averaged, and on a logo the pixels that matter — glyph
strokes, hairlines — are a few percent of the image. A model can destroy all of
them and barely move the average.

**Look at the output. Treat metrics as a smoke test only.**

### 3. The training data does not resemble real input

This is the root cause, and the most expensive lesson.

Training data was synthetic flat-colour graphics with bold text. Real
FekthorKit inputs are **fine neutral line art on white**. Measured on
`fixtures/inputs/artist-lineart.png`, damaged and then restored:

| | line colour (RGB) | channel spread |
|---|---|---|
| original | (39, 39, 39) | 0.1 — neutral |
| damaged input | (79, 79, 79) | 0.0 — neutral, lighter |
| Model A | (86, 80, 79) | 6.8 — slight tint |
| Model A v2 | (113, 99, 106) | **14.6 — clearly tinted** |

Both models make the lines **lighter than the damage did**, and both introduce
a colour cast where the original is perfectly neutral. On this input class —
a large share of what Fekthor handles — **doing nothing is better than either
model.**

No architecture change fixes a distribution mismatch.

---

## Why the icon models fail

Fine-tuning `Qwen2.5-Coder` (0.5B and 1.5B) on 43,983 SVG examples covering
26,231 concepts produced:

| measure | 0.5B | 1.5B |
|---|---|---|
| renders validly | 53% | 41% |
| stays inside the 24x24 canvas | 25% | 41% |
| not a verbatim copy of training data | 78% | 78% |

Tripling the parameter count did not fix it. Inspecting the output shows the
failure is more fundamental than the numbers suggest:

- **Conditioning is ignored.** Prompts for `house`, `heart`, `star` and `trash`
  all produced circles.
- **The model emits non-SVG tokens mid-path** — observed: `gems`, `editar`,
  `第一部`, `雇主和雇员`. It falls back to its base vocabulary, which means the
  fine-tune barely took.
- **Degenerate repetition** — `M12 19.5C12 19.5 12 19.5…` repeated until the
  token limit.
- **It memorised an artefact of the data pipeline.** The circle it emits
  constantly is the exact string the corpus builder's `<circle>`-to-path
  converter produces, which became the single most common pattern in training.

**A general-purpose LLM asked for an SVG icon produces better results than
this.** That comparison was never run before training, and it should have been
the first thing done — a specialised model has to beat the free general one to
justify existing.

---

## What would have to change

1. **Build the acceptance test first.** `fixtures/inputs/` is the bar. Any run
   that has not been eyeballed against those files has not been evaluated.
2. **Train on data that resembles them.** Line art on white, neutral strokes,
   the actual damage real uploads carry — not only synthetic flat-colour
   graphics.
3. **Judge by eye, on real files.** Keep metrics for regression detection, not
   for deciding whether something works.
4. **Benchmark against doing nothing, and against the classical filter**
   (FekthorKit's Gradient mode already uses Kuwahara in `Preprocess.swift`).
   Model A beat neither on the cases that matter.
5. **For icons, benchmark against a general LLM before training anything.**

It is also worth seriously considering that classical filtering plus a good
vectoriser is the right answer. Nothing in this work demonstrated that a
learned cleanup model is *necessary* — only that a badly matched one is worse
than nothing.

---

## What is reusable

Not everything here is a write-off:

- **The training pipeline works.** Data generation, training, crash-resume via
  `--auto_resume`, and publishing to three destinations. It survived several
  crashes and restarted itself correctly.
- **The icon corpus is real work**: 82,821 icons from 16 libraries, normalised
  to a shared 24x24 grid with the rescaling verified pixel-lossless over 3,000
  samples (arc flags and relative deltas handled correctly — the easy thing to
  get silently wrong).
- **The diagnosis**, so a future attempt does not repeat it.

---

## Where the weights are

Archived for reproducibility. **Not for deployment.**

| location | contents |
|---|---|
| R2 `imagekid-models/training/2026-07-27/` | Models A, B, C, both icon LMs |
| R2 `imagekid-models/training/model_a_v2_2026-07-27/` | Model A v2 |
| GitHub Release `v1` on `imagekid-training` | Models A, B, C, both icon LMs |
| GitHub Release `v2-model-a` | Model A v2 |
| `pytorch/` in this repo (Git LFS) | Models A, B, C, A v2 |

Full experiment log, per-run costs and measurements:
[`RESULTS.md`](https://github.com/silvandiepen/imagekid-training/blob/main/RESULTS.md).

---

## Cost

Roughly **$38** across two days on a rented RTX 4090 at ~$0.70/hr: ~$30 for the
five original models, ~$8 for the v2 investigation and retrain.

The ~$8 spent diagnosing bought real information cheaply — the flat-loss
hypothesis was killed for $1, the architecture cause confirmed for $0.50.

The ~$30 spent training the original five is the part that was avoidable. Ten
minutes with `fixtures/inputs/` beforehand would have shown the training
distribution was wrong before any GPU was rented.
