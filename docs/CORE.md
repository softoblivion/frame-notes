# Core

Owner: core worker (v1.0.0 build), then integrator. Package `src/frame_notes/core/`, stdlib only, never imports
pywebview or the shell. Tests: `tests/test_core_*.py` (`unittest`), every test sets `FRAME_NOTES_HOME` to a scratch
folder and writes video and notes fixtures into a temporary directory only. Behaviour rules: `ARCHITECTURE.md`
sections 4 to 8. This file is the contract the shell and the tests are written against.

## `timecode.py`

```python
NOMINAL_RATES = (23.976, 24.0, 25.0, 29.97, 30.0, 48.0, 50.0, 59.94, 60.0, 100.0, 120.0)

@dataclass(frozen=True)
class TimecodeBase:
    fps: float        # nominal rate used for frame <-> seconds (23.976 is 24000/1001, 29.97 is 30000/1001, 59.94 is 60000/1001)
    base: int         # frames per timecode second: round(fps) (24 for 23.976, 30 for 29.97, 60 for 59.94)
    drop: bool        # True only for 29.97 and 59.94
    digits: int       # frame digits: 2, or 3 when base >= 100
    separator: str    # ":" non-drop, ";" drop

def nominal_rate(fps: float) -> float
    # snaps fps to the nearest NOMINAL_RATES entry when within 0.5 %, else returns fps unchanged;
    # fps <= 0 or not finite raises ValueError
def timecode_base(fps: float) -> TimecodeBase
def frame_to_timecode(frame: int, tb: TimecodeBase) -> str
    # "h:mm:ss:ff": hours unpadded, minutes and seconds two digits, frames tb.digits wide, tb.separator;
    # frame < 0 raises ValueError
def timecode_to_frame(text: str, tb: TimecodeBase) -> int
    # accepts "h:mm:ss:ff", "mm:ss:ff", "ss:ff" with ":" or ";" (either separator for either base),
    # surrounding whitespace ignored; ValueError on anything else, on ff >= base, and on a dropped number
    # (e.g. "0;01;00;00" at 29.97)
def seconds_to_frame(seconds: float, fps: float) -> int        # floor(seconds * fps + 1e-4), never negative
def frame_to_seconds(frame: int, fps: float) -> float          # (frame + 0.5) / fps, the frame centre, used for seeking
def duration_frames(seconds: float, fps: float) -> int         # round(seconds * fps)
def format_fps(fps: float) -> str                              # "25", "29.97", "23.976": up to 3 decimals, trailing zeros cut
def parse_fps(text: str) -> float                              # inverse of format_fps, ValueError on junk
```

Drop-frame rule (SMPTE): `drop_per_minute = 2` at 29.97 and `4` at 59.94; those frame numbers are skipped at the
start of every minute except minutes 0, 10, 20, 30, 40, 50. Conversion from a frame count `n` to the displayed
number with `fpm = base * 60 - d` (frames per drop minute) and `fp10 = fpm * 10 + d` (frames per ten minutes):

```
q, m = divmod(n, fp10)
shown = n + 9 * d * q + (d * ((m - d) // fpm) if m > d else 0)
```

then `h, mm, ss, ff` from `shown` with `base`. The inverse (`timecode_to_frame`) reverses the same formula and
rejects `ff < d` when `ss == 0` and `mm % 10 != 0`.

Values the tests must assert (checked against After Effects display):

| fps | frame | timecode |
|---|---|---|
| 25 | 0 | `0:00:00:00` |
| 25 | 307 | `0:00:12:07` |
| 25 | 90000 | `1:00:00:00` |
| 23.976 | 24 | `0:00:01:00` |
| 29.97 | 1799 | `0;00;59;29` |
| 29.97 | 1800 | `0;01;00;02` |
| 29.97 | 17982 | `0;10;00;00` |
| 29.97 | 107892 | `1;00;00;00` |
| 59.94 | 3600 | `0;01;00;04` |
| 120 | 125 | `0:00:01:005` |

`timecode_to_frame` is the exact inverse for every row; `"12:07"` at 25 fps is 307; `"0:00:12:25"` at 25 fps
raises ValueError.

## `notes.py`

