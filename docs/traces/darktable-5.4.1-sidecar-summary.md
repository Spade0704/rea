# Sidecar summary — darktable 5.4.1 modules for the DarkCockpit AI

Short map for the Python sidecar that writes darktable history. The long evidence note is `docs/traces/darktable-5.4.1-modules.md`. Operation names and `modversion` values below are the XMP `operation` / `modversion` pair. Pack the C struct member names, not the UI labels.

The frontend (`frontend/static/app.js`, blob `b3c9eae0d4c73f646afa3bc467bbd41d718787d5`) currently posts only `edit/exposure` `{ev, black}`, `edit/shadow-lift` `{ev, lo, hi, invert}`, `ai/auto`, `ai/prompt`, and `save-sidecar`. It does not encode filmic, sigmoid, color balance, or denoise structs. Those stay in the sidecar.

## Already implemented

- `exposure` v7, 28 bytes. Matches `exposure.c`. Manual EC is `mode=0`, `exposure` in stops (default 0, range -18..18), `black` default 0 (range -1..1). Leave `compensate_hilite_pres=1`.
- `temperature` v4, 20 bytes. Codec matches the struct. Do not treat `NEUTRAL` (`multipliers 1`, `preset 0`) as the scene-referred default. Preset 0 is as-shot. On a scene-referred image the GUI sets preset 4 (`DT_IOP_TEMP_D65_LATE`). The four multipliers are camera-space, not Kelvin. Clone them from an existing XMP until the camera matrix is available.

## One tone mapper only

New raw images use pipe order v5.0 (`DT_IOP_ORDER_V50`). These four sit in one slot and must not all be enabled:

| Workflow pref `plugins/darkroom/workflow` | Auto-applied module | modversion | Contrast field (static default)                   |
| ----------------------------------------- | ------------------- | ---------- | ------------------------------------------------- |
| `scene-referred (filmic)`                 | `filmicrgb`         | 6          | `contrast` 1.0 (range 0..5)                       |
| `scene-referred (sigmoid)`                | `sigmoid`           | 3          | `middle_grey_contrast` 1.5 (range 0.1..10)        |
| `scene-referred (AgX)`                    | `agx`               | 7          | `curve_contrast_around_pivot` 2.4 (range 0.1..10) |

`filmic` (the older Lab module) is a different operation. Do not write it when the plan says filmic rgb.

`filmicrgb` `reload_defaults` then shifts `black_point_source` / `white_point_source` from exposure bias when the image has a color matrix and the workflow is scene-referred. The static defaults (black -8 EV, white +4 EV, middle gray 18.45) are the struct defaults, not that adjusted preset. Prefer cloning a darktable-authored history step when the AI only nudges contrast.

## Lightroom-shaped edits → which module

| Lightroom control                      | Write this                                                                                                                                                                                    | Do not write                                                                                           |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Exposure                               | `exposure.exposure` (stops)                                                                                                                                                                   | tone-equalizer exposure                                                                                |
| Contrast                               | the one enabled tone mapper's contrast field                                                                                                                                                  | `colorbalancergb.contrast` unless the plan is a small scene-referred contrast (range -1..1, default 0) |
| Highlights / Shadows / Whites / Blacks | `toneequal` v2 bands, in EV, range -2..2, default 0                                                                                                                                           | `filmicrgb` white/black relative exposure (that is the tone-mapper range, not a local band)            |
| Texture / Clarity                      | `diffuse` v2 (`sharpness` default 0, range -1..1)                                                                                                                                             | `sharpen` unless the plan is the Lab unsharp mask                                                      |
| Dehaze                                 | `hazeremoval` v3 `strength` (default 0.2, range -1..1)                                                                                                                                        |                                                                                                        |
| Vibrance                               | `colorbalancergb.vibrance` (default 0, range -1..1)                                                                                                                                           | `vibrance` (Lab, default amount 25) for scene-referred work                                            |
| Saturation                             | `colorbalancergb.saturation_global` (default 0, range -1..1)                                                                                                                                  |                                                                                                        |
| Tone curve                             | `rgbcurve` v1. Identity is 2 nodes per channel, `(0,0)` and `(1,1)`, interpolator 2 (`MONOTONE_HERMITE`)                                                                                      | `tonecurve` (display-referred, after the tone mapper)                                                  |
| HSL                                    | `colorequal` v4. Eight hues (red, orange, yellow, green, cyan, blue, lavender, magenta). Saturation and brightness default 1 (range 0..2). Hue default 0 (range -180..180)                    | `colorzones` for new scene-referred edits                                                              |
| Color grading                          | `colorbalancergb` v5 shadows / midtones / highlights `Y`,`C`,`H`. Luma and chroma default 0. Hue default 0 (range 0..360). `saturation_formula` default 1 (darktable UCS 2022)                | legacy `colorbalance`                                                                                  |
| Sharpen                                | `sharpen` v1 if Lab is intended: `radius` 2 (0..99), `amount` 0.5 (0..2), `threshold` 0.5 (0..100). Scene-referred local contrast is `diffuse`                                                |                                                                                                        |
| Noise reduction                        | `denoiseprofile` v12 on raw, before `colorin`. Do not zero-fill `a[3]`,`b[3]`. `reload_defaults` copies a matched noise profile and sets `a[0]=-1` when it autodetects. Clone an in-app step. | `nlmeans` (UI name "astrophoto denoise") for ordinary color noise                                      |
| Masking                                | Parametric mask inside that module's `blend_params`                                                                                                                                           | Drawn mask geometry until a round-trip exists                                                          |

