# Structure — T-001-02: snooze-button

## Files Changed

| File | Change |
|------|--------|
| `bedtime_claude.py` | Modified — only file in the project |

No new files. No deletions.

---

## Import Changes

**Add** `subprocess` and `os` to the existing import block (lines 7–11):

```python
import tkinter as tk
import threading    # already present, still unused but harmless
import time
import sys
import os           # ADD
import subprocess   # ADD
```

`os` is needed for `os.path.abspath(__file__)` to get the script's own path at runtime.

---

## New Instance State (`__init__`)

Two new boolean fields added to `__init__` after the existing state fields (around line 148):

```python
self.snooze_mode = False      # True after snooze click — changes bubble text
self._launch_snooze = False   # True if fade was triggered by snooze click
```

These are initialized alongside `self.fade_out`, `self.alpha`, etc.

---

## `_draw_speech_bubble` — Text Selection

**Current** (line 209):
```python
text=MESSAGE,
```

**New** (conditional inside the method body, before the `create_text` call):
```python
display_text = "Fine. 15 more minutes. 😑" if self.snooze_mode else MESSAGE
```

Then:
```python
text=display_text,
```

The method signature does not change. All existing call sites remain valid.

---

## New Method: `_on_canvas_click(event)`

Added to `BedtimeClaude`, between `_draw_frame` and `_animate`:

```python
def _on_canvas_click(self, event):
    """Handle canvas click — snooze if bubble is visible."""
    if not self.bubble_visible or self.fade_out:
        return
    self.snooze_mode = True
    self._draw_frame()
    self.root.after(800, self._begin_snooze_fade)
```

Guards:
- `not self.bubble_visible` — no-op during walk phase
- `self.fade_out` — no-op if already fading (prevents double-trigger)

After setting `snooze_mode`, immediately redraws to show the snooze message, then schedules the fade start 800ms later.

---

## New Method: `_begin_snooze_fade()`

```python
def _begin_snooze_fade(self):
    """Transition from snooze message display into fade-out."""
    self._launch_snooze = True
    self.fade_out = True
```

This is called 800ms after the snooze click. At this point `_animate` is still running its `root.after` loop. On the next tick it will see `self.fade_out = True` and begin the fade sequence.

No need to call `_animate()` directly — the existing loop handles it.

---

## `_animate` — Fade Completion Branch

**Current** (lines 224–231):
```python
if self.fade_out:
    self.alpha -= 0.04
    if self.alpha <= 0:
        self.root.destroy()
        return
    self.root.attributes('-alpha', max(0, self.alpha))
    self.root.after(40, self._animate)
    return
```

**New** — insert subprocess spawn before `root.destroy()`:
```python
if self.fade_out:
    self.alpha -= 0.04
    if self.alpha <= 0:
        if self._launch_snooze:
            self._spawn_snooze_relaunch()
        self.root.destroy()
        return
    self.root.attributes('-alpha', max(0, self.alpha))
    self.root.after(40, self._animate)
    return
```

---

## New Method: `_spawn_snooze_relaunch()`

```python
def _spawn_snooze_relaunch(self):
    """Launch a detached background process to relaunch this script in 15 minutes."""
    script = os.path.abspath(__file__)
    python = sys.executable
    subprocess.Popen(
        ['sh', '-c', f'sleep 900 && {python} {script}'],
        start_new_session=True,
        close_fds=True,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )
```

Isolated into its own method for clarity and testability. Called only from `_animate` when `_launch_snooze` is True and alpha ≤ 0.

---

## Canvas Click Binding

In `__init__`, after `self.canvas.pack()` and before `self._draw_frame()`:

```python
self.canvas.bind('<Button-1>', self._on_canvas_click)
```

Binding on the canvas (not the root window) is correct — the canvas covers the entire window, and we want clicks on any drawn pixel (sprite or bubble) to trigger the handler. Transparent-color pixels on macOS will pass through, but drawn pixels will be caught.

---

## Method Order in Class

Final method order in `BedtimeClaude`:

1. `__init__`
2. `_draw_pixel_frame`
3. `_draw_speech_bubble`
4. `_draw_frame`
5. `_on_canvas_click`       ← NEW
6. `_begin_snooze_fade`     ← NEW
7. `_spawn_snooze_relaunch` ← NEW
8. `_animate`

---

## Module-Level Changes

None. `PALETTE`, `FRAMES`, `MESSAGES`, `MESSAGE`, `PIXEL`, `SPRITE_W`, `SPRITE_H` are all unchanged.

---

## Interface Summary

No public API. The app is a single `__main__` script. All changes are internal to the `BedtimeClaude` class.

---

## Change Size Estimate

| Location | Lines changed |
|----------|--------------|
| Import block | +2 |
| `__init__` state fields | +2 |
| `_draw_speech_bubble` text | +2 (replace 1) |
| `__init__` canvas binding | +1 |
| `_on_canvas_click` (new method) | ~8 |
| `_begin_snooze_fade` (new method) | ~4 |
| `_spawn_snooze_relaunch` (new method) | ~10 |
| `_animate` fade branch | +2 (insert) |
| **Total** | **~31 lines net** |

The script grows from 268 lines to approximately 300 lines.