```python
FORMAT_VERSION = 1
KINDS = ("FIX", "ADD", "REMOVE")

@dataclass
class Region:
    shape: str          # "rect" or "point"
    x0: float; y0: float; x1: float; y1: float   # fractions 0..1 of the frame; a point has x0 == x1, y0 == y1
    # text form: "rect 0.540,0.360 0.880,0.800" / "point 0.512,0.300" (three decimals, rect normalised so x0 <= x1, y0 <= y1)
    def to_text(self) -> str
    @staticmethod
    def from_text(text: str) -> "Region"       # ValueError on junk or values outside 0..1
    def to_dict(self) -> dict; @staticmethod def from_dict(d: dict) -> "Region"

@dataclass
class Note:
    id: str             # 8 lowercase hex characters, random, unique inside the file; not written to the file
    frame: int
    kind: str           # one of KINDS
    text: str           # one line; tabs, CR and LF replaced by spaces, stripped
    region: Region | None = None
    image: str | None = None      # file name inside frames/, no path
    order: int = 0      # creation sequence inside the session, breaks ties between notes on one frame

@dataclass
class NotesFile:
    source: str                   # file name of the video (no path)
    fps: float
    duration_frames: int | None
    notes: list[Note]
    extra: list[str]              # lines that did not parse, in original order, written back after the notes

FRAMES_DIR_NAME = "frames"

# Layout (ARCHITECTURE.md section 6): everything inside one folder next to the video.
def notes_dir_for(video: Path) -> Path       # same folder as the video, "<stem>_notes"
def notes_path_for(video: Path) -> Path      # notes_dir_for(video) / "<stem>_notes.txt" (data file)
def export_path_for(video: Path) -> Path     # notes_dir_for(video) / "<stem>_notes.md" (readable export)
def images_dir_for(video: Path) -> Path      # notes_dir_for(video) / "frames"
def legacy_notes_path_for(video: Path) -> Path   # 1.0 layout: "<stem>_notes.txt" beside the video
def image_name(note: Note, tb: TimecodeBase, taken: set[str]) -> str
    # "<timecode with ':' and ';' replaced by '-'>_<KIND>.png"; "_2", "_3"... appended while the name is in `taken`
def parse(text: str) -> NotesFile
    # header: "# Frame Notes <n>" (n > FORMAT_VERSION raises NotesFormatError), "# source: ...", "# fps: ...",
    # "# duration: <timecode>" (optional); a missing source or fps header raises NotesFormatError.
    # Note lines: timecode TAB KIND TAB text [TAB region [TAB image]]; column 4 is a region when it starts with
    # "rect " or "point ", otherwise it is the image name; a line that fails goes to `extra`. Blank lines ignored.
def render(nf: NotesFile) -> str
    # header, notes sorted by (frame, order), extra lines; LF line endings, trailing newline; "# duration" omitted when None
def render_readable(nf: NotesFile, images: bool = True) -> str
    # the readable export of ARCHITECTURE.md section 6: "# <source>", blank, summary "<fps> fps · <duration> ·
    # FIX n · ADD n · REMOVE n" (duration part left out when None), blank, one "- **<timecode>** · <KIND> · <text>"
    # line per note sorted by (frame, order) (" · <text>" left out when the text is empty), followed by
    # "  ![](frames/<image>)" when `images` and the note has a plain image name; "No notes." when there are none.
    # Extra lines are not exported. LF endings, trailing newline.
class NotesFormatError(ValueError)   # message is a user-facing English sentence

class NotesLayoutError(OSError)      # migrate / delete refused or failed; message is a user-facing English sentence

def migrate_layout(video: Path, log: logging.Logger) -> bool
    # Moves a 1.0 layout into the folder (ARCHITECTURE.md section 6, "Life cycle"); True when anything moved.
    # 1. nothing to do when neither the legacy data file nor a *.png directly in notes_dir_for(video) exists;
    # 2. a legacy data file is read and parsed first: NotesFormatError (or a source of another video) -> no move,
    #    returns False (Session.open then reports the legacy file as in 1.0);
    # 3. *.png directly in the folder -> frames/ with os.replace (a name already in frames/ is left and logged);
    # 4. the legacy data file -> notes_path_for(video) with one os.replace, unless a data file is already there
    #    (then the legacy file stays and a warning is logged);
    # each step logs at INFO. OSError of a step is raised as NotesLayoutError("The notes of <name> could not be moved
    # into the <stem>_notes folder."). The export is written by the session afterwards.
def notes_summary(video: Path) -> dict | None
    # {"notes": n, "frames": m} from the data file (the new one, else the legacy one beside the video); m counts
    # notes with a plain image name. None when there is no data file; also None when it cannot be read or parsed.
def delete_notes(video: Path, log: logging.Logger) -> dict
    # Removes notes_dir_for(video) and a legacy data file beside the video; returns {"notes": n, "frames": m,
    # "path": str(folder)} (counts as notes_summary before the removal, frames = PNG files removed).
    # Refuses with NotesLayoutError when: nothing exists ("There are no notes for <name>."); the folder is not a
    # plain directory (symlink, junction, file); a data file names another source ("The notes folder <stem>_notes
    # belongs to <source>."); the folder holds anything other than the data file, the export, frames/ and *.png
    # directly inside (1.0 layout), each optionally with a ".part" / ".rename" leftover, or frames/ holds anything
    # but *.png and those leftovers ("The notes folder <stem>_notes holds other files. Delete it by hand.").
    # The check runs before anything is removed; then files are unlinked and the folders removed with rmdir.
```

