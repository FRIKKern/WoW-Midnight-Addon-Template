---
title: "The Mode System: Four Development Philosophies"
description: "How the WoW addon AI mode system works — four development philosophies that change how every agent writes, reviews, and scaffolds addon code for Midnight 12.0+."
---

# The Mode System: Four Development Philosophies

Every addon exists on a spectrum between "safe and boring" and "powerful and fragile." The mode system lets you tell the AI exactly where on that spectrum you want to build.

A **mode** is a development philosophy preset that fundamentally changes how every AI agent behaves — from code generation and scaffolding to code review and architectural recommendations. When you set a mode, you're not just changing a config flag. You're changing the AI's entire mental model of what "good addon code" looks like.

## What Are Modes?

Modes are stored as a single line in your project's `.claude/modes/active-mode.md` file. That one line — a canonical mode name — propagates to every agent, every command, and every review pass. When the Coder agent generates a frame hook, it checks the active mode. When the Reviewer agent flags a pattern, it checks the active mode. When the Scaffold agent creates boilerplate, it checks the active mode.

Switch modes with a single command:

```
/wow-mode enhancement-artist
```

The active mode affects:

- **Code generation** — Which APIs and patterns the Coder and Scaffold agents use
- **Code review severity** — What the Reviewer flags as violations vs. acceptable patterns
- **Scaffolding boilerplate** — Which standard patterns are included in new addon files
- **Architectural recommendations** — How the Generalist and Skin Designer agents approach problems

The default mode is **Enhancement Artist** — the philosophy behind the "BetterX" addon movement that dominates the Midnight ecosystem. Most developers never need to change it. But when you're building an ElvUI-class UI replacement or a zero-overhead raid tool, the right mode makes the AI a fundamentally better collaborator.

!!! tip "Modes are per-project, not global"
    Each project has its own `active-mode.md`. You can have a Boundary Pusher nameplate addon and a Blizzard Faithful guild tool open in different terminals, each with its own mode. The mode travels with the project, not with your Claude Code installation.

---

## The Four Modes

### :material-shield-check: Blizzard Faithful

> *"Official APIs only. If Blizzard didn't document it, we don't use it. Zero patch-day risk."*

Blizzard Faithful is the most conservative mode. It constrains all code generation to documented, stable WoW APIs — the kind listed on warcraft.wiki.gg with official signatures and return values. Every pattern it produces has survived multiple patch cycles because it relies only on Blizzard's public API contract. When in doubt, the Faithful mode leaves it out.

**Personality:** Boring, safe, and bulletproof.

#### What It Allows

| Pattern | Example |
|---|---|
| Settings API | `Settings.RegisterVerticalLayoutCategory()`, `Settings.CreateCheckbox()` |
| Addon Compartment | `AddonCompartmentFunc` in TOC, `Settings.OpenToCategory()` |
| `EventUtil.ContinueOnAddOnLoaded()` | Clean initialization without manual `ADDON_LOADED` filtering |
| Documented `C_` namespace calls | `C_Spell.GetSpellInfo()`, `C_Timer.After()`, `C_DamageMeter.*` |
| Standard frame creation | `CreateFrame()` with official Blizzard templates |
| Event dispatch tables | `RegisterEvent()` + handler table pattern |
| Secret Values with graceful degradation | `issecretvalue()` checks with widget-safe fallbacks |

!!! danger "What It Forbids"
    - `hooksecurefunc()` on any Blizzard frame or global function
    - `getmetatable()` on any frame you didn't create
    - `GetRegions()` on Blizzard frames
    - Noop pattern (`frame.SetNormalTexture = function() end`)
    - UIHider reparenting (hiding Blizzard frames via hidden parent)
    - Accessing undocumented frame children (`.NineSlice`, `.Background`, `.Border`)
    - `SetScript()` or `HookScript()` on any Blizzard frame
    - `LoadAddOn()` to force-load Blizzard UI modules
    - Direct parenting to Blizzard frames (parent to `UIParent` instead)

#### Best For

- **First-time addon developers** learning correct patterns from the start
- **Guild tools** and utility addons that must never break
- **Corporate/competitive addons** where reliability is non-negotiable
- **Settings-only addons** that configure but don't touch Blizzard UI
- **Data display addons** that create their own frames from scratch

