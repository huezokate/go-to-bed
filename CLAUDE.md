# CLAUDE.md

## Project

go-to-bed (Python desktop app) — A pixel art Claude that walks across your screen at bedtime and tells you to sleep.

### Run

```bash
python3 bedtime_claude.py
```

Schedule with cron (runs at 11pm nightly):
```
0 23 * * * /usr/bin/python3 /Users/KaterinaHuezo/Documents/projects/go-to-bed/bedtime_claude.py
```

### Source Layout

```
bedtime_claude.py    # Single-file app — pixel frames, animation loop, speech bubble
```

### Stack

- Python 3, `tkinter` (stdlib only — no dependencies)
- Transparent always-on-top window, pixel art rendered via canvas rectangles

### Directory Conventions

```
docs/active/tickets/    # Ticket files (markdown with YAML frontmatter)
docs/active/stories/    # Story files (same frontmatter pattern)
docs/active/work/       # Work artifacts, one subdirectory per ticket ID
```

---

The RDSPI workflow definition is in docs/knowledge/rdspi-workflow.md and is injected into agent context by lisa automatically.