## `probe.py`

```python
@dataclass
class ProbeResult:
    container: str              # "mp4", "webm" or "unknown"
    fps: float | None           # from the first video track; None when not found
    duration: float | None      # seconds
    width: int | None
    height: int | None
    frames: int | None          # sample count of the video track when known

def probe(path: Path) -> ProbeResult
```

- Raises `OSError` only when the file cannot be opened or read; damaged or foreign content yields
  `container="unknown"` with `None` fields, never an exception.
- MP4 / MOV (`ftyp` or a `moov` within the first boxes; `mdat` skipped by its size, 64-bit `largesize` and
  `size == 0` to end of file handled): `moov/mvhd` timescale and duration (fallback duration); first `trak` whose
  `mdia/hdlr` handler is `vide`: `tkhd` width and height (16.16 fixed, version 0 and 1), `mdia/mdhd` timescale
  and duration (versions 0 and 1), `stbl/stts`: `fps = timescale / most frequent sample delta`, `frames = sum of
  counts`. Boxes may appear in any order; `moov` after `mdat` is common.
- WebM / Matroska (EBML header `1A 45 DF A3`): `Segment` (`18538067`, size may be unknown) -> `Info` (`1549A966`):
  `TimecodeScale` (`2AD7B1`, default 1 000 000 ns), `Duration` (`4489`, float, in timecode units); `Tracks`
  (`1654AE6B`) -> first `TrackEntry` (`AE`) with `TrackType` (`83`) == 1: `DefaultDuration` (`23E383`, ns per
  frame) -> `fps = 1e9 / DefaultDuration`; `Video` (`E0`): `PixelWidth` (`B0`), `PixelHeight` (`BA`). Stop
  scanning at the first `Cluster` (`1F43B675`).
- Tests build the containers in code (a small box / EBML writer inside the test module), no binary fixtures.

## `paths.py`

```python
APP_DIR_NAME = "frame-notes_data"
ENV_HOME = "FRAME_NOTES_HOME"

class AppPaths:
    home: Path            # os.environ[ENV_HOME] as given; else `<base>rame-notes_data` where <base> is the folder of
                          # sys.executable when frozen, else the project root (the parent of `src`); AppPaths(home=...) overrides
    settings_file: Path   # home / "settings.json"
    log_dir: Path         # home / "logs"
    updates_dir: Path     # home / "updates"
    exe_path: Path | None # Path(sys.executable) when frozen, None from source
    def ensure(self) -> None   # creates home, log_dir, updates_dir

def atomic_write_text(path: Path, text: str) -> None     # UTF-8 without BOM, "\n" kept as is, via "<name>.part" + os.replace
def atomic_write_bytes(path: Path, data: bytes) -> None
def atomic_write_json(path: Path, obj) -> None            # json.dumps(ensure_ascii=False, indent=2)
```

