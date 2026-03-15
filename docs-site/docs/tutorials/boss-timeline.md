---
title: "Build-Along: BossTimeline+ — Boss Timeline Enhancer"
description: "Advanced tutorial for building a boss ability timeline enhancer in WoW 12.0+ — custom event colors, role-based recommendations, audio alerts, and phase annotations using C_EncounterEvents."
---

# Build-Along: BossTimeline+

:material-clock-time-eight: **35 minutes** | :material-fire: **Difficulty: Advanced** | :material-lightning-bolt: **Mode: Boundary Pusher**

---

## Step 1: What We're Building

WeakAuras refused to ship for Midnight. The team's Patreon statement was unambiguous: *"the core value proposition of WeakAuras isn't compatible with the direction Blizzard is taking the game."* For millions of raiders who relied on WeakAura import strings for encounter preparation, that left a crater.

DBM and BigWigs survived — but dramatically reduced. They can no longer independently detect boss mechanics through combat log parsing. They work as enhancement layers over Blizzard's native boss warnings: custom timers, audio packs, visual reskins. Better than nothing, but a fraction of what they were.

Into this gap, Blizzard shipped the **native Boss Ability Timeline** — a built-in UI element that shows upcoming boss abilities on a scrolling timeline during encounters. It's functional. It's accurate (Blizzard feeds it directly from the encounter engine). And it's *basic*. No color coding by ability type. No role-specific guidance. No custom audio. No phase annotations.

That's what we're building. **BossTimeline+** enhances Blizzard's native timeline with:

- **Color-coded encounter events** — damage abilities in red, healing checks in green, movement mechanics in blue, add spawns in purple, phase transitions in gold
- **Personal ability recommendations** — role-aware suggestions anchored to timeline events ("Use Defensive," "Spread," "Stack," "Interrupt")
- **Audio alerts** — configurable sounds that fire ahead of timeline events, using the `C_CombatAudioAlert` system
- **Phase annotations** — visual separators and strategy notes at phase transition points

This addon operates entirely within Midnight's constraints. We don't parse combat logs (gone). We don't read secret health values (can't). We hook Blizzard's own timeline frame and add cosmetic layers on top. The data comes from `C_EncounterEvents` — Blizzard tells us what's coming; we make it look better and sound better.

### What You'll Learn

| Concept | Why It Matters |
|---------|---------------|
| `C_EncounterEvents` namespace | Brand new in 12.0.1 — the sanctioned way to access encounter event data |
| `C_CombatAudioAlert` | Expanded audio alert system for addons |
| `hooksecurefunc` on encounter frames | Post-hooking Blizzard's timeline UI |
| Secret Values boundaries | What you can and cannot do with encounter data |
| Boundary Pusher safety nets | `pcall` wrappers, `-- BOUNDARY` comments, fallback paths |

!!! info "New APIs — verify before you ship"
    `C_EncounterEvents` is brand new in 12.0.1. API surfaces this fresh can shift in hotfixes. Use `/wow-research C_EncounterEvents` and `/wow-verify` to confirm function signatures before shipping to players. Throughout this tutorial, we'll flag where verification is especially important.

---

## Step 2: The C_EncounterEvents API

### :material-new-box: A New Namespace for a New Era

Before Midnight, boss mod addons like DBM and BigWigs built their encounter knowledge from `COMBAT_LOG_EVENT_UNFILTERED`. They parsed every combat event — spell casts, damage ticks, debuff applications — and reconstructed encounter state from raw data. It was powerful, fragile, and exactly what Blizzard's "Addon Disarmament" targeted.

`C_EncounterEvents` replaces that approach with a **curated, structured API** for encounter information. Instead of parsing raw combat streams, addons receive pre-categorized encounter event data that Blizzard controls. The encounter engine decides what information to expose; addons decide how to present it.

### Key Functions

!!! info "Verify these signatures"
    Use `/wow-research C_EncounterEvents` to confirm these function signatures against the latest 12.0.1 state. New namespaces can receive hotfix changes.

```lua
-- Get all events for the current encounter
-- Returns a table of EncounterEventInfo structures
local events = C_EncounterEvents.GetEventsForEncounter(encounterID)

-- Get events filtered by type
local damageEvents = C_EncounterEvents.GetEventsForEncounter(encounterID,
    Enum.EncounterEventType.SpellCast)

-- Get the current encounter's timeline data
-- Returns ordered event data as displayed on the native timeline
local timelineData = C_EncounterEvents.GetTimelineData()

-- Check if encounter events are available for a given boss
local hasData = C_EncounterEvents.HasEventsForEncounter(encounterID)
```

### Event Types

The `Enum.EncounterEventType` enumeration categorizes encounter events:

| Enum Value | Description | Example |
|------------|-------------|---------|
| `SpellCast` | Boss ability cast | Blast Nova, Shadow Bolt Volley |
| `PhaseTransition` | Phase change | "Phase 2 begins at 60% health" |
| `AddSpawn` | Add wave spawn | "Summoner calls reinforcements" |
| `PeriodicDamage` | Recurring damage pulse | Pulsing AoE, ticking DoT |
| `EnvironmentalEffect` | Environmental hazard | Floor fire, expanding void zone |

### The EncounterEventInfo Structure

Each event returned by `C_EncounterEvents` contains structured data:

```lua
-- EncounterEventInfo fields (verify with /wow-verify)
local event = {
    eventID = 12345,                               -- Unique event identifier
    encounterID = 2902,                             -- Boss encounter ID
    eventType = Enum.EncounterEventType.SpellCast,  -- Category
    spellID = 400215,                               -- Associated spell (if any)
    timeOffset = 45.0,                              -- Seconds from pull
    duration = 3.0,                                 -- Cast/channel duration
    description = "Blast Nova",                     -- Display text
    severityLevel = 2,                              -- 1=minor, 2=medium, 3=critical
}
```

### How This Differs from CLEU

| Aspect | Old (CLEU) | New (C_EncounterEvents) |
|--------|------------|------------------------|
| Data source | Raw combat stream | Curated encounter data |
| Availability | Real-time, every event | Pre-computed timeline |
| Granularity | Every damage tick, every heal | Major abilities only |
| Control | Addon interprets freely | Blizzard controls what's exposed |
| Reliability | Could miss events under load | Guaranteed delivery |
| Patch resilience | Broke constantly | Versioned and stable |

The tradeoff is clear: less power, more stability. For a timeline enhancer, the curated data is exactly what we need.

---

## Step 3: Project Setup

### File Structure

```
Interface/AddOns/BossTimelinePlus/
├── BossTimelinePlus.toc
├── Core.lua
├── Timeline.lua
├── Alerts.lua
└── Config.lua
```

### BossTimelinePlus.toc

```toc
## Interface: 120001
## Title: BossTimeline+
## Notes: Enhances Blizzard's boss ability timeline with colors, recommendations, alerts, and phase annotations.
## Author: YourName
## Version: 1.0.0
## Category: Boss Encounters
## IconTexture: Interface\Icons\Spell_Holy_BorrowedTime
## SavedVariables: BossTimelinePlusDB
## AddonCompartmentFunc: BossTimelinePlus_OnCompartmentClick
## Dependencies: Blizzard_EncounterBar

Core.lua
Timeline.lua
Alerts.lua
Config.lua
```

!!! warning "Dependency on Blizzard_EncounterBar"
    We declare a hard dependency on `Blizzard_EncounterBar` because our addon hooks its frames. If Blizzard renames or restructures this addon in a future patch, our TOC will fail cleanly rather than throwing runtime errors against a missing frame.

---

## Step 4: Hooking Blizzard's Timeline Frame

This is where the Boundary Pusher mode earns its name. We're hooking directly into Blizzard's encounter timeline UI — a frame that didn't exist before 12.0.

### Core.lua — Addon Skeleton and Timeline Discovery

```lua
-- Core.lua
-- Mode: Boundary Pusher | BossTimeline+
-- Hooks Blizzard's native encounter timeline for visual enhancement.

local addonName, ns = ...

-- ============================================================
-- DEFAULTS
-- ============================================================
local defaults = {
    enabled = true,
    colorEvents = true,
    showRecommendations = true,
    audioAlerts = true,
    phaseAnnotations = true,
    alertLeadTime = 5,    -- seconds before event to play alert
    volume = 0.7,
}

-- ============================================================
-- COMBAT QUEUE (mandatory for any frame modification)
-- ============================================================
local combatQueue = {}

function ns:RunAfterCombat(fn)
    if InCombatLockdown() then
        combatQueue[#combatQueue + 1] = fn
        return false
    end
    fn()
    return true
end

local queueFrame = CreateFrame("Frame")
queueFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
queueFrame:SetScript("OnEvent", function()
    for i = 1, #combatQueue do
        local ok, err = pcall(combatQueue[i])
        if not ok then geterrorhandler()(err) end
        combatQueue[i] = nil
    end
end)

-- ============================================================
-- INITIALIZATION
-- ============================================================
local loader = CreateFrame("Frame")
loader:RegisterEvent("ADDON_LOADED")
loader:SetScript("OnEvent", function(self, event, addon)
    if addon ~= addonName then return end
    self:UnregisterEvent("ADDON_LOADED")

    -- Initialize SavedVariables
    if not BossTimelinePlusDB then
        BossTimelinePlusDB = {}
    end
    for k, v in pairs(defaults) do
        if BossTimelinePlusDB[k] == nil then
            BossTimelinePlusDB[k] = v
        end
    end
    ns.db = BossTimelinePlusDB

    -- Discover and hook timeline frame
    ns:HookTimeline()

    -- Register settings
    ns:RegisterConfig()
end)

-- Global compartment handler
function BossTimelinePlus_OnCompartmentClick()
    Settings.OpenToCategory(ns.categoryID)
end
```

