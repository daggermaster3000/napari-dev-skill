---
name: napari-dev
description: Build and debug napari-based microscopy apps and plugins — dock panels, Labels/Points/Image layers, mouse and keyboard interaction, thread workers, window sizing on laptops, multiscale/dask/.ims data, units and scale bars, Cellpose segmentation, movie/figure export, macOS app bundles, and companion Streamlit explorers for the exported data. Use this skill whenever the user is writing or fixing code that imports napari, qtpy or cellpose, mentions a napari viewer, dock widget, layer, canvas, "panel", magicgui, thread_worker, or a Streamlit/plotly app over microscopy results — even if they only describe a symptom ("the window can't be resized", "the button does nothing", "the scale bar says pixels", "cellpose says CPU").
---

# napari development

Practical rules for building napari applications that hold up on real data and
real screens. They come from building a full microscopy viewer (readers for
Imaris/TIFF/OME-Zarr, a dozen dock panels, Cellpose segmentation, analysis
exports, Streamlit explorers). Most rules exist because the obvious approach
failed quietly — read the *why* so you can recognise the same failure in a new
shape.

## How to work

1. **Find the seams first.** Before adding a feature, look for what the codebase
   already has: a panel registry, a layer-grouping helper, a writer, a
   close/write/reopen helper, a palette function. Reusing them keeps behaviour
   consistent (units, colours, names) — two implementations of the same thing
   drift apart.
2. **Keep the logic out of Qt.** Put the maths, file I/O and rendering in a plain
   module (`foo.py`, "no Qt here"), and make the widget a thin view
   (`widgets/foo_widget.py`). The plain module is testable without a display and
   reusable from scripts, batch jobs and Streamlit.
3. **Verify through the real UI, not just the method.** A test that calls
   `widget.new_counter()` directly will not catch that the button passes Qt's
   `checked` flag into it. Drive the actual button (`button.click()`), the actual
   canvas (`QTest.mousePress` on `canvas.native`), then look at a screenshot
   (`widget.grab().save(...)`). See `references/testing.md`.
4. **Measure the problem before fixing it.** "Window not resizable" turned out to
   be a minimum size of 1591×1306 on a 1512×859 screen — one print of
   `window.minimumSizeHint()` and each dock's hint found it. Print the numbers.

## The rules that bite most often

### Windows, docks and screens → `references/windows-and-docks.md`
- A `QMainWindow` cannot shrink below the **sum of its docks' minimum sizes**.
  Tall forms and long toolbars make that larger than a laptop screen: macOS then
  leaves the window off-screen and unresizable. Wrap every panel in a
  `QScrollArea` (widgetResizable) so docks have tiny minimums; give the scroll
  area a `sizeHint` that follows its content (capped at half the screen) or Qt
  will starve panels on first layout.
- napari restores the saved window size (possibly from an external monitor) and
  fullscreen state. Fit the window to `screen.availableGeometry()` after show,
  again on leaving fullscreen/maximised (`WindowStateChange`) and on
  `windowHandle().screenChanged`.
- After wrapping, set dock proportions explicitly with
  `window.resizeDocks([...all docks in a column...], sizes, Qt.Vertical)` —
  sizing one dock lets Qt steal from whichever neighbour it likes.
- A dock whose widget you swap keeps the old minimum pinned: reset
  `dock.setMinimumSize(50, 50)` and `dock.layout().invalidate()`.
- Many panels: start them hidden and open them from your **own** menu-bar menu
  of `dock.toggleViewAction()`s (napari rebuilds its own menus and drops added
  actions). napari's dock close button *destroys* the dock — set
  `dock.destroyOnClose = dock.hide`.
- Remember the session (open panels, settings, files, view) by saving on the
  window's `QEvent.Close` — at `aboutToQuit` the viewer is already empty — and
  restore in order: panels, files, settings, view.
- napari's stylesheet overrides `setFont`; style emphasised labels with
  `setStyleSheet("font-size: 22px; font-weight: bold;")`.

### Interaction and threads → `references/interaction.md`
- `QPushButton.clicked` passes `checked: bool`. Never connect it straight to a
  method with an optional first parameter (`def new(self, data=None)` receives
  `False`). Give the slot `*_args` or no parameters.
- Canvas clicks: append to `layer.mouse_drag_callbacks`, keep the layer in
  `pan_zoom` mode and **selected** (callbacks only reach the active layer), and
  use the generator pattern — `yield` on press, loop over `mouse_move`, treat it
  as a click only if the screen position moved less than a few pixels.
- Edit labels with `layer.data_setitem(indices, value)`: it refreshes the
  display and records undo (Ctrl+Z) for free.
