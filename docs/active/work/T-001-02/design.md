# Design — T-001-02: snooze-button

## Problem Statement

Add snooze behavior: clicking the canvas while the bubble is visible should:
1. Replace bubble text with "Fine. 15 more minutes. 😑"
2. Fade out (same as normal exit)
3. Launch a background process that waits 900 seconds then relaunches the script
4. Leave no zombie processes if the window is closed via other means

---

## Approach A — `subprocess` + shell `sleep 900 && python3 script` detached

Spawn a detached shell command: `sh -c 'sleep 900 && python3 /path/to/script &'` using `subprocess.Popen` with `start_new_session=True` (Unix) to detach from the parent process group.

**How it works:** The shell process runs in the background. When the parent Python process exits, the shell process (and its `sleep 900`) continues independently because it's in its own session. After 900 seconds, `python3 script` launches.

**Pros:**
- Simple, no daemon threads needed
- `start_new_session=True` ensures the child is fully detached — no zombie risk
- Works on macOS with stdlib only
- If Claude's window is force-closed mid-fade, the background shell process still runs (this is desired behavior per the ticket: snooze was already committed to)

**Cons:**
- Two processes (shell + sleep) visible in `ps` during the 15-minute window — minor cosmetic issue
- Shell dependency (sh), though that's always available on macOS

---

## Approach B — Python `threading.Timer` with `subprocess.Popen` after delay

Set a `threading.Timer(900, launch_fn)` where `launch_fn` calls `subprocess.Popen([sys.executable, script_path])`. The timer thread lives inside the current process.

**Pros:**
- Pure Python, no shell invocation
- Timer is inspectable / cancellable

**Cons:**
- The timer lives in the current process. Since `root.destroy()` exits the mainloop but doesn't kill threads, the process stays alive for 15 minutes with the timer thread running. This is effectively a zombie Python process consuming memory.
- On macOS, a headless process with an open tkinter import may behave unexpectedly.
- More complex: need to ensure the process exits cleanly after the timer fires, not before.

**Rejected** — keeping the parent process alive for 900 seconds just to schedule a relaunch is wasteful and fragile.

---

## Approach C — Write a `launchd` plist or `at` job

Use `atq`/`at` or `launchd` to schedule a one-shot job.

**Pros:** System-native scheduling

**Cons:**
- `at` is disabled by default on modern macOS
- `launchd` plist creation is complex and overkill for a 15-minute one-shot
- Requires elevated permissions or user-specific setup

**Rejected** — complexity far exceeds the need.

---

## Decision: Approach A

Use `subprocess.Popen` with `start_new_session=True` to spawn a detached shell command:

```python
import subprocess, sys, os

script = os.path.abspath(__file__)
subprocess.Popen(
    ['sh', '-c', f'sleep 900 && {sys.executable} {script}'],
    start_new_session=True,
    close_fds=True,
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
)
```

`start_new_session=True` calls `setsid()` on the child, placing it in a new process group and session. The child is fully independent — when the parent exits, the child is reparented to PID 1 (launchd on macOS), not killed. No zombies because the shell exits cleanly after spawning the final `python3` command, and launchd reaps orphans.

---

## Bubble Text Override

`_draw_speech_bubble` currently reads the `MESSAGE` global directly (line 209). Rather than introducing a parameter (which would require updating all call sites), the cleanest approach is:

- Add a `self.snooze_mode` boolean (default False)
- In `_draw_speech_bubble`, read `self.snooze_mode` to decide which text to display

This keeps the existing call signature intact and is readable at the draw site.

Alternative: add an optional `text=None` parameter to `_draw_speech_bubble` and pass it from `_draw_frame`. This is slightly cleaner but requires changes to `_draw_frame` as well. Given there are only two text states (normal and snooze), the boolean is simpler.

**Decision:** `self.snooze_mode` boolean on the instance.

---

## Click Handler Design

Bind `<Button-1>` to the canvas in `__init__`. The handler:

1. Checks `self.bubble_visible` — if False, returns (no-op during walk)
2. Checks `self.fade_out` — if True, returns (already fading, don't double-trigger)
3. Sets `self.snooze_mode = True`
4. Calls `_draw_frame()` to immediately update bubble text
5. Schedules `self._begin_snooze_fade()` via `root.after(800, ...)` — brief pause so user sees the snooze message before fade
6. Sets `self.bubble_visible` to keep bubble shown during that 800ms

`_begin_snooze_fade()`:
1. Sets `self.fade_out = True`
2. Sets `self._launch_snooze = True` (flag to trigger subprocess on fade completion)
3. Calls `_animate()` is already running via `root.after`, so no manual trigger needed — the next `_animate` tick will see `fade_out=True` and begin fading

On fade completion (where `root.destroy()` currently is):
- Check `self._launch_snooze`; if True, spawn the background process before destroying

---

## Zombie Prevention

The acceptance criterion "No zombie processes left behind if the user closes the window through other means" means: if the user force-quits or the window is closed before snooze is triggered, no background process should exist.

The background subprocess is only spawned at fade completion, inside `_animate()` when `alpha <= 0`. If the window is killed externally (e.g., `kill` signal or Force Quit), `_animate` never reaches that branch — the subprocess is never spawned. No zombie.

If the window is closed via Cmd+Q or the OS closes it, `root.destroy()` is called from outside — again, `_animate` never spawns the subprocess. Clean.

The only case where the 15-minute background process exists is when the user deliberately clicked to snooze and the fade completed normally. This is correct behavior.

---

## State Machine (Updated)

```
WALKING
  → center ± 10px → PAUSED (bubble_visible=True, pause_timer=60)
      [click during PAUSED] → SNOOZE_PENDING (snooze_mode=True, draw updated bubble, after 800ms → fade_out=True, _launch_snooze=True)
      → timer hits 0 → WALKING (normal exit path)
          → x < -total_w - 20 → FADE_OUT (_launch_snooze=False)
              → alpha ≤ 0 → root.destroy()

SNOOZE_PENDING → FADE_OUT (_launch_snooze=True)
    → alpha ≤ 0 → spawn background process → root.destroy()
```

---

## What Is Not Changing

- Pixel art frames, palette, animation timing
- Normal walk-off fade path
- Window setup / transparency trick
- `_draw_pixel_frame` method
- Message selection logic (the `MESSAGES` list and `MESSAGE` global)