#### Code Example: Settings-Only Addon

A minimal addon using only documented, stable APIs. This code will survive every patch without modification:

```lua
-- Mode: Blizzard Faithful | Safe across all patches
local addonName, ns = ...

ns.defaults = {
    enabled = true,
    scale = 1.0,
    showTooltips = true,
}

-- Safe initialization via documented EventUtil
EventUtil.ContinueOnAddOnLoaded(addonName, function()
    ns.db = MyAddonDB or CopyTable(ns.defaults)
    MyAddonDB = ns.db

    -- Settings panel via documented Settings API
    local category, layout = Settings.RegisterVerticalLayoutCategory(addonName)
    ns.categoryID = category:GetID()

    local enabledSetting = Settings.RegisterAddOnSetting(
        category, "enabled", "enabled",
        ns.db, type(true), "Enable Addon", true
    )
    Settings.CreateCheckbox(category, enabledSetting,
        "Toggle the addon on or off.")

    local scaleSetting = Settings.RegisterAddOnSetting(
        category, "scale", "scale",
        ns.db, type(1.0), "UI Scale", 1.0
    )
    local scaleOptions = Settings.CreateSliderOptions(0.5, 2.0, 0.1)
    Settings.CreateSlider(category, scaleSetting, scaleOptions,
        "Adjust the scale of addon frames.")

    Settings.RegisterAddOnCategory(category)
end)

-- Addon Compartment (minimap presence) — declared in TOC
function MyAddon_OnAddonCompartmentClick()
    Settings.OpenToCategory(ns.categoryID)
end
```

!!! tip "When to choose Faithful"
    If your addon never needs to touch a Blizzard frame — if it creates its own UI, reads data through documented APIs, and displays results in its own frames — Blizzard Faithful is the right choice. You'll never worry about patch day.

---

### :material-palette: Enhancement Artist (DEFAULT)

> *"Skin, hook, extend — never replace. Make Blizzard's UI beautiful without breaking it."*

Enhancement Artist is the default mode and the philosophy behind the "Better Addon" movement that defines the Midnight addon ecosystem. It lets you reach into Blizzard's own frames and make them look and work better — class-colored health bars, cleaner textures, repositioned elements — while preserving all of Blizzard's built-in functionality. Users keep their familiar layouts, Edit Mode settings, and muscle memory. You just make it all *better*.

**Personality:** Respectful, creative, and resilient.

#### Key Rules

Every piece of Enhancement Artist code follows these non-negotiable rules:

1. **Always `hooksecurefunc()`** — never pre-hook or override Blizzard functions
2. **Always `IsForbidden()` checks** — before modifying any hooked frame
3. **Always recursion guards** — when hooks trigger methods that fire hooks
4. **Always store original state** — save `ogPoint`, `ogParent`, `ogWidth` before modifying
5. **Always defer out of combat** — via the `RunAfterCombat` queue pattern
6. **Always `HookScript()`** — never `SetScript()` on Blizzard frames

!!! danger "What It Forbids"
    - `SetScript()` on any Blizzard frame (use `HookScript()`)
    - `:Hide()` on protected frames (use hidden frame reparenting)
    - Pre-hooking or overriding Blizzard functions (only post-hook)
    - Raw arithmetic on health/power values (may be Secret Values)
    - `CreateFrame()` replacements for existing Blizzard frames
    - Direct calls to `EditModeManagerFrame:RegisterSystemFrame()` (causes taint)

#### Standard Patterns

The Enhancement Artist toolkit is built on a set of reusable patterns that form the foundation of every "BetterX" addon:

| Pattern | Purpose |
|---|---|
| Hidden frame | Reparent stripped textures to hide them without `:Hide()` |
| Combat queue | Defer frame operations until `PLAYER_REGEN_ENABLED` |
| `StripTextures()` | Remove Blizzard textures by reparenting to hidden frame |
| `ApplyBackdrop()` | Apply clean backdrop via `Mixin(frame, BackdropTemplateMixin)` |
| Safe hook | `hooksecurefunc` with `IsForbidden()` + recursion guard |
| Original state preservation | Save frame properties before modification for restore |
| Edit Mode awareness | Hook `EventRegistry` for `EditMode.Enter` / `EditMode.Exit` |
| ScrollBox skinning | Hook `OnDataRangeChanged` callback for list element skinning |

