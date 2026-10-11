# darktable 5.4.1 module trace (DarkCockpit)

Source observation of the modules that correspond to Lightroom Basic, Tone Curve, HSL, Color Grading, Detail, and Masking. The sidecar-facing summary is `docs/traces/darktable-5.4.1-sidecar-summary.md`.

## Provenance

| Fact                                         | Value                                                                                                                                                                         |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Consumer tree                                | `Spade0704/DarkCockpit`, path `darktable-5.4.1/`                                                                                                                              |
| Content commit GitHub served for these paths | `87e7bad212d7a594e48eeb7ec6872fe27cf043c2`                                                                                                                                    |
| Upstream tag used to read the sources        | `darktable-org/darktable` `release-5.4.1`, commit `c3f96cadf0ab739965a55f692a061070e19604fd`                                                                                  |
| Identity check                               | Git blob SHA-1 of the DarkCockpit file equals the upstream blob for every module file cited below, plus `src/common/iop_order.c` (`2e4e9617522af9b986cdd703ec2553855e231276`) |

Matched blobs: `exposure.c` `2b23da4d…`, `filmicrgb.c` `e8d2d75d…`, `sigmoid.c` `cbd96d77…`, `agx.c` `1c1229b7…`, `colorbalancergb.c` `13cf1f84…`, `toneequal.c` `81a29351…`, `rgbcurve.c` `a976f5fe…`, `colorzones.c` `e79d373e…`, `colorequal.c` `d68a4f88…`, `sharpen.c` `544909fc…`, `diffuse.c` `b2a49e6e…`, `denoiseprofile.c` `1f57e362…`.

DarkCockpit's README says `darktable-5.4.1/` stays pristine until a C phase forks it. The matched blobs agree for these files. This note does not claim the rest of the tree was compared.

License of those module sources: GPL-3.0-or-later (file headers). Analysis was reading that source. No proprietary binary was opened.

## Method

Coordinate: DarkCockpit layout (`README.md`, `docs/architecture-scoping.md`, `sidecar/serialize/`). The AI MVP writes XMP/styles; it does not recompile darktable.

Verify: blob identity against upstream `release-5.4.1` before treating a local sparse checkout as DarkCockpit's tree.

What was read: `src/common/iop_order.h`, `src/common/iop_order.c` (`v50_order`, `v50_jpg_order`, `_table_to_list`, `_ioppr_reset_iop_order`), `src/develop/blend.h`, `src/develop/blend.c` (`_default_blendop_params`), and the `dt_iop_*_params_t` structs plus `init` / `reload_defaults` / `init_presets` in the module files.

What REA ran:

- `rea analyze-javascript-application` on the in-repo synthetic fixture `website/public/examples/notes-electron` (allowed; source-owned). Evidence id `ev_f1297b4253e8b5c5f30d9fccf49f120badf027da4ad1c125e3b80946459aa074`. That proves the built CLI can emit a JavaScript application graph. It is not a darktable graph.
- `rea analyze-javascript-application` on a local excerpt of DarkCockpit `frontend/static/app.js`. One file parsed, 0 parse failures, 379 AST nodes, 0 module findings. The excerpt did not contain the iop structs. The full frontend source was read through the GitHub API (blob `b3c9eae0d4c73f646afa3bc467bbd41d718787d5`). It posts exposure, a shadow-lift mask, and AI routes. It does not define param structs.

What REA did not run: native call graphs and runtime traces. `rea doctor` reported Hopper missing, `GHIDRA_INSTALL_DIR` unset, and IDA MCP unconfigured. darktable was not compiled or executed in this environment. No pixel-pipe timings or live parameter dumps were taken. Claims below are source observations unless marked as a derivation.

## Pipe order

`src/common/iop_order.h`:

- `DT_IOP_ORDER_V50 = 4` is the default for raw (`DT_DEFAULT_IOP_ORDER_RAW`).
- `DT_IOP_ORDER_V50_JPG = 5` is the default for JPEG/TIFF/PNG.

`_table_to_list` copies `v50_order[]` in source order. `_ioppr_reset_iop_order` then writes integer positions 100, 200, 300… in that same order, overwriting the float stored in the union. Sorting with `dt_sort_iop_list_by_order` therefore follows array order. The float literals are leftover keys. `lut3d` is initialized with 36.0 but appears after `filmicrgb` in the array, so a fresh v5.0 pipe runs it after the tone mapper.

### Raw v5.0, modules that matter here

Input toward output, array order from `v50_order[]`:

1. `temperature` (white balance; before demosaic)
2. `rawdenoise`
3. `demosaic`
4. `denoiseprofile` (profiled denoise on linear raw RGB)
5. `hazeremoval`
6. `exposure`
7. `toneequal` (comment: last module that needs an enlarged `roi_in`)
8. `colorin`
9. `diffuse` (same 28.5 slot as `channelmixerrgb`, `blurs`, `primaries`; array order among them is channelmixerrgb, diffuse, censorize, negadoctor, blurs, primaries)
10. `nlmeans` (UI name "astrophoto denoise")
11. `sharpen` (Lab; comment in the order table calls it weaker than `atrous`)
12. `colorequal`
13. `colorbalancergb`
14. `rgbcurve`
15. `filmic`, then `sigmoid`, then `agx`, then `filmicrgb`
16. After the tone mapper: `tonecurve`, then later `vibrance`, then `colorzones`

`mask_manager` sits just after `exposure`. It is the mask bookkeeping module, not a tone control.

### JPEG v5.0 difference

`v50_jpg_order[]` places `colorin` first among the formerly linear modules, then repeats `denoiseprofile` through `equalizer` after it. `denoiseprofile` on a JPEG therefore runs after the input color profile. The tone-mapper sequence in that array is still `filmic`, `sigmoid`, `agx`, `filmicrgb`.

### Which tone mapper is on

These are alternative scene-to-display curves. `init_presets` auto-applies exactly one, from `plugins/darkroom/workflow`:

| Preference string          | Module      | Evidence                     |
| -------------------------- | ----------- | ---------------------------- |
| `scene-referred (filmic)`  | `filmicrgb` | `filmicrgb.c` `init_presets` |
| `scene-referred (sigmoid)` | `sigmoid`   | `sigmoid.c` `init_presets`   |
| `scene-referred (AgX)`     | `agx`       | `agx.c` `init_presets`       |

`filmicrgb` `reload_defaults` sets `default_enabled` to false, then, when the image supports a matrix correction and the workflow is scene-referred, adds `0.5 * exposure` to `black_point_source` and `0.8 * exposure` to `white_point_source`, with `exposure = 0.7f - exposure_bias`. That adjustment is a derivation from the function, not a measurement on a sample raw.

## Lightroom panel → darktable module

| Lightroom                                   | Scene-referred module               | Why                                                                                                                                                                                                                  |
| ------------------------------------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Basic → White Balance                       | `temperature` v4                    | Raw channel multipliers. Not a Kelvin float in the struct.                                                                                                                                                           |
| Basic → Exposure                            | `exposure` v7 `exposure`            | Stops. `black` is black-level correction, not the Blacks slider.                                                                                                                                                     |
| Basic → Contrast                            | Enabled tone mapper                 | `filmicrgb.contrast` default 1; `sigmoid.middle_grey_contrast` default 1.5; `agx.curve_contrast_around_pivot` default 2.4. `colorbalancergb.contrast` is a smaller scene-referred contrast (default 0, range -1..1). |
| Basic → Highlights, Shadows, Whites, Blacks | `toneequal` v2                      | Nine EV bands, -2..2, default 0. See the rename table below.                                                                                                                                                         |
| Basic → Texture, Clarity                    | `diffuse` v2                        | `sharpness` default 0, range -1..1. `sharpen` is an older Lab unsharp mask and runs earlier, in Lab, which the order comment discourages for this job.                                                               |
| Basic → Dehaze                              | `hazeremoval` v3                    | `strength` default 0.2, range -1..1. `distance` default 0.2.                                                                                                                                                         |
| Basic → Vibrance                            | `colorbalancergb.vibrance`          | Default 0, range -1..1. The Lab `vibrance` module (default amount 25, range 0..100) sits after the tone mapper.                                                                                                      |
| Basic → Saturation                          | `colorbalancergb.saturation_global` | Default 0, range -1..1. Per-hue saturation is `colorequal`.                                                                                                                                                          |
| Tone Curve                                  | `rgbcurve` v1                       | Identity curve in RGB. `tonecurve` is display-referred and later.                                                                                                                                                    |
| HSL                                         | `colorequal` v4                     | Eight hue nodes, scene-referred, before the tone mapper. `colorzones` is Lab and after the tone mapper.                                                                                                              |
| Color Grading                               | `colorbalancergb` v5                | Shadows / midtones / highlights luminance, chroma, and hue, plus global.                                                                                                                                             |
| Detail → Sharpening                         | `sharpen` v1 or `diffuse`           | Lab unsharp mask vs scene-referred diffusion.                                                                                                                                                                        |
| Detail → Noise Reduction                    | `denoiseprofile` v12                | Before `colorin` on raw. Profile coefficients come from the camera match, not from zeros.                                                                                                                            |
| Masking                                     | `blend_params` on the module        | Parametric blendif is inside the blend struct. Drawn masks are separate `masks_history` rows.                                                                                                                        |

