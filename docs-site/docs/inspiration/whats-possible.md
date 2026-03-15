---
title: "What's Possible: Pushing Boundaries in 12.0"
description: "A showcase of the cutting edge of WoW 12.0 addon development — new patterns, impressive community addons, advanced techniques, and what's coming next."
---

# What's Possible: Pushing Boundaries in 12.0

The Great Addon Purge of 2026 didn't end addon development. It *restarted* it.

Midnight shipped with **437 new APIs**, removed **138 old ones**, and introduced **76 new events**. The combat log is gone. Secret Values replaced raw data. Entire addon categories vanished overnight. And yet — two weeks into launch — the addon ecosystem is more inventive than it has been in years.

This page is about what's *possible* now. Not what was lost, but what's being built.

---

## The New Landscape

### :material-earth: A New World of APIs

Patch 12.0 was the largest API change in WoW's history. The numbers alone tell the story:

| Category | Count |
|----------|-------|
| New APIs added | 437 |
| APIs removed | 138 |
| New events | 76 |
| Events removed | 12 |
| New namespaces | 14 (including `C_Housing`, `C_DurationUtil`, `C_CooldownViewer`) |

But numbers undersell the shift. Midnight didn't just add and remove functions — it fundamentally changed *what addon code is for*. Before 12.0, addon code was an interpreter: read game state, make decisions, take actions. After 12.0, addon code is a stylist: receive opaque data, present it beautifully, get out of the way.

### :material-lightning-bolt: The Great Addon Purge of 2026

On March 2, 2026, hundreds of addons broke simultaneously. WeakAuras refused to ship. Hekili became mathematically impossible. GTFO went silent. Rotation helpers, combat log parsers, and data-driven boss mods all hit the Secret Values wall.

But from the wreckage came something unexpected: **innovation under constraint**.

- **New addon categories emerged.** Player housing launched with zero addon support on day one. Within 48 hours, HomeBound had its first release. Within two weeks, an entire housing addon ecosystem existed — decoration managers, layout importers, furniture catalogs.
- **Old patterns evolved.** ElvUI went from "UI replacement" to "UI skin." Details! went from "analytical powerhouse" to "beautiful meter display." Boss mods went from "mechanic solvers" to "timeline enhancers."
- **New patterns were invented.** Duration Objects, the combat queue pattern, swarm aura filtering, secret-safe StatusBar hooks — techniques that didn't exist six months ago are now standard practice.

### :material-clock-fast: The Window Is Open

Blizzard has announced an **API freeze** until Patch 12.0.5. The 437 new APIs, the 76 new events, the Secret Values system — all of it is stable until further notice. Season 1 launches March 17. Mythic raids unlock March 24. Millions of players are settling into Midnight right now, and the addon ecosystem has gaps the size of continents.

There has never been a better time to build a WoW addon.

---

## Before & After — The 12.0 Paradigm Shift

The changes in Midnight aren't incremental. They represent a fundamental paradigm shift in how addons interact with the game. Here are the four biggest transformations, with concrete code comparisons.

### :material-shield-lock: Combat Data: From Raw to Curated

The most dramatic change. Before Midnight, addons had full access to combat data through `COMBAT_LOG_EVENT_UNFILTERED` and raw `UnitHealth()`/`UnitPower()` values. After Midnight, combat data flows through the **Secret Values** system — addons can display it, but cannot read, compare, or branch on it.

=== "Before (11.x)"

    ```lua
    -- Full access to combat events and raw health values
    local frame = CreateFrame("Frame")
    frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
    frame:SetScript("OnEvent", function(self, event)
        local timestamp, subevent, _, sourceGUID, sourceName,
              _, _, destGUID, destName, _, _, spellID, spellName,
              _, amount = CombatLogGetCurrentEventInfo()

        if subevent == "SPELL_DAMAGE" then
            -- Read exact damage numbers
            myDB.totalDamage = myDB.totalDamage + amount

            -- Make decisions based on values
            if amount > 50000 then
                PlaySound(SOUNDKIT.RAID_WARNING)
            end
        end
    end)

    -- Read health directly, do percentage math
    local hp = UnitHealth("target")
    local maxHp = UnitHealthMax("target")
    local pct = hp / maxHp
    if pct < 0.2 then
        ShowExecuteOverlay()
    end
    ```