#### Best For

- **"BetterX" addons** that improve Blizzard UI without replacing it
- **Frame skinning addons** that add class colors, clean textures, or visual polish
- **UI enhancement addons** that reposition or resize Blizzard elements
- **Quality-of-life addons** that add functionality to existing Blizzard frames

**Reference implementations:** BetterBlizzFrames, BetterBlizzPlates, ActionBarsEnhanced, Platynator.

#### Code Example: Hook CompactUnitFrame for Class Colors

A simple enhancement that adds class-colored health bars to the default raid frames — the quintessential Enhancement Artist pattern:

```lua
-- Mode: Enhancement Artist | Enhance don't replace
local addonName, ns = ...

-- Recursion guard
local isProcessing = false

-- Hook the raid frame health bar update
hooksecurefunc("CompactUnitFrame_UpdateHealthColor", function(frame)
    if isProcessing then return end
    if frame:IsForbidden() then return end
    if not frame.healthBar then return end

    isProcessing = true

    local unit = frame.unit
    if unit and UnitExists(unit) and UnitIsPlayer(unit) then
        local _, class = UnitClass(unit)
        local color = RAID_CLASS_COLORS[class]
        if color then
            frame.healthBar:SetStatusBarColor(color.r, color.g, color.b)
        end
    end

    isProcessing = false
end)
```

!!! tip "When to choose Enhancement Artist"
    If your addon's purpose is to make a Blizzard frame look or work better — without replacing it — Enhancement Artist is your mode. It's the default for a reason: it produces addons that are powerful enough to be useful but safe enough to survive patches with minimal maintenance.

---

### :material-rocket-launch: Boundary Pusher

> *"Push WoW's addon API to its absolute limits. If the top addons do it, we do it better."*

Boundary Pusher unlocks the aggressive, advanced techniques used by ElvUI, BetterBags, Cell, Plater, and OmniCD. Every technique in this mode is battle-tested in production addons with millions of downloads — but every technique also carries patch-day risk. This mode generates ElvUI/WeakAuras-class code that pushes the envelope, and it demands safety nets for every aggressive pattern.

**Personality:** Fearless, disciplined, and always prepared to fall back.

#### Unlocked Techniques

| Technique | What It Does |
|---|---|
| Metatable hooks (`getmetatable()` + `__index`) | Hook every frame of a type at the class level (e.g., all Cooldowns) |
| Global Cooldown_MT pattern | Apply behavior to ALL frames of a type without iterating instances |
| `hooksecurefunc` on ANY Blizzard function | No limits — hook any global or frame method |
| `GetRegions()` stripping | Iterate and modify/hide Blizzard textures region by region |
| Noop pattern | Block Blizzard texture re-application: `frame.SetNormalTexture = function() end` |
| UIHider reparenting | Hide protected frames by reparenting to a hidden parent |
| `SetOverrideBinding()` | Override keybinds per-frame without changing saved bindings |
| Deferred flag pattern | Set flags in hooks, process in next OnUpdate tick to avoid re-entrancy |
| Taint isolation parent frame | BetterBags pattern: intermediate parent to stop taint propagation |
| ScrollBox factory hooks | Skin every list element Blizzard creates |
| Classification nameplate tricks | Color/prioritize nameplates by mob elite/rare/boss classification |

!!! warning "Required Safety Nets"
    Every aggressive technique **must** include these safeguards. No exceptions:

    1. **`-- BOUNDARY` comment** marking every aggressive hook with fallback description
    2. **`pcall()` wrapper** around metatable access (metatables change between patches)
    3. **Fallback path** when hooked function signatures change
    4. **Version check** where possible: gate techniques behind `GetBuildInfo()` TOC version

#### Best For

- **Complete UI replacements** (ElvUI-class)
- **Nameplate addons** (Plater/Platynator-class)
- **Cooldown/ability trackers** (OmniCD-class)
- **Bag replacements** (BetterBags-class)
- **Raid frame addons** (Cell-class)
- **Boss mods** (DBM/BigWigs-class)

