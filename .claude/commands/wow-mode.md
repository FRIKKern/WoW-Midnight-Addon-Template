Set WoW addon development mode: $ARGUMENTS

You manage the addon development mode for this project. The mode system fundamentally changes how all WoW addon agents write, review, and scaffold code.

## Available Modes

| Mode | Aliases | Philosophy |
|------|---------|-----------|
| `blizzard-faithful` | faithful, blizzard, safe, conservative | Official APIs only. No Blizzard frame hooks. Patch-proof. |
| `boundary-pusher` | boundary, pusher, aggressive, advanced, elvui | Undocumented APIs, metatable hooks, creative workarounds. ElvUI-class. |
| `enhancement-artist` | enhance, artist, better, skin, hook | hooksecurefunc, Mixin, frame skinning. "Enhance don't replace." DEFAULT. |
| `performance-zealot` | performance, perf, zealot, fast, lean | Minimal memory, throttled updates, object pooling, event-driven. |

## Process

1. Parse `$ARGUMENTS` for a mode name or alias
2. If no argument or "status": Read `.claude/modes/active-mode.md` and display current mode with full description
3. If valid mode name or alias:
   - Resolve alias to canonical name (e.g., "safe" → "blizzard-faithful")
   - Write ONLY the canonical mode name to `.claude/modes/active-mode.md` (overwrite entire file)
   - Read `.claude/modes/{canonical-name}.md` and display a summary of the mode's philosophy and key rules
4. If invalid argument: Show available modes and aliases

## Output Format

When setting a mode, display:
- Mode name and icon
- 3-5 key philosophy points
- "All /wow-create, /wow-review, and agent interactions will now follow {mode} rules."

When showing status, also show brief descriptions of all 4 modes so the user knows their options.