### Hooking the Timeline

The native encounter timeline lives inside `EncounterBar`. We hook its update methods to inject our visual layers.

```lua
-- Core.lua (continued)

function ns:HookTimeline()
    -- BOUNDARY: EncounterBar frame structure may change in 12.0.5
    -- Fallback: if frame not found, addon disables gracefully
    local ok, timelineFrame = pcall(function()
        return EncounterBar and EncounterBar.TimelineFrame
    end)

    if not ok or not timelineFrame then
        ns.enabled = false
        return
    end

    ns.timelineFrame = timelineFrame

    -- Post-hook the timeline's event rendering
    -- BOUNDARY: UpdateTimeline method name may change
    local isProcessing = false
    hooksecurefunc(timelineFrame, "UpdateTimeline", function(self)
        if isProcessing then return end
        if self.IsForbidden and self:IsForbidden() then return end
        if not ns.db.enabled then return end

        isProcessing = true

        if ns.db.colorEvents then
            ns:ApplyEventColors(self)
        end
        if ns.db.showRecommendations then
            ns:ApplyRecommendations(self)
        end
        if ns.db.phaseAnnotations then
            ns:ApplyPhaseAnnotations(self)
        end

        isProcessing = false
    end)

    -- Hook event marker creation for individual marker styling
    -- BOUNDARY: CreateEventMarker may not exist — pcall wrap
    local hasMarkerHook = pcall(function()
        hooksecurefunc(timelineFrame, "CreateEventMarker", function(self, eventData)
            if self:IsForbidden() then return end
            ns:StyleEventMarker(self, eventData)
        end)
    end)

    if not hasMarkerHook then
        -- Fallback: hook at the timeline level instead of per-marker
        ns.markerHookFallback = true
    end
end
```

!!! danger "Boundary Technique: pcall-wrapped frame discovery"
    Every access to Blizzard's internal frame structure is wrapped in `pcall()`. If Blizzard renames `EncounterBar.TimelineFrame` in a hotfix, the addon disables itself instead of throwing errors. The `-- BOUNDARY` comments mark exactly where patch breakage is expected.

---

## Step 5: Color-Coding Encounter Events

The first visual enhancement: color-coded timeline markers based on event type. Damage abilities glow red. Healing checks glow green. Movement mechanics glow blue. This transforms the monochrome timeline into an at-a-glance raid awareness tool.

### Timeline.lua — Event Colors

```lua
-- Timeline.lua
-- Mode: Boundary Pusher | BossTimeline+
-- Color-codes encounter timeline events by type.

local addonName, ns = ...

-- ============================================================
-- EVENT TYPE -> COLOR MAPPING
-- ============================================================

local EVENT_COLORS = {
    -- Damage events: red family
    [Enum.EncounterEventType.SpellCast]          = {r = 0.9, g = 0.2, b = 0.2, a = 0.85},
    [Enum.EncounterEventType.PeriodicDamage]     = {r = 0.8, g = 0.3, b = 0.1, a = 0.85},
    [Enum.EncounterEventType.EnvironmentalEffect]= {r = 1.0, g = 0.5, b = 0.0, a = 0.85},

    -- Add spawns: purple
    [Enum.EncounterEventType.AddSpawn]           = {r = 0.6, g = 0.2, b = 0.8, a = 0.85},

    -- Phase transitions: gold
    [Enum.EncounterEventType.PhaseTransition]    = {r = 1.0, g = 0.82, b = 0.0, a = 0.90},
}

-- Default color for uncategorized events
local DEFAULT_COLOR = {r = 0.5, g = 0.5, b = 0.5, a = 0.7}

-- Severity-based intensity multipliers
local SEVERITY_MULT = {
    [1] = 0.6,  -- minor: dimmed
    [2] = 1.0,  -- medium: full color
    [3] = 1.0,  -- critical: full color + glow
}
```

| Event Type | Color | Hex | Rationale |
|------------|-------|-----|-----------|
| Damage (SpellCast) | Red | `#E63333` | Universal "danger" color |
| Periodic Damage | Orange-red | `#CC4D1A` | Ongoing threat, slightly muted |
| Environmental | Orange | `#FF8000` | Positional awareness |
| Add Spawn | Purple | `#9933CC` | Distinct from damage — requires target switch |
| Phase Transition | Gold | `#FFD100` | High visibility — strategic moment |
| Movement (custom) | Blue | `#3399FF` | Calm but important — repositioning |
| Healing Check (custom) | Green | `#33CC66` | Healer awareness — incoming damage |

!!! tip "The color table above is a documentation reference"
    The actual Lua color table is defined at the top of the file. The Markdown table is for your reference when customizing colors.

