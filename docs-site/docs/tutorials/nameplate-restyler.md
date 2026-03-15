---
title: "Build-Along: PlateStylist — Nameplate Restyler"
description: "Advanced build-along tutorial for a full-featured WoW nameplate restyler addon using Boundary Pusher techniques — class colors, threat indicators, cast bar enhancements, mob icons, and aura filters for Patch 12.0+ Midnight."
---

# Build-Along: PlateStylist — Nameplate Restyler

**Difficulty:** :material-star::material-star::material-star: Advanced |
**Mode:** :material-rocket: Boundary Pusher |
**Reading time:** ~35 minutes |
**Interface:** 120001

---

## Step 1: What We're Building

Nameplates are the most information-dense UI element in World of Warcraft. Tanks use them to track threat. Healers scan them for debuffs. DPS watch them for interrupt windows. In Mythic+ and PvP, nameplate readability is the difference between a clean pull and a wipe.

Blizzard's default nameplates are functional but visually sparse. They show health, a cast bar, and a name — all in the same neutral tones regardless of context. PlateStylist changes that. By the end of this tutorial, your nameplates will show:

- **Class-colored health bars** for player nameplates (enemy and friendly)
- **Threat-based coloring** that shifts from green to yellow to red as your threat changes
- **Enhanced cast bars** with spell name text, cast time remaining, and a visual distinction between interruptible and non-interruptible casts
- **Mob classification icons** — elite dragons, rare stars, world boss skulls — drawn from Blizzard's Atlas system
- **Aura indicators** using the four new aura filter categories added in 12.0: `CROWD_CONTROL`, `BIG_DEFENSIVE`, `RAID_PLAYER_DISPELLABLE`, and `RAID_IN_COMBAT`

This is not a toy addon. Platynator proved that deep nameplate customization is fully possible in 12.0 without replacing the secure frame infrastructure. PlateStylist follows the same approach: hook after Blizzard, modify presentation, never touch protected state.

You will learn how to work with `NAME_PLATE` events, hook into `CompactUnitFrame` update functions, use `GetRegions()` to strip default textures, apply metatable hooks for persistent styling, handle combat lockdown safely, and use the new aura filter system. These are Boundary Pusher techniques — they work, but they push against the edges of what Blizzard intended. Every boundary technique in this tutorial is wrapped in safety checks and fallback paths.

!!! tip "Prerequisites"

    This tutorial assumes you've completed the [Getting Started](../getting-started.md) guide, read [Coding for Midnight](../midnight-patterns.md), and understand the [Security Model](../security.md). You should be comfortable with `hooksecurefunc`, `CreateFrame`, and the event dispatch pattern.

---

## Step 2: Why Boundary Pusher Mode

An Enhancement Artist approach to nameplates would use `hooksecurefunc("CompactUnitFrame_UpdateHealth", ...)` to recolor the health bar after Blizzard updates it. That works for simple color changes. But PlateStylist goes further:

- **Stripping default textures** — Blizzard's nameplate health bar has a built-in border texture and background. To apply a custom backdrop, you need to hide these defaults using `GetRegions()`. Enhancement Artist mode forbids `GetRegions()` because it accesses internal frame structure that can change between patches.

- **Metatable hooks** — To ensure our styling persists even when Blizzard code resets the health bar color, we hook the `SetStatusBarColor` method on the health bar's metatable. This intercepts Blizzard's own calls and re-applies our colors after. Enhancement Artist mode forbids metatable manipulation.

- **The noop pattern** — Some Blizzard textures are re-applied every frame update. Rather than fighting them, we replace their `SetTexture` method with a no-op function. The texture exists but does nothing. This is a boundary technique that would be flagged in Enhancement Artist mode.

Every boundary technique in this tutorial is marked with a `-- BOUNDARY` comment and wrapped in a `pcall` safety block. If any technique fails — because Blizzard changed the frame structure, renamed an internal function, or tightened the security model — the addon degrades gracefully to default nameplates rather than throwing errors.

=== ":material-palette: Enhancement Artist Approach"

    ```lua
    -- Safe but limited: only recolor after Blizzard updates
    hooksecurefunc("CompactUnitFrame_UpdateHealth", function(frame)
        if frame:IsForbidden() then return end
        -- Can recolor, but Blizzard may override next frame
        frame.healthBar:SetStatusBarColor(1, 0, 0)
    end)
    ```

=== ":material-rocket: Boundary Pusher Approach"

    ```lua
    -- Persistent: hook the metatable so our color survives Blizzard resets
    -- BOUNDARY: metatable hook on SetStatusBarColor
    local ok = pcall(function()
        local mt = getmetatable(frame.healthBar).__index
        hooksecurefunc(mt, "SetStatusBarColor", function(self, r, g, b)
            if self._plateStylistColor then
                rawset(self, "_plateStylistOverride", true)
            end
        end)
    end)
    if not ok then
        -- Fallback: use hooksecurefunc on the global function
        hooksecurefunc("CompactUnitFrame_UpdateHealth", ApplyColor)
    end
    ```

!!! danger "Boundary Technique: Safety Contract"

    Every `-- BOUNDARY` comment in this tutorial marks code that accesses Blizzard internals. These techniques:

    1. **May break between patches** — Blizzard can rename or restructure internal frames
    2. **Must be wrapped in `pcall`** — Never let a boundary technique error propagate
    3. **Must have a fallback path** — If the technique fails, the addon must still function
    4. **Must never modify secure state** — Only presentation, never protected attributes

---

## Step 3: Project Setup

Create the following file structure in your `Interface/AddOns/` directory:

```
PlateStylist/
├── PlateStylist.toc
├── Core.lua
├── Plates.lua
├── CastBar.lua
└── Config.lua
```

**PlateStylist.toc:**

```toc
## Interface: 120001
## Title: PlateStylist
## Notes: Nameplate restyler with class colors, threat indicators, and cast bar enhancements.
## Author: YourName
## Version: 1.0.0
## SavedVariables: PlateStylistDB
## IconTexture: Interface\Icons\Spell_Shadow_SoulGem

Core.lua
Plates.lua
CastBar.lua
Config.lua
```

**Core.lua** handles namespace setup, event dispatch, SavedVariables initialization, and the combat lockdown queue. Every other file builds on top of `ns`.

---

## Step 4: Nameplate Event System

The nameplate lifecycle is driven by three events:

| Event | When It Fires | What You Do |
|-------|---------------|-------------|
| `NAME_PLATE_CREATED` | A nameplate frame is created (pooled, reused) | One-time frame setup |
| `NAME_PLATE_UNIT_ADDED` | A nameplate is assigned to a unit | Apply per-unit styling |
| `NAME_PLATE_UNIT_REMOVED` | A nameplate is unassigned from a unit | Clean up per-unit state |

When `NAME_PLATE_UNIT_ADDED` fires, you receive a `unitID` like `"nameplate1"`. Call `C_NamePlate.GetNamePlateForUnit(unitID)` to get the actual frame. The frame hierarchy looks like this:

```
NamePlateN (root frame)
└── UnitFrame (CompactUnitFrame)
    ├── healthBar (StatusBar)
    │   ├── background (Texture)
    │   └── border (Texture)
    ├── castBar (StatusBar)
    │   ├── Icon (Texture)
    │   ├── Text (FontString)
    │   └── BorderShield (Texture)
    ├── name (FontString)
    ├── RaidTargetFrame
    └── BuffFrame
```

!!! warning "IsForbidden() — Always Check First"

    Some nameplate frames are *forbidden* — they belong to units that the security model protects from addon modification. **Every single function** in PlateStylist that touches a nameplate frame must start with:

    ```lua
    if frame:IsForbidden() then return end
    ```

    Calling any method on a forbidden frame causes a taint error that can cascade through the entire UI. There is no recovery. Check first, always.

Here is the event dispatcher for nameplate lifecycle:

```lua
-- Core.lua
local ADDON_NAME, ns = ...
ns.ADDON_NAME = ADDON_NAME
ns.db = {}
ns.plates = {} -- Track active nameplates

local defaults = {
    classColors = true,
    threatColors = true,
    castBarEnhance = true,
    mobIcons = true,
    auraIndicators = true,
}

local frame = CreateFrame("Frame")
local events = {}

function events:ADDON_LOADED(addonName)
    if addonName ~= ADDON_NAME then return end
    PlateStylistDB = PlateStylistDB or {}
    for k, v in pairs(defaults) do
        if PlateStylistDB[k] == nil then
            PlateStylistDB[k] = v
        end
    end
    ns.db = PlateStylistDB
    frame:UnregisterEvent("ADDON_LOADED")
end

function events:NAME_PLATE_UNIT_ADDED(unitID)
    local plate = C_NamePlate.GetNamePlateForUnit(unitID)
    if not plate then return end
    local unitFrame = plate.UnitFrame
    if not unitFrame or unitFrame:IsForbidden() then return end
    ns.plates[unitID] = plate
    ns.StylePlate(unitFrame, unitID)
end

function events:NAME_PLATE_UNIT_REMOVED(unitID)
    local plate = ns.plates[unitID]
    if plate and plate.UnitFrame and not plate.UnitFrame:IsForbidden() then
        ns.CleanupPlate(plate.UnitFrame, unitID)
    end
    ns.plates[unitID] = nil
end

-- Combat lockdown queue
local combatQueue = {}

function ns.QueueForCombatEnd(func)
    combatQueue[#combatQueue + 1] = func
end

function events:PLAYER_REGEN_ENABLED()
    for i, func in ipairs(combatQueue) do
        func()
        combatQueue[i] = nil
    end
end

for event in pairs(events) do
    frame:RegisterEvent(event)
end
frame:SetScript("OnEvent", function(self, event, ...)
    if events[event] then events[event](self, ...) end
end)
```

---

## Step 5: Stripping Default Textures

This is the first Boundary Pusher technique. Blizzard's nameplate health bar comes with built-in border and background textures that clash with custom styling. To apply a clean custom backdrop, you need to suppress these defaults.

`GetRegions()` returns all texture and font string regions attached to a frame. By iterating them, you can identify and hide the ones you want to replace.

!!! danger "Boundary Technique: GetRegions() Stripping"

    `GetRegions()` accesses Blizzard's internal frame structure. The number, order, and type of regions can change between patches. Always wrap in `pcall` and fall back to default styling if it fails.

    ```lua
    -- BOUNDARY: GetRegions() stripping — suppress default textures
    local ok, err = pcall(function()
        local regions = { frame.healthBar:GetRegions() }
        for _, region in ipairs(regions) do
            if region:IsObjectType("Texture") then
                local tex = region:GetTexture()
                if tex and (tex:find("NamePlate") or tex:find("UI%-Castbar")) then
                    region:SetTexture(nil)
                    region:Hide()
                end
            end
        end
    end)
    if not ok then
        -- Fallback: leave default textures in place, style on top
        ns.fallbackMode = true
    end
    ```

The **noop pattern** takes this further. Some textures are re-applied by Blizzard's update cycle every frame. Hiding them once isn't enough — they come back. The noop pattern replaces the texture's `SetTexture` method with an empty function so Blizzard's own code calls it but nothing happens:

!!! danger "Boundary Technique: Noop Pattern"

    Replacing a widget method with a no-op is aggressive. The original method is lost unless you save a reference. Always store the original for restoration.

    ```lua
    -- BOUNDARY: noop pattern — prevent Blizzard from re-applying textures
    local function ApplyNoop(region)
        if region._origSetTexture then return end -- already nooped
        region._origSetTexture = region.SetTexture
        region.SetTexture = function() end
    end

    local function RestoreFromNoop(region)
        if region._origSetTexture then
            region.SetTexture = region._origSetTexture
            region._origSetTexture = nil
        end
    end
    ```

=== ":material-palette: Enhancement Artist"

    ```lua
    -- No texture stripping. Create an overlay on top of defaults.
    local overlay = frame.healthBar:CreateTexture(nil, "OVERLAY")
    overlay:SetAllPoints()
    overlay:SetColorTexture(0, 0, 0, 0.5)
    ```

=== ":material-rocket: Boundary Pusher"

    ```lua
    -- BOUNDARY: Strip defaults, apply clean backdrop
    pcall(function()
        for _, region in ipairs({ frame.healthBar:GetRegions() }) do
            if region:IsObjectType("Texture") then
                ApplyNoop(region)
            end
        end
    end)
    -- Now apply our own backdrop without visual conflict
    ```

---

## Step 6: Custom Health Bar Styling

With default textures suppressed, we can apply a clean custom backdrop and class-colored health bars. The health bar is a `StatusBar` widget — we control its color via `SetStatusBarColor()`.

**Plates.lua** handles all health bar and threat styling:

```lua
-- Plates.lua
local ADDON_NAME, ns = ...

local THREAT_COLORS = {
    [0] = { 0.69, 0.69, 0.69 }, -- grey: not on threat table
    [1] = { 1.00, 1.00, 0.47 }, -- yellow: threat rising
    [2] = { 1.00, 0.60, 0.00 }, -- orange: close to pulling
    [3] = { 1.00, 0.00, 0.00 }, -- red: tanking
}

local noop = function() end

local function StripTextures(healthBar)
    -- BOUNDARY: GetRegions() stripping
    local ok = pcall(function()
        for _, region in ipairs({ healthBar:GetRegions() }) do
            if region:IsObjectType("Texture") and not region._psNooped then
                region._psOrigSetTexture = region.SetTexture
                region.SetTexture = noop
                region:Hide()
                region._psNooped = true
            end
        end
    end)
    return ok
end

local function RestoreTextures(healthBar)
    pcall(function()
        for _, region in ipairs({ healthBar:GetRegions() }) do
            if region._psNooped then
                region.SetTexture = region._psOrigSetTexture
                region._psOrigSetTexture = nil
                region._psNooped = nil
                region:Show()
            end
        end
    end)
end

local function CreateBackdrop(healthBar)
    if healthBar._psBackdrop then return end
    local bd = CreateFrame("Frame", nil, healthBar, "BackdropTemplate")
    bd:SetPoint("TOPLEFT", -2, 2)
    bd:SetPoint("BOTTOMRIGHT", 2, -2)
    bd:SetBackdrop({
        bgFile = "Interface\\Buttons\\WHITE8x8",
        edgeFile = "Interface\\Buttons\\WHITE8x8",
        edgeSize = 1,
    })
    bd:SetBackdropColor(0, 0, 0, 0.7)
    bd:SetBackdropBorderColor(0, 0, 0, 1)
    bd:SetFrameLevel(healthBar:GetFrameLevel() - 1)
    healthBar._psBackdrop = bd
end

local function ApplyClassColor(healthBar, unit)
    if not UnitIsPlayer(unit) then return false end
    local _, class = UnitClass(unit)
    if not class then return false end
    local color = RAID_CLASS_COLORS[class]
    if color then
        healthBar:SetStatusBarColor(color.r, color.g, color.b)
        return true
    end
    return false
end

local function ApplyThreatColor(healthBar, unit)
    local _, _, _, status = UnitDetailedThreatSituation("player", unit)
    if not status then return false end
    local color = THREAT_COLORS[status]
    if color then
        healthBar:SetStatusBarColor(color[1], color[2], color[3])
        return true
    end
    return false
end
```

The key decision is priority: **threat colors override class colors** when you're in combat with a mob. Class colors apply to players and out-of-combat mobs.

```lua
local function UpdateHealthBarColor(healthBar, unit)
    if not ns.db.classColors and not ns.db.threatColors then return end

    -- Threat takes priority when in combat with the unit
    if ns.db.threatColors and UnitThreatSituation("player", unit) then
        if ApplyThreatColor(healthBar, unit) then return end
    end

    -- Class colors for players
    if ns.db.classColors then
        if ApplyClassColor(healthBar, unit) then return end
    end
end
```

To make the color persistent across Blizzard's own updates, hook the global update function:

```lua
hooksecurefunc("CompactUnitFrame_UpdateHealth", function(frame)
    if frame:IsForbidden() then return end
    if not frame.unit then return end

    local healthBar = frame.healthBar
    if not healthBar then return end

    UpdateHealthBarColor(healthBar, frame.unit)
end)

hooksecurefunc("CompactUnitFrame_UpdateAggroFlash", function(frame)
    if frame:IsForbidden() then return end
    if not frame.unit then return end

    local healthBar = frame.healthBar
    if not healthBar then return end

    if ns.db.threatColors then
        ApplyThreatColor(healthBar, frame.unit)
    end
end)
```

---

## Step 7: Cast Bar Enhancement

