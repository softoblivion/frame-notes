# Architecture

Frame Notes is a Windows desktop utility for reviewing a rendered video and logging change requests at exact
frames. The reviewer opens a video, plays or steps through it, pauses on a frame and presses one key per kind of
request. Every note is stored with an After Effects style timecode in a plain text file next to the video, so the
editor can jump to each frame in After Effects or Premiere without conversion.

## 1. Stack

- Python 3.14 (`py -3.14`), stdlib-only core.
- UI: HTML / CSS / JS (ES modules) in pywebview 6.2.1 on WebView2 (`gui="edgechromium"`), pythonnet 3.2.1.
  Video playback uses the browser `<video>` element of WebView2 (H.264 / AAC in MP4, VP9 / AV1 in WebM; HEVC and
  ProRes do not play, see section 9).
- Packaging: PyInstaller 6.22.3, one-file windowed exe `FrameNotes.exe`, about 12 MB; needs the Microsoft Edge
  WebView2 Runtime on the target machine (preinstalled on Windows 11).
- Recipes followed: `D:\Projects\_workspace\RECIPES.md` sections 1 (self-update) and 4 (pywebview UI). Section 3
  (managed ffmpeg) is planned for proxies of unsupported codecs, not part of the first version.

## 2. Layout

| Path | Holds |
|---|---|
| `src/frame_notes/__init__.py` | `__version__` (the only place of the version) |
| `src/frame_notes/core/` | stdlib-only engine: `paths.py`, `settings.py`, `timecode.py`, `notes.py`, `probe.py`, `session.py` |
| `src/frame_notes/shell/` | `main.py` (entry point, window, single instance), `bridge.py` (JS <-> Python API and event pump), `winutil.py` (Explorer, clipboard, DWM, frameless resize) |
| `src/frame_notes/ui/` | `index.html`, `css/`, `js/`, `assets/` (icon, embedded fonts); served by pywebview's local HTTP server |
| `tests/` | `unittest` suite, mirrors `src/frame_notes/` |
| `build/` | `build.ps1`, `FrameNotes.spec`, `entry.py`, icon generator |
| `scripts/` | `run_dev.bat` and other development helpers, not shipped |
| `docs/` | this file, module docs, `BUILD.md`, `releases/`, `decisions/`, `assets/` |
| `release/` | build output, ignored |

## 3. Modules

| Module | Path | Owning doc | Owner |
|---|---|---|---|
| Core: timecode, notes file, probe, session, settings, paths | `src/frame_notes/core/` | `docs/CORE.md` | integrator |
| Shell: window, bridge, Windows helpers | `src/frame_notes/shell/` | `docs/BRIDGE_API.md` (the only definition of the JS <-> Python contract) | integrator |
| UI: player, timeline, notes list, keyboard | `src/frame_notes/ui/` | `docs/UI.md` | integrator |
| Self-update | shared library (section 10) | the library's own doc | integrator |
| Build and release | `build/` | `docs/BUILD.md` | integrator |

The owning doc of a module defines its contract (public API, events, payload types, error cases). A contract
change is made in the doc first (`D:\Projects\_workspace\PARALLEL.md`). Module docs are written together with the
module; until then this file is the specification.

## 4. Behaviour

### Opening a video

- Ways in: drag and drop onto the window, the **Open** button (native file dialog), a path as the first command
  line argument (Explorer "Open with", a shortcut), the recent-files list.
- On open the core probes the file (section 7), looks for an existing notes file (section 6) and loads it; the UI
  shows the video, the timeline with the existing notes and the notes list. The last playback position of that
  file is restored from settings.
- One video at a time; opening another one replaces the current session (nothing is lost: the notes file is
  always saved).

### Playback and stepping

| Key | Action |
|---|---|
| `Space` | play / pause |
| `←` / `→` | one frame back / forward (pauses) |
| `Shift+←` / `Shift+→` | ten frames |
| `Home` / `End` | first / last frame |
| `J` / `K` / `L` | reverse / pause / forward, repeated presses raise the speed (editor habit; optional in the first version) |
| click on the timeline | seek to that frame; drag scrubs |