Nothing outside this module builds a path inside the data folder.

## `selfupdate.py` (configuration of the vendored updater)

The updater itself is the vendored package `frame_notes.vendor.selfupdate` (`ARCHITECTURE.md` section 10; never
edited here). This module is the one place that knows the Frame Notes values:

```python
from ..vendor.selfupdate import APPLY_FLAG, SelfUpdater, UpdateConfig, run_apply   # re-exported for the shell

REPO = "softoblivion/frame-notes"
ASSET_NAME = "FrameNotes.exe"
APP_NAME = "Frame Notes"
RELEASE_URL_ENV = "FRAME_NOTES_RELEASE_URL"

def build_updater(paths: AppPaths, version: str, on_status: Callable[[dict], None],
                  on_toast: Callable[[str, str], None] | None = None) -> SelfUpdater
    # UpdateConfig(repo=REPO, asset_name=ASSET_NAME, app_name=APP_NAME, version=version,
    #              updates_dir=str(paths.updates_dir), exe_path=str(paths.exe_path) if paths.exe_path else None,
    #              on_status=on_status, on_toast=on_toast, release_url_env=RELEASE_URL_ENV,
    #              logger_name="frame_notes.selfupdate")
```

```python
from ..vendor.selfupdate import UpdateScheduler   # re-exported

def build_scheduler(updater: SelfUpdater) -> UpdateScheduler
    # UpdateScheduler(updater, logger_name="frame_notes.selfupdate.schedule"). The period is fixed by the library
    # (CHECK_PERIOD = 300 s with a small random offset, doubling after failed checks up to an hour, no check without
    # a network connection); there is no setting. The shell starts it after the launch check and calls stop() on
    # exit; the scheduler skips a period while an update waits for the restart.
```

Test: `build_updater` with a scratch `AppPaths` yields a `SelfUpdater` whose `config` carries exactly these values
and whose `status()["stage"]` is `idle`; no network is touched.

## `settings.py`

```python
DEFAULTS = {
    "window": {"x": None, "y": None},  # position only; the window size is fixed (ARCHITECTURE.md, section 11)
    "last_dir": "",
    "recent": [],                      # [{"path": str, "last_frame": int, "opened_at": int}], newest first, at most 20
    "fps_overrides": {},               # {absolute video path: fps}
    "auto_download_updates": True,
    "notes_panel_open": True,
}
RETIRED_KEYS = ("update_check_interval",)   # written by 1.1.0; dropped on load, never written back
RETIRED_WINDOW_KEYS = ("width", "height", "maximized")   # written up to 1.3.0; dropped from `window` on load

class Settings:
    def __init__(self, path: Path)
    def load(self) -> None             # missing or corrupt file -> DEFAULTS, a corrupt file is renamed to settings.json.bad;
                                       # RETIRED_KEYS are removed without an error
    def save(self) -> None             # atomic_write_json
    data: dict                         # the current values (deep copy of DEFAULTS merged with the file)
    def touch_recent(self, path: Path, last_frame: int) -> None
    def forget_recent(self, path: Path) -> bool   # removes the entry (case-insensitive path match); True when found
    def set_fps_override(self, path: Path, fps: float | None) -> None
    def fps_override(self, path: Path) -> float | None
```

## `session.py`

```python
class SessionError(Exception)          # message is a user-facing English sentence

class Session:
    def __init__(self, settings: Settings, on_change: Callable[[dict], None], log: logging.Logger)
    def open(self, video: Path) -> dict          # probe + load notes; SessionError when the file is missing,
                                                 # when the notes file names another source, or when it is a newer format
    def close(self) -> None                      # saves settings (recent, last frame), no file writes otherwise
    def payload(self) -> dict                    # see below
    def set_fps(self, fps: float | None) -> dict # None clears the override; rewrites the notes file (timecodes change)
    def set_last_frame(self, frame: int) -> None # remembered in settings.recent, saved at most once per 2 s
    def add_note(self, frame: int, kind: str, text: str, region: dict | None, image_png: bytes | None) -> dict
    def update_note(self, note_id: str, text: str | None = None, region: dict | None | Literal["keep"] = "keep",
                    image_png: bytes | None = None) -> dict
    def remove_note(self, note_id: str) -> None
    def notes_text(self) -> str                  # render() of the current state
    def readable_text(self) -> str               # render_readable(images=False) of the current state ("" like notes_text)
```