Cast bars are where nameplate addons earn their keep. Knowing which casts are interruptible — and how much time is left — is critical in M+ and PvP. Blizzard's default cast bar shows a progress bar and a shield icon for non-interruptible casts, but no spell name text and no time remaining.

**CastBar.lua** hooks into the nameplate cast bar to add these elements:

```lua
-- CastBar.lua
local ADDON_NAME, ns = ...

local INTERRUPTIBLE_COLOR = { 0.30, 0.70, 1.00 }    -- blue
local NON_INTERRUPTIBLE_COLOR = { 0.70, 0.30, 0.30 } -- muted red

local function SetupCastBar(castBar)
    if castBar._psEnhanced then return end
    if castBar:IsForbidden() then return end

    -- Spell name text
    local spellText = castBar:CreateFontString(nil, "OVERLAY", "GameFontHighlightSmall")
    spellText:SetPoint("LEFT", castBar, "LEFT", 4, 0)
    spellText:SetJustifyH("LEFT")
    spellText:SetWidth(castBar:GetWidth() * 0.65)
    spellText:SetWordWrap(false)
    castBar._psSpellText = spellText

    -- Cast time remaining
    local timeText = castBar:CreateFontString(nil, "OVERLAY", "GameFontHighlightSmall")
    timeText:SetPoint("RIGHT", castBar, "RIGHT", -4, 0)
    timeText:SetJustifyH("RIGHT")
    castBar._psTimeText = timeText

    castBar._psEnhanced = true
end

local function UpdateCastBar(castBar, unit)
    if not castBar._psEnhanced then return end
    if castBar:IsForbidden() then return end

    local spellName, _, _, startTimeMS, endTimeMS, _, _, notInterruptible =
        UnitCastingInfo(unit)

    if not spellName then
        spellName, _, _, startTimeMS, endTimeMS, _, notInterruptible =
            UnitChannelInfo(unit)
    end

    if spellName then
        castBar._psSpellText:SetText(spellName)
        castBar._psSpellText:Show()

        -- Color based on interruptibility
        if notInterruptible then
            castBar:SetStatusBarColor(unpack(NON_INTERRUPTIBLE_COLOR))
        else
            castBar:SetStatusBarColor(unpack(INTERRUPTIBLE_COLOR))
        end

        -- Time remaining on OnUpdate
        if not castBar._psOnUpdate then
            castBar:HookScript("OnUpdate", function(self)
                if not self._psTimeText then return end
                local _, _, _, sMS, eMS = UnitCastingInfo(unit)
                if not sMS then
                    _, _, _, sMS, eMS = UnitChannelInfo(unit)
                end
                if sMS and eMS then
                    local remaining = (eMS - GetTime() * 1000) / 1000
                    if remaining > 0 then
                        self._psTimeText:SetText(format("%.1fs", remaining))
                        self._psTimeText:Show()
                    else
                        self._psTimeText:Hide()
                    end
                else
                    self._psTimeText:Hide()
                end
            end)
            castBar._psOnUpdate = true
        end
    else
        castBar._psSpellText:Hide()
        castBar._psTimeText:Hide()
    end
end
```

Hook the cast bar into the nameplate lifecycle:

```lua
-- Hook cast events on nameplates
hooksecurefunc("CompactUnitFrame_UpdateAll", function(frame)
    if frame:IsForbidden() then return end
    if not ns.db.castBarEnhance then return end
    if not frame.unit then return end

    local castBar = frame.CastBar or frame.castBar
    if castBar and not castBar:IsForbidden() then
        SetupCastBar(castBar)
        UpdateCastBar(castBar, frame.unit)
    end
end)
```

!!! warning "Cast Bar Hooks and Combat"

    Cast bar frames on nameplates are generally not protected — you can modify their appearance during combat. However, `HookScript("OnUpdate", ...)` on a cast bar runs every frame. Keep the callback lean: no table allocations, no string concatenation (use `format()`), and no API calls beyond `UnitCastingInfo` / `UnitChannelInfo`. A slow OnUpdate callback on 40 nameplates will tank your framerate.

---

## Step 8: Mob Classification Icons

Every mob in WoW has a classification: normal, elite, rare, rareelite, worldboss, or minus (trivial). Showing a small icon next to the nameplate name instantly communicates what you're looking at.

Blizzard provides Atlas icons for each classification type. Atlas icons are resolution-independent and always available — no need to bundle texture files.

```lua
-- Classification icon mapping (Atlas references)
local CLASSIFICATION_ATLAS = {
    elite       = "nameplates-icon-elite-gold",
    rareelite   = "nameplates-icon-elite-silver",
    rare        = "nameplates-icon-star-silver",
    worldboss   = "nameplates-icon-elite-gold",
    minus       = nil, -- no icon for trivial mobs
}

local function SetupClassificationIcon(unitFrame)
    if unitFrame._psClassIcon then return end
    local icon = unitFrame:CreateTexture(nil, "OVERLAY")
    icon:SetSize(14, 14)
    icon:SetPoint("LEFT", unitFrame.name, "RIGHT", 2, 0)
    icon:Hide()
    unitFrame._psClassIcon = icon
end

local function UpdateClassificationIcon(unitFrame, unit)
    if not ns.db.mobIcons then return end
    if not unitFrame._psClassIcon then
        SetupClassificationIcon(unitFrame)
    end

    local classification = UnitClassification(unit)
    local atlas = CLASSIFICATION_ATLAS[classification]

    if atlas then
        unitFrame._psClassIcon:SetAtlas(atlas)
        unitFrame._psClassIcon:Show()
    else
        unitFrame._psClassIcon:Hide()
    end
end
```