`basicadj` v2 exists (exposure, contrast, brightness, saturation, vibrance in one struct) and the order comment says its use should be discouraged. The sidecar should not target it for new edits.

## Module contracts

Ranges and defaults are the `$MIN` / `$MAX` / `$DEFAULT` annotations on the param struct, unless a later sentence cites `init` or `reload_defaults`. Enum integers are the C ordinals. `gboolean` and `enum` are 4-byte little-endian integers in the XMP hex the sidecar already uses for exposure.

### exposure — v7 — group basic|technical

`dt_iop_exposure_params_t` (sidecar already asserts 28 bytes):

| Member                     | Default  | Range                           | Role                         |
| -------------------------- | -------- | ------------------------------- | ---------------------------- |
| `mode`                     | 0 manual | 0 manual, 1 automatic deflicker |                              |
| `black`                    | 0        | -1..1                           | Black level correction       |
| `exposure`                 | 0        | -18..18                         | Exposure compensation, stops |
| `deflicker_percentile`     | 50       | 0..100                          |                              |
| `deflicker_target_level`   | -4       | -18..18                         |                              |
| `compensate_exposure_bias` | 0        |                                 |                              |
| `compensate_hilite_pres`   | 1        |                                 |                              |

`sidecar/serialize/exposure.py` matches this struct, including `compensate_hilite_pres` default true.

### temperature — v4 — group basic|grading

`float red, green, blue, various` each range 0..8, no `$DEFAULT`. `int preset`. Size 20 matches `sidecar/serialize/temperature.py`.

Preset ordinals in `temperature.c`: 0 as-shot, 1 spot, 2 user, 3 D65, 4 D65 late. `gui_update` sets the default preset to 4 when the workflow is scene-referred, else 0. `default_enabled` is true for raw. The multipliers are filled from the camera, not from 1.0. The sidecar's `NEUTRAL` dict is an identity the Python file chose for cloning, not the scene-referred GUI default.

### filmicrgb — v6 — group tone|technical

Static defaults (abridged to the fields a Lightroom-shaped plan touches):

| Member                            | Default    | Range     | UI                           |
| --------------------------------- | ---------- | --------- | ---------------------------- |
| `grey_point_source`               | 18.45      | 0..100    | middle gray luminance        |
| `black_point_source`              | -8         | -16..-0.1 | black relative exposure      |
| `white_point_source`              | 4          | 0.1..16   | white relative exposure      |
| `grey_point_target`               | 18.45      | 1..50     | target middle gray           |
| `black_point_target`              | 0.01517634 | 0..20     | target black luminance       |
| `white_point_target`              | 100        | 0..1600   | target white luminance       |
| `output_power`                    | 4          | 1..10     | hardness                     |
| `latitude`                        | 0.01       | 0.01..99  |                              |
| `contrast`                        | 1          | 0..5      |                              |
| `saturation`                      | 0          | -200..200 | extreme luminance saturation |
| `balance`                         | 0          | -50..50   | shadows ↔ highlights balance |
| `preserve_color`                  | 3          |           | RGB power norm               |
| `version` (color science)         | 4          |           | v7 (2023)                    |
| `auto_hardness`                   | true       |           |                              |
| `custom_grey`                     | false      |           |                              |
| `shadows` / `highlights` curve    | 0          |           | hard (poly 4)                |
| `spline_version`                  | 2          |           | v3 (2021)                    |
| `noise_level`                     | 0.2        | 0..6      |                              |
| `noise_distribution`              | 1          |           | gaussian                     |
| `enable_highlight_reconstruction` | false      |           |                              |

Color-science ordinals: 0 v3 (2019) … 4 v7 (2023). Preserve-color ordinals: 0 no, 1 max RGB, 2 luminance Y, 3 RGB power norm, 4 euclidean legacy, 5 euclidean. Curve: 0 hard, 1 soft, 2 safe.

### sigmoid — v3 — group tone|technical

