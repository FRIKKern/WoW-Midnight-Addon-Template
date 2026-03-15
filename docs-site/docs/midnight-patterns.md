---
title: "Coding for Midnight: The New Paradigm"
description: "The definitive guide to WoW addon coding patterns for Patch 12.0+ (Midnight) — complete, copy-pasteable examples for the new API."
---

# Coding for Midnight: The New Paradigm

Midnight didn't just change *what* addons can do — it changed *how* you should think about building them. The old approach (create custom frames, read all game data, build parallel UI systems) is dead. The new approach: **skin Blizzard's containers, display within the rules, and build where Blizzard doesn't.**

This page provides complete, copy-pasteable code patterns for every major addon development scenario in Patch 12.0+.

!!! tip "Prerequisites"
    This page assumes you've read [Midnight (Patch 12.0)](midnight.md) for the policy context and [Security Model](security.md) for taint/protected function fundamentals. Here we focus on *code* — how to actually implement addons that work under the new rules.

---

## The Philosophy: Skin, Don't Replace

### The Old Way vs. The New Way

=== "Old Way (Pre-Midnight)"

    ```lua
    -- OLD: Build your own unit frame from scratch
    -- Read all data directly, create parallel UI
    local myFrame = CreateFrame("Frame", "MyUnitFrame", UIParent)
    local healthBar = CreateFrame("StatusBar", nil, myFrame)

    myFrame:RegisterEvent("UNIT_HEALTH")
    myFrame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
    myFrame:SetScript("OnEvent", function(self, event, ...)
        if event == "UNIT_HEALTH" then
            local unit = ...
            -- Read health directly, do math, update bar
            local hp = UnitHealth(unit)
            local maxHp = UnitHealthMax(unit)
            healthBar:SetValue(hp / maxHp * 100)

            -- Change color based on health percentage
            local pct = hp / maxHp
            if pct < 0.2 then
                healthBar:SetStatusBarColor(1, 0, 0)
            elseif pct < 0.5 then
                healthBar:SetStatusBarColor(1, 1, 0)
            else
                healthBar:SetStatusBarColor(0, 1, 0)
            end
        elseif event == "COMBAT_LOG_EVENT_UNFILTERED" then
            -- Parse combat log for threat, damage, etc.
            local _, subevent = CombatLogGetCurrentEventInfo()
            -- ... complex combat log parsing
        end
    end)
    ```

=== "New Way (Midnight 12.0+)"

    ```lua
    -- NEW: Hook and skin Blizzard's existing unit frames
    -- Let Blizzard handle data flow; you handle appearance

    -- Restyle Blizzard's CompactUnitFrame (used in raid/party)
    hooksecurefunc("CompactUnitFrame_UpdateHealth", function(frame)
        if frame:IsForbidden() then return end

        -- Change the health bar texture
        frame.healthBar:SetStatusBarTexture(
            "Interface\\AddOns\\MyAddon\\Textures\\StatusBar"
        )
    end)

    hooksecurefunc("CompactUnitFrame_UpdateName", function(frame)
        if frame:IsForbidden() then return end

        -- Restyle the name text
        frame.name:SetFont("Fonts\\FRIZQT__.TTF", 11, "OUTLINE")
        frame.name:SetShadowOffset(1, -1)
    end)
    ```

### Why This Matters

In Midnight, combat data flows through Blizzard's **Secret Values** pipeline. When you create your own frames and try to read combat data directly, you hit the secret value wall — values return as opaque tokens that can't be used in Lua conditionals. But Blizzard's own frames receive this data through internal channels and display it correctly.

By hooking Blizzard's frames, you ride on top of their data pipeline. You change how things *look* without needing to read the underlying *values*.

---

## Pattern 1: Skinning Blizzard Containers

The core skill of Midnight addon development. This pattern lets you dramatically change the appearance of Blizzard's UI without breaking secure behavior or fighting the Secret Values system.

### Skinning Raid Frames

```lua
local addonName, ns = ...

-- Cache for performance
local hooksecurefunc = hooksecurefunc

-- Your custom texture paths
local HEALTH_TEXTURE = "Interface\\AddOns\\" .. addonName .. "\\Textures\\StatusBar"
local FONT_PATH = "Interface\\AddOns\\" .. addonName .. "\\Fonts\\Main.ttf"

-- Hook into CompactUnitFrame updates
-- This fires whenever Blizzard updates a raid/party frame
hooksecurefunc("CompactUnitFrame_UpdateAll", function(frame)
    if frame:IsForbidden() then return end

    -- Restyle the health bar
    if frame.healthBar then
        frame.healthBar:SetStatusBarTexture(HEALTH_TEXTURE)
    end

    -- Restyle the power bar (mana/energy/rage)
    if frame.powerBar then
        frame.powerBar:SetStatusBarTexture(HEALTH_TEXTURE)
    end

    -- Restyle the name
    if frame.name then
        frame.name:SetFont(FONT_PATH, 10, "OUTLINE")
    end

    -- Add a custom border if we haven't already
    if not frame.__myAddonBorder then
        local border = frame:CreateTexture(nil, "OVERLAY")
        border:SetTexture("Interface\\AddOns\\" .. addonName .. "\\Textures\\Border")
        border:SetAllPoints(frame)
        frame.__myAddonBorder = border
    end
end)
```

!!! warning "Always check `IsForbidden()`"
    Some frames are marked as **forbidden** — they belong to Blizzard's secure environment and cannot be modified by addon code. Always check `frame:IsForbidden()` before touching any frame you receive from a hook. Accessing a forbidden frame throws an error.

!!! danger "Load order gotcha: CompactRaidFrames"
    In Midnight, CompactUnitFrame code was moved into `Interface/AddOns/Blizzard_CompactRaidFrames/`. Your hooks won't fire if Blizzard's addon hasn't loaded yet. Wait for it:
    ```lua
    local frame = CreateFrame("Frame")
    frame:RegisterEvent("ADDON_LOADED")
    frame:SetScript("OnEvent", function(self, event, addon)
        if addon == "Blizzard_CompactRaidFrames" then
            self:UnregisterEvent("ADDON_LOADED")
            -- NOW it's safe to hook CompactUnitFrame functions
            hooksecurefunc("CompactUnitFrame_UpdateAll", MyHookFunc)
        end
    end)
    ```

### Skinning the Cast Bar

```lua
-- Hook Blizzard's PlayerCastingBarFrame
-- This is the main player cast bar in Midnight
hooksecurefunc(PlayerCastingBarFrame, "OnShow", function(self)
    -- Change the bar texture
    self:SetStatusBarTexture(HEALTH_TEXTURE)
    self:SetStatusBarColor(0.2, 0.6, 1.0)

    -- Restyle the border
    if self.Border then
        self.Border:SetVertexColor(0.3, 0.3, 0.3)
    end

    -- Restyle the cast text
    if self.Text then
        self.Text:SetFont(FONT_PATH, 11, "OUTLINE")
    end
end)

-- Hook the target cast bar too
if TargetFrameSpellBar then
    hooksecurefunc(TargetFrameSpellBar, "OnShow", function(self)
        self:SetStatusBarTexture(HEALTH_TEXTURE)
    end)
end
```

### Skinning Nameplates

```lua
-- Nameplates use the CompactUnitFrame system
-- Hook NamePlateDriverFrame for new nameplate creation
hooksecurefunc(NamePlateDriverFrame, "OnNamePlateAdded", function(self, namePlateUnitToken)
    local nameplate = C_NamePlate.GetNamePlateForUnit(namePlateUnitToken)
    if not nameplate then return end

    local unitFrame = nameplate.UnitFrame
    if not unitFrame or unitFrame:IsForbidden() then return end

    -- Restyle the nameplate health bar
    if unitFrame.healthBar then
        unitFrame.healthBar:SetStatusBarTexture(HEALTH_TEXTURE)
        unitFrame.healthBar:SetHeight(12)
    end

    -- Restyle the nameplate name
    if unitFrame.name then
        unitFrame.name:SetFont(FONT_PATH, 9, "OUTLINE")
        unitFrame.name:SetVertexColor(1, 1, 1)
    end
end)
```

