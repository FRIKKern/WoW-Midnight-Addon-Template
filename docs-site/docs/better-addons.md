---
title: "Building Better Addons"
description: "The complete guide to enhancing Blizzard's UI in Patch 12.0+ — the Better addon ecosystem, Cooldown Manager, Edit Mode, Masque skinning, and how to build your own enhancement addon."
---

# Building Better Addons

Midnight changed the rules. Combat data flows through **Secret Values**, `COMBAT_LOG_EVENT_UNFILTERED` is gone, and addons that replace Blizzard's secure frames break under the new security model. But addons that *enhance* Blizzard's UI — reskinning frames, extending Edit Mode, customizing the Cooldown Manager — work better than ever.

This page covers the **"Better" addon philosophy**: the ecosystem of addons that enhance rather than replace, the Blizzard systems you can hook into, and step-by-step patterns for building your own enhancement addon in Patch 12.0+.

!!! tip "Prerequisites"
    This page assumes you've read [Coding for Midnight](midnight-patterns.md) for the core API patterns and Secret Values workflow. Here we focus on the *ecosystem* and *systems* — where to enhance and how others have done it.

---

## The "Better" Philosophy

From [Blizzard's combat philosophy blog post](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight):

> "In essence, combat events are in a black box; addons can change the size or shape of the box, and they can paint it a different color, but what they can't do is look inside the box."

This isn't just a design suggestion — it's **enforced by the game engine** in 12.0+. Addons that only modify presentation (skinning, layout, cosmetics) continue to work. Addons that process combat data or replace secure frames break.

The "Better" addon ecosystem emerged from this reality. The philosophy:

- **Preserve secure frame integrity** — hook after Blizzard code, never override
- **Work with Blizzard updates** — don't depend on internal frame structure
- **Use `hooksecurefunc()`, not overrides** — run *after* Blizzard code, adding behavior on top
- **Avoid taint** — reparent to hidden frames instead of calling `:Hide()` on protected frames
- **Defer combat-sensitive operations** — check `InCombatLockdown()` and retry via `C_Timer.After()`

---

## The Better Addon Ecosystem

A growing family of addons follow the "enhance, don't replace" pattern. Here are the key players for Midnight.

### True Enhancers

These addons modify Blizzard's existing frames without replacing them — the gold standard for Midnight compatibility.

| Addon | Downloads | What It Enhances | Midnight Ready |
|-------|-----------|------------------|----------------|
| [**BetterBlizzFrames**](https://www.curseforge.com/wow/addons/betterblizzframes) | 3.9M+ | Player, Target, Focus, Party, Raid frames | Yes — dedicated `midnight/` branch |
| [**BetterBlizzPlates**](https://www.curseforge.com/wow/addons/betterblizzplates) | — | Default nameplates | Yes — multi-expansion TOC |
| [**BetterAddonList**](https://www.curseforge.com/wow/addons/betteraddonlist) | 231K+ | `/addons` management list | Yes |
| [**Better Vendor Price**](https://www.curseforge.com/wow/addons/better-vendor-price) | — | Item tooltips (vendor pricing) | Unknown |
| [**Better Fishing**](https://www.curseforge.com/wow/addons/better-fishing) | 4.28M+ | Soft Targeting fishing hook | Likely |

### Hybrid Extensions

These extend existing Blizzard panels with new tabs, data, or functionality.

| Addon | Downloads | What It Extends |
|-------|-----------|-----------------|
| [**BetterWardrobe**](https://www.curseforge.com/wow/addons/better-wardrobe-and-transmog) | 8.2M+ | Transmog collection, dressing room |
| [**BetterCharacterPanel**](https://www.curseforge.com/wow/addons/bettercharacterpanel) | 551K+ | Character and Inspect panels |
| [**Mount Journal Enhanced**](https://www.curseforge.com/wow/addons/mount-journal-enhanced) | 3.7M+ | Mount collection sorting/filters |

### Full Replacements

These replace Blizzard frames entirely — a riskier pattern under Midnight, but some systems (like bags) are non-combat and remain viable.

| Addon | Downloads | What It Replaces |
|-------|-----------|-----------------|
| [**BetterBags**](https://www.curseforge.com/wow/addons/better-bags) | 3.6M+ | Bag frames (successor to AdiBags) |
| [**BetterFriendlist**](https://www.curseforge.com/wow/addons/betterfriendlist) | 56K+ | Friends list frame |

!!! warning "Full replacements and Midnight"
    Full replacement addons that target **non-combat** systems (bags, friends list, transmog) can still work in Midnight. Replacements that target **combat-adjacent** systems (unit frames, nameplates, action bars) face significant challenges under the stricter security model.

### "Enhance Don't Replace" Philosophy Addons

These addons are explicitly designed around the enhancement philosophy:

| Addon | Description |
|-------|-------------|
| [**BlizzFramesPlus**](https://www.curseforge.com/wow/addons/blizzframesplus) | Lightweight QoL for default unit frames. *"Not to replace, but to extend."* |
| [**Improved Blizzard UI**](https://www.curseforge.com/wow/addons/improved-blizzard-ui) | General improvements — styling, functionality, restructuring |
| [**ClassUIEnhanced**](https://www.curseforge.com/wow/addons/classuienhanced) | Class cooldowns, resources, cast bars in one anchored layout |
| [**MidnightUI**](https://www.curseforge.com/wow/addons/midnightui-midnight-ready) | Built from scratch for Midnight constraints — combat-safe, taint-aware |
| [**UnitFramesImproved**](https://www.curseforge.com/wow/addons/unitframesimproved) | Better stat visibility on Player and Target frames |

---

## Blizzard Systems You Can Enhance

Midnight gives you six major systems with official or semi-official extension points.

### Cooldown Manager (11.1.5+)

Blizzard's built-in cooldown tracking system — the primary sanctioned way to track combat cooldowns in Midnight.

- **Disabled by default** — enable in Options > Advanced Options
- Requires level 10+
- Categories: Essential, Utility, Hidden by Default (cooldowns) and Icons, Bars, Hidden by Default (buffs)
- Positioned in Edit Mode alongside other HUD elements
- Uses `EditModeCooldownViewerSystemMixin`
- **11.2.5 (Oct 2025):** Added drag-and-drop reordering, category reassignment, per-spec display

!!! info "Why the Cooldown Manager matters"
    With combat log events gone and spell cooldown data returning Secret Values, the Cooldown Manager is the **officially supported way** for players to track cooldowns. Enhancement addons that customize its appearance and extend its functionality are the new "WeakAuras for cooldowns."

### Edit Mode (10.0+)

Allows players to rearrange HUD elements. No official addon registration API exists, but community libraries fill the gap.

| Library | Author | Approach |
|---------|--------|----------|
| [**EditModeExpanded**](https://github.com/teelolws/EditModeExpanded) | teelolws | `lib:RegisterFrame(frame, name, db)` — extensive registration |
| **LibEditMode** | p3lim | `lib:AddFrame(frame, callback, default, name)` — parallel selection system |
| **LibEditModeOverride** | plusmouse | Programmatic via `C_EditMode.GetLayouts()`/`SaveLayouts()` |

Key events for Edit Mode integration:

```lua
-- Detect when Edit Mode is entered/exited
EventRegistry:RegisterCallback("EditMode.Enter", function()
    -- Show your addon's Edit Mode controls
end)

EventRegistry:RegisterCallback("EditMode.Exit", function()
    -- Hide Edit Mode controls, apply final positions
end)
```

### Settings Panel (10.0+ / 11.0.2+)

The modern addon settings system with auto-layout controls.

```lua
-- Register a settings category
local category = Settings.RegisterVerticalLayoutCategory("My Addon")

-- Add controls (11.0.2+ signatures)
local setting = Settings.RegisterAddOnSetting(category,
    "MyAddon_Toggle", "toggle", MyAddon_DB,
    type(false), "Enable Feature", false
)
Settings.CreateCheckbox(category, setting, "Enables the main feature.")

Settings.RegisterAddOnCategory(category)
```

### Addon Compartment (10.1.0+)

Unified minimap dropdown — register via TOC metadata for zero-code integration.

```toc
## AddonCompartmentFunc: MyAddon_OnClick
## AddonCompartmentFuncOnEnter: MyAddon_OnEnter
## AddonCompartmentFuncOnLeave: MyAddon_OnLeave
## IconTexture: Interface\Icons\INV_Misc_QuestionMark
```

### Action Bar System (12.0+)

Consolidated into the `C_ActionBar` namespace. The cooldown frame on `ActionButtonTemplate` was split into three children: `cooldown`, `chargeCooldown`, and `lossOfControlCooldown`.

### ScrollBox / DataProvider

The modern scrolling system (replacing `FauxScrollFrame` and `HybridScrollFrame`). Used by BetterAddonList and others to extend Blizzard's list frames.

---

## The Enhancement Philosophy in Practice

Three distinct hooking strategies emerge across the ecosystem. Understanding these patterns is essential for building Midnight-compatible addons.

### Pattern A: Enhancement/Overlay (Recommended)

Used by BetterBlizzFrames, ActionBarsEnhanced, BetterBlizzPlates.

```
┌─────────────────────────────────────┐
│ Blizzard Frame (unchanged)          │
│  └─ hooksecurefunc runs AFTER       │
│     └─ Addon modifies visual props  │
│        (color, texture, visibility) │
│     └─ Addon adds child frames      │
│        (castbars, indicators)       │
└─────────────────────────────────────┘
```

- Uses `hooksecurefunc()` exclusively — never pre-hooks
- Stores original state for restoration
- Defers all operations out of combat lockdown
- Never touches secure frame hierarchy
- **Most Midnight-compatible pattern**

### Pattern B: Complete Replacement

Used by BetterBags, BetterFriendlist. Riskier under Midnight.

```
┌─────────────────────────────────────┐
│ Hidden "Sneaky" Frame               │
│  └─ Blizzard Frame (reparented)     │
│                                     │
│ Addon Replacement Frame (visible)   │
│  └─ Intercepts keybinds             │
│  └─ Hooks bag bar buttons           │
└─────────────────────────────────────┘
```

- Reparents Blizzard frames to hidden frame (avoids `:Hide()` taint)
- Overrides keybinds via `SetOverrideBinding()`
- Uses deferred flag pattern (boolean flags processed in OnUpdate)

### Pattern C: CDM/Edit Mode Extension (Ideal for Midnight)

Used by ArcUI, CDMx, Edit Mode Expanded.

```
┌─────────────────────────────────────┐
│ Blizzard System (CDM / Edit Mode)   │
│  └─ Addon registers with system     │
│     └─ Adds conditions/groups       │
│     └─ Extends configuration        │
│     └─ Anchors frames relative to   │
│        system frames                │
└─────────────────────────────────────┘
```

- Registers with Blizzard's built-in systems
- Extends condition logic (new group conditions)
- Provides library APIs for other addons
- **Uses Blizzard's own extension points**

---

## Cooldown Manager Addons

The Cooldown Manager (CDM) has become the most actively enhanced Blizzard system in Midnight. Here's the ecosystem and how these addons work.

### The CDM Addon Landscape

| Addon | Focus | Key Feature |
|-------|-------|-------------|
| [**BetterCooldownManager**](https://www.curseforge.com/wow/addons/bettercooldownmanager) | All-in-one | Pixel borders, power bars, custom spell/trinket bars, anchor API |
| [**ArcUI**](https://www.curseforge.com/wow/addons/arc-ui) | Full CDM + extras | DoT bars, charge tracking, 7 new group conditions, 4 resource styles |
| [**Cooldown Manager Centered**](https://www.curseforge.com/wow/addons/cooldown-manager-centered) | Layout | Centered icons, rotation assist, custom aura swipes |
| [**CDMx**](https://www.curseforge.com/wow/addons/cooldown-manager-extended-cdmx) | Keybinds + trinkets | Auto keybind display, trinket auto-detection, Masque support |
| [**Enhanced Cooldown Manager**](https://www.curseforge.com/wow/addons/enhanced-cooldown-manager) | Power + auras | Power bar, aura bars anchored below CDM |
| [**CooldownManagerCustomizer**](https://www.curseforge.com/wow/addons/cooldownmanagercustomizer) | Show/hide | Per-SpellID visibility control |
| [**Cooldown Manager Control**](https://www.curseforge.com/wow/addons/cooldown-manager-control) | Layout control | Per-container orientation and growth direction |
| [**Cooldown Forge**](https://www.curseforge.com/wow/addons/cooldown-forge) | Profiles | Profile management for CDM per character/spec |

### How Cooldown Text Addons Work in Midnight

Traditional cooldown text addons like OmniCC and tullaCooldownCount overlay text on Blizzard's `Cooldown` widget. In Midnight, cooldown duration data can be secret — so these addons had to adapt.

**tullaCTC** (Cooldown Text Customizer) shows the approach:

```lua
local addonName, ns = ...

-- tullaCTC's Midnight adaptation:
-- 1. Hook Blizzard's Cooldown widget to intercept SetCooldown calls
-- 2. Use Duration Provider API for secret-safe duration access
-- 3. Apply conditional coloring via Rules API

-- Hook the cooldown widget's countdown FontString
hooksecurefunc(CooldownFrameMixin, "SetCooldown", function(self, start, duration)
    if self:IsForbidden() then return end

    -- Get the built-in countdown text element (12.0+)
    local countdown = self:GetCountdownFontString()
    if not countdown then return end

    -- Restyle the countdown text
    countdown:SetFont("Interface\\AddOns\\" .. addonName .. "\\Fonts\\Main.ttf",
        ns.db.fontSize, "OUTLINE")
    countdown:SetTextColor(ns:GetColorForDuration(duration))
end)
```

!!! info "Duration Objects replace raw timing"
    In 12.0+, use `C_DurationUtil.CreateDuration()` and `Cooldown:SetCooldownFromDurationObject()` for Secret Values-compatible cooldown display:

    ```lua
    local duration = C_DurationUtil.CreateDuration()

    -- New spell cooldown APIs return Duration objects
    local cooldownDuration = C_Spell.GetSpellCooldownDuration(spellID)

    -- Pass directly to the Cooldown widget — no math needed
    cooldownFrame:SetCooldownFromDurationObject(cooldownDuration)
    ```

### Anchoring Custom Frames to the CDM

The pattern used by Enhanced Cooldown Manager, BetterCooldownManager, and others:

```lua
local addonName, ns = ...

-- Wait for the Cooldown Manager to load
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, addon)
    if addon == "Blizzard_CooldownViewer" then
        self:UnregisterEvent("ADDON_LOADED")
        ns:SetupCDMExtension()
    end
end)

function ns:SetupCDMExtension()
    -- Create a custom bar anchored below the Cooldown Manager
    local powerBar = CreateFrame("StatusBar", nil, UIParent)
    powerBar:SetSize(200, 8)
    powerBar:SetStatusBarTexture("Interface\\TargetingFrame\\UI-StatusBar")
    powerBar:SetStatusBarColor(0.0, 0.4, 1.0)
    powerBar:SetMinMaxValues(0, 1)

    -- Anchor relative to the CDM frame
    -- CooldownViewerFrame is the parent of the CDM display
    if CooldownViewerFrame then
        powerBar:SetPoint("TOP", CooldownViewerFrame, "BOTTOM", 0, -4)
    end

    -- Background
    local bg = powerBar:CreateTexture(nil, "BACKGROUND")
    bg:SetAllPoints()
    bg:SetColorTexture(0, 0, 0, 0.6)

    -- Update with player power (safe — secondary resources are whitelisted)
    local updater = CreateFrame("Frame")
    updater:RegisterUnitEvent("UNIT_POWER_UPDATE", "player")
    updater:SetScript("OnEvent", function()
        local pct = UnitPowerPercent("player")
        powerBar:SetValue(pct)
    end)
end
```

---

## Edit Mode Integration

Edit Mode has no official addon registration API. Calling `EditModeManagerFrame:RegisterSystemFrame()` from addon code causes taint. Community libraries solve this.

### Using EditModeExpanded

[EditModeExpanded](https://github.com/teelolws/EditModeExpanded) (1.3M+ downloads) provides a library API that other addons can use to make their frames movable in Edit Mode.

```lua
local addonName, ns = ...

-- Embed the library
local EME = LibStub("EditModeExpanded-1.0")

-- Create your addon's display frame
local display = CreateFrame("Frame", "MyAddonDisplay", UIParent)
display:SetSize(200, 100)
display:SetPoint("CENTER")

-- Register with Edit Mode Expanded
-- This makes the frame draggable in Edit Mode with per-profile saved positions
EME:RegisterFrame(display, "My Addon Display", ns.db)
```

### Manual Edit Mode Awareness

If you don't want a library dependency, you can still react to Edit Mode state:

```lua
local addonName, ns = ...

local myFrame = CreateFrame("Frame", "MyAddonFrame", UIParent,
    "BackdropTemplate")
myFrame:SetSize(150, 40)
myFrame:SetPoint("CENTER")
myFrame:SetBackdrop({
    bgFile = "Interface\\Buttons\\WHITE8X8",
    edgeFile = "Interface\\Buttons\\WHITE8X8",
    edgeSize = 1,
})
myFrame:SetBackdropColor(0, 0, 0, 0.7)
myFrame:SetBackdropBorderColor(0.3, 0.3, 0.3)

-- Make it draggable
myFrame:SetMovable(true)
myFrame:EnableMouse(true)
myFrame:RegisterForDrag("LeftButton")
myFrame:SetScript("OnDragStart", myFrame.StartMoving)
myFrame:SetScript("OnDragStop", function(self)
    self:StopMovingOrSizing()
    -- Save position
    local point, _, relPoint, x, y = self:GetPoint()
    ns.db.position = { point, relPoint, x, y }
end)

-- Show selection glow during Edit Mode
local selectionGlow = myFrame:CreateTexture(nil, "OVERLAY")
selectionGlow:SetAllPoints()
selectionGlow:SetColorTexture(0, 0.5, 1, 0.15)
selectionGlow:Hide()

EventRegistry:RegisterCallback("EditMode.Enter", function()
    selectionGlow:Show()
    myFrame:EnableMouse(true)
end)

EventRegistry:RegisterCallback("EditMode.Exit", function()
    selectionGlow:Hide()
end)
```

### Edit Mode Addon Ecosystem

| Addon | Downloads | Focus |
|-------|-----------|-------|
| [**Edit Mode Expanded**](https://www.curseforge.com/wow/addons/edit-mode-expanded) | 1.3M+ | Register additional frames, resize support, per-profile positions |
| [**Edit Mode Tweaks**](https://www.curseforge.com/wow/addons/edit-mode-tweaks) | — | 1-pixel nudge, click-to-select, visibility conditions |
| [**Edit Mode Features**](https://www.curseforge.com/wow/addons/edit-mode-features) | — | Grid snapping, element linking, coordinate display |

---

## UI Skinning Addons

Skinning — changing how Blizzard's UI *looks* without changing how it *works* — is the safest form of addon enhancement and the most Midnight-compatible.

### Masque: The Button Skinning Framework

[Masque](https://github.com/SFX-WoW/Masque) is a framework that reskins button widgets (action bar buttons, inventory slots, etc.) without touching their functionality. Skin authors create texture packs; bridge addons connect Masque to specific Blizzard buttons.

| Bridge Addon | What It Skins |
|-------------|---------------|
| [**Masque Blizzard Bars**](https://github.com/kstange/MasqueBlizzBars) | Action Bars 1-8, Pet, Vehicle, Stance, **CDM trackers** |
| [**Masque Blizzard Inventory**](https://github.com/kstange/MasqueBlizzInv) | Backpack, Bank, Guild Bank, Mail, Merchants, Loot |

!!! tip "Masque and the Cooldown Manager"
    Masque Blizzard Bars (v12.0.1.2) supports skinning CDM trackers — Essential, Utility, Tracked Bars, and Tracked Buffs. Each is an independent Masque group, so players can theme cooldown icons separately from action bar buttons.

### Frame Colorization

| Addon | What It Does |
|-------|-------------|
| [**FrameColor**](https://www.curseforge.com/wow/addons/framecolor-hud-action-bar-color) | Colorizes action bars, HUD elements, windows — modular, supports Bartender4/Dominos |
| [**Platynator**](https://www.curseforge.com/wow/addons/platynator) | Visual nameplate designer — click to customize, drag to reposition. Built for Midnight |
| [**Dark Theme UI**](https://github.com/NYKO7/World-Of-Warcraft-Dark-Theme-UI-Texture-Pack) | Texture replacement pack (not an addon — installed to `/Interface/` folder) |

### The BetterBlizzFrames Approach

BetterBlizzFrames ([GitHub](https://github.com/Bodify/BetterBlizzFrames)) is the gold standard for frame enhancement. Its techniques are worth studying:

=== "Hook Global Functions"

    ```lua
    -- Run code AFTER Blizzard updates a frame
    hooksecurefunc("CompactUnitFrame_UpdateRoleIcon", function(frame)
        if frame:IsForbidden() then return end
        -- Modify the role icon appearance
    end)

    hooksecurefunc("CompactUnitFrame_UpdateHealPrediction", function(frame)
        if frame:IsForbidden() then return end
        -- Customize heal prediction visuals
    end)
    ```

=== "Recursion Guard"

    ```lua
    -- Prevent infinite loops when hooking methods that
    -- you also call inside the hook
    hooksecurefunc(frame, "SetVertexColor", function(self)
        if self.changing then return end
        self.changing = true
        self:SetDesaturated(true)
        self:SetVertexColor(0.4, 0.4, 0.4)
        self.changing = false
    end)
    ```

=== "Hidden Frame Reparenting"

    ```lua
    -- Hide Blizzard elements WITHOUT calling :Hide()
    -- (which would cause taint on protected frames)
    local hiddenFrame = CreateFrame("Frame")
    hiddenFrame:Hide()

    -- Store original state for restoration
    frame.ogPoint = { frame:GetPoint() }
    frame.ogParent = frame:GetParent()

    -- Reparent to hide
    frame:SetParent(hiddenFrame)
    ```

=== "Combat Deferral"

    ```lua
    -- Never modify protected frames during combat
    local function SafeModify(frame, callback)
        if InCombatLockdown() then
            -- Retry after combat ends
            local waiter = CreateFrame("Frame")
            waiter:RegisterEvent("PLAYER_REGEN_ENABLED")
            waiter:SetScript("OnEvent", function(self)
                self:UnregisterAllEvents()
                callback(frame)
            end)
        else
            callback(frame)
        end
    end
    ```

---

## Getting Started: Building a "Better" Addon

Here's a step-by-step walkthrough for building a Midnight-compatible enhancement addon using the namespace pattern.

### Step 1: TOC File

```toc
## Interface: 120001
## Title: BetterExample
## Notes: Enhances Blizzard's Player Frame with class-colored health bars and dark mode
## Author: YourName
## Version: 1.0.0
## SavedVariables: BetterExampleDB
## AddonCompartmentFunc: BetterExample_OnCompartmentClick
## IconTexture: Interface\Icons\INV_Misc_Gear_01

Init.lua
Core.lua
```

### Step 2: Initialization (Init.lua)

```lua
local addonName, ns = ...

-- Defaults for SavedVariables
ns.defaults = {
    classColorHealth = true,
    darkMode = true,
    healthBarTexture = "Interface\\TargetingFrame\\UI-StatusBar",
    scale = 1.0,
}

-- Global compartment click handler
function BetterExample_OnCompartmentClick()
    Settings.OpenToCategory(ns.categoryID)
end

-- Wait for SavedVariables to load
local loader = CreateFrame("Frame")
loader:RegisterEvent("ADDON_LOADED")
loader:SetScript("OnEvent", function(self, event, addon)
    if addon ~= addonName then return end
    self:UnregisterEvent("ADDON_LOADED")

    -- Initialize SavedVariables with defaults
    if not BetterExampleDB then
        BetterExampleDB = {}
    end
    for k, v in pairs(ns.defaults) do
        if BetterExampleDB[k] == nil then
            BetterExampleDB[k] = v
        end
    end
    ns.db = BetterExampleDB

    -- Register settings
    ns:RegisterSettings()

    -- Apply enhancements
    ns:ApplyEnhancements()
end)
```

### Step 3: Core Enhancement Logic (Core.lua)

```lua
local addonName, ns = ...

function ns:RegisterSettings()
    local category = Settings.RegisterVerticalLayoutCategory("BetterExample")
    ns.categoryID = category:GetID()

    -- Class-colored health toggle
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BetterExample_ClassColor", "classColorHealth", ns.db,
            type(true), "Class-Colored Health Bars", true
        )
        Settings.CreateCheckbox(category, setting,
            "Color health bars by class color on Player and Target frames.")
        setting:SetValueChangedCallback(function()
            ns:RefreshFrames()
        end)
    end

    -- Dark mode toggle
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BetterExample_DarkMode", "darkMode", ns.db,
            type(true), "Dark Mode", true
        )
        Settings.CreateCheckbox(category, setting,
            "Darken frame backgrounds and borders.")
        setting:SetValueChangedCallback(function()
            ns:RefreshFrames()
        end)
    end

    -- Scale slider
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BetterExample_Scale", "scale", ns.db,
            type(1.0), "Frame Scale", 1.0
        )
        local options = Settings.CreateSliderOptions(0.5, 2.0, 0.05)
        options:SetLabelFormatter(MinimalSliderWithSteppersMixin.Label.Right)
        Settings.CreateSlider(category, setting, options,
            "Adjust the scale of enhanced frames.")
        setting:SetValueChangedCallback(function()
            ns:RefreshFrames()
        end)
    end

    Settings.RegisterAddOnCategory(category)
end

function ns:ApplyEnhancements()
    local CLASS_COLORS = RAID_CLASS_COLORS

    -- Hook PlayerFrame health bar updates
    hooksecurefunc(PlayerFrame.PlayerFrameContent.PlayerFrameContentMain
        .HealthBarsContainer.HealthBar, "UpdateFillBar", function(self)
        if self:IsForbidden() then return end
        if not ns.db.classColorHealth then return end

        local _, class = UnitClass("player")
        local color = CLASS_COLORS[class]
        if color then
            self:SetStatusBarColor(color.r, color.g, color.b)
        end
    end)

    -- Hook TargetFrame health bar
    hooksecurefunc("TargetFrame_UpdateAuras", function(self)
        if self:IsForbidden() then return end
        if not ns.db.classColorHealth then return end

        local unit = self.unit
        if not unit or not UnitIsPlayer(unit) then return end

        local _, class = UnitClass(unit)
        local color = CLASS_COLORS[class]
        if color and self.HealthBar then
            self.HealthBar:SetStatusBarColor(color.r, color.g, color.b)
        end
    end)

    -- Dark mode: hook frame texture updates
    if ns.db.darkMode then
        ns:ApplyDarkMode()
    end
end

function ns:ApplyDarkMode()
    -- Darken the player frame background
    local frames = { PlayerFrame, TargetFrame, FocusFrame }
    for _, unitFrame in ipairs(frames) do
        if unitFrame and not unitFrame:IsForbidden() then
            -- Add a dark overlay behind the frame
            if not unitFrame.__darkOverlay then
                local overlay = unitFrame:CreateTexture(nil, "BACKGROUND", nil, -8)
                overlay:SetAllPoints()
                overlay:SetColorTexture(0, 0, 0, 0.3)
                unitFrame.__darkOverlay = overlay
            end
        end
    end
end

function ns:RefreshFrames()
    -- Re-apply enhancements when settings change
    -- Combat-safe: visual-only operations
    if PlayerFrame and not PlayerFrame:IsForbidden() then
        if ns.db.darkMode then
            if PlayerFrame.__darkOverlay then
                PlayerFrame.__darkOverlay:Show()
            end
        else
            if PlayerFrame.__darkOverlay then
                PlayerFrame.__darkOverlay:Hide()
            end
        end

        PlayerFrame:SetScale(ns.db.scale)
    end

    if TargetFrame and not TargetFrame:IsForbidden() then
        TargetFrame:SetScale(ns.db.scale)
    end
end
```

### Step 4: Key Techniques Summary

!!! abstract "Enhancement Addon Checklist"
    1. **Always `hooksecurefunc()`** — never pre-hook or override Blizzard functions
    2. **Always check `IsForbidden()`** — some frames are off-limits
    3. **Store original state** — `frame.ogPoint = {frame:GetPoint()}`
    4. **Use recursion guards** — `if self.changing then return end`
    5. **Defer out of combat** — `InCombatLockdown()` + `C_Timer.After()` retry
    6. **Reparent, don't `:Hide()`** — use hidden frame reparenting for protected frames
    7. **Register with Blizzard systems** — Settings Panel, Addon Compartment, Edit Mode
    8. **Use Duration Objects** — `C_DurationUtil.CreateDuration()` for cooldown data
    9. **Use the namespace pattern** — `local addonName, ns = ...` for clean encapsulation
    10. **Target Interface `120001`** — addons lacking `120000`+ won't load without override

---

## Midnight Compatibility Matrix

| Category | Addon | Status | Notes |
|----------|-------|--------|-------|
| **Unit Frames** | BetterBlizzFrames | :white_check_mark: | Dedicated `midnight/` branch, 12 modules |
| **Nameplates** | BetterBlizzPlates | :white_check_mark: | Multi-expansion TOC |
| **Nameplates** | Platynator | :white_check_mark: | Built from scratch for Midnight |
| **Bags** | BetterBags | :white_check_mark: | Fixed `EquipmentManager_UnpackLocation` removal |
| **Addon List** | BetterAddonList | :white_check_mark: | Interface 120001 |
| **CDM** | BetterCooldownManager | :white_check_mark: | Explicitly Midnight-ready |
| **CDM** | ArcUI | :white_check_mark: | Highlighted by Wowhead |
| **CDM** | CDMx | :white_check_mark: | Masque + Edit Mode integration |
| **CDM Text** | tullaCTC | :white_check_mark: | Rewritten for Secrets API |
| **Edit Mode** | Edit Mode Expanded | :white_check_mark: | v12.0-027 |
| **Action Bars** | ActionBarsEnhanced | :white_check_mark: | v2.4.6 |
| **Skinning** | Masque Blizzard Bars | :white_check_mark: | v12.0.1.2 |
| **Skinning** | Masque Blizzard Inventory | :white_check_mark: | v12.0.0.0 |
| **Skinning** | FrameColor | :white_check_mark: | v4.5.0+ Midnight beta |
| **UI Suite** | MidnightUI | :white_check_mark: | Built for Midnight |
| **UI Suite** | ClassUIEnhanced | :white_check_mark: | 12.0.1 support |
| **Raid Frames** | Enhanced Raid Frames | :x: | **Archived** — developer confirmed incompatibility |

---

## Further Reading

- [Coding for Midnight: The New Paradigm](midnight-patterns.md) — Core API patterns, Secret Values, code examples
- [Cooldown Manager UI Guide (Wowhead)](https://www.wowhead.com/guide/ui/cooldown-manager-setup)
- [Useful Add-Ons to Customize the Cooldown Manager in Midnight (Wowhead)](https://www.wowhead.com/news/useful-add-ons-to-customize-the-cooldown-manager-with-in-midnight-379921)
- [Combat Philosophy and Addon Disarmament (Blizzard)](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight)
- [WoW's Game Director on Addon Philosophy (PC Gamer)](https://www.pcgamer.com/games/world-of-warcraft/wows-game-director-says-they-dont-want-combat-addons-to-do-anything-the-base-ui-doesnt-the-overarching-goal-of-the-changes-in-midnight-is-to-level-the-playing-field/)
- [Patch 12.0.0 API Changes (Warcraft Wiki)](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes)
- [Settings API (Warcraft Wiki)](https://warcraft.wiki.gg/wiki/Settings_API)
- [Addon Compartment (Warcraft Wiki)](https://warcraft.wiki.gg/wiki/Addon_compartment)