### Applying Colors to Markers

```lua
-- ============================================================
-- COLOR APPLICATION
-- ============================================================

-- Custom categories for known spells (extensible per encounter)
-- Maps spellID -> custom event category
ns.spellCategories = {
    -- Example: Manaforge encounter abilities
    -- [400215] = "movement",     -- Arcane Shift
    -- [400300] = "healing",      -- Mana Detonation (heal check)
    -- [400188] = "interrupt",    -- Nether Channeling
}

local CUSTOM_COLORS = {
    movement  = {r = 0.2, g = 0.6, b = 1.0, a = 0.85},
    healing   = {r = 0.2, g = 0.8, b = 0.4, a = 0.85},
    interrupt = {r = 1.0, g = 1.0, b = 0.3, a = 0.85},
}

function ns:GetEventColor(eventData)
    -- Check custom spell category first
    if eventData.spellID and ns.spellCategories[eventData.spellID] then
        local category = ns.spellCategories[eventData.spellID]
        return CUSTOM_COLORS[category] or DEFAULT_COLOR
    end

    -- Fall back to event type color
    local color = EVENT_COLORS[eventData.eventType] or DEFAULT_COLOR

    -- Apply severity multiplier
    local mult = SEVERITY_MULT[eventData.severityLevel] or 1.0
    return {
        r = color.r * mult,
        g = color.g * mult,
        b = color.b * mult,
        a = color.a,
    }
end

function ns:ApplyEventColors(timelineFrame)
    -- BOUNDARY: iterating timeline children — structure may shift
    local ok, markers = pcall(function()
        return timelineFrame.eventMarkers or {}
    end)
    if not ok then return end

    for _, marker in pairs(markers) do
        if marker.eventData and not marker:IsForbidden() then
            local color = ns:GetEventColor(marker.eventData)
            -- Apply color overlay to the marker's icon/background
            if marker.Icon then
                marker.Icon:SetVertexColor(color.r, color.g, color.b)
            end
            if marker.Background then
                marker.Background:SetColorTexture(color.r, color.g, color.b, color.a)
            end

            -- Critical events get a glow
            if marker.eventData.severityLevel == 3 then
                ns:ApplyGlow(marker, color)
            end
        end
    end
end

function ns:StyleEventMarker(timelineFrame, eventData)
    -- Called per-marker if the CreateEventMarker hook succeeded
    local color = ns:GetEventColor(eventData)
    -- Individual marker styling handled here when available
end

function ns:ApplyGlow(marker, color)
    if not marker.__btpGlow then
        local glow = marker:CreateTexture(nil, "OVERLAY")
        glow:SetTexture("Interface\\SpellActivationOverlay\\IconAlert")
        glow:SetPoint("CENTER")
        glow:SetSize(marker:GetWidth() * 1.5, marker:GetHeight() * 1.5)
        glow:SetBlendMode("ADD")
        marker.__btpGlow = glow
    end
    marker.__btpGlow:SetVertexColor(color.r, color.g, color.b, 0.4)
    marker.__btpGlow:Show()
end
```

!!! success "Milestone: color-coded timeline"
    At this point, the addon transforms the monochrome timeline into a color-coded ability map. Tanks can spot damage clusters (red). Healers can see healing checks (green). Everyone can see phase transitions (gold). This alone makes the addon useful.

---

## Step 6: Personal Ability Recommendations

The killer feature. Based on the player's role and specialization, BossTimeline+ anchors short text recommendations to timeline events: "Use Defensive," "Spread," "Stack," "Interrupt."

This doesn't use any combat data. It uses **player spec** (always available) and **encounter event metadata** (from `C_EncounterEvents`) to generate static recommendations. The player configures which events get which recommendations.

### Role Detection

```lua
-- Timeline.lua (continued)

-- ============================================================
-- ROLE-BASED RECOMMENDATIONS
-- ============================================================

local ROLE_LABELS = {
    TANK    = "TANK",
    HEALER  = "HEALER",
    DAMAGER = "DPS",
}

local function GetPlayerRole()
    local specIndex = GetSpecialization()
    if not specIndex then return "DPS" end
    local role = GetSpecializationRole(specIndex)
    return ROLE_LABELS[role] or "DPS"
end
```

### Recommendation Data