**Reference implementations:** ElvUI, BetterBags, Cell, Plater, OmniCD, DBM, BigWigs.

#### Code Example: Metatable Hook with Safety

A Cooldown metatable hook that affects every Cooldown frame in the game — with full `pcall` safety and fallback:

```lua
-- Mode: Boundary Pusher | Push the limits
local addonName, ns = ...

-- BOUNDARY: May break if Blizzard changes Cooldown metatable
-- Fallback: per-frame hooksecurefunc on individual cooldown frames
local function SetupCooldownHook()
    local ok, cd = pcall(CreateFrame, "Cooldown", nil, nil, "CooldownFrameTemplate")
    if not ok or not cd then
        ns:PrintDebug("Cooldown metatable hook failed — using fallback")
        return false
    end

    local ok2, mt = pcall(getmetatable, cd)
    if not ok2 or not mt or not mt.__index then
        ns:PrintDebug("Cooldown metatable unavailable — using fallback")
        return false
    end

    local Cooldown_MT = mt.__index
    hooksecurefunc(Cooldown_MT, "SetCooldown", function(self, start, duration)
        if self:IsForbidden() then return end
        if not start or type(start) ~= "number" then return end  -- signature check
        if duration <= 1.5 then return end  -- ignore GCD

        -- Add custom cooldown text overlay to ALL cooldown frames
        if not self.myText then
            self.myText = self:CreateFontString(nil, "OVERLAY", "GameFontNormalLarge")
            self.myText:SetPoint("CENTER")
        end
        self.myText:SetText(math.ceil(duration))
        C_Timer.After(duration, function()
            if self.myText then self.myText:SetText("") end
        end)
    end)

    return true
end

-- Try aggressive hook, fall back to safe pattern
if not SetupCooldownHook() then
    ns:SetupPerFrameCooldownTracking()  -- Enhancement Artist fallback
end
```

!!! danger "Even Boundary Pushers Have Lines"
    These are absolute prohibitions even in Boundary Pusher mode:

    - Direct memory/C-side access or `loadstring` tricks
    - Modifying SavedVariables of other addons
    - Replacing Blizzard's event dispatch or `SetScript` on frames you don't own
    - Breaking other addons' hooks (always chain, never replace)
    - Bypassing `InCombatLockdown()` checks
    - Caching secret values across restriction boundaries

---

### :material-speedometer: Performance Zealot

> *"Every allocation counts. Zero waste. Your addon should be invisible to `/eventtrace`."*

Performance Zealot prioritizes absolute minimum overhead. Every table creation, every global lookup, every unthrottled handler is a tax on the player's framerate. Code generated in this mode is lean, pre-allocated, and event-driven. If it can be cached, it is cached. If it can be pooled, it is pooled. If it can be avoided, it is avoided.

**Personality:** Ruthlessly efficient, measurement-obsessed.

#### Key Rules

1. **Never** create tables inside functions called more than once per second
2. **Always** cache globals as locals at file scope (`local GetTime = GetTime`)
3. **Always** throttle OnUpdate with elapsed accumulator (minimum 0.1s interval)
4. **Always** use events over polling — register for specific events only
5. **Always** unregister events when data is no longer needed
6. **Always** use `wipe(t)` over `t = {}` for table reuse
7. **Prefer** `RegisterUnitEvent("EVENT", "unit")` over `RegisterEvent("EVENT")`
8. **Require** a `/perf` diagnostic slash command in every addon

#### The 10 Anti-Patterns

Any of these in generated code is an automatic rejection during review:

| # | Anti-Pattern | Fix |
|---|---|---|
| 1 | `{}` inside a function called >1/sec | `wipe()` + reuse pre-allocated table |
| 2 | `a .. b .. c` in OnUpdate | `format()` with pre-built format string |
| 3 | Unthrottled OnUpdate handler | Elapsed accumulator, min 0.1s |
| 4 | Event handler without early-return for irrelevant units | Check `unit` arg first, return early |
| 5 | `pairs()` on integer-keyed tables in hot paths | `ipairs()` or numeric `for` loop |
| 6 | Closures (`function() end`) inside frequently-called functions | Upvalues or pre-defined functions |
| 7 | `RegisterEvent` when `RegisterUnitEvent` would suffice | `RegisterUnitEvent("EVENT", "player")` |
| 8 | Multiple OnUpdate frames across files | One centralized tick frame |
| 9 | String keys in inner loops | Numeric keys or direct field access |
| 10 | Unused event registrations left active | `UnregisterEvent` when inactive |