- `open()` first calls `migrate_layout(video, log)` (a `NotesLayoutError` becomes `SessionError` with its message),
  then reads `notes_path_for(video)`; when that file does not exist but a legacy data file does (one that
  migration left in place), the legacy file is validated as in 1.0 and its errors are raised (another source,
  newer format). After a migration the export is written once (when the frame rate is known).
- Every mutation writes the data file with `atomic_write_text`, then the export (`export_path_for`, same
  conditions), and the image (when given) with `atomic_write_bytes` into `frames/` (created on demand; removed
  when empty after a removal), then calls `on_change(self.payload())`. A failed export write is reported like a
  failed data file write (`SessionError("The notes could not be saved to <path>")`). A note with no region never has an image; replacing a region without a new
  image keeps the old image file; `remove_note` deletes the image file.
- A video that has no notes file and no notes gets no folder; once the data file exists it is kept (header only,
  with the export, when the last note is removed).
- `fps` precedence: settings override, else probe, else the `# fps:` header of an existing notes file (reported as
  `fps_source: "override"`), else `None` (payload `fps: null`, `fps_text: ""`, `timecode: null`; `add_note` and
  `update_note` raise `SessionError` until the reviewer sets a rate). `set_fps` keeps frame numbers, rewrites the
  timecodes, renames the image files to the new timecodes (all or nothing), records the override when it differs
  from the probed rate and clears it when it equals it.
- A failed write keeps the note in memory, still calls `on_change`, then raises `SessionError("The notes could not
  be saved to <path>")`; the next successful mutation writes everything.
- `on_change` runs while the session lock is held: the shell only enqueues the payload there. `open()` returns the
  payload and does not call `on_change`.
- `payload()["video"]["fps"]` is the exact rate (30000/1001 for 29.97); `fps_text` is the nominal text. `duration`
  is `frames / fps` when the probe knows the frame count, else the container duration; `duration_frames` follows.
- Image names from a hand-edited file that contain a path component are never used for file operations
  (`image_path: null`); the `source` header is compared case-insensitively; a kind is accepted in any case.
- Regions are validated (`Region.from_dict`); text is normalised to one line; an unknown kind raises
  `SessionError("Unknown note type")`.

### `payload()`

```json
{
  "video": {"path": "D:\\clips\\promo_v3.mp4", "name": "promo_v3.mp4", "dir": "D:\\clips",
            "width": 1920, "height": 1080, "duration": 92.4, "duration_frames": 2310,
            "fps": 25.0, "fps_text": "25", "fps_source": "file" | "override" | "unknown",
            "notes_dir": "D:\\clips\\promo_v3_notes",
            "notes_path": "D:\\clips\\promo_v3_notes\\promo_v3_notes.txt",
            "export_path": "D:\\clips\\promo_v3_notes\\promo_v3_notes.md",
            "images_dir": "D:\\clips\\promo_v3_notes\\frames",
            "last_frame": 0},
  "timecode": {"base": 25, "drop": false, "digits": 2, "separator": ":"},
  "notes": [{"id": "3f9a2c1d", "frame": 307, "timecode": "0:00:12:07", "kind": "FIX",
             "text": "logo flickers for two frames",
             "region": {"shape": "rect", "x0": 0.54, "y0": 0.36, "x1": 0.88, "y1": 0.8},
             "image": "0-00-12-07_FIX.png", "image_path": "D:\\clips\\promo_v3_notes\\frames\\0-00-12-07_FIX.png",
             "order": 1}],
  "counts": {"FIX": 1, "ADD": 0, "REMOVE": 0}
}
```

`payload()` with no video open returns `{"video": null, "timecode": null, "notes": [], "counts": {...zeros}}`.
Notes are sorted by `(frame, order)`. The shell adds its own keys (`version`, `url`, `recent`, settings) on top
of this object; the core never knows about URLs.