- Long work runs on `napari.qt.threading.thread_worker`; send progress to the
  GUI thread through a Qt `Signal` on a small `QObject` relay. Never touch a
  widget from the worker.
- Shortcuts: `viewer.bind_key("Control-Shift-P", lambda v, h=handler: h(),
  overwrite=True)`. "Control" is ⌘ on macOS. Wrap in try/except — napari may
  own the key.

### Data, scale and units → `references/data-and-io.md`
- Read lazily (dask), pick a pyramid level by the output size you need, and
  materialise only the reduced result (a projection, a thumbnail).
- From napari 0.8 the scale bar reads `layer.units`; `viewer.scale_bar.unit`
  no longer exists. **Every** layer you add must carry the same units as the
  image, or one dimensionless layer makes napari drop units for all
  ("Inconsistent units across layers").
- A derived layer (labels, projection, counter) copies the source's `scale`,
  `translate` and `units`; use the finest level's shape and the *base* scale,
  not the scale of a level the 3D renderer swapped in.
- HDF5 will not open for writing a file open for reading. To write into an open
  sample: remove its layers, release the reader handle, write, reopen, restore
  the camera.
- Store provenance with what you save (who drew a region, which channel a label
  map came from) as attributes on the stored object, and make sure the loader
  keeps them — exports can only report what survives the round trip.

### Cellpose → `references/cellpose.md`
Decide and show the device before running; sizes in µm converted with the voxel
size; `2D + stitch` in your own loop for progress and gap-bridging; CP3 and CP4
model families can't coexist; guard `sys.stdout is None` under `pythonw`; set
`KMP_DUPLICATE_LIB_OK` before importing torch.

### Startup and packaging → `references/packaging-and-startup.md`
Plugin-engine modal dialog, numba cache dir, splash/busy overlay as a separate
process, Windows `.lnk` / macOS `.app` launchers, and macOS `FileOpen` events
(files dropped on the app icon never reach `argv`; the command line is also
replayed as `FileOpen` events — ignore paths already in `sys.argv`).

### Streamlit explorers over exported data → `references/streamlit-explorers.md`
Shared reader/stats module for all apps, `st.cache_data`, group-by columns,
normalisation dropdowns, one colour store every palette goes through, spatial
heatmaps, `AppTest` headless checks, and decoding plotly's base64-packed arrays
in tests.

### Explorer UI → `references/streamlit-ui.md`
Read before adding a tab, control or app. Same skeleton every time: wide page,
sidebar only for whole-app choices (Workbook, Colours, Rows, Cells), a caption of
what survived the filters under the title, one question per tab with `Table`
last, controls left / result right in `st.columns([1, 3])`. One vocabulary for
states — `info` (load or install this), `warning` (change a choice), `error`
(broke), `caption` (partial) — never an empty chart or a traceback. Reuse the
existing labels ("Colour by", "Group by"); key every widget with its tab prefix.

### Explorer colour → `references/streamlit-colour.md`
Read before choosing any colour. One shared palette (8 categorical, named
neutrals), assigned by position in a fixed order — reference group first,
unknown last and grey — so a value keeps its colour in every plot and under any
filter. Sequential ramp for amounts, diverging only for signed differences and
symmetric about zero; pin and share the range. Every mapping goes out through
`paint(kind, …)`.

### Graph aesthetics → `references/streamlit-explorers.md` ("house style")
Every plot, interactive or exported: **every mark outlined in black** over its
group colour (violins, boxes, bars, dots, filled shapes), **violins without an
inner box plot**, lines and heatmaps left alone. Apply it in one wrapper that
every chart goes through (`chart()` → `outlined(figure)`), and the same edges in
matplotlib report figures.

## Checklist before saying a napari feature is done

- The real control was exercised (button click / canvas mouse event), not just
  the method.
- A screenshot was looked at, at a realistic window size (try ~1500×860).
- New layers carry the source's scale, translate and units.
- Long operations run off the GUI thread and report progress.
- Anything written into a data file survives close → reopen, and the test uses
  a *copy* of the sample file.
- Tests that save settings (names, colours, remembered rules) redirect them to a
  temp dir instead of the user's real files.
- The pure-logic module has unit tests that run without a display.
- New explorer UI follows `references/streamlit-ui.md` (layout, states, wording)
  and takes its colours from the shared palette through `paint()`
  (`references/streamlit-colour.md`) — no hex literal in a plot.
- New plots go through the house-style wrapper (black outlines, violins without
  an inner box), not a raw `st.plotly_chart` or unstyled matplotlib call.
