# MyAddon

## Overview

A World of Warcraft addon for **Midnight (Patch 12.0+)**. Interface version: `120001`.

This addon uses the namespace pattern (`local addonName, ns = ...`), the modern Settings API, and Addon Compartment integration. It does NOT use Ace3 — all patterns are pure Blizzard API.

## Critical: WoW 12.0 Midnight API Rules

### REMOVED — Do NOT Generate Code Using These:

| Removed Function/Feature | Replacement |
|--------------------------|-------------|
| `COMBAT_LOG_EVENT_UNFILTERED` | Gone entirely. Use unit events (`UNIT_HEALTH`, `UNIT_AURA`, `UNIT_SPELLCAST_*`) |
| `CombatLogGetCurrentEventInfo()` | Gone. No replacement for parsing combat log |
| `UnitHealth()` for math/conditionals | Returns secret values in combat. Use `UnitHealthPercent()` + `StatusBar:SetValue()` |
| `UnitHealthMax()` for math | Returns secret values in combat. Use `UnitHealthPercent()` |
| `GetSpellCooldown()` | Use `C_Spell.GetSpellCooldown()` or `C_Spell.GetSpellCooldownDuration()` |
| `GetSpellInfo()` | Use `C_Spell.GetSpellInfo()` (returns a table, not multiple values) |
| `GetSpellTexture()` | Use `C_Spell.GetSpellTexture()` |
| `IsSpellKnown()` | Use `C_Spell.IsSpellDataCached()` and `C_SpellBook.IsSpellBookItemKnown()` |
| `ActionHasRange()` | Use `C_ActionBar.IsActionInRange()` |
| `IsUsableAction()` | Use `C_ActionBar.IsUsableAction()` |
| `GetActionCooldown()` | Use `C_ActionBar.GetActionCooldown()` |
| `GetActionTexture()` | Use `C_ActionBar.GetActionTexture()` |
| `InterfaceOptions_AddCategory()` | Use `Settings.RegisterAddOnCategory()` |
| `InterfaceOptionsFrame_OpenToCategory()` | Use `Settings.OpenToCategory()` |
| `EasyMenu()` / `UIDropDownMenu_*` | Use `MenuUtil.CreateContextMenu()` |
| `GetMouseFocus()` | Use `GetMouseFoci()` (returns table) |
| `SendAddonMessage()` during encounters | Blocked in M+, PvP, and boss encounters |
| `CombatLogAddFilter()` / `CombatLogResetFilter()` | Removed entirely |
| `SetMinResize()` / `SetMaxResize()` | Use `Frame:SetResizeBounds()` |

### Secret Values (CRITICAL for Combat-Related Code):

In Midnight, during M+/PvP/encounters/combat, many APIs return **secret values** — opaque containers that CANNOT be:
- Compared (`if health < 100` → ERROR)
- Used in arithmetic (`health / maxHealth` → ERROR)
- Used as table keys
- Tested for truthiness

Secret values CAN be:
- Stored in variables
- Passed to widget setters (`StatusBar:SetValue()`, `FontString:SetText()`)
- Concatenated with strings
- Checked with `issecretvalue(val)` → true/false
- Displayed via `ColorCurve:Evaluate()` and `Duration:SetCooldownFromDurationObject()`

**Always guard combat code:**
```lua
local value = UnitHealth("target")
if issecretvalue(value) then
    -- Use widget-safe display path
    healthBar:SetValue(UnitHealthPercent("target"))
else
    -- Can do math/conditionals freely
    local pct = value / UnitHealthMax("target")
end
```

**What is NOT secret** (always readable):
- Player secondary resources (Combo Points, Holy Power, Soul Shards, Chi, Runes, etc.)
- `UnitIsUnit()` comparisons with target/focus/mouseover/softenemy
- Profession and Skyriding spells

### Key 12.0 APIs to Use:

| API | Purpose |
|-----|---------|
| `issecretvalue(val)` | Check if a value is a secret |
| `C_Secrets.HasSecretRestrictions()` | Check if restrictions are active |
| `UnitHealthPercent(unit)` | Health as 0-1 (may be secret, but StatusBar accepts it) |
| `UnitPowerPercent(unit, type)` | Power as 0-1 (may be secret) |
| `C_DurationUtil.CreateDuration()` | Create a duration object for cooldown display |
| `Cooldown:SetCooldownFromDurationObject(dur)` | Display cooldown from duration object |
| `C_Spell.GetSpellCooldownDuration(spellID)` | Get cooldown as duration object |
| `C_DamageMeter.*` | Access built-in damage meter data |
| `C_EncounterTimeline.*` | Raid encounter timeline/mechanics |
| `Frame:RegisterEventCallback(event, cb)` | Callback-based event registration (no frame needed) |
| `StatusBar:SetTimerDuration(duration)` | Self-updating timer bar |
| `Region:SetAlphaFromBoolean(visible)` | Secret-safe visibility |

## Architecture

### File Load Order (defined in .toc):
1. `Libs/` — External libraries (LibStub, CallbackHandler)
2. `Init.lua` — Namespace setup, event dispatcher, ADDON_LOADED, PLAYER_LOGIN, slash commands
3. `Core.lua` — Feature logic, frame creation, hooks, combat queue
4. `Config.lua` — Settings panel registration

### Namespace Pattern:
```lua
local addonName, ns = ...
-- ns is shared between ALL .lua files in the TOC
-- Attach everything to ns.* — never create globals except SavedVariables and SLASH_*
```

### Event Dispatch Pattern:
```lua
local eventHandlers = {}
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = eventHandlers[event]
    if handler then handler(self, event, ...) end
end)
```

### Combat Lockdown Pattern:
```lua
if InCombatLockdown() then
    combatQueue[#combatQueue + 1] = func  -- defer
else
    func()  -- run now
end
-- Flush queue on PLAYER_REGEN_ENABLED
```

## Coding Conventions

### Always:
- Use `local` for everything (performance + no global pollution)
- Check `frame:IsForbidden()` before modifying hooked frames
- Check `InCombatLockdown()` before modifying secure/protected frames
- Use `hooksecurefunc()` to modify Blizzard frames (post-hook, never pre-hook)
- Use `C_Timer.After()` or `C_Timer.NewTicker()` instead of OnUpdate polling
- Use the `ns.*` namespace pattern for cross-file communication
- Localize hot-path globals as upvalues (e.g., `local format = format`)

### Never:
- Generate `COMBAT_LOG_EVENT_UNFILTERED` handlers (event is gone in 12.0)
- Call deprecated functions listed in the table above
- Create global variables (except SavedVariables and SLASH_* commands)
- Call `:Hide()` on secure Blizzard frames (causes taint — reparent to hidden frame instead)
- Do arithmetic or comparisons on values that might be secret
- Use `SendAddonMessage()` during encounters
- Use `UIDropDownMenu_*` functions (removed — use `MenuUtil`)

## Settings API (11.0.2+ Signatures):
```lua
-- Register: Settings.RegisterAddOnSetting(categoryTbl, variable, variableKey, variableTbl, variableType, name, defaultValue)
-- Controls: Settings.CreateCheckbox(category, setting, tooltip)
-- Controls: Settings.CreateSlider(category, setting, options, tooltip)
-- Controls: Settings.CreateDropdown(category, setting, getOptions, tooltip)
-- Finalize: Settings.RegisterAddOnCategory(category)
```

## Build & Release

### Lint:
```
luacheck .
```

### Release:
```
git tag v1.0.0 -m "Initial release"
git push --tags
```
GitHub Actions runs BigWigsMods/packager to build and upload.

### Test in-game:
```
/reload                    — Reload UI
/dump SomeVariable         — Inspect values
/console scriptErrors 1    — Show Lua errors inline
```
Install BugGrabber + BugSack for error capture.

## Key Reference URLs

- https://warcraft.wiki.gg/wiki/World_of_Warcraft_API
- https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes
- https://warcraft.wiki.gg/wiki/Patch_12.0.1/API_changes
- https://warcraft.wiki.gg/wiki/Settings_API
- https://warcraft.wiki.gg/wiki/Addon_compartment
- https://warcraft.wiki.gg/wiki/TOC_format
- https://warcraft.wiki.gg/wiki/Events
- https://warcraft.wiki.gg/wiki/Handling_events
- https://warcraft.wiki.gg/wiki/SecureActionButtonTemplate
- https://github.com/BigWigsMods/packager/wiki
