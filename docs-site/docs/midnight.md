# Midnight (Patch 12.0) — What Addon Developers Need to Know

Patch 12.0 (Midnight) introduced the **largest set of addon API changes since WoW launched**. This page covers everything addon developers need to understand: the new combat restrictions, the Secret Values system, deprecated API removals, migration strategies, and how to build addons that thrive under the new rules.

---

## Interface Number

**Current value for WoW 12.0.1 (Midnight):** `120001`

Update your `.toc` file:

```toc
## Interface: 120001
```

!!! warning "Interface number is required"
    The client flags addons with outdated Interface numbers. Users must manually enable "Load out of date addons" to use them — many won't. Always ship with the current Interface number.

---

## The "Skin, Don't Replace" Philosophy

Midnight's addon policy can be summarized in one sentence:

> **Addons can change how information looks, but not what information the player acts on.**

Blizzard's design goals:

1. **Level the playing field** — addons should not provide competitive advantages in combat over players using the default UI.
2. **Preserve personalization** — visual customization of raid frames, nameplates, action bars, fonts, and art remains fully supported.
3. **Combat is a black box** — real-time combat data can be *displayed* by addons but not *interpreted* for automated decision-making.

This means the era of addons "solving" encounter mechanics (automated rotation helpers, boss callouts with actionable instructions) is over. Addons that *skin* and *restyle* the UI continue to work.

---

## Secret Values System

Instead of outright removing APIs or moving frames into the secure environment, Blizzard introduced a new technology called **Secret Values**.

### How Secret Values Work

- Combat-related data (health changes, damage events, spell casts) is tagged as a **secret value**.
- Addons can **display** secret values (bind them to UI elements like health bars, text strings, or progress indicators).
- Addons **cannot read, compare, or branch on** secret values in Lua logic.

Think of it as a one-way pipe: data flows from the game engine into UI widgets, but addons cannot inspect what's in the pipe.

```lua
-- BEFORE Midnight: addons could read and act on combat data
local health = UnitHealth("target")
if health < 1000 then
    -- trigger some automated response
end

-- AFTER Midnight: the value is "secret" in combat contexts
-- You can still bind it to a StatusBar or FontString for display,
-- but conditional logic based on its value is restricted.
```

### What Secret Values Affect

| Data Type | Can Display? | Can Branch On? |
|-----------|:---:|:---:|
| Unit health/power bars | Yes | No |
| Cast bar progress | Yes | No |
| Buff/debuff icons & durations | Yes | No |
| Combat log event details | No (see CLEU section) | No |
| Spell cooldowns (non-whitelisted) | Restricted | No |
| Nameplate threat/cast indicators | Yes | No |

!!! info "Evolving system"
    Blizzard evaluates which APIs can accept secret values on a case-by-case basis. The restrictions have been loosened multiple times since Alpha. Check the [Patch 12.0.0/API changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) page for the latest state.

---

## COMBAT_LOG_EVENT_UNFILTERED (CLEU) — Removed for Addons

The single biggest breaking change in Midnight:

!!! danger "CLEU is gone"
    `COMBAT_LOG_EVENT_UNFILTERED` and `CombatLogGetCurrentEventInfo()` are no longer available to addons. Combat Log chat tab messages have been converted to **KStrings** that addons cannot parse.

### What This Breaks

- **Boss mods** (DBM, BigWigs) — can no longer parse combat events for mechanic detection
- **Damage meters** (Details!, Recount) — lost granular real-time parsing
- **Combat log recorders** (Warcraft Logs uploader) — affected but accommodated (see below)
- **WeakAuras triggers** — combat event-based triggers no longer function
- **Threat meters** — cannot compute threat from combat events

### What Replaces It

Blizzard has built native UI systems to replace the most critical addon functionality:

| Former Addon Feature | Native Replacement |
|---------------------|-------------------|
| Boss encounter timers | Built-in boss ability timeline |
| Damage/healing meters | Native damage meter (basic) |
| Nameplate customization | Nameplate presets (Classic, Modern, Bold) with threat indicators |
| Important cast alerts | Native "important cast" visual cues |
| Target/focus warnings | Built-in target alerts |

!!! warning "Native damage meter limitations"
    The built-in damage meter does not persist across logouts and lacks the breakdown granularity that raiders and M+ players expect. Blizzard has acknowledged this and is iterating on improvements.

### Relaxations for Logging Addons

Blizzard acknowledged the initial restrictions were too broad. Combat log data access has been partially restored for **post-combat analysis**:

- Damage meters can still aggregate and display data, though with less real-time granularity
- `CHAT_MSG_ADDON_LOGGED` continues to work for combat log recording tools
- Restrictions are **tightest during active encounters** (raid boss fights, M+ timed runs) and relax between pulls

