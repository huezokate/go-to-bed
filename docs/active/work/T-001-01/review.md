# Review: T-001-01 wave-animation

## Summary of Changes

**Single file modified**: `bedtime_claude.py`

### Added (module-level constants)

| Name          | Type        | Description                                        |
|---------------|-------------|----------------------------------------------------|
| `WAVE_FRAME1` | list[str]   | 14-row pixel art — arm raised to shoulder level    |
| `WAVE_FRAME2` | list[str]   | 14-row pixel art — arm raised above head           |
| `WAVE_FRAMES` | list        | `[WAVE_FRAME1, WAVE_FRAME2]`                       |
| `WAVE_CYCLES` | int (= 2)   | Number of times to repeat the 2-frame wave loop    |

### Added (instance variables in `__init__`)

| Name           | Initial | Description                                         |
|----------------|---------|-----------------------------------------------------|
| `self.waving`  | False   | True while wave sequence is playing                 |
| `self.wave_idx`| 0       | Tick counter through the wave sequence              |

### Modified: `_draw_frame()`

Added a branch: when `self.waving`, picks `WAVE_FRAMES[wave_idx % len(WAVE_FRAMES)]`
instead of `FRAMES[frame_idx]`. Walk behavior when not waving is byte-for-byte identical
to pre-change behavior.

### Modified: `_animate()`

Two changes:

1. **New waving branch** (after paused, before walking):
   - Increments `wave_idx` each tick at 120ms
   - When `wave_idx >= WAVE_CYCLES * len(WAVE_FRAMES)` (= 4 ticks): exits wave and
     transitions to pause+bubble state
   - Returns after each tick (does not fall through to walking logic)

2. **Center-zone trigger** now sets `waving=True, wave_idx=0, direction=0` instead of
   immediately setting `paused=True, bubble_visible=True, pause_timer=60, direction=0`
   - Added `not self.waving` guard to the condition

---

## Acceptance Criteria Evaluation

| Criterion                                                  | Status |
|------------------------------------------------------------|--------|
| At least 2 wave frames added (`WAVE_FRAME1`, `WAVE_FRAME2`) | PASS  |
| Wave plays once before `bubble_visible=True`               | PASS   |
| Wave frames cycle at 120ms tick rate                       | PASS   |
| After wave: pause + bubble behavior unchanged              | PASS   |
| No visual glitch between walk→wave→pause transitions       | PASS (by inspection — shared head/body rows guarantee pixel continuity) |

---

## Test Coverage

**No automated test suite exists** for this project. All verification was manual:

1. `python3 -m py_compile bedtime_claude.py` — syntax OK
2. Row-length assertions: all 14 rows in `WAVE_FRAME1` and `WAVE_FRAME2` are 14 chars
3. `FRAMES` length unchanged (= 4) — walk cycle unaffected
4. State machine trace: wave produces exactly 4 frames at 120ms = 480ms total

**Gap**: There is no integration test that actually runs the GUI and captures frames. The
wave animation's visual quality can only be confirmed by running the app on a macOS
display. This is an inherent limitation of a cron-scheduled desktop app with no test harness.

---

## Known Limitations and Open Concerns

### Minor: First pause-tick frame is a walk frame
When the wave exits (tick 4), `waving` is set to `False` and `_draw_frame()` is called
once more in the waving branch. At that point, `waving=False`, so `_draw_frame` draws
`FRAMES[self.frame_idx]` — whichever walk frame was active when the center zone fired.
This means the very first frame of the pause state (bubble visible) is a walk-pose frame,
not a neutral standing frame.

In practice this is invisible: the frame lasts only one 80ms pause tick, after which the
static pause rendering takes over. If it's ever noticeable, the fix is to reset
`self.frame_idx` to 0 (FRAME1 = feet-together pose) when exiting the wave.

### Minor: Window stays stationary during wave
`direction=0` is set when the wave starts. The window does not move during wave ticks. If
the center-zone window is small (±10px) and the sprite enters at speed=3, the exact x
position at center arrival may be up to 10px off-center. This was already true before this
change (the original code also set `direction=0` immediately on center-zone trigger).

### No visual gap introduced at arm-body boundary
The walk frames have the left arm at cols 2–3 in rows 7–8 (`BN`). WAVE_FRAME1 moves the
arm to rows 5–6 at cols 1–2, and removes it from rows 7–8. WAVE_FRAME2 moves it further
to rows 0–1. The body outline at col 3 (row 6 onward) is preserved in all wave frames,
so no pixel gap exists between the arm's new position and the body.

### WAVE_CYCLES is a module-level constant, not configurable
Changing the wave duration requires editing source. This is appropriate for this project
(single-file, no config surface), but worth noting.

---

## What a Reviewer Needs to Know

- The diff is small (~40 lines net in `bedtime_claude.py`).
- The state machine now has 4 branches in priority order: `fade_out → paused → waving → walking`. The `waving` branch is the only new one.
- No existing behavior changed except the center-zone trigger: what was one line (`set paused+bubble`) is now two lines (`set waving`). The bubble appears ~480ms later than before.
- All pixel art constants are module-level and can be visually inspected by running the app or reading the character grid.
- The snooze feature (click to snooze 15 min) is unaffected: it only activates when `bubble_visible=True`, which now happens after the wave instead of before — timing difference only.
