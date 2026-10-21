# 0001 — One plain-text notes file next to the video, After Effects timecodes

Date: 2026-10-21

## Context

The reviewer logs change requests while watching a render; the editor needs to find each frame in After Effects
or Premiere. The user asked for one output file written next to the source video, a timeline that shows the
notes, and timecodes in the After Effects form.

## Decision

- One file per video, `<name>_notes.txt`, tab-separated `timecode  TYPE  text` lines under a short `# ` header
  (format version, source file name, fps, duration). The same file is the app's own state: there is no second
  project or sidecar file, and the app reloads it on open.
- Timecodes follow AE display rules (`h:mm:ss:ff`, drop-frame with `;` for 29.97 / 59.94), so the column pastes
  into the AE time field as it is.
- Three note types bound to the keys `1` / `2` / `3`: FIX, ADD, REMOVE.
- Saving is automatic on every change; no Save button.

## Consequences

- The file is readable and editable by hand and pastes into a chat; unknown lines survive a save.
- The format is a contract: changes bump the header version and keep a reader for the old form.
- A name collision (`promo.mp4` and `promo.mov` in one folder) is caught by the `source` header, not by the
  file name.

## Rejected

- JSON state plus a text export (two files next to the video; the text would drift from the state).
- `<name>.<ext>_notes.txt` (unambiguous, but a cluttered name for the common case).
- `<name>.notes.txt` (a dot suffix reads like a second extension; the user chose the underscore form).
- Premiere / AE marker import formats (CSV, XMP): useful later as an export, not as the primary file; the
  reviewer's deliverable is a readable list.