!!! tip "Atlas vs. File Paths"

    Always prefer `SetAtlas("atlas-name")` over `SetTexture("Interface\\...")` for Blizzard's built-in icons. Atlas entries are maintained by Blizzard and won't break when texture files are reorganized. Use `/dump C_Texture.GetAtlasInfo("atlas-name")` in-game to verify an Atlas exists.

---

## Step 9: Aura Indicators with New Filters

Patch 12.0 introduced four new aura filter categories that dramatically simplify nameplate aura display. Previously, addons maintained their own spell ID lists to classify auras. Now Blizzard provides built-in filters:

| Filter | What It Contains |
|--------|------------------|
| `CROWD_CONTROL` | Stuns, roots, fears, polymorphs, hex, etc. |
| `BIG_DEFENSIVE` | Major defensives (Divine Shield, Ice Block, etc.) |
| `RAID_PLAYER_DISPELLABLE` | Debuffs you can dispel with your class |
| `RAID_IN_COMBAT` | Auras relevant during combat |

These filters work with `C_UnitAuras.GetAuraDataByIndex()` via the `filter` parameter. Instead of building a 500-entry spell ID table, you pass a filter string and get exactly the auras that match.

```lua
-- Aura indicator setup with object pooling
local AURA_FILTERS = { "CROWD_CONTROL", "BIG_DEFENSIVE" }
local MAX_AURAS = 4

local auraPool = {}

local function GetAuraIcon(parent)
    local icon = tremove(auraPool)
    if not icon then
        icon = CreateFrame("Frame", nil, parent)
        icon:SetSize(16, 16)
        icon.texture = icon:CreateTexture(nil, "OVERLAY")
        icon.texture:SetAllPoints()
        icon.cooldown = CreateFrame("Cooldown", nil, icon, "CooldownFrameTemplate")
        icon.cooldown:SetAllPoints()
        icon.cooldown:SetDrawEdge(false)
    end
    icon:SetParent(parent)
    icon:Show()
    return icon
end

local function ReleaseAuraIcon(icon)
    icon:Hide()
    icon:ClearAllPoints()
    auraPool[#auraPool + 1] = icon
end

local function UpdateAuraIndicators(unitFrame, unit)
    if not ns.db.auraIndicators then return end

    -- Release existing icons
    if unitFrame._psAuraIcons then
        for _, icon in ipairs(unitFrame._psAuraIcons) do
            ReleaseAuraIcon(icon)
        end
        wipe(unitFrame._psAuraIcons)
    else
        unitFrame._psAuraIcons = {}
    end

    local count = 0
    for _, filter in ipairs(AURA_FILTERS) do
        local i = 1
        while count < MAX_AURAS do
            local auraData = C_UnitAuras.GetAuraDataByIndex(unit, i, filter)
            if not auraData then break end

            count = count + 1
            local icon = GetAuraIcon(unitFrame)
            icon:SetPoint("BOTTOMLEFT", unitFrame.healthBar, "TOPLEFT",
                (count - 1) * 18, 4)
            icon.texture:SetTexture(auraData.icon)

            if auraData.duration and auraData.duration > 0 then
                icon.cooldown:SetCooldown(
                    auraData.expirationTime - auraData.duration,
                    auraData.duration
                )
            else
                icon.cooldown:Clear()
            end

            unitFrame._psAuraIcons[count] = icon
            i = i + 1
        end
    end
end
```

!!! tip "Object Pooling"

    Never call `CreateFrame()` inside a per-nameplate update function. With 40 nameplates active, creating frames every update cycle generates massive garbage collection pressure. The object pool pattern above reuses icons across nameplate recycles, keeping allocations near zero during gameplay.

Hook aura updates into the nameplate lifecycle via `UNIT_AURA`:

```lua
-- Register for aura updates on nameplate units
hooksecurefunc("CompactUnitFrame_UpdateAuras", function(frame)
    if frame:IsForbidden() then return end
    if not frame.unit then return end
    UpdateAuraIndicators(frame, frame.unit)
end)
```

---

## Step 10: Combat Lockdown and Safety

Nameplate frames are partially secure. During combat, you can still modify textures, colors, and font strings — these are presentation-only changes. But you **cannot** create new frames, change frame parents, or modify protected attributes while `InCombatLockdown()` returns true.

PlateStylist handles this with three safety layers:

**Layer 1: IsForbidden check on every entry point**

```lua
-- Every public function starts with this
if frame:IsForbidden() then return end
```

**Layer 2: Combat lockdown queue for frame creation**

```lua
local function EnsureBackdrop(healthBar)
    if healthBar._psBackdrop then return end
    if InCombatLockdown() then
        ns.QueueForCombatEnd(function()
            CreateBackdrop(healthBar)
        end)
        return
    end
    CreateBackdrop(healthBar)
end
```

**Layer 3: pcall on all boundary techniques**

```lua
-- BOUNDARY: Every GetRegions/metatable operation is pcall-wrapped
local function SafeStripTextures(healthBar)
    local ok, err = pcall(StripTextures, healthBar)
    if not ok then
        -- Log for debugging, continue with defaults
        if ns.debug then
            print("|cffff6600PlateStylist:|r texture strip failed:", err)
        end
    end
    return ok
end
```

!!! warning "Combat Lockdown Rules"

    | Safe During Combat | Blocked During Combat |
    |---|---|
    | `SetStatusBarColor()` | `CreateFrame()` |
    | `SetTexture()` / `SetAtlas()` | `SetParent()` on secure frames |
    | `SetText()` on FontStrings | `RegisterForClicks()` |
    | `SetAlpha()` | Modifying `SecureActionButton` attributes |
    | `Show()` / `Hide()` on non-secure children | `SetAttribute()` on secure frames |

    When in doubt, defer to `PLAYER_REGEN_ENABLED`. The combat queue pattern ensures nothing is lost — operations are queued and executed the moment combat ends.