| Member                    | Default | Range     | UI                                                              |
| ------------------------- | ------- | --------- | --------------------------------------------------------------- |
| `middle_grey_contrast`    | 1.5     | 0.1..10   | contrast                                                        |
| `contrast_skewness`       | 0       | -1..1     | skew                                                            |
| `display_white_target`    | 100     | 20..1600  | target white                                                    |
| `display_black_target`    | 0.0152  | 0..15     | target black                                                    |
| `color_processing`        | 0       |           | per channel (1 = RGB ratio)                                     |
| `hue_preservation`        | 100     | 0..100    |                                                                 |
| `red/green/blue_inset`    | 0       | 0..0.99   | attenuation                                                     |
| `red/green/blue_rotation` | 0       | -0.4..0.4 |                                                                 |
| `purity`                  | 0       | 0..1      | recover purity                                                  |
| `base_primaries`          | 0       |           | working profile (1 Rec.2020, 2 Display P3, 3 Adobe RGB, 4 sRGB) |

The "Adobe RGB" enumerator is a primaries preset inside an open-source module. It is not a reason to analyze Adobe software.

### agx — v7 — group tone|technical

Included because v5.0 places it in the same tone-mapper slot. A plan that enables filmic or sigmoid should leave `agx` off.

Look: `look_lift` 0, `look_slope` 1, `look_brightness` 1, `look_saturation` 1, `look_original_hue_mix_ratio` 0. Range: black relative EV -10 (range -20..-0.1), white relative EV 6.5 (0.1..20), dynamic range scaling 0.1. Curve: pivot x 0.606060606061, pivot y 0.18, contrast around pivot 2.4, toe/shoulder power 1.5, gamma 2.2, `auto_gamma` false, target black 0, target white 1. `base_primaries` default 2 (Rec.2020). Primary insets and rotations default 0. `master_outset_ratio` and `master_unrotation_ratio` default 1.

### colorbalancergb — v5 — group color|grading

4×4 color grading plus the 4.x/5.x extras. All Y/C defaults are 0. Hue (`*_H`) is 0..360, default 0. `hue_angle` is -180..180, default 0.

| Member                                                                                    | Default | Range   | UI                                       |
| ----------------------------------------------------------------------------------------- | ------- | ------- | ---------------------------------------- |
| `shadows_weight`, `highlights_weight`                                                     | 1       | 0..3    | fall-off                                 |
| `white_fulcrum`                                                                           | 0       | -16..16 |                                          |
| `chroma_shadows`, `chroma_highlights`, `chroma_global`, `chroma_midtones`                 | 0       | -1..1   |                                          |
| `saturation_global`, `saturation_highlights`, `saturation_midtones`, `saturation_shadows` | 0       | -1..1   |                                          |
| `brilliance_global`, `brilliance_highlights`, `brilliance_midtones`, `brilliance_shadows` | 0       | -1..1   |                                          |
| `mask_grey_fulcrum`                                                                       | 0.1845  | 0..1    | mask middle-gray fulcrum                 |
| `vibrance`                                                                                | 0       | -1..1   | global vibrance                          |
| `grey_fulcrum`                                                                            | 0.1845  | 0..1    | contrast gray fulcrum                    |
| `contrast`                                                                                | 0       | -1..1   |                                          |
| `saturation_formula`                                                                      | 1       |         | darktable UCS (2022). 0 is JzAzBz (2021) |

Shadows, midtones, highlights, and global each have `_Y`, `_C`, `_H`.

### toneequal — v2 — group basic|grading

Band centers from `centers_params[]` in `toneequal.c`: -8 through 0 EV. UI labels do not match the member names:

| C member            | UI              | Center |
| ------------------- | --------------- | ------ |
| `noise`             | blacks          | -8     |
| `ultra_deep_blacks` | deep shadows    | -7     |
| `deep_blacks`       | shadows         | -6     |
| `blacks`            | light shadows   | -5     |
| `shadows`           | mid-tones       | -4     |
| `midtones`          | dark highlights | -3     |
| `highlights`        | highlights      | -2     |
| `whites`            | whites          | -1     |
| `speculars`         | speculars       | 0      |

Each band: min -2, max 2, default 0.

Other members: `blending` 5 (0.01..100, "smoothing diameter"), `smoothing` 1.414213562 (no range annotation; comment says `sqrtf(2)`), `feathering` 1 (0.01..10000), `quantization` 0 (0..2), `contrast_boost` 0 (-16..16), `exposure_boost` 0 (-16..16), `details` 4 (`DT_TONEEQ_EIGF`; 0 no, 1 averaged guided, 2 guided, 3 averaged EIGF, 4 EIGF), `method` 4 (`DT_TONEEQ_NORM_2`, "RGB euclidean norm"; 0 RGB average, 1 HSL lightness, 2 HSV value, 3 RGB sum), `iterations` 1 (1..20).