```lua
-- Recommendation types with icons and text
local RECOMMENDATIONS = {
    defensive  = {text = "Use Defensive",  icon = "Interface\\Icons\\Spell_Holy_SealOfSacrifice",   color = {1, 0.8, 0.2}},
    spread     = {text = "Spread",         icon = "Interface\\Icons\\Ability_Druid_StarFall",        color = {0.3, 0.7, 1}},
    stack      = {text = "Stack",          icon = "Interface\\Icons\\Spell_Holy_AuraOfLight",        color = {0.2, 1, 0.4}},
    interrupt  = {text = "Interrupt",      icon = "Interface\\Icons\\Ability_Kick",                  color = {1, 0.3, 0.3}},
    moveOut    = {text = "Move Out",       icon = "Interface\\Icons\\Ability_Rogue_Sprint",          color = {0.3, 0.7, 1}},
    soak       = {text = "Soak",           icon = "Interface\\Icons\\Spell_Frost_ArcticWinds",       color = {0.5, 0.8, 1}},
    cooldown   = {text = "Raid CD",        icon = "Interface\\Icons\\Spell_Holy_DivineIllumination", color = {1, 0.9, 0.3}},
}

-- Per-encounter, per-role recommendation mappings
-- Key: spellID, Value: table of {role = recommendation_key}
ns.encounterRecommendations = {
    -- Example entries (users extend this via /btpconfig)
    -- [400215] = {TANK = "defensive", HEALER = "cooldown", DPS = "spread"},
    -- [400300] = {TANK = "soak", HEALER = "cooldown", DPS = "moveOut"},
}
```

### Rendering Recommendations

```lua
-- Recommendation frame pool
local recPool = {}

local function AcquireRecFrame(parent)
    local frame = tremove(recPool)
    if not frame then
        frame = CreateFrame("Frame", nil, parent, "BackdropTemplate")
        frame:SetSize(120, 20)
        frame:SetBackdrop({
            bgFile = "Interface\\Buttons\\WHITE8x8",
            edgeFile = "Interface\\Buttons\\WHITE8x8",
            edgeSize = 1,
        })
        frame:SetBackdropColor(0, 0, 0, 0.7)
        frame:SetBackdropBorderColor(0.3, 0.3, 0.3, 0.8)

        frame.icon = frame:CreateTexture(nil, "ARTWORK")
        frame.icon:SetSize(16, 16)
        frame.icon:SetPoint("LEFT", 2, 0)

        frame.text = frame:CreateFontString(nil, "OVERLAY", "GameFontNormalSmall")
        frame.text:SetPoint("LEFT", frame.icon, "RIGHT", 4, 0)
    end
    frame:SetParent(parent)
    frame:Show()
    return frame
end

local function ReleaseRecFrame(frame)
    frame:Hide()
    frame:ClearAllPoints()
    recPool[#recPool + 1] = frame
end

function ns:ApplyRecommendations(timelineFrame)
    -- Release existing recommendation frames
    if ns.activeRecs then
        for _, frame in ipairs(ns.activeRecs) do
            ReleaseRecFrame(frame)
        end
    end
    ns.activeRecs = {}

    local role = GetPlayerRole()

    -- BOUNDARY: accessing eventMarkers — pcall safety
    local ok, markers = pcall(function()
        return timelineFrame.eventMarkers or {}
    end)
    if not ok then return end

    for _, marker in pairs(markers) do
        if marker.eventData and not marker:IsForbidden() then
            local spellID = marker.eventData.spellID
            local recs = ns.encounterRecommendations[spellID]

            if recs and recs[role] then
                local recKey = recs[role]
                local recData = RECOMMENDATIONS[recKey]
                if recData then
                    local recFrame = AcquireRecFrame(marker)
                    recFrame:SetPoint("TOP", marker, "BOTTOM", 0, -2)
                    recFrame.icon:SetTexture(recData.icon)
                    recFrame.text:SetText(recData.text)
                    recFrame.text:SetTextColor(unpack(recData.color))
                    ns.activeRecs[#ns.activeRecs + 1] = recFrame
                end
            end
        end
    end
end
```

!!! tip "Making recommendations extensible"
    The `encounterRecommendations` table is intentionally sparse. Players populate it through the config UI (Step 9) or by editing SavedVariables directly. Community-maintained recommendation packs can be distributed as simple Lua tables — no WeakAura import strings needed.

---

## Step 7: Audio Alert System

Configurable audio alerts that fire a set number of seconds before a timeline event occurs. This uses `PlaySound()` with built-in sound kit IDs and optionally the `C_CombatAudioAlert` namespace for integration with Blizzard's own alert categories.

### Alerts.lua

