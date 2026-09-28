# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

 single-file ComfyUI custom node package. All code lives in `__init__.py`there
is no build step, test suite, or linter config. Three nodes —
`ResolutionPreset` `ResolutionSelectorMP`  `ResolutionAlignToGrid` — all pure
integer maths with no torch/numpy involvement.

`ResolutionPreset`'s combo options are **generated at import time** by
`build_presets()` from same `lattice_step()` solver uses, so list can
never disagree with `solve_exact()`. Do not hand-edit resolutions into it.
per-ratio grid choice (coarsest alignment yielding `PRESET_MIN_ENTRIES` sizes)
exists because fine-lattice ratios like 1:1 have L=16 and would otherwise emit
~105 entries; changing `PRESET_MIN_MP` / `PRESET_MAX_MP` / `PRESET_MIN_ENTRIES`
silently reshapes every label, and **labels are public API** — saved
workflow stores selected label string, so altering format or tuning
constants breaks existing graphs.

Registered through ComfyUI's **V3 schema API**, not legacy
`NODE_CLASS_MAPPINGS` dict: `comfy_entrypoint()` returns `ComfyExtension` whose
`get_node_list()` yields `IO.ComfyNode` subclasses. Inputs are declared in
`define_schema()` and arrive as named kwargs to `execute()`.

## Local environment

- ComfyUI checkout: `/home/dreevelle/comfy/ComfyUI`
- Python venv: `/home/dreevelle/comfy-env/bin/python`
-  package is published, `ComfyUI/custom_nodes/comfyui-resolution-tools`
  (lowercase, registry id) is **ComfyUI-Manager install** that tracks
  registry releases. This working tree is not wired into ComfyUI.

To test change before releasing it, symlink this tree in under its CamelCase
name so it doesn't collide with Manager install:

```bash
ln -sfn /home/dreevelle/Projects/ComfyUI-ResolutionTools \
        /home/dreevelle/comfy/ComfyUI/custom_nodes/ComfyUI-ResolutionTools
```

**Remove that symlink before doing anything in ComfyUI-Manager for this
package.** Manager's remove/replace path does `shutil.rmtree` on *resolved*
path, so it deletes symlink target's contents instead of unlinking — on
2026-08-09 that wiped this working tree, `.git` included, and it had to be
restored by re-cloning. Commit and push before any Manager operation, and never
leave symlinked dev copy in place while installing or updating published
version.

## Verifying changes

`__init__.py` imports `comfy_api.latest` at module scope, so it needs ComfyUI on
`sys.path` — but maths functions are module-level and dependency-free,
they can be exercised directly:

```bash
cd /home/dreevelle/comfy/ComfyUI && PYTHONPATH=/home/dreevelle/comfy/ComfyUI \
/home/dreevelle/comfy-env/bin/python -c "
import importlib.util
spec = importlib.util.spec_from_file_location('rt', '/home/dreevelle/Projects/ComfyUI-ResolutionTools/__init__.py')
m = importlib.util.module_from_spec(spec); spec.loader.exec_module(m)
print(m.solve_exact(9, 16, 16, 2.36e6))   # -> (1152, 2048)
"
```

Two invariants any change to `solve_exact` must preserve, and which are worth
re-fuzzing over random `(ratio, alignment, target)` triples:

1. `width % alignment == 0 and height % alignment == 0`
2. `width * h_ratio == height * w_ratio` (ratio held exactly, no drift)

Known-good anchors: `9:16 @ m=16, 2.36 MP -> 1152x2048`
`16:9 @ m=32, 1.33 MP -> 1536x864` `21:9 @ m=16, 2.37 MP -> 2352x1008`.

## Why the maths is what it is

**Real megapixels.** Core's `comfy_extras/nodes_resolution.py:78` uses
`megapixels * 1024 * 1024`. That's SDXL-era convention where "1 MP" meant
1024x1024. This package deliberately diverges to `* 1_000_000`. Do not "fix"
this to match core — divergence is point, and README documents it.

