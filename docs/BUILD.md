# Build, run and troubleshoot

Toolchain and versions follow the pywebview family baseline of `D:\Projects\_workspace\RECIPES.md`, section 4;
the versions actually used are recorded here once the first build runs.

## 1. Toolchain

| Part | Version | Note |
|---|---|---|
| Python | 3.14 (`py -3.14`) | python.org install manager |
| pywebview | 6.2.1 (`>=6.2.1,<7`) | `gui="edgechromium"` |
| pythonnet | 3.2.1 | WebView2 backend of pywebview |
| PyInstaller | 6.22.3 (`>=6.22`) | one-file, windowed |
| Pillow | `>=12` | icon generator only |

Runtime requirement on the target machine: Microsoft Edge WebView2 Runtime (preinstalled on Windows 11 and current
Windows 10). The exe needs no Python.

Requirements files: `requirements.txt` (runtime: pywebview, pythonnet), `requirements-dev.txt` (PyInstaller,
Pillow). `src/frame_notes/core` is stdlib-only.

## 2. Development run

```powershell
py -3.14 -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt -r requirements-dev.txt
$env:FRAME_NOTES_HOME = "$env:TEMP\frame_notes_scratch"
scripts\run_dev.bat [path\to\video.mp4]
```

`scripts\run_dev.bat` runs `python -m frame_notes.shell.main --debug %*` with `src` on `PYTHONPATH`. Flags:

- `--debug`: WebView2 dev tools (F12), DEBUG logging on the console.
- `--ui-dir PATH`: serve the UI from another folder.
- A positional path opens that video at start.

UI alone in a browser with a mock bridge: `python -m http.server 5173 --directory src\frame_notes\ui`, then
`http://localhost:5173/?mock=1` (the mock serves a short sample clip and a few notes).

## 3. Tests

```powershell
$env:FRAME_NOTES_HOME = "$env:TEMP\frame_notes_scratch"
.venv\Scripts\python -m unittest discover -s tests
```

Core tests cover timecode conversion (non-drop and drop-frame values checked against After Effects), the notes
file (round trip, foreign lines kept, header checks), the probe (fixture MP4 / WebM headers in `tests/fixtures/`),
settings and paths. Shell tests run the bridge against a fake window and never open a real one.

## 4. Build

```powershell
powershell -ExecutionPolicy Bypass -File build\build.ps1              # venv, deps, tests, icon, exe, SHA256SUMS
powershell -ExecutionPolicy Bypass -File build\build.ps1 -SkipTests
```

Output in `release\` (ignored): `FrameNotes.exe`, `SHA256SUMS` (`<sha256>  FrameNotes.exe`, two spaces).

## 5. Release

Per `D:\Projects\_workspace\CONVENTIONS.md`, "Versioning and releases": notes in `docs/releases/vX.Y.Z.md` in the
shared template, version bump commit `Release vX.Y.Z`, tag `vX.Y.Z` on `dev`, GitHub Release titled `vX.Y.Z`
with `FrameNotes.exe` and `SHA256SUMS` attached. A finished minor or patch version is released as soon as tests,
build, checklist and notes are in order; a major version waits for an explicit decision.

## 6. Troubleshooting

Symptom / cause / fix entries specific to this project; cross-project ones go to
`D:\Projects\_workspace\NOTES.md`.
