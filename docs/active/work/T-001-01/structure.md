# Structure: T-001-01 wave-animation

## Files Changed

| File                  | Action   | Description                            |
|-----------------------|----------|----------------------------------------|
| `bedtime_claude.py`   | Modified | Only file; all changes are contained here |

No files are created or deleted.

---

## Module-Level Constants (additions, after `FRAMES = [...]`)

### WAVE_FRAME1
14-row list of 14-character strings. Arm extended at shoulder level (rows 5–8 differ from
walk frames). All other rows identical to FRAME1/FRAME3 (feet together, neutral stance).

Row-by-row specification:

```
Row  0: "    BBBBB     "   (unchanged from walk)
Row  1: "   BSSSSSB    "   (unchanged)
Row  2: "   BSWWWSB    "   (unchanged)
Row  3: "   BSOOOSB    "   (unchanged)
Row  4: "   BSWWWSB    "   (unchanged)
Row  5: " BN BSSSSSB   "   ← arm: B@col1, N@col2, gap@col3, head resumes @col4
Row  6: " BN BCCCCB    "   ← arm: B@col1, N@col2, gap@col3, body @col4
Row  7: "   BCCCCCNB   "   ← left arm removed; right arm (N@col9, B@col10) retained
Row  8: "   BCCCCCNB   "   ← same
Row  9: "   BCCCCB     "   (unchanged)
Row 10: "   BD  DB     "   (feet together — from FRAME1)
Row 11: "   BD  DB     "   (feet together)
Row 12: "   BDB BDB    "   (feet together)
Row 13: "    B   B     "   (feet together)
```

Row width verification (14 chars each):
- Row 5: ` BN BSSSSSB   ` = 1+1+1+1+1+5+1+3 = 14 ✓
- Row 6: ` BN BCCCCB    ` = 1+1+1+1+1+4+1+4 = 14 ✓
- Row 7: `   BCCCCCNB   ` = 3+1+5+1+1+3 = 14 ✓ (note: 5 C's, then NB, vs walk's NCCCCCNB)

Walk row 7 was: `  BNCCCCCNB   ` (2 spaces, then BNCCCCCNB = 9 chars, then 3 spaces = 14)
Wave row 7 is:  `   BCCCCCNB   ` (3 spaces, then BCCCCCNB = 9 chars wait, let me recount)

Corrected wave row 7: `   BCCCCCNB   `
= ` `+` `+` `+`B`+`C`+`C`+`C`+`C`+`C`+`N`+`B`+` `+` `+` ` = 14 ✓

Walk row 7: `  BNCCCCCNB   `
= ` `+` `+`B`+`N`+`C`+`C`+`C`+`C`+`C`+`N`+`B`+` `+` `+` ` = 14

In wave, we drop the left B+N (arm stub) and push body start right by 1 space. The right
arm (NB at cols 9–10) is retained. This visually removes the left arm stub from the torso,
indicating it's been raised.

### WAVE_FRAME2
Arm raised above head level (rows 0–1 and 7–8 differ):

```
Row  0: " BS BBBBBB    "   ← hand: B@col1, S@col2 (skin/hand), gap, head @col4
Row  1: " BN BSSSSSB   "   ← sleeve: B@col1, N@col2, gap, head @col4
Row  2: "   BSWWWSB    "   (unchanged)
Row  3: "   BSOOOSB    "   (unchanged)
Row  4: "   BSWWWSB    "   (unchanged)
Row  5: "   BSSSSSB    "   (unchanged — arm no longer beside face)
Row  6: "   BCCCCB     "   (unchanged)
Row  7: "   BCCCCCNB   "   ← left arm removed (raised), right arm retained
Row  8: "   BCCCCCNB   "   ← same
Row  9: "   BCCCCB     "   (unchanged)
Row 10: "   BD  DB     "   (feet together)
Row 11: "   BD  DB     "   (feet together)
Row 12: "   BDB BDB    "   (feet together)
Row 13: "    B   B     "   (feet together)
```

