# Interaction: buttons, canvas clicks, labels editing, threads, shortcuts

## Qt signal arguments

`clicked` emits `checked: bool`; `toggled` emits the new state;
`currentIndexChanged` emits an int; `textChanged` a str. PyQt passes the argument
whenever the slot can accept it. So:

```python
self._new_button.clicked.connect(self.new_counter)
def new_counter(self, data=None):   # BUG: data == False on every click
```

The panel then treated `False` as "saved counts of the wrong shape" and quietly
did nothing. Keep public slots argument-free (`def new_counter(self, *_args)`)
and route internal calls with data through a private method.

## Clicking on the canvas

```python
layer.mouse_drag_callbacks.append(self._on_mouse)
viewer.layers.selection.active = layer   # callbacks only reach the active layer
layer.mode = "pan_zoom"                   # drags still pan

def _on_mouse(self, layer, event):
    if event.type != "mouse_press" or getattr(event, "button", 1) != 1:
        return
    start_world = tuple(event.position)          # world coords, all viewer dims
    start_px = np.asarray(event.pos, float)      # canvas pixels
    shift = "Shift" in {str(getattr(m, "name", m)) for m in (event.modifiers or ())}
    yield
    while event.type == "mouse_move":
        yield
    if np.abs(np.asarray(event.pos, float) - start_px).max() > 4:
        return                                   # it was a pan
    data_pos = (np.asarray(start_world)[-layer.ndim:] - translate) / scale
    ...
```

- Use the **last `layer.ndim`** world coordinates: a 2D layer over a TZYX image
  gets a 4-component `event.position`.
- Refuse gracefully in 3D display (`viewer.dims.ndisplay == 3`): the click
  position is a ray there.
- Treat a click that lands on an existing object as a slip ("already counted"),
  not a new object painted around the old one.

## Editing labels

`layer.data_setitem(indices, value)` updates the display and pushes onto the
layer's undo history (Ctrl+Z works). Compute indices in a window around the
click (a disc/ball from the diameter in µm divided by the layer scale) instead
of scanning the whole array; exclude pixels already labelled so touching objects
stay separate. Count objects with `np.unique` (or `bincount`) on a debounced
timer connected to `layer.events.paint` and `layer.events.data`, since users
may also paint with napari's own brush.

Use a 2D label map (YX) over a 3D stack when a mark should count once for the
whole stack; a full ZYX map of a whole brain is gigabytes.

## Threads

```python
class _Relay(QObject):
    message = Signal(str)
    progress = Signal(int, int)

from napari.qt.threading import thread_worker

@thread_worker
def _job():
    return work(progress=relay.progress.emit, should_cancel=lambda: self._cancelled)

w = _job(); w.returned.connect(self._done); w.errored.connect(self._fail)
w.finished.connect(self._clear); self._worker = w; w.start()
```

- The worker never touches widgets; signals are queued onto the GUI thread.
- Cancellation: a flag the loop polls; raise a dedicated `Cancelled` exception
  and delete partial output files in `finally`.
- Compute labels/names that depend on GUI state *before* starting the worker and
  pass them in.
- A generator worker (`yield`) lets a grid fill in as items are read; signals
  emitted in order arrive in order, so a `returned` handler runs after all
  previously emitted progress signals.
- Anything slow that runs before the first visible result (reading a whole
  stack to fit contrast) should still report progress ("Reading GFP: plane 40 of
  120").

## Shortcuts

```python
for key, handler in (("Control-O", open_images), ("Control-Shift-P", batch_mip)):
    try:
        viewer.bind_key(key, lambda _v, _h=handler: _h(), overwrite=True)
    except Exception:
        logger.debug("could not bind %s", key)
```

"Control" maps to ⌘ on macOS. Bind the default argument (`_h=handler`) or every
lambda calls the last handler. Put the shortcut in the button's tooltip.

## Application-wide drag and drop

An event filter on `QApplication.instance()` that claims `DragEnter/DragMove/Drop`
only when a URL looks loadable by your readers, and defers loading with
`QTimer.singleShot(0, ...)` so the drag source is not blocked. Everything else
falls through to napari.

## Remembered user settings

Small per-user preferences (the user's name for provenance, remembered rules,
colours) belong in a JSON file in a per-user app-data dir, behind a function
(`settings_file()`) that tests can redirect. Start from a sensible default (the
account's full name via `pwd.getpwuid(os.getuid()).pw_gecos`, falling back to
`getpass.getuser()`), and save on `editingFinished` and again before the action
that uses it.
