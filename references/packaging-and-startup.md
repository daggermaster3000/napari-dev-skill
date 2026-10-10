# Startup, launchers and packaging

## Startup blockers

- **Plugin-engine modal dialog.** If any installed plugin uses the deprecated
  engine, napari shows a modal "Installed Plugin Warning" during window
  construction; from a desktop shortcut nothing loads until it is dismissed.
  Mark the currently installed shimmed plugins as already warned
  (`settings.plugins.already_warned_shimmed_plugins`,
  `only_new_shimmed_plugins_warning = True`) — new plugins still warn.
- **numba cache in a read-only prefix** (system-wide conda): colormap import
  stalls for minutes every launch. Set `NUMBA_CACHE_DIR` to a writable per-user
  folder before importing napari, unless already set.
- **Remote Desktop**: OpenGL falls back to software; window creation can take
  minutes. Not your code.
- Log to a per-user file (`%LOCALAPPDATA%` / `~/Library/Caches`) and show a
  dialog with the traceback on startup failure — a shortcut has no console.

## Launchers

- Keep one stable launcher script (`launch_viewer.py`) that puts the project on
  `sys.path` and calls the CLI entry point.
- **Windows**: a `.lnk` targeting `pythonw.exe launch_viewer.py` (pywin32, or
  a VBScript via `cscript` as fallback). Windows appends dropped files to argv.
- **macOS**: a minimal `.app` bundle — `Contents/Info.plist`,
  `Contents/MacOS/<exe>` (a `sh` script), `Contents/Resources/icon.icns`.
  - The script `exec`s the absolute interpreter path (do **not** resolve venv
    symlinks — that starts the base interpreter without the venv's packages),
    prepends its `bin` to `PATH`, drops `-psn_*` args.
  - `exec` keeps the PID LaunchServices launched, so Apple Events for the bundle
    reach Python and macOS shows the bundle's name.
  - Declare `CFBundleDocumentTypes` with `LSHandlerRank: Alternate` (offered under
    Open With, never stealing the default) and `public.folder` for Zarr stores.
  - Register with `lsregister -f`. Mark bundles with your own
    `CFBundleIdentifier` and only delete bundles carrying it on uninstall.
  - A `--console` variant can be a `.command` file (opens in Terminal).

## macOS FileOpen events

Files dropped on the app/Dock icon or opened with *Open With* arrive as
`QEvent.FileOpen`, never in `argv`. Install an application event filter right
after the `QApplication` exists (before the splash pumps events), queue paths
until the viewer is built, then hand them over in one batch. macOS also replays
the command line as `FileOpen` events — including the launcher script itself —
so ignore any path equal (after `realpath`) to an entry of `sys.argv`.
PyQt5 cannot construct `QFileOpenEvent` in tests; call the filter with a stand-in
object that has `type()` and `file()`.

## Quitting a test instance

`osascript -e 'tell application id "…" to quit'` can hang on napari's
confirm-close dialog when layers are open. Kill the test process by PID instead
of `pkill -f` patterns that may match the user's own session.
