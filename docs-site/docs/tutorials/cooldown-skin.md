---
title: "Build-Along: BetterCooldowns — Cooldown Manager Skin"
description: Build an OmniCC-style cooldown text addon using Duration Objects, hooksecurefunc, and the Enhancement Artist philosophy for WoW Midnight 12.0+.
---

# Build-Along: BetterCooldowns

:material-clock-outline: **Estimated time:** 30 minutes
:material-signal-cellular-3: **Difficulty:** Intermediate-Advanced
:material-palette: **Mode:** Enhancement Artist
:material-tag: **Interface:** 120001 (Midnight 12.0.1)

---

## Step 1: What We're Building

BetterCooldowns adds OmniCC-style countdown text to Blizzard's native cooldown spirals. Instead of replacing Blizzard's cooldown system, we **overlay** it — adding readable countdown text, smooth color transitions, and a pulse animation when abilities are about to come off cooldown.

Here is what the finished addon does:

- **Countdown text** on every cooldown frame — action bars, inventory, pet bar, everything
- **Color transitions** — green text above 30 seconds, yellow from 10-30 seconds, red under 10 seconds
- **Decimal precision** — shows "5.1" for the final 5 seconds, "30s" for longer, "2m" for minutes
- **Pulse animation** — a subtle scale bounce when cooldowns drop below 3 seconds
- **Fully configurable** — toggle, minimum duration threshold, font size, pulse on/off

The Enhancement Artist philosophy guides every decision. We hook Blizzard's cooldown system with `hooksecurefunc`, add our FontString on top of existing frames, and let Blizzard handle the actual cooldown tracking. We never call `SetCooldown` ourselves, we never replace any frame, and we never touch protected state during combat.

### What You Will Learn

| Concept | Where |
|---------|-------|
| Duration Objects (new in 12.0) | Step 2 |
| `SetCooldownFromDurationObject()` | Step 2 |
| `hooksecurefunc` on Cooldown methods | Step 4 |
| `IsForbidden()` guards on secure frames | Step 4 |
| Throttled `OnUpdate` handlers | Step 5 |
| `AnimationGroup` with `ScaleAnimation` | Step 6 |
| Action bar button enumeration | Step 7 |
| Combat lockdown queue pattern | Step 7, Step 9 |
| Modern Settings API | Step 8 |

!!! tip "Enhancement Artist pattern"
    This tutorial follows the Enhancement Artist mode's core rule: **skin, hook, extend — never replace**. Blizzard tracks the cooldowns. Blizzard draws the spiral. We just add text on top.

---

## Step 2: Understanding Duration Objects

Duration Objects are one of the most important new concepts in WoW 12.0. They wrap time-based combat data — spell cooldowns, buff durations, GCD timers — into opaque containers that UI widgets can consume directly.

!!! info "Duration Objects in 12.0"
    Duration Objects were introduced alongside Secret Values to solve a specific problem: how can addons display cooldown spirals when the raw start time and duration might be secret? The answer is a sealed object that Blizzard's cooldown widget knows how to render, without exposing the underlying numbers to addon code.

### How Duration Objects Work

A Duration Object is obtained from API functions like `C_Spell.GetSpellCooldownDuration()`. It cannot be opened, inspected, or used in math — it is a sealed container. But it can be passed directly to a Cooldown frame:

```lua
local durationObj = C_Spell.GetSpellCooldownDuration(spellID)
cooldownFrame:SetCooldownFromDurationObject(durationObj)
```

The Cooldown frame reads the duration internally and renders the spiral animation. Your addon code never sees the raw numbers. This is the Secret Values system at work — **display is permitted, logic is not**.

### What This Means for BetterCooldowns

Our addon hooks `SetCooldown` and `SetCooldownFromDurationObject` — both are called by Blizzard when a cooldown starts. In the open world, `SetCooldown(start, duration)` gives us real numbers we can use for text formatting. In instances, the same frame might receive a Duration Object instead, and the `start` and `duration` arguments to `SetCooldown` may be Secret Values.

=== "Open World"

    ```lua
    -- SetCooldown called with real numbers
    -- start = 145892.456, duration = 30
    -- We CAN do math: remaining = duration - (GetTime() - start)
    -- Our countdown text works perfectly
    ```

=== "Inside Instances"

    ```lua
    -- SetCooldown called — but start/duration may be Secret Values
    -- We CANNOT do math on secrets
    -- Fallback: hide our text, let Blizzard's spiral speak for itself
    -- The spiral still works because it receives a Duration Object internally
    ```

!!! warning "The golden rule"
    Duration Objects are for **display** — pass them to widgets. They are not for **logic** — you cannot read their values, do arithmetic, or branch on them. Our addon must detect when it is inside a restricted context and gracefully hide its countdown text.