Frame accuracy: the UI seeks to the centre of the target frame (`(frame + 0.5) / fps`) and derives the current
frame as `floor(currentTime * fps + 1e-4)`; both formulas live in one JS module and mirror `core/timecode.py`.
The time display shows the AE timecode of the current frame and the total duration.

### Notes

| Key | Action |
|---|---|
| `1` | add a **FIX** note at the current frame (something is wrong and must be corrected) |
| `2` | add an **ADD** note (something is missing and must be added) |
| `3` | add a **REMOVE** note (something must be taken out) |
| `Enter` | save the text of the note being edited (empty text is allowed) |
| `Esc` | cancel: a just-created note is removed, an edit of an existing note is reverted |
| `F2` or double click | edit the text of the selected note |
| `Delete` | remove the selected note |
| `↑` / `↓` | select the previous / next note and seek to its frame |
| `Ctrl+C` (no text field focused) | copy the notes file text to the clipboard |
| `Ctrl+O` | open a video |

Pressing `1` / `2` / `3` during playback pauses first, then adds the note at the paused frame. A new note opens a
one-line text field focused for typing; while that field is focused the playback keys are plain characters. Two
notes may share a frame. Notes are sorted by frame, then by creation order.

### Timeline

- A horizontal bar across the window below the video: ruler with timecodes at a step chosen from the zoom
  level, playhead, one marker per note coloured by type (FIX amber, ADD green, REMOVE red). Hover shows the
  timecode and text; click selects the note and seeks.
- Mouse wheel zooms around the pointer, `Shift` + wheel pans; `0` resets the zoom to the whole video.
- The notes list (right side, collapsible) shows every note as `timecode  TYPE  text`; the selected note is
  highlighted in both places.

### Saving

- The notes file is written on every change (add, edit, remove) with an atomic replace; there is no Save button.
  The window title shows the video name; a failed write shows an error banner with the path and keeps the notes in
  memory until the next change succeeds.
- Removing the last note leaves an empty notes file (header only) so the reviewer sees that the review happened.
  A video with no notes file and no notes gets no file.

## 5. Timecode

Rules follow After Effects timecode display (Project Settings > Time Display Style > Timecode, footage timecode
base from the frame rate):

- Format `h:mm:ss:ff`: hours without padding, minutes and seconds two digits, frames two digits (three when the
  frame rate is 100 or more). Example: `0:00:12:07`, `1:02:03:24`.
- Non-drop-frame for integer rates and for 23.976 (`:` separators, frames count 0 to round(fps) - 1).
- Drop-frame for 29.97 and 59.94 (`;` separators, `0;00;12;07`), SMPTE rule: two frame numbers (four at 59.94)
  skipped at the start of every minute except every tenth. After Effects applies drop-frame to these rates by
  default, so the timecodes match what the editor sees in the AE timeline.
- The rate used for the timecode base is the probed frame rate rounded to the nearest of 23.976, 24, 25, 29.97,
  30, 48, 50, 59.94, 60, 100, 120; other rates use `round(fps)` as the base without drop-frame. The reviewer can
  override the rate from the header control (needed for files whose container lies about the rate).
- Frame numbers are zero-based internally; the first frame displays as `0:00:00:00` (AE comp start time 0).
- `core/timecode.py` owns frame <-> seconds <-> timecode conversion with unit tests against known AE values; no
  other module does frame math.

## 6. Notes file

Written next to the video: `<video name without extension>.notes.txt` (`promo_v3.mp4` -> `promo_v3.notes.txt`).
UTF-8 without BOM, LF line endings, tab-separated columns.

```
# Frame Notes 1
# source: promo_v3.mp4
# fps: 25
# duration: 0:01:32:10
0:00:12:07	FIX	logo flickers for two frames
0:00:40:00	ADD	subtitle for the second speaker
0:01:02:15	REMOVE	second take of the handshake
```

- Header lines start with `# `; the first line carries the format version. `source` is the file name (no path)
  the notes belong to; on open the app refuses to attach a notes file whose `source` names a different file and
  offers to start a new one (`promo_v3.notes.txt` may belong to `promo_v3.mov`).
- Note lines: `<timecode>\t<TYPE>\t<text>`; `TYPE` is `FIX`, `ADD` or `REMOVE`; text is one line (tabs and line
  breaks typed by the user are replaced by spaces). Lines that do not parse are kept as-is on save, in place, so a
  hand-edited file is not damaged.
