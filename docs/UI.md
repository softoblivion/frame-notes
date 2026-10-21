# UI

Owner: UI worker (v1.0.0 build), then integrator. Scope: everything under `src/frame_notes/ui/`. Behaviour comes
from `ARCHITECTURE.md` sections 4, 5, 9 and 11; the calls and events are the contract of `BRIDGE_API.md`; data
shapes are `Session.payload()` of `CORE.md`. Plain HTML / CSS / ES modules, no build step, no framework, no
network resources, system fonts only (`Segoe UI`, `Cascadia Mono` / `Consolas` for timecodes). Dark theme only.

## 1. Files

| File | Holds |
|---|---|
| `index.html` | the whole DOM: header, banner, stage (video, overlay, note editor, hint, empty state), transport, notes panel, timeline, toasts, drop veil, key help |
| `css/tokens.css` | colours (surfaces, ink, accent, note types), fonts, radii, fixed sizes |
| `css/base.css` | reset, buttons, badges, `kbd`, menus, toasts, error banner |
| `css/layout.css` | window layout, header (fps control, counts, version / update chip and popover), drop veil, key help |
| `css/viewer.css` | stage, overlay, floating note editor, hint line, empty state, transport row |
| `css/timeline.css` | ruler, track, markers, playhead, scroll bar, hover card |
| `css/panel.css` | notes panel |
| `js/main.js` | entry point: builds the modules, applies `state`, transport, overlay refresh, bridge events |
| `js/bridge.js` | `api.<call>()`, `on(event, fn)`, `window.frameNotes.dispatch`, `connect()`; loads `mock.js` with `?mock=1` |
| `js/mock.js` | in-browser fake of the shell (section 7) |
| `js/app.js` | UI state hub (section 3), note type colours and keys, small DOM helpers |
| `js/frames.js` | frame math and timecode formatting, mirror of `core/timecode.py` |
| `js/player.js` | video element: frame-accurate seek and step, play / pause, live counter, rate measurement |
| `js/overlay.js` | SVG over the displayed frame, region drawing (drag = rect, click = point) |
| `js/region.js` | region geometry and drawing shared by the overlay (SVG) and the capture (canvas) |
| `js/capture.js` | frame image of a note: native-size frame plus region, PNG, base64 |
| `js/composer.js` | the note being created or edited: text field, region tool, save / cancel |
| `js/timeline.js` | ruler, markers, playhead, zoom / pan / scrub, hover card |
| `js/notes.js` | notes panel list, selection, inline edit host |
| `js/header.js` | open, file name and size, fps control, counts, Copy split menu, Reveal menu, Recent menu, Keys, panel toggle |
| `js/update.js` | version chip and update popover (with the check interval select) |
| `js/empty.js` | empty state, recent list, drag and drop |
| `js/recent.js` | recent entry rendering shared by the empty state and the header Recent menu, the entry context menu |
| `js/keys.js` | keyboard map |
| `js/ui.js` | toasts, error banner, `guard()` for calls, popover helper |
| `js/icons.js` | transport icons (inline SVG) |
| `assets/icon.svg` | page icon |

## 2. Layout

```
+-------------------------------------------------------------------------------------------+
| Open  promo_v3.mp4 1920x1080  [25 v] fps from file  FIX 1 ADD 0 REMOVE 2  Copy|v Reveal Recent Keys Notes v1.1.0 |
+-------------------------------------------------------------------------------------------+
| error banner (closable, only when there is an error)                                      |
+------------------------------------------------------------------+------------------------+
|                                                                  | Notes 3              > |
|                 video (letterboxed, object-fit: contain)         | 0:00:12:07 FIX   [img] |
|                 + SVG overlay over the displayed frame           | logo flickers ...      |
|                 + floating note editor / hint line               | ...                    |
+------------------------------------------------------------------+                        |
| |< <| > |> >|   0:00:12:07 / 0:01:32:10  frame 307    1 fix · 2 add · 3 remove |          |
+------------------------------------------------------------------+------------------------+
| ruler  0:00:00:00      0:00:10:00      0:00:20:00 ...                                     |
| track  |logo flickers...   |subtitle...    playhead                                       |
| scroll bar (only while zoomed)                                                            |
+-------------------------------------------------------------------------------------------+
```

