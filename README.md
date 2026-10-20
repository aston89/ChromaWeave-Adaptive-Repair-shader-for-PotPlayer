# ChromaWeave Adaptive Repair
**A pre-resize chroma-repair shader for PotPlayer, designed to make the next stages of the video pipeline's job easier.**

ChromaWeave works on the decoded image before the renderer enlarges it. It combines luma-guided and chroma-guided edge protection, a multiscale 3×3/5×5 chroma estimate, fine-detail guards and conservative outlier repair. The goal is not to sharpen or upscale the image by itself, but to make colour transitions more coherent before resizing and post-processing.

## Table of contents

- [Why ChromaWeave exists](#why-chromaweave-exists)
- [What it does](#what-it-does)
- [What it does not do](#what-it-does-not-do)
- [Where it fits in the pipeline](#where-it-fits-in-the-pipeline)
- [Installation in PotPlayer](#installation-in-potplayer)
- [How the algorithm works](#how-the-algorithm-works)
- [Tuning parameters](#tuning-parameters)
- [Diagnostic modes](#diagnostic-modes)
- [Benefits Limitations Compatibility](#Benefits-Limitations-Compatibility)
- [Origin](#origin)

## Why ChromaWeave exists
Low-resolution video can contain chroma bleeding, block-like colour transitions and small inconsistencies around saturated edges. These defects may be subtle at the decoded source size, then become more noticeable after a large upscale or additional image processing.

A conventional sharpening pass is not the right first response: sharpening can make an existing chroma defect more visible. A broad blur is not a good answer either, because it can wash out legitimate colour details and make the image feel less defined.

ChromaWeave was developed as a **source-preparation step**. It tries to regularise suspicious chroma variation while protecting the shape of real edges and fine colour features. The intended perceptual result is a small but useful improvement in colour-edge cleanliness and a subtle lift in the image clarity rather than an obvious smoothing or sharpening effect.

## What it does
The shader combines several local decisions:

- **Luma-guided edge protection.** It estimates a local luminance gradient and reduces the influence of samples that are likely to lie across a brightness edge.
- **Chroma-edge detection.** It estimates gradients in the internal Cb/Cr-like components independently of luminance, helping it recognise colour boundaries that have little luma contrast.
- **Multiscale chroma estimation.** It forms a local 3×3 estimate and a wider 5×5 estimate, then adapts their blend to local edges, texture, and fine-detail confidence.
- **Fine chromatic-detail protection.** Directional support from neighbouring samples helps avoid treating a coherent thin colour feature as an isolated defect.
- **Robust outlier repair.** It compares the centre pixel with a neighbourhood estimate that excludes the centre itself. An isolated deviation is considered for repair only when the surrounding samples broadly agree and the edge/detail guards do not indicate a plausible real feature.
- **Channel-aware main correction.** The principal multiscale correction activates Cb and Cr separately, while the outlier-repair stage retains a joint chroma decision.
- **Luma preservation.** After the chroma adjustment, the shader rebuilds RGB using the centre pixel's original luma and retains its alpha value.

These mechanisms are designed to cooperate. No single test can perfectly distinguish every compression artefact from every genuine colour detail, so decisions are based on several signals rather than one threshold alone.

## What it does not do
 is deliberately narrower than a full video restoration or upscaling system.

- It **does not** access the decoder's native YV12/YUV 4:2:0 planes. It receives an RGB texture and derives an internal Y/Cb/Cr-like representation from it.
- It **cannot restore colour information that was discarded** by chroma subsampling, compression or an earlier conversion. It can only make a spatially informed correction to the decoded image.
- It is **not a general-purpose codec deblocker** and does not explicitly know the original codec's block grid.
- It does not perform super-resolution, AI reconstruction, denoising of all image components, deinterlacing, temporal filtering, or sharpening.
- It cannot always distinguish an isolated one-pixel colour detail from an isolated chroma artefact. This ambiguity is inherent when decisions are made from a small spatial neighbourhood without source history.
- It is not guaranteed to improve every source. If the source's chroma is already clean, the safest choice may be to leave the shader disabled for that material.

## Where it fits in the pipeline

 is intended for the **pre-resize** stage:

```text
Decoded source frame
        |
        v
 Adaptive Repair  [Pre-Resize]
        |
        v
Renderer resize / upscale
(e.g. B-Spline 2 pass or another selected resizer)
        |
        v
Optional post-resize edge refinement
(e.g. EdgeWeave_LX)
        |
        v
Display output
```

The division of labour is intentional:

1. ** prepares the source** by reducing selected colour inconsistencies before they are enlarged.
2. **The renderer's resizer creates the output-sized image.**  is not a replacement for Spline, B-Spline, DXVA-HD or another scaler.
3. **A post-resize shader may refine the final contours.** EdgeWeave_LX is one example of a complementary downstream stage; it is not required by  itself.

## Installation in PotPlayer

1. Save `ChromaWeave_Adaptive_Repair.txt` into the custom pixel-shader location used by your PotPlayer installation.
2. Open PotPlayer's video pixel-shader controls and add/select the shader in the **Pre-Resize Pixel Shaders** stage. Menu wording may vary by version or localisation.
3. Keep it out of the post-resize stage. Its sample offsets are intended to operate on the decoded source-size texture.
4. For a baseline comparison, disable other chroma-preparation shaders and keep the renderer's resize filter and post-resize shaders unchanged.
5. Compare the same frames with ChromaWeave enabled and disabled. Start with saturated colour edges, broad flat fills, thin coloured details, and soft gradients.

The shader uses `DEBUG_MODE` as a compile-time setting in the TXT file. PotPlayer does not expose these `#define` values as runtime controls. Change the value manually, save the txt (ctrl-s) and disable/enable the shader to apply the new config (configure a shortcut in F2 menu, easier).

## How the algorithm works

### 1. Internal colour representation
Each sampled RGB value is converted to a compact luma/chroma-like representation. This is an image-space transform; the shader is not operating directly on the original encoded YV12 planes.

### 2. Edge and texture analysis
A Sobel-like local gradient estimates the luminance-edge normal. Separate Cb/Cr gradients estimate chromatic edges, and neighbourhood continuity is used to assess whether a colour edge appears coherent. A local luma-spread estimate helps distinguish relatively flat regions from more textured areas.

### 3. Guided multiscale estimate
A spatially weighted 5×5 neighbourhood produces the wider chroma estimate. A central 3×3 estimate provides a more local alternative. Sample weights are reduced by luma differences and by penalties for crossing detected luma/chroma edges. The shader adapts the blend instead of applying an unconditional blur to every pixel.

### 4. Fine-detail protection and outlier decision
Directional colour support helps protect coherent small features. Separately, the outlier path estimates neighbour consensus with the centre sample excluded. If the neighbours disagree strongly, the area is not treated as a simple isolated outlier. Edge, texture, and detail-confidence terms can further reduce the proposed repair.

### 5. Reconstruction
The main correction uses separate activation for Cb and Cr. The outlier correction remains joint to reduce the chance of over-correcting a small colour anomaly in only one component. The result is converted back to RGB with the centre luma preserved.

## Tuning parameters
All tuning controls are `#define` constants near the beginning of the shader. They are not exposed as PotPlayer sliders in the tested setup.

| Parameter | Default | Purpose |
|---|---:|---|
| `MAX_CHROMA_REPAIR` | `0.58` | Maximum strength of the main chroma correction. |
| `CHROMA_CHANGE_FLOOR` | `0.0040` | Minimum Cb/Cr target difference before the main correction activates. |
| `CHROMA_CHANGE_GAIN` | `24.0` | Controls how quickly activation rises above that floor. |
| `WIDE_ESTIMATE_BLEND` | `0.66` | Baseline contribution of the wider 5×5 estimate before adaptive guards. |
| `LUMA_RANGE_REJECTION` | `4.2` | Reduces contributions from neighbours with different luminance. |
| `LUMA_EDGE_GAIN` | `2.2` | Sensitivity of the luma-edge confidence estimate. |
| `LUMA_CROSS_EDGE_REJECTION` | `2.0` | Penalty for sampling across a luma edge. |
| `CHROMA_EDGE_GAIN` | `4.0` | Sensitivity of chromatic-edge detection. |
| `CHROMA_EDGE_PROTECTION` | `0.82` | Influence of coherent chromatic edges on protection. |
| `CHROMA_CROSS_EDGE_REJECT` | `2.0` | Penalty for sampling across a chromatic edge. |
| `FINE_DETAIL_WIDE_GUARD` | `0.88` | Reduces wide-estimate blending where fine detail is supported. |
| `FINE_DETAIL_MAIN_GUARD` | `0.86` | Reduces the main correction around supported detail. |
| `FINE_DETAIL_OUTLIER_GUARD` | `0.98` | Strong protection against classifying supported details as outliers. |
| `OUTLIER_DELTA_THRESHOLD` | `0.015` | Centre-to-neighbour chroma deviation needed before outlier strength rises. |
| `OUTLIER_DELTA_GAIN` | `22.0` | Controls outlier-magnitude sensitivity. |
| `OUTLIER_SPREAD_LIMIT` | `0.038` | Neighbour spread below which consensus is favoured. |
| `OUTLIER_SPREAD_GAIN` | `28.0` | Controls how quickly the consensus score changes around the limit. |
| `OUTLIER_REPAIR_STRENGTH` | `0.78` | Maximum strength of the isolated-outlier correction. |
| `FINAL_LUMA_EDGE_GUARD` | `0.20` | Additional restraint on main correction near strong luma edges. |
| `FINAL_CHROMA_EDGE_GUARD` | `0.12` | Additional restraint on main correction near strong chroma edges. |
| `LUMA_TEXTURE_GAIN` | `3.2` | Sensitivity of the local luma-texture estimate. |
| `SOURCE_TEXTURE_REJECTION` | `0.48` | Reduces broad regularisation in textured regions. |

**Tuning advice:** start from the supplied values. Change one parameter at a time and compare the same frames. Lowering thresholds makes the shader act on more differences, but can also soften legitimate detail or alter colour transitions. Higher repair strength is not automatically better.

## Diagnostic modes
Set `DEBUG_MODE` manually, reload the shader, and restore `DEBUG_MODE 0` for normal playback. Diagnostic output replaces the video; it is not an overlay.

| `DEBUG_MODE` | View | Interpretation |
|---:|---|---|
| `0` | Normal image | Standard ChromaWeave output. |
| `1` | Solid magenta | Branch/load sanity check; if the movie remains visible, the edited branch may not be active or recompiled. |
| `2` | Grayscale | Fine-detail protection confidence. Brighter areas indicate stronger detail support. |
| `3` | RGB false colour | Red = outlier magnitude; green = neighbour spread/disagreement; blue = final outlier-repair confidence. |
| `4` | Grayscale proxy | Low-cost 3×3 estimate of local chroma-repair activity. This is a diagnostic proxy, not an exact map of the full 5×5 correction. |
| `5` | Grayscale map | Raw centre-to-neighbour chroma deviation before outlier thresholds. Brightness indicates local chroma difference, not necessarily an artefact. |
| `6` | Grayscale map | Requested target chroma difference before final correction gain. |
| `7` | Grayscale proxy | Candidate confidence from a lightweight isolated-outlier classifier. It is not a pixel-exact map of the full repair. |

Modes 4, 5, and 7 deliberately use early, lightweight diagnostic calculations to keep their instruction cost manageable. They help answer different questions and should not be interpreted as interchangeable maps. In particular, raw chroma variation in mode 5 does not mean that the pixel is definitely defective.

## Performance and compatibility
- **Target:** PotPlayer-style HLSL pixel-shader pipeline, with a Shader Model 3.0-era syntax style.
- **Sampling:** the main image path samples a 5×5 source neighbourhood (25 texture fetches per output pixel), plus additional arithmetic for edge guidance, variance estimates, and repair confidence.
- **Placement:** pre-resize, where the shader works on the source-size frame rather than processing the full enlarged output.
- **Expected cost:** depends on resolution, GPU, driver, renderer, and shader compiler. No universal frame-rate guarantee is made.
- **Compatibility:** the shader was tested in the PotPlayer pipeline and reports successful use across multiple films. Other players (mpc-hc, vlc, etc etc) should be compatible, just change the extension fronm "txt" to "hlsl" and it should work even on older gpu's.

## Benefits Limitations Compatibility
Less upscaling (resize) artifacts like banding, blocking etc etc.
Cleaner chroma boundaries and a small increase in perceived image coherence without an obvious smoothing or sharpening effect.
It's especially usefull before a resizer of any kind from dxva-hd processor to a lanczos or spline/bspline interpolation shaders where residual colour irregularities may become more apparent after resizing. 
No ground-truth reconstruction benchmark is supplied; the shader is intended for perceptual source preparation, not pixel-perfect restoration.
Decoded RGB frames do not expose the codec's native chroma samples or block layout.
A spatial filter cannot always distinguish an isolated real colour detail from an isolated chroma defect.
There is no temporal history, motion compensation, or guarantee against every form of flicker.
Stronger correction may improve some sources and harm others; the supplied tuning favours fine-detail preservation rather than maximum block suppression.

## Origin
This project was inspired by the class of issues addressed in the “YV12 chroma upsampling” shader by Kurt Bernhard “Leak” Pruenner. 

