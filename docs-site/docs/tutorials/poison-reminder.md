---
title: "Build-Along: PoisonPal — Rogue Poison Reminder"
description: Build a complete rogue poison reminder addon from scratch using modern WoW 12.0+ APIs. Covers event handling, aura-based poison detection, Settings API, Addon Compartment, and chat alerts.
---

# Build-Along: PoisonPal — Rogue Poison Reminder

**Difficulty:** :material-star: :material-star-outline: :material-star-outline: Beginner |
**Mode:** :material-shield-check: Blizzard Faithful |
**Reading time:** ~20 minutes |
**Interface:** 120001 (Patch 12.0.1)

---

## Step 1: What We're Building

PoisonPal is a small, focused addon that solves a real problem every Rogue knows: forgetting to apply poisons before a dungeon or raid. It watches your poison buffs and warns you when a lethal or non-lethal poison is missing.

### Features

- :material-alert-circle: **Missing poison alerts** — warns in chat when you enter the world without poisons applied
- :material-cog: **Settings panel** — enable/disable, sound toggle, login check toggle
- :material-dots-vertical: **Addon Compartment button** — left-click to check status, right-click for settings
- :material-sword: **Rogue-only** — silently disables on non-Rogue characters

### Why This Is a Great First Addon

PoisonPal uses the core building blocks you'll need for almost any addon:

- **Event-driven logic** — reacting to `UNIT_AURA` and `PLAYER_ENTERING_WORLD` instead of polling every frame
- **`C_UnitAuras.GetPlayerAuraBySpellID()`** — the modern aura API for checking player buffs by spell ID
- **Settings API** — the modern Blizzard settings panel (not the deprecated `InterfaceOptions` system)
- **Addon Compartment** — the 10.1+ minimap dropdown that replaces LibDataBroker for simple addons
- **Class filtering** — checking `UnitClass()` to only run on relevant characters

By the end, you'll have a working addon you can install and use immediately, built entirely with official Blizzard APIs in `blizzard-faithful` mode.

!!! danger "Poisons Are Buffs, Not Weapon Enchants"
    In modern retail WoW (Dragonflight onward), Rogue poisons are **player buffs** — not temporary weapon enchants. You activate a poison ability and it persists as a buff until cancelled or replaced. This means:

    - **`GetWeaponEnchantInfo()` does NOT detect poisons.** It only detects temporary weapon enchants like sharpening stones or weightstones.
    - **`UNIT_INVENTORY_CHANGED` does NOT fire** when poisons are applied or removed.
    - **Poisons do not expire.** There is no timer to track — just presence or absence.

    The correct API is `C_UnitAuras.GetPlayerAuraBySpellID(spellID)`, and the correct event is `UNIT_AURA`. AI coding assistants trained on older WoW code will frequently generate `GetWeaponEnchantInfo()` for poison detection — this is wrong for any WoW version from Dragonflight onward.

!!! tip "Plugin Shortcut"
    Want to skip the manual steps? You can generate a similar addon instantly:
    ```
    /wow-create "faithful rogue poison reminder with settings panel"
    ```
    But building it by hand teaches you the patterns you'll use in every addon.

---

## Step 2: Prerequisites

Before you start, make sure you have:

- **Claude Code** with the better-addons plugin installed (see [Plugin Overview](../plugin/overview.md))
- **A text editor** — VS Code with the [WoW API extension](https://marketplace.visualstudio.com/items?itemName=Ketho.wow-api) is recommended
- **WoW 12.0+** game client for testing (you need a Rogue character)

### Set Your Development Mode

Open Claude Code and set `blizzard-faithful` mode. This ensures all generated code uses only official, documented APIs:

```
/wow-mode faithful
```

!!! info "Why Blizzard Faithful?"
    For a beginner tutorial, `blizzard-faithful` is the safest choice. The code we write will use only documented APIs, avoid hooking Blizzard frames, and survive any future patch without changes. The other modes (`enhancement-artist`, `boundary-pusher`, `performance-zealot`) are for when you want to push the boundaries — not where you want to start.

---

## Step 3: Project Setup

Create a new folder for the addon. The folder name must match the `.toc` filename exactly — WoW uses the folder name to find the manifest.

```bash
mkdir PoisonPal
cd PoisonPal
```

Our addon will have three files:

```
PoisonPal/
  PoisonPal.toc    -- Manifest: metadata, file load order
  Core.lua         -- Initialization, events, poison checking, alerts
  Config.lua       -- Settings panel
```

!!! info "Why not Init.lua + Core.lua?"
    The starter template splits initialization and logic into separate files. For a small addon like PoisonPal, that's unnecessary overhead. We'll combine everything into `Core.lua` and keep `Config.lua` separate since the Settings API code is self-contained. When your addon grows, split files by responsibility.

### Create the TOC File

The `.toc` file is your addon's manifest. WoW reads it to decide what to load and in what order.

```toc
## Interface: 120001
## Title: PoisonPal
## Notes: Rogue poison reminder — warns when poisons are missing.
## Author: YourName
## Version: @project-version@
## SavedVariables: PoisonPalDB
## IconTexture: Interface\Icons\Ability_Rogue_DualWeild
## AddonCompartmentFunc: PoisonPal_OnCompartmentClick
## AddonCompartmentFuncOnEnter: PoisonPal_OnCompartmentEnter
## AddonCompartmentFuncOnLeave: PoisonPal_OnCompartmentLeave

Core.lua
Config.lua
```

Let's break down every field:

| Field | Purpose |
|-------|---------|
| `Interface: 120001` | Targets Patch 12.0.1. Addons below `120000` require "Load out of date addons" to be enabled. |
| `SavedVariables: PoisonPalDB` | The global Lua variable name WoW persists between sessions. Available after `ADDON_LOADED`. |
| `IconTexture` | Shown in the addon list and the Addon Compartment dropdown. Use any standard WoW icon path. |
| `AddonCompartmentFunc` | The global function WoW calls when the player clicks your entry in the minimap dropdown. |
| `@project-version@` | A token that BigWigsMods/packager replaces with the git tag at build time. Shows "dev" locally. |

!!! warning "Common Mistake: Folder Name Mismatch"
    If your folder is named `PoisonPal` but the file is `poisonpal.toc`, the addon won't load. The `.toc` filename must match the folder name exactly, including capitalization on case-sensitive systems.

---

## Step 4: Core Logic — Event Handling

Create `Core.lua`. This file handles everything: namespace setup, event dispatch, poison checking, slash commands, alerts, and the Addon Compartment.

### Namespace and Defaults

Every file in a WoW addon receives two arguments: the addon name (string) and a shared namespace table. We use the namespace to share data between `Core.lua` and `Config.lua`.

```lua
local addonName, ns = ...

ns.addonName = addonName

-- Default settings (merged into SavedVariables on first load)
ns.defaults = {
    enabled = true,          -- Master toggle
    playSound = true,        -- Play a sound with warnings
    showOnLogin = true,      -- Check poisons on login
}
```

### Poison Spell ID Tables

Modern Rogue poisons are player buffs. We need to know the spell IDs for each poison so we can check for them using the aura API. Rogues have two poison categories — **lethal** and **non-lethal** — and should always have one of each active.

```lua
-- Lethal poisons (spell IDs for the player buff)
local LETHAL_POISONS = {
    [2823]   = "Deadly Poison",
    [315584] = "Instant Poison",
    [8679]   = "Wound Poison",
    [381637] = "Amplifying Poison",
}

-- Non-lethal poisons (spell IDs for the player buff)
local NON_LETHAL_POISONS = {
    [3408]   = "Crippling Poison",
    [5761]   = "Numbing Poison",
    [381664] = "Atrophic Poison",
}
```

!!! info "Where Do These Spell IDs Come From?"
    You can find spell IDs on [Wowhead](https://www.wowhead.com/) by searching for the poison name. The spell ID is in the URL (e.g., `wowhead.com/spell=2823/deadly-poison`). You can also use `/dump C_UnitAuras.GetPlayerAuraBySpellID(2823)` in-game to verify a specific ID returns data when the poison is active.

### Event Dispatcher

The table-dispatch pattern maps event names to handler functions. When WoW fires an event, we look it up in the table and call the matching handler. This is cleaner and faster than a chain of `if/elseif` checks.

```lua
-- Event dispatch table
local eventHandlers = {}
local eventFrame = CreateFrame("Frame")
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = eventHandlers[event]
    if handler then handler(self, event, ...) end
end)

local function RegisterEvent(event, handler)
    eventHandlers[event] = handler
    eventFrame:RegisterEvent(event)
end

local function UnregisterEvent(event)
    eventHandlers[event] = nil
    eventFrame:UnregisterEvent(event)
end

-- Export to namespace for Config.lua
ns.RegisterEvent = RegisterEvent
ns.UnregisterEvent = UnregisterEvent
```

### Class Check

PoisonPal only makes sense for Rogues. We check the player's class early and skip all initialization if it doesn't match. The second return value from `UnitClass()` is the locale-independent class token — always `"ROGUE"` regardless of language.

```lua
-- Rogue class check (locale-independent)
local _, playerClass = UnitClass("player")
local isRogue = (playerClass == "ROGUE")
```

### Poison Check Logic

The core of PoisonPal uses `C_UnitAuras.GetPlayerAuraBySpellID()` to check whether the player has an active poison buff. We iterate through each poison table and return the name of the first match found.

```lua
-- Check if the player has any buff from a given poison table
local function FindActivePoison(poisonTable)
    for spellID, name in pairs(poisonTable) do
        local aura = C_UnitAuras.GetPlayerAuraBySpellID(spellID)
        if aura then
            return name
        end
    end
    return nil
end

-- Returns: lethalName or nil, nonLethalName or nil
local function GetPoisonStatus()
    local lethal = FindActivePoison(LETHAL_POISONS)
    local nonLethal = FindActivePoison(NON_LETHAL_POISONS)
    return lethal, nonLethal
end

local function CheckPoisons()
    if not isRogue or not ns.db or not ns.db.enabled then return end

    local lethal, nonLethal = GetPoisonStatus()
    local warnings = {}

    if not lethal then
        warnings[#warnings + 1] = "Lethal poison is MISSING!"
    end

    if not nonLethal then
        warnings[#warnings + 1] = "Non-lethal poison is MISSING!"
    end

    return warnings, lethal, nonLethal
end

-- Export for Addon Compartment use
ns.CheckPoisons = CheckPoisons
ns.GetPoisonStatus = GetPoisonStatus
```

!!! danger "GetWeaponEnchantInfo() Does NOT Detect Poisons"
    This is the most common mistake when building a poison reminder addon. AI coding assistants trained on pre-Dragonflight code will generate `GetWeaponEnchantInfo()` for poison detection. **This has not worked for poisons since Dragonflight (10.0).** Poisons are now player buffs detected via `C_UnitAuras.GetPlayerAuraBySpellID(spellID)`.

    ```lua
    -- ❌ WRONG — does NOT detect modern poisons:
    local hasMainHand, mainExpiration = GetWeaponEnchantInfo()

    -- ✅ CORRECT — poisons are player buffs:
    local aura = C_UnitAuras.GetPlayerAuraBySpellID(2823)  -- Deadly Poison
    if aura then
        -- Poison is active
    end
    ```

### Alert Output

We keep alerts simple — chat messages with color coding. This is `blizzard-faithful` mode: no custom frames needed for basic warnings.

```lua
local CHAT_PREFIX = "|cff00cc66[PoisonPal]|r "

local function PrintWarnings(warnings)
    if not warnings or #warnings == 0 then return end

    for _, msg in ipairs(warnings) do
        print(CHAT_PREFIX .. "|cffff6600" .. msg .. "|r")
    end

    -- Play warning sound if enabled
    if ns.db.playSound then
        PlaySound(SOUNDKIT.RAID_WARNING, "Master")
    end
end

local function PrintStatus()
    local warnings, lethal, nonLethal = CheckPoisons()

    if lethal then
        print(CHAT_PREFIX .. "|cff00ff00Lethal:|r " .. lethal)
    else
        print(CHAT_PREFIX .. "|cffff6600Lethal:|r None applied")
    end

    if nonLethal then
        print(CHAT_PREFIX .. "|cff00ff00Non-lethal:|r " .. nonLethal)
    else
        print(CHAT_PREFIX .. "|cffff6600Non-lethal:|r None applied")
    end
end

ns.PrintStatus = PrintStatus
```

### Event Handlers

Now we wire up the events that trigger poison checks:

```lua
-- ADDON_LOADED: initialize SavedVariables
RegisterEvent("ADDON_LOADED", function(self, event, loadedAddon)
    if loadedAddon ~= addonName then return end
    UnregisterEvent("ADDON_LOADED")

    -- Skip everything for non-Rogues
    if not isRogue then return end

    -- Initialize SavedVariables
    if not PoisonPalDB then
        PoisonPalDB = {}
    end
    ns.db = PoisonPalDB

    -- Merge defaults for new/missing keys
    for key, defaultValue in pairs(ns.defaults) do
        if ns.db[key] == nil then
            ns.db[key] = defaultValue
        end
    end

    -- Register slash commands
    SLASH_POISONPAL1 = "/poisonpal"
    SLASH_POISONPAL2 = "/pp"
    SlashCmdList["POISONPAL"] = function(input)
        local cmd = strlower(strtrim(input or ""))
        if cmd == "config" or cmd == "options" then
            if ns.settingsCategoryID then
                Settings.OpenToCategory(ns.settingsCategoryID)
            end
        elseif cmd == "check" then
            PrintStatus()
        else
            print(CHAT_PREFIX .. "Commands:")
            print("  /pp check   — Check poison status now")
            print("  /pp config  — Open settings")
        end
    end

    -- Register settings panel
    if ns.RegisterSettings then
        ns:RegisterSettings()
    end
end)

-- PLAYER_ENTERING_WORLD: check poisons on login and instance entry
RegisterEvent("PLAYER_ENTERING_WORLD", function(self, event, isLogin, isReload)
    if not isRogue or not ns.db or not ns.db.enabled then return end

    if ns.db.showOnLogin and (isLogin or isReload) then
        -- Delay slightly — aura data may not be ready immediately
        C_Timer.After(2, function()
            local warnings = CheckPoisons()
            if warnings and #warnings > 0 then
                PrintWarnings(warnings)
            end
        end)
    end
end)

-- UNIT_AURA: fires when buffs change on a unit (including poison application/removal)
RegisterEvent("UNIT_AURA", function(self, event, unit)
    if unit ~= "player" then return end
    if not isRogue or not ns.db or not ns.db.enabled then return end

    -- Check after a brief delay (aura data updates slightly after the event)
    C_Timer.After(0.5, function()
        local warnings = CheckPoisons()
        if warnings and #warnings > 0 then
            PrintWarnings(warnings)
        end
    end)
end)
```

!!! warning "Why UNIT_AURA, not UNIT_INVENTORY_CHANGED?"
    Rogue poisons are **player buffs** in modern WoW. `UNIT_AURA` fires when buffs and debuffs change on a unit — this is the correct event for detecting poison application and removal. `UNIT_INVENTORY_CHANGED` fires when equipment changes, which does **not** include poison buffs. This is the opposite of what older addon guides (and AI-generated code) will tell you.

!!! tip "RegisterUnitEvent for Efficiency"
    For production addons, prefer `eventFrame:RegisterUnitEvent("UNIT_AURA", "player")` over `eventFrame:RegisterEvent("UNIT_AURA")`. The unit-filtered version only fires for the specified unit, avoiding unnecessary handler calls when other units' auras change. We use the simpler form here for clarity.

### Periodic Check

Since poisons don't expire in modern WoW, the periodic check is a safety net — it catches edge cases like poisons being removed by mechanics or the player cancelling a buff.

```lua
RegisterEvent("PLAYER_LOGIN", function()
    if not isRogue then return end
    UnregisterEvent("PLAYER_LOGIN")

    -- Periodic check every 60 seconds as a safety net
    ns.checkTicker = C_Timer.NewTicker(60, function()
        if not ns.db or not ns.db.enabled then return end
        local warnings = CheckPoisons()
        if warnings and #warnings > 0 then
            PrintWarnings(warnings)
        end
    end)
end)
```

!!! success "Checkpoint"
    At this point, `Core.lua` handles class detection, SavedVariables, event-driven poison monitoring via aura detection, slash commands, and chat alerts. Save the file and move on to the settings panel.

---

## Step 5: Alert System

Our `blizzard-faithful` approach keeps alerts in chat output using `print()` with WoW color codes. This is intentional — no custom frames to maintain, no taint risk, no combat lockdown issues. Chat messages work everywhere, always.

The color coding gives visual urgency:

| Color Code | Hex | Used For |
|-----------|-----|----------|
| `\|cff00cc66` | Green | PoisonPal prefix |
| `\|cffff6600` | Orange | Missing poison warnings |
| `\|cff00ff00` | Bright green | Active poison names / all-clear |

For the sound alert, `PlaySound(SOUNDKIT.RAID_WARNING, "Master")` plays the familiar raid warning sound through the master audio channel, so it's audible even if other sound channels are muted.

!!! tip "Want a Visual Alert Frame?"
    A `blizzard-faithful` addon can still create its own frames — the restriction is against hooking or modifying *Blizzard's* frames. If you want a popup alert instead of chat messages, create a simple frame with `CreateFrame("Frame", nil, UIParent, "BackdropTemplate")`, position it, and show/hide it on warnings. See the [starter template's Core.lua](../starter-template.md) for a complete draggable frame example.

---

## Step 6: Settings Panel

Create `Config.lua`. This uses the modern Settings API introduced in 10.0 and updated in 11.0.2 — **not** the deprecated `InterfaceOptions_AddCategory()` or AceConfig.

```lua
local addonName, ns = ...

function ns:RegisterSettings()
    -- Create a vertical layout category (auto-stacking controls)
    local category = Settings.RegisterVerticalLayoutCategory(addonName)
    ns.settingsCategoryID = category:GetID()

    -- ================================================================
    -- Checkbox: Enable/Disable
    -- ================================================================
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "PoisonPal_Enabled",    -- Unique ID
            "enabled",              -- Key in ns.db
            ns.db,                  -- SavedVariables table
            type(true),             -- Variable type (boolean)
            "Enable PoisonPal",     -- Display name
            ns.defaults.enabled     -- Default value
        )

        Settings.CreateCheckbox(category, setting,
            "Enable or disable poison monitoring and alerts.")

        setting:SetValueChangedCallback(function(_, newValue)
            ns.db.enabled = newValue
            if newValue then
                print("|cff00cc66[PoisonPal]|r Enabled.")
            else
                print("|cff00cc66[PoisonPal]|r Disabled.")
            end
        end)
    end

    -- ================================================================
    -- Checkbox: Play Sound
    -- ================================================================
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "PoisonPal_Sound",
            "playSound",
            ns.db,
            type(true),
            "Play Warning Sound",
            ns.defaults.playSound
        )

        Settings.CreateCheckbox(category, setting,
            "Play the raid warning sound when poisons are missing.")
    end

    -- ================================================================
    -- Checkbox: Check on Login
    -- ================================================================
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "PoisonPal_ShowOnLogin",
            "showOnLogin",
            ns.db,
            type(true),
            "Check on Login",
            ns.defaults.showOnLogin
        )

        Settings.CreateCheckbox(category, setting,
            "Automatically check poison status when you log in or reload the UI.")
    end

    -- Register the category (must be last)
    Settings.RegisterAddOnCategory(category)
end
```

### How the Settings API Works

The Settings API is a three-step process:

1. **Register a category** — `Settings.RegisterVerticalLayoutCategory(name)` creates a section in Game Menu :material-arrow-right: Options :material-arrow-right: AddOns that auto-stacks controls vertically.

2. **Register settings** — `Settings.RegisterAddOnSetting()` binds a setting to a key in your SavedVariables table. The 11.0.2+ signature takes 7 arguments:

    ```lua
    Settings.RegisterAddOnSetting(
        categoryTbl,   -- From RegisterVerticalLayoutCategory
        variable,      -- Unique string ID (convention: "AddonName_Setting")
        variableKey,   -- Key name in the table (string)
        variableTbl,   -- Your SavedVariables table (ns.db)
        variableType,  -- type() of the value: type(true), type(1), type("")
        name,          -- Display name for the player
        defaultValue   -- Used by the "Defaults" button
    )
    ```

3. **Create controls** — `Settings.CreateCheckbox()`, `Settings.CreateSlider()`, and `Settings.CreateDropdown()` generate the visual controls. Each takes the category, the setting object, and tooltip text.

!!! danger "Do Not Use the Old API"
    `InterfaceOptions_AddCategory()` and `InterfaceOptionsFrame_OpenToCategory()` were removed in Dragonflight. If you see addon code using these, it's outdated. Always use `Settings.RegisterAddOnCategory()` and `Settings.OpenToCategory()`.

---

## Step 7: Addon Compartment

The Addon Compartment is the minimap dropdown introduced in 10.1.0. It replaced the old approach of creating a minimap button with LibDBIcon. Three TOC fields register your addon, and three global functions handle interaction.

We already declared the TOC fields in Step 3. Now add the callback functions to the bottom of `Core.lua`:

```lua
-- ============================================================================
-- Addon Compartment Callbacks
-- ============================================================================
-- These MUST be global functions — the TOC references them by name.
-- They only execute for Rogues because CheckPoisons() has the class guard.

-- Left-click: check poison status. Right-click: open settings.
function PoisonPal_OnCompartmentClick(addonName, buttonName)
    if not isRogue then
        print(CHAT_PREFIX .. "PoisonPal only works on Rogue characters.")
        return
    end

    if buttonName == "RightButton" then
        if ns.settingsCategoryID then
            Settings.OpenToCategory(ns.settingsCategoryID)
        end
    else
        PrintStatus()
    end
end

-- Tooltip on hover
function PoisonPal_OnCompartmentEnter(addonName, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("PoisonPal")

    if isRogue then
        local lethal, nonLethal = GetPoisonStatus()

        if lethal then
            GameTooltip:AddLine("Lethal: " .. lethal, 0, 1, 0)
        else
            GameTooltip:AddLine("Lethal: NONE", 1, 0.4, 0)
        end

        if nonLethal then
            GameTooltip:AddLine("Non-lethal: " .. nonLethal, 0, 1, 0)
        else
            GameTooltip:AddLine("Non-lethal: NONE", 1, 0.4, 0)
        end

        GameTooltip:AddLine(" ")
        GameTooltip:AddLine("Left-click to check status.", 0.7, 0.7, 0.7)
        GameTooltip:AddLine("Right-click for settings.", 0.7, 0.7, 0.7)
    else
        GameTooltip:AddLine("Only active on Rogue characters.", 0.7, 0.7, 0.7)
    end

    GameTooltip:Show()
end

function PoisonPal_OnCompartmentLeave(addonName, menuButtonFrame)
    GameTooltip:Hide()
end
```

The tooltip is the real value of the Addon Compartment integration. Players can hover over PoisonPal in the minimap dropdown and see their poison status at a glance — green for active (with poison name), orange for missing — without clicking anything.

!!! info "Addon Compartment vs. LibDBIcon"
    The Addon Compartment is a native Blizzard feature — no library dependency required. It's the `blizzard-faithful` way to add a minimap presence. `LibDBIcon` creates a standalone minimap button with more customization options, but it requires an external library and adds complexity. For most addons, the Compartment is sufficient.

---

## Step 8: Testing and Review

### Install the Addon

Copy the `PoisonPal` folder to your WoW AddOns directory:

```bash
cp -r PoisonPal "/path/to/World of Warcraft/_retail_/Interface/AddOns/"
```

### In-Game Testing

1. Launch WoW and log in on a **Rogue** character
2. Check the addon list (Game Menu :material-arrow-right: AddOns) — PoisonPal should appear with its icon
3. After loading in, you should see a warning if poisons aren't applied
4. Apply poisons and type `/pp check` — you should see the poison names displayed
5. Open settings with `/pp config` and verify the controls work
6. Find PoisonPal in the minimap Addon Compartment dropdown and hover for the tooltip
7. Log in on a **non-Rogue** character — PoisonPal should be completely silent

### Useful Debug Commands

```
/reload                     -- Reload the UI without logging out
/dump PoisonPalDB           -- Inspect your SavedVariables
/dump C_UnitAuras.GetPlayerAuraBySpellID(2823)  -- Check Deadly Poison buff
/console scriptErrors 1     -- Show Lua errors inline
```

!!! tip "Install BugGrabber + BugSack"
    These two addons capture Lua errors and display them in a browsable window instead of spamming your chat. Essential for addon development.

### Run a Review

With better-addons installed, run a code review:

```
/wow-review
```

The Reviewer agent checks for deprecated APIs, taint risks, Secret Values issues, combat lockdown violations, and Lua 5.1 compatibility. PoisonPal should score well — we used only documented APIs, avoided hooking any Blizzard frames, and don't touch any values that could be secret.

### Common Mistakes

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Addon not found" in addon list | Folder name doesn't match `.toc` filename | Ensure both are exactly `PoisonPal` |
| No warnings appear | SavedVariables `enabled` is false, or not a Rogue | Check `/dump PoisonPalDB` and character class |
| Poisons applied but addon says missing | Using `GetWeaponEnchantInfo()` instead of `C_UnitAuras` | Modern poisons are buffs — use `C_UnitAuras.GetPlayerAuraBySpellID()` |
| Settings panel missing | `RegisterSettings` called before `ADDON_LOADED` | Ensure Settings registration happens inside the ADDON_LOADED handler |
| Slash commands don't work | `SLASH_POISONPAL1` not set as global | Slash command globals must not be `local` |
| Warnings fire on every aura change | No debounce on `UNIT_AURA` handler | Use `C_Timer.After(0.5, ...)` to debounce |

---

## Step 9: Publishing

When your addon is ready for public release, you need two things: a `.pkgmeta` file for the BigWigsMods/packager, and a GitHub Actions workflow.

### .pkgmeta

Create `.pkgmeta` in the project root:

```yaml
package-as: PoisonPal

enable-nolib-creation: no

ignore:
  - .github
  - .vscode
  - .luacheckrc
  - README.md
  - CLAUDE.md
```

Since PoisonPal uses no external libraries, the config is minimal. `package-as` must match your `.toc` and folder name.

### GitHub Actions

Create `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: BigWigsMods/packager@v2
        with:
          args: -g ${{ github.event.repository.full_name }}
        env:
          CF_API_KEY: ${{ secrets.CF_API_KEY }}
          WAGO_API_TOKEN: ${{ secrets.WAGO_API_TOKEN }}
          GITHUB_OAUTH: ${{ secrets.GITHUB_TOKEN }}
```

### Release Workflow

1. Create projects on [CurseForge](https://www.curseforge.com/) and/or [Wago.io](https://addons.wago.io/)
2. Add your project IDs to the `.toc`:
    ```toc
    ## X-Curse-Project-ID: 123456
    ## X-Wago-ID: abcdefgh
    ```
3. Add API keys as GitHub repository secrets (`CF_API_KEY`, `WAGO_API_TOKEN`)
4. Tag and push:
    ```bash
    git tag v1.0.0 -m "Initial release"
    git push --tags
    ```

The packager builds a distributable `.zip`, replaces `@project-version@` with the tag, and uploads to your configured platforms.

---

## Step 10: Complete Code

Here are the three files in their entirety. Copy these into a `PoisonPal` folder, place it in your WoW AddOns directory, and you have a working addon.

### PoisonPal.toc

```toc
## Interface: 120001
## Title: PoisonPal
## Notes: Rogue poison reminder — warns when poisons are missing.
## Author: YourName
## Version: @project-version@
## SavedVariables: PoisonPalDB
## IconTexture: Interface\Icons\Ability_Rogue_DualWeild
## AddonCompartmentFunc: PoisonPal_OnCompartmentClick
## AddonCompartmentFuncOnEnter: PoisonPal_OnCompartmentEnter
## AddonCompartmentFuncOnLeave: PoisonPal_OnCompartmentLeave

Core.lua
Config.lua
```

### Core.lua

```lua
-- ============================================================================
-- Core.lua — PoisonPal: Rogue Poison Reminder
-- ============================================================================
-- Monitors poison buffs via C_UnitAuras, warns when missing.
-- Only active on Rogue characters. Uses blizzard-faithful patterns.
-- ============================================================================

local addonName, ns = ...

ns.addonName = addonName

-- Default settings
ns.defaults = {
    enabled = true,
    playSound = true,
    showOnLogin = true,
}

-- Lethal poisons (spell IDs for the player buff)
local LETHAL_POISONS = {
    [2823]   = "Deadly Poison",
    [315584] = "Instant Poison",
    [8679]   = "Wound Poison",
    [381637] = "Amplifying Poison",
}

-- Non-lethal poisons (spell IDs for the player buff)
local NON_LETHAL_POISONS = {
    [3408]   = "Crippling Poison",
    [5761]   = "Numbing Poison",
    [381664] = "Atrophic Poison",
}

-- ============================================================================
-- Event Dispatcher
-- ============================================================================

local eventHandlers = {}
local eventFrame = CreateFrame("Frame")
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = eventHandlers[event]
    if handler then handler(self, event, ...) end
end)

local function RegisterEvent(event, handler)
    eventHandlers[event] = handler
    eventFrame:RegisterEvent(event)
end

local function UnregisterEvent(event)
    eventHandlers[event] = nil
    eventFrame:UnregisterEvent(event)
end

ns.RegisterEvent = RegisterEvent
ns.UnregisterEvent = UnregisterEvent

-- ============================================================================
-- Class Check
-- ============================================================================

local _, playerClass = UnitClass("player")
local isRogue = (playerClass == "ROGUE")

-- ============================================================================
-- Poison Detection (Aura-based)
-- ============================================================================

local function FindActivePoison(poisonTable)
    for spellID, name in pairs(poisonTable) do
        local aura = C_UnitAuras.GetPlayerAuraBySpellID(spellID)
        if aura then
            return name
        end
    end
    return nil
end

local function GetPoisonStatus()
    local lethal = FindActivePoison(LETHAL_POISONS)
    local nonLethal = FindActivePoison(NON_LETHAL_POISONS)
    return lethal, nonLethal
end

local function CheckPoisons()
    if not isRogue or not ns.db or not ns.db.enabled then return end

    local lethal, nonLethal = GetPoisonStatus()
    local warnings = {}

    if not lethal then
        warnings[#warnings + 1] = "Lethal poison is MISSING!"
    end

    if not nonLethal then
        warnings[#warnings + 1] = "Non-lethal poison is MISSING!"
    end

    return warnings, lethal, nonLethal
end

ns.CheckPoisons = CheckPoisons
ns.GetPoisonStatus = GetPoisonStatus

-- ============================================================================
-- Chat Output
-- ============================================================================

local CHAT_PREFIX = "|cff00cc66[PoisonPal]|r "

local function PrintWarnings(warnings)
    if not warnings or #warnings == 0 then return end
    for _, msg in ipairs(warnings) do
        print(CHAT_PREFIX .. "|cffff6600" .. msg .. "|r")
    end
    if ns.db.playSound then
        PlaySound(SOUNDKIT.RAID_WARNING, "Master")
    end
end

local function PrintStatus()
    local warnings, lethal, nonLethal = CheckPoisons()

    if lethal then
        print(CHAT_PREFIX .. "|cff00ff00Lethal:|r " .. lethal)
    else
        print(CHAT_PREFIX .. "|cffff6600Lethal:|r None applied")
    end

    if nonLethal then
        print(CHAT_PREFIX .. "|cff00ff00Non-lethal:|r " .. nonLethal)
    else
        print(CHAT_PREFIX .. "|cffff6600Non-lethal:|r None applied")
    end
end

ns.PrintStatus = PrintStatus

-- ============================================================================
-- ADDON_LOADED
-- ============================================================================

RegisterEvent("ADDON_LOADED", function(self, event, loadedAddon)
    if loadedAddon ~= addonName then return end
    UnregisterEvent("ADDON_LOADED")

    if not isRogue then return end

    -- Initialize SavedVariables
    if not PoisonPalDB then
        PoisonPalDB = {}
    end
    ns.db = PoisonPalDB

    for key, defaultValue in pairs(ns.defaults) do
        if ns.db[key] == nil then
            ns.db[key] = defaultValue
        end
    end

    -- Slash commands
    SLASH_POISONPAL1 = "/poisonpal"
    SLASH_POISONPAL2 = "/pp"
    SlashCmdList["POISONPAL"] = function(input)
        local cmd = strlower(strtrim(input or ""))
        if cmd == "config" or cmd == "options" then
            if ns.settingsCategoryID then
                Settings.OpenToCategory(ns.settingsCategoryID)
            end
        elseif cmd == "check" then
            PrintStatus()
        else
            print(CHAT_PREFIX .. "Commands:")
            print("  /pp check   — Check poison status now")
            print("  /pp config  — Open settings")
        end
    end

    -- Register settings panel
    if ns.RegisterSettings then
        ns:RegisterSettings()
    end
end)

-- ============================================================================
-- PLAYER_ENTERING_WORLD
-- ============================================================================

RegisterEvent("PLAYER_ENTERING_WORLD", function(self, event, isLogin, isReload)
    if not isRogue or not ns.db or not ns.db.enabled then return end

    if ns.db.showOnLogin and (isLogin or isReload) then
        C_Timer.After(2, function()
            local warnings = CheckPoisons()
            if warnings and #warnings > 0 then
                PrintWarnings(warnings)
            end
        end)
    end
end)

-- ============================================================================
-- UNIT_AURA — poison buff applied/removed
-- ============================================================================

RegisterEvent("UNIT_AURA", function(self, event, unit)
    if unit ~= "player" then return end
    if not isRogue or not ns.db or not ns.db.enabled then return end

    C_Timer.After(0.5, function()
        local warnings = CheckPoisons()
        if warnings and #warnings > 0 then
            PrintWarnings(warnings)
        end
    end)
end)

-- ============================================================================
-- Periodic Check — safety net for edge cases
-- ============================================================================

RegisterEvent("PLAYER_LOGIN", function()
    if not isRogue then return end
    UnregisterEvent("PLAYER_LOGIN")

    ns.checkTicker = C_Timer.NewTicker(60, function()
        if not ns.db or not ns.db.enabled then return end
        local warnings = CheckPoisons()
        if warnings and #warnings > 0 then
            PrintWarnings(warnings)
        end
    end)
end)

-- ============================================================================
-- Addon Compartment Callbacks (must be global)
-- ============================================================================

function PoisonPal_OnCompartmentClick(addonName, buttonName)
    if not isRogue then
        print(CHAT_PREFIX .. "PoisonPal only works on Rogue characters.")
        return
    end

    if buttonName == "RightButton" then
        if ns.settingsCategoryID then
            Settings.OpenToCategory(ns.settingsCategoryID)
        end
    else
        PrintStatus()
    end
end

function PoisonPal_OnCompartmentEnter(addonName, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("PoisonPal")

    if isRogue then
        local lethal, nonLethal = GetPoisonStatus()

        if lethal then
            GameTooltip:AddLine("Lethal: " .. lethal, 0, 1, 0)
        else
            GameTooltip:AddLine("Lethal: NONE", 1, 0.4, 0)
        end

        if nonLethal then
            GameTooltip:AddLine("Non-lethal: " .. nonLethal, 0, 1, 0)
        else
            GameTooltip:AddLine("Non-lethal: NONE", 1, 0.4, 0)
        end

        GameTooltip:AddLine(" ")
        GameTooltip:AddLine("Left-click to check status.", 0.7, 0.7, 0.7)
        GameTooltip:AddLine("Right-click for settings.", 0.7, 0.7, 0.7)
    else
        GameTooltip:AddLine("Only active on Rogue characters.", 0.7, 0.7, 0.7)
    end

    GameTooltip:Show()
end

function PoisonPal_OnCompartmentLeave(addonName, menuButtonFrame)
    GameTooltip:Hide()
end
```

### Config.lua

```lua
-- ============================================================================
-- Config.lua — PoisonPal Settings Panel
-- ============================================================================
-- Uses the modern Settings API (11.0.2+ signatures).
-- Creates controls in Game Menu > Options > AddOns > PoisonPal.
-- ============================================================================

local addonName, ns = ...

function ns:RegisterSettings()
    local category = Settings.RegisterVerticalLayoutCategory(addonName)
    ns.settingsCategoryID = category:GetID()

    -- Enable/Disable
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "PoisonPal_Enabled",
            "enabled",
            ns.db,
            type(true),
            "Enable PoisonPal",
            ns.defaults.enabled
        )
        Settings.CreateCheckbox(category, setting,
            "Enable or disable poison monitoring and alerts.")
        setting:SetValueChangedCallback(function(_, newValue)
            ns.db.enabled = newValue
            if newValue then
                print("|cff00cc66[PoisonPal]|r Enabled.")
            else
                print("|cff00cc66[PoisonPal]|r Disabled.")
            end
        end)
    end

    -- Play Sound
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "PoisonPal_Sound",
            "playSound",
            ns.db,
            type(true),
            "Play Warning Sound",
            ns.defaults.playSound
        )
        Settings.CreateCheckbox(category, setting,
            "Play the raid warning sound when poisons are missing.")
    end

    -- Check on Login
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "PoisonPal_ShowOnLogin",
            "showOnLogin",
            ns.db,
            type(true),
            "Check on Login",
            ns.defaults.showOnLogin
        )
        Settings.CreateCheckbox(category, setting,
            "Automatically check poison status when you log in or reload the UI.")
    end

    Settings.RegisterAddOnCategory(category)
end
```

---

## What You Learned

This tutorial covered the core building blocks of modern WoW addon development:

| Concept | What You Used |
|---------|--------------|
| **Project structure** | `.toc` manifest, file load order, SavedVariables declaration |
| **Namespace pattern** | `local addonName, ns = ...` for cross-file communication |
| **Event dispatch** | Table-dispatch pattern with `RegisterEvent` / `UnregisterEvent` helpers |
| **Initialization order** | `ADDON_LOADED` for DB, `PLAYER_LOGIN` for features, `PLAYER_ENTERING_WORLD` for world state |
| **Aura detection API** | `C_UnitAuras.GetPlayerAuraBySpellID()` for poison buff detection |
| **Timer API** | `C_Timer.After()` for delays, `C_Timer.NewTicker()` for periodic checks |
| **Settings API** | `RegisterVerticalLayoutCategory`, `RegisterAddOnSetting`, `CreateCheckbox` |
| **Addon Compartment** | TOC fields + global callback functions for minimap dropdown integration |
| **Slash commands** | `SLASH_NAME1` globals + `SlashCmdList` handler |
| **Publishing** | `.pkgmeta` + GitHub Actions + BigWigsMods/packager |

### Common AI Mistakes This Tutorial Avoids

| Mistake | Why It's Wrong | What We Did Instead |
|---------|---------------|-------------------|
| Using `GetWeaponEnchantInfo()` for poisons | Poisons are player buffs since Dragonflight, not weapon enchants | `C_UnitAuras.GetPlayerAuraBySpellID()` |
| Listening to `UNIT_INVENTORY_CHANGED` | Equipment events don't fire for buff changes | `UNIT_AURA` event |
| Tracking poison expiry timers | Modern poisons are permanent until cancelled | Check presence/absence only |
| Using `InterfaceOptionsCheckButtonTemplate` | Deprecated since Dragonflight | Modern Settings API |

### Next Steps

- **[Starter Template](../starter-template.md)** — The full template with more patterns: draggable frames, combat lockdown queue, hooksecurefunc hooks
- **[API Cheat Sheet](../api-cheatsheet.md)** — Verified 12.0.1 API signatures for quick reference
- **[Code Patterns](../better-patterns.md)** — 10 production patterns from real addons
- **[Plugin Commands](../plugin/commands.md)** — All 10 `/wow-*` commands with detailed usage
- **[Enhancement Tutorials](../enhancement-tutorials.md)** — Advanced tutorials for frame skinning and UI enhancement