---

## Spell Whitelist System

Not all spell data is equally restricted. Blizzard maintains a **whitelist** of spells whose state, charges, and cooldowns are fully accessible to addons.

### Whitelisted Categories

- **Class resources** that the base UI already tracks (e.g., Enhancement Shaman Maelstrom Weapon stacks, Devoker Demon Hunter Soul Fragments)
- **Major cooldowns** that Blizzard deems appropriate for addon tracking
- **Empowered cast data** — number of stages and percentage of cast time per stage have been un-restricted

### Requesting Whitelist Additions

Blizzard evaluates requests on a case-by-case basis. To request a spell be whitelisted:

1. Post in the [UI and Macro forum](https://us.forums.blizzard.com/en/wow/c/ui-and-macro/) with a clear justification
2. Explain *why* addon access to this spell is needed for UI presentation (not competitive advantage)
3. Reference similar whitelisted spells if applicable

!!! info "Community-driven process"
    The whitelist has grown significantly since Alpha. Check the [Wowhead addon changes article](https://www.wowhead.com/news/majority-of-addon-changes-finalized-for-midnight-pre-patch-whitelisted-spells-379738) for the latest list.

---

## Addon Messaging Restrictions in Instances

!!! danger "Breaking change for raid/dungeon addons"
    Addon-to-addon communication via `C_ChatInfo.SendAddonMessage()` is restricted during **active encounters** in instances (raid boss fights, M+ timed runs).

### What Changed

- **During active encounters**: addon messaging channels are throttled or blocked to prevent addons from sharing parsed combat data between players
- **Between encounters**: addon messaging works normally (loot councils, break timers, keystone sharing, raid note distribution)
- **Outside instances**: no changes

### Migration Strategy

```lua
-- Check if messaging is available before sending
local function SafeSendAddonMessage(prefix, message, channel)
    -- Queue messages during encounters, send after
    if IsEncounterInProgress() then
        -- Store for later
        tinsert(ns.pendingMessages, {prefix, message, channel})
        return false
    end
    C_ChatInfo.SendAddonMessage(prefix, message, channel)
    return true
end

-- Flush pending messages when encounter ends
local frame = CreateFrame("Frame")
frame:RegisterEvent("ENCOUNTER_END")
frame:SetScript("OnEvent", function()
    for _, msg in ipairs(ns.pendingMessages) do
        C_ChatInfo.SendAddonMessage(unpack(msg))
    end
    wipe(ns.pendingMessages)
end)
```

### SendAddonMessage Throttling

The existing throttle (10 messages per prefix, regenerating at 1/second) remains, but enforcement is stricter in 12.0. Batch and compress data aggressively:

- Use [LibSerialize](https://www.curseforge.com/wow/addons/libserialize) + [LibDeflate](https://www.curseforge.com/wow/addons/libdeflate) for payload compression
- Chunk messages into 255-byte segments with sequence numbers
- Avoid sending redundant data — diff against previously sent state

---

## Deprecated APIs Removed in 12.0

All functions deprecated during the 11.x cycle have been **fully removed** in Patch 12.0.0. If your addon still references any of these, it will error on load.

### Removed API Categories

| Category | Examples of Removed Functions | Replacement |
|----------|------------------------------|-------------|
| **BattleNet** | Legacy `BNGetFriendInfo()` variants | `C_BattleNet` namespace |
| **ChatInfo** | Old chat channel functions | `C_ChatInfo` namespace |
| **ChatFrame** | Direct chat frame manipulation | Updated ChatFrame API |
| **CombatLog** | `CombatLogGetCurrentEventInfo()` | Secret Values / native UI |
| **SpellBook** | `GetSpellInfo()`, old spell query functions | `C_Spell.GetSpellInfo()`, `C_Spell.GetSpellCooldown()` |
| **InstanceEncounter** | Legacy encounter functions | `C_EncounterJournal` namespace |
| **SpellScript** | Old spell scripting API | Updated spell API |

### Common Migration Examples

**Spell information:**

```lua
-- REMOVED in 12.0:
-- local name, rank, icon = GetSpellInfo(spellID)

-- Use instead:
local info = C_Spell.GetSpellInfo(spellID)
if info then
    local name = info.name
    local icon = info.iconID
end
```

**Spell cooldowns:**

```lua
-- REMOVED in 12.0:
-- local start, duration, enabled = GetSpellCooldown(spellID)

-- Use instead:
local cooldown = C_Spell.GetSpellCooldown(spellID)
if cooldown then
    local start = cooldown.startTime
    local duration = cooldown.duration
end
```

**Addon metadata:**

```lua
-- Old style (still works but prefer C_ namespace):
-- local version = GetAddOnMetadata("MyAddon", "Version")

-- Preferred:
local version = C_AddOns.GetAddOnMetadata("MyAddon", "Version")
```

!!! tip "Finding replacements"
    The [Warcraft Wiki API changes page](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) documents every removed function and its replacement. The in-game `/api` command (from Blizzard's Documentation addon) also reflects the current API surface.

---

## Frame Manipulation Changes

### The Container Skinning Model

Midnight shifts addon UI work toward **skinning Blizzard-provided containers** rather than creating entirely custom replacements for core UI elements.

**What this means in practice:**

- Blizzard provides data containers (unit frames, cast bars, nameplate widgets) with structured data bindings
- Addons can restyle these containers: change textures, fonts, sizes, colors, layouts
- Addons should **not** create wholly independent frames that duplicate Blizzard's data pipelines

```lua
-- PREFERRED: Skin an existing Blizzard nameplate
hooksecurefunc("CompactUnitFrame_UpdateAll", function(frame)
    if not frame:IsForbidden() then
        -- Restyle the frame
        frame.healthBar:SetStatusBarTexture("Interface\\AddOns\\MyAddon\\statusbar")
        frame.name:SetFont("Fonts\\FRIZQT__.TTF", 10, "OUTLINE")
    end
end)

-- AVOID: Creating parallel frames that try to read
-- the same data through now-restricted APIs
```

### Protected Frame Changes

The rules for secure frame manipulation during combat remain the same but are **enforced more strictly** in 12.0:

- `frame:Show()`, `frame:Hide()`, `frame:SetPoint()`, `frame:SetAttribute()` on secure frames remain blocked in combat
- New frames created by addons that interact with combat data may be subject to additional restrictions
- Always use the deferred action pattern (queue changes for `PLAYER_REGEN_ENABLED`)

### Nameplate Customization

Nameplates received significant native improvements in Midnight:

- Multiple built-in presets (Classic-inspired, Bold Purple, Modern)
- Native threat visualization (fire effect when pulling aggro)
- Addons can still customize nameplate appearance but with tighter bounds on what data they can read from nameplate units during combat

!!! warning "Nameplate depth at risk"
    The depth of nameplate customization available to addons is considered **at-risk** and may change in 12.0.x patches. Test thoroughly and monitor API change summaries.

---

## Updating Existing Addons for Midnight

### Migration Checklist

Use this checklist when updating an addon from 11.x to 12.0:

- [ ] **Update `## Interface:` to `120001`**
- [ ] **Search for removed API calls** — grep your codebase for functions from the [removed API list](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes)
- [ ] **Replace deprecated spell functions** — `GetSpellInfo()` → `C_Spell.GetSpellInfo()`, etc.
- [ ] **Remove CLEU dependencies** — any `COMBAT_LOG_EVENT_UNFILTERED` handlers must be removed or replaced with native UI hooks
- [ ] **Audit addon messaging** — ensure `SendAddonMessage()` calls handle in-encounter restrictions gracefully
- [ ] **Remove private addon channel usage in instances** — private channels in instances are restricted; use `"PARTY"` or `"RAID"` channels instead
- [ ] **Replace direct frame creation with container skinning** — where possible, hook and restyle Blizzard frames rather than creating parallel UI elements
- [ ] **Test combat functionality** — combat-related features are the most likely to be broken; test in dungeons and raids
- [ ] **Check the spell whitelist** — if your addon tracks specific spell cooldowns/charges, verify they are whitelisted or request access
- [ ] **Test addon communication** — verify your addon's messaging works correctly both during and between encounters

### Automated API Audit

A quick way to find potential issues in your codebase:

```bash
# Search for removed/deprecated function calls
grep -rn "GetSpellInfo\|GetSpellCooldown\|CombatLogGetCurrentEventInfo" MyAddon/
grep -rn "COMBAT_LOG_EVENT_UNFILTERED" MyAddon/
grep -rn "GetAddOnMetadata" MyAddon/  # Should use C_AddOns.GetAddOnMetadata

# Search for direct frame manipulation patterns that may need updating
grep -rn "CreateFrame.*SecureActionButtonTemplate" MyAddon/
grep -rn ":SetAttribute(" MyAddon/
```

---

## Building New Addons for Midnight

If you're starting a new addon in 12.0, design around these principles from the start:

### 1. Skin, Don't Replace

Focus on restyling existing Blizzard UI elements rather than building parallel systems:

```lua
-- Good: Hook and restyle Blizzard's cast bar
hooksecurefunc(CastingBarFrame, "OnUpdate", function(self)
    -- Customize appearance
end)

-- Avoid: Building your own cast bar that reads spell data directly
```

### 2. Focus on Gaps

Build in areas Blizzard doesn't natively support well:

- **Inventory management** — bag sorting, item categorization, equipment sets
- **Auction house tools** — pricing, batch posting, market analysis
- **Social features** — guild management, group finding enhancements
- **Quality of life** — auto-repair, vendor trash selling, coordinate display
- **Out-of-combat information** — achievement tracking, collection management, transmog tools

### 3. Use Blizzard Data Containers

When you need to display combat-adjacent information, use Blizzard's provided data containers rather than trying to extract raw data:

```lua
-- Use Blizzard's StatusBar with built-in data binding
local bar = CreateFrame("StatusBar", nil, parent)
bar:SetMinMaxValues(0, 100)
-- Let Blizzard's systems update the value through proper channels
```

### 4. Plan for Combat Lockdown

Structure your addon so UI configuration happens entirely outside of combat:

```lua
local addonName, ns = ...

function ns:ApplyLayout()
    if InCombatLockdown() then
        ns.layoutPending = true
        return
    end
    -- Do frame positioning, showing/hiding, attribute setting here
end

local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_REGEN_ENABLED")
frame:SetScript("OnEvent", function()
    if ns.layoutPending then
        ns.layoutPending = false
        ns:ApplyLayout()
    end
end)
```

### 5. Stay Updated

The 12.0.x patch cycle will continue to adjust API restrictions:

- Monitor the [Warcraft Wiki API changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) page
- Follow the [WoW UI and Macro forum](https://us.forums.blizzard.com/en/wow/c/ui-and-macro/)
- Join the [WoWUIDev Discord](https://discord.gg/wowuidev) for real-time discussion with other addon developers

---

## Stable vs At-Risk API Surfaces

### Stable — Safe to Build On

These APIs are unlikely to see further restrictions in 12.0.x patches:

| API Surface | Notes |
|-------------|-------|
| **SavedVariables** | Account and per-character persistence is unchanged |
| **Slash commands** | Registration and handling work identically to pre-Midnight |
| **Addon messaging** (outside instances) | Guild, whisper, and non-instance party/raid channels are stable |
| **Frame skinning** | Textures, fonts, colors, layouts on non-secure frames |
| **Non-combat events** | `ADDON_LOADED`, `PLAYER_LOGIN`, `BAG_UPDATE`, `MERCHANT_SHOW`, etc. |
| **C_AddOns namespace** | Addon metadata, load-on-demand control |
| **Settings API** | `Settings.RegisterAddOnCategory()` and related functions |
| **Addon Compartment** | Minimap button registration via TOC fields |

### At-Risk — May Change in 12.0.x Patches

These areas have seen iteration during Alpha/Beta and may continue to be adjusted:

| API Surface | Risk Level | Notes |
|-------------|:---:|-------|
| **Combat data access depth** | High | The boundary between displayable and actionable combat data is still being refined |
| **Nameplate customization** | Medium | Extent of addon access to nameplate unit data in combat may tighten |
| **Protected frame manipulation** | Medium | Enforcement is stricter; edge cases around secure frame interaction continue to be patched |
| **Spell whitelist scope** | Low-Medium | Whitelist is expanding but individual spells may be added or removed |
| **In-instance addon messaging** | Medium | Throttling and restriction rules during encounters may be adjusted |

!!! tip "Track changes proactively"
    The [Warcraft Wiki API Change Summaries](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) page is essential for tracking changes across patches. Bookmark it and check before each PTR cycle.

---

## Key Resources

- [Patch 12.0.0/API changes — Warcraft Wiki](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes)
- [Patch 12.0.0/Planned API changes — Warcraft Wiki](https://warcraft.wiki.gg/wiki/Patch_12.0.0/Planned_API_changes)
- [How Midnight's Changes Impact Combat Addons — Blizzard](https://news.blizzard.com/en-us/article/24244638/how-midnights-upcoming-game-changes-will-impact-combat-addons)
- [Combat Philosophy and Addon Disarmament — Blizzard](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight)
- [Midnight UI Updates — Blizzard](https://news.blizzard.com/en-us/article/24223311/midnight-get-up-to-speed-with-user-interface-updates)
- [Whitelisted Spells Detailed — Wowhead](https://www.wowhead.com/news/majority-of-addon-changes-finalized-for-midnight-pre-patch-whitelisted-spells-379738)
- [API Change Summaries — Wowpedia](https://wowpedia.fandom.com/wiki/API_change_summaries)