### The Dual-Path Architecture

BetterCooldowns uses a simple dual-path approach:

1. **Unrestricted** (open world, non-encounter): read `start` and `duration` from `SetCooldown`, compute remaining time, display text
2. **Restricted** (instances, encounters, M+, PvP): detect secrets with `issecretvalue()`, hide our countdown text, let Blizzard's native spiral handle display

This graceful degradation is the hallmark of a well-built Midnight addon.

---

## Step 3: Project Setup

Create the following directory structure:

```
BetterCooldowns/
    BetterCooldowns.toc
    Core.lua
    Display.lua
    Config.lua
```

### BetterCooldowns.toc

```toc
## Interface: 120001
## Title: BetterCooldowns
## Notes: Countdown text overlay for Blizzard cooldown frames
## Author: You
## Version: 1.0.0
## SavedVariables: BetterCooldownsDB
## IconTexture: Interface\Icons\Spell_Holy_BorrowedTime
## Category: Combat

Core.lua
Display.lua
Config.lua
```

The file load order matters. `Core.lua` initializes the namespace and event system. `Display.lua` contains the cooldown hook and text rendering. `Config.lua` adds the Settings panel.

!!! tip "Icon choice"
    `Spell_Holy_BorrowedTime` is a clock icon — fitting for a cooldown addon. Browse icons at wowhead.com/icons or use `/dump GetSpellTexture(spellID)` in-game.

---

## Step 4: Hooking Blizzard's Cooldown System

This is the core of the addon. We hook into Blizzard's cooldown pipeline so that every time a cooldown starts, our code runs afterward to add countdown text.

### Core.lua — Namespace and Events

```lua
-- Mode: Enhancement Artist | Enhance don't replace
local addonName, ns = ...

-- Defaults
ns.defaults = {
    enabled = true,
    minDuration = 2.5,
    fontSize = 14,
    pulseEnabled = true,
    pulseThreshold = 3,
}

-- Event dispatch
local eventHandlers = {}

function eventHandlers:ADDON_LOADED(loadedAddon)
    if loadedAddon ~= addonName then return end

    -- Initialize SavedVariables with defaults
    if not BetterCooldownsDB then
        BetterCooldownsDB = CopyTable(ns.defaults)
    else
        for k, v in pairs(ns.defaults) do
            if BetterCooldownsDB[k] == nil then
                BetterCooldownsDB[k] = v
            end
        end
    end
    ns.db = BetterCooldownsDB

    self:UnregisterEvent("ADDON_LOADED")
end

function eventHandlers:PLAYER_LOGIN()
    ns:InitDisplay()
end

local eventFrame = CreateFrame("Frame")
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = eventHandlers[event]
    if handler then
        handler(self, ...)
    end
end)
for event in pairs(eventHandlers) do
    eventFrame:RegisterEvent(event)
end
```

### Display.lua — The Cooldown Hook

This is where the Enhancement Artist philosophy comes alive. We hook `SetCooldown` on the Cooldown frame metatable — one hook that catches **every** cooldown frame in the entire UI.

```lua
-- Mode: Enhancement Artist | Enhance don't replace
local addonName, ns = ...

-- Active cooldown overlays
local activeFrames = {}

-- ================================================================
-- Cooldown Metatable Hook
-- ================================================================

-- Get the shared metatable for all Cooldown frames
local tempCD = CreateFrame("Cooldown", nil, nil, "CooldownFrameTemplate")
local Cooldown_MT = getmetatable(tempCD).__index
tempCD = nil -- discard the temp frame

local function OnCooldownSet(self, start, duration)
    -- CRITICAL: Never touch forbidden frames
    if self:IsForbidden() then return end

    -- Skip disabled or short cooldowns (GCD, etc.)
    if not ns.db or not ns.db.enabled then return end
    if duration and not issecretvalue(duration) and duration < ns.db.minDuration then
        -- Hide any existing overlay
        if self.BetterCD then
            self.BetterCD:Hide()
        end
        return
    end

    -- Secret Values check: if start or duration is secret, hide our text
    if issecretvalue(start) or issecretvalue(duration) then
        if self.BetterCD then
            self.BetterCD:Hide()
        end
        return
    end

    -- Create or retrieve our overlay
    if not self.BetterCD then
        self.BetterCD = ns:CreateOverlay(self)
    end

    -- Store timing data for OnUpdate
    local overlay = self.BetterCD
    overlay.start = start
    overlay.duration = duration
    overlay.endTime = start + duration
    overlay:Show()

    -- Track for cleanup
    activeFrames[self] = true
end

-- Hook with hooksecurefunc — runs AFTER Blizzard's SetCooldown
hooksecurefunc(Cooldown_MT, "SetCooldown", OnCooldownSet)

function ns:InitDisplay()
    -- Hook is already installed via the metatable.
    -- This function exists for future initialization if needed.
end
```