!!! info "Nameplate depth at risk"
    The depth of nameplate customization is considered **at-risk** in 12.0.x patches. The examples above (texture, font, color changes) are considered safe. Reading combat data from nameplate units may be further restricted. See [Midnight: Nameplate Customization](midnight.md#nameplate-customization).

---

## Pattern 2: Working with Secret Values

Secret Values are the cornerstone of Midnight's addon restrictions. Understanding what you *can* do with them is just as important as knowing what you *can't*.

### Secret Value Utility Functions

Midnight provides utility functions for working with secret values defensively:

```lua
-- Check if a value is secret (returns boolean)
local isSecret = issecretvalue(someValue)

-- Wrap values as secret (useful for testing/hardening)
local wrapped = secretwrap(value1, value2, ...)

-- Strip secret values from returns (replaces secrets with nil)
local clean1, clean2 = scrubsecretvalues(val1, val2)

-- Remove secret access from the calling function entirely
dropsecretaccess()

-- Check if a specific aura should be secret
local shouldBeSecret = C_Secrets.ShouldSpellAuraBeSecret(spellID)
local shouldBeSecret = C_Secrets.ShouldUnitAuraInstanceBeSecret(
    unit, auraInstanceID)
```

!!! tip "Defensive coding with `issecretvalue()`"
    Use `issecretvalue()` to guard code paths that might receive secret data. This prevents the `"table index is secret"` runtime crashes that plagued addons in the first week of Midnight.

### What You CAN Do: ColorCurve and StatusBar Display

The primary mechanism for displaying secret health/power data is through **ColorCurve** objects and **StatusBar** widgets. These accept secret values directly — the engine handles the display internally.

```lua
-- Create a health bar that displays secret values correctly
local healthBar = CreateFrame("StatusBar", nil, UIParent)
healthBar:SetSize(200, 20)
healthBar:SetPoint("CENTER")
healthBar:SetStatusBarTexture("Interface\\TargetingFrame\\UI-StatusBar")
healthBar:SetMinMaxValues(0, 1)

-- Background
local bg = healthBar:CreateTexture(nil, "BACKGROUND")
bg:SetAllPoints()
bg:SetColorTexture(0, 0, 0, 0.5)

-- Create a ColorCurve for health-based coloring (red → yellow → green)
local healthCurve = C_CurveUtil.CreateColorCurve()
healthCurve:SetType(Enum.LuaCurveType.Linear)
healthCurve:AddPoint(0.0, CreateColor(1, 0, 0, 1))   -- 0% health = red
healthCurve:AddPoint(0.3, CreateColor(1, 1, 0, 1))   -- 30% health = yellow
healthCurve:AddPoint(0.7, CreateColor(0, 1, 0, 1))   -- 70%+ health = green
-- Max 256 points per curve

-- In your UNIT_HEALTH handler:
local function UpdateHealthDisplay(unit)
    -- UnitHealthPercent returns a secret value in combat
    -- but StatusBar:SetValue() accepts secret values directly!
    local healthPct = UnitHealthPercent(unit)
    healthBar:SetValue(healthPct)

    -- ColorCurve:Evaluate() also accepts secret values
    local color = healthCurve:Evaluate(healthPct)
    healthBar:GetStatusBarTexture():SetVertexColor(color:GetRGB())
end
```

!!! abstract "New Percentage APIs"
    Midnight added percentage-based unit APIs designed for the secret value workflow:

    - `UnitHealthPercent(unit)` — returns 0-1 range (may be secret)
    - `UnitHealthMissing(unit)` — missing health amount (may be secret)
    - `UnitPowerPercent(unit, powerType)` — returns 0-1 range (may be secret)
    - `UnitPowerMissing(unit, powerType)` — missing power (may be secret)

    These are designed for use with `StatusBar:SetValue()` and `ColorCurve:Evaluate()`.

### Duration Objects: Replacing Cooldown/Aura Timing

Several `C_Spell` and `C_UnitAuras` timing APIs were replaced by **Duration objects** — opaque containers for time data that work with secret values.

```lua
-- Create a duration container
local duration = C_DurationUtil.CreateDuration()

-- Configure the duration (multiple methods available)
duration:SetTimeSpan(startTime, endTime)
duration:SetTimeFromStart(startTime, durationSec, modRate)
duration:SetTimeFromEnd(endTime, durationSec, modRate)

-- Query duration (returns secret values in restricted content)
local elapsed = duration:GetElapsedDuration()
local remaining = duration:GetRemainingDuration()
local progress = duration:EvaluateElapsedProgress()

-- Attach to a StatusBar for timer/cooldown display
-- The engine handles the animation internally
statusBar:SetTimerDuration(duration)

-- Reverse fill for channeled spells
statusBar:SetTimerDuration(duration, Enum.StatusBarFillDirection.Reverse)

-- New casting duration APIs (return Duration objects)
local castDuration = UnitCastingDuration("player")
local channelDuration = UnitChannelDuration("player")
```

### Heal Prediction Calculator

```lua
-- Create the prediction calculator (create once, reuse)
local hpCalc = CreateUnitHealPredictionCalculator()

-- Get detailed heal prediction (in UNIT_HEALTH handler)
-- Returns secret values in restricted content, but can be
-- used directly with StatusBar overlays
local myIncoming, otherIncoming, absorb, healAbsorb =
    UnitGetDetailedHealPrediction("target", hpCalc)
```

### What You CAN'T Do: Branch on Secret Values

```lua
-- THIS WILL FAIL in combat contexts in Midnight:
local function CheckHealth()
    local hp = UnitHealth("target")  -- returns a secret value in combat

    -- Cannot compare, branch, or do math with secret values
    if hp < 1000 then        -- ERROR: cannot compare secret value
        DoSomething()
    end

    local pct = hp / UnitHealthMax("target")  -- ERROR: cannot do math
end
```

### The Workaround: Pre-Combat Caching

Data that becomes secret during combat is often freely readable *before* combat starts. Cache what you need:

```lua
local addonName, ns = ...

-- Pre-combat data cache
ns.cachedData = {}

local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_REGEN_DISABLED")  -- entering combat
frame:RegisterEvent("PLAYER_REGEN_ENABLED")   -- leaving combat
frame:RegisterEvent("GROUP_ROSTER_UPDATE")

frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_DISABLED" then
        -- Combat starting! Cache everything we might need
        ns:CacheGroupData()
    elseif event == "PLAYER_REGEN_ENABLED" then
        -- Combat over — data is freely readable again
        wipe(ns.cachedData)
    elseif event == "GROUP_ROSTER_UPDATE" then
        -- Group changed — update cache if not in combat
        if not InCombatLockdown() then
            ns:CacheGroupData()
        end
    end
end)

function ns:CacheGroupData()
    wipe(self.cachedData)

    local groupType = IsInRaid() and "raid" or "party"
    local maxMembers = IsInRaid() and MAX_RAID_MEMBERS or 4

    for i = 1, maxMembers do
        local unit = groupType .. i
        if UnitExists(unit) then
            self.cachedData[unit] = {
                name = UnitName(unit),
                class = select(2, UnitClass(unit)),
                role = UnitGroupRolesAssigned(unit),
                specID = GetInspectSpecialization(unit),
                maxHealth = UnitHealthMax(unit),
            }
        end
    end
end
```

### The Whitelist: What's Still Readable

Some combat data remains freely accessible because it's on Blizzard's **spell whitelist**. These are primarily class resources that the default UI already tracks:

```lua
-- These secondary resources are NOT secret (fully accessible):
-- Combo Points, Holy Power, Soul Shards, Arcane Charges,
-- Chi, Runes, Maelstrom Weapon stacks, Soul Fragments,
-- Essence (Evoker), and more

-- You CAN still do this in combat:
local function GetComboState()
    local combo = GetComboPoints("player", "target")
    if combo >= 5 then
        -- This works! Combo points are whitelisted
        return "FULL"
    end
    return combo
end

-- Empowered cast data is also whitelisted:
local function GetEmpowerStage()
    local info = C_Spell.GetEmpowerStageInfo(spellID)
    -- Stage count and cast percentages are accessible
    return info
end
```

!!! tip "Check the whitelist"
    The spell whitelist grows with each patch. Always check the [Wowhead whitelist article](https://www.wowhead.com/news/majority-of-addon-changes-finalized-for-midnight-pre-patch-whitelisted-spells-379738) and [Warcraft Wiki API changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) for the current state.

---

## Pattern 3: Post-Combat Data Access

Combat data restrictions are **tightest during active encounters** and relax between pulls. This creates an opportunity for analysis tools.

### Capturing Post-Combat Snapshots

```lua
local addonName, ns = ...

ns.encounterLog = {}
ns.currentEncounter = nil

local frame = CreateFrame("Frame")
frame:RegisterEvent("ENCOUNTER_START")
frame:RegisterEvent("ENCOUNTER_END")
frame:RegisterEvent("PLAYER_REGEN_ENABLED")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "ENCOUNTER_START" then
        local encounterID, encounterName, difficultyID, groupSize = ...
        ns.currentEncounter = {
            id = encounterID,
            name = encounterName,
            difficulty = difficultyID,
            startTime = GetTime(),
            data = {},
        }

    elseif event == "ENCOUNTER_END" then
        local encounterID, encounterName, difficultyID, groupSize, success = ...
        if ns.currentEncounter then
            ns.currentEncounter.endTime = GetTime()
            ns.currentEncounter.success = (success == 1)
            ns.currentEncounter.duration = ns.currentEncounter.endTime
                - ns.currentEncounter.startTime

            -- Data is more accessible now — capture a snapshot
            ns:CapturePostEncounterSnapshot()

            tinsert(ns.encounterLog, ns.currentEncounter)
            ns.currentEncounter = nil
        end

    elseif event == "PLAYER_REGEN_ENABLED" then
        -- Combat ended (not necessarily an encounter)
        -- Good time to process any queued data
        ns:ProcessQueuedData()
    end
end)

function ns:CapturePostEncounterSnapshot()
    -- Between pulls, combat data restrictions relax
    -- Capture what's available for analysis
    local snapshot = {
        timestamp = time(),
        group = {},
    }

    local groupType = IsInRaid() and "raid" or "party"
    local maxMembers = IsInRaid() and MAX_RAID_MEMBERS or 4

    for i = 1, maxMembers do
        local unit = groupType .. i
        if UnitExists(unit) then
            tinsert(snapshot.group, {
                name = UnitName(unit),
                class = select(2, UnitClass(unit)),
                role = UnitGroupRolesAssigned(unit),
                alive = not UnitIsDeadOrGhost(unit),
                health = UnitHealth(unit),        -- readable post-combat
                maxHealth = UnitHealthMax(unit),
            })
        end
    end

    if ns.currentEncounter then
        ns.currentEncounter.snapshot = snapshot
    end
end
```

### Building a Between-Pull Analysis Display

```lua
-- Show analysis between pulls, hide during combat
local analysisFrame = CreateFrame("Frame", "MyAddonAnalysis", UIParent,
    "BackdropTemplate")
analysisFrame:SetSize(300, 200)
analysisFrame:SetPoint("RIGHT", UIParent, "RIGHT", -20, 0)
analysisFrame:SetBackdrop({
    bgFile = "Interface\\DialogFrame\\UI-DialogBox-Background",
    edgeFile = "Interface\\DialogFrame\\UI-DialogBox-Border",
    tile = true, tileSize = 32, edgeSize = 16,
    insets = { left = 4, right = 4, top = 4, bottom = 4 },
})
analysisFrame:Hide()

local titleText = analysisFrame:CreateFontString(nil, "OVERLAY", "GameFontNormal")
titleText:SetPoint("TOP", 0, -10)
titleText:SetText("Post-Pull Analysis")

local bodyText = analysisFrame:CreateFontString(nil, "OVERLAY", "GameFontHighlightSmall")
bodyText:SetPoint("TOPLEFT", 12, -30)
bodyText:SetPoint("BOTTOMRIGHT", -12, 12)
bodyText:SetJustifyH("LEFT")
bodyText:SetJustifyV("TOP")

-- Show between pulls, hide during combat
local controller = CreateFrame("Frame")
controller:RegisterEvent("PLAYER_REGEN_ENABLED")
controller:RegisterEvent("PLAYER_REGEN_DISABLED")

controller:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_ENABLED" then
        -- Combat ended — show analysis
        ns:UpdateAnalysisDisplay()
        analysisFrame:Show()
    elseif event == "PLAYER_REGEN_DISABLED" then
        -- Combat started — hide
        analysisFrame:Hide()
    end
end)

function ns:UpdateAnalysisDisplay()
    local lastEncounter = ns.encounterLog[#ns.encounterLog]
    if not lastEncounter then
        bodyText:SetText("No encounter data yet.")
        return
    end

    local lines = {
        format("|cffffd100%s|r (%s)", lastEncounter.name,
            lastEncounter.success and "|cff00ff00Kill|r" or "|cffff0000Wipe|r"),
        format("Duration: %.1f seconds", lastEncounter.duration or 0),
        "",
    }

    if lastEncounter.snapshot then
        local alive, dead = 0, 0
        for _, player in ipairs(lastEncounter.snapshot.group) do
            if player.alive then alive = alive + 1 else dead = dead + 1 end
        end
        tinsert(lines, format("Alive: %d | Dead: %d", alive, dead))
    end

    bodyText:SetText(table.concat(lines, "\n"))
end
```

---

## Pattern 4: Visual-Only Frame Modifications

These operations are **always safe** — they modify appearance without touching secure behavior or secret values.

### Safe Visual Operations

```lua
-- All of these are safe to call on non-forbidden frames,
-- even during combat:

-- Change texture color (tinting)
texture:SetVertexColor(r, g, b, a)

-- Change transparency
frame:SetAlpha(0.8)
texture:SetAlpha(0.5)

-- Change scale
frame:SetScale(1.2)

-- Change texture
texture:SetTexture("Interface\\Path\\To\\Texture")
texture:SetAtlas("AtlasName")
texture:SetColorTexture(r, g, b, a)  -- solid color

-- Change font
fontString:SetFont("Fonts\\FRIZQT__.TTF", 12, "OUTLINE")
fontString:SetTextColor(r, g, b, a)
fontString:SetShadowOffset(1, -1)

-- Change status bar appearance
statusBar:SetStatusBarTexture("Interface\\Path\\To\\Texture")
statusBar:SetStatusBarColor(r, g, b, a)
statusBar:SetFillStyle("STANDARD")  -- or "REVERSE", "CENTER"

-- Add overlays (non-secure child elements)
local glow = frame:CreateTexture(nil, "OVERLAY")
glow:SetTexture("Interface\\Buttons\\CheckButtonGlow")
glow:SetAllPoints()
glow:SetBlendMode("ADD")
```

### What Triggers Taint (Avoid These on Secure Frames in Combat)

```lua
-- These operations on SECURE frames will cause taint issues
-- during InCombatLockdown():

frame:Show()              -- TAINT if secure frame
frame:Hide()              -- TAINT if secure frame
frame:SetPoint(...)       -- TAINT if secure frame
frame:SetAttribute(...)   -- TAINT if secure frame
frame:SetParent(...)      -- TAINT if secure frame
frame:ClearAllPoints()    -- TAINT if secure frame
frame:EnableMouse(true)   -- TAINT if secure frame
```

!!! tip "How to tell if a frame is secure"
    A frame is secure if it was created with a secure template (like `SecureActionButtonTemplate`) or if it's part of Blizzard's secure UI hierarchy. Your own frames created with plain `CreateFrame("Frame")` are not secure and can be freely modified at any time.

### Adding a Highlight Overlay to Any Frame

```lua
-- Reusable function to add a glow/highlight to any frame
local function AddHighlight(frame, r, g, b, a)
    if frame.__highlight then
        frame.__highlight:Show()
        frame.__highlight:SetVertexColor(r, g, b, a or 0.3)
        return frame.__highlight
    end

    local highlight = frame:CreateTexture(nil, "OVERLAY")
    highlight:SetAllPoints()
    highlight:SetColorTexture(r, g, b, a or 0.3)
    highlight:SetBlendMode("ADD")
    frame.__highlight = highlight

    return highlight
end

local function RemoveHighlight(frame)
    if frame.__highlight then
        frame.__highlight:Hide()
    end
end

-- Usage: highlight a nameplate frame
AddHighlight(someNameplateFrame, 1, 0, 0, 0.2)  -- red tint
```

---

## Pattern 5: Event-Based Information Without CLEU

`COMBAT_LOG_EVENT_UNFILTERED` is gone. But many unit-specific events still fire and provide useful data. Here's what's available and how to use it.

### Available Combat-Adjacent Events

```lua
local addonName, ns = ...
local frame = CreateFrame("Frame")

-- These events STILL FIRE in Midnight:
local availableEvents = {
    -- Unit state events (fire for specific units)
    "UNIT_HEALTH",          -- unit health changed
    "UNIT_POWER_UPDATE",    -- unit power (mana/energy/rage) changed
    "UNIT_AURA",            -- buffs/debuffs changed on unit
    "UNIT_MAXHEALTH",       -- max health changed
    "UNIT_MAXPOWER",        -- max power changed
    "UNIT_ABSORB_AMOUNT_CHANGED",
    "UNIT_HEAL_PREDICTION",

    -- Cast events
    "UNIT_SPELLCAST_START",       -- unit started casting
    "UNIT_SPELLCAST_STOP",        -- unit stopped casting
    "UNIT_SPELLCAST_SUCCEEDED",   -- cast completed successfully
    "UNIT_SPELLCAST_FAILED",      -- cast failed
    "UNIT_SPELLCAST_INTERRUPTED", -- cast was interrupted
    "UNIT_SPELLCAST_CHANNEL_START",
    "UNIT_SPELLCAST_CHANNEL_STOP",
    "UNIT_SPELLCAST_EMPOWER_START",
    "UNIT_SPELLCAST_EMPOWER_STOP",

    -- Target/threat events
    "UNIT_THREAT_LIST_UPDATE",    -- threat table changed
    "UNIT_THREAT_SITUATION_UPDATE", -- threat status changed
    "PLAYER_TARGET_CHANGED",
    "PLAYER_FOCUS_CHANGED",

    -- Encounter events (still fully available)
    "ENCOUNTER_START",
    "ENCOUNTER_END",
    "BOSS_KILL",
    "INSTANCE_ENCOUNTER_ENGAGE_UNIT",

    -- Group events
    "GROUP_ROSTER_UPDATE",
    "READY_CHECK",
    "ROLE_CHANGED_INFORM",

    -- Combat state
    "PLAYER_REGEN_DISABLED",  -- entered combat
    "PLAYER_REGEN_ENABLED",   -- left combat
    "PLAYER_DEAD",
    "PLAYER_ALIVE",
    "PLAYER_UNGHOST",
}
```

!!! danger "Key difference: display vs. branch"
    These events still *fire*, and the associated API functions (`UnitHealth()`, `UnitPower()`, etc.) still return values. But in combat contexts, those values may be **secret values** that can be displayed but not used in conditional logic. The events themselves are information that "something changed" — you just can't always inspect *what* changed.

### Building a Unit Frame with Available Events

```lua
local addonName, ns = ...

-- Create a custom display frame that works within Midnight's rules
local function CreateUnitDisplay(unit, parent)
    local display = CreateFrame("Frame", nil, parent or UIParent,
        "BackdropTemplate")
    display:SetSize(180, 50)
    display.unit = unit

    -- Background
    display:SetBackdrop({
        bgFile = "Interface\\Buttons\\WHITE8X8",
        edgeFile = "Interface\\Buttons\\WHITE8X8",
        edgeSize = 1,
    })
    display:SetBackdropColor(0, 0, 0, 0.7)
    display:SetBackdropBorderColor(0.3, 0.3, 0.3, 1)

    -- Name text (freely readable, not a secret value)
    display.nameText = display:CreateFontString(nil, "OVERLAY",
        "GameFontNormalSmall")
    display.nameText:SetPoint("TOPLEFT", 6, -6)

    -- Health bar — use StatusBar and let the update function
    -- set it through display-safe channels
    display.healthBar = CreateFrame("StatusBar", nil, display)
    display.healthBar:SetPoint("BOTTOMLEFT", 4, 4)
    display.healthBar:SetPoint("BOTTOMRIGHT", -4, 4)
    display.healthBar:SetHeight(16)
    display.healthBar:SetStatusBarTexture(
        "Interface\\TargetingFrame\\UI-StatusBar")
    display.healthBar:SetStatusBarColor(0, 0.8, 0)
    display.healthBar:SetMinMaxValues(0, 1)

    -- Health bar background
    local bg = display.healthBar:CreateTexture(nil, "BACKGROUND")
    bg:SetAllPoints()
    bg:SetColorTexture(0.15, 0.15, 0.15, 1)

    -- Health text overlay
    display.healthText = display.healthBar:CreateFontString(nil, "OVERLAY",
        "GameFontHighlightSmall")
    display.healthText:SetPoint("CENTER")

    -- Event handling
    display:RegisterUnitEvent("UNIT_HEALTH", unit)
    display:RegisterUnitEvent("UNIT_MAXHEALTH", unit)
    display:RegisterEvent("PLAYER_TARGET_CHANGED")

    display:SetScript("OnEvent", function(self, event, ...)
        ns:UpdateUnitDisplay(self)
    end)

    -- Initial update
    ns:UpdateUnitDisplay(display)

    return display
end

function ns:UpdateUnitDisplay(display)
    local unit = display.unit
    if not UnitExists(unit) then
        display:Hide()
        return
    end
    display:Show()

    -- Name and class color — always freely readable
    local name = UnitName(unit)
    local _, class = UnitClass(unit)
    local color = RAID_CLASS_COLORS[class] or RAID_CLASS_COLORS["PRIEST"]

    display.nameText:SetText(name)
    display.nameText:SetTextColor(color.r, color.g, color.b)

    -- Health — set the bar value
    -- Outside combat: these are normal numbers
    -- In combat: these may be secret values, but setting them
    -- on a StatusBar for display is allowed
    local hp = UnitHealth(unit)
    local maxHp = UnitHealthMax(unit)

    if maxHp > 0 then
        display.healthBar:SetMinMaxValues(0, maxHp)
        display.healthBar:SetValue(hp)
    end

    -- Health text (may show as secret/blank in combat
    -- depending on 12.0.x patch state)
    display.healthText:SetFormattedText("%s / %s",
        AbbreviateNumbers(hp), AbbreviateNumbers(maxHp))
end

-- Create displays
ns.targetDisplay = CreateUnitDisplay("target")
ns.targetDisplay:SetPoint("CENTER", UIParent, "CENTER", 0, -200)
```

### Cast Bar Tracking (Still Works)

```lua
-- UNIT_SPELLCAST events are still available
-- You can build cast bar displays

local castBar = CreateFrame("StatusBar", nil, UIParent)
castBar:SetSize(250, 20)
castBar:SetPoint("CENTER", 0, -150)
castBar:SetStatusBarTexture("Interface\\TargetingFrame\\UI-StatusBar")
castBar:SetStatusBarColor(1, 0.7, 0)
castBar:Hide()

local castText = castBar:CreateFontString(nil, "OVERLAY", "GameFontHighlight")
castText:SetPoint("CENTER")

local bg = castBar:CreateTexture(nil, "BACKGROUND")
bg:SetAllPoints()
bg:SetColorTexture(0, 0, 0, 0.6)

local castFrame = CreateFrame("Frame")
castFrame:RegisterUnitEvent("UNIT_SPELLCAST_START", "target")
castFrame:RegisterUnitEvent("UNIT_SPELLCAST_STOP", "target")
castFrame:RegisterUnitEvent("UNIT_SPELLCAST_SUCCEEDED", "target")
castFrame:RegisterUnitEvent("UNIT_SPELLCAST_INTERRUPTED", "target")
castFrame:RegisterUnitEvent("UNIT_SPELLCAST_CHANNEL_START", "target")
castFrame:RegisterUnitEvent("UNIT_SPELLCAST_CHANNEL_STOP", "target")

castFrame:SetScript("OnEvent", function(self, event, unit, ...)
    if event == "UNIT_SPELLCAST_START" then
        local castGUID = ...
        local info = C_Spell.GetSpellInfo(select(3, UnitCastingInfo(unit)))
        if info then
            castText:SetText(info.name)
        end
        castBar:SetMinMaxValues(0, 1)
        castBar:SetValue(0)
        castBar:Show()

    elseif event == "UNIT_SPELLCAST_STOP"
        or event == "UNIT_SPELLCAST_SUCCEEDED"
        or event == "UNIT_SPELLCAST_INTERRUPTED" then
        castBar:Hide()

    elseif event == "UNIT_SPELLCAST_CHANNEL_START" then
        castBar:SetStatusBarColor(0, 0.7, 1)
        castBar:Show()

    elseif event == "UNIT_SPELLCAST_CHANNEL_STOP" then
        castBar:Hide()
    end
end)

-- OnUpdate for smooth cast bar progress
castBar:SetScript("OnUpdate", function(self, elapsed)
    local name, _, _, startTimeMS, endTimeMS = UnitCastingInfo("target")
    if name then
        local now = GetTime() * 1000
        local progress = (now - startTimeMS) / (endTimeMS - startTimeMS)
        self:SetValue(math.min(progress, 1))
    else
        -- Check for channel
        local cName, _, _, cStart, cEnd = UnitChannelInfo("target")
        if cName then
            local now = GetTime() * 1000
            local progress = 1 - ((now - cStart) / (cEnd - cStart))
            self:SetValue(math.max(progress, 0))
        else
            self:Hide()
        end
    end
end)
```

---

## Pattern 6: EventRegistry (Frame-Free Event Handling)

Blizzard's `EventRegistry` provides a modern, frame-free alternative to the traditional `CreateFrame()` + `RegisterEvent()` pattern. It inherits from `CallbackRegistryMixin` and is increasingly used in Blizzard's own Midnight UI code.

### Basic Usage

```lua
-- Register for a frame event WITHOUT creating a frame
EventRegistry:RegisterFrameEventAndCallback("PLAYER_ENTERING_WORLD",
    function(ownerID, isLogin, isReload)
        if isLogin then
            print("Welcome to Azeroth!")
        end
    end
)

-- Register for custom Blizzard UI callbacks
EventRegistry:RegisterCallback("MountJournal.OnShow", function(ownerID)
    print("Mount journal opened")
end)
```

### With Owner for Cleanup

```lua
local myAddon = {}

-- Register with an owner reference
EventRegistry:RegisterFrameEventAndCallback("PLAYER_REGEN_ENABLED",
    function(ownerID)
        -- Do something when combat ends
    end,
    myAddon  -- owner handle
)

-- Later, cleanly unregister:
EventRegistry:UnregisterCallback("PLAYER_REGEN_ENABLED", myAddon)
```

### Passing Extra Arguments

```lua
-- Extra arguments are appended after event arguments
EventRegistry:RegisterFrameEventAndCallback("UNIT_HEALTH",
    function(ownerID, unit, myExtraArg)
        -- unit comes from the event
        -- myExtraArg is "hello" (passed at registration)
    end,
    nil,       -- owner (nil = auto-assigned)
    "hello"    -- extra arg
)
```

!!! tip "When to use EventRegistry vs. traditional frames"
    Use `EventRegistry` for lightweight event listeners that don't need a visible frame. Use the traditional `CreateFrame()` + `RegisterEvent()` pattern when you need the frame for UI display anyway, or when you need `RegisterUnitEvent()` for unit-specific filtering (which `EventRegistry` doesn't support directly).

---

## Pattern 7: Addon Compartment (Minimap Replacement)

The **Addon Compartment** is Blizzard's official replacement for LibDBIcon minimap buttons. It provides a unified dropdown accessible from a single minimap button, keeping the minimap clean.

### TOC-Based Registration (Simplest)

```toc
## Interface: 120001
## Title: My Cool Addon
## Notes: Does cool things
## Author: YourName
## Version: 1.0.0
## SavedVariables: MyCoolAddonDB
## AddonCompartmentFunc: MyCoolAddon_OnCompartmentClick
## AddonCompartmentFuncOnEnter: MyCoolAddon_OnCompartmentEnter
## AddonCompartmentFuncOnLeave: MyCoolAddon_OnCompartmentLeave

MyCoolAddon.lua
```

```lua
-- In MyCoolAddon.lua:
local addonName, ns = ...

-- These must be GLOBAL functions (the names match the TOC fields)
function MyCoolAddon_OnCompartmentClick(addonName, buttonInfo)
    -- buttonInfo contains which mouse button was used
    if buttonInfo.buttonName == "LeftButton" then
        -- Toggle your main window
        ns:ToggleMainWindow()
    elseif buttonInfo.buttonName == "RightButton" then
        -- Open settings
        Settings.OpenToCategory(addonName)
    end
end

function MyCoolAddon_OnCompartmentEnter(addonName, menuButtonFrame)
    -- Show tooltip on hover
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:AddLine("|cff00ff00My Cool Addon|r")
    GameTooltip:AddLine("Left-click: Toggle window", 1, 1, 1)
    GameTooltip:AddLine("Right-click: Settings", 1, 1, 1)
    GameTooltip:Show()
end

function MyCoolAddon_OnCompartmentLeave(addonName, menuButtonFrame)
    GameTooltip:Hide()
end
```

### Programmatic Registration (More Control)

```lua
-- Register via code instead of TOC fields
-- Useful for addons with dynamic compartment behavior
local addonName, ns = ...

local function OnCompartmentClick(_, buttonInfo)
    if buttonInfo.buttonName == "LeftButton" then
        ns:ToggleMainWindow()
    end
end

local function OnCompartmentEnter(_, frame)
    GameTooltip:SetOwner(frame, "ANCHOR_LEFT")
    GameTooltip:AddLine("My Addon")
    GameTooltip:AddLine(format("Version: %s",
        C_AddOns.GetAddOnMetadata(addonName, "Version")), 0.8, 0.8, 0.8)
    GameTooltip:Show()
end

local function OnCompartmentLeave()
    GameTooltip:Hide()
end

-- Register programmatically (available since 10.1)
if AddonCompartmentFrame and AddonCompartmentFrame.RegisterAddon then
    AddonCompartmentFrame:RegisterAddon({
        text = "My Addon",
        icon = "Interface\\Icons\\INV_Misc_Gear_01",
        notCheckable = true,
        func = OnCompartmentClick,
        funcOnEnter = OnCompartmentEnter,
        funcOnLeave = OnCompartmentLeave,
    })
end
```

!!! tip "Dual support for LibDBIcon users"
    If you're migrating from LibDBIcon, you can support both systems during the transition:
    ```lua
    -- Check if LibDBIcon is available, fall back to compartment
    if LibStub and LibStub("LibDBIcon-1.0", true) then
        -- Use LibDBIcon (still works in Midnight)
    else
        -- Use Addon Compartment
    end
    ```
    That said, the Addon Compartment is the officially supported path going forward.

---

## Pattern 8: Settings Panel Integration

The modern Settings API (introduced in 10.0, required for Midnight) replaces the old `InterfaceOptions_AddCategory` pattern.

### Complete Settings Panel

```lua
local addonName, ns = ...

-- Default settings
local defaults = {
    showHealthText = true,
    barTexture = "Default",
    fontSize = 12,
    opacity = 1.0,
    classColors = true,
}

-- Initialize saved variables
local function InitializeDB()
    if not MyCoolAddonDB then
        MyCoolAddonDB = CopyTable(defaults)
    end
    -- Fill in any missing defaults (new settings added in updates)
    for key, value in pairs(defaults) do
        if MyCoolAddonDB[key] == nil then
            MyCoolAddonDB[key] = value
        end
    end
    ns.db = MyCoolAddonDB
end

-- Build the settings panel
local function CreateSettingsPanel()
    -- Create the category
    local category = Settings.RegisterVerticalLayoutCategory(addonName)

    -- Header text
    local function FormatDescription(text)
        return CreateColor(0.8, 0.8, 0.8):WrapTextInColorCode(text)
    end
    category.layoutInfo = { description = FormatDescription(
        "Configure appearance and behavior settings."
    )}

    -- Toggle: Show Health Text
    do
        local variable = "showHealthText"
        local name = "Show Health Text"
        local tooltip = "Display numerical health values on unit frames."

        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable, ns.db,
            type(true), name, defaults[variable]
        )
        setting:SetValueChangedCallback(function(_, val)
            ns.db[variable] = val
            ns:RefreshUI()
        end)
        Settings.CreateCheckbox(category, setting, tooltip)
    end

    -- Toggle: Use Class Colors
    do
        local variable = "classColors"
        local name = "Use Class Colors"
        local tooltip = "Color health bars by player class."

        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable, ns.db,
            type(true), name, defaults[variable]
        )
        setting:SetValueChangedCallback(function(_, val)
            ns.db[variable] = val
            ns:RefreshUI()
        end)
        Settings.CreateCheckbox(category, setting, tooltip)
    end

    -- Slider: Font Size
    do
        local variable = "fontSize"
        local name = "Font Size"
        local tooltip = "Size of text on unit frames."

        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable, ns.db,
            type(1.0), name, defaults[variable]
        )
        setting:SetValueChangedCallback(function(_, val)
            ns.db[variable] = val
            ns:RefreshUI()
        end)

        local options = Settings.CreateSliderOptions(8, 24, 1)
        options:SetLabelFormatter(MinimalSliderWithSteppersMixin
            .Label.Right, function(val)
            return format("%d pt", val)
        end)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    -- Slider: Opacity
    do
        local variable = "opacity"
        local name = "Frame Opacity"
        local tooltip = "Transparency of addon frames."

        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable, ns.db,
            type(1.0), name, defaults[variable]
        )
        setting:SetValueChangedCallback(function(_, val)
            ns.db[variable] = val
            ns:RefreshUI()
        end)

        local options = Settings.CreateSliderOptions(0.1, 1.0, 0.05)
        options:SetLabelFormatter(MinimalSliderWithSteppersMixin
            .Label.Right, function(val)
            return format("%d%%", val * 100)
        end)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    -- Dropdown: Bar Texture
    do
        local variable = "barTexture"
        local name = "Bar Texture"
        local tooltip = "Select the texture used for status bars."

        local function GetOptions()
            local container = Settings.CreateControlTextContainer()
            container:Add("Default", "Default (Blizzard)")
            container:Add("Flat", "Flat")
            container:Add("Smooth", "Smooth")
            container:Add("Striped", "Striped")
            return container:GetData()
        end

        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable, ns.db,
            type(""), name, defaults[variable]
        )
        setting:SetValueChangedCallback(function(_, val)
            ns.db[variable] = val
            ns:RefreshUI()
        end)
        Settings.CreateDropdown(category, setting, GetOptions, tooltip)
    end

    -- Register the category
    Settings.RegisterAddOnCategory(category)
    ns.settingsCategory = category
end

-- Initialize on load
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, loadedAddon)
    if loadedAddon == addonName then
        self:UnregisterEvent("ADDON_LOADED")
        InitializeDB()
        CreateSettingsPanel()
    end
end)

-- Slash command to open settings
SLASH_MYCOOLADDON1 = "/mca"
SlashCmdList["MYCOOLADDON"] = function(msg)
    Settings.OpenToCategory(ns.settingsCategory:GetID())
end
```

---

## Pattern 9: Secure Button Templates in Midnight

Secure buttons remain the **only** way for addons to trigger protected actions (casting spells, targeting units, using items) through player clicks. The rules haven't changed, but enforcement is stricter.

### Basic Secure Action Button

```lua
local addonName, ns = ...

-- IMPORTANT: Create and configure secure frames OUTSIDE of combat
-- InCombatLockdown() must be false when you call SetAttribute()

local function CreateSecureSpellButton(spellName, parent)
    local btn = CreateFrame("Button", nil, parent or UIParent,
        "SecureActionButtonTemplate")
    btn:SetSize(40, 40)

    -- Configure the secure action
    btn:SetAttribute("type", "spell")
    btn:SetAttribute("spell", spellName)

    -- Visual setup (non-secure, can be modified anytime)
    local icon = btn:CreateTexture(nil, "BACKGROUND")
    icon:SetAllPoints()

    -- Try to get the spell icon
    local info = C_Spell.GetSpellInfo(spellName)
    if info then
        icon:SetTexture(info.iconID)
    else
        icon:SetTexture("Interface\\Icons\\INV_Misc_QuestionMark")
    end

    -- Highlight on mouseover
    local highlight = btn:CreateTexture(nil, "HIGHLIGHT")
    highlight:SetAllPoints()
    highlight:SetColorTexture(1, 1, 1, 0.2)

    -- Cooldown overlay
    local cooldown = CreateFrame("Cooldown", nil, btn, "CooldownFrameTemplate")
    cooldown:SetAllPoints()
    btn.cooldown = cooldown

    -- Push texture (visual feedback on click)
    local pushed = btn:CreateTexture(nil, "OVERLAY")
    pushed:SetAllPoints()
    pushed:SetColorTexture(1, 1, 1, 0.1)
    btn:SetPushedTexture(pushed)

    return btn
end

-- Usage:
local fireballBtn = CreateSecureSpellButton("Fireball")
fireballBtn:SetPoint("CENTER", UIParent, "CENTER", 0, -100)
-- When the player CLICKS this button, it casts Fireball
```

### Secure Button Type Reference

| Type | Description | Key Attributes |
|------|-------------|----------------|
| `spell` | Cast a spell | `spell`, `unit` |
| `item` | Use an item | `item` |
| `macro` | Run a macro | `macrotext` |
| `action` | Execute action bar slot | `action` |
| `cancelaura` | Cancel a buff | `spell`, `index` |
| `stop` | Stop casting | (none) |
| `target` | Target a unit | `unit` |
| `focus` | Set focus | `unit` |
| `assist` | Assist a unit | `unit` |

!!! warning "macrotext may be restricted in Midnight"
    An exploit allowing arbitrary function calls via creative `SecureActionButtonTemplate` attribute usage was patched in 12.0. Reports indicate `macrotext` execution may be restricted in certain contexts. Test your secure buttons on live — prefer `type = "spell"` with modifier attributes over `macrotext` where possible.

### Secure Button with Multiple Actions

```lua
-- A button that does different things based on modifier keys
local multiBtn = CreateFrame("Button", nil, UIParent,
    "SecureActionButtonTemplate")
multiBtn:SetSize(40, 40)
multiBtn:RegisterForClicks("AnyUp", "AnyDown")

-- Default left click: cast Fireball
multiBtn:SetAttribute("type1", "spell")
multiBtn:SetAttribute("spell1", "Fireball")

-- Shift+left click: cast Pyroblast
multiBtn:SetAttribute("shift-type1", "spell")
multiBtn:SetAttribute("shift-spell1", "Pyroblast")

-- Ctrl+left click: cast Flamestrike (AoE)
multiBtn:SetAttribute("ctrl-type1", "spell")
multiBtn:SetAttribute("ctrl-spell1", "Flamestrike")

-- Right-click: target self
multiBtn:SetAttribute("type2", "target")
multiBtn:SetAttribute("target-slot2", "player")

-- Alt+right-click: use item (e.g., healthstone)
multiBtn:SetAttribute("alt-type2", "item")
multiBtn:SetAttribute("alt-item2", "Healthstone")
```

### Pre-Combat Setup Pattern

```lua
-- The key pattern: configure ALL secure attributes before combat,
-- then let them work during combat without modification

local addonName, ns = ...
ns.secureButtons = {}

function ns:SetupSecureButtons()
    if InCombatLockdown() then
        -- Cannot modify secure frames in combat!
        -- Queue for after combat
        ns.needsSecureSetup = true
        return
    end

    -- Create/update secure buttons based on current spec
    local specID = GetSpecializationInfo(GetSpecialization())
    local buttons = ns:GetButtonConfigForSpec(specID)

    for i, config in ipairs(buttons) do
        local btn = ns.secureButtons[i]
        if not btn then
            btn = CreateFrame("Button", addonName .. "Btn" .. i,
                UIParent, "SecureActionButtonTemplate")
            btn:SetSize(40, 40)
            ns.secureButtons[i] = btn
        end

        -- Set ALL attributes now (before combat)
        btn:SetAttribute("type", config.type)
        btn:SetAttribute("spell", config.spell)
        btn:SetPoint(unpack(config.position))
        btn:Show()
    end

    -- Hide unused buttons
    for i = #buttons + 1, #ns.secureButtons do
        ns.secureButtons[i]:Hide()
    end
end

-- Rebuild on spec change (always fires out of combat)
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_SPECIALIZATION_CHANGED")
frame:RegisterEvent("PLAYER_REGEN_ENABLED")
frame:RegisterEvent("PLAYER_LOGIN")

frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_ENABLED" and ns.needsSecureSetup then
        ns.needsSecureSetup = false
        ns:SetupSecureButtons()
    elseif event == "PLAYER_SPECIALIZATION_CHANGED"
        or event == "PLAYER_LOGIN" then
        ns:SetupSecureButtons()
    end
end)
```

!!! danger "The golden rule of secure frames"
    Every `SetAttribute()` call on a secure frame **must** happen outside of combat (`InCombatLockdown() == false`). There are no exceptions, no workarounds, and no clever hacks. Plan your secure frame configuration to happen during initialization, spec changes, and `PLAYER_REGEN_ENABLED`.

---

## Pattern 10: Addon Communication with Compression

Addon messaging in Midnight has stricter throttling, and messages during active encounters are restricted. Efficient data encoding is essential.

### Using LibDeflate for Compression

**LibDeflate** is the community standard for addon data compression — a pure Lua DEFLATE/zlib implementation with 3-4x compression ratios:

```lua
-- LibDeflate usage (include as an embedded library)
local LibDeflate = LibStub("LibDeflate")

-- Compress data for addon channel transmission
local function CompressForChannel(data)
    local serialized = ns:Serialize(data)
    local compressed = LibDeflate:CompressDeflate(serialized, { level = 5 })
    -- Encode for safe transmission over addon channels
    return LibDeflate:EncodeForWoWAddonChannel(compressed)
end

-- Decompress received data
local function DecompressFromChannel(encoded)
    local decoded = LibDeflate:DecodeForWoWAddonChannel(encoded)
    if not decoded then return nil end
    local decompressed = LibDeflate:DecompressDeflate(decoded)
    if not decompressed then return nil end
    return ns:Deserialize(decompressed)
end
```

!!! info "Why LibDeflate over C_EncodingUtil"
    `C_EncodingUtil` provides Base64 encoding but not compression. LibDeflate provides both compression (DEFLATE algorithm) and channel-safe encoding in one package, making it the standard for addon communication.

### Complete Addon Communication System

```lua
local addonName, ns = ...

local PREFIX = addonName
local CHUNK_SIZE = 240  -- Leave room for headers in 255-byte limit
local THROTTLE_DELAY = 0.12  -- Seconds between messages to avoid throttle

-- Register prefix
C_ChatInfo.RegisterAddonMessagePrefix(PREFIX)

------------------------------------------------------------
-- Sending
------------------------------------------------------------

-- Simple serialization (for real addons, use LibSerialize)
function ns:Serialize(data)
    if type(data) == "table" then
        local parts = {}
        for k, v in pairs(data) do
            tinsert(parts, tostring(k) .. "=" .. tostring(v))
        end
        return table.concat(parts, ";")
    end
    return tostring(data)
end

-- Send a message, handling chunking and encounter restrictions
function ns:SendMessage(data, channel)
    -- Check encounter restrictions
    if IsEncounterInProgress() then
        -- Queue for after encounter
        ns.pendingMessages = ns.pendingMessages or {}
        tinsert(ns.pendingMessages, { data = data, channel = channel })
        return false
    end

    local payload = ns:Serialize(data)

    -- Small enough for a single message?
    if #payload <= CHUNK_SIZE then
        C_ChatInfo.SendAddonMessage(PREFIX, "S:" .. payload, channel)
        return true
    end

    -- Chunk it
    local chunks = {}
    for i = 1, #payload, CHUNK_SIZE do
        tinsert(chunks, payload:sub(i, i + CHUNK_SIZE - 1))
    end

    local totalChunks = #chunks
    local messageID = format("%X", math.random(0, 0xFFFF))

    for i, chunk in ipairs(chunks) do
        C_Timer.After((i - 1) * THROTTLE_DELAY, function()
            local header = format("C:%s:%d:%d:", messageID, i, totalChunks)
            C_ChatInfo.SendAddonMessage(PREFIX, header .. chunk, channel)
        end)
    end

    return true
end

------------------------------------------------------------
-- Receiving
------------------------------------------------------------

ns.incomingChunks = {}  -- [senderID] = { chunks = {}, total = N }

local function ProcessMessage(payload, sender)
    -- Your message handling logic here
    -- payload is the deserialized data string
end

local receiver = CreateFrame("Frame")
receiver:RegisterEvent("CHAT_MSG_ADDON")
receiver:RegisterEvent("ENCOUNTER_END")

receiver:SetScript("OnEvent", function(self, event, ...)
    if event == "CHAT_MSG_ADDON" then
        local prefix, message, channel, sender = ...
        if prefix ~= PREFIX then return end

        local msgType = message:sub(1, 2)

        if msgType == "S:" then
            -- Single message
            ProcessMessage(message:sub(3), sender)

        elseif msgType == "C:" then
            -- Chunked message
            local _, id, idx, total, data = strsplit(":", message, 5)
            idx = tonumber(idx)
            total = tonumber(total)

            local key = sender .. ":" .. id
            if not ns.incomingChunks[key] then
                ns.incomingChunks[key] = { chunks = {}, total = total }
            end

            local assembly = ns.incomingChunks[key]
            assembly.chunks[idx] = data

            -- Check if we have all chunks
            local complete = true
            for i = 1, total do
                if not assembly.chunks[i] then
                    complete = false
                    break
                end
            end

            if complete then
                local fullPayload = table.concat(assembly.chunks)
                ns.incomingChunks[key] = nil
                ProcessMessage(fullPayload, sender)
            end
        end

    elseif event == "ENCOUNTER_END" then
        -- Flush pending messages
        if ns.pendingMessages then
            for _, msg in ipairs(ns.pendingMessages) do
                ns:SendMessage(msg.data, msg.channel)
            end
            wipe(ns.pendingMessages)
        end
    end
end)

-- Clean up stale incomplete chunks periodically
C_Timer.NewTicker(60, function()
    wipe(ns.incomingChunks)
end)
```

!!! warning "Encounter restrictions"
    `C_ChatInfo.SendAddonMessage()` is throttled or blocked during active encounters in instances. Always check `IsEncounterInProgress()` and queue messages for delivery after `ENCOUNTER_END`. See [Midnight: Addon Messaging Restrictions](midnight.md#addon-messaging-restrictions-in-instances).

---

## Pattern 11: Edit Mode Integration

Blizzard's Edit Mode lets players reposition UI elements with a drag-and-drop interface. Making your addon's frames respect Edit Mode provides a polished, integrated experience.

### Making Frames Draggable (Non-Edit-Mode)

```lua
-- Basic frame dragging with position saving
local addonName, ns = ...

local function MakeMovable(frame, savedVarsKey)
    frame:SetMovable(true)
    frame:EnableMouse(true)
    frame:RegisterForDrag("LeftButton")
    frame:SetClampedToScreen(true)

    frame:SetScript("OnDragStart", function(self)
        if not InCombatLockdown() then
            self:StartMoving()
        end
    end)

    frame:SetScript("OnDragStop", function(self)
        self:StopMovingOrSizing()
        -- Save position
        local point, _, relPoint, x, y = self:GetPoint()
        if ns.db then
            ns.db[savedVarsKey] = { point, relPoint, x, y }
        end
    end)

    -- Restore saved position
    if ns.db and ns.db[savedVarsKey] then
        local pos = ns.db[savedVarsKey]
        frame:ClearAllPoints()
        frame:SetPoint(pos[1], UIParent, pos[2], pos[3], pos[4])
    end
end

-- Usage:
MakeMovable(myFrame, "mainFramePosition")
```

### Responding to Edit Mode State

```lua
-- Detect when Edit Mode is active and show anchor indicators
local addonName, ns = ...

local function IsEditModeActive()
    return EditModeManagerFrame and EditModeManagerFrame:IsShown()
end

-- Show positioning helpers when Edit Mode is active
local function OnEditModeChanged()
    local isActive = IsEditModeActive()

    for _, frame in ipairs(ns.movableFrames or {}) do
        if frame.__editModeIndicator then
            frame.__editModeIndicator:SetShown(isActive)
        end
    end
end

-- Add an Edit Mode indicator to a frame
local function AddEditModeIndicator(frame)
    local indicator = frame:CreateTexture(nil, "OVERLAY")
    indicator:SetAllPoints()
    indicator:SetColorTexture(0.2, 0.4, 1.0, 0.15)
    indicator:Hide()

    local label = frame:CreateFontString(nil, "OVERLAY", "GameFontNormalSmall")
    label:SetPoint("CENTER")
    label:SetText("|cff6699ffDrag to move|r")
    label:SetParent(indicator)

    frame.__editModeIndicator = indicator

    ns.movableFrames = ns.movableFrames or {}
    tinsert(ns.movableFrames, frame)
end

-- Watch for Edit Mode changes
if EditModeManagerFrame then
    hooksecurefunc(EditModeManagerFrame, "Show", OnEditModeChanged)
    hooksecurefunc(EditModeManagerFrame, "Hide", OnEditModeChanged)
end
```

---

## Pattern 12: The Modern C_ Namespace API

Midnight removed many legacy global functions. Here's a quick-reference for the modern replacements you'll use constantly.

### Spell API Migration

```lua
------------------------------------------------------------
-- C_Spell (replaces GetSpellInfo, GetSpellCooldown, etc.)
------------------------------------------------------------

-- Get spell info
local info = C_Spell.GetSpellInfo(spellID)
-- Returns: { name, iconID, castTime, minRange, maxRange,
--            spellID, originalIconID }

-- Get spell cooldown (whitelisted spells only in combat)
local cd = C_Spell.GetSpellCooldown(spellID)
-- Returns: { startTime, duration, isEnabled, modRate }

-- Get spell charges
local charges = C_Spell.GetSpellCharges(spellID)
-- Returns: { currentCharges, maxCharges, cooldownStartTime,
--            cooldownDuration, chargeModRate }

-- Check if spell is known
local isKnown = C_Spell.IsSpellDataCached(spellID)

-- Get spell description
local desc = C_Spell.GetSpellDescription(spellID)

-- Request spell data load (for spells not yet cached)
C_Spell.RequestLoadSpellData(spellID)
```

### Addon Metadata API

```lua
------------------------------------------------------------
-- C_AddOns (replaces GetAddOnMetadata, etc.)
------------------------------------------------------------

-- Get addon metadata
local version = C_AddOns.GetAddOnMetadata(addonName, "Version")
local title = C_AddOns.GetAddOnMetadata(addonName, "Title")
local notes = C_AddOns.GetAddOnMetadata(addonName, "Notes")
local author = C_AddOns.GetAddOnMetadata(addonName, "Author")

-- Check addon state
local isLoaded = C_AddOns.IsAddOnLoaded(addonName)
local isLoadable = C_AddOns.IsAddOnLoadable(addonName)

-- Get number of addons
local numAddons = C_AddOns.GetNumAddOns()

-- Load on demand
C_AddOns.LoadAddOn("SomeOptionalAddon")
```

### Unit API (What's Still Available)

```lua
------------------------------------------------------------
-- Unit functions — still available, but values may be
-- secret during combat for health/power data
------------------------------------------------------------

-- Always available (not secret):
local name, realm = UnitName(unit)
local _, class, classID = UnitClass(unit)
local level = UnitLevel(unit)
local race, raceEn = UnitRace(unit)
local sex = UnitSex(unit)
local isPlayer = UnitIsPlayer(unit)
local isDead = UnitIsDead(unit)
local isGhost = UnitIsGhost(unit)
local isConnected = UnitIsConnected(unit)
local role = UnitGroupRolesAssigned(unit)
local guid = UnitGUID(unit)
local exists = UnitExists(unit)
local isFriend = UnitIsFriend("player", unit)
local reaction = UnitReaction("player", unit)

-- Available but may return secret values in combat:
local health = UnitHealth(unit)
local maxHealth = UnitHealthMax(unit)
local power = UnitPower(unit, powerType)
local maxPower = UnitPowerMax(unit, powerType)
```

### Timer Utilities

```lua
------------------------------------------------------------
-- C_Timer — preferred over OnUpdate for delayed actions
------------------------------------------------------------

-- One-shot timer
C_Timer.After(2.0, function()
    print("2 seconds have passed!")
end)

-- Repeating timer
local ticker = C_Timer.NewTicker(1.0, function()
    print("Every second!")
end)
-- Cancel it:
ticker:Cancel()

-- Repeating with a limit
local limitedTicker = C_Timer.NewTicker(0.5, function()
    print("Tick!")
end, 10)  -- fires 10 times, then stops
```

---

## Pattern 13: Complete Midnight Addon Template

Putting it all together — a complete, minimal addon skeleton that follows all Midnight best practices.

### File Structure

```
MyMidnightAddon/
├── MyMidnightAddon.toc
├── Core.lua
├── Settings.lua
└── Textures/
    └── StatusBar.tga
```

### MyMidnightAddon.toc

```toc
## Interface: 120001
## Title: My Midnight Addon
## Notes: A template addon built for Patch 12.0+ best practices
## Author: YourName
## Version: 1.0.0
## SavedVariables: MyMidnightAddonDB
## AddonCompartmentFunc: MyMidnightAddon_OnClick
## AddonCompartmentFuncOnEnter: MyMidnightAddon_OnEnter
## AddonCompartmentFuncOnLeave: MyMidnightAddon_OnLeave
## IconTexture: Interface\Icons\INV_Misc_Gear_01

Core.lua
Settings.lua
```

### Core.lua

```lua
local addonName, ns = ...

-- Upvalue frequently used globals
local CreateFrame = CreateFrame
local hooksecurefunc = hooksecurefunc
local InCombatLockdown = InCombatLockdown
local UnitExists = UnitExists
local UnitName = UnitName
local UnitClass = UnitClass
local C_Timer = C_Timer

------------------------------------------------------------
-- Addon Compartment (global functions for TOC fields)
------------------------------------------------------------

function MyMidnightAddon_OnClick(_, buttonInfo)
    if buttonInfo.buttonName == "LeftButton" then
        ns:ToggleMainFrame()
    elseif buttonInfo.buttonName == "RightButton" then
        if ns.settingsCategory then
            Settings.OpenToCategory(ns.settingsCategory:GetID())
        end
    end
end

function MyMidnightAddon_OnEnter(_, frame)
    GameTooltip:SetOwner(frame, "ANCHOR_LEFT")
    GameTooltip:AddLine("|cff00ccffMy Midnight Addon|r")
    GameTooltip:AddLine("Left-click: Toggle", 0.8, 0.8, 0.8)
    GameTooltip:AddLine("Right-click: Settings", 0.8, 0.8, 0.8)
    GameTooltip:Show()
end

function MyMidnightAddon_OnLeave()
    GameTooltip:Hide()
end

------------------------------------------------------------
-- Initialization
------------------------------------------------------------

local defaults = {
    enabled = true,
    locked = false,
    position = nil,
}

local function Initialize()
    -- Set up saved variables
    if not MyMidnightAddonDB then
        MyMidnightAddonDB = CopyTable(defaults)
    end
    for k, v in pairs(defaults) do
        if MyMidnightAddonDB[k] == nil then
            MyMidnightAddonDB[k] = v
        end
    end
    ns.db = MyMidnightAddonDB

    -- Set up UI
    ns:CreateMainFrame()

    -- Hook Blizzard frames (the Midnight way)
    ns:SetupSkinning()

    print(format("|cff00ccff%s|r v%s loaded. Type /mma to toggle.",
        addonName, C_AddOns.GetAddOnMetadata(addonName, "Version")))
end

------------------------------------------------------------
-- Event Handler
------------------------------------------------------------

local eventFrame = CreateFrame("Frame")
local events = {}

function events:ADDON_LOADED(loadedAddon)
    if loadedAddon == addonName then
        eventFrame:UnregisterEvent("ADDON_LOADED")
        Initialize()
    end
end

function events:PLAYER_REGEN_ENABLED()
    -- Process anything queued during combat
    if ns.pendingLayout then
        ns.pendingLayout = false
        ns:UpdateLayout()
    end
end

for event in pairs(events) do
    eventFrame:RegisterEvent(event)
end

eventFrame:SetScript("OnEvent", function(self, event, ...)
    events[event](self, ...)
end)

------------------------------------------------------------
-- Main Frame
------------------------------------------------------------

function ns:CreateMainFrame()
    local f = CreateFrame("Frame", "MyMidnightAddonFrame", UIParent,
        "BackdropTemplate")
    f:SetSize(250, 150)
    f:SetPoint("CENTER")
    f:SetBackdrop({
        bgFile = "Interface\\Buttons\\WHITE8X8",
        edgeFile = "Interface\\Buttons\\WHITE8X8",
        edgeSize = 1,
    })
    f:SetBackdropColor(0.05, 0.05, 0.1, 0.9)
    f:SetBackdropBorderColor(0.3, 0.3, 0.5, 1)

    -- Make movable (when not locked)
    f:SetMovable(true)
    f:EnableMouse(true)
    f:SetClampedToScreen(true)
    f:RegisterForDrag("LeftButton")

    f:SetScript("OnDragStart", function(self)
        if not InCombatLockdown() and not ns.db.locked then
            self:StartMoving()
        end
    end)

    f:SetScript("OnDragStop", function(self)
        self:StopMovingOrSizing()
        local point, _, relPoint, x, y = self:GetPoint()
        ns.db.position = { point, relPoint, x, y }
    end)

    -- Restore position
    if ns.db.position then
        f:ClearAllPoints()
        local p = ns.db.position
        f:SetPoint(p[1], UIParent, p[2], p[3], p[4])
    end

    -- Title
    local title = f:CreateFontString(nil, "OVERLAY", "GameFontNormal")
    title:SetPoint("TOP", 0, -8)
    title:SetText("|cff00ccffMy Midnight Addon|r")

    -- Content
    local content = f:CreateFontString(nil, "OVERLAY",
        "GameFontHighlightSmall")
    content:SetPoint("CENTER", 0, -10)
    content:SetWidth(f:GetWidth() - 20)
    content:SetJustifyH("CENTER")
    content:SetText("This addon follows Midnight 12.0\ncoding patterns.")

    -- Close button
    local close = CreateFrame("Button", nil, f, "UIPanelCloseButton")
    close:SetPoint("TOPRIGHT", -2, -2)

    ns.mainFrame = f

    if not ns.db.enabled then
        f:Hide()
    end
end

function ns:ToggleMainFrame()
    if ns.mainFrame:IsShown() then
        ns.mainFrame:Hide()
        ns.db.enabled = false
    else
        ns.mainFrame:Show()
        ns.db.enabled = true
    end
end

function ns:UpdateLayout()
    -- Safe to modify frames here (called outside combat)
    if ns.mainFrame then
        ns.mainFrame:SetAlpha(ns.db.opacity or 1.0)
    end
end

------------------------------------------------------------
-- Blizzard Frame Skinning
------------------------------------------------------------

function ns:SetupSkinning()
    -- Example: restyle compact unit frame health bars
    hooksecurefunc("CompactUnitFrame_UpdateHealth", function(frame)
        if frame:IsForbidden() then return end
        if frame.healthBar then
            -- Subtle visual change: add a gradient overlay
            if not frame.__myOverlay then
                local overlay = frame.healthBar:CreateTexture(nil, "ARTWORK")
                overlay:SetAllPoints()
                overlay:SetTexture("Interface\\Buttons\\WHITE8X8")
                overlay:SetGradient("VERTICAL",
                    CreateColor(0, 0, 0, 0.3),
                    CreateColor(0, 0, 0, 0))
                frame.__myOverlay = overlay
            end
        end
    end)
end

------------------------------------------------------------
-- Slash Command
------------------------------------------------------------

SLASH_MYMIDNIGHTADDON1 = "/mma"
SlashCmdList["MYMIDNIGHTADDON"] = function(msg)
    msg = strlower(strtrim(msg))

    if msg == "settings" or msg == "config" then
        if ns.settingsCategory then
            Settings.OpenToCategory(ns.settingsCategory:GetID())
        end
    elseif msg == "lock" then
        ns.db.locked = not ns.db.locked
        print(format("|cff00ccff%s|r: Frame %s.",
            addonName, ns.db.locked and "locked" or "unlocked"))
    else
        ns:ToggleMainFrame()
    end
end
```

!!! tip "Use this as your starting point"
    This template includes all the Midnight best practices: addon compartment integration, Settings API, saved variables with defaults, event dispatch, combat lockdown safety, frame skinning via hooks, and draggable frames with position persistence. Copy it, rename it, and start building.

---

## Quick Reference: What Changed

| Task | Pre-Midnight | Midnight 12.0+ |
|------|-------------|-----------------|
| Get spell info | `GetSpellInfo(id)` | `C_Spell.GetSpellInfo(id)` |
| Get spell cooldown | `GetSpellCooldown(id)` | `C_Spell.GetSpellCooldown(id)` |
| Get addon metadata | `GetAddOnMetadata(name, field)` | `C_AddOns.GetAddOnMetadata(name, field)` |
| Parse combat log | `COMBAT_LOG_EVENT_UNFILTERED` + `CombatLogGetCurrentEventInfo()` | **Removed.** Use encounter events, unit events, or Blizzard's native UI. |
| Build unit frames | Create custom frames, read `UnitHealth()` freely | Skin Blizzard's `CompactUnitFrame` via `hooksecurefunc()` |
| Track boss abilities | Parse CLEU for spell casts | Use `ENCOUNTER_START`/`ENCOUNTER_END`, Blizzard's boss timeline |
| Minimap button | LibDBIcon-1.0 | Addon Compartment (TOC fields or `AddonCompartmentFrame:RegisterAddon()`) |
| Settings panel | `InterfaceOptions_AddCategory()` | `Settings.RegisterAddOnCategory()` |
| Read combat data in Lua | Direct API calls | **Secret Values** — display only, no branching |
| Compress messages | LibDeflate | LibDeflate (still the standard) + `C_EncodingUtil` for Base64 |
| Track spell cooldowns | Read freely in combat | Only for **whitelisted** spells; others are secret |
| Health bar coloring | `UnitHealth()` + manual math | `UnitHealthPercent()` + `ColorCurve:Evaluate()` (secret-safe) |
| Cooldown/duration timers | Raw timestamp math | `C_DurationUtil.CreateDuration()` + `StatusBar:SetTimerDuration()` |
| Heal prediction | `UnitGetIncomingHeals()` | `CreateUnitHealPredictionCalculator()` + `UnitGetDetailedHealPrediction()` |
| Check if value is secret | N/A | `issecretvalue(value)` |
| Event handling (no frame) | `CreateFrame()` + `RegisterEvent()` | `EventRegistry:RegisterFrameEventAndCallback()` |

---

## Midnight-Specific Gotchas

A quick reference for breaking changes that will bite you if you're porting pre-Midnight code:

!!! danger "GameTooltipTemplate no longer inherits BackdropTemplate"
    In 12.0, `SharedTooltipTemplate` and `GameTooltipTemplate` **no longer inherit BackdropTemplate**. If your addon creates tooltip frames, you must explicitly include `BackdropTemplate` in the template string:
    ```lua
    -- Pre-Midnight (worked):
    local tt = CreateFrame("GameTooltip", "MyTooltip", UIParent,
        "GameTooltipTemplate")
    tt:SetBackdrop(...)  -- ERRORS in 12.0!

    -- Midnight (correct):
    local tt = CreateFrame("GameTooltip", "MyTooltip", UIParent,
        "GameTooltipTemplate, BackdropTemplate")
    tt:SetBackdrop(...)  -- works
    ```

!!! danger "Unit identifiers are secret in instances"
    In Midnight, creature unit **names, GUIDs, and IDs** are made secret while in an instance — not just during combat. This means `UnitName(unit)` on enemy mobs returns a secret value inside dungeons and raids.

!!! warning "`secretunwrap` was removed"
    The `secretunwrap` function was removed from the global table in 12.0. Code that relied on it to extract secret values will error.

---

## Reading Blizzard's Source Code

The best way to learn Midnight patterns is to read how Blizzard implements their own UI. Two essential resources:

- **[Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source)** — Community-maintained mirror of Blizzard's UI source. The `live` branch tracks current retail (12.0+). Browse `Interface/` for FrameXML.
- **[Townlong Yak FrameXML](https://www.townlong-yak.com/framexml/65294)** — Build-specific browsable source. Build 65294 is 12.0.0.
- **In-game extraction:** Run `/run ExportInterfaceFiles("code")` to export the current client's FrameXML to your `_retail_/BlizzardInterfaceCode/` directory.

---

## Further Reading

- [Midnight (Patch 12.0)](midnight.md) — Policy context, Secret Values details, full migration checklist
- [Security Model](security.md) — Taint system, protected functions, combat lockdown patterns
- [Events System](events.md) — Available events and registration patterns
- [Frames & Widgets](frames-widgets.md) — Frame type hierarchy and widget methods
- [Lua API Reference](lua-api.md) — Complete API surface
- [Warcraft Wiki — Patch 12.0.0/API changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes)
- [Warcraft Wiki — Secret Values](https://warcraft.wiki.gg/wiki/Secret_Values)
- [Warcraft Wiki — Widget API](https://warcraft.wiki.gg/wiki/Widget_API)
- [Warcraft Wiki — Settings API](https://warcraft.wiki.gg/wiki/Settings_API)
- [Warcraft Wiki — EventRegistry](https://warcraft.wiki.gg/wiki/EventRegistry)
- [Warcraft Wiki — SecureActionButtonTemplate](https://warcraft.wiki.gg/wiki/SecureActionButtonTemplate)
- [Gethe/wow-ui-source (GitHub)](https://github.com/Gethe/wow-ui-source) — Blizzard's UI source code mirror
- [WoWUIDev Discord](https://discord.gg/wowuidev) — Where addon developers discuss and share code