---

## Step 11: Complete Code

Below is the complete, copy-pasteable code for every file. Create the `PlateStylist` folder in your `Interface/AddOns/` directory and paste each file.

### PlateStylist.toc

```toc
## Interface: 120001
## Title: PlateStylist
## Notes: Nameplate restyler with class colors, threat indicators, cast bar enhancements, and mob classification icons.
## Author: YourName
## Version: 1.0.0
## SavedVariables: PlateStylistDB
## IconTexture: Interface\Icons\Spell_Shadow_SoulGem

Core.lua
Plates.lua
CastBar.lua
Config.lua
```

### Core.lua

```lua
local ADDON_NAME, ns = ...
ns.ADDON_NAME = ADDON_NAME
ns.plates = {}
ns.fallbackMode = false
ns.debug = false

local defaults = {
    classColors = true,
    threatColors = true,
    castBarEnhance = true,
    mobIcons = true,
    auraIndicators = true,
}

local frame = CreateFrame("Frame")
local events = {}

function events:ADDON_LOADED(addonName)
    if addonName ~= ADDON_NAME then return end
    PlateStylistDB = PlateStylistDB or {}
    for k, v in pairs(defaults) do
        if PlateStylistDB[k] == nil then
            PlateStylistDB[k] = v
        end
    end
    ns.db = PlateStylistDB
    frame:UnregisterEvent("ADDON_LOADED")
end

function events:NAME_PLATE_UNIT_ADDED(unitID)
    local plate = C_NamePlate.GetNamePlateForUnit(unitID)
    if not plate then return end
    local unitFrame = plate.UnitFrame
    if not unitFrame or unitFrame:IsForbidden() then return end
    ns.plates[unitID] = plate
    ns.StylePlate(unitFrame, unitID)
end

function events:NAME_PLATE_UNIT_REMOVED(unitID)
    local plate = ns.plates[unitID]
    if plate and plate.UnitFrame and not plate.UnitFrame:IsForbidden() then
        ns.CleanupPlate(plate.UnitFrame, unitID)
    end
    ns.plates[unitID] = nil
end

-- Combat lockdown queue
local combatQueue = {}

function ns.QueueForCombatEnd(func)
    combatQueue[#combatQueue + 1] = func
end

function events:PLAYER_REGEN_ENABLED()
    for i = 1, #combatQueue do
        combatQueue[i]()
        combatQueue[i] = nil
    end
end

for event in pairs(events) do
    frame:RegisterEvent(event)
end
frame:SetScript("OnEvent", function(self, event, ...)
    if events[event] then events[event](self, ...) end
end)

SLASH_PLATESTYLIST1 = "/platestylist"
SLASH_PLATESTYLIST2 = "/ps"
SlashCmdList["PLATESTYLIST"] = function(msg)
    if msg == "debug" then
        ns.debug = not ns.debug
        print("|cff00ccffPlateStylist:|r debug", ns.debug and "ON" or "OFF")
    else
        print("|cff00ccffPlateStylist|r v1.0.0")
        print("  /ps debug — Toggle debug output")
    end
end
```

### Plates.lua

