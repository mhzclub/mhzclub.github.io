# MHz Club Serial Command Protocol (MVP)

**Version:** 0.2 (draft)  
**Transport:** RS-232 (8N1), default 9600 bps (configurable later)  
**Framing:** ASCII lines terminated with LF (`\n`)  
**Case:** Command keywords are case-insensitive; filenames/strings are case-sensitive  
**Max line length:** 128 bytes (extra bytes ignored until `\n`)  
**Responses:** `OK` on success, or `ERR <CODE> [detail]` on failure

---

## 1) TEXT — 7-segment text
**Syntax**
```
TEXT "<string>"
```
- Draws classic monospaced 7-segment glyphs: digits, common letters (H, I, L, O, U, Y, A–F, etc.), `-`, `_`, `=`, space.
- A period `.` attaches a decimal point (DP) to the previous glyph.
- No separate mode required; takes effect immediately.

**Examples**
```
TEXT "66"
TEXT "HI"
TEXT "LO"
TEXT "133.3"
```

**Errors**
- `ERR TOO_LONG` if it won’t fit the current layout/window.
- `OK` even if unsupported chars are present (those render blank).

---

## 2) SETMHZ — convenience wrapper for numbers
**Syntax**
```
SETMHZ <value>
```
- `<value>` may be integer or decimal (e.g., `66`, `133.3`).
- If there’s a decimal point, DP is rendered; otherwise whole digits.

**Examples**
```
SETMHZ 66
SETMHZ 133.3
SETMHZ 200
```

**Errors**
- `ERR BAD_ARGS` (non-numeric/negative)
- `ERR TOO_LONG` (formatted number can’t fit)

---

## 3) CLEAR — blank the display
**Syntax**
```
CLEAR
```
- Clears to the current background color.  
- Does not change other settings.

**Errors**
- None.

---

## 4) BRIGHT — backlight level (percent)
**Syntax**
```
BRIGHT <level>
```
- `<level>` = integer 0–100 (0 = off, 100 = max).

**Errors**
- `ERR BAD_ARGS` (non-integer or out of range)

---

## 5) OFFSET — positional shift (pixels)
**Syntax**
```
OFFSET <x> <y>
```
- Signed integers; +x right, +y down.  
- Applies to subsequent renders.

**Errors**
- `ERR BAD_ARGS`  
- Implementation may clip silently when out of bounds.

---

## 6) SCALE — size factor
**Syntax**
```
SCALE <factor>
```
- Any positive float (e.g., `0.5`, `1.333`, `2.0`).  
- Applies to subsequent renders.

**Errors**
- `ERR BAD_ARGS` (non-numeric or `<= 0`)

---

## 7) SHOW — display image from SD
**Syntax**
```
SHOW <filename>
```
- Still images first (MVP): BMP (uncompressed 16/24-bit).  
- Honors current `SCALE` and `OFFSET`.

**Errors**
- `ERR FILE_NOT_FOUND`  
- `ERR UNSUPPORTED` (format)  
- `ERR TOO_LARGE` (memory/buffer)

---

## Transitions (MVP)

### 8) FADE — fade out backlight
**Syntax**
```
FADE <ms>
```
- Gradually dims backlight to 0 over `<ms>` milliseconds.

### 9) FADEIN — fade in backlight
**Syntax**
```
FADEIN <ms>
```
- Gradually restores backlight from 0 up to last `BRIGHT` setting over `<ms>`.

### 10) SLIDEOUT — slide current frame off-screen
**Syntax**
```
SLIDEOUT <LEFT|RIGHT|UP|DOWN> <ms>
```
- Moves content off in given direction. Ends blank.

### 11) WIPEOUT — wipe clear in direction
**Syntax**
```
WIPEOUT <LEFT|RIGHT|UP|DOWN> <ms>
```
- Clears screen by sweeping a bar in given direction.

---

## Scripts & Timing (MVP)

> A *command file* is a plain text file of these same commands. Use `RUN` to execute any command file—for presets, demos, intros, or idle loops. There is no separate PROFILE concept.

### 12) RUN — execute a command file
**Syntax**
```
RUN <filename> [LOOP]
```
- Executes a plain text file of commands from SD.  
- Default path: `/scripts/<filename>` if no path given.  
- `LOOP` (optional): when end-of-file is reached, restart from the top until `STOP`.  
- Comments: lines starting with `#` or `;` are ignored.  
- Blank lines are ignored.  
- Nesting: a command file **may not call `RUN`** (to avoid recursion).

**Behavior**
- **Stops on first error** in the file and returns that error (strict mode).  
- While running (including during `DELAY`), serial input is still read; `STOP` can abort at any time.

**Limits (defaults)**
- Max file size: 8 KB  
- Max lines: 256  
- Max line length: 128 bytes  
- Safety timeout for a single `RUN`: 60 s (ignored if `LOOP` is used)

**Errors**
- `ERR FILE_NOT_FOUND`  
- `ERR BAD_ARGS`  
- `ERR NESTING_LIMIT`  
- Any error returned by a line within the file (propagated)

**Examples**
```
RUN intro.scr
RUN idle.scr LOOP
```

### 13) DELAY — pause
**Syntax**
```
DELAY <ms>
```
- Pauses file/script execution for `<ms>` milliseconds.  
- Typical range: 50–5000 ms. Values outside are clamped to 1–30000 ms.

**Errors**
- `ERR BAD_ARGS`

**Example**
```
DELAY 250
```

### 14) STOP — abort script/transition
**Syntax**
```
STOP
```
- Immediately aborts the currently running `RUN` (including `LOOP`) or any in-progress transition.  
- Returns `OK`; device goes to idle ready state.

**Errors**
- None

**Example (script-as-animation)**
```
# /scripts/logo_cycle.scr
SHOW logo1.bmp
DELAY 250
SHOW logo2.bmp
DELAY 250
SHOW logo3.bmp
DELAY 250
FADE 200
CLEAR
FADEIN 200
```
```
RUN logo_cycle.scr LOOP   # loops until STOP
```

---

## Error codes
- `ERR UNKNOWN_COMMAND`
- `ERR BAD_ARGS`
- `ERR FILE_NOT_FOUND`
- `ERR UNSUPPORTED`
- `ERR TOO_LONG`
- `ERR OUT_OF_RANGE`
- `ERR NESTING_LIMIT`

---

## Notes & defaults
- Defaults on boot: `BRIGHT 100`, `OFFSET 0 0`, `SCALE 1.0`.  
- Commands are atomic; responses are line-buffered.  
- Scripts can serve as simple animations (SHOW + DELAY).  
- Future extensions (non-breaking): smoother frame streaming (`SHOWRAW`), query commands, binary mode.