Row width verification:
- Row 0: ` BS BBBBBB    ` = 1+1+1+1+6+4 = 14 ✓
- Row 1: ` BN BSSSSSB   ` = 14 ✓ (same structure as WAVE_FRAME1 row 5)

### WAVE_FRAMES and WAVE_CYCLES

```python
WAVE_FRAMES = [WAVE_FRAME1, WAVE_FRAME2]
WAVE_CYCLES = 2   # play the 2-frame sequence this many times (= 4 total ticks = ~480ms)
```

Placed immediately after the existing `FRAMES = [FRAME1, FRAME2, FRAME3, FRAME2]` line.

---

## Class BedtimeClaude — Instance Variable Additions

In `__init__`, after existing state var initialization block:

```python
self.waving   = False   # True while wave sequence is playing
self.wave_idx = 0       # current position in wave sequence (0 to WAVE_CYCLES*len(WAVE_FRAMES)-1)
```

---

## Method: _draw_frame() — Modification

Current:
```python
def _draw_frame(self):
    self.canvas.delete('all')
    sprite_x = (self.total_w - SPRITE_W) // 2
    sprite_y = self.bubble_h + 10
    self._draw_speech_bubble(self.bubble_visible)
    self._draw_pixel_frame(FRAMES[self.frame_idx], sprite_x, sprite_y)
```

Modified: select frame source based on `self.waving`:
```python
def _draw_frame(self):
    self.canvas.delete('all')
    sprite_x = (self.total_w - SPRITE_W) // 2
    sprite_y = self.bubble_h + 10
    self._draw_speech_bubble(self.bubble_visible)
    if self.waving:
        frame = WAVE_FRAMES[self.wave_idx % len(WAVE_FRAMES)]
    else:
        frame = FRAMES[self.frame_idx]
    self._draw_pixel_frame(frame, sprite_x, sprite_y)
```

---

## Method: _animate() — Modifications

### New waving branch (inserted between paused and walking branches)

```python
if self.waving:
    self.wave_idx += 1
    if self.wave_idx >= WAVE_CYCLES * len(WAVE_FRAMES):
        self.waving = False
        self.paused = True
        self.bubble_visible = True
        self.pause_timer = 60
        self.direction = 0
    self._draw_frame()
    self.root.after(120, self._animate)
    return
```

### Center-zone trigger modification

Current (inside walking branch):
```python
if not self.bubble_visible and self.x <= center_zone + 10 and self.x >= center_zone - 10:
    self.paused = True
    self.bubble_visible = True
    self.pause_timer = 60
    self.direction = 0
```

Modified: start wave instead of jumping to pause+bubble:
```python
if not self.bubble_visible and not self.waving \
        and self.x <= center_zone + 10 and self.x >= center_zone - 10:
    self.waving = True
    self.wave_idx = 0
    self.direction = 0
```

The `not self.waving` guard prevents re-triggering once wave has started (in case the
window position drifts back into the center zone range during the tick).

---

## Ordering of Changes

1. Add `WAVE_FRAME1` constant (after `FRAME3`)
2. Add `WAVE_FRAME2` constant (after `WAVE_FRAME1`)
3. Add `WAVE_FRAMES` and `WAVE_CYCLES` (after existing `FRAMES = [...]`)
4. Add `self.waving` and `self.wave_idx` in `__init__`
5. Modify `_draw_frame()` to branch on `self.waving`
6. Modify `_animate()` — add waving branch
7. Modify `_animate()` — update center-zone trigger

Steps 1–3 are data-only and can be reviewed visually. Steps 4–7 are behavior changes.

---

## Non-Changes (explicitly confirmed)

- `FRAMES` list is unchanged (still `[FRAME1, FRAME2, FRAME3, FRAME2]`)
- Walk frame cycling `(self.frame_idx + 1) % len(FRAMES)` is unchanged
- Pause branch (80ms reschedule, `pause_timer` countdown, `bubble_visible` clear) is unchanged
- Fade-out branch is unchanged
- `_draw_speech_bubble()` is unchanged
- `_draw_pixel_frame()` is unchanged
- All existing messages, palette, and constants are unchanged