```lua
local ADDON_NAME, ns = ...

local noop = function() end

local THREAT_COLORS = {
    [0] = { 0.69, 0.69, 0.69 },
    [1] = { 1.00, 1.00, 0.47 },
    [2] = { 1.00, 0.60, 0.00 },
    [3] = { 1.00, 0.00, 0.00 },
}

local CLASSIFICATION_ATLAS = {
    elite     = "nameplates-icon-elite-gold",
    rareelite = "nameplates-icon-elite-silver",
    rare      = "nameplates-icon-star-silver",
    worldboss = "nameplates-icon-elite-gold",
}

-- Aura system
local AURA_FILTERS = { "CROWD_CONTROL", "BIG_DEFENSIVE" }
local MAX_AURAS = 4
local auraPool = {}

local function GetAuraIcon(parent)
    local icon = tremove(auraPool)
    if not icon then
        icon = CreateFrame("Frame", nil, parent)
        icon:SetSize(16, 16)
        icon.texture = icon:CreateTexture(nil, "OVERLAY")
        icon.texture:SetAllPoints()
        icon.cooldown = CreateFrame("Cooldown", nil, icon, "CooldownFrameTemplate")
        icon.cooldown:SetAllPoints()
        icon.cooldown:SetDrawEdge(false)
    end
    icon:SetParent(parent)
    icon:Show()
    return icon
end

local function ReleaseAuraIcon(icon)
    icon:Hide()
    icon:ClearAllPoints()
    auraPool[#auraPool + 1] = icon
end

-- BOUNDARY: Texture stripping
local function StripTextures(healthBar)
    -- BOUNDARY: GetRegions() stripping — suppress default textures
    local ok = pcall(function()
        for _, region in ipairs({ healthBar:GetRegions() }) do
            if region:IsObjectType("Texture") and not region._psNooped then
                region._psOrigSetTexture = region.SetTexture
                region.SetTexture = noop
                region:Hide()
                region._psNooped = true
            end
        end
    end)
    if not ok then
        ns.fallbackMode = true
    end
    return ok
end

local function RestoreTextures(healthBar)
    pcall(function()
        for _, region in ipairs({ healthBar:GetRegions() }) do
            if region._psNooped then
                region.SetTexture = region._psOrigSetTexture
                region._psOrigSetTexture = nil
                region._psNooped = nil
                region:Show()
            end
        end
    end)
end

local function CreateBackdrop(healthBar)
    if healthBar._psBackdrop then return end
    if InCombatLockdown() then
        ns.QueueForCombatEnd(function() CreateBackdrop(healthBar) end)
        return
    end
    local bd = CreateFrame("Frame", nil, healthBar, "BackdropTemplate")
    bd:SetPoint("TOPLEFT", -2, 2)
    bd:SetPoint("BOTTOMRIGHT", 2, -2)
    bd:SetBackdrop({
        bgFile = "Interface\\Buttons\\WHITE8x8",
        edgeFile = "Interface\\Buttons\\WHITE8x8",
        edgeSize = 1,
    })
    bd:SetBackdropColor(0, 0, 0, 0.7)
    bd:SetBackdropBorderColor(0, 0, 0, 1)
    bd:SetFrameLevel(math.max(1, healthBar:GetFrameLevel() - 1))
    healthBar._psBackdrop = bd
end

local function ApplyClassColor(healthBar, unit)
    if not UnitIsPlayer(unit) then return false end
    local _, class = UnitClass(unit)
    if not class then return false end
    local color = RAID_CLASS_COLORS[class]
    if color then
        healthBar:SetStatusBarColor(color.r, color.g, color.b)
        return true
    end
    return false
end

local function ApplyThreatColor(healthBar, unit)
    local _, _, _, status = UnitDetailedThreatSituation("player", unit)
    if not status then return false end
    local color = THREAT_COLORS[status]
    if color then
        healthBar:SetStatusBarColor(color[1], color[2], color[3])
        return true
    end
    return false
end

local function UpdateHealthBarColor(healthBar, unit)
    if not ns.db.classColors and not ns.db.threatColors then return end
    if ns.db.threatColors and UnitThreatSituation("player", unit) then
        if ApplyThreatColor(healthBar, unit) then return end
    end
    if ns.db.classColors then
        if ApplyClassColor(healthBar, unit) then return end
    end
end

local function SetupClassificationIcon(unitFrame)
    if unitFrame._psClassIcon then return end
    local icon = unitFrame:CreateTexture(nil, "OVERLAY")
    icon:SetSize(14, 14)
    icon:SetPoint("LEFT", unitFrame.name, "RIGHT", 2, 0)
    icon:Hide()
    unitFrame._psClassIcon = icon
end

local function UpdateClassificationIcon(unitFrame, unit)
    if not ns.db.mobIcons then return end
    if not unitFrame._psClassIcon then
        if InCombatLockdown() then return end
        SetupClassificationIcon(unitFrame)
    end
    local classification = UnitClassification(unit)
    local atlas = CLASSIFICATION_ATLAS[classification]
    if atlas then
        unitFrame._psClassIcon:SetAtlas(atlas)
        unitFrame._psClassIcon:Show()
    else
        unitFrame._psClassIcon:Hide()
    end
end

local function UpdateAuraIndicators(unitFrame, unit)
    if not ns.db.auraIndicators then return end
    if unitFrame._psAuraIcons then
        for _, icon in ipairs(unitFrame._psAuraIcons) do
            ReleaseAuraIcon(icon)
        end
        wipe(unitFrame._psAuraIcons)
    else
        unitFrame._psAuraIcons = {}
    end

    local count = 0
    for _, filter in ipairs(AURA_FILTERS) do
        local i = 1
        while count < MAX_AURAS do
            local auraData = C_UnitAuras.GetAuraDataByIndex(unit, i, filter)
            if not auraData then break end
            count = count + 1
            local icon = GetAuraIcon(unitFrame)
            icon:SetPoint("BOTTOMLEFT", unitFrame.healthBar, "TOPLEFT",
                (count - 1) * 18, 4)
            icon.texture:SetTexture(auraData.icon)
            if auraData.duration and auraData.duration > 0 then
                icon.cooldown:SetCooldown(
                    auraData.expirationTime - auraData.duration,
                    auraData.duration
                )
            else
                icon.cooldown:Clear()
            end
            unitFrame._psAuraIcons[count] = icon
            i = i + 1
        end
    end
end

-- Public API
function ns.StylePlate(unitFrame, unitID)
    if unitFrame:IsForbidden() then return end
    local healthBar = unitFrame.healthBar
    if not healthBar then return end

    if not ns.fallbackMode then
        StripTextures(healthBar)
    end
    CreateBackdrop(healthBar)
    UpdateHealthBarColor(healthBar, unitID)
    UpdateClassificationIcon(unitFrame, unitID)
    UpdateAuraIndicators(unitFrame, unitID)
end

function ns.CleanupPlate(unitFrame, unitID)
    if unitFrame._psAuraIcons then
        for _, icon in ipairs(unitFrame._psAuraIcons) do
            ReleaseAuraIcon(icon)
        end
        wipe(unitFrame._psAuraIcons)
    end
    if unitFrame._psClassIcon then
        unitFrame._psClassIcon:Hide()
    end
end

-- Global hooks for persistent updates
hooksecurefunc("CompactUnitFrame_UpdateHealth", function(frame)
    if frame:IsForbidden() then return end
    if not frame.unit then return end
    local healthBar = frame.healthBar
    if not healthBar then return end
    UpdateHealthBarColor(healthBar, frame.unit)
end)

hooksecurefunc("CompactUnitFrame_UpdateAggroFlash", function(frame)
    if frame:IsForbidden() then return end
    if not frame.unit or not frame.healthBar then return end
    if ns.db.threatColors then
        ApplyThreatColor(frame.healthBar, frame.unit)
    end
end)

hooksecurefunc("CompactUnitFrame_UpdateAuras", function(frame)
    if frame:IsForbidden() then return end
    if not frame.unit then return end
    UpdateAuraIndicators(frame, frame.unit)
end)
```