**Lattice solving.** `lattice_step()` computes smallest scale increment that
keeps both axes on alignment grid while holding exact ratio. Core rounds
each axis independently, which drifts ratio (9:16 at 2.36 MP becomes 0.5649
instead of 0.5625). `solve_nearest()` preserves core's behaviour for when
lattice is too coarse; it is opt-in via `exact_ratio=False`.

**Alignment = VAE spatial downscale x DiT patch size.** Verified against
ComfyUI source rather than assumed:

- Krea 2 -> 16. `latent_formats.LatentFormat.spacial_downscale_ratio = 8`
  (inherited by `Wan21`which `supported_models.Krea2` uses) x `patch=2` in
  `comfy/ldm/krea2/model.py`.
- MiniMax H3 -> 32. `latent_formats.MiniMaxH3Video.spacial_downscale_ratio = 16`
  x `patch_size=(1, 2, 2)` in `comfy/ldm/minimax/model.py`.
- SDXL/SD1.5 -> 64. UNet, no patchify; three downsample stages on 8x VAE.

 failure mode when unaligned is `comfy.ldm.common_dit.pad_to_patch_size`
which pads with `padding_mode="circular"` and appends only at end — hence
bottom/right band, never top/left. If future model changes VAE or patch size,
 alignment table in README `ALIGNMENT_TOOLTIP` both need updating.

## Conventions

- Version is duplicated in `__version__` (`__init__.py`) `[project].version`
  (`pyproject.toml`) — bump both.
- `dependencies` stays empty; this package is stdlib-only by design.
- Node IDs (`ResolutionSelectorMP` `ResolutionAlignToGrid`) are public API.
  Renaming one breaks every saved workflow that uses it.

## Releasing

Intended for Comfy Registry as `comfyui-resolution-tools` under publisher
`dreevelle`same setup as ComfyUI-FastImageSequence.

**Changing `[project].version` in `pyproject.toml` on `main` publishes
release.** `.github/workflows/publish_action.yml` watches that path and runs
`Comfy-Org/publish-node-action` with `REGISTRY_ACCESS_TOKEN` repo secret.
Don't touch version field unless release is intended; republishing
existing version fails with `400 The node version already exists`.

`.comfyignore` keeps `CLAUDE.md`  `.github/` out of published archive.
`comfy node validate` runs registry checks locally; `comfy node pack`
produces exact archive that would be uploaded.

`assets/icon.png` is registry icon, pointed at by `[tool.comfy] Icon` as
raw GitHub URL on `main`. registry stores *URL*, not bytes,
Manager hotlinks it, so **replacing this file on `main` restyles every card with
no republish**. Only changing URL needs version bump, because `icon` is
node-level metadata written as side effect of publishing version.

 background is **transparent on purpose**. frontend's `PackBanner` draws
 icon twice into 7:3 box — once as backdrop, `bg-cover` plus `blur(10px)`
at `opacity-30`and once `object-contain` on top. opaque icon therefore shows
as hard rectangle: `bg-cover` crops to centre and zooms ~2.3x, so it samples
mostly glow and lifts surround to ~47, while contained copy still shows
 icon's own near-black corners at ~17. Deriving alpha from luminance removes
 edge completely, and it is right derivation because artwork is
additive glow on near-black. cost, accepted deliberately, is that pale
mark washes out on ComfyUI's light theme.

Regenerate from brand original, capped at registry's 400x400 square
maximum. Downscale in linear light; naive sRGB-space resize dulls ring
stroke and blurs bar separations inside D:

```bash
magick ~/Projects/alora/dev/assets/brand/dreevelle_pfp.png \
  -colorspace RGB -filter Lanczos -resize 400x400 -colorspace sRGB \
  \( +clone -grayscale Rec601Luma -level 7.45%,100% \) \
  -alpha off -compose CopyOpacity -composite \
  -background none -alpha Background -strip assets/icon.png
oxipng -o max --strip safe assets/icon.png
```

Do not set `[tool.comfy] Banner`. `PackBanner` resolves `banner_url || icon`
 banner *replaces* mark rather than sitting behind it.

