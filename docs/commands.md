# MHz Club Serial Command Protocol (MVP)

**Version:** 0.34 (draft, 31 Aug 2025 — CRC32 + Orientation/Mirror + Utility cmds + Startup search order + .mhz scripts)  
**Transport:** RS‑232 (8N1), default 9600 bps (configurable later)  
**Framing:** ASCII lines terminated with LF (`\n`) unless noted  
**Case:** Command keywords are case‑insensitive; filenames/strings are case‑sensitive  
**Max line length:** 128 bytes for ASCII commands (binary payloads defined below)  
**Responses:** `OK` or `ERR <CODE> [detail]`  
**Error policy:** Parser **stops on first error** for command files (`RUN`).

---

## 1) TEXT — 7‑segment text
```
TEXT "<string>"
```
- Monospaced 7‑segment glyphs (digits, common letters, `-`, `_`, `=`, space).  
- Period `.` attaches a decimal point to the previous glyph.  
- Honors ORIENT/MIRROR/SCALE/OFFSET.

---

## 2) SETMHZ — convenience for numbers
```
SETMHZ <value>
```
- Integer or decimal; DP rendered if decimal.  
- Honors ORIENT/MIRROR/SCALE/OFFSET.

---

## 3) CLEAR — blank display
```
CLEAR
```

---

## 4) BRIGHT — backlight percent
```
BRIGHT <level>
```
- `<level>` 0–100.

---

## 5) OFFSET — positional shift (px)
```
OFFSET <x> <y>
```

---

## 6) SCALE — size factor
```
SCALE <factor>
```
- Positive float.

---

## 7) SHOW — display image from SD
```
SHOW <filename>
```
- BMP only (uncompressed 16/24‑bit).  
- Honors ORIENT/MIRROR/SCALE/OFFSET.

---

## 8–11) Transitions
```
FADE <ms>
FADEIN <ms>
SLIDEOUT <LEFT|RIGHT|UP|DOWN> <ms>
WIPEOUT <LEFT|RIGHT|UP|DOWN> <ms>
```
- Directions are **viewer‑relative** regardless of ORIENT/MIRROR.

---

## 12–14) Scripts & Timing
```
RUN <filename> [LOOP]
DELAY <ms>
STOP
```
- `RUN` executes a text file of commands with extension **.mhz** (no recursion: `RUN` inside a `RUN` forbidden).  
- `LOOP` repeats until `STOP`.  
- Comments starting with `#` or `;` are ignored.  
- Parser for command files **stops on first error**.

**Defaults/limits for RUN**  
- Max file size: 8 KB; ≤256 lines; ≤128‑byte lines.  
- Safety timeout per `RUN`: 60 s (ignored if `LOOP`).

**Script locations**  
- Relative names (e.g., `RUN demo.mhz`) resolve relative to current working directory on SD (default root).  
- You may also specify paths, e.g., `RUN scripts/logo_cycle.mhz`.

**Startup script (auto‑run on boot)**  
- On boot, after SD mounts, firmware attempts to run a startup script in this order:  
  1) `/startup.mhz` (root)  
  2) `/scripts/startup.mhz`  
- If neither exists → no script is run.  
- If found → executed as if by `RUN`, **stopping on first error**.  
- Recommended contents: `SETBAUD`, `COLOR`, `ORIENT`, splash `SHOW`, etc.  
- Suggested boot log lines: `OK STARTUP /startup.mhz`, `OK STARTUP /scripts/startup.mhz`, or `OK STARTUP NONE`.

---

## 15–19) Binary File Transfer (CRC32‑based)
Purpose: let the host (e.g., DOOM) push assets (e.g., face bitmaps) to SD once, then reuse via `SHOW`.  
Design: **chunked, CRC32‑verified, idempotent**, stop‑and‑wait flow.

> **CRC32 parameters** (for both chunks and total): polynomial 0xEDB88320, init 0xFFFFFFFF, input/output reflected, final XOR 0xFFFFFFFF (ZIP/Ethernet standard).

### 15) FILESTAT — query file on SD
```
FILESTAT <path>
```
Reply:  
- `OK FILE <size> <crc32>` — file exists (crc32 = total file CRC32, hex lowercase).  
- `ERR FILE_NOT_FOUND`.

### 16) PUTBEGIN — start upload
```
PUTBEGIN <path> <size> <crc32_total>
```
- If file exists with same size+CRC32 → `OK SKIP`.  
- Else → prepare temp file, reply `OK READY`.

### 17) PUTCHUNK — send one chunk
```
PUTCHUNK <seq> <offset> <len> <crc32_chunk>
<binary payload of exactly <len> bytes follows immediately>
```
- `<seq>`: sequential chunk index starting at 0.  
- `<offset>`: absolute byte offset.  
- `<len>`: 1..1024 bytes.  
- `<crc32_chunk>`: CRC32 of payload only.

