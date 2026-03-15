Set up and rename a cloned WoW Midnight addon template: $ARGUMENTS

Template repo: https://github.com/FRIKKern/WoW-Midnight-Addon-Template
Clone it, cd into template/, then run `/wow-setup YourAddonName`

## Process

### Step 1: Determine the Addon Name

- If `$ARGUMENTS` is provided, use it as the addon name (e.g., `RareTracker`)
- If empty, use the current directory name as the addon name
- The name MUST be PascalCase with no spaces or special characters (letters, numbers, underscores only)
- If the name contains spaces, convert to PascalCase (e.g., "rare tracker" → "RareTracker")

### Step 2: Verify Template Structure

Check that the current directory contains the template files:
- Look for `MyAddon.toc` — this confirms it's an unmodified template clone
- If `MyAddon.toc` doesn't exist, check if a renamed `.toc` file already exists — the template may have already been set up. If so, tell the user and stop.
- Required files: `Init.lua`, `Core.lua`, `Config.lua`, `CLAUDE.md`, `.pkgmeta`, `.luacheckrc`

### Step 3: Compute Replacement Strings

From the addon name (e.g., `RareTracker`), derive:

| Pattern | Example | Used For |
|---------|---------|----------|
| `AddonName` | `RareTracker` | TOC filename, SavedVariables, function names, namespace |
| `ADDONNAME` | `RARETRACKER` | SlashCmdList key, SLASH_ globals |
| `addonname` | `raretracker` | Slash command text (e.g., `/raretracker`) |
| `AddonNameDB` | `RareTrackerDB` | SavedVariables global |
| `AddonNameCharDB` | `RareTrackerCharDB` | Per-character SavedVariables |

Also derive a short slash alias (first letters or abbreviation, e.g., `rt` for RareTracker). Use the first letter of each PascalCase word, lowercased.

### Step 4: Rename Files

1. Rename `MyAddon.toc` → `{AddonName}.toc`

### Step 5: Replace in All Files

Perform these replacements across ALL files in the directory (`.toc`, `.lua`, `.md`, `.yml`, `.luacheckrc`, `.pkgmeta`). Order matters — do longer patterns first to avoid partial matches:

1. `MyAddonCharDB` → `{AddonName}CharDB`
2. `MyAddonDB` → `{AddonName}DB`
3. `MyAddonInfoFrame` → `{AddonName}InfoFrame`
4. `MyAddon_OnCompartmentClick` → `{AddonName}_OnCompartmentClick`
5. `MyAddon_OnCompartmentEnter` → `{AddonName}_OnCompartmentEnter`
6. `MyAddon_OnCompartmentLeave` → `{AddonName}_OnCompartmentLeave`
7. `MyAddon` → `{AddonName}` (catches TOC title, .pkgmeta package-as, CLAUDE.md heading, README, etc.)
8. `SLASH_MYADDON` → `SLASH_{ADDONNAME}`
9. `SlashCmdList["MYADDON"]` → `SlashCmdList["{ADDONNAME}"]`
10. `"/myaddon"` → `"/{addonname}"`
11. `"/ma"` → `"/{alias}"` (the short alias from step 3)
12. `/myaddon` → `/{addonname}` (in comments, help text, README)

Use the Edit tool with `replace_all: true` for each replacement. Read each file before editing.

### Step 6: Update Author (Optional)

If the user provided an author name in $ARGUMENTS (e.g., `/wow-setup RareTracker by Pelle`), update `## Author: YourName` in the TOC.

### Step 7: Verify

After all replacements:
1. Confirm the `.toc` file has `## Interface: 120001`
2. Confirm `## SavedVariables:` references the correct DB name
3. Confirm `## AddonCompartmentFunc:` references the correct function name
4. Confirm no remaining `MyAddon` references exist (grep for it)
5. Confirm no remaining `MYADDON` references exist
6. Confirm no remaining `myaddon` references exist (except in git history)

### Step 8: Summary

Print a summary:

```
Template set up for: {AddonName}

Renamed files:
  {AddonName}.toc
  Init.lua — namespace, /addonname + /alias commands
  Core.lua — {AddonName}InfoFrame
  Config.lua — Settings panel
  CLAUDE.md — AI instructions
  .pkgmeta — package-as: {AddonName}
  .luacheckrc — globals updated

Slash commands: /{addonname}, /{alias}
SavedVariables: {AddonName}DB

Next steps:
  1. Edit Init.lua — update ns.defaults with your settings
  2. Edit Core.lua — replace the demo info frame with your features
  3. Edit Config.lua — add your settings controls
  4. Test: symlink to WoW AddOns folder, /reload
  5. Release: git tag v1.0.0 && git push --tags
```

## Rules

- This is a RENAME operation only — do not modify logic, add features, or restructure code
- Be idempotent — if MyAddon.toc doesn't exist, check for an existing renamed TOC and stop
- Use Edit tool with `replace_all: true` for replacements, NOT sed/awk
- Read files before editing them
- Do not touch files outside the current directory
