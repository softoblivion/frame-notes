# 0002 — One notes folder per video with a readable export

Date: 2026-10-21

Amends 0001 (the data file stays the one source of truth; its location and the files around it change).

## Context

In 1.0 a review left two items next to the video: `<stem>_notes.txt` and, once a note had a region, the folder
`<stem>_notes\` with the frame images. The user asked for everything of a review to sit in one folder, and for a
version of the notes that reads well for a person (in a chat, in a Markdown viewer with the frame images), next
to the data file the app reads back.

## Decision

- One folder next to the video, `<stem>_notes\`, holding the data file `<stem>_notes.txt` (format unchanged,
  version 1, the only file the app reads), the readable export `<stem>_notes.md` and `frames\` with the images.
  The image column keeps the bare file name; the folder is implied.
- The export is plain Markdown without tables (heading, summary line, one list item per note, the frame image on
  the item's second line), rewritten from the data file every time the data file is written and never parsed,
  so it cannot drift from the state.
- The 1.0 layout is moved on open, once, by renames in an order that keeps readable files after a crash (images
  first, data file last); a data file already in the folder wins over a legacy one.
- Deleting a review removes the whole folder, and only after checking that it holds nothing the app did not write.

## Consequences

- The video folder gets one entry per reviewed video instead of two.
- 1.0 cannot find notes moved by 1.1 (it looks beside the video); opening a migrated video in 1.0 starts an empty
  review there. A legacy file written that way is left alone by 1.1 when the folder already has a data file.
- `promo.mp4` and `promo.mov` still share one folder name; the `source` header keeps telling them apart, and
  Delete notes refuses a folder whose data file names the other video.

## Rejected

- A format version bump to 2: the line format did not change, and a bump would make 1.0 refuse files for no gain.
- Keeping the data file beside the video and moving only the images into a folder (still two entries).
- An HTML export (heavier, and it does not read well raw or pasted into a chat).
- Reading the Markdown back as state (two writable sources; the tab-separated file stays the contract).