```lua
-- Alerts.lua
-- Mode: Boundary Pusher | BossTimeline+
-- Fires audio alerts ahead of timeline events.

local addonName, ns = ...

-- ============================================================
-- SOUND MAPPING
-- ============================================================

-- Sound kit IDs for different event severities
local ALERT_SOUNDS = {
    minor    = SOUNDKIT.UI_RAID_BOSS_WHISPER_WARNING or 37666,
    medium   = SOUNDKIT.RAID_WARNING or 8959,
    critical = SOUNDKIT.UI_RAID_BOSS_DEFEATED or 34132,
}

-- Custom sounds per event type
local EVENT_SOUNDS = {
    [Enum.EncounterEventType.SpellCast]       = ALERT_SOUNDS.medium,
    [Enum.EncounterEventType.PhaseTransition] = ALERT_SOUNDS.critical,
    [Enum.EncounterEventType.AddSpawn]        = ALERT_SOUNDS.medium,
    [Enum.EncounterEventType.PeriodicDamage]  = ALERT_SOUNDS.minor,
    [Enum.EncounterEventType.EnvironmentalEffect] = ALERT_SOUNDS.minor,
}

-- ============================================================
-- ALERT SCHEDULING
-- ============================================================

local scheduledAlerts = {}

function ns:ScheduleAlerts(encounterID)
    ns:ClearAlerts()

    if not ns.db.audioAlerts then return end

    -- BOUNDARY: C_EncounterEvents may not be available — pcall
    local ok, events = pcall(C_EncounterEvents.GetEventsForEncounter, encounterID)
    if not ok or not events then return end

    local leadTime = ns.db.alertLeadTime or 5

    for _, eventData in ipairs(events) do
        -- Schedule alert (leadTime) seconds before the event
        local alertTime = eventData.timeOffset - leadTime
        if alertTime > 0 then
            local timer = C_Timer.NewTimer(alertTime, function()
                ns:FireAlert(eventData)
            end)
            scheduledAlerts[#scheduledAlerts + 1] = timer
        end
    end
end

function ns:FireAlert(eventData)
    if not ns.db.audioAlerts then return end

    local soundID = EVENT_SOUNDS[eventData.eventType] or ALERT_SOUNDS.minor

    -- Play at configured volume
    PlaySound(soundID, "Master", false, false)

    -- For critical events, also use C_CombatAudioAlert if available
    -- BOUNDARY: C_CombatAudioAlert expanded in 12.0.1 — verify availability
    if eventData.severityLevel == 3 then
        local alertOk = pcall(function()
            if C_CombatAudioAlert and C_CombatAudioAlert.PlayAlert then
                C_CombatAudioAlert.PlayAlert(
                    Enum.CombatAudioAlertSeverity.Critical
                )
            end
        end)
        -- Fallback: already played via PlaySound above
    end
end

function ns:ClearAlerts()
    for _, timer in ipairs(scheduledAlerts) do
        timer:Cancel()
    end
    wipe(scheduledAlerts)
end

-- ============================================================
-- ENCOUNTER LIFECYCLE
-- ============================================================

local encounterFrame = CreateFrame("Frame")
encounterFrame:RegisterEvent("ENCOUNTER_START")
encounterFrame:RegisterEvent("ENCOUNTER_END")
encounterFrame:SetScript("OnEvent", function(self, event, encounterID, ...)
    if event == "ENCOUNTER_START" then
        ns:ScheduleAlerts(encounterID)
    elseif event == "ENCOUNTER_END" then
        ns:ClearAlerts()
    end
end)
```

!!! info "ENCOUNTER_START and ENCOUNTER_END survive Midnight"
    These events are explicitly whitelisted by Blizzard. They fire when boss encounters begin and end, providing the encounter ID, encounter name, difficulty, and group size. They do NOT provide combat data — they're lifecycle hooks.

!!! danger "Boundary Technique: C_CombatAudioAlert"
    The `C_CombatAudioAlert` namespace was expanded in 12.0.1 but its exact surface may shift. Every call is wrapped in `pcall()` with a `PlaySound()` fallback. If Blizzard changes the alert API, the addon degrades to standard sound kit playback.

---

## Step 8: Phase Annotations

Phase transitions are the most important moments in a raid encounter. BossTimeline+ draws visual separators at phase boundaries and optionally displays strategy notes.

