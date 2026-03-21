# Review — T-001-02: snooze-button

## Summary

Single file modified: `bedtime_claude.py`. Script grew from 268 to 303 lines (+35 net). The change adds a snooze interaction to the existing bedtime animation — clicking the speech bubble dismisses Claude and schedules a 15-minute relaunch.

---

## Files Changed

| File | Change | Lines |
|------|--------|-------|
| `bedtime_claude.py` | Modified | +35 |

No files created or deleted.

---

## What Changed

### Imports (lines 11–13)
Added `os`, `shlex`, `subprocess`. All stdlib. `shlex` used for safe path quoting in shell command construction.

### `__init__` additions (lines 152–155)
Two new state fields:
- `self.snooze_mode = False` — controls which text `_draw_speech_bubble` renders
- `self._launch_snooze = False` — flag read at fade completion to decide whether to spawn the relaunch process

Canvas click binding: `self.canvas.bind('<Button-1>', self._on_canvas_click)`

### `_draw_speech_bubble` (line 213)
`text=MESSAGE` replaced with a local `display_text` that switches on `self.snooze_mode`. The snooze message ("Fine. 15 more minutes. 😑") is shown when `snooze_mode` is True.

### `_on_canvas_click` (lines 230–236) — NEW
Guards: returns immediately if `bubble_visible` is False (walk phase) or `fade_out` is True (already fading — double-click protection). Sets `snooze_mode`, redraws to show snooze text, schedules `_begin_snooze_fade` in 800ms.

### `_begin_snooze_fade` (lines 238–241) — NEW
Sets `_launch_snooze = True` and `fade_out = True`. The existing `_animate` loop picks this up on its next tick (≤40ms) and begins the same fade sequence used for normal exit.

### `_spawn_snooze_relaunch` (lines 243–254) — NEW
Constructs `sh -c 'sleep 900 && python3 /path/to/script'` with `shlex.quote` on both the interpreter and script paths. Spawns via `subprocess.Popen` with `start_new_session=True` (setsid — fully detached), `close_fds=True`, stdout/stderr to DEVNULL.

### `_animate` fade branch (lines 260–261)
Added two lines before `root.destroy()`: checks `self._launch_snooze` and calls `_spawn_snooze_relaunch()` if True.

---

## Acceptance Criteria Verification

| Criterion | Status | Notes |
|-----------|--------|-------|
| Clicking anywhere on canvas while bubble visible triggers snooze | DONE | `<Button-1>` on canvas; guarded by `bubble_visible` |
| On snooze: window fades out (same fade as normal exit) | DONE | Reuses existing `fade_out` path in `_animate` |
| After fade, background process scheduled for 15 minutes | DONE | `_spawn_snooze_relaunch` called at `alpha <= 0` |
| Snooze message replaces bubble text briefly before fade | DONE | 800ms display window; `snooze_mode` flag drives text |
| Clicking while walking does nothing | DONE | `not self.bubble_visible` guard in `_on_canvas_click` |
| No zombie processes if window closed via other means | DONE | Subprocess only spawned at `alpha <= 0`; external kills never reach that branch |

---

## Test Coverage

No automated tests exist in the project (stdlib-only, single-file desktop app with a GUI loop). All verification is manual.

**Manual test matrix:**

| Test | How to verify |
|------|--------------|
| Walk-phase click does nothing | Launch app; click before Claude reaches center — no visible change |
| Bubble-phase click shows snooze text | Wait for bubble; click — "Fine. 15 more minutes. 😑" appears |
| Fade starts after 800ms | Observe fade begins ~800ms after click |
| Background process spawned | After fade: `ps aux \| grep "sleep 900"` shows one process |
| Normal exit has no background process | Let Claude walk off naturally; `ps aux \| grep "sleep 900"` is empty |
| Double-click protection | Click twice rapidly; only one `sleep 900` process in `ps` |
| External kill — no zombie | `kill -9 <pid>` during walk; no `sleep 900` in `ps` |

---

## Open Concerns

### 1. Background process persists on reboot (minor)
If the user shuts down or reboots within the 15-minute window, the `sleep 900` process is killed by the OS and Claude will not reappear. Acceptable for a casual bedtime reminder — no persistent scheduling was requested.

### 2. Snooze can only fire once per session
Once `snooze_mode = True` and `fade_out = True`, the window fades and exits. There's no "chain snooze" (clicking snooze on the relaunched app triggers another snooze). This works correctly — each relaunch is a fresh process with fresh state.

### 3. macOS transparent pixel click passthrough
On macOS, clicks on `#000001` transparent pixels may pass through to underlying windows. The canvas binding catches clicks on drawn pixels. A click on the transparent gap between sprite and bubble edge may not register. In practice, the visible sprite and bubble fill most of the canvas, so this is unlikely to frustrate users. Not a regression — click handling didn't exist before.

### 4. `bubble_visible` guard vs. `paused` guard
The click guard uses `self.bubble_visible` (not `self.paused`). These are set together when the pause begins, but `paused` is cleared (timer hits 0) while `bubble_visible` is also cleared simultaneously. The behavior is equivalent. Using `bubble_visible` is slightly more semantically direct for the intent "only snooze when bubble is shown."

### 5. 800ms display window hardcoded
The snooze message is shown for 800ms before fade. This is a reasonable "flash" duration but is not configurable. No concern for the current use case.

---

## Known Limitations

- No automated test suite — GUI + mainloop makes unit testing non-trivial without mocking tkinter
- The 15-minute delay is hardcoded (900 seconds in shell command); not exposed as a parameter

---

## Critical Issues

None. The change is small, additive, and confined to a single file. All acceptance criteria are met. Paths are safely quoted. The subprocess spawn is guarded to prevent execution outside the intended code path.