#### Best For

- **Raid environments** (20+ players, dozens of auras, high event throughput)
- **Low-spec machines** where every KB and every ms matters
- **Always-on addons** (damage meters, nameplates, unit frames)
- **High-frequency processing** (combat parsing, health tracking, nameplate updates)

#### Code Example: Object Pool with Diagnostics

The canonical Performance Zealot pattern — an object pool that eliminates `CreateFrame()` calls in hot paths, with the required `/perf` diagnostic:

```lua
-- Mode: Performance Zealot | Zero-waste addon
local addonName, ns = ...

-- Performance: cached globals
local tremove, wipe = table.remove, table.wipe
local CreateFrame = CreateFrame
local format = string.format
local GetAddOnMemoryUsage = GetAddOnMemoryUsage
local UpdateAddOnMemoryUsage = UpdateAddOnMemoryUsage

-- Object pool: never CreateFrame() in a hot path
local pool = {}
local activeCount = 0

local function Acquire()
    local obj = tremove(pool)
    if not obj then
        obj = CreateFrame("Frame", nil, UIParent)
        obj:SetSize(24, 24)
        obj.icon = obj:CreateTexture(nil, "ARTWORK")
        obj.icon:SetAllPoints()
    end
    obj:Show()
    activeCount = activeCount + 1
    return obj
end

local function Release(obj)
    obj:Hide()
    obj:ClearAllPoints()
    obj.icon:SetTexture(nil)
    activeCount = activeCount - 1
    pool[#pool + 1] = obj
end

local function ReleaseAll()
    -- Caller provides the active list, we drain it
    -- No table allocation: caller wipes their own reference table
end

-- Required: /perf diagnostic
SLASH_MYADDON_PERF1 = "/myaddonperf"
SlashCmdList["MYADDON_PERF"] = function()
    UpdateAddOnMemoryUsage()
    local mem = GetAddOnMemoryUsage(addonName)
    print(format(
        "|cff00ff00%s|r: %.1f KB | %d active | %d pooled",
        addonName, mem, activeCount, #pool
    ))
end
```

!!! tip "Performance Zealot stacks with other philosophies"
    You can think "like a Performance Zealot" while writing Enhancement Artist or Boundary Pusher code. The mode system enforces one active mode, but the performance anti-patterns are worth internalizing regardless. If you find yourself writing `{}` inside an OnUpdate handler, you already know what the Zealot would say.

---

## Choosing Your Mode

### Decision Matrix

| | :material-shield-check: Faithful | :material-palette: Artist | :material-rocket-launch: Pusher | :material-speedometer: Zealot |
|---|---|---|---|---|
| **Risk tolerance** | Zero | Low | High | Low |
| **Patch survival** | Guaranteed | High | Medium | High |
| **Code complexity** | Simple | Moderate | High | Moderate |
| **Blizzard frame access** | None | Hook & extend | Full manipulation | Depends on need |
| **Best for** | Utilities, data display | "BetterX" addons | UI replacements | Raid/perf-critical |
| **Experience level** | Beginner | Intermediate | Advanced | Intermediate+ |

### Choosing by Question

Ask yourself these questions in order:

**1. Does this addon touch Blizzard frames at all?**

- No :material-arrow-right: **Blizzard Faithful**. You're building your own UI from scratch with documented APIs.

**2. Do you need to *replace* Blizzard frames, or just *improve* them?**

- Improve them :material-arrow-right: **Enhancement Artist**. Hook, skin, extend.
- Replace them :material-arrow-right: **Boundary Pusher**. You need the aggressive toolbox.

**3. Is this addon performance-critical? (Raid frames, nameplates, combat processing)**

- Yes :material-arrow-right: **Performance Zealot**. Every allocation counts.
- No :material-arrow-right: Stay with your choice from step 2.