```lua
-- Timeline.lua (continued)

-- ============================================================
-- PHASE ANNOTATIONS
-- ============================================================

local phasePool = {}

local function AcquirePhaseFrame(parent)
    local frame = tremove(phasePool)
    if not frame then
        frame = CreateFrame("Frame", nil, parent)
        frame:SetSize(2, 1)  -- thin vertical line, height set dynamically

        frame.line = frame:CreateTexture(nil, "OVERLAY")
        frame.line:SetColorTexture(1.0, 0.82, 0.0, 0.6)  -- gold
        frame.line:SetAllPoints()

        frame.label = frame:CreateFontString(nil, "OVERLAY", "GameFontNormalSmall")
        frame.label:SetPoint("BOTTOM", frame, "TOP", 0, 4)
        frame.label:SetTextColor(1.0, 0.82, 0.0)

        frame.note = frame:CreateFontString(nil, "OVERLAY", "GameFontNormalTiny")
        frame.note:SetPoint("TOP", frame, "BOTTOM", 0, -2)
        frame.note:SetTextColor(0.8, 0.8, 0.8)
        frame.note:SetWidth(100)
        frame.note:SetJustifyH("CENTER")
    end
    frame:SetParent(parent)
    frame:Show()
    return frame
end

local function ReleasePhaseFrame(frame)
    frame:Hide()
    frame:ClearAllPoints()
    phasePool[#phasePool + 1] = frame
end

-- Phase strategy notes (configurable, per encounter)
ns.phaseNotes = {
    -- Example: [encounterID] = { [phaseNumber] = "note text" }
    -- [2902] = { [2] = "Bloodlust here", [3] = "Spread for final phase" },
}

function ns:ApplyPhaseAnnotations(timelineFrame)
    -- Release existing phase markers
    if ns.activePhases then
        for _, frame in ipairs(ns.activePhases) do
            ReleasePhaseFrame(frame)
        end
    end
    ns.activePhases = {}

    -- BOUNDARY: accessing eventMarkers for phase events
    local ok, markers = pcall(function()
        return timelineFrame.eventMarkers or {}
    end)
    if not ok then return end

    local timelineHeight = timelineFrame:GetHeight()
    local phaseCount = 0

    for _, marker in pairs(markers) do
        if marker.eventData
           and marker.eventData.eventType == Enum.EncounterEventType.PhaseTransition
           and not marker:IsForbidden() then

            phaseCount = phaseCount + 1
            local phaseFrame = AcquirePhaseFrame(timelineFrame)

            -- Position at the marker's X coordinate, spanning full timeline height
            phaseFrame:SetHeight(timelineHeight)
            phaseFrame:SetPoint("CENTER", marker, "CENTER", 0, 0)

            -- Label
            phaseFrame.label:SetText("Phase " .. phaseCount + 1)

            -- Strategy note (if configured)
            local encounterID = marker.eventData.encounterID
            local notes = ns.phaseNotes[encounterID]
            if notes and notes[phaseCount + 1] then
                phaseFrame.note:SetText(notes[phaseCount + 1])
                phaseFrame.note:Show()
            else
                phaseFrame.note:Hide()
            end

            ns.activePhases[#ns.activePhases + 1] = phaseFrame
        end
    end
end
```

!!! success "Milestone: complete visual enhancement"
    With phase annotations added, the timeline now shows color-coded abilities, role-specific recommendations, and gold phase separators with strategy notes. The visual transformation from Blizzard's default is dramatic — and every bit of it is combat-safe.

---

## Step 9: Secret Values Considerations

BossTimeline+ is deliberately designed to operate within Midnight's constraints. Here's what works, what doesn't, and why our architecture avoids the problem entirely.

### :material-check-circle: What Works

| Feature | Why It's Safe |
|---------|--------------|
| `C_EncounterEvents` data | Curated by Blizzard — not raw combat data |
| Encounter timers (`timeOffset`) | Pre-computed offsets, not real-time measurements |
| Phase detection | From `Enum.EncounterEventType.PhaseTransition`, not health thresholds |
| `ENCOUNTER_START` / `ENCOUNTER_END` | Explicitly whitelisted lifecycle events |
| `GetSpecialization()` / `GetSpecializationRole()` | Player identity data — never restricted |
| `PlaySound()` | Audio playback — unaffected by Secret Values |
| Visual overlays on timeline markers | Cosmetic-only — modifies presentation, not data |

### :material-close-circle: What Doesn't Work

| Feature | Why It Fails |
|---------|-------------|
| Reading exact boss health to trigger phase alerts | Health values are secret in combat |
| Comparing damage numbers for severity ranking | Damage amounts are secret |
| Conditional triggers ("if debuff count > 3, alert") | Cannot branch on secret values |
| Real-time encounter state reconstruction | No CLEU, no raw combat data |
| Dynamic timer adjustment based on DPS | Cannot measure DPS |

### :material-shield-check: Our Architecture Avoids the Problem

```
┌───────────────────────────────────────────────────┐
│ Blizzard Encounter Engine                         │
│  └─ C_EncounterEvents (curated, pre-computed)     │
│     └─ BossTimeline+ reads event metadata         │
│        └─ Applies colors (cosmetic)               │
│        └─ Anchors recommendations (static data)   │
│        └─ Schedules audio (timer-based)            │
│        └─ Draws phase markers (from event type)    │
│                                                   │
│  EncounterBar.TimelineFrame (Blizzard renders)    │
│     └─ BossTimeline+ hooks AFTER render           │
│        └─ Adds visual layers on top               │
│        └─ Never reads secret data                 │
│        └─ Never branches on combat state          │
└───────────────────────────────────────────────────┘
```

!!! warning "Secret Values are not a limitation for this addon"
    Because BossTimeline+ operates on **pre-computed encounter metadata** rather than real-time combat data, Secret Values don't constrain us at all. We read event type, spell ID, time offset, severity, and description — none of which are secret. The timeline's visual content is rendered by Blizzard; we only add cosmetic overlays.

---

## Step 10: Configuration and Complete Code

### Config.lua — Settings Panel