!!! danger "IsForbidden() is not optional"
    Some cooldown frames belong to secure Blizzard UI elements. Accessing properties on a forbidden frame crashes the client. The `IsForbidden()` check on line 3 of `OnCooldownSet` is **mandatory** — remove it and you will get hard crashes in dungeons.

!!! tip "Why metatable hook instead of individual hooks?"
    Hooking the metatable's `SetCooldown` catches every Cooldown frame created anywhere in the UI — action bars, inventory, pet bar, other addons. No need to enumerate individual frames. This is the same technique ElvUI and OmniCD use.

### The Recursion Guard Question

In this case, we do not need a recursion guard. Our hook only reads data and creates overlay frames — it never calls `SetCooldown` itself. If your hook called a method on the same frame that triggers the hook, you would need the guard:

```lua
-- NOT needed here, but shown for reference:
local isProcessing = false
hooksecurefunc(Cooldown_MT, "SetCooldown", function(self, ...)
    if isProcessing then return end
    isProcessing = true
    -- ... code that might trigger SetCooldown again ...
    isProcessing = false
end)
```

---

## Step 5: Countdown Text Display

Now we build the visual overlay — a FontString that sits on top of each cooldown frame, showing formatted countdown text with color transitions.

Add this to `Display.lua`, above the hook:

```lua
-- ================================================================
-- Color Thresholds
-- ================================================================

local COLOR_LONG    = { r = 0.3, g = 1.0, b = 0.3 }  -- Green: > 30s
local COLOR_MEDIUM  = { r = 1.0, g = 0.82, b = 0.0 }  -- Yellow: 10-30s
local COLOR_SHORT   = { r = 1.0, g = 0.2, b = 0.2 }   -- Red: < 10s
local COLOR_EXPIRING = { r = 1.0, g = 0.0, b = 0.0 }   -- Bright red: < 3s

local function GetColorForRemaining(remaining)
    if remaining > 30 then
        return COLOR_LONG
    elseif remaining > 10 then
        return COLOR_MEDIUM
    elseif remaining > 3 then
        return COLOR_SHORT
    else
        return COLOR_EXPIRING
    end
end

-- ================================================================
-- Time Formatting
-- ================================================================

local function FormatTime(remaining)
    if remaining >= 3600 then
        return format("%dh", remaining / 3600)
    elseif remaining >= 60 then
        return format("%dm", remaining / 60)
    elseif remaining >= 10 then
        return format("%d", remaining)
    else
        return format("%.1f", remaining)
    end
end

-- ================================================================
-- Overlay Creation
-- ================================================================

function ns:CreateOverlay(cooldownFrame)
    local overlay = CreateFrame("Frame", nil, cooldownFrame)
    overlay:SetAllPoints(cooldownFrame)
    overlay:SetFrameLevel(cooldownFrame:GetFrameLevel() + 5)

    -- Countdown text
    local text = overlay:CreateFontString(nil, "OVERLAY")
    text:SetFont("Fonts\\FRIZQT__.TTF", ns.db.fontSize or 14, "OUTLINE")
    text:SetPoint("CENTER", overlay, "CENTER", 0, 0)
    text:SetJustifyH("CENTER")
    overlay.text = text

    -- Throttled OnUpdate
    local elapsed_total = 0
    overlay:SetScript("OnUpdate", function(self, elapsed)
        elapsed_total = elapsed_total + elapsed
        if elapsed_total < 0.1 then return end
        elapsed_total = 0

        local now = GetTime()
        local remaining = self.endTime - now

        if remaining <= 0 then
            self:Hide()
            activeFrames[cooldownFrame] = nil
            return
        end

        -- Update text
        text:SetText(FormatTime(remaining))

        -- Update color
        local color = GetColorForRemaining(remaining)
        text:SetTextColor(color.r, color.g, color.b)

        -- Trigger pulse check
        if ns.db.pulseEnabled and remaining <= ns.db.pulseThreshold then
            ns:CheckPulse(self, remaining)
        end
    end)

    return overlay
end
```

### Why 0.1 Second Throttle?

The `OnUpdate` handler fires every frame — 60+ times per second. Updating text that frequently is wasteful and creates garbage from `format()` calls. A 0.1 second throttle means we update 10 times per second, which looks perfectly smooth to the eye while generating 6x less garbage.

!!! tip "Performance rule of thumb"
    For visual updates that humans read as text, 10 updates per second (0.1s throttle) is the sweet spot. For smooth animations like progress bars, 30 updates per second (0.033s throttle) is sufficient. Only truly frame-critical rendering needs every-frame updates.

### Font Scaling

