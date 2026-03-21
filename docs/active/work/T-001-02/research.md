# Research — T-001-02: snooze-button

## Overview

Single-file Python desktop app (`bedtime_claude.py`, 268 lines). Stdlib only — `tkinter`, `threading`, `time`, `sys`, `random`. No external dependencies.

---

## Class: BedtimeClaude (lines 105–263)

### `__init__` (lines 106–153)

Sets up the tkinter root window and canvas, initializes all animation state, and calls `self._animate()` then `self.root.mainloop()`.

**Window properties:**
- `overrideredirect(True)` — no title bar, no chrome
- `attributes('-topmost', True)` — always on top
- `attributes('-alpha', 0.97)` — near-opaque
- `configure(bg='#000001')` + `wm_attributes('-transparentcolor', '#000001')` — transparent background trick (macOS/Windows)

**Canvas layout:**
- `total_w = max(bubble_w + 20, SPRITE_W + 40)` — 300px wide
- `total_h = SPRITE_H + bubble_h + 20` — sprite below bubble zone
- Single `tk.Canvas` fills the window

**State fields:**
- `self.x`, `self.y` — window position (screen coords)
- `self.frame_idx` — current walk animation frame (0–3)
- `self.direction` — movement direction: `-1` left, `0` stopped, `-1` resumed
- `self.speed` — pixels per tick (starts 3, increases to 4 after pause)
- `self.paused` — bool: currently paused in center showing bubble
- `self.pause_timer` — countdown ticks (60 ticks ≈ 5 seconds at 80ms)
- `self.bubble_visible` — bool: speech bubble drawn
- `self.fade_out` — bool: fading out
- `self.alpha` — current window alpha (0.97 → 0 during fade)

---

## Methods

### `_draw_pixel_frame(frame_data, offset_x, offset_y)` (lines 155–166)

Iterates the 14-row string array, maps chars to `PALETTE` colors, draws filled rectangles (`PIXEL=5` display pixels each). Transparent pixels (` ` or missing) are skipped.

### `_draw_speech_bubble(show)` (lines 168–214)

Draws a rounded blue rectangle + tail polygon + border arcs/lines + text. Text is the globally-selected `MESSAGE` string. If `show` is False, returns immediately (no-op).

The bubble text is hardcoded to the `MESSAGE` global at draw time (line 209: `text=MESSAGE`).

### `_draw_frame()` (lines 216–221)

Clears canvas (`delete('all')`), draws bubble (if `bubble_visible`), draws sprite centered horizontally at `sprite_y = bubble_h + 10`.

### `_animate()` (lines 223–263)

The main loop — called recursively via `root.after()`. Three branches:

1. **Fade out** (`self.fade_out`): decrements alpha by 0.04 every 40ms, destroys root when alpha ≤ 0.
2. **Paused** (`self.paused`): decrements `pause_timer` each 80ms tick. When timer hits 0: clears `paused`, hides bubble, resumes leftward movement at speed 4.
3. **Walking**: moves window by `direction * speed`, checks for center-screen pause trigger, checks for off-screen-left fade trigger, cycles frame index.

**Timing:**
- Walking ticks: 120ms (`root.after(120, self._animate)`)
- Paused ticks: 80ms (`root.after(80, self._animate)`)
- Fade ticks: 40ms (`root.after(40, self._animate)`)

---

## Animation State Machine

```
WALKING (x from sw+20, moving left)
  → reaches center ± 10px → PAUSED (timer=60, bubble_visible=True)
      → timer hits 0 → WALKING (speed=4, bubble_visible=False)
          → x < -total_w - 20 → FADE_OUT
              → alpha ≤ 0 → root.destroy()
```

---

## No Click Handling (Current State)

`BedtimeClaude.__init__` does **not** bind any click events to the canvas or root window. No `<Button-1>` binding exists anywhere in the file. This is the gap this ticket fills.

---

## Relevant Constraints

### Transparency and Hit-Testing

The transparent color trick (`#000001` as transparent) means mouse clicks on transparent pixels may pass through to underlying windows on macOS. Click events on actual drawn pixels (sprite, bubble) should be catchable via canvas bindings.

### Thread Safety

`_animate()` runs entirely on the main tkinter thread via `root.after()`. There is no separate animation thread. This is intentional — tkinter is not thread-safe and all widget operations must happen on the main thread.

`threading` is imported but not used in the current code.

### Process Lifecycle

The app runs as a foreground process. `root.mainloop()` blocks `__init__` until the window is destroyed. When the window is destroyed (`root.destroy()`), the process exits naturally.

The script path is hardcoded in the cron entry: `/Users/KaterinaHuezo/Documents/projects/go-to-bed/bedtime_claude.py`. The snooze relaunch needs to find this path at runtime.

### `sys.argv[0]` and `__file__`

`sys` is imported. `__file__` is available at module level and gives the script's absolute path — reliable for relaunch without hardcoding.

### `subprocess` Module

Not currently imported. Required for the snooze relaunch background process.

---

## Snooze Trigger Window

The ticket requires clicking to trigger snooze "while the bubble is visible" (`self.bubble_visible == True`). This is only True during the PAUSED state. The acceptance criteria further require that clicking during walk (outside pause/bubble phase) does nothing.

The `self.paused` flag (or `self.bubble_visible`) is the correct guard for this.

---

## Fade-Out Path

Current fade-out is triggered only when `self.x < -self.total_w - 20` (walked off left edge). The snooze should reuse this same fade-out sequence, but after fade completes, instead of just destroying the root (line 227), it needs to also launch a background relaunch subprocess.

---

## Message Text

`MESSAGE` is a module-level global set once at import time via `random.choice(MESSAGES)`. The snooze message ("Fine. 15 more minutes. 😑") needs to replace this text temporarily in the bubble. Options: mutate the global, or pass a parameter through `_draw_speech_bubble`.

`_draw_speech_bubble` currently reads `MESSAGE` directly (line 209). It does not take a `text` parameter.

---

## Files

| File | Role |
|------|------|
| `bedtime_claude.py` | Entire app — the only file to modify |

---

## Summary of Gaps

1. No click handler on canvas/root
2. `subprocess` not imported
3. `_draw_speech_bubble` hardcoded to `MESSAGE` global
4. Fade-out completion (`root.destroy()`) does not schedule relaunch
5. No mechanism to guard snooze clicks to the paused phase only
6. No cleanup/deduplication guard against zombie relaunched processes