### CastBar.lua

```lua
local ADDON_NAME, ns = ...

local INTERRUPTIBLE_COLOR = { 0.30, 0.70, 1.00 }
local NON_INTERRUPTIBLE_COLOR = { 0.70, 0.30, 0.30 }
local format = format

local function SetupCastBar(castBar)
    if castBar._psEnhanced or castBar:IsForbidden() then return end

    local spellText = castBar:CreateFontString(nil, "OVERLAY", "GameFontHighlightSmall")
    spellText:SetPoint("LEFT", castBar, "LEFT", 4, 0)
    spellText:SetJustifyH("LEFT")
    spellText:SetWidth(castBar:GetWidth() * 0.65)
    spellText:SetWordWrap(false)
    castBar._psSpellText = spellText

    local timeText = castBar:CreateFontString(nil, "OVERLAY", "GameFontHighlightSmall")
    timeText:SetPoint("RIGHT", castBar, "RIGHT", -4, 0)
    timeText:SetJustifyH("RIGHT")
    castBar._psTimeText = timeText

    castBar._psEnhanced = true
end

local function UpdateCastBar(castBar, unit)
    if not castBar._psEnhanced or castBar:IsForbidden() then return end

    local spellName, _, _, startTimeMS, endTimeMS, _, _, notInterruptible =
        UnitCastingInfo(unit)
    if not spellName then
        spellName, _, _, startTimeMS, endTimeMS, _, notInterruptible =
            UnitChannelInfo(unit)
    end

    if spellName then
        castBar._psSpellText:SetText(spellName)
        castBar._psSpellText:Show()

        if notInterruptible then
            castBar:SetStatusBarColor(unpack(NON_INTERRUPTIBLE_COLOR))
        else
            castBar:SetStatusBarColor(unpack(INTERRUPTIBLE_COLOR))
        end

        if not castBar._psOnUpdate then
            castBar:HookScript("OnUpdate", function(self)
                if not self._psTimeText then return end
                local _, _, _, sMS, eMS = UnitCastingInfo(unit)
                if not sMS then
                    _, _, _, sMS, eMS = UnitChannelInfo(unit)
                end
                if sMS and eMS then
                    local remaining = (eMS - GetTime() * 1000) / 1000
                    if remaining > 0 then
                        self._psTimeText:SetText(format("%.1fs", remaining))
                        self._psTimeText:Show()
                    else
                        self._psTimeText:Hide()
                    end
                else
                    self._psTimeText:Hide()
                end
            end)
            castBar._psOnUpdate = true
        end
    else
        castBar._psSpellText:Hide()
        castBar._psTimeText:Hide()
    end
end

hooksecurefunc("CompactUnitFrame_UpdateAll", function(frame)
    if frame:IsForbidden() then return end
    if not ns.db.castBarEnhance then return end
    if not frame.unit then return end

    local castBar = frame.CastBar or frame.castBar
    if castBar and not castBar:IsForbidden() then
        SetupCastBar(castBar)
        UpdateCastBar(castBar, frame.unit)
    end
end)
```

### Config.lua

```lua
local ADDON_NAME, ns = ...

local function InitSettings()
    local category = Settings.RegisterVerticalLayoutCategory(ADDON_NAME)

    local function AddToggle(variable, name, tooltip, default)
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable, ns.db, Settings.VarType.Boolean,
            name, default
        )
        Settings.CreateCheckbox(category, setting, tooltip)
    end

    AddToggle("classColors", "Class-Colored Health Bars",
        "Color player nameplates by class.", true)
    AddToggle("threatColors", "Threat-Based Colors",
        "Color nameplates by your current threat level.", true)
    AddToggle("castBarEnhance", "Enhanced Cast Bars",
        "Show spell names, cast time, and interrupt indicators.", true)
    AddToggle("mobIcons", "Mob Classification Icons",
        "Show elite, rare, and world boss icons.", true)
    AddToggle("auraIndicators", "Aura Indicators",
        "Show crowd control and defensive auras above nameplates.", true)

    Settings.RegisterAddOnCategory(category)
end

local f = CreateFrame("Frame")
f:RegisterEvent("PLAYER_LOGIN")
f:SetScript("OnEvent", function()
    InitSettings()
    f:UnregisterEvent("PLAYER_LOGIN")
end)
```

!!! tip "Build This Instantly"

    Want PlateStylist generated and customized for your needs? Use the AI scaffold command:

    ```
    /wow-create PlateStylist — Nameplate restyler with class colors, threat
    coloring, cast bar enhancement, mob icons, aura indicators with
    CROWD_CONTROL and BIG_DEFENSIVE filters, object pool pattern,
    GetRegions stripping, BackdropTemplate, Boundary Pusher mode
    ```

---

That's PlateStylist — a full nameplate restyler built with Boundary Pusher techniques, wrapped in safety checks, and targeting Midnight 12.0's new API surface. The key takeaways:

- **Hook, don't replace.** `hooksecurefunc` on `CompactUnitFrame_*` functions gives you persistent control over nameplate appearance without taint.
- **Boundary techniques need safety nets.** Every `GetRegions()` call, every noop pattern, every metatable hook is wrapped in `pcall` with a fallback path.
- **Object pools for per-nameplate widgets.** Never create frames in hot-path update functions. Pool and reuse.
- **New aura filters replace spell ID lists.** `CROWD_CONTROL` and `BIG_DEFENSIVE` give you curated aura categories without maintaining your own database.

Next steps: [Enhancement Tutorials](../enhancement-tutorials.md) for more build-alongs, or [Midnight Coding Patterns](../midnight-patterns.md) for the full API reference.