The FontString inherits the cooldown frame's scale. On a 32x32 action button, 14pt text fills the space well. On a 16x16 inventory cooldown, it will be proportionally smaller. This is intentional — the text scales with the cooldown frame automatically.

---

## Step 6: Pulse Animation

When a cooldown drops below 3 seconds, we play a subtle scale bounce to draw the player's eye. This uses WoW's `AnimationGroup` system — smooth, GPU-accelerated, and zero garbage.

Add this to `Display.lua`:

```lua
-- ================================================================
-- Pulse Animation
-- ================================================================

local function CreatePulseAnimation(overlay)
    local ag = overlay:CreateAnimationGroup()

    -- Scale up
    local scaleUp = ag:CreateAnimation("Scale")
    scaleUp:SetScale(1.4, 1.4)
    scaleUp:SetDuration(0.15)
    scaleUp:SetOrder(1)
    scaleUp:SetSmoothing("IN")

    -- Scale back down
    local scaleDown = ag:CreateAnimation("Scale")
    scaleDown:SetScale(1 / 1.4, 1 / 1.4)
    scaleDown:SetDuration(0.15)
    scaleDown:SetOrder(2)
    scaleDown:SetSmoothing("OUT")

    overlay.pulseAnim = ag
    return ag
end

function ns:CheckPulse(overlay, remaining)
    -- Only pulse once per second crossing
    local secondMark = math.floor(remaining)
    if overlay.lastPulseMark == secondMark then return end
    overlay.lastPulseMark = secondMark

    -- Create animation on first use
    if not overlay.pulseAnim then
        CreatePulseAnimation(overlay)
    end

    -- Play if not already playing
    if not overlay.pulseAnim:IsPlaying() then
        overlay.pulseAnim:Play()
    end
end
```

The key detail is the **threshold crossing check**. We track `lastPulseMark` — the last integer second we pulsed on. Without this, the pulse would fire every 0.1 seconds during the final 3 seconds, creating a jittery mess. Instead, it fires once at 3, once at 2, and once at 1.

!!! tip "Animation performance"
    WoW's animation system is hardware-accelerated. An `AnimationGroup` with `ScaleAnimation` costs essentially nothing compared to manually setting scale in `OnUpdate`. Always prefer the animation system for visual effects.

---

## Step 7: Action Bar Integration

Our metatable hook already catches action bar cooldowns automatically. But there are edge cases: dynamic bar swaps, vehicle bars, and the initial login state where buttons exist before our hook runs. This step handles those.

Add this to `Display.lua`:

```lua
-- ================================================================
-- Action Bar Integration
-- ================================================================

-- Combat queue for deferred operations
local combatQueue = {}

local function RunAfterCombat(fn)
    if InCombatLockdown() then
        combatQueue[#combatQueue + 1] = fn
        return false
    end
    fn()
    return true
end

-- Flush combat queue
local combatFrame = CreateFrame("Frame")
combatFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
combatFrame:SetScript("OnEvent", function()
    for i = 1, #combatQueue do
        combatQueue[i]()
        combatQueue[i] = nil
    end
end)

-- Action bar button names to scan
local ACTION_BAR_BUTTONS = {}
local barPrefixes = {
    "ActionButton",          -- Main bar (1-12)
    "MultiBarBottomLeftButton",
    "MultiBarBottomRightButton",
    "MultiBarRightButton",
    "MultiBarLeftButton",
    "MultiBar5Button",
    "MultiBar6Button",
    "MultiBar7Button",
}

for _, prefix in ipairs(barPrefixes) do
    for i = 1, 12 do
        ACTION_BAR_BUTTONS[#ACTION_BAR_BUTTONS + 1] = prefix .. i
    end
end

-- Scan existing cooldowns on login
function ns:ScanExistingCooldowns()
    for _, buttonName in ipairs(ACTION_BAR_BUTTONS) do
        local button = _G[buttonName]
        if button then
            local cooldown = button.cooldown or button.Cooldown
            if cooldown and not cooldown:IsForbidden() then
                local start, duration = cooldown:GetCooldown()
                if start and duration and duration > 0
                   and not issecretvalue(start)
                   and not issecretvalue(duration)
                   and duration >= (ns.db.minDuration or 2.5) then
                    OnCooldownSet(cooldown, start, duration)
                end
            end
        end
    end
end
```

!!! danger "Combat lockdown"
    Action bar buttons are **protected frames**. Any operation that modifies their structure — adding children, changing attributes, showing/hiding — can taint them during combat. Our overlay frames are created as children of the Cooldown frame (not the action button itself), which is safe. But if you ever need to restructure the frame hierarchy, always use `RunAfterCombat()`.

### Handling Bar Swaps

When a player enters a vehicle, uses a bonus action bar, or overrides their action bar, Blizzard calls `SetCooldown` on the new bar's cooldown frames. Our metatable hook catches this automatically — no special handling needed.

