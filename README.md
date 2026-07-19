# ImageKid Core ML models

Converted, on-device Core ML models used by [ImageKid](https://github.com/silvandiepen/ImageKid)
through its `ImageKidInference` package. Generated with the scripts in the main
repo's `tools/coreml-conversion` — this repository just versions the built
`.mlpackage` artifacts so they aren't lost or rebuilt from scratch each time.

Large weight blobs are stored with **Git LFS** (`git lfs install` before cloning).

## Models

| File | Purpose | Input | License | Commercial use |
| --- | --- | --- | --- | --- |
| `models/BiRefNet.mlpackage` | Background removal (best quality) | 1024×1024 RGB | **MIT** (ZhengPeng7/BiRefNet) | ✅ |
| `models/U2Net.mlpackage` | Background removal (lighter) | 320×320 RGB | **Apache-2.0** (xuebinqin/U-2-Net) | ✅ |
| `models/RealESRGAN.mlpackage` | 4× upscale (fixed 256 input) | 256×256 RGB | **BSD-3-Clause** (xinntao/Real-ESRGAN) | ✅ |

All three are permissively licensed and cleared for commercial redistribution.
Retain each upstream project's copyright notice. See the upstream repositories for
full license text.

> **Runtime note:** BiRefNet must run with `MLModelConfiguration.computeUnits =
> .cpuOnly`. On the Neural Engine or GPU it runs in fp16 and overflows on its Swin
> backbone's high-activation regions, returning NaNs. U²-Net and Real-ESRGAN run
> fine on the default (Neural Engine) path.

## Regenerating

See `tools/coreml-conversion` in the ImageKid repo:

```bash
python convert_birefnet.py  --output ./out/BiRefNet.mlpackage
python convert_u2net.py     --output ./out/U2Net.mlpackage
python convert_realesrgan.py --output ./out/RealESRGAN.mlpackage --fixed-size 256
```
