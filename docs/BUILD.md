# Build, run and troubleshoot

Python, pywebview on WebView2 and a one-file PyInstaller build; the versions below are the ones verified on
2026-10-21.

## 1. Toolchain

| Part | Version | Note |
|---|---|---|
| Python | 3.14.5 (`py -3.14`) | python.org install manager |
| pywebview | 6.2.1 (`>=6.2.1,<7`) | `gui="edgechromium"`, `private_mode=True` |
| pythonnet | 3.2.1 | WebView2 backend of pywebview |
| PyInstaller | 6.22.3 (`>=6.22`) | one-file, windowed, no UPX |
| Pillow | 13.0.0 (`>=12`) | icon generator only |

Runtime requirement on the target machine: Microsoft Edge WebView2 Runtime (preinstalled on Windows 11 and current
Windows 10). The exe needs no Python.

Requirements files: `requirements.txt` (runtime: pywebview, pythonnet), `requirements-dev.txt` (PyInstaller,
Pillow). `pyproject.toml` carries the project metadata (`src` layout, `setuptools`, version read from
`frame_notes.__version__`); the build does not install the package, it puts `src` on `PYTHONPATH` instead.
`src/frame_notes/core` is stdlib-only.

## 2. Development run

```powershell
powershell -ExecutionPolicy Bypass -File build\build.ps1 -SkipBuild    # creates .venv and installs both requirements files
$env:FRAME_NOTES_HOME = "$env:TEMP\frame_notes_scratch"
scripts\run_dev.bat [--ui-dir PATH] [path\to\video.mp4]
```

`scripts\run_dev.bat` sets `PYTHONPATH=src` and runs `.venv\Scripts\python -m frame_notes.shell.main --debug %*`
(`py -3.14` when `.venv` does not exist). `FRAME_NOTES_HOME` is left to the caller: without it the data folder is
`frame-notes_data\` in the project root. Flags:

- `--debug`: WebView2 dev tools (F12), DEBUG logging on the console (always set by `run_dev.bat`).
- `--ui-dir PATH`: serve the UI from another folder (a folder with an `index.html`).
- A positional path opens that video once the page has called `ready()` (through the `open_request` event).

Local server: `http://127.0.0.1:<port>/`, the first free port from 42011 to 42030 (`shell\server.py`); the window
loads that address, so the page, the video (`/media/<token>/<name>`) and the note images (`/image/<token>/<name>`)
share one origin. Logs: `<data folder>\logs\app.log` (rotating, 1 MB x 3). A development run also checks GitHub for
a newer release (the updater's normal behaviour); applying an update works only in the packaged exe.

UI alone in a browser with a mock bridge: `python -m http.server 5173 --directory src\frame_notes\ui`, then
`http://localhost:5173/?mock=1` (the mock serves a short sample clip and a few notes).

## 3. Tests

```powershell
$env:FRAME_NOTES_HOME = "$env:TEMP\frame_notes_scratch"
$env:PYTHONPATH = "src"
.venv\Scripts\python -m unittest discover -s tests -t .                          # everything
.venv\Scripts\python -m unittest discover -s tests -t . -p "test_shell_*.py"     # shell only
```

Core tests cover timecode conversion (non-drop and drop-frame values checked against After Effects), the notes
file (round trip, foreign lines kept, header checks), the probe, settings, paths, the session and the updater
configuration. Shell tests never open a real window: `test_shell_server.py` runs the HTTP server on loopback
(static files, path traversal, `Range` / `HEAD`, token invalidation, images); `test_shell_bridge.py` runs the
bridge against a fake session, a fake window and a fake updater (event buffering and coalescing, JS escaping, error
messages, base64 image decoding, state composition, self-update wiring). Each test module also puts `src` on
`sys.path`, so a plain `python -m unittest discover -s tests -t .` works without `PYTHONPATH`.

## 4. Build

```powershell
powershell -ExecutionPolicy Bypass -File build\build.ps1              # venv, deps, tests, icon, exe, SHA256SUMS
powershell -ExecutionPolicy Bypass -File build\build.ps1 -SkipTests
powershell -ExecutionPolicy Bypass -File build\build.ps1 -SkipBuild   # only the venv and the dependencies
```

Steps: create `.venv` (`py -3.14`) and install both requirements files; run the whole suite with a scratch
`FRAME_NOTES_HOME`; `build\make_icon.py` writes `build\icon.ico` (sizes 16 to 256; uses
`src\frame_notes\ui\assets\icon-1024.png` instead of the drawn marker when that file exists); PyInstaller with
`build\FrameNotes.spec` (output `build\dist`, work files `build\work`); copy to `release\FrameNotes.exe`; write
`release\SHA256SUMS`.

Output in `release\` (ignored): `FrameNotes.exe`, `SHA256SUMS` (`<sha256>  FrameNotes.exe`, two spaces).

`build\FrameNotes.spec` details: entry `build\entry.py` (calls `frame_notes.shell.main.main()`); `src` on
`pathex`; `src\frame_notes\ui` bundled as data under `frame_notes/ui` (the frozen app reads it from
`sys._MEIPASS/frame_notes/ui`, resolved in `main.resolve_ui_dir`); `build\icon.ico` bundled at the root of the
archive and used as the exe icon; `collect_submodules("frame_notes")` covers `core`, `shell` and
`vendor.selfupdate`; Windows version resource written to `build\work\_version_info.txt` from `__version__`; the spec
stops when `src\frame_notes\ui\index.html` is missing.

Verified 2026-10-21 (the UI folder did not exist yet): the spec was run on a scratch copy of the tree with a
placeholder page as the UI: PyInstaller 6.22.3 finished in about 8 s, `FrameNotes.exe` is 11.6 MB, starts, serves the
bundled page from the extraction folder, answers `ready()`, opens the video given on the command line, and ends
cleanly when the window is closed (settings saved, process gone). `build\build.ps1` as a whole is verified up to the
environment step (`-SkipBuild`); its tests, icon and PyInstaller steps were run separately by hand.

## 5. Release

Notes in `docs/releases/vX.Y.Z.md` in the layout of `docs/releases/README.md`, version bump commit `Release vX.Y.Z`, tag `vX.Y.Z` on `dev`, GitHub Release titled `vX.Y.Z`
with `FrameNotes.exe` and `SHA256SUMS` attached. A finished minor or patch version is released as soon as tests,
build, checklist and notes are in order; a major version waits for an explicit decision.

## 6. Troubleshooting

Symptom / cause / fix entries for this project.

### The frozen exe shows two processes
- Symptom: Task Manager lists two `FrameNotes.exe`; ending the first one leaves the window running.
- Cause: the PyInstaller one-file bootloader is the parent process, the app runs in the child.
- Fix: none needed; closing the window ends both. When scripting a close, send it to the process that has the
  window title.