```lua
-- Hook for bar swap events — re-scan after swap completes
hooksecurefunc("ActionButton_UpdateCooldown", function(self)
    if self:IsForbidden() then return end
    local cooldown = self.cooldown or self.Cooldown
    if cooldown and not cooldown:IsForbidden() then
        local start, duration = cooldown:GetCooldown()
        if start and duration and duration > 0
           and not issecretvalue(start)
           and not issecretvalue(duration)
           and duration >= (ns.db.minDuration or 2.5) then
            OnCooldownSet(cooldown, start, duration)
        end
    end
end)
```

Update `ns:InitDisplay()` in `Display.lua` to call the initial scan:

```lua
function ns:InitDisplay()
    -- Delay initial scan slightly so all bars are loaded
    C_Timer.After(1, function()
        ns:ScanExistingCooldowns()
    end)
end
```

---

## Step 8: Settings Panel

Add a Settings panel so users can configure the addon without editing Lua.

### Config.lua

```lua
-- Mode: Enhancement Artist | Enhance don't replace
local addonName, ns = ...

local function RegisterSettings()
    local category, layout = Settings.RegisterVerticalLayoutCategory(addonName)
    ns.categoryID = category:GetID()

    -- Enable/Disable
    do
        local variable = "enabled"
        local name = "Enable Countdown Text"
        local tooltip = "Show countdown numbers on cooldown frames"
        local defaultValue = ns.defaults.enabled
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        Settings.CreateCheckbox(category, setting, tooltip)
    end

    -- Minimum Duration
    do
        local variable = "minDuration"
        local name = "Minimum Duration (seconds)"
        local tooltip = "Only show text for cooldowns longer than this"
        local defaultValue = ns.defaults.minDuration
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        local options = Settings.CreateSliderOptions(1, 10, 0.5)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    -- Font Size
    do
        local variable = "fontSize"
        local name = "Font Size"
        local tooltip = "Size of the countdown text"
        local defaultValue = ns.defaults.fontSize
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        local options = Settings.CreateSliderOptions(8, 24, 1)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    -- Pulse Animation
    do
        local variable = "pulseEnabled"
        local name = "Pulse Animation"
        local tooltip = "Pulse the countdown text when cooldowns are almost ready"
        local defaultValue = ns.defaults.pulseEnabled
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        Settings.CreateCheckbox(category, setting, tooltip)
    end

    -- Pulse Threshold
    do
        local variable = "pulseThreshold"
        local name = "Pulse Threshold (seconds)"
        local tooltip = "Start pulsing when cooldown is below this many seconds"
        local defaultValue = ns.defaults.pulseThreshold
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        local options = Settings.CreateSliderOptions(1, 10, 1)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    Settings.RegisterAddOnCategory(category)
end

-- Slash command
SLASH_BETTERCOOLDOWNS1 = "/bcd"
SlashCmdList["BETTERCOOLDOWNS"] = function(msg)
    if msg == "reset" then
        BetterCooldownsDB = CopyTable(ns.defaults)
        ns.db = BetterCooldownsDB
        print("|cff00ccffBetterCooldowns|r: Settings reset to defaults. /reload to apply.")
    else
        Settings.OpenToCategory(ns.categoryID)
    end
end

EventUtil.ContinueOnAddOnLoaded(addonName, RegisterSettings)
```

The modern Settings API automatically binds to your `ns.db` table — when the user moves a slider, the value is written directly to `BetterCooldownsDB.fontSize`. No manual callback wiring needed.

---

## Step 9: Edge Cases and Combat Lockdown

### The Combat Lockdown Dance

Action bar cooldown frames are children of secure button frames. While our overlay is a child of the Cooldown frame (not the button), some operations can still propagate taint in subtle ways.

!!! danger "Never do these during combat"
    - Do not call `SetParent()` on any frame attached to an action button
    - Do not call `SetFrameStrata()` on the overlay if its parent is protected
    - Do not iterate `_G` to find action button frames during combat
    - Do not create new frames as children of secure frames during combat

The combat queue pattern handles the rare cases where we need to defer:

```lua
-- Example: applying font size changes during combat
local function ApplyFontSize(overlay, size)
    RunAfterCombat(function()
        if overlay and overlay.text then
            overlay.text:SetFont("Fonts\\FRIZQT__.TTF", size, "OUTLINE")
        end
    end)
end
```

### Instance vs. Open World Behavior

=== "Open World"

    Full functionality. `SetCooldown(start, duration)` provides real numbers. Our countdown text, color transitions, and pulse animation all work as designed. The player gets the complete BetterCooldowns experience.

