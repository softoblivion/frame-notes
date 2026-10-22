# Architecture

Frame Notes is a Windows desktop utility for reviewing a rendered video and logging change requests at exact
frames. The reviewer opens a video, plays or steps through it, pauses on a frame and presses one key per kind of
request. Every note is stored with an After Effects style timecode in a plain text file inside a notes folder next
to the video, so the editor can jump to each frame in After Effects or Premiere without conversion.

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
| `src/frame_notes/shell/` | `main.py` (entry point, window, single instance), `bridge.py` (JS <-> Python API and event pump), `winutil.py` (Explorer, clipboard, message box, screen and work-area metrics) |
| `src/frame_notes/ui/` | `index.html`, `css/`, `js/`, `assets/` (icon, embedded fonts); served by pywebview's local HTTP server |
| `tests/` | `unittest` suite, mirrors `src/frame_notes/` |
| `build/` | `build.ps1`, `FrameNotes.spec`, `entry.py`, icon generator |
| `scripts/` | `run_dev.bat` and other development helpers, not shipped |
| `docs/` | this file, module docs, `BUILD.md`, `releases/`, `decisions/`, `assets/` |
| `release/` | build output, ignored |

## 3. Modules

| Module | Path | Owning doc | Owner (v1.0.0 build) |
|---|---|---|---|
| Core: timecode, notes file, probe, session, settings, paths | `src/frame_notes/core/`, `src/frame_notes/__init__.py`, `tests/test_core_*.py` | `docs/CORE.md` | core worker |
| Shell: local server, window, bridge, Windows helpers | `src/frame_notes/shell/`, `tests/test_shell_*.py`, `build/`, `scripts/`, `requirements*.txt`, `pyproject.toml` | `docs/BRIDGE_API.md` (the only definition of the JS <-> Python contract) | shell worker |
| UI: player, region tool, timeline, notes list, keyboard | `src/frame_notes/ui/` | `docs/UI.md` | UI worker |
| Self-update | `src/frame_notes/vendor/selfupdate/` (vendored, never edited here; synced by the integrator), configuration in `core/selfupdate.py` | section 10 here; the library's own docs | integrator (vendor), core worker (configuration) |
| Build and release | `build/` | `docs/BUILD.md` | shell worker, then integrator |

The owning doc of a module defines its contract (public API, events, payload types, error cases). A contract
change is made in the doc first (`D:\Projects\_workspace\PARALLEL.md`); `CORE.md` and `BRIDGE_API.md` are written
by the integrator, `UI.md` by the UI worker from this file. This file is the behaviour specification.

## 4. Behaviour

### Opening a video

- Ways in: drag and drop onto the window, the **Open** button (native file dialog), a path as the first command
  line argument (Explorer "Open with", a shortcut), the recent-files list.