- Header (44 px): `Open`; file name (ellipsis, full path in the tooltip) and muted `WxH`; fps select with the 11
  nominal rates (plus the current rate when it is not one of them) and a muted `from file` / `override` hint;
  `Reset` next to `override` calls `set_fps(null)`. When `fps_source` is `unknown` the control is outlined amber,
  the select shows `Set the frame rate`, and once playback has measured a rate a `Use 25` button appears. Counts per
  type (the type names hide below 1180 px). `Copy` split button (the main part copies the data file text; the
  caret opens a menu: `Notes file` with `Ctrl+C`, `Readable text` with `Ctrl+Shift+C`), `Reveal` menu (Video /
  Notes file / Frames), `Recent` menu (the recent entries of section 2a), `Keys`, `Notes` (panel toggle), and the
  version chip at the far right. Without a video only `Open`, `Keys` and the version show.
- Stage: `<video>` without controls fills the stage with `object-fit: contain`. The overlay `<svg>` is positioned
  to the displayed frame rectangle (`min(cw/vw, ch/vh)` scale, centred) and gets `viewBox = 0 0 w h`; the rectangle
  is recomputed by a `ResizeObserver` on the stage and on `loadedmetadata` / `resize` of the video, so the overlay
  follows window resizes, the banner and the panel toggle.
- Transport (48 px): first / previous / play-pause / next / last, current timecode (20 px mono), duration,
  `frame N` (hidden below 1100 px), and the key legend whose keys are also buttons.
- Notes panel (320 px, right of the viewer only): header with count and a hide button, one row per note
  (timecode, type badge, up to three lines of text, 72 x 41 thumbnail when `image_url` is set); selected row
  highlighted; empty list text explains the keys.
- Timeline (full window width): ruler 24 px, track 66 px (three marker rows), 6 px scroll bar while zoomed.
- Toasts: bottom centre above the timeline, 2.5 s, at most three. Error banner: under the header, closable,
  replaced by the next error.
- Empty state (no video): dashed drop target with `Open video` (and `Choose file` in the mock), `Ctrl+O` hint,
  recent list (section 2a).
- Note type colours: FIX `#EF9F27`, ADD `#639922`, REMOVE `#E24B4A` for strokes, stems and dots; labels use a light
  ink on a dark tint of the same hue (`--fix-ink` on `--fix-bg` and so on) for contrast on the dark UI.

## 2a. Recent entries and their context menu (`js/recent.js`)