=== "Dungeons / Raids / M+ / PvP"

    Graceful degradation. When `issecretvalue(start)` or `issecretvalue(duration)` returns true, we hide our overlay text. Blizzard's native cooldown spiral continues to function because it receives Duration Objects internally. The player sees the standard spiral without our text — functional, just less informative.

### Restriction Detection

For more granular control, detect restriction state changes:

```lua
function eventHandlers:ADDON_RESTRICTION_CHANGED()
    if C_Secrets.HasSecretRestrictions() then
        -- Hide all overlays
        for cd, _ in pairs(activeFrames) do
            if cd.BetterCD then
                cd.BetterCD:Hide()
            end
        end
    else
        -- Re-scan to restore overlays
        ns:ScanExistingCooldowns()
    end
end
```

!!! warning "Do not cache secret state"
    Restriction state can change mid-session — entering a dungeon, starting a boss encounter, queuing for PvP. Always check `issecretvalue()` on each update cycle rather than caching a "we're in an instance" flag. The `ADDON_RESTRICTION_CHANGED` event gives you transition points, but individual values can become secret at different times.

---

## Step 10: Complete Code

Here is the full, copy-pasteable code for all four files. Create a `BetterCooldowns` folder in your `Interface/AddOns/` directory and paste each file.

### BetterCooldowns.toc

```toc
## Interface: 120001
## Title: BetterCooldowns
## Notes: Countdown text overlay for Blizzard cooldown frames
## Author: You
## Version: 1.0.0
## SavedVariables: BetterCooldownsDB
## IconTexture: Interface\Icons\Spell_Holy_BorrowedTime
## Category: Combat

Core.lua
Display.lua
Config.lua
```

### Core.lua

```lua
-- Mode: Enhancement Artist | Enhance don't replace
local addonName, ns = ...

-- Defaults
ns.defaults = {
    enabled = true,
    minDuration = 2.5,
    fontSize = 14,
    pulseEnabled = true,
    pulseThreshold = 3,
}

-- Event dispatch
local eventHandlers = {}

function eventHandlers:ADDON_LOADED(loadedAddon)
    if loadedAddon ~= addonName then return end

    if not BetterCooldownsDB then
        BetterCooldownsDB = CopyTable(ns.defaults)
    else
        for k, v in pairs(ns.defaults) do
            if BetterCooldownsDB[k] == nil then
                BetterCooldownsDB[k] = v
            end
        end
    end
    ns.db = BetterCooldownsDB

    self:UnregisterEvent("ADDON_LOADED")
end

function eventHandlers:PLAYER_LOGIN()
    ns:InitDisplay()
end

function eventHandlers:ADDON_RESTRICTION_CHANGED()
    if C_Secrets.HasSecretRestrictions() then
        for cd, _ in pairs(ns.activeFrames or {}) do
            if cd.BetterCD then
                cd.BetterCD:Hide()
            end
        end
    else
        C_Timer.After(0.5, function()
            ns:ScanExistingCooldowns()
        end)
    end
end

local eventFrame = CreateFrame("Frame")
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = eventHandlers[event]
    if handler then
        handler(self, ...)
    end
end)
for event in pairs(eventHandlers) do
    eventFrame:RegisterEvent(event)
end
```

### Display.lua