- On open the core probes the file (section 7), moves a 1.0 notes layout into the notes folder (section 6, "Life
  cycle"), looks for an existing data file and loads it; the UI shows the video, the timeline with the existing
  notes and the notes list. The last playback position of that file is restored from settings.
- One video at a time; opening another one replaces the current session (nothing is lost: the notes file is
  always saved).

### Recent videos

- The recent list (at most 20) shows in the empty state and, while a video is open, in the header's **Recent**
  menu. An entry shows the name, the folder, the last position and how many notes and frame images it has
  (read from its data file; nothing when it has no notes folder). A click opens it.
- A right click (or the context menu key) on an entry opens a small menu: **Open**, **Reveal folder** (the notes
  folder selected in Explorer, or the video when there is no folder), **Delete notes** (section 6; the item turns
  into `Delete 3 notes and 2 frames?` and needs a second click; a toast confirms), **Remove from recent** (only
  the list entry; files stay). The menu works for the video that is open too: deleting its notes reloads it empty.
  A missing video keeps Reveal folder and Delete notes when its notes folder still exists.

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
| `Ctrl+C` (no text field focused) | copy the data file text to the clipboard |
| `Ctrl+Shift+C` (no text field focused) | copy the readable text (section 6) to the clipboard |
| `Ctrl+O` | open a video |

Pressing `1` / `2` / `3` during playback pauses first, then starts a note at the paused frame. Two notes may
share a frame. Notes are sorted by frame, then by creation order.

### Pointing at the frame (region tool)

A note can carry one region that marks the element the note is about, so the reviewer does not have to describe
it in words.

1. After `1` / `2` / `3` the video is paused, the cursor over the video becomes a crosshair and a hint reads
   "Drag to mark a region, click to put a point, or type the note".
2. Drag draws a rectangle (shown in the note type's colour, 2 px, with a dimmed frame outside it); a click without
   a drag puts a point marker (a small circle with a short tail). One region per note; a second drag replaces
   the first. Regions are stored in fractions of the frame (0..1 of width and height), independent of the
   displayed size.
3. The text field opens next to the region (or below the video when there is none), focused. `Enter` saves (empty
   text allowed; a note may consist of a region alone), `Esc` cancels the whole new note. While the field is
   focused the playback keys are plain characters.
4. On save with a region the UI captures the current frame from the `<video>` element into a canvas at the
   video's native size, draws the region on it the way it is shown on screen, encodes PNG and sends it with the
   note. The core writes it to the `frames\` folder (section 6). A note without a region gets no image.
5. Selecting a note (list, marker, `↑` / `↓`) seeks to its frame and shows its region over the video; the region
   stays visible while the playhead is on that frame. `F2` edits the text; the region of an existing note is
   replaced by dragging a new one while the note is being edited; `Backspace` in the empty text field of an edit
   removes the region.

### Timeline

- A horizontal bar across the window below the video: ruler with timecodes at a step chosen from the zoom
  level, playhead, one marker per note coloured by type (FIX amber, ADD green, REMOVE red) with a short label
  (the first words of the text, cut to the space available; hidden when markers collide). Hover shows a card
  with the timecode, the type, the full text and, for a note with a region, a thumbnail of its image. Click
  selects the note and seeks.
- Mouse wheel over the timeline zooms around the pointer, `Shift` + wheel pans; `0` resets the zoom to the whole
  video. Dragging in the ruler scrubs.
- The notes list (right side, collapsible) shows every note as `timecode  TYPE  text` with a thumbnail when the
  note has an image; the selected note is highlighted in both places.

### Saving

- The data file and the readable export are written on every change (add, edit, remove, frame rate) with an
  atomic replace; there is no Save button. The window title shows the video name; a failed write shows an error
  banner with the path and keeps the notes in memory until the next change succeeds.
- Removing the last note leaves the notes folder with a header-only data file so the reviewer sees that the review
  happened. A video with no notes gets no folder (section 6, "Life cycle").

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

Since 1.1.0 everything of one review lives in one folder next to the video, `<video name without extension>_notes\`
(`promo_v3.mp4` -> `promo_v3_notes\`); decision record `docs/decisions/0002-notes-folder.md`.

```
promo_v3.mp4
promo_v3_notes\
    promo_v3_notes.txt               data file: the source of truth, read back by the app
    promo_v3_notes.md                readable export: rewritten on every change, never read back
    frames\0-00-12-07_FIX.png        frame images of notes with a region
```

### Data file `<stem>_notes.txt`

UTF-8 without BOM, LF line endings, tab-separated columns.

```
# Frame Notes 1
# source: promo_v3.mp4
# fps: 25
# duration: 0:01:32:10
0:00:12:07	FIX	logo flickers for two frames	rect 0.540,0.360 0.880,0.800	0-00-12-07_FIX.png
0:00:40:00	ADD	subtitle for the second speaker
0:01:02:15	REMOVE	second take of the handshake	point 0.512,0.300	0-01-02-15_REMOVE.png
```

- Header lines start with `# `; the first line carries the format version. `source` is the file name (no path)
  the notes belong to; on open the app refuses to attach a notes file whose `source` names a different file
  (`promo_v3_notes\` may belong to `promo_v3.mov`).
- Note lines: `<timecode>\t<TYPE>\t<text>[\t<region>[\t<image>]]`; `TYPE` is `FIX`, `ADD` or `REMOVE`; text is one
  line (tabs and line breaks typed by the user are replaced by spaces). `region` is `rect x0,y0 x1,y1` or
  `point x,y` in fractions of the frame with three decimals; `image` is the file name only, inside `frames\`:
  `<timecode with '-' instead of ':' or ';'>_<TYPE>.png`, with `_2`, `_3` added when the name is taken. Lines
  that do not parse are kept on save (after the notes, in their original order), so a hand-edited file is not
  damaged.
- The line format is the one of 1.0 and keeps format version 1: only the location of the files changed, which the
  migration below handles, so a 1.0 file is read as it is.
- The image shows the whole frame at the video's native size with the region drawn on it, so the editor sees
  what is meant without the app. Removing a note deletes its image; a `frames\` folder that becomes empty is
  removed.
- The timecode column is the one the editor types into the AE time field.

### Readable export `<stem>_notes.md`

Plain Markdown without tables, so it reads well raw (Notepad, a chat) and in any Markdown viewer. Written next to
the data file every time the data file is written; never parsed.

```
# promo_v3.mp4

25 fps · 0:01:32:10 · FIX 1 · ADD 1 · REMOVE 1

- **0:00:12:07** · FIX · logo flickers for two frames
  ![](frames/0-00-12-07_FIX.png)
- **0:00:40:00** · ADD · subtitle for the second speaker
- **0:01:02:15** · REMOVE · second take of the handshake
  ![](frames/0-01-02-15_REMOVE.png)
```

- Heading: the video file name. Summary line: frame rate, duration (left out when unknown), count per type.
- One list item per note in timecode order; a note with a frame image continues on an indented second line with
  the image, linked relative to the folder, so viewers show it inline right after the text. Note text is
  written as typed (no Markdown escaping). A review without notes reads `No notes.` under the summary. Lines of
  the data file that did not parse are not exported.
- The header **Copy** menu copies the data file text (`Ctrl+C`) or the readable text (`Ctrl+Shift+C`); the
  clipboard copy of the readable text leaves the image lines out, because relative links mean nothing in a chat.

### Life cycle

- A video with no notes and no notes folder gets nothing on disk; the folder with the data file and the export
  appears with the first note. Removing the last note keeps the folder with the header-only data file and the
  export (`No notes.`), so the editor sees that the review happened.
- Migration of the 1.0 layout (`<stem>_notes.txt` beside the video and / or images directly in `<stem>_notes\`)
  runs once, on open: the images directly in the folder move into `frames\` first, then the data file moves into
  the folder with one rename, then the export is written. Every step is a rename on the same volume, so a crash
  midway leaves readable files and the next open finishes the move. A legacy data file that does not parse or
  whose `source` names another video is left in place and the open reports it as in 1.0. When the folder already
  holds a data file, that file wins and a legacy file beside the video is left untouched (logged). Every migration
  is logged with what moved.
- **Delete notes** (Recent context menu) removes the whole `<stem>_notes\` folder of that video, and a legacy data
  file beside the video, after an inline confirmation. It refuses when the folder holds anything the app does not
  write (another file, a sub-folder other than `frames\`) or when the data file belongs to another video.

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
| `settings.json` | window position (x, y; the size is fixed, section 11; the keys `width`, `height` and `maximized` of older versions are dropped on load), last folder, recent files (path, last frame, fps override), `check_updates` (always on, kept for forward compatibility), `auto_download_updates`, `notes_panel_open` (a key from an older version, `update_check_interval`, is dropped on load) |
| `logs\app.log` | rotating application log |
| `updates\` | downloaded releases and `apply.log` of the self-updater |

The notes folder of a reviewed video (section 6) is the only thing written outside this folder. pywebview runs in `private_mode`;
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

Recipe 1 of `RECIPES.md`: check on every launch in the background and then every 5 minutes while the app
runs (fixed by the library, no setting; a failed check is retried after 10, 20, 40 and at most 60 minutes, and the
timer skips a round while Windows reports no network connection), download to the data folder on the reviewer's
click (or automatically when `auto_download_updates` is on, the default), verify size and SHA-256 against `SHA256SUMS`, swap
the exe on **Restart**. Release assets are exactly `FrameNotes.exe` and `SHA256SUMS`; tags `vX.Y.Z`.

Implementation: the `selfupdate` package, vendored at `src/frame_notes/vendor/selfupdate/` (package
`frame_notes.vendor.selfupdate`, version in its `VENDORED.txt`). It is maintained outside this repository with its
own tests and shared with Media Saver; the copy here is never edited by hand, only replaced as a whole by the
maintainer's sync script (`vendor-sync.ps1 -Lib selfupdate -Project frame-notes`), followed by this project's tests
and build. It is stdlib-only and imports nothing outside itself.

Configuration (`UpdateConfig`, built in one place, e.g. `src/frame_notes/core/selfupdate.py`):

| Field | Frame Notes value |
|---|---|
| `repo` | `softoblivion/frame-notes` |
| `asset_name` | `FrameNotes.exe` (downloads: `updates\FrameNotes-<version>.exe`) |
| `app_name` | `Frame Notes` (messages: "Frame Notes is up to date", "Downloading Frame Notes X.Y.Z… 42%", toast "Frame Notes updated to vX.Y.Z") |
| `version` | `frame_notes.__version__` |
| `updates_dir` | `<data>\updates`, where `<data>` is `frame-notes_data` beside the exe (or `FRAME_NOTES_HOME`), from `core/paths.py` |
| `exe_path` | the running exe when frozen (`sys.executable`), `None` from source |
| `on_status` / `on_toast` | the bridge's `update` event / a toast |
| `release_url_env` | `FRAME_NOTES_RELEASE_URL` (end-to-end test against a local server) |
| `logger_name` | `frame_notes.selfupdate` |
| `legacy_files` | none |

Update notice (the update sheet): the first time in a run that the status is `available`, or `downloading` /
`ready` for a version not announced yet, the UI opens a modal sheet over the window with the new version, the
installed version, size and date, the release notes rendered from Markdown by a small escaping renderer, and the
buttons **Install** and **Later**. **Install** at `available` calls `download_update()` and arms the install:
the sheet shows the progress and, once the file is verified (`ready`), calls `apply_update()` at once (the app has
no long-running work, so there is no restart warning). With automatic download the sheet may open while the
download runs or is done; the button then reads `Downloading… 42%` (disabled) and `Restart to update` (disabled
when `can_apply` is false, as when running from source). A failed download shows the message and **Try again**.
**Later**, `Esc` and a click on the dimmed area close the sheet and disarm the install; the sheet does not open by
itself again for the same version in that run, and the version chip opens it (instead of the popover) while a newer
version is known. The sheet never opens for `up_to_date` or after a failed check. It is UI only: the shell and
the bridge contract are unchanged (`docs/UI.md`, "Update sheet").

Entry point: `argv[0] == APPLY_FLAG` (`--apply-update`) goes to `run_apply(argv[1:])` before the single-instance
mutex and before any window. Calls: `startup()` once per launch, `check_background()` on launch and on the
button, the package's `UpdateScheduler(updater)` started after the launch check (`stop()` on exit),
`download_background()`, `apply()` (True -> shut down the way the close button does). The status payload
is the package's `UpdateStatus`, documented unchanged in the bridge contract. `build.ps1` writes
`release\SHA256SUMS` as `<sha256>  FrameNotes.exe`; the PyInstaller spec collects the `frame_notes` submodules,
which includes the vendored package.

## 11. Shell: window and local server

- The shell runs its own HTTP server on `127.0.0.1` (first free port from 42011) that serves both the UI folder
  and the opened video (`/media/<token>/<name>`, with HTTP range requests so the `<video>` element can seek).
  UI and video share one origin, so the UI may draw the video into a canvas and read the pixels back (a video
  from another origin would taint the canvas and block the frame capture). pywebview's own server is not used.
- One window with the native Windows frame (no custom title bar), dark UI, title `Frame Notes` plus the
  video name. Several instances may run at once: there is no single-instance lock, each window is its own review.
- Fixed window (the workspace rule of 2026-10-21, "Fixed window size by default"): `resizable=False`, which makes
  pywebview use the `FixedSingle` border and `MaximizeBox = False` (`webview/platforms/winforms.py`): no sizing
  border, no maximize button, and a double click on the title bar does not maximize. There is no `min_size` and
  no size or maximize handling in the shell. The size is 1280 x 800 logical pixels (the outer window; the client
  area is a little smaller), scaled down in whole steps when the work area of the monitor the window opens on
  (monitor minus taskbar, in logical pixels, from `winutil.work_area`) is smaller: the first of 1280x800,
  1152x720, 1024x640, 960x600 that fits wins, the last one otherwise (`choose_window_size` in `shell/main.py`,
  unit tested). Only the position is saved (`window.x`, `window.y` in `settings.json`) and restored; a saved
  position that would leave the window off every monitor is ignored (`restore_position`), as is a minimized
  window's position. The `width`, `height` and `maximized` keys of earlier versions are ignored and dropped.
- Python -> JS events go through one pump thread per `RECIPES.md` section 4; the `js_api` object exposes only the
  contract methods of `docs/BRIDGE_API.md`.
- Exit saves the window position and the last frame, then `os._exit(0)` after the clean shutdown.

## 12. Data flow

```
video file ──open──> core.probe ──fps, duration, size──┐
                     core.notes.parse(<name>_notes.txt) ┤──> core.session (notes in memory, sorted)
                                                        │          │ every change: data file + export + PNG
UI <video> + timeline <──event state ── bridge <──> core.session.save (atomic replace)
UI keys 1/2/3, region, text ──calls (add_note, update_note, remove_note)──> bridge ──> session
```

The UI holds no state of its own beyond the playhead, the zoom and the note being edited: the notes list it
renders is the one the session sent after the last change.

## 13. Version

`src/frame_notes/__init__.py`, `__version__ = "1.4.0"`; build script and UI read it from there. SemVer; the first
release is 1.0.0 (the user's rule for every project): from there minor = features, patch = fixes, major =
breaking change for users (notes file format, key bindings).
