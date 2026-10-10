# Cellpose inside a napari app

## Device

- Decide and display the device before the run (card name, memory, CUDA/MPS/CPU).
  Cellpose silently falls back to CPU; on a 3D stack that is minutes vs seconds.
- "CPU" is almost always the default PyPI torch wheel (`+cpu`) in *the interpreter
  the app runs on* (per-user site-packages differ between accounts). Fix: install
  `torch` **and** `torchvision` from the same CUDA index
  (`--index-url https://download.pytorch.org/whl/cu130`); a mismatched torchvision
  fails at `import cellpose.models` with `operator torchvision::nms does not exist`.
- Size `batch_size` from free VRAM instead of the default 8.

## Units

- `diameter` is in XY pixels, `anisotropy` is the Z/XY voxel ratio. Take µm from
  the user and convert with the layer's calibrated voxel size, *after* any
  lateral decimation. A 5 µm nucleus stays 5 µm at any pixel size.
- `min_size` is a pixel count; there is no maximum — apply a max diameter to the
  finished label map (the only place it works for stitched objects) and renumber.

## Modes on a stack

| Mode | Notes |
|---|---|
| `2D + stitch` | Usually best on anisotropic stacks. Run the per-plane loop yourself (progress per plane, faster than cellpose's internal path), then stitch. |
| `3D` | Flows in 3D; slow (tens of minutes on large stacks). |
| `2D per plane` | Labels unconnected across Z; make labels unique across planes. |
| `2D on max projection` | Seconds; merges anything overlapping in Z. Good for sparse objects in thin stacks. |

- Cellpose 4 rejects `z_axis` in plain 2D mode (`2D image processing selected,
  but z_axis is not None`): feed 2D planes one call at a time.
- Cellpose stitches plane *i* with *i+1* only; one missed plane splits an object.
  A "stitch gap" (in µm of depth) that lets only *broken* chains bridge, best IoU
  first, one match per direction, cut splits 73→26 and merges 52→34 on a real
  crop. Offset plane labels only after stitching (the stitcher expects labels
  from 1 per plane).

## Models

- Discover models instead of hard-coding: what the installed cellpose ships,
  user-trained models, loose weights in `~/.cellpose/models`; add a rescan button.
- Cellpose 4 cannot load Cellpose 3 zoo models (`cyto3`, `nuclei`…) — filter them
  out rather than failing minutes into a run; the families cannot coexist in one
  environment (use a second environment/shortcut for both).
- `cpdino*` needs the DINOv3 backbone (`pip install git+https://github.com/facebookresearch/dinov3`);
  without it the weights download (≈1 GB) and then a `NameError` appears.
- Prefer an already-downloaded model as the initial selection.

## Process hygiene

- Under `pythonw` (Windows shortcut) `sys.stdout/err` are `None`; tqdm progress
  bars crash with `'NoneType' object has no attribute 'write'` after the slow
  part. Point missing streams at `os.devnull` at package import.
- torch and conda MKL each ship Intel OpenMP → `OMP: Error #15` and a C-level
  abort depending on import order. Set `KMP_DUPLICATE_LIB_OK=TRUE` before torch is
  imported (unless the user set it).
- Run on a `thread_worker`; send "plane 4 of 7…" through a signal.

## Results

Return a Labels layer at the source layer's scale and units, plus a per-object
table (voxels, volume µm³, equivalent diameter, centroid µm, intensity stats from
a *measure* channel that may differ from the *segment* channel). Measure via the
indices of labelled voxels so memory scales with the labelled fraction. Store the
label map in the source file (with the channel it was measured on) so batch
analysis — and hand-made label maps such as manual cell counts — go through the
same path.