```lua
-- Mode: Enhancement Artist | Enhance don't replace
local addonName, ns = ...

-- Active cooldown overlays (shared with Core for restriction cleanup)
local activeFrames = {}
ns.activeFrames = activeFrames

-- ================================================================
-- Combat Queue
-- ================================================================

local combatQueue = {}

local function RunAfterCombat(fn)
    if InCombatLockdown() then
        combatQueue[#combatQueue + 1] = fn
        return false
    end
    fn()
    return true
end

local combatFrame = CreateFrame("Frame")
combatFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
combatFrame:SetScript("OnEvent", function()
    for i = 1, #combatQueue do
        combatQueue[i]()
        combatQueue[i] = nil
    end
end)

-- ================================================================
-- Color Thresholds
-- ================================================================

local COLOR_LONG     = { r = 0.3, g = 1.0, b = 0.3 }
local COLOR_MEDIUM   = { r = 1.0, g = 0.82, b = 0.0 }
local COLOR_SHORT    = { r = 1.0, g = 0.2, b = 0.2 }
local COLOR_EXPIRING = { r = 1.0, g = 0.0, b = 0.0 }

local function GetColorForRemaining(remaining)
    if remaining > 30 then
        return COLOR_LONG
    elseif remaining > 10 then
        return COLOR_MEDIUM
    elseif remaining > 3 then
        return COLOR_SHORT
    else
        return COLOR_EXPIRING
    end
end

-- ================================================================
-- Time Formatting
-- ================================================================

local function FormatTime(remaining)
    if remaining >= 3600 then
        return format("%dh", remaining / 3600)
    elseif remaining >= 60 then
        return format("%dm", remaining / 60)
    elseif remaining >= 10 then
        return format("%d", remaining)
    else
        return format("%.1f", remaining)
    end
end

-- ================================================================
-- Pulse Animation
-- ================================================================

local function CreatePulseAnimation(overlay)
    local ag = overlay:CreateAnimationGroup()

    local scaleUp = ag:CreateAnimation("Scale")
    scaleUp:SetScale(1.4, 1.4)
    scaleUp:SetDuration(0.15)
    scaleUp:SetOrder(1)
    scaleUp:SetSmoothing("IN")

    local scaleDown = ag:CreateAnimation("Scale")
    scaleDown:SetScale(1 / 1.4, 1 / 1.4)
    scaleDown:SetDuration(0.15)
    scaleDown:SetOrder(2)
    scaleDown:SetSmoothing("OUT")

    overlay.pulseAnim = ag
    return ag
end

local function CheckPulse(overlay, remaining)
    local secondMark = math.floor(remaining)
    if overlay.lastPulseMark == secondMark then return end
    overlay.lastPulseMark = secondMark

    if not overlay.pulseAnim then
        CreatePulseAnimation(overlay)
    end

    if not overlay.pulseAnim:IsPlaying() then
        overlay.pulseAnim:Play()
    end
end

-- ================================================================
-- Overlay Creation
-- ================================================================

local function CreateOverlay(cooldownFrame)
    local overlay = CreateFrame("Frame", nil, cooldownFrame)
    overlay:SetAllPoints(cooldownFrame)
    overlay:SetFrameLevel(cooldownFrame:GetFrameLevel() + 5)

    local text = overlay:CreateFontString(nil, "OVERLAY")
    text:SetFont("Fonts\\FRIZQT__.TTF", ns.db.fontSize or 14, "OUTLINE")
    text:SetPoint("CENTER", overlay, "CENTER", 0, 0)
    text:SetJustifyH("CENTER")
    overlay.text = text

    local elapsed_total = 0
    overlay:SetScript("OnUpdate", function(self, elapsed)
        elapsed_total = elapsed_total + elapsed
        if elapsed_total < 0.1 then return end
        elapsed_total = 0

        local now = GetTime()
        local remaining = self.endTime - now

        if remaining <= 0 then
            self:Hide()
            activeFrames[cooldownFrame] = nil
            return
        end

        text:SetText(FormatTime(remaining))

        local color = GetColorForRemaining(remaining)
        text:SetTextColor(color.r, color.g, color.b)

        if ns.db.pulseEnabled and remaining <= ns.db.pulseThreshold then
            CheckPulse(self, remaining)
        end
    end)

    return overlay
end

ns.CreateOverlay = CreateOverlay

-- ================================================================
-- Cooldown Metatable Hook
-- ================================================================

local tempCD = CreateFrame("Cooldown", nil, nil, "CooldownFrameTemplate")
local Cooldown_MT = getmetatable(tempCD).__index
tempCD = nil

local function OnCooldownSet(self, start, duration)
    if self:IsForbidden() then return end
    if not ns.db or not ns.db.enabled then return end

    if duration and not issecretvalue(duration) and duration < ns.db.minDuration then
        if self.BetterCD then
            self.BetterCD:Hide()
        end
        return
    end

    if issecretvalue(start) or issecretvalue(duration) then
        if self.BetterCD then
            self.BetterCD:Hide()
        end
        return
    end

    if not self.BetterCD then
        self.BetterCD = CreateOverlay(self)
    end

    local overlay = self.BetterCD
    overlay.start = start
    overlay.duration = duration
    overlay.endTime = start + duration
    overlay.lastPulseMark = nil
    overlay:Show()

    activeFrames[self] = true
end

hooksecurefunc(Cooldown_MT, "SetCooldown", OnCooldownSet)

-- Also hook Clear to hide overlay when cooldown is cancelled
hooksecurefunc(Cooldown_MT, "Clear", function(self)
    if self:IsForbidden() then return end
    if self.BetterCD then
        self.BetterCD:Hide()
    end
    activeFrames[self] = nil
end)

-- ================================================================
-- Action Bar Scanning
-- ================================================================

local ACTION_BAR_BUTTONS = {}
local barPrefixes = {
    "ActionButton",
    "MultiBarBottomLeftButton",
    "MultiBarBottomRightButton",
    "MultiBarRightButton",
    "MultiBarLeftButton",
    "MultiBar5Button",
    "MultiBar6Button",
    "MultiBar7Button",
}

for _, prefix in ipairs(barPrefixes) do
    for i = 1, 12 do
        ACTION_BAR_BUTTONS[#ACTION_BAR_BUTTONS + 1] = prefix .. i
    end
end

function ns:ScanExistingCooldowns()
    for _, buttonName in ipairs(ACTION_BAR_BUTTONS) do
        local button = _G[buttonName]
        if button then
            local cooldown = button.cooldown or button.Cooldown
            if cooldown and not cooldown:IsForbidden() then
                local start, duration = cooldown:GetCooldown()
                if start and duration and duration > 0
                   and not issecretvalue(start)
                   and not issecretvalue(duration)
                   and duration >= (ns.db.minDuration or 2.5) then
                    OnCooldownSet(cooldown, start, duration)
                end
            end
        end
    end
end

hooksecurefunc("ActionButton_UpdateCooldown", function(self)
    if self:IsForbidden() then return end
    local cooldown = self.cooldown or self.Cooldown
    if cooldown and not cooldown:IsForbidden() then
        local start, duration = cooldown:GetCooldown()
        if start and duration and duration > 0
           and not issecretvalue(start)
           and not issecretvalue(duration)
           and duration >= (ns.db.minDuration or 2.5) then
            OnCooldownSet(cooldown, start, duration)
        end
    end
end)

function ns:InitDisplay()
    C_Timer.After(1, function()
        ns:ScanExistingCooldowns()
    end)
end
```