!!! tip "Start with Enhancement Artist"
    When in doubt, start with the default. Enhancement Artist is the sweet spot for most addon developers — powerful enough to build impressive addons, safe enough to survive patch days. Graduate to Boundary Pusher when you hit a wall that hooks alone can't solve.

---

## Mode in Action: Same Addon, Four Ways

To see how modes change real code, let's implement the same feature four different ways: **add class color to the target frame's name text**.

=== ":material-shield-check: Blizzard Faithful"

    Faithful mode **cannot** do this directly. The target frame's name `FontString` is an internal Blizzard frame child — accessing it is forbidden. Instead, Faithful suggests an alternative: create your own overlay text.

    ```lua
    -- Mode: Blizzard Faithful | Safe across all patches
    -- Cannot access TargetFrame name text directly.
    -- Alternative: create our own class-colored name display.
    local addonName, ns = ...

    local nameFrame = CreateFrame("Frame", nil, UIParent)
    nameFrame:SetSize(200, 20)
    nameFrame:SetPoint("TOP", UIParent, "TOP", 0, -20)

    local nameText = nameFrame:CreateFontString(nil, "OVERLAY",
        "GameFontNormalLarge")
    nameText:SetPoint("CENTER")

    local events = {}

    function events:PLAYER_TARGET_CHANGED()
        if not UnitExists("target") then
            nameFrame:Hide()
            return
        end
        local name = UnitName("target")
        if UnitIsPlayer("target") then
            local _, class = UnitClass("target")
            local color = RAID_CLASS_COLORS[class]
            if color then
                nameText:SetText(color:WrapTextInColorCode(name))
            else
                nameText:SetText(name)
            end
        else
            nameText:SetText(name)
        end
        nameFrame:Show()
    end

    local eventFrame = CreateFrame("Frame")
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        local handler = events[event]
        if handler then handler(self, ...) end
    end)
    for event in pairs(events) do
        eventFrame:RegisterEvent(event)
    end
    ```

    !!! info "The Faithful trade-off"
        You get a class-colored name, but it's a separate display element — not integrated into the target frame itself. The upside: this code will work on every patch, forever.

=== ":material-palette: Enhancement Artist"

    Enhancement Artist hooks the target frame's update function and applies class color to the existing name text — enhancing without replacing.

    ```lua
    -- Mode: Enhancement Artist | Enhance don't replace
    local addonName, ns = ...

    local isProcessing = false

    hooksecurefunc("TargetFrame_UpdateName", function(self)
        if isProcessing then return end
        if self:IsForbidden() then return end

        local name = self.name
        if not name then return end

        isProcessing = true

        local unit = self.unit
        if unit and UnitExists(unit) and UnitIsPlayer(unit) then
            local _, class = UnitClass(unit)
            local color = RAID_CLASS_COLORS[class]
            if color then
                name:SetTextColor(color.r, color.g, color.b)
            end
        end

        isProcessing = false
    end)
    ```

    !!! info "The Artist trade-off"
        Clean integration into the existing target frame. Uses `hooksecurefunc` so it runs *after* Blizzard sets the name, overriding the color. Patch risk is low — if `TargetFrame_UpdateName` is renamed, the hook silently does nothing.