- The file is readable without the app and pastes into a chat as it is. The timecode column is the one the
  editor types into the AE time field.

## 7. Probe (frame rate, duration, size)

`core/probe.py` reads the container with the stdlib only:

- MP4 / MOV (`ftyp`, `moov`): `mvhd` timescale and duration; the video `trak` -> `mdhd` timescale, `stts` entries
  -> frame count; fps = timescale / sample delta, duration = frame count / fps; `tkhd` width and height.
- WebM / Matroska (EBML): `TimecodeScale`, `Duration`, the video track's `DefaultDuration`, `PixelWidth` /
  `PixelHeight`.
- Anything else, or a container without the fields: fps unknown. The UI then measures the rate from
  `requestVideoFrameCallback` presentation times over the first second of playback and asks the reviewer to confirm
  or pick one from the list in section 5.

The probe never decodes video. `ffprobe` is not required; when it is present on `PATH` it is used as a second
opinion for the frame rate (future option, not in the first version).

## 8. Runtime data

Everything the app writes for itself lives in `frame-notes_data\` beside `FrameNotes.exe` (the project root when
run from source), resolved by `core/paths.py` only; `FRAME_NOTES_HOME` overrides the folder (tests and manual runs
always set it to a scratch location).

| Path | Content |
|---|---|
| `settings.json` | window size and position, last folder, recent files (path, last frame, fps override), `check_updates` (always on, kept for forward compatibility), `auto_download_updates` |
| `logs\app.log` | rotating application log |
| `updates\` | downloaded releases and `apply.log` of the self-updater |

The notes file (section 6) is the only file written outside this folder. pywebview runs in `private_mode`;
no WebView2 profile folder is kept.

## 9. Video formats

WebView2 plays what Edge plays: MP4 with H.264 or AV1 video and AAC audio, WebM with VP8 / VP9 / AV1 and Opus /
Vorbis, Ogg. Not played: HEVC (H.265) without the Windows HEVC extension, ProRes, DNxHD, MOV containers with
uncompressed or PCM-only tracks. Review renders from After Effects and Premiere are normally H.264 MP4, which is
the first version's target.

Planned (not in the first version): a managed `ffmpeg.exe` per `RECIPES.md` section 3 that transcodes an
unsupported file into a proxy in `frame-notes_data\proxies\` on open, with the original's frame rate kept for the
timecodes.

## 10. Self-update

Recipe 1 of `RECIPES.md`: check on every launch in the background (`HEAD .../releases/latest`), download to the
data folder on the reviewer's click (or automatically when `auto_download_updates` is on), verify size and
SHA-256 against `SHA256SUMS`, swap the exe on **Restart**. Release assets are exactly `FrameNotes.exe` and
`SHA256SUMS`.

The implementation exists in Media Saver (`app/core/selfupdate.py` with its `Net`, checksum and atomic-write
helpers). Workspace rule (`CONVENTIONS.md`, "Workspace folders"): code needed by a second project moves to
`_shared\<lib>\` as a project of its own and is consumed as a dependency, never copied. Decision pending with the
user: extract the updater into `D:\Projects\_shared\selfupdate\` (MIT), parametrised by repository, asset name,
app name, data folder and version, and make Media Saver depend on it too. Until that library exists the update
feature is not built here.

## 11. Data flow

```
video file ──open──> core.probe ──fps, duration, size──┐
                     core.notes.load(<name>.notes.txt) ─┤──> core.session (notes in memory, sorted)
                                                        │          │ every change
UI <video> + timeline <──events (session, notes)── bridge <──> core.notes.save (atomic replace)
UI keys 1/2/3, edits ──calls (add_note, edit_note, remove_note, seek)──> bridge ──> session
```

The UI holds no state of its own beyond the playhead and the zoom: the notes list it renders is the one the
session sent after the last change. Python -> JS events go through one pump thread per `RECIPES.md` section 4.

## 12. Version

`src/frame_notes/__init__.py`, `__version__ = "0.1.0"`; build script and UI read it from there. SemVer, pre-1.0:
minor = features, patch = fixes.