=== "After (12.0)"

    ```lua
    -- COMBAT_LOG_EVENT_UNFILTERED is gone for addons
    -- Health values are secret in combat contexts
    -- Instead: hook Blizzard's own display and use Duration Objects

    -- Secret-safe health bar enhancement
    hooksecurefunc(healthBar, "SetValue", function(self, value)
        if self:IsForbidden() then return end

        -- Apply class coloring (safe — doesn't read the value)
        local unit = self.unit
        if unit and UnitExists(unit) and UnitIsPlayer(unit) then
            local _, class = UnitClass(unit)
            local color = RAID_CLASS_COLORS[class]
            if color then
                self:SetStatusBarColor(color.r, color.g, color.b)
            end
        end
        -- DO NOT do math on 'value' — it may be a secret
    end)

    -- Duration Objects for cooldown display
    local cooldownDuration = C_Spell.GetSpellCooldownDuration(spellID)
    cooldownFrame:SetCooldownFromDurationObject(cooldownDuration)
    -- The widget handles the secret value internally
    ```

!!! success "What this enables"
    Addons that focus on *presentation* — class-colored health bars, custom textures, restyled cooldown displays — work seamlessly. The data flows through Blizzard's pipeline; you style the output.

!!! danger "What this prevents"
    Automated decision-making based on combat state. No "if health below X, do Y." No combat log parsing for damage meters. No rotation helpers, no predictive healing, no automated callouts.

### :material-message-text: Addon Communication: From Always-On to Context-Aware

=== "Before (11.x)"

    ```lua
    -- Send messages freely at any time
    C_ChatInfo.SendAddonMessage("MYPREFIX", data, "RAID")

    -- Receive and process immediately
    local frame = CreateFrame("Frame")
    frame:RegisterEvent("CHAT_MSG_ADDON")
    frame:SetScript("OnEvent", function(self, event, prefix, msg, channel, sender)
        if prefix == "MYPREFIX" then
            ProcessRaidData(msg, sender) -- Always works
        end
    end)
    ```

=== "After (12.0)"

    ```lua
    -- Messages are throttled/blocked during active encounters in instances
    -- Design for intermittent communication

    local pendingMessages = {}

    local function SafeSendMessage(prefix, data, channel)
        -- Attempt to send; queue if we're in a restricted context
        local success = pcall(C_ChatInfo.SendAddonMessage, prefix, data, channel)
        if not success then
            pendingMessages[#pendingMessages + 1] = {prefix, data, channel}
        end
    end

    -- Flush pending messages when combat ends
    local flusher = CreateFrame("Frame")
    flusher:RegisterEvent("PLAYER_REGEN_ENABLED")
    flusher:SetScript("OnEvent", function()
        for i, msg in ipairs(pendingMessages) do
            C_ChatInfo.SendAddonMessage(unpack(msg))
            pendingMessages[i] = nil
        end
    end)
    ```

!!! info "Design implication"
    Raid coordination addons (MRT, raid assignment tools) must now design for **eventual consistency** rather than real-time sync. Pre-pull data sharing works. Mid-encounter broadcasting does not.

### :material-palette: UI Modification: From Replace to Enhance

=== "Before (11.x)"

    ```lua
    -- Build an entirely parallel UI system
    local myUnitFrame = CreateFrame("Frame", "MyCustomUnitFrame", UIParent)
    local healthBar = CreateFrame("StatusBar", nil, myUnitFrame)
    local powerBar = CreateFrame("StatusBar", nil, myUnitFrame)
    local castBar = CreateFrame("StatusBar", nil, myUnitFrame)
    local nameText = myUnitFrame:CreateFontString(nil, "OVERLAY")

    -- Hide Blizzard's frame (causes taint in 12.0!)
    PlayerFrame:Hide()

    -- Register for all the events yourself
    myUnitFrame:RegisterEvent("UNIT_HEALTH")
    myUnitFrame:RegisterEvent("UNIT_POWER_UPDATE")
    myUnitFrame:RegisterEvent("UNIT_AURA")
    myUnitFrame:RegisterEvent("PLAYER_TARGET_CHANGED")
    -- ... 20+ more events

    -- Manually process everything
    myUnitFrame:SetScript("OnEvent", function(self, event, ...)
        -- Hundreds of lines of event handling
    end)
    ```

