# Data, scale, units and files

## Reading lazily

- Readers return dask arrays (per channel, per pyramid level). Keep them lazy;
  reduce first (`array.max(axis=z)`, a slice at the displayed timepoint) and
  `np.asarray` only the result. A 271×2040×2040 channel is 2.2 GB materialised
  and ~8 MB projected.
- Pick the level by what you will output: the coarsest level whose YX is still
  ≥ the target size (thumbnails, movie panels, slide tiles). Imaris pyramids
  bottom out near 64 px — too coarse for a useful preview.
- When a 3D renderer swaps a multiscale layer to one level, stash the original
  level list and base scale in `layer.metadata` so other code can still reach
  full resolution and the right voxel size.
- Z is found from the reader's axis string (`"TZYX"`), not guessed by position;
  a file that says it has no Z is not flattened over time.

## Units and the scale bar

- napari ≥ 0.8: `ScaleBarOverlay` has no `unit`; the bar reads `layer.units`,
  which defaults to `pixel`. Pass `units=("µm",)*ndim` with `scale`. Check for the
  field (`"unit" in type(viewer.scale_bar).model_fields`) instead of the version.
- One layer without units (a Shapes ROI layer, labels, a projection) makes napari
  log "Inconsistent units across layers" and drop units for everything. Give every
  created layer the units of its source layer (a `units_like(layer)` /
  `world_units(viewer, ndim)` helper).
- Derived layers copy `scale`, `translate`, `units`, and for images `colormap`,
  `blending`, `contrast_limits` so they look like their source.

## Grouping layers into samples

Each file becomes one layer per channel. Group by `metadata["mv_metadata"].file_path`
to get "samples" with ordered channels (colour from the colormap, contrast
limits, pyramid levels, current step). Render figures/movies from that model,
not from screenshots, when the output should not depend on the view.

## Writing into files that are open

HDF5 will not open for writing what is open for reading, and readers usually
keep the `h5py.File` for the life of the process. To write (ROIs, label maps,
manual counts) into a sample that is on screen:

1. remember the camera (`center`, `zoom`, `angles`) and `dims.current_step`;
2. remove the layers whose source is that file; release the reader's handle;
3. write (compressed; store voxel size and provenance as attributes);
4. reopen through the app's normal open path; restore the camera; move your own
   overlay layer back on top.

Put this in one shared helper module used by every panel that writes.

Keep derived data inside the source file in its own group (e.g. `/ARGUS`) rather
than a sidecar folder that gets orphaned when files are moved. Never modify the
vendor's groups. Read back what you wrote to confirm it is there.

**Provenance survives only if the loader keeps it.** A loader that rebuilds
objects from a fixed set of attributes silently drops everything else (who drew
the region, the source channel). Load the remaining attributes into an `attrs`
dict and pass them through to the export; then add the export column in one
place (post-process the finished sheets) rather than in every sheet builder.

## Stage positions and overviews

Imaris stores absolute stage extents (`ExtMin0..2`, `ExtMax0..2`). With them:
- overview mosaics (`name_F0000` series, single plane) can be stitched by stage
  position with feathered overlaps;
- samples can be placed on the overview as boxes;
- overviews can be kept out of sample lists and batch jobs: a file is an overview
  if it is a single plane with a stage position and is an `_F####` field, contains
  another file's footprint, or is named "overview". Z-stacks never count (a 20×
  stack containing a 40× close-up is still a sample). "Flat" means the Z axis has
  one plane, not "shape[0] == 1" (that is time).

## Batch jobs over folders

- List files recursively but skip macOS `._*` companions and the job's own output
  folder (never re-process last run's output).
- Offer "keep the subfolder structure" (`out / src.parent.relative_to(root)`);
  when pooling, detect name collisions up front and add the folder to the name —
  otherwise the second `fish1` is silently "already there".
- One failing file is reported, not fatal. Report skipped files by name, and log
  outputs relative to the output folder so duplicates are distinguishable.
- Duplicate sample names across folders: append the differing group values
  ("fish1 (DMSO)") so every exported row traces back to one file.
- Group columns (genotype, treatment…) come from file and folder names by rules
  the user can see and edit (field number, one of these words, whole folder,
  regex) with a live preview; a rule that finds nothing yields "", never a guess.
