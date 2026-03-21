# Plan: T-001-01 wave-animation

## Implementation Steps

### Step 1 — Add WAVE_FRAME1 pixel art constant

**What**: Add `WAVE_FRAME1` list after the existing `FRAME3` definition, before `FRAMES`.

**Verification**: Count each row to confirm 14 characters. Visual inspection of arm at
cols 1–2 beside head (cols 4–9), left arm absent from rows 7–8.

**Commit**: "add WAVE_FRAME1 pixel art (arm at shoulder level)"

---

### Step 2 — Add WAVE_FRAME2 pixel art constant

**What**: Add `WAVE_FRAME2` list after `WAVE_FRAME1`.

**Verification**: Count each row. Confirm hand (S) at row 0 col 2, sleeve (N) at row 1
col 2, arm absent from rows 7–8, head rows unchanged.

**Commit**: "add WAVE_FRAME2 pixel art (arm raised above head)"

---

### Step 3 — Add WAVE_FRAMES list and WAVE_CYCLES constant

**What**: Add immediately after `FRAMES = [FRAME1, FRAME2, FRAME3, FRAME2]`:
```python
WAVE_FRAMES = [WAVE_FRAME1, WAVE_FRAME2]
WAVE_CYCLES = 2
```

**Verification**: `len(WAVE_FRAMES) == 2`, `WAVE_CYCLES == 2`, total wave ticks = 4.

**Commit**: bundle with step 2 (or separate — both are data-only)

---

### Step 4 — Add waving state variables to __init__

**What**: In `__init__`, after the existing state var block (after `self.alpha = 0.97`),
add:
```python
self.waving   = False
self.wave_idx = 0
```

**Verification**: The `__init__` state block initializes these before `_draw_frame()` and
`_animate()` are called, so no risk of uninitialized access.

**Commit**: "add waving state vars to __init__"

---

### Step 5 — Modify _draw_frame() to select wave frame when waving

**What**: Replace the final `self._draw_pixel_frame(FRAMES[self.frame_idx], ...)` call
with a conditional:
```python
if self.waving:
    frame = WAVE_FRAMES[self.wave_idx % len(WAVE_FRAMES)]
else:
    frame = FRAMES[self.frame_idx]
self._draw_pixel_frame(frame, sprite_x, sprite_y)
```

**Verification**:
- When `waving=False`, behavior identical to before.
- When `waving=True`, uses `wave_idx % len(WAVE_FRAMES)` so even if `wave_idx` has
  advanced past the list length, index is safe.

**Commit**: "draw wave frames when waving=True in _draw_frame"

---

### Step 6 — Add waving branch to _animate()

**What**: Insert after the `paused` branch (and its `return`), before the walking branch:

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

**Logic trace**:
- Entry: `wave_idx=0`, `waving=True`
- Ticks 1–3: increment wave_idx, draw frame, return at 120ms
- Tick 4: `wave_idx=4`, `4 >= 2*2=4` → True, exit wave, set paused+bubble, draw, return
- Next tick: hits `paused` branch, runs pause countdown normally

**Boundary check**: wave completes after exactly `WAVE_CYCLES * len(WAVE_FRAMES) = 4` ticks
= 480ms. The `_draw_frame()` call after the exit block draws the first pause frame with
`waving=False`, so it uses `FRAMES[self.frame_idx]` — this will be the last walk frame
index (whatever it was when center zone fired). That's a normal walk frame, which is
acceptable for the first pause tick; Claude will be stationary with the bubble visible.

**Commit**: "add waving branch to _animate with completion → pause+bubble"

---

### Step 7 — Update center-zone trigger to start wave instead of pause

**What**: In the walking branch, modify the center-zone `if` block:

Current:
```python
if not self.bubble_visible and self.x <= center_zone + 10 and self.x >= center_zone - 10:
    self.paused = True
    self.bubble_visible = True
    self.pause_timer = 60
    self.direction = 0
```

New:
```python
if not self.bubble_visible and not self.waving \
        and self.x <= center_zone + 10 and self.x >= center_zone - 10:
    self.waving = True
    self.wave_idx = 0
    self.direction = 0
```

**Guard explanation**: `not self.bubble_visible` was the original one-shot guard. We add
`not self.waving` to prevent re-triggering if the window is still in the center zone on
the same tick. (Since we set `direction=0`, the window won't move, but the condition could
be re-evaluated if `_animate` is somehow called without the waving branch returning early.)

**Commit**: "trigger wave on center-zone arrival instead of immediate pause+bubble"

---

## Testing Strategy

This app has no test suite; it is a single-purpose GUI script. Verification is manual.

### Manual Verification Checklist

1. **Run the app**: `python3 bedtime_claude.py`
2. **Walk phase**: Claude should walk left across the screen, cycling through walk frames.
   No wave frames should appear during the walk.
3. **Wave trigger**: When Claude reaches screen center, he should stop and begin the wave
   animation. Arm should visibly alternate between shoulder-level and raised-above-head
   poses for approximately 4 frames (~480ms).
4. **Transition to bubble**: After the wave sequence, the speech bubble should appear.
   No flash or blank frame should be visible at the walk→wave or wave→pause transitions.
5. **Pause duration**: Bubble should remain visible for ~5 seconds (60 ticks × 80ms).
6. **Resume and exit**: Claude resumes walking left, exits the left edge, fades out, and
   the window closes.
7. **No regression**: Behavior identical to before except for the wave sequence at center.

### Acceptance Criteria Mapping

| Criterion                                               | Verified by                      |
|---------------------------------------------------------|----------------------------------|
| At least 2 wave frames added to FRAMES (or equiv.)     | Code review of WAVE_FRAMES list  |
| Wave plays once before bubble_visible=True              | Observe step 3 above             |
| Wave frames cycle at 120ms                              | root.after(120) in waving branch |
| After wave: pause + show bubble as before               | Observe steps 4–6 above          |
| No visual glitches at transitions                       | Observe steps 2, 3, 4 above      |

### Static Checks

- Confirm all wave frame rows are 14 chars (count manually or `assert all(len(r)==14 ...)`).
- Confirm `WAVE_FRAMES` and `WAVE_CYCLES` are defined before `BedtimeClaude` class.
- Confirm `__init__` initializes both `waving` and `wave_idx` before first `_animate` call.

---

## Risk Register

| Risk                                              | Likelihood | Mitigation                             |
|---------------------------------------------------|------------|----------------------------------------|
| Wave frame row length ≠ 14, misaligns pixels      | Low        | Count each row before committing       |
| wave_idx modulo off-by-one causes wrong frame     | Low        | Trace logic in step 6 above            |
| Center zone re-triggers wave a second time        | Low        | `not self.waving` guard in step 7      |
| walk frame index left at unexpected value         | Low        | frame_idx not modified during wave     |
| Pause branch fires before wave completes          | None       | `paused` not set until wave exits      |