- One renderer for both places: the empty state list and the header `Recent` menu (a popover of up to 20 rows,
  360 px wide; the open video's row is marked `open`). A row: name, folder (muted), position `frame N`, and a muted
  `3 notes · 2 frames` when `notes_count` is not `null` (`1 note`, frames part left out when 0, `no notes` for 0).
  A row whose video is missing (`exists: false`) is greyed; a click on it does nothing.
- Click or `Enter` on a row: `open_video(path)` (the header popover closes first).
- Right click, the context menu key or `Shift+F10` on a row: a context menu at the pointer (at the row for the
  keyboard), clamped into the window, above everything else; `Esc`, a click outside, a scroll or a window blur
  close it. Items:
  - `Open` (disabled when the video is missing);
  - `Reveal folder` -> `reveal_recent(path)` (disabled when the video is missing and `notes_dir` is `null`);
  - `Delete notes` (disabled when `notes_count` is `null`): the first click turns the item into a red
    `Delete 3 notes and 2 frames?` (`Delete the empty notes folder?` for 0 / 0) and keeps the menu open; the second
    click calls `delete_notes(path)` and applies the returned State (the shell sends the toast); leaving the item
    with the pointer for 3 s or pressing `Esc` resets it;
  - a separator, then `Remove from recent` -> `forget_recent(path)`, applies the returned State.
- Both calls go through `guard()`; an open note edit is saved before deleting the open video's notes.

## 3. State model

The UI renders what the shell sent and keeps only what the shell does not know.

| From `state` (never edited in the UI) | Kept by the UI (`js/app.js`) |
|---|---|
| `video` (name, size, url, fps, `fps_source`, `duration_frames`, `last_frame`) | `frame`: current frame; `time`: current time (display while the rate is unknown) |
| `notes` with `timecode`, `region`, `image_url` | `playing` |
| `counts`, `recent`, `settings`, `version` | `selectedId` (dropped when the note disappears or another video opens) |
| `timecode` (not used for display; the base is derived from `video.fps`) | `edit`: the note being created or edited (below) |
| | timeline zoom `{start, span}` (in `Timeline`), `panelOpen` (mirrors `settings.notes_panel_open`), `update` (last `UpdateStatus`) |

- Every `state` event and every call that returns a `State` (`open_video`, `set_fps`, `set_setting`) goes through
  `applyState()`: store it, derive `tb` / `fps` (`timecodeBase(video.fps)`, `null` when `fps_source` is `unknown`),
  load the video when `url` changed (seek to `last_frame` on `loadedmetadata`), re-render. A new video path cancels
  an open edit and clears the selection.
- After `add_note` resolves, the returned note id becomes the selection; the list and markers update from the
  following `state` event.
- `set_last_frame(frame)` is sent 600 ms after the playhead settles while paused.
- `app.edit` = `{mode: "create" | "edit", id?, kind, frame, region, origRegion, origText, regionDirty,
  host: "video" | "row"}`. One `<input>` element is moved between the floating editor (`host: "video"`) and the
  edited row of the panel (`host: "row"`; used by double click, and by `F2` while the panel is open).

## 4. Frame logic (`js/frames.js`, `js/player.js`)

- `timecodeBase(fps)`: snap to the nearest nominal rate within 0.5 %; exact rate 24000/1001, 30000/1001,
  60000/1001 for 23.976 / 29.97 / 59.94; base `round(nominal)`; drop-frame for 29.97 and 59.94; three frame digits
  at 100 and above. `formatTimecode` implements the drop-frame formula of `CORE.md` literally.
- Current frame `floor(currentTime * fps + 1e-4)`, clamped to `duration_frames - 1`; seek target
  `(frame + 0.5) / fps`.
- `seekFrame(f)` pauses, sets `currentTime`, shows `f` at once, and resolves with the frame read back after
  `seeked` (2.5 s safety timeout). Steps during a seek in flight are relative to the pending target, so a held
  arrow key never skips or repeats frames.
- Playback: `requestVideoFrameCallback` drives the counter from `metadata.mediaTime` (the frame actually
  presented); without it a `requestAnimationFrame` loop reads `currentTime`. Pause snaps to the last presented
  frame and seeks to its centre.
- Rate unknown: stepping uses 30 fps, the counter shows `h:mm:ss.mmm`, notes cannot be started (toast `Set the
  frame rate to add notes`, the fps control flashes and takes focus). During playback 25 presentation times are
  collected; the lower-quartile delta gives the rate, snapped to a nominal one, offered as `Use <rate>`.
- The timecode stored with a note always comes from the payload (`note.timecode`); the JS formatter is used only
  for the live counter, the duration, the ruler and the mock.

## 5. Note flow (`js/composer.js`, `js/overlay.js`, `js/capture.js`)

1. `1` / `2` / `3` (or the legend buttons): pause and wait for the settled frame, clear the selection, open the
   floating editor (type badge, timecode, focused field) above the bottom of the frame, show the hint line
   `Drag to mark a region, click to put a point, or type the note` and the crosshair over the stage.
2. Drag (more than 4 px) draws a rect, live, in the type colour (2 px, outside dimmed to 45 %); a click puts a point
   (8 px ring, short tail to the upper right, dark halo). Pointer events are taken on the whole stage and clamped
   to the frame, so a drag may start in the letterbox. A new drag or click replaces the region. The editor moves
   next to the region (right, else left, else below, else above; clamped into the stage).
3. `Enter` saves (empty text allowed), `Esc` cancels the new note, a pointer press outside the stage, the editor and
   the edited row saves. While the field has focus every key is text.
4. Save with a region: the frame (re-seeked first if the playhead moved) is drawn synchronously into an
   `OffscreenCanvas` at `videoWidth x videoHeight`, the region is drawn with the same constants scaled by
   `videoWidth / displayed width`, then encoded with `convertToBlob('image/png')` and sent as base64 in
   `add_note({frame, kind, text, region, image_png})`. Without a region `image_png` is `null`. If the frame cannot
   be read (a cross-origin video taints the canvas) the note is saved without an image and the banner says so.
5. Editing (`F2` / double click): seeks to the note's frame, shows its region, field prefilled. A drag or click
   replaces the region; `Backspace` in the empty field removes it. `update_note` gets `text` only when it changed,
   `region` (and a new `image_png` for a non-null region) only when the region changed; nothing is sent when nothing
   changed.
6. Regions are `{shape: "rect", x0, y0, x1, y1}` normalised (`x0 <= x1`, `y0 <= y1`) or `{shape: "point", x0: x,
   y0: y, x1: x, y1: y}`, fractions of the frame rounded to three decimals.
7. The selected note's region is drawn over the video while the playhead is on its frame and playback is paused.

## 6. Timeline (`js/timeline.js`)

- View `{start, span}` in seconds; minimum span 12 frames; reset to the whole video on `loadedmetadata` and on `0`.
- Ruler step: the smallest of 1, 2, 5, 10 frames, 1, 2, 5, 10, 30 s, 1, 2, 5, 10, 30, 60 min (in timecode
  seconds, i.e. multiples of the base) whose spacing fits a label plus 22 px; labels are timecodes; minor ticks at
  1/5 or 1/2 of the step when at least 6 px apart. Rate unknown: steps in seconds, clock labels.
- Markers: notes grouped by frame; one stem per frame, one tag per note in rows of 18 px (three rows; the third
  shows `+n` when more notes share the frame). Tag label = first six words (or the type when empty), clipped with
  an ellipsis to the space before the next frame's stem, a 7 px colour chip when less than 30 px is left.
- Hover card above the timeline: type badge, timecode, full text, thumbnail when `image_url` is set. Click on a tag:
  select and seek; double click: edit.
- Wheel: zoom around the pointer (`exp(deltaY * 0.0015)`); `Shift` + wheel or a horizontal wheel: pan; press and
  drag in the ruler or the empty track: scrub (pauses, one seek per animation frame). The scroll bar thumb drags,
  a click on its track centres the view there. While playing the view pages when the playhead leaves it; after a
  jump outside the view it centres on the playhead.

## 7. Keyboard (`js/keys.js`)

Active when no `input`, `textarea`, `select` or contenteditable has focus; `Space`, arrows, `Home`, `End`, `PageUp`,
`PageDown` never scroll the page. Buttons give focus back after a mouse click so `Space` stays play / pause.

| Key | Action |
|---|---|
| `Space` | play / pause |
| `←` / `→` | one frame back / forward (pauses) |
| `Shift+←` / `Shift+→` | ten frames |
| `Home` / `End` | first / last frame |
| `↑` / `↓` | previous / next note (seeks); without a selection: the nearest note before / at or after the playhead |
| `1` / `2` / `3` | new FIX / ADD / REMOVE note at the current frame |
| `F2` | edit the selected note (or the first note on the current frame) |
| `Delete` | remove the selected note, toast `Note removed`, select the next one |
| `Ctrl+C` | `copy_notes_text()` (only when no page text is selected) |
| `Ctrl+Shift+C` | `copy_readable_text()` (same condition) |
| `Ctrl+O` | `open_video(null)` |
| `0` | reset the timeline zoom |
| `?` | key help (also the `Keys` button; `Esc` closes) |
| in the note field: `Enter` / `Esc` / `Backspace` (empty field) | save / cancel / remove the region |

`Ctrl` shortcuts match the physical key (`KeyC`, `KeyO`), so they work in any keyboard layout.

## 8. Bridge use

- Start: `bridge.connect()` waits for `pywebviewready` (or a filled `window.pywebview.api`), calls `ready()`, applies
  the returned `State`, then delivers events queued meanwhile, then asks `update_status()`. After 8 s without the
  bridge the banner says so (in a browser without `?mock=1`).
- Events: `state` -> `applyState`; `error` -> banner; `toast` -> toast; `open_request` -> save an open edit, then
  `open_video(path)`; `update` -> version chip.
- Rejected calls show their message in the banner (`guard()`); `remove_note` success shows the local toast `Note
  removed`. `Notes copied` is expected as a `toast` event from the shell (the mock sends it).
- Drag and drop: the first dropped file with a video extension (else the first file); `pywebviewFullPath` ->
  `open_video(path)`; no path outside the mock -> nothing: the shell's own drop listener reads the path and sends `open_request` (BRIDGE_API.md, section 5)
- Script errors and unhandled rejections are sent to `log("error", ...)`.

### Version chip and updates (`js/update.js`)

| `stage` | Chip |
|---|---|
| `idle`, `up_to_date` | `v1.0.0`, muted |
| `checking` | `v1.0.0`, muted, pulsing dot |
| `available` | `v1.1.0 available`, accent |
| `downloading` | `Downloading 42%`, accent |
| `ready` with `can_apply` | `Restart to update`, filled accent |
| `ready` without `can_apply` | `v1.1.0 downloaded`, accent (the message says why it cannot apply) |
| `error` | `v1.0.0`, muted, amber dot, `message` as tooltip |

The chip opens a popover: `message`, release notes as pre-wrapped plain text (max 200 px, scrolls) while a release
is available / downloading / ready, `Check for updates` (disabled while checking or downloading), `Download`
(enabled when `available`), `Restart` (enabled when `ready` and `can_apply`; `true` from `apply_update()` disables
everything and shows nothing more), `Download updates automatically` bound to
`settings.auto_download_updates` through `set_setting`, and a `Check for updates` select bound to
`settings.update_check_interval` through `set_setting` with the options `Every 30 minutes` (`30m`), `Every hour`
(`1h`), `Every 6 hours` (`6h`), `Every day` (`1d`), `Every week` (`1w`).

## 9. Mock mode (`?mock=1`)

`js/mock.js` implements every call of `BRIDGE_API.md` with the same shapes, in memory:

| Parameter | Effect |
|---|---|
| `video=<url>` | opened on `ready()` (relative to the page); also listed in `recent` |
| `fps=<rate>` / `fps=unknown` | the "probed" rate, default 25; `unknown` gives `fps_source: "unknown"` |
| `update=available` | a check 1.2 s after start finds `1.1.0`; with auto download on it downloads over 3 s to `ready` with `can_apply: true` |

- Probe: a hidden `<video>` reads width, height and duration; `duration_frames = round(duration * fps)`.
- Notes: ids of 8 hex characters, `order` counter, timecodes from `js/frames.js`, image names per `ARCHITECTURE.md`
  section 6, `image_url` = data URL of the received PNG; `notes_text()` renders the notes file format.
- `open_video(null)` opens a browser file picker; dropped and picked files are opened as object URLs (the file name
  travels in the URL fragment). A recent entry with `exists: false` (`promo_v2.mp4`) answers `File not found.`.
- `copy_notes_text()` / `copy_readable_text()` write to the browser clipboard and send their toasts; `reveal()` and
  `reveal_recent()` send a toast (or reject `No notes file yet.` / `No images yet.` / `File not found.`);
  `apply_update()` resolves `true` when ready.
- Recent: the sample video and a missing `promo_v2.mp4` that still has notes (`notes_count: 4`, `frames_count: 1`);
  `delete_notes()` clears the counts (and the open session's notes when it is the open video), `forget_recent()`
  drops the entry; counts of the open video follow its notes.

## 10. Testing

The frame capture needs the video on the page's origin, so serve the UI folder and a clip from one server.

1. Sample clips with the frame number burned in (ffmpeg from `PATH`; the font path needs the escaped colon):
   ```
   ffmpeg -f lavfi -i testsrc2=size=1280x720:rate=25 -t 12 -vf "drawtext=fontfile='C\:/Windows/Fonts/consola.ttf':text='%{n}':fontsize=72:fontcolor=white:box=1:boxcolor=black@0.7:x=40:y=40" -pix_fmt yuv420p -c:v libx264 sample.mp4
   ```
   and the same with `rate=30000/1001 -t 70` as `sample2997.mp4` for drop-frame.
2. Serve both from one origin with HTTP range support (Python's `http.server` has none; seeking still works for a
   small clip that is fully buffered). A scratch script that maps `/` to `src/frame_notes/ui/` and `/*.mp4` to the
   clip folder is enough. Never put clips into the repository.
3. Open `http://127.0.0.1:5173/index.html?mock=1&video=sample.mp4` and check:
   - step with `←` / `→` / `Shift+→`, `Home`, `End`: the counter equals the burned-in number (last frame of the
     12 s clip: `0:00:11:24`); play and pause: the counter matches the paused frame;
   - `sample2997.mp4&fps=29.97`: frame 1800 shows `0;01;00;02`, frame 1799 `0;00;59;29`;
   - `1`, drag a rect, type, `Enter`: marker, row, thumbnail and hover card with the captured frame and region;
     `2` with a click puts a point; two notes on one frame stack; labels clip before the next marker;
   - `↑` / `↓`, click on a row and on a marker: seek and region display; `F2` / double click: inline edit, new
     region, `Backspace` in the empty field removes it; `Delete` and the toast; `Esc` cancels a new note;
   - wheel zoom around the pointer, `Shift` + wheel pan, scroll bar, `0`, scrub in the ruler;
   - resize the window and toggle the panel and the banner: the overlay rectangle equals the displayed frame
     (compare `#overlay` and the video's contain rectangle with `getBoundingClientRect()`);
   - `fps=unknown`: amber control, notes refused with a toast, `Use 25` after a second of playback; changing the rate
     in the select re-times the notes;
   - `update=available`: chip states and popover; empty state without `video=`; Reveal errors in the banner;
   - the console stays free of errors.
4. The real window (pywebview, `pywebviewFullPath`, Explorer reveal, clipboard, update apply) is checked with the
   shell (`scripts\run_dev.bat`), not by the mock.
