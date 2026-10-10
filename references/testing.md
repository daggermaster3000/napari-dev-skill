# Testing napari features

## Layers of tests

1. **Pure modules (no Qt)** — pytest, fast, run everywhere. Geometry, stats,
   file formats, rendering to numpy arrays, output paths.
2. **GUI smoke script** — builds the real viewer (`show=False`, or `show=True`
   when screenshots matter), exercises each panel end to end, prints `ok/FAIL`
   lines. Use copies of sample files for anything that writes.
3. **Driven interaction + screenshot** — for anything a user clicks.

## Drive the real controls

```python
button.click()                       # passes Qt's checked flag, like a user
combo.setCurrentIndex(i); spin.setValue(v)
canvas = viewer.window._qt_viewer.canvas.native
cx, cy = canvas.width() // 2, canvas.height() // 2
QTest.mousePress(canvas, Qt.LeftButton, Qt.NoModifier, QPoint(cx, cy))
QTest.mouseRelease(canvas, Qt.LeftButton, Qt.NoModifier, QPoint(cx, cy))
QTest.mouseMove(canvas, QPoint(cx + 60, cy))          # a drag between press/release
QTest.mousePress(canvas, Qt.LeftButton, Qt.ShiftModifier, QPoint(cx, cy))
```

Check where a click landed: the camera centre in world units maps to the canvas
centre; `(center - translate) / scale` is the expected data position.

Pump events between steps (`QCoreApplication.processEvents()` in a loop with short
sleeps) and wait on real conditions (`worker is None`, rows present) with a
timeout, rather than fixed sleeps.

## Look at it

```python
widget.grab().save("panel.png")                       # one panel
viewer.window._qt_window.grab().save("window.png")    # whole window
```

Look at the image before declaring done. Screenshots found: a side list eating
the whole dialog width (splitter needed `setSizes`), labels too small after
`fitInView` (use `ItemIgnoresTransformations`), a clipped "Value" column, a
counter label that ignored `setFont`, dots too large for the image, a panel
squeezed to a sliver after scroll-wrapping.

Run at a realistic laptop size (e.g. 1512×859 available) — many layout bugs only
exist there.

## Gotchas

- Print with `flush=True` in scripts; buffered stdout is lost if Qt aborts on exit.
- `QT_QPA_PLATFORM=offscreen` has no OpenGL: skip canvas screenshots there.
- Floating a dock in a test and then closing the viewer can raise a napari
  `closeEvent` "wrapped C/C++ object … deleted" warning — a test artefact.
- Tests that write settings (the user's name, colours, remembered rules,
  onboarding state) must redirect the settings path to a temp dir.
- Pre-existing failures: note them once and confirm they also fail without your
  change (`git stash` → run → `git stash pop`), so they are not blamed on it.
- For long headless runs, run in the background and time each step; a single
  slow first run is compilation, repeated slow runs are a loop.
