# MHz Club Serial Command Protocol (MVP)
**Version:** 0.1 (draft)  
**Transport:** RS-232 (8N1), default 9600 bps (configurable later)  
**Framing:** ASCII lines terminated with LF (`\n`)  
**Case:** Command keywords are case-insensitive; filenames/strings are case-sensitive  
**Max line length:** 128 bytes (extra bytes ignored until `\n`)  
**Responses:** `OK` on success, or `ERR <CODE> [detail]` on failure
---
## 1) TEXT — 7-segment text
**Syntax**
TEXT "<string>"
- Draws classic monospaced 7-segment glyphs: digits, common letters (H, I, L, O, U, Y, A–F, etc.), `-`, `_`, `=`, space.
- A period `.` attaches a decimal point (DP) to the previous glyph.
- No separate mode required; takes effect immediately.
**Examples**
TEXT "66"
TEXT "HI"
TEXT "LO"
TEXT "133.3"
**Errors**
- `ERR TOO_LONG` if it won’t fit the current layout/window.
- `OK` even if unsupported chars are present (those render blank).
---
## 2) SETMHZ — convenience wrapper for numbers
**Syntax**
SETMHZ <value>
- `<value>` may be integer or decimal (e.g., `66`, `133.3`).
- If there’s a decimal point, DP is rendered; otherwise whole digits.
**Examples**
SETMHZ 66
SETMHZ 133.3
SETMHZ 200
**Errors**
- `ERR BAD_ARGS` (non-numeric/negative)
- `ERR TOO_LONG` (formatted number can’t fit)
---
## 3) CLEAR — blank the display
**Syntax**
CLEAR
- Clears to the current background color.
- Does not change other settings.
**Errors**
- None.
---
## 4) BRIGHT — backlight level (percent)
**Syntax**
BRIGHT <level>
- `<level>` = integer 0–100 (0 = off, 100 = max).
**Errors**
- `ERR BAD_ARGS` (non-integer or out of range)
---
## 5) OFFSET — positional shift (pixels)
**Syntax**
OFFSET <x> <y>
- Signed integers; +x right, +y down.
- Applies to subsequent renders.
**Errors**
- `ERR BAD_ARGS`
- Implementation may clip silently when out of bounds.
---
## 6) SCALE — size factor
**Syntax**
SCALE <factor>
- Any positive float (e.g., `0.5`, `1.333`, `2.0`).
- Applies to subsequent renders.
**Errors**
- `ERR BAD_ARGS` (non-numeric or `<= 0`)
---
## 7) SHOW — display image from SD
**Syntax**
SHOW <filename>
- Still images first (MVP): BMP (uncompressed 16/24-bit).
- Honors current `SCALE` and `OFFSET`.
**Errors**
- `ERR FILE_NOT_FOUND`
- `ERR UNSUPPORTED` (format)
- `ERR TOO_LARGE` (memory/buffer)
---
## 8) PROFILE — load a batch of commands
**Syntax**
PROFILE <name>
- Loads `/profiles/<name>.prof` from SD.
- File format = plain text commands (this same protocol), one per line.
- Invalid lines are skipped; valid commands are applied.
- Returns `OK` on completion (or `ERR FILE_NOT_FOUND` if missing).
**Example** (`/profiles/486DX.prof`)
OFFSET 10 4
SCALE 1.25
BRIGHT 80
TEXT "66"
---
## Error codes
- `ERR UNKNOWN_COMMAND`
- `ERR BAD_ARGS`
- `ERR FILE_NOT_FOUND`
- `ERR UNSUPPORTED`
- `ERR TOO_LONG`
- `ERR OUT_OF_RANGE` (optional usage)
---
## Notes & defaults
- Defaults on boot: `BRIGHT 100`, `OFFSET 0 0`, `SCALE 1.0`.
- Commands are atomic; responses are line-buffered.
- Future extensions (non-breaking): animations, query commands, binary mode.