```lua
-- Config.lua
-- Mode: Boundary Pusher | BossTimeline+
-- Native Settings API configuration.

local addonName, ns = ...

function ns:RegisterConfig()
    local category = Settings.RegisterVerticalLayoutCategory("BossTimeline+")
    ns.categoryID = category:GetID()

    -- Master toggle
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BTP_Enabled", "enabled", ns.db,
            type(true), "Enable BossTimeline+", true
        )
        Settings.CreateCheckbox(category, setting,
            "Enable all BossTimeline+ enhancements on the boss ability timeline.")
    end

    -- Color events
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BTP_Colors", "colorEvents", ns.db,
            type(true), "Color-Code Events", true
        )
        Settings.CreateCheckbox(category, setting,
            "Apply color coding to timeline events by type (damage, adds, phases).")
    end

    -- Recommendations
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BTP_Recs", "showRecommendations", ns.db,
            type(true), "Show Ability Recommendations", true
        )
        Settings.CreateCheckbox(category, setting,
            "Display role-based recommendations beneath timeline events.")
    end

    -- Audio alerts
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BTP_Audio", "audioAlerts", ns.db,
            type(true), "Audio Alerts", true
        )
        Settings.CreateCheckbox(category, setting,
            "Play audio alerts before upcoming encounter events.")
    end

    -- Phase annotations
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BTP_Phases", "phaseAnnotations", ns.db,
            type(true), "Phase Annotations", true
        )
        Settings.CreateCheckbox(category, setting,
            "Show phase separators and strategy notes on the timeline.")
    end

    -- Alert lead time
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "BTP_LeadTime", "alertLeadTime", ns.db,
            type(1), "Alert Lead Time (seconds)", 5
        )
        local options = Settings.CreateSliderOptions(1, 15, 1)
        options:SetLabelFormatter(MinimalSliderWithSteppersMixin.Label.Right)
        Settings.CreateSlider(category, setting, options,
            "How many seconds before an event to play the audio alert.")
    end

    Settings.RegisterAddOnCategory(category)
end
```

### Complete File Summary

!!! success "What you've built"
    A four-file addon that transforms Blizzard's basic boss ability timeline into a rich, role-aware, color-coded, audio-enhanced raid preparation tool — all within Midnight's API constraints.

| File | Lines | Responsibility |
|------|-------|---------------|
| `Core.lua` | ~90 | Initialization, SavedVariables, combat queue, timeline discovery and hooking |
| `Timeline.lua` | ~200 | Event color mapping, recommendation rendering, phase annotations, object pooling |
| `Alerts.lua` | ~80 | Audio alert scheduling, encounter lifecycle, C_CombatAudioAlert integration |
| `Config.lua` | ~70 | Native Settings API panel with six controls |

### Installation

1. Copy the `BossTimelinePlus/` folder to `Interface/AddOns/`
2. `/reload` in-game
3. Enter a boss encounter — the timeline will render with colors, recommendations, and phase markers
4. Open Settings > Addons > BossTimeline+ to configure

### Extending the Addon

The two data tables that drive customization are:

- **`ns.encounterRecommendations`** — maps `spellID` to `{ROLE = "recommendation_key"}`. Populate this for each raid boss.
- **`ns.phaseNotes`** — maps `encounterID` to `{[phaseNum] = "strategy note"}`. Add per-encounter strategy text.

Community recommendation packs can be distributed as simple Lua files that populate these tables:

```lua
-- ManaforgeRecommendations.lua (community pack example)
local _, ns = ...

-- Manaforge Atrium encounter recommendations
ns.encounterRecommendations[400215] = {TANK = "defensive", HEALER = "cooldown", DPS = "spread"}
ns.encounterRecommendations[400300] = {TANK = "soak",      HEALER = "cooldown", DPS = "moveOut"}
ns.encounterRecommendations[400188] = {TANK = "defensive", HEALER = "healing",  DPS = "interrupt"}

ns.phaseNotes[2902] = {
    [2] = "Bloodlust here — burn boss before adds spawn",
    [3] = "Spread for Arcane Detonation — healers top the raid",
}
```

!!! tip "Think of this as the new WeakAura import string"
    Before Midnight, raid leaders shared WeakAura packs. In 12.0+, they can share recommendation tables — small Lua snippets that plug into BossTimeline+ and give role-specific guidance for each encounter. Same workflow, within the rules.

---

## Where to Go Next

| Resource | Link |
|----------|------|
| Verify C_EncounterEvents API | Use `/wow-research C_EncounterEvents` |
| Verify code patterns | Use `/wow-verify` on your hooks |
| Boundary Pusher mode reference | [Modes: Boundary Pusher](../plugin/modes.md) |
| Midnight coding patterns | [Coding for Midnight](../midnight-patterns.md) |
| Enhancement tutorials | [Enhancement Tutorials](../enhancement-tutorials.md) |
| Cutting-edge addon news | [Cutting Edge](../cutting-edge.md) |
