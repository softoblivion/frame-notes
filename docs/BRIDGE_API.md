# Bridge API: the JS <-> Python contract

Owner: shell worker (v1.0.0 build), then integrator. The only definition of what the UI may call and what the
shell sends. Implemented by `src/frame_notes/shell/bridge.py` (pywebview `js_api`, pattern of `RECIPES.md`
section 4: only the contract methods are public attributes, everything else is underscored) and consumed by
`src/frame_notes/ui/js/bridge.js`, which also provides the mock for `?mock=1`.

## 1. Calls (JS -> Python)

All calls are `window.pywebview.api.<name>(...)` and return a promise. A failure rejects with an English sentence
for the reviewer (never a traceback; the full traceback goes to the log). Arguments and results are plain JSON.

| Call | Arguments | Returns | Notes |
|---|---|---|---|
| `ready()` | | `State` | called once when the page has loaded; the shell starts the event pump after it and opens the argv video, if any |
| `open_video(path)` | `string \| null` | `State \| null` | `null` opens the native file dialog (filters: Video `*.mp4;*.m4v;*.mov;*.webm;*.mkv`, All files); `null` result when the dialog was cancelled. Errors: file missing, notes file belongs to another source, newer notes format |
| `get_state()` | | `State` | |
| `set_fps(fps)` | `number \| null` | `State` | `null` clears the override |
| `set_last_frame(frame)` | `number` | `null` | fire-and-forget; the shell throttles the settings write |
| `add_note(note)` | `{frame, kind, text, region, image_png}` | `Note` | `region`: `Region \| null`; `image_png`: base64 string of a PNG or `null`; `kind` one of `FIX`, `ADD`, `REMOVE`. The core returns the whole payload; the bridge extracts the new note (highest `order`) and the `state` event carries the rest |
| `update_note(patch)` | `{id, text?, region?, image_png?}` | `Note` | missing keys keep the value; `region: null` removes the region and its image |
| `remove_note(id)` | `string` | `null` | |
| `notes_text()` | | `string` | the data file text as it would be written |
| `copy_notes_text()` | | `null` | puts `notes_text()` on the Windows clipboard; toast `Notes copied` |
| `readable_text()` | | `string` | the readable text (`Session.readable_text()`: the export without the image lines) |
| `copy_readable_text()` | | `null` | puts `readable_text()` on the Windows clipboard; toast `Readable notes copied` |
| `copy_text(text)` | `string` | `null` | puts `text` on the Windows clipboard; no toast (the page shows its own, `Copied 0:01:12:05`). Needs no open video. Errors: `Invalid text.` for a non-string, a blank text or one longer than 200 characters; the clipboard error of the Copy calls |
| `reveal(which)` | `"video" \| "notes" \| "images"` | `null` | opens Explorer with the item selected: the video; the data file inside the notes folder (`notes`); the `frames` folder (`images`). Error when it does not exist yet |
| `reveal_recent(path)` | `string` | `null` | for a recent entry: Explorer with its notes folder selected, or the video when there is no folder. Error `File not found.` when neither exists |
| `delete_notes(path)` | `string` | `State` | removes the notes folder of that video (`core.notes.delete_notes`); toast `Deleted 3 notes and 2 frames.` (counts from the result, `1 note`, `1 frame`, the frames part left out when 0; `Deleted the empty notes folder.` when both are 0). When `path` is the open video the session is opened again (empty) before the state is returned. Errors: the refusals of `delete_notes` |
| `forget_recent(path)` | `string` | `State` | removes the entry from `recent` and saves the settings; files are not touched; unknown path is not an error |
| `set_setting(key, value)` | `string, any` | `State` | allowed keys: `auto_download_updates` (bool), `notes_panel_open` (bool); any other key is `Unknown setting.`, a value of the wrong type `Invalid setting value.` |
| `log(level, message)` | `string, string` | `null` | `level` in `debug`, `info`, `warning`, `error`; written to `app.log` as `ui: <message>` |
| `update_status()` | | `UpdateStatus` | the current self-update status |
| `check_updates()` | | `UpdateStatus` | starts a check in the background (ignored while a check or download runs); progress arrives as `update` events |
| `download_update()` | | `UpdateStatus` | starts the download of the release found by the last check |
| `apply_update()` | | `boolean` | `true`: the downloaded exe was started in apply mode and the app closes itself right after the call returns; `false`: nothing to apply (the status event says why) |

Self-update wiring (shell side): the updater is built once by `core.selfupdate.build_updater(...)` (`CORE.md`) with
`on_status` bound to the `update` event; `main.py` checks `argv[0] == APPLY_FLAG` first of all and returns
`run_apply(argv[1:])` before logging, settings or any window; after `ready()` the shell calls `startup()` then
`check_background()`; when a status with stage `available` arrives and `settings.auto_download_updates` is on,
the shell calls `download_background()` itself. After the launch check the shell starts the package's
`UpdateScheduler(updater)`, which checks every 5 minutes (waiting longer after failed checks, skipping the check
without a network connection; no setting); the shutdown calls `scheduler.stop()`. From source (`exe_path` is `None`) the check and download work,
`apply_update()` returns `false` and the status message says it applies to the packaged exe only.

## 2. Events (Python -> JS)

Delivered as `window.frameNotes.dispatch(name, payload)` through `window.evaluate_js`; buffered until `ready()`.
Only `state` is coalesced (the latest wins); `toast`, `error`, `open_request` and `update` are delivered in order. JSON is produced with `ensure_ascii=False` and U+2028 /
U+2029 escaped.

