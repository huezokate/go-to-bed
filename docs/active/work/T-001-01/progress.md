# Progress: T-001-01 wave-animation

## Status: Complete

## Steps Completed

### Step 1–3: Wave frame constants added
- `WAVE_FRAME1`: arm at shoulder level (rows 5–8 modified; feet together)
- `WAVE_FRAME2`: arm raised above head (rows 0–1 and 7–8 modified; feet together)
- `WAVE_FRAMES = [WAVE_FRAME1, WAVE_FRAME2]`
- `WAVE_CYCLES = 2`
- All 14 rows in each wave frame verified at 14 characters.

### Step 4: State variables
- `self.waving = False` added to `__init__`
- `self.wave_idx = 0` added to `__init__`

### Step 5: _draw_frame() updated
- When `self.waving`, draws `WAVE_FRAMES[self.wave_idx % len(WAVE_FRAMES)]`
- When not waving, draws `FRAMES[self.frame_idx]` (unchanged behavior)

### Step 6: Waving branch added to _animate()
- Placed after paused branch, before walking branch
- Advances `wave_idx` each tick (120ms)
- On completion (`wave_idx >= WAVE_CYCLES * len(WAVE_FRAMES) = 4`): exits wave,
  sets `paused=True`, `bubble_visible=True`, `pause_timer=60`

### Step 7: Center-zone trigger updated
- Now sets `waving=True, wave_idx=0, direction=0` instead of immediately entering pause
- Guard: `not self.waving` prevents double-trigger

## Deviations from Plan
None. Implementation followed the plan exactly.

## Verification
- `python3 -m py_compile bedtime_claude.py` → OK
- Row length assertions for both wave frames → passed
- `WAVE_CYCLES * len(WAVE_FRAMES) = 4` ticks at 120ms = 480ms wave duration
- `FRAMES` list unchanged (len=4)

## What Remains
Nothing. All 7 plan steps complete.