Replies:  
- `OK CHUNK <seq>` — accepted & written.  
- `ERR CHUNK <seq> CHECKSUM` — resend same chunk.  
- `ERR CHUNK <seq> OUT_OF_ORDER` — wrong seq/offset; resend.  
- `ERR CHUNK <seq> RANGE` — `<offset>+<len>` exceeds announced `<size>`.

> **Flow control:** Sender waits for each line reply before sending the next chunk (stop‑and‑wait).

### 18) PUTEND — finalize upload
```
PUTEND <crc32_total>
```
- Close temp file, compute total CRC32, compare to `<crc32_total>`.  
- `OK STORED` on match; else `ERR CHECKSUM` (or `ERR LENGTH` if bytes mismatch).

### 19) PUTABORT — cancel upload
```
PUTABORT
```
- Discard partial; reply `OK ABORTED`.

**Receiver defaults/limits**  
- Max file size: 8 MB.  
- Max chunk: 1024 bytes.  
- One upload at a time; `ERR BUSY` if another active.  
- Timeout: if no chunk ≤5 s after `OK READY`, auto‑abort.

---

## 20) ORIENT — screen rotation (0°/90°/180°/270°)
```
ORIENT <0|90|180|270>
```
- `0` = upright (default), `90` = rotate CW 90°, `180` = upside‑down, `270` = rotate CW 270°.  
- Applies to all rendering (`TEXT`, `SHOW`, transitions).  
- Persist until changed (device may also persist to config if implemented).

Errors: `ERR BAD_ARGS` for other values.

### 21) MIRROR — horizontal/vertical flip
```
MIRROR <NONE|H|V|HV>
```
- `NONE` (default), `H` (horizontal), `V` (vertical), `HV` (both).  
- Applies to all rendering. Useful for unusual mounting/bezels.

Errors: `ERR BAD_ARGS`.

**Transform notes**  
- Transitions remain **viewer‑relative** after rotation/mirroring.  
- 7‑segment text at 90°/270° is rotated as an image (segments rotate accordingly).  
- Composition order: apply **ORIENT** then **MIRROR**.

---

## 22) PING — link test
```
PING
```
- Replies: `OK PONG`.

---

## 23) INFO — firmware/device info
```
INFO
```
- Replies:  
  ```
  OK INFO v0.34 Platform=PicoRP2040 LCD=240x240 SD=OK
  ```
- Fields: firmware version, Platform identifier, LCD resolution, SD status.

---

## 24) COLOR / COLOUR — set text colors
```
COLOR <fg> <bg>
```
- `<fg>` and `<bg>` = 16‑bit RGB565 hex values (e.g., 0xFFFF = white).  
- Applies to `TEXT`.  
- Defaults: fg=0xFFFF, bg=0x0000.

Both spellings accepted.

---

## 25) RESET — restore defaults
```
RESET
```
- Restores: BRIGHT=100, SCALE=1.0, OFFSET=0 0, ORIENT=0, MIRROR=NONE, COLOR fg=white/bg=black, BAUD=9600.  
- Clears screen.  
- Reply: `OK RESET`.

---

## 26) SETBAUD — change serial baud rate
```
SETBAUD <bps>
```
- `<bps>` allowed: 9600, 19200, 38400, 57600, 115200.  
- Reply: `OK SETBAUD <bps>`.  
- Takes effect immediately after reply. Host must switch baud after acknowledgment.  
- Errors: `ERR BAD_ARGS` if unsupported.

---

## Error Codes
`ERR UNKNOWN_COMMAND`, `ERR BAD_ARGS`, `ERR FILE_NOT_FOUND`, `ERR UNSUPPORTED`, `ERR TOO_LONG`, `ERR OUT_OF_RANGE`, `ERR NESTING_LIMIT`, `ERR CHECKSUM`, `ERR RANGE`, `ERR LENGTH`, `ERR OUT_OF_ORDER`, `ERR BUSY`.

---

## Defaults & Notes
- Boot defaults: BRIGHT 100, SCALE 1.0, OFFSET 0 0, ORIENT 0, MIRROR NONE, COLOR fg=white/bg=black, BAUD=9600.  
- Scripts double as animations (`SHOW` + `DELAY`).  
- Binary transfer uses CRC32 for both chunks and whole‑file identity.  
- **Startup script auto‑run search order:** `/startup.mhz` then `/scripts/startup.mhz`.  
- Future extensions (non‑breaking): smoother frame streaming (`SHOWRAW`).  