=== "After (12.0)"

    ```lua
    -- Enhance Blizzard's existing frame — it handles all the data
    local hiddenFrame = CreateFrame("Frame")
    hiddenFrame:Hide()

    -- Skin the player frame with post-hooks
    hooksecurefunc("PlayerFrame_Update", function(self)
        if self:IsForbidden() then return end

        -- Strip default textures, apply custom look
        for _, region in pairs({self:GetRegions()}) do
            if region.SetTexture then
                region:SetParent(hiddenFrame)
            end
        end

        -- Apply a clean dark backdrop
        if not self.backdropApplied then
            Mixin(self, BackdropTemplateMixin)
            self:SetBackdrop({
                bgFile = "Interface\\Buttons\\WHITE8x8",
                edgeFile = "Interface\\Buttons\\WHITE8x8",
                edgeSize = 1,
            })
            self:SetBackdropColor(0.1, 0.1, 0.1, 0.85)
            self:SetBackdropBorderColor(0, 0, 0, 1)
            self.backdropApplied = true
        end
    end)
    ```

!!! tip "The golden rule"
    Use `hooksecurefunc()` to modify AFTER Blizzard renders. Use `HookScript()` instead of `SetScript()`. Use hidden frame reparenting instead of `:Hide()`. Store original state before modifying. Your addon becomes a cosmetic layer on top of a fully functional Blizzard frame.

### :material-cog: Settings & Configuration: From LibDBIcon to Native

=== "Before (11.x)"

    ```lua
    -- Required three libraries just to have settings
    local AceDB = LibStub("AceDB-3.0")
    local AceConfig = LibStub("AceConfig-3.0")
    local AceConfigDialog = LibStub("AceConfigDialog-3.0")
    local LibDBIcon = LibStub("LibDBIcon-1.0")

    -- Complex options table
    local options = {
        type = "group",
        args = {
            enable = {
                type = "toggle",
                name = "Enable",
                order = 1,
                get = function() return db.enable end,
                set = function(_, v) db.enable = v end,
            },
            scale = {
                type = "range",
                name = "Scale",
                min = 0.5, max = 2.0, step = 0.05,
                order = 2,
                get = function() return db.scale end,
                set = function(_, v) db.scale = v end,
            },
        },
    }
    AceConfig:RegisterOptionsTable("MyAddon", options)
    AceConfigDialog:AddToBlizOptions("MyAddon")

    -- Minimap icon required LibDBIcon + LibDataBroker
    local ldb = LibStub("LibDataBroker-1.1"):NewDataObject("MyAddon", {
        type = "launcher",
        icon = "Interface\\Icons\\INV_Misc_Gear_01",
        OnClick = function() AceConfigDialog:Open("MyAddon") end,
    })
    LibDBIcon:Register("MyAddon", ldb, db.minimap)
    ```

=== "After (12.0)"

    ```lua
    -- Zero libraries needed. Native Settings API + Addon Compartment.
    local category = Settings.RegisterVerticalLayoutCategory("MyAddon")

    -- Toggle
    local enableSetting = Settings.RegisterAddOnSetting(category,
        "MyAddon_Enable", "enable", MyAddonDB,
        type(true), "Enable", true
    )
    Settings.CreateCheckbox(category, enableSetting, "Enable the addon.")

    -- Slider
    local scaleSetting = Settings.RegisterAddOnSetting(category,
        "MyAddon_Scale", "scale", MyAddonDB,
        type(1.0), "Scale", 1.0
    )
    local scaleOpts = Settings.CreateSliderOptions(0.5, 2.0, 0.05)
    scaleOpts:SetLabelFormatter(MinimalSliderWithSteppersMixin.Label.Right)
    Settings.CreateSlider(category, scaleSetting, scaleOpts, "Frame scale.")

    Settings.RegisterAddOnCategory(category)

    -- Minimap button: zero code — just TOC metadata
    -- ## AddonCompartmentFunc: MyAddon_OnClick
    -- ## IconTexture: Interface\Icons\INV_Misc_Gear_01
    ```

!!! success "Why this matters"
    New addons can ship with professional settings panels and minimap integration using **zero external libraries**. No Ace3 dependency. No LibDataBroker. No LibDBIcon. The native API handles layout, scrolling, and persistence automatically.

---

## Impressive Community Addons

Two weeks into Midnight, the community has already produced addons that showcase what's possible under the new rules. These are the standouts — not just for their functionality, but for the *patterns* they demonstrate.

### :material-home: HomeBound — The Housing Pioneer

| | |
|---|---|
| **Downloads** | 3M+ |
| **Category** | Player Housing |
| **Why it matters** | First major addon for an entirely new game system |

Midnight's player housing system launched with no addon API at all. Within the first week, Blizzard added `C_Housing` — a new namespace with 23 functions for querying furniture placement, room state, and visitor management. HomeBound was the first addon to capitalize on it.