| Event | Payload | When |
|---|---|---|
| `state` | `State` | after `ready()`, after every change of the session (open, note added / changed / removed, fps change), after `set_setting` |
| `error` | `{message}` | a failure outside a call (file write failed on save, argv file missing) |
| `toast` | `{message}` | short confirmations (`Notes copied`) |
| `open_request` | `{path}` | the video given on the command line, sent after `ready()`; the UI calls `open_video(path)` (so an unsaved note edit can finish first). A missing argv file is reported as an `error` event instead |
| `update` | `UpdateStatus` | every change of the self-update status (check, download progress, ready, error) |

`UpdateStatus` (the vendored `selfupdate` package's payload, unchanged):

```json
{"stage": "idle | checking | up_to_date | available | downloading | ready | error",
 "current": "1.0.0", "latest": "1.1.0", "notes": "release body in Markdown or null", "size": 12345678,
 "published_at": 1760000000.0, "progress": 0.42, "message": "Downloading Frame Notes 1.1.0… 42%",
 "checked_at": 1760000000.0, "can_apply": false}
```

## 3. Payload types

`State` = `Session.payload()` of `docs/CORE.md` plus the shell keys:

```json
{
  "version": "1.0.0",
  "video": { ...core fields..., "url": "http://127.0.0.1:42011/media/8f1c2e.../promo_v3.mp4" },
  "timecode": {...}, "notes": [...], "counts": {...},
  "recent": [{"path": "D:\\clips\\promo_v3.mp4", "name": "promo_v3.mp4", "last_frame": 307, "exists": true,
              "notes_count": 3, "frames_count": 2, "notes_dir": "D:\\clips\\promo_v3_notes", "is_open": true}],
  "settings": {"auto_download_updates": true, "notes_panel_open": true}
}
```

`recent[]`: `notes_count` / `frames_count` come from `core.notes.notes_summary(path)` (`null` both when the video
has no data file, new or legacy); the shell caches the summary per data file by modification time and size, so a
state event reads a data file only after it changed. `notes_dir` is `notes_dir_for(path)` when that folder exists,
else `null`. `is_open` marks the entry of the open video.

`Note` and `Region` are the objects of `CORE.md` (`Note` includes `timecode`, `image` and `image_path`;
`image_path` is also served as `"image_url": "http://127.0.0.1:<port>/image/<token>/<name>"` so the UI can show
thumbnails without reading the disk).

## 4. Local server (`shell/server.py`)

One `ThreadingHTTPServer` on `127.0.0.1`, the first free port from 42011 to 42030, started before the window:

| Route | Serves |
|---|---|
| `/` and `/<file>` | the UI folder (`src/frame_notes/ui/`, or the bundled copy, or `--ui-dir`): safe path join, MIME by extension (`.js` as `text/javascript`, `.mjs` too), `Cache-Control: no-store` |
| `/media/<token>/<name>` | the opened video file; `Accept-Ranges: bytes`, `Range` requests answered with `206` and `Content-Range`, `HEAD` supported, `Content-Type` from the extension (`video/mp4`, `video/webm`, `video/quicktime`, `video/x-matroska`) |
| `/image/<token>/<name>` | a PNG from the `frames` folder of the opened video; `name` must match an existing note image |

The token is a fresh `secrets.token_hex(16)` for every opened video; the previous token stops working. Anything
else is `404`. The server never lists directories and never serves a path outside the UI folder, the opened
video, or the images folder.

After `open_video` the shell sets the native window title to `<video name> - Frame Notes` (pywebview does not mirror
`document.title`). The server answers only requests whose `Host` is loopback, serves `/media` for the exact file
name only and `/image` for existing `.png` files directly inside the `frames` folder.

## 5. Drag and drop

pywebview reveals the full path of a dropped file only to a Python-side DOM listener (`window.dom.document.events.drop`,
which makes the WebView2 backend collect the paths on drop and add `pywebviewFullPath` to the files of the event
dict the listener receives); the JavaScript `File` objects never carry it. So the shell registers that listener
once the window is shown, takes the first dropped file with a video extension and emits `open_request {path}`;
the UI's own `drop` handler only prevents the default navigation and shows the drop veil. In the browser mock the
UI opens the dropped file through an object URL instead.

## 6. Errors and messages

Messages are one English sentence each, neutral and factual (`CONVENTIONS.md`, UI copy):

| Situation | Message |
|---|---|
| video path does not exist | `File not found.` |
| notes file names another source | `promo_v3_notes.txt belongs to promo_v3.mov. Rename or move it to start a new review.` |
| newer notes format | `The notes file was written by a newer version of Frame Notes.` |
| notes file cannot be written | `Could not write promo_v3_notes.txt. The notes stay in memory until the next change succeeds.` |
| unknown kind / bad region | `Invalid note.` |
| reveal with nothing to show | `No notes file yet.` / `No images yet.` |
| delete with nothing to delete | `There are no notes for promo_v3.mp4.` |
| notes folder of another video | `The notes folder promo_v3_notes belongs to promo_v3.mov.` |
| notes folder with foreign content | `The notes folder promo_v3_notes holds other files. Delete it by hand.` |
| 1.0 layout cannot be moved | `The notes of promo_v3.mp4 could not be moved into the promo_v3_notes folder.` |