=== ":material-rocket-launch: Boundary Pusher"

    Boundary Pusher hooks at the metatable level with full `pcall` safety, catching the update earlier and with more control.

    ```lua
    -- Mode: Boundary Pusher | Push the limits
    local addonName, ns = ...

    -- BOUNDARY: May break if TargetFrame metatable changes
    -- Fallback: hooksecurefunc on TargetFrame_UpdateName
    local function SetupTargetNameHook()
        local targetFrame = TargetFrame
        if not targetFrame then return false end

        local ok, mt = pcall(getmetatable, targetFrame)
        if not ok or not mt or not mt.__index then
            return false
        end

        -- Hook at the frame method level for maximum control
        hooksecurefunc(mt.__index, "UpdateName", function(self)
            if self:IsForbidden() then return end
            local name = self.name
            if not name then return end

            local unit = self.unit
            if not unit or not UnitExists(unit) then return end

            if UnitIsPlayer(unit) then
                local _, class = UnitClass(unit)
                local color = RAID_CLASS_COLORS[class]
                if color then
                    name:SetTextColor(color.r, color.g, color.b)
                end
            else
                -- Color NPCs by reaction
                local reaction = UnitReaction(unit, "player")
                if reaction then
                    local color = FACTION_BAR_COLORS[reaction]
                    if color then
                        name:SetTextColor(color.r, color.g, color.b)
                    end
                end
            end
        end)

        return true
    end

    if not SetupTargetNameHook() then
        -- Fallback to Enhancement Artist approach
        hooksecurefunc("TargetFrame_UpdateName", function(self)
            if self:IsForbidden() then return end
            if not self.name or not self.unit then return end
            if UnitIsPlayer(self.unit) then
                local _, class = UnitClass(self.unit)
                local color = RAID_CLASS_COLORS[class]
                if color then
                    self.name:SetTextColor(color.r, color.g, color.b)
                end
            end
        end)
    end
    ```

    !!! info "The Pusher trade-off"
        More power (NPC reaction colors, metatable-level hook), but more code and more patch-day risk. The `pcall` + fallback pattern ensures it degrades gracefully.

=== ":material-speedometer: Performance Zealot"

    Performance Zealot caches everything — class colors, unit lookups, the color table itself — and throttles updates to avoid burning CPU on rapid target switches.

    ```lua
    -- Mode: Performance Zealot | Zero-waste addon
    local addonName, ns = ...

    -- Performance: cached globals
    local UnitExists = UnitExists
    local UnitIsPlayer = UnitIsPlayer
    local UnitClass = UnitClass
    local RAID_CLASS_COLORS = RAID_CLASS_COLORS

    -- Pre-cache class colors as simple r,g,b to avoid table lookups
    local classColorCache = {}
    for class, color in pairs(RAID_CLASS_COLORS) do
        classColorCache[class] = { color.r, color.g, color.b }
    end

    -- Throttle: don't recolor on every micro-update
    local lastUnit = nil
    local lastClass = nil

    hooksecurefunc("TargetFrame_UpdateName", function(self)
        if self:IsForbidden() then return end

        local unit = self.unit
        if not unit or not UnitExists(unit) then return end

        -- Skip if same target, same class (no work needed)
        if unit == lastUnit then return end
        lastUnit = unit

        local name = self.name
        if not name then return end

        if UnitIsPlayer(unit) then
            local _, class = UnitClass(unit)
            if class == lastClass then return end  -- same class, skip
            lastClass = class

            local c = classColorCache[class]
            if c then
                name:SetTextColor(c[1], c[2], c[3])
            end
        else
            lastClass = nil
        end
    end)
    ```

    !!! info "The Zealot trade-off"
        Same visual result as Enhancement Artist, but with cached color lookups, duplicate-target early returns, and zero table allocations in the hot path. The difference is invisible for a target frame (it updates rarely), but these habits matter when applied to nameplate or raid frame addons processing dozens of units per second.

---

## Switching Modes

### Commands

```
/wow-mode blizzard-faithful    # Full name
/wow-mode faithful             # Alias
/wow-mode safe                 # Another alias
/wow-mode status               # Show current mode
/wow-mode                      # Same as status
```

### All Aliases

| Mode | Aliases |
|---|---|
| `blizzard-faithful` | `faithful`, `blizzard`, `safe`, `conservative` |
| `enhancement-artist` | `enhance`, `artist`, `better`, `skin`, `hook` |
| `boundary-pusher` | `boundary`, `pusher`, `aggressive`, `advanced`, `elvui` |
| `performance-zealot` | `performance`, `perf`, `zealot`, `fast`, `lean` |

### What Happens When You Switch

1. `active-mode.md` is overwritten with the canonical mode name
2. All subsequent agent interactions use the new mode's rules
3. Code generation patterns change immediately
4. Review criteria adjust to the new mode's standards

The switch is instant and affects the current project only. No restart needed — the next time any agent reads the active mode file, it picks up the change.

!!! tip "Inline mode hints"
    You can also hint at a mode without formally switching. When using `/wow-create`, simply describe what you want: *"create a faithful settings addon"* or *"build a boundary-pusher nameplate addon."* The agents will recognize the mode name in your description and apply those rules for that task.