### Config.lua

```lua
-- Mode: Enhancement Artist | Enhance don't replace
local addonName, ns = ...

local function RegisterSettings()
    local category, layout = Settings.RegisterVerticalLayoutCategory(addonName)
    ns.categoryID = category:GetID()

    do
        local variable = "enabled"
        local name = "Enable Countdown Text"
        local tooltip = "Show countdown numbers on cooldown frames"
        local defaultValue = ns.defaults.enabled
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        Settings.CreateCheckbox(category, setting, tooltip)
    end

    do
        local variable = "minDuration"
        local name = "Minimum Duration (seconds)"
        local tooltip = "Only show text for cooldowns longer than this value. Filters out the GCD."
        local defaultValue = ns.defaults.minDuration
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        local options = Settings.CreateSliderOptions(1, 10, 0.5)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    do
        local variable = "fontSize"
        local name = "Font Size"
        local tooltip = "Size of the countdown text on cooldown frames"
        local defaultValue = ns.defaults.fontSize
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        local options = Settings.CreateSliderOptions(8, 24, 1)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    do
        local variable = "pulseEnabled"
        local name = "Pulse Animation"
        local tooltip = "Play a scale pulse when cooldowns are almost ready"
        local defaultValue = ns.defaults.pulseEnabled
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        Settings.CreateCheckbox(category, setting, tooltip)
    end

    do
        local variable = "pulseThreshold"
        local name = "Pulse Threshold (seconds)"
        local tooltip = "Start the pulse animation below this many seconds remaining"
        local defaultValue = ns.defaults.pulseThreshold
        local setting = Settings.RegisterAddOnSetting(
            category, variable, variable,
            ns.db, type(defaultValue), name, defaultValue
        )
        local options = Settings.CreateSliderOptions(1, 10, 1)
        Settings.CreateSlider(category, setting, options, tooltip)
    end

    Settings.RegisterAddOnCategory(category)
end

SLASH_BETTERCOOLDOWNS1 = "/bcd"
SlashCmdList["BETTERCOOLDOWNS"] = function(msg)
    if msg == "reset" then
        BetterCooldownsDB = CopyTable(ns.defaults)
        ns.db = BetterCooldownsDB
        print("|cff00ccffBetterCooldowns|r: Settings reset to defaults. /reload to apply.")
    else
        Settings.OpenToCategory(ns.categoryID)
    end
end

EventUtil.ContinueOnAddOnLoaded(addonName, RegisterSettings)
```

---

## What You Have Built

BetterCooldowns is a complete Enhancement Artist addon that demonstrates the patterns used by production addons like OmniCD, Arc UI, and ElvUI's cooldown module:

- **Metatable hook** catches every cooldown in the UI with a single `hooksecurefunc`
- **IsForbidden() guards** prevent crashes on secure frames
- **issecretvalue() checks** gracefully degrade in restricted contexts
- **Throttled OnUpdate** keeps CPU usage minimal
- **AnimationGroup** provides smooth, GPU-accelerated pulse effects
- **Combat queue** defers protected operations safely
- **Modern Settings API** gives users a native-feeling configuration panel

The Enhancement Artist philosophy kept us honest at every step: we hooked, we overlaid, we enhanced — but we never replaced a single Blizzard frame.

!!! tip "Next steps"
    - Add Masque support so skin addons can style your cooldown overlays
    - Hook `SetCooldownFromDurationObject` for completeness
    - Add per-frame font scaling based on cooldown frame size
    - Register with EditModeExpanded for user-positionable overlay anchors
    - Try switching to `/wow-mode boundary` and refactoring to use the Cooldown metatable `__index` hook for even broader coverage