HomeBound demonstrates the **"build where Blizzard doesn't"** philosophy. There's no existing UI to replace or enhance — player housing is virgin territory. The addon provides furniture search, layout saving, decoration catalogs, and visitor management that Blizzard's default housing UI simply doesn't offer.

The housing addon category didn't exist before March 2, 2026. Two weeks later, it includes HomeBound, Housing Decor Guide (tracking all 1,690 decoration items), and Advanced Decoration Tools (copy/paste, batch placement, smart rotation). An entire ecosystem built from nothing.

### :material-star-four-points: Platynator — Design Within the Box

| | |
|---|---|
| **Downloads** | Growing rapidly |
| **Category** | Nameplates |
| **Why it matters** | Built from scratch for Midnight constraints |

Platynator is the poster child for creative constraint. Where Plater v635 shipped with a list of features removed (NPC-specific coloring gone, aura-based scaling gone, fixate detection gone), Platynator was designed from the ground up knowing what the rules were.

The result is a nameplate addon that does *less* but feels *better*. Click to customize. Drag to reposition. No LUA scripting required. It targets the aesthetic-focused player who wants beautiful nameplates without touching combat data — and it delivers.

### :material-shield-star: Northern Sky Raid Tools — Filling the WeakAuras Void

| | |
|---|---|
| **Downloads** | 3.2M+ |
| **Category** | Raid Tools |
| **Why it matters** | Replaces WeakAura raid packs within API constraints |

With WeakAuras gone, raid leaders needed something. Northern Sky Raid Tools stepped in with timer-based reminders, assignment displays, and cooldown tracking — all built within the Secret Values framework. It doesn't try to detect boss mechanics through combat parsing. Instead, it provides **configurable timers** that raiders can set up based on known boss ability schedules.

The addon represents a philosophical shift: from "the addon tells you what's happening" to "you tell the addon when things happen, and it helps you communicate that to your team."

### :material-palette-swatch: ElvUI v15.0.0 — The Great Transformation

| | |
|---|---|
| **Downloads** | 200M+ lifetime |
| **Category** | UI Suite |
| **Why it matters** | The most dramatic adaptation in addon history |

ElvUI's journey to Midnight is a story in itself. In October 2025, the team announced they were cancelling Midnight support entirely. In December 2025, they reversed course. On launch day, v15.0.0 shipped — with Style Filters and Portraits removed, but the core UI skin intact.

The transformation from "UI replacement" to "UI skin" required rearchitecting the addon's relationship with Blizzard frames. Where ElvUI previously created parallel frame hierarchies, it now hooks and reskins the default ones. The result is an ElvUI that works *with* Edit Mode, respects combat lockdown, and survives patches without emergency hotfixes.

### :material-timeline-clock: Better Timeline — Enhancing the Native Boss Timeline

| | |
|---|---|
| **Downloads** | Growing |
| **Category** | Boss Encounter |
| **Why it matters** | Enhancement pattern applied to Blizzard's newest system |

Blizzard's native Boss Ability Timeline was designed specifically to replace what boss mods used to do. Better Timeline takes that native timeline and makes it *better* — custom colors, adjustable sizing, ability filtering, and layout options that Blizzard's default doesn't offer.

This is the purest expression of the Enhancement Artist philosophy. Blizzard builds the system. Blizzard provides the data. The addon makes it look and work the way the player wants. No combat parsing needed. No Secret Values conflicts. Just pure UI enhancement.

---

## Techniques That Push Boundaries

These are the advanced patterns that power the most impressive Midnight addons. Each one solves a specific constraint in a creative way.

### :material-hook: The Metatable Hook Pattern

For cases where `hooksecurefunc()` isn't enough — when you need to intercept method calls on *all instances* of a frame type, not just known frames:

```lua
-- Hook ALL StatusBars, including ones created after your addon loads
local StatusBarMeta = getmetatable(CreateFrame("StatusBar")).__index

-- Wrap with pcall for safety — metatable hooks can be fragile
hooksecurefunc(StatusBarMeta, "SetStatusBarColor", function(self, r, g, b, a)
    if self:IsForbidden() then return end
    -- This fires for EVERY StatusBar in the game
    -- Use frame name or parent checks to filter
    if self:GetParent() and self:GetParent().unit then
        -- This is likely a unit frame health/power bar
        -- Apply your enhancement
    end
end)
```

