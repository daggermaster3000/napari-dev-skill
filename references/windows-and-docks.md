# Windows, docks and screens

## Contents
1. Diagnosing "window not responsive / not resizable"
2. Scroll-wrapping panels
3. Fitting to the screen (restore, fullscreen, monitor changes)
4. Sharing space between docks
5. Registering panels
6. Small things: fonts, tables, busy overlay, splash

## 1. Diagnose with numbers

```python
w = viewer.window._qt_window
print(w.screen().availableGeometry(), w.minimumSizeHint())
for d in w.findChildren(QDockWidget):
    if d.isVisible():
        print(d.windowTitle(), w.dockWidgetArea(d), d.minimumSizeHint(),
              d.widget().minimumSizeHint(), len(w.tabifiedDockWidgets(d)))
```

If `minimumSizeHint` exceeds the available screen, that is the bug: Qt cannot
shrink a main window below the sum of its docks' minimums, and on a single Mac
screen the window ends up partly off-screen, the resize corner unreachable, and
every edge drag fights the dock layout (feels frozen). Typical culprits: long
forms (segmentation), a toolbar row of buttons, napari's own layer-controls dock
(its minimum grows with the layer type), several docks stacked in one column.

## 2. Wrap panels in scroll areas

```python
class _PanelScrollArea(QScrollArea):
    def sizeHint(self):
        hint, inner = super().sizeHint(), self.widget()
        if inner is None:
            return hint
        want, area = inner.sizeHint(), screen_rect(self)
        w, h = want.width(), want.height()
        if not area.isEmpty():
            w, h = min(w, area.width() // 2), min(h, area.height() // 2)
        return QSize(max(hint.width(), w), max(hint.height(), h))

def scrollable(widget, *, vertical=True):
    a = _PanelScrollArea(); a.setWidgetResizable(True); a.setFrameShape(QFrame.NoFrame)
    a.setVerticalScrollBarPolicy(Qt.ScrollBarAsNeeded if vertical else Qt.ScrollBarAlwaysOff)
    a.setWidget(widget)
    if not vertical:  # a one-row toolbar: fixed height, scroll sideways
        h = widget.sizeHint().height()
        st = a.style()
        if not st.styleHint(QStyle.SH_ScrollBar_Transient, None, a):
            h += st.pixelMetric(QStyle.PM_ScrollBarExtent, None, a)
        a.setFixedHeight(h); a.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Fixed)
    return a
```

- Add `scrollable(widget)` as the dock widget; keep a reference to the inner
  widget as "the panel" (other code, the tour, tests talk to it).
- napari adds a vertical stretch to the widget it is given; with a scroll area
  that no longer reaches your panel. Call `dock._maybe_add_vertical_stretch(panel)`
  (guarded) so controls stay packed at the top.
- napari's own layer controls: `dock = viewer.window._qt_viewer.dockLayerControls;
  dock.setWidget(scrollable(dock.widget())); dock.setMinimumSize(50, 50);
  dock.layout().invalidate()` — without the reset the old minimum stays pinned.
- A control inside a scroll area may be scrolled out of view: before pointing at
  it (guided tour, highlight), walk its parents and call
  `scroll_area.ensureWidgetVisible(widget)` on each `QScrollArea`.

## 3. Fit to the screen, and keep fitting

napari restores `settings.application.window_size/position/fullscreen`, which
may come from a larger or different monitor. Fit after show and whenever the
window could outgrow its screen again:

```python
def fit_to_screen(win):
    if win.isFullScreen() or win.isMaximized():
        return
    area = win.screen().availableGeometry(); frame = win.frameGeometry()
    if area.contains(frame):
        return
    ew, eh = frame.width() - win.width(), frame.height() - win.height()
    w, h = min(frame.width(), area.width()), min(frame.height(), area.height())
    win.resize(max(w - ew, win.minimumWidth()), max(h - eh, win.minimumHeight()))
    win.move(min(max(frame.x(), area.left()), area.right() + 1 - w),
             min(max(frame.y(), area.top()), area.bottom() + 1 - h))
```

Install an event filter on the main window: on `QEvent.Show` and
`QEvent.WindowStateChange`, and on `windowHandle().screenChanged`, schedule
`QTimer.singleShot(0, fit)` (after the platform finishes the state change).
Leaving fullscreen restores the stale normal geometry — that is exactly when it
is needed.

## 4. Share space between docks

After wrapping, panels no longer demand their size, so Qt may give a panel a
93-px sliver. Size whole columns at once after the window is laid out
(`QTimer.singleShot(50, ...)`):

```python
window.resizeDocks([controls, layer_list, experiment], [h*.25, h*.2, h*.55], Qt.Vertical)
window.resizeDocks([experiment], [w*.25], Qt.Horizontal)
```

A bottom transport bar: size it to `panel.sizeHint().height() + title bar`.

## 5. Registering panels

A manifest (identifier, title, lazy factory, area, `tabify_with`, order,
`start_hidden`) beats hand-wiring in the app: one `register_panel(...)` per
panel, factories import their widget inside the function (heavy deps stay off
the startup path), a failing panel is logged and skipped. Tabify with
`window.tabifyDockWidget(target, dock)` then `target.raise_()`.

## 6. Small things

- napari's stylesheet overrides `QLabel.setFont`. Use a per-widget
  `setStyleSheet`.
- Fit a small table to its rows instead of leaving empty space: header height +
  sum of row heights + frame, `setFixedHeight`, cap at N rows.
- Tables of editable rules in a narrow dock: stretch every column
  (`horizontalHeader().setSectionResizeMode(QHeaderView.Stretch)`), turn the
  horizontal scroll bar off, and keep combo labels short ("Part #", "Regex") with
  the long explanation in `setItemData(i, text, Qt.ToolTipRole)`.
- A live preview table next to settings ("what the export will say") makes a
  rules UI understandable; mark derived/changed cells (bold, grey "—" for empty).
- A frozen main thread cannot paint anything, including a "busy" animation. A
  busy overlay has to be a second process fed by a watchdog thread that only
  touches plain data and a pipe — never Qt objects from another thread.
- Splash: create the `QApplication` (napari's `get_qapp()`) first, build the
  viewer with `show=False`, pump the splash between build steps, show the window
  only when complete.
- `QGraphicsView` overlays (boxes and labels on an overview image): cosmetic pens
  (`pen.setCosmetic(True)`) and `ItemIgnoresTransformations` on labels keep them
  readable at any zoom; set `QSplitter.setSizes` explicitly or a side list can
  take the whole width; `fitInView` again on `showEvent`.
