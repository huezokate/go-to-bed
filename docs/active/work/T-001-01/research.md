# Research: T-001-01 wave-animation

## Overview

`bedtime_claude.py` is a single-file Python/tkinter desktop app. A pixel-art Claude sprite
walks right-to-left across the bottom of the screen, pauses at center to show a speech
bubble, then walks off the left edge and fades out. All animation logic lives in one class.

---

## File: bedtime_claude.py

### Pixel Art System

Frames are lists of 14-character strings, each representing a 14×14 pixel grid. Characters
map to colors via `PALETTE`:

```
' '=transparent  'B'=dark outline  'S'=skin  'W'=white  'O'=orange
'R'=rust  'G'=gray  'P'=pink  'Y'=yellow  'D'=dark legs  'L'=light
'C'=Claude blue  'N'=dark blue (sleeve/arm)
```

Each character becomes a `PIXEL=5` px square drawn by `canvas.create_rectangle`. Row 0 is
the top of the sprite; row 13 is the bottom.

**Current walk frames (FRAME1–FRAME3):**

```
Row  0: "    BBBBB     "   head top
Row  1: "   BSSSSSB    "   face top
Row  2: "   BSWWWSB    "   eyes (white)
Row  3: "   BSOOOSB    "   orange nose/stripe
Row  4: "   BSWWWSB    "   eyes (white)
Row  5: "   BSSSSSB    "   face bottom / chin
Row  6: "   BCCCCB     "   body top
Row  7: "  BNCCCCCNB   "   body mid — N at col 2 (left arm), N at col 10 (right arm)
Row  8: "  BNCCCCCNB   "   body mid (same)
Row  9: "   BCCCCB     "   body bottom
Row 10: varies per frame  — leg positions
Row 11: varies per frame
Row 12: varies per frame
Row 13: varies per frame
```

Arms appear as single `N`/`B` pixels flanking the torso at rows 7–8:
- Left arm (viewer's left) at col 2 (`B`) and col 3 (`N`) within those rows
- Right arm at col 9 (`N`) and col 10 (`B`)

`FRAMES = [FRAME1, FRAME2, FRAME3, FRAME2]` — 4-frame walk cycle. All walk frames share
identical head/body rows 0–9; only rows 10–13 (legs) differ.

`SPRITE_W = 14 * 5 = 70 px`
`SPRITE_H = 14 * 5 = 70 px`

---

### Window Layout

```
total_w = max(bubble_w + 20, SPRITE_W + 40) = max(300, 110) = 300 px
total_h = SPRITE_H + bubble_h + 20 = 70 + 80 + 20 = 170 px
```

Canvas layers (bottom-to-top in draw order within `_draw_frame`):
1. Speech bubble (drawn first if `bubble_visible`)
2. Sprite (drawn on top via `_draw_pixel_frame`)

Sprite is centered horizontally in the window:
```python
sprite_x = (total_w - SPRITE_W) // 2  # = (300 - 70) // 2 = 115
sprite_y = bubble_h + 10               # = 80 + 10 = 90
```

---

### Animation State Machine

State variables on `BedtimeClaude`:

| Variable         | Type  | Initial | Description                              |
|------------------|-------|---------|------------------------------------------|
| `frame_idx`      | int   | 0       | Index into `FRAMES`                      |
| `direction`      | int   | -1      | -1=left, 0=stopped, 1=right              |
| `speed`          | int   | 3       | Pixels per tick when moving              |
| `paused`         | bool  | False   | True during center-stop with bubble      |
| `pause_timer`    | int   | 0       | Ticks remaining in pause (60 → ~5 sec)   |
| `bubble_visible` | bool  | False   | Controls speech bubble render            |
| `fade_out`       | bool  | False   | Triggers alpha fade and window destroy   |
| `alpha`          | float | 0.97    | Current window alpha                     |

**`_animate()` control flow (called every 120ms normally, 80ms during pause, 40ms fade):**

```
if fade_out:
    decrement alpha, destroy at 0, reschedule at 40ms
    return

if paused:
    decrement pause_timer
    if pause_timer == 0: clear paused/bubble, set direction=-1, speed=4
    redraw, reschedule at 80ms
    return

# walking branch:
move x by direction*speed
if x in center_zone ± 10 and not bubble_visible:
    set paused=True, bubble_visible=True, pause_timer=60, direction=0
if x < -(total_w + 20):
    set fade_out=True
cycle frame_idx = (frame_idx + 1) % len(FRAMES)
redraw, reschedule at 120ms
```

**Key observation:** `bubble_visible` is set to `True` in the same tick that `paused` is set.
There is no intermediate state between walking and paused-with-bubble.

---

### Center Zone Trigger

```python
center_zone = self.sw // 2 - self.total_w // 2
if not self.bubble_visible and self.x <= center_zone + 10 and self.x >= center_zone - 10:
    ...
```

This fires once (guarded by `not self.bubble_visible`). The ±10 px window ensures the
trigger is hit even if `speed=3` steps skip the exact pixel.

---

### Drawing Methods

- `_draw_frame()`: clears canvas, draws bubble (if visible), draws current frame
- `_draw_pixel_frame(frame_data, offset_x, offset_y)`: iterates rows/cols, skips
  transparent (' '), draws colored rectangles
- `_draw_speech_bubble(show)`: draws rounded rect + tail + text using `create_*` primitives

---

## Constraints and Observations Relevant to Ticket

1. **Single file**: all changes go in `bedtime_claude.py`.

2. **No new dependencies**: stdlib only — no PIL, no image files.

3. **14-char row width is fixed** by `SPRITE_W = 14 * PIXEL`. Any wave frame must also use
   14-character rows to avoid rendering misalignment.

4. **Frame rows 0–9 are identical across all walk frames** (head + body). Wave frames can
   share this upper half and only differ in the arm position and leg position.

5. **Arm representation**: arms are single-char stubs at cols 2–3 (left arm) and cols 9–10
   (right arm) in rows 7–8. There is visual space to extend an arm upward toward or above
   the head at cols 1–2 without overlapping the head (head occupies cols 3–9 in rows 0–5).

6. **120ms tick rate**: the acceptance criteria require wave frames to cycle at 120ms —
   the same rate as walk frames. This means no tick-rate change is needed; the walk branch
   already fires every 120ms and cycles `frame_idx`.

7. **State transition point**: the wave must be inserted between the center-zone trigger
   (currently sets `paused=True` and `bubble_visible=True` immediately) and the moment
   `bubble_visible` actually becomes `True`.

8. **Feet during wave**: acceptance criteria do not specify feet behavior during wave. Using
   a neutral standing pose (feet together, as in FRAME1/FRAME3) is the natural choice for
   a static wave.

9. **Frame cycling during wave**: if wave frames are added to the `FRAMES` list, the
   existing `(frame_idx + 1) % len(FRAMES)` loop would mix wave and walk frames. A
   separate wave frame list or explicit index range is needed.

10. **No visual glitch criterion**: walk ends → wave plays → pause/bubble begins. The
    transition frames must be compatible with the head/body pixel layout so no artifacts
    appear at frame boundaries.