!!! tip "When to use metatable hooks"
    Use them when you need to catch frames that don't exist yet at addon load time — dynamically created nameplates, party frames that appear when groups form, or ScrollBox elements that are created on demand. Always wrap in safety checks.

!!! danger "When NOT to use metatable hooks"
    Never use them to *override* behavior. Only post-hook. And always include `IsForbidden()` checks — metatable hooks will fire on forbidden frames that would crash the client if modified.

### :material-clock-outline: Duration Objects & Secret Values Display

The new way to display time-based combat information without touching secret values directly:

```lua
-- Create a duration display that respects Secret Values
local function CreateDurationDisplay(parent, spellID)
    local display = parent:CreateFontString(nil, "OVERLAY", "GameFontNormal")

    -- Get a Duration Object — it's opaque but displayable
    local function Update()
        local cooldownDuration = C_Spell.GetSpellCooldownDuration(spellID)
        if cooldownDuration then
            -- Pass directly to a Cooldown widget — no math needed
            parent.cooldown:SetCooldownFromDurationObject(cooldownDuration)
        end
    end

    -- Use C_Timer for periodic updates (not OnUpdate — too expensive)
    C_Timer.NewTicker(0.1, Update)
    return display
end
```

!!! info "Duration Objects are the key abstraction"
    `C_DurationUtil.CreateDuration()` and related APIs let you pass timing data to UI widgets without ever reading the underlying numbers. The widget handles the countdown internally. This is how tullaCTC, OmniCC, and CDM addons display cooldown text in Midnight.

### :material-shield-sword: The Combat Queue Pattern

The foundational pattern for any addon that modifies secure frames. Queue operations during combat, flush when combat ends:

```lua
local combatQueue = {}
local isCombat = false

local function RunAfterCombat(fn)
    if InCombatLockdown() then
        combatQueue[#combatQueue + 1] = fn
        return false -- operation was queued
    end
    fn()
    return true -- operation ran immediately
end

-- Flush queue on combat end
local queueFrame = CreateFrame("Frame")
queueFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
queueFrame:RegisterEvent("PLAYER_REGEN_DISABLED")
queueFrame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_DISABLED" then
        isCombat = true
    elseif event == "PLAYER_REGEN_ENABLED" then
        isCombat = false
        for i = 1, #combatQueue do
            local ok, err = pcall(combatQueue[i])
            if not ok then
                -- Log but don't break the queue
                geterrorhandler()(err)
            end
            combatQueue[i] = nil
        end
    end
end)
```

!!! tip "Every enhancement addon needs this"
    If you call `SetPoint()`, `SetParent()`, `SetSize()`, or any other secure frame method during combat lockdown, you will taint the frame and potentially break combat functionality for the player. The combat queue pattern is not optional — it is mandatory.

### :material-filter: Swarm Aura Filtering

Midnight introduced four new aura filter categories that give addons more granular control over buff and debuff display:

```lua
-- New filter categories in 12.0
local AURA_FILTERS = {
    "HELPFUL",                -- Buffs (existing)
    "HARMFUL",                -- Debuffs (existing)
    "PLAYER",                 -- Cast by player (existing)
    "RAID",                   -- Raid-wide auras (existing)
    "CROWD_CONTROL",          -- NEW: Stuns, roots, fears, polymorphs, hex
    "BIG_DEFENSIVE",          -- NEW: Major defensives (Divine Shield, Ice Block)
    "RAID_PLAYER_DISPELLABLE", -- NEW: Debuffs you can dispel with your class
    "RAID_IN_COMBAT",         -- NEW: Auras relevant during combat
}

-- Example: Show only purgeable enemy buffs on nameplates
local function GetPurgeableAuras(unit)
    local auras = {}
    for i = 1, 40 do
        local auraData = C_UnitAuras.GetBuffDataByIndex(unit, i, "HELPFUL|STEALABLE")
        if not auraData then break end
        auras[#auras + 1] = auraData
    end
    return auras
end
```

### :material-recycle: The Object Pool Pattern

For addons that create and destroy many frames dynamically (nameplate indicators, scrolling text, buff icons), object pooling prevents garbage collection stalls:

```lua
local FramePool = {}
FramePool.__index = FramePool

function FramePool:New(frameType, parent, template)
    return setmetatable({
        frameType = frameType,
        parent = parent,
        template = template,
        active = {},
        inactive = {},
    }, self)
end

function FramePool:Acquire()
    local frame = tremove(self.inactive)
    if not frame then
        frame = CreateFrame(self.frameType, nil, self.parent, self.template)
    end
    self.active[frame] = true
    frame:Show()
    return frame
end

function FramePool:Release(frame)
    self.active[frame] = nil
    frame:Hide()
    frame:ClearAllPoints()
    self.inactive[#self.inactive + 1] = frame
end

function FramePool:ReleaseAll()
    for frame in pairs(self.active) do
        self:Release(frame)
    end
end

-- Usage: pooling nameplate indicators
local indicatorPool = FramePool:New("Frame", UIParent, "BackdropTemplate")

hooksecurefunc("CompactUnitFrame_UpdateAuras", function(frame)
    if frame:IsForbidden() then return end
    -- Release old indicators, acquire new ones from pool
    if frame.__indicators then
        for _, indicator in ipairs(frame.__indicators) do
            indicatorPool:Release(indicator)
        end
    end
    frame.__indicators = {}

    -- Create indicators for visible auras
    for i = 1, 4 do
        local indicator = indicatorPool:Acquire()
        indicator:SetParent(frame)
        indicator:SetSize(8, 8)
        indicator:SetPoint("BOTTOMRIGHT", frame, "BOTTOMRIGHT", -2 - (i-1) * 10, 2)
        frame.__indicators[i] = indicator
    end
end)
```

!!! tip "When to pool"
    Use object pooling when frames are created and destroyed frequently — nameplate add-ons, scrolling combat text replacements, dynamic buff displays. For static frames (settings panels, main addon windows), direct `CreateFrame()` is fine.

---

## What's Coming Next

### :material-snowflake: The API Freeze

Blizzard has confirmed that the current API surface is **frozen until Patch 12.0.5**. No new removals. No new restrictions. No new Secret Values categories. The 437 new APIs, the Duration Object system, the Housing namespace — all stable.

This is the development window. Every addon built today will continue working through Season 1, through Mythic race, through the first content patch. The foundation is set.

### :material-crystal-ball: Predicted Developments in 12.0.5

Based on Blizzard's public statements, community manager responses, and patterns from previous expansions:

- **More Housing APIs.** The `C_Housing` namespace shipped with 23 functions. Player housing is Midnight's flagship feature. Expect furniture crafting integration, visitor permissions, and possibly housing plot marketplace APIs.
- **Potential Secret Values relaxation.** Blizzard has loosened Secret Values restrictions multiple times during alpha and beta. Community pressure — particularly from the accessibility community — may lead to further relaxation for personal combat state.
- **Cooldown Manager expansion.** The CDM received major updates in 11.1.5 and 11.2.5. More customization hooks and condition APIs are likely.

### :material-keyboard: WeakAuras: The Conditions for Return

The WeakAuras team has publicly stated three conditions that would allow them to ship a Midnight version:

1. **Secret value arithmetic** — the ability to compute new secret values from existing ones (e.g., `secretA + secretB = secretC`)
2. **Personal combat state exemption** — reverting restrictions for the player's own combat data (not other players')
3. **Complete system reversal** — rolling back Secret Values entirely (deemed unlikely by the team)

As of March 2026, none of these conditions have been met. But the door remains open.

---

## Start Building

### :material-rocket-launch: The Opportunity

The addon ecosystem has gaps everywhere. Player housing needs better tools. The Cooldown Manager needs more skins. Edit Mode needs more registered frames. Nameplates need more enhancers. Boss timelines need more customization. The Settings API is underutilized. The Addon Compartment is half-empty.

Every one of these is an addon waiting to be built.

### :material-map: Where to Go From Here

| Resource | What You'll Find |
|----------|------------------|
| [Plugin Overview](../plugin/overview.md) | How the better-addons AI toolkit works |
| [Coding for Midnight](../midnight-patterns.md) | Copy-pasteable code patterns for every 12.0 scenario |
| [Building Better Addons](../better-addons.md) | The enhancement ecosystem and how to join it |
| [Blizzard Systems](../blizzard-systems.md) | Edit Mode, CDM, Settings API, Addon Compartment |
| [Code Templates](../code-templates.md) | Complete starter templates for common addon types |
| [Cutting Edge](../cutting-edge.md) | The latest news from the addon trenches |

### :material-hammer-wrench: The Best Time to Build

The API is frozen. The player base is massive. The competition is thin. The old guard is diminished or adapting. The tools — including the AI-powered development workflow you're reading about right now — have never been better.

The best addons of the Midnight era haven't been written yet.

Build one.