### rgbcurve — v1 — group tone|grading

`curve_num_nodes[]` default 2. `curve_type[]` default 2 (`MONOTONE_HERMITE`; 0 cubic spline, 1 Catmull-Rom). `curve_autoscale` default 0 (RGB, linked channels). `compensate_middle_grey` default 0. `preserve_colors` default 1 (`DT_RGB_NORM_LUMINANCE`).

Node storage is `[3][20]` (`MAX_ANCHORS` is 20 in `src/common/curve_tools.h`). Introspection zeros the nodes. `init` then sets node index 1 on R, G, and B to `(1, 1)`. The default curve is the identity segment `(0,0)–(1,1)`. Named presets in the same file install 6- and 7-node curves; those are not the module default.

### colorequal — v4 — group color

This is the scene-referred HSL. Eight hues × three graphs.

| Graph      | Members                         | Default | Range     |
| ---------- | ------------------------------- | ------- | --------- |
| Saturation | `sat_red` … `sat_magenta`       | 1       | 0..2      |
| Hue        | `hue_red` … `hue_magenta`       | 0       | -180..180 |
| Brightness | `bright_red` … `bright_magenta` | 1       | 0..2      |

Hue order: red, orange, yellow, green, cyan, blue, lavender, magenta. Also `threshold` 0.1 (0..0.3), `smoothing_hue` 1 (0.05..2), `contrast` 0 (-1..1), `white_level` 1 (-2..16), `chroma_size` 1.5 (1..10), `param_size` 1 (1..128), `use_filter` true, `hue_shift` 0 (-23..23).

### colorzones — v5 — group color|grading

Lab module, after the tone mapper. `$DEFAULT` on `strength` is 0 (-200..200) and `mode` is 0 smooth (1 strong). `channel` default annotation is hue (2). Lightness is 0, chroma is 1.

`init` calls `_reset_parameters(..., hue, splines v2)`, which sets 2 nodes per channel, curve type 1 (Catmull-Rom), y = 0.5. For the hue channel, `_reset_nodes` is called with `touch_edges` false, so x is `(k+0.5)/2`: 0.25 and 0.75, not 0 and 1. A zero-filled struct is not the default this `init` installs. `DT_IOP_COLORZONES_MAXNODES` is 20. `DT_IOP_COLORZONES_BANDS` (8) is used by presets, not by this reset. `default_enabled` is false.

### sharpen — v1 — group correct|effects

`radius` 2 (0..99), `amount` 0.5 (0..2), `threshold` 0.5 (0..100). Colorspace Lab. UI name is the C_ string "sharpen".

### diffuse — v2 — group effects

UI name "diffuse or sharpen". Defaults: `iterations` 1 (0..500), `sharpness` 0 (-1..1), `radius` 8 (0..2048), `regularization` 0 (0..4), `variance_threshold` 0 (-2..2), four `anisotropy_*` 0 (-10..10), `threshold` 0 (0..8), four speed terms `first`…`fourth` 0 (-1..1), `radius_center` 0 (0..1024). Positive `sharpness` is the scene-referred sharpen direction; the struct default is neutral.

### denoiseprofile — v12 — group correct|technical

UI name "denoise (profiled)". Struct defaults from the annotations and `reload_defaults` before the profile match:

| Member                          | Struct / reload default | Notes                                                                                                             |
| ------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `radius`                        | 1                       | 0..12, "patch size"                                                                                               |
| `nbhood`                        | 7                       | 1..30, "search radius"                                                                                            |
| `strength`                      | 1                       | 0.001..1000                                                                                                       |
| `shadows`                       | 1                       | 0..1.8, "preserve shadows"                                                                                        |
| `bias`                          | 0                       | -1000..100                                                                                                        |
| `scattering`                    | 0                       | 0..20                                                                                                             |
| `central_pixel_weight`          | 0.1                     | 0..10                                                                                                             |
| `overshooting`                  | 1                       | 0.001..1000                                                                                                       |
| `mode`                          | 1                       | wavelets. 0 non-local means, 3 nlmeans auto, 4 wavelets auto, 2 compute variance                                  |
| `wavelet_color_mode`            | 1                       | Y0U0V0. 0 is RGB                                                                                                  |
| `wb_adaptive_anscombe`          | true                    |                                                                                                                   |
| `fix_anscombe_and_nlmeans_norm` | true                    |                                                                                                                   |
| `use_new_vst`                   | true                    |                                                                                                                   |
| `compensate_hilite_pres`        | true in the annotation  | `reload_defaults` sets it from ISO highlight-preservation shift (`iso_shift > 0`)                                 |
| `y[][]`                         | annotation default 0.5  | 6 curves × 7 bands                                                                                                |
| `a[3]`, `b[3]`                  | no `$DEFAULT`           | `reload_defaults` copies `dt_iop_denoiseprofile_get_auto_profile`. On autodetect it sets `a[0] = -1` as a signal. |

