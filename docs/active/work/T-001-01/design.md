# Design: T-001-01 wave-animation

## Problem Restatement

Insert a short wave animation (2+ frames) between the walk-to-center arrival and the
speech-bubble display. Wave frames must cycle at 120ms. After the sequence completes,
existing pause+bubble behavior resumes unchanged.

---

## Options Considered

### Option A — Append Wave Frames to FRAMES, use index ranges

Add WAVE_FRAME1 and WAVE_FRAME2 to the end of `FRAMES`. Track a mode with an integer
phase or enum: `MODE_WALK`, `MODE_WAVE`, `MODE_PAUSE`. In the walking branch, when the
center zone triggers, switch to `MODE_WAVE` and reset `frame_idx` to the first wave frame
index. In the wave branch, advance `frame_idx` through wave frame indices; on completion,
switch to `MODE_PAUSE` and set `bubble_visible`.

**Pros:**
- Wave frames live in the same `FRAMES` list as walk frames — single source of truth for
  all pixel art.
- `_draw_frame()` needs no changes; it already uses `FRAMES[self.frame_idx]`.

**Cons:**
- `frame_idx` now has meaning tied to mode. A bug that resets `frame_idx = 0` during wave
  mode would snap back to a walk frame silently.
- len(FRAMES) grows; the walk cycle `(frame_idx + 1) % len(FRAMES)` would roll over into
  wave frames if mode logic is ever missed.
- Index arithmetic (walk frames are 0–3, wave frames are 4–5) is implicit and fragile.

---

### Option B — Separate WAVE_FRAMES list, new `waving` state bool + `wave_idx` counter

Keep `FRAMES` as the walk-only list. Add `WAVE_FRAMES = [WAVE_FRAME1, WAVE_FRAME2]` as a
separate constant. Add instance vars `self.waving = False` and `self.wave_idx = 0`.

In `_animate()`, add a new branch for `waving` (after the fade/pause checks, before the
walk branch). In the walk branch, when the center zone fires, set `self.waving = True` and
`self.wave_idx = 0` instead of immediately setting `paused` and `bubble_visible`.

In the waving branch:
- Draw `WAVE_FRAMES[self.wave_idx]`
- Advance `self.wave_idx`
- If `self.wave_idx >= len(WAVE_FRAMES) * WAVE_CYCLES`: exit wave, set `paused`, `bubble_visible`
- Reschedule at 120ms (same as walk)

`_draw_frame()` must be told which frame list to use. Simplest approach: pass the frame
data directly, or add a `waving` check inside `_draw_frame`.

**Pros:**
- Clean separation: walk frames and wave frames cannot bleed into each other.
- `FRAMES` walk cycle is unmodified — `(frame_idx + 1) % len(FRAMES)` is unchanged.
- State is explicit and readable: `if self.waving` is clear.
- Easy to tune: change `WAVE_CYCLES` constant to repeat the wave more/fewer times.

**Cons:**
- One extra list (`WAVE_FRAMES`) and two extra instance vars.
- `_draw_frame` needs to pick between `FRAMES[frame_idx]` and `WAVE_FRAMES[wave_idx]`;
  minor code change.

---

### Option C — Inline wave as part of pause_timer pre-phase

Instead of a new state, treat the wave as a fixed-duration leader before `bubble_visible`
is set. When center zone fires: set `paused=True`, `pause_timer=WAVE_TICKS + 60`. In the
pause branch, only set `bubble_visible=True` once `pause_timer <= 60`. During the first
`WAVE_TICKS` ticks, use `pause_timer` modulo to pick a wave frame.

**Pros:**
- No new state variables.
- Minimal diff.

**Cons:**
- Overloads `pause_timer` with two responsibilities (wave timing + pause timing).
- The modulo arithmetic (`(WAVE_TICKS - pause_timer) % len(WAVE_FRAMES)`) is confusing.
- Difficult to adjust wave independently from pause duration.
- The pause branch already runs at 80ms; wave frames should run at 120ms. Mixing them in
  one branch forces a compromise tick rate or additional complexity.

---

## Decision: Option B

**Chosen: Option B — separate WAVE_FRAMES list + explicit `waving` state.**

Rationale:

1. **Tick rate compliance**: the walking branch already fires every 120ms. Option B keeps
   wave frames in the walking branch tick loop (before the walk/bubble split) — or in a
   parallel waving branch at the same 120ms rate — trivially satisfying the acceptance
   criterion. Option C runs at 80ms (the pause branch rate) and would need extra logic.

2. **No FRAMES mutation**: Option A's index-range approach is brittle when FRAMES grows or
   when someone adds walk frames later. Option B keeps the walk cycle sealed.

3. **Explicit state**: `if self.waving` is immediately readable. The state machine has
   clear, non-overlapping branches: fade → pause → waving → walking. No implicit coupling
   between frame index and mode.

4. **Minimal risk of glitch**: since `_draw_frame` will be directed to `WAVE_FRAMES[wave_idx]`
   explicitly, there is no possibility of a walk frame leaking into the wave phase.

---

## Wave Frame Design

Wave frames reuse the identical head (rows 0–5) and legs-together pose from FRAME1. Only
the arm position changes.

**WAVE_FRAME1 — arm at shoulder level (just starting to raise)**

The left arm (viewer's left, cols 2–3 in rows 7–8) is extended upward to row 5–6, making
the arm appear at shoulder height. The torso rows 7–8 drop the left-side arm pixels.

```
Row 5: " BN BSSSSSB   "   arm (B at col 1, N at col 2) beside face bottom
Row 6: " BN BCCCCB    "   arm beside body top
Row 7: "   BCCCCCNB   "   body — left arm removed, right arm (NB at cols 9-10) stays
Row 8: "   BCCCCCNB   "   body (same)
```

**WAVE_FRAME2 — arm raised above head (full wave height)**

```
Row 0: " BS BBBBBB    "   hand (S=skin) at col 2, head at cols 4-9
Row 1: " BN BSSSSSB   "   arm sleeve beside face top
Row 2: "   BSWWWSB    "   face continues (arm ends above)
Row 7: "   BCCCCCNB   "   body — left arm removed (raised), right arm stays
Row 8: "   BCCCCCNB   "   body (same)
```

**WAVE_CYCLES = 2** — the 2-frame sequence plays twice (4 frames total = ~480ms wave),
giving a brief but perceptible wave before the bubble appears.

---

## State Machine After Change

```
fade_out  →  fade and destroy
paused    →  show bubble, count down, then resume walk
waving    →  cycle wave frames at 120ms; on completion → paused + bubble_visible
walking   →  move, detect center zone → waving
```

Priority order in `_animate()`:
1. fade_out check (early return)
2. paused check (early return, 80ms reschedule)
3. **waving check (early return, 120ms reschedule)**   ← NEW
4. walking logic (move, center-zone check, frame cycle, 120ms reschedule)

---

## Changes Summary

| Location           | Change                                                              |
|--------------------|---------------------------------------------------------------------|
| Module-level       | Add `WAVE_FRAME1`, `WAVE_FRAME2`, `WAVE_FRAMES`, `WAVE_CYCLES`     |
| `__init__`         | Add `self.waving = False`, `self.wave_idx = 0`                     |
| `_draw_frame()`    | Use `WAVE_FRAMES[wave_idx]` when `waving`, else `FRAMES[frame_idx]`|
| `_animate()`       | Add waving branch; change center-zone trigger to start wave        |

Total estimated diff: ~30–40 lines added or modified in one file.
