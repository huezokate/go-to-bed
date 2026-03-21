# Plan — T-001-02: snooze-button

## Ordered Implementation Steps

### Step 1 — Add imports

Add `os` and `subprocess` to the import block.

**File:** `bedtime_claude.py`, lines 7–11
**Change:** Insert `import os` and `import subprocess` after `import sys`

**Verify:** File parses cleanly (`python3 -c "import bedtime_claude"` without running GUI; or just check no syntax error).

---

### Step 2 — Add instance state fields

Add `self.snooze_mode = False` and `self._launch_snooze = False` to `__init__`, after the existing `self.fade_out` and `self.alpha` lines.

**File:** `bedtime_claude.py`, `__init__` method
**Change:** Two new lines after line 148

**Verify:** `self.snooze_mode` and `self._launch_snooze` accessible on instance.

---

### Step 3 — Update `_draw_speech_bubble` to support snooze text

Replace `text=MESSAGE` with a local variable that checks `self.snooze_mode`.

**File:** `bedtime_claude.py`, `_draw_speech_bubble` method
**Change:**
```python
# before the create_text call:
display_text = "Fine. 15 more minutes. 😑" if self.snooze_mode else MESSAGE
# in create_text:
text=display_text,
```

**Verify:** Normal run still shows original message. (Snooze text not yet reachable until click handler is added.)

---

### Step 4 — Add canvas click binding in `__init__`

Bind `<Button-1>` on the canvas to `self._on_canvas_click`.

**File:** `bedtime_claude.py`, `__init__` method
**Change:** Add `self.canvas.bind('<Button-1>', self._on_canvas_click)` before the `self._draw_frame()` call

**Verify:** App still launches; clicking does nothing yet (method doesn't exist yet — add a stub or do steps 4–5 together).

*Note: Add steps 4 and 5 together since the binding references the method.*

---

### Step 5 — Implement `_on_canvas_click`

Add the click handler method.

**File:** `bedtime_claude.py`, new method after `_draw_frame`
**Change:**
```python
def _on_canvas_click(self, event):
    if not self.bubble_visible or self.fade_out:
        return
    self.snooze_mode = True
    self._draw_frame()
    self.root.after(800, self._begin_snooze_fade)
```

**Verify (manual):**
- Launch app, click canvas while walking → nothing happens (bubble not visible)
- Launch app, wait for Claude to reach center and bubble to appear → click → bubble text changes to "Fine. 15 more minutes. 😑"
- 800ms later the app should start fading (once step 6 is in)

---

### Step 6 — Implement `_begin_snooze_fade`

Add the method that triggers the fade from snooze context.

**File:** `bedtime_claude.py`, new method after `_on_canvas_click`
**Change:**
```python
def _begin_snooze_fade(self):
    self._launch_snooze = True
    self.fade_out = True
```

**Verify (manual):** After clicking during bubble phase — snooze message appears for ~800ms, then window fades out at the same rate as normal exit.

---

### Step 7 — Implement `_spawn_snooze_relaunch`

Add the subprocess spawn method.

**File:** `bedtime_claude.py`, new method after `_begin_snooze_fade`
**Change:**
```python
def _spawn_snooze_relaunch(self):
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

**Verify (unit):** Call the method directly in a test; confirm a child process exists with `sleep 900` in its args; kill it immediately after to avoid waiting 15 minutes.

---

### Step 8 — Hook spawn into `_animate` fade completion

Modify the fade-out branch to call `_spawn_snooze_relaunch` when `_launch_snooze` is True.

**File:** `bedtime_claude.py`, `_animate` method
**Change:** Before `self.root.destroy()` in the fade branch:
```python
if self.alpha <= 0:
    if self._launch_snooze:
        self._spawn_snooze_relaunch()
    self.root.destroy()
    return
```

**Verify (manual):** Full snooze flow: click during bubble → snooze text → fade → window closes. Check `ps aux | grep sleep` shows a `sleep 900` process. Normal walk-off path: no `sleep 900` process spawned.

---

## Testing Strategy

### Manual Test Cases

| # | Scenario | Expected |
|---|----------|----------|
| 1 | Launch; click canvas while Claude is walking | Nothing happens |
| 2 | Wait for bubble; click | Bubble text changes to snooze message |
| 3 | After snooze click; wait ~800ms | Window begins fading |
| 4 | After fade completes | `ps aux | grep "sleep 900"` shows background process |
| 5 | Normal walk-off (no click) | No `sleep 900` process |
| 6 | Click multiple times during bubble | Only one background process spawned (double-click guard: `self.fade_out` check) |
| 7 | Force-quit window during walk | No background process |
| 8 | Force-quit window after snooze click but before fade completes | Background process may or may not be spawned depending on timing — acceptable |

### Verification of No Zombie

After test case 4:
```bash
ps aux | grep "sleep 900"
# should show one process owned by user
# kill it: kill <pid>
```

After test case 5:
```bash
ps aux | grep "sleep 900"
# should show nothing
```

### Double-Click Guard

The `self.fade_out` check in `_on_canvas_click` prevents a second snooze spawn if the user clicks again during the 800ms snooze message display window. Once `_begin_snooze_fade` sets `fade_out=True`, subsequent clicks are no-ops.

---

## Commit Strategy

All changes are in a single file, tightly coupled. One atomic commit:

```
feat: add snooze button — click bubble to dismiss and relaunch in 15min
```

The change is small (~31 lines net), reviewable in one pass.

---

## Risks

| Risk | Mitigation |
|------|------------|
| Script path with spaces in `__file__` | `os.path.abspath(__file__)` returns the real path; if it contains spaces, the shell command `sh -c 'sleep 900 && python3 /path with spaces/...'` would break. Use `shlex.quote` or pass as list with explicit args. |
| `sys.executable` path with spaces | Same risk; same fix. |
| macOS transparent window click passthrough | Clicks on `#000001` transparent pixels may not register. Acceptable — clicking on the sprite or bubble (opaque) will register. Documented limitation. |

### Path-with-Spaces Fix (preemptive)

Apply in Step 7: use `shlex.quote` for both paths in the shell command, or restructure to use a Python wrapper instead of shell `&&`.

Actually cleaner: use a two-argument `Popen` that runs Python directly after the sleep, avoiding shell interpolation entirely:

```python
# Safer alternative if paths have spaces:
subprocess.Popen(
    [python, '-c', f'import time; time.sleep(900); import subprocess; subprocess.Popen(["{script}"])'],
    start_new_session=True,
    close_fds=True,
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
)
```

But this is harder to read. The `shlex.quote` approach keeps it clean:

```python
import shlex
cmd = f'sleep 900 && {shlex.quote(python)} {shlex.quote(script)}'
subprocess.Popen(['sh', '-c', cmd], ...)
```

Apply this in Step 7.