The built-in preset "wavelets: chroma only" is different: `strength` 1.2, `shadows` 0, `a[0]` -1, and the Y0 curve's `y` is 0 while other curves stay at 0.5. That preset is not the struct default.

`rawdenoise` v2 is earlier in the raw pipe (`threshold` default 0.01). `nlmeans` v2 is "astrophoto denoise" (`radius` 2, `strength` 50, `luma` 0.5, `chroma` 1) and runs after `colorin`. Ordinary color-noise plans should use `denoiseprofile` and should clone a darktable-authored param block rather than invent `a[]`/`b[]`.

### hazeremoval — v3 — group correct|technical

`strength` 0.2 (-1..1), `distance` 0.2 (0..1), `compatibility_mode` false, `adaptive` true.

### vibrance — v2 — group color|grading

`amount` 25 (0..100). Lab, after the tone mapper. Scene-referred vibrance is `colorbalancergb.vibrance`.

## Masks

`dt_develop_blend_params_t` in `src/develop/blend.h` is the per-module blend record stored with the history step. `src/develop/blend.c` initializes `_default_blendop_params` as:

- `mask_mode` = `DEVELOP_MASK_DISABLED`
- `blend_cst` = none, replaced with the module colorspace when `dt_develop_blend_init_blend_parameters` runs. Scene-referred RGB modules get `DEVELOP_BLEND_CS_RGB_SCENE`
- `blend_mode` = `DEVELOP_BLEND_NORMAL2` (`0x18`)
- `blend_parameter` = 0
- `opacity` = 100
- `mask_combine` = `DEVELOP_COMBINE_NORM_EXCL`
- `mask_id` = 0
- `blendif` = 0 (no parametric channel enabled)
- feathering radius 0, guide `DEVELOP_MASK_GUIDE_IN_AFTER_BLUR`, blur radius 0, contrast 0, brightness 0, details 0
- `feather_version` = 1
- `blendif_parameters`: 16 channels × 4 floats, each channel `0, 0, 1, 1` (the parametric window is fully open)
- `blendif_boost_factors`: 16 zeros, then for scene-referred Jz and Cz (in and out) set to -6.64385619 so the GUI displays a boost of 0
- raster mask source empty, instance 0, id invalid, invert false

Parametric masks (Lightroom luminance/color range) are the `blendif` bitfield plus those parameter floats. They do not need a `masks_history` row. Drawn masks (brush, path, gradient) use `mask_id` and point blobs. DarkCockpit's architecture note already records that drawn external authoring is unproven. The frontend's shadow-lift controls (`ev`, `lo`, `hi`, `invert`) are a luminance window on top of an exposure change; they belong in blend params of `exposure` or in a `toneequal` band, not in a new iop name.

`DEVELOP_BLENDIF_SIZE` is 16. Input and output channels overlap the same indexes with different meanings per colorspace (Lab L/a/b, RGB gray/R/G/B, HSL H/S/L, JzCzhz). Which meaning applies follows `blend_cst`.

## Unknowns

- No runtime confirmation that a fresh 5.4.1 raw in DarkCockpit's installed binary uses `DT_IOP_ORDER_V50`. The source default says it does. The installed workflow preference string was not read.
- `sizeof` was not measured with a compiler except where the sidecar already asserts it (exposure 28, temperature 20). Other modules still need a size assert before a writer is trusted. Padding was not separately checked.
- `filmicrgb` noise and reconstruction fields outside the table above were read and omitted here because they are not the Lightroom mapping. They are still part of the v6 struct and must be packed if the whole struct is written.
- Drawn-mask point layout was not re-derived. The architecture note's "research spike" stands.
- REA produced no native call graph of `process()` for these modules.