## Tone equalizer names are not the UI labels

`toneequal` v2. Every band is a float, min -2, max 2, default 0. The C member name is what XMP packs. The UI string is different:

| C member            | UI label        | Band center |
| ------------------- | --------------- | ----------- |
| `noise`             | blacks          | -8 EV       |
| `ultra_deep_blacks` | deep shadows    | -7 EV       |
| `deep_blacks`       | shadows         | -6 EV       |
| `blacks`            | light shadows   | -5 EV       |
| `shadows`           | mid-tones       | -4 EV       |
| `midtones`          | dark highlights | -3 EV       |
| `highlights`        | highlights      | -2 EV       |
| `whites`            | whites          | -1 EV       |
| `speculars`         | speculars       | 0 EV        |

Also pack `blending` 5, `smoothing` 1.414213562, `feathering` 1, `quantization` 0, `contrast_boost` 0, `exposure_boost` 0, `details` 4 (`DT_TONEEQ_EIGF`), `method` 4 (`DT_TONEEQ_NORM_2`, "RGB euclidean norm"), `iterations` 1. `fields.py` still has no array/nested-struct codec; this struct is scalars and two enums, so it fits the current codec. Confirm `sizeof` before trusting a writer.

## Masks

Default blend (`src/develop/blend.c` `_default_blendop_params`): `mask_mode` disabled, blend normal (`DEVELOP_BLEND_NORMAL2` = `0x18`), opacity 100, combine norm-exclusive, `mask_id` 0, blendif off. Parametric (blendif) sliders live in `blendif_parameters` (64 floats, default open ranges `0,0,1,1` per slot) and `blendif_boost_factors`. Scene-referred modules should use blend colorspace `DEVELOP_BLEND_CS_RGB_SCENE`. For Jz/Cz, boost factors are set to -6.64385619 so the GUI shows 0.

A luminance shadow lift is a parametric mask on `exposure` (or `toneequal`), not a new iop. Drawn points in `masks_history` are still a research spike. Author blendif first.

## Pipe position (raw, v5.0 array order)

Relevant sequence, input to output: `temperature` → `rawdenoise` → `demosaic` → `denoiseprofile` → `hazeremoval` → `exposure` → `toneequal` → `colorin` → `diffuse` → `nlmeans` → `sharpen` → `colorequal` → `colorbalancergb` → `rgbcurve` → `filmic` → `sigmoid` → `agx` → `filmicrgb` → later `tonecurve`, `vibrance`, `colorzones`.

The float keys in `v50_order[]` are not the runtime order. `_ioppr_reset_iop_order` overwrites them with 100, 200, 300… in array order. `lut3d` is stored with float 36 but sits after `filmicrgb` in the array, so it runs after the tone mapper.

JPEG/TIFF/PNG order `DT_IOP_ORDER_V50_JPG` moves `denoiseprofile` through `equalizer` to after `colorin`.

## Serializer rule

Mirror each `dt_iop_*_params_t` field-for-field, little-endian, `gboolean` and `enum` as 4 bytes. Assert `sizeof`. Curves (`rgbcurve`, `colorzones`) are arrays: `rgbcurve` is 3 channels × 20 nodes (`MAX_ANCHORS`), default 2 nodes, and `init` sets node 1 to `(1,1)` after introspection zeros node 0. `colorzones` v5 `init` does not leave the curve at zero: 2 nodes, Catmull-Rom (1), y=0.5, and because the default channel is hue the x positions are 0.25 and 0.75. `strength` 0, mode smooth (0), splines version 2.
