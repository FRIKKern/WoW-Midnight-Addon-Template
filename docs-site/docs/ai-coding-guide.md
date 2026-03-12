---
title: The AI-Assisted WoW Addon Developer's Guide
description: Everything AI needs to know to write correct Midnight (12.0+) addon code
---

# The AI-Assisted WoW Addon Developer's Guide

**Everything AI needs to know to write correct Midnight (12.0+) addon code**

---

!!! danger "The Problem"

    AI models **hallucinate WoW API functions**, use deprecated patterns from 2015, and have no idea that Midnight (12.0) rewrote the rules. `GetSpellInfo()` is dead. `COMBAT_LOG_EVENT_UNFILTERED` is gone. The addon paradigm has fundamentally shifted.

!!! success "The Solution"

    This guide + our custom agents. Paste the [System Prompt](#the-system-prompt) into any AI before asking it to write WoW code. Use the [Deprecated Function Map](#the-deprecated-function-map) to catch hallucinations. Follow the [Golden Rules](#the-golden-rules) and your AI-generated code will actually work.

---

## The Golden Rules

These are non-negotiable. Every piece of AI-generated WoW addon code must follow all ten.

!!! abstract "Rule 1 — Interface: 120001"

    Always. No exceptions. This is WoW 12.0.1 Midnight. Your `.toc` file must contain:

    ```toc
    ## Interface: 120001
    ```

    Previous values like `110002`, `100207`, `110100` are **wrong** for Midnight.

!!! abstract "Rule 2 — Lua 5.1"

    WoW's Lua is frozen in 2006. These do **not** exist:

    - No `goto` statements
    - No bitwise operators (`&`, `|`, `~`, `<<`, `>>`)
    - No `_ENV` (use `setfenv`/`getfenv` if you must)
    - No `load()` with environment arg (use `loadstring()`)
    - No integer division `//`

    Use `bit.band()`, `bit.bor()`, `bit.lshift()` for bitwise operations.

!!! abstract "Rule 3 — C_ Namespace Functions"

    `GetSpellInfo()` is **DEAD**. `C_Spell.GetSpellInfo()` is alive. Blizzard has been migrating all global functions into `C_` namespaces since Dragonflight. If the AI writes a bare global function, **check if a C_ version exists**.

    ```lua
    -- WRONG (deprecated/removed)
    local name, _, icon = GetSpellInfo(12345)

    -- CORRECT (returns a table!)
    local info = C_Spell.GetSpellInfo(12345)
    local name, icon = info.name, info.iconID
    ```

!!! abstract "Rule 4 — Namespace Everything"

    Every `.lua` file must start with the addon namespace. No globals except SavedVariables.

    ```lua
    local addonName, ns = ...
    ```

    Store all shared state on `ns`. Never write `MyAddon_Data = {}` in global scope.

!!! abstract "Rule 5 — Events, Not Polling"

    Use the table dispatch pattern. Register for events. **Never** use `OnUpdate` for what an event can do.

    ```lua
    local frame = CreateFrame("Frame")
    local events = {}

    function events:PLAYER_LOGIN()
        print("Logged in!")
    end

    function events:ADDON_LOADED(loadedAddon)
        if loadedAddon == addonName then
            -- init SavedVariables here
        end
    end

    frame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then
            events[event](self, ...)
        end
    end)

    for event in pairs(events) do
        frame:RegisterEvent(event)
    end
    ```

!!! abstract "Rule 6 — Combat Lockdown"

    Always check `InCombatLockdown()` before modifying protected frames. Queue the operation and execute on `PLAYER_REGEN_ENABLED`.

    ```lua
    local pendingActions = {}

    local function SafeFrameOp(action)
        if InCombatLockdown() then
            table.insert(pendingActions, action)
            return
        end
        action()
    end

    function events:PLAYER_REGEN_ENABLED()
        for _, action in ipairs(pendingActions) do
            action()
        end
        wipe(pendingActions)
    end
    ```

!!! abstract "Rule 7 — Skin, Don't Replace"

    Midnight's #1 design rule. **Enhance** Blizzard's frames, don't create custom replacements for standard UI elements. Hook into existing frames with `hooksecurefunc()`.

    ```lua
    -- WRONG: Replacing Blizzard's tooltip
    local myTooltip = CreateFrame("GameTooltip", "MyCustomTooltip", UIParent)

    -- CORRECT: Enhancing Blizzard's tooltip
    GameTooltip:HookScript("OnTooltipSetItem", function(self)
        local _, link = self:GetItem()
        if link then
            self:AddLine("My custom info here", 0.5, 0.8, 1.0)
        end
    end)
    ```

!!! abstract "Rule 8 — No CLEU"

    `COMBAT_LOG_EVENT_UNFILTERED` is **gone** for addons in Midnight. Use specific unit events instead:

    | Instead of CLEU for... | Use this event |
    |------------------------|----------------|
    | Tracking damage/healing | `UNIT_HEALTH`, post-combat data |
    | Tracking buffs/debuffs | `UNIT_AURA` + `C_UnitAuras` |
    | Tracking casts | `UNIT_SPELLCAST_START`, `_SUCCEEDED`, `_FAILED` |
    | Tracking deaths | `UNIT_HEALTH` reaching 0, `PLAYER_DEAD` |
    | Combat summary | `ENCOUNTER_END`, `PLAYER_REGEN_ENABLED` |

!!! abstract "Rule 9 — Verify Every Function"

    Before using **any** API function, verify it exists at:

    `https://warcraft.wiki.gg/wiki/API_FunctionName`

    If the wiki page says "Removed in patch 12.0" — it's dead. Find the replacement.

!!! abstract "Rule 10 — Complete, Not Snippets"

    Always generate full `.toc` + `.lua` files, never code fragments. A snippet that works in isolation will fail when loaded as an addon because it's missing the namespace, event registration, or SavedVariables initialization.

---

## The Quick Start

A minimal but **correct** Midnight addon template with every line explained:

=== "MyAddon.toc"

    ```toc
    ## Interface: 120001
    ## Title: My Addon
    ## Notes: A correctly structured Midnight addon
    ## Author: YourName
    ## Version: 1.0.0
    ## SavedVariables: MyAddonDB
    ## IconTexture: Interface\Icons\INV_Misc_QuestionMark

    # Load order matters — list files in dependency order
    MyAddon.lua
    ```

=== "MyAddon.lua"

    ```lua
    -- Every file starts with the addon namespace
    -- addonName = "MyAddon" (string), ns = shared namespace table
    local addonName, ns = ...

    -- Default settings (merged with SavedVariables on load)
    ns.defaults = {
        enabled = true,
        scale = 1.0,
        welcomeShown = false,
    }

    -- Event handler table — dispatch pattern, no if/elseif chains
    local events = {}

    -- SavedVariables are ONLY safe to read after ADDON_LOADED fires
    function events:ADDON_LOADED(loadedAddon)
        if loadedAddon ~= addonName then return end

        -- Initialize SavedVariables with defaults
        if not MyAddonDB then
            MyAddonDB = {}
        end
        for key, value in pairs(ns.defaults) do
            if MyAddonDB[key] == nil then
                MyAddonDB[key] = value
            end
        end
        ns.db = MyAddonDB

        -- Unregister — this event only matters once for us
        self:UnregisterEvent("ADDON_LOADED")
    end

    function events:PLAYER_LOGIN()
        if ns.db and not ns.db.welcomeShown then
            print("|cff00ccff[MyAddon]|r Welcome to Midnight! Type /myaddon for options.")
            ns.db.welcomeShown = true
        end
    end

    -- Create the event frame and wire up dispatch
    local frame = CreateFrame("Frame")
    frame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then
            events[event](self, ...)
        end
    end)
    for event in pairs(events) do
        frame:RegisterEvent(event)
    end

    -- Slash command — always provide one for user interaction
    SLASH_MYADDON1 = "/myaddon"
    SlashCmdList["MYADDON"] = function(msg)
        local cmd = msg:lower():trim()
        if cmd == "reset" then
            MyAddonDB = nil
            ReloadUI()
        else
            print("|cff00ccff[MyAddon]|r Options:")
            print("  /myaddon reset — Reset all settings")
        end
    end
    ```

---

## The Deprecated Function Map

!!! warning "AI models hallucinate these constantly"

    The left column is what AI will write. The right column is what actually works in Midnight.

| Deprecated / Removed | Correct Replacement (12.0+) | Notes |
|---|---|---|
| `GetSpellInfo(id)` | `C_Spell.GetSpellInfo(id)` | Returns a **table** (`info.name`, `info.iconID`), not multiple values |
| `GetSpellDescription(id)` | `C_Spell.GetSpellDescription(id)` | Returns string |
| `GetSpellTexture(id)` | `C_Spell.GetSpellTexture(id)` | Returns textureID |
| `GetSpellCooldown(id)` | `C_Spell.GetSpellCooldown(id)` | Returns a **table** |
| `GetSpellCharges(id)` | `C_Spell.GetSpellCharges(id)` | Returns a **table** |
| `IsSpellKnown(id)` | `C_Spell.IsSpellDataCached(id)` | Behavior changed |
| `GetItemInfo(id)` | `C_Item.GetItemInfo(id)` | Async — may return nil, use callback |
| `GetItemIcon(id)` | `C_Item.GetItemIconByID(id)` | — |
| `GetContainerNumSlots(bag)` | `C_Container.GetContainerNumSlots(bag)` | — |
| `GetContainerItemInfo(bag, slot)` | `C_Container.GetContainerItemInfo(bag, slot)` | Returns a **table** |
| `GetContainerItemLink(bag, slot)` | `C_Container.GetContainerItemLink(bag, slot)` | — |
| `PickupContainerItem(bag, slot)` | `C_Container.PickupContainerItem(bag, slot)` | — |
| `UseContainerItem(bag, slot)` | `C_Container.UseContainerItem(bag, slot)` | — |
| `GetAddOnInfo(index)` | `C_AddOns.GetAddOnInfo(index)` | — |
| `GetNumAddOns()` | `C_AddOns.GetNumAddOns()` | — |
| `IsAddOnLoaded(name)` | `C_AddOns.IsAddOnLoaded(name)` | — |
| `GetSpecialization()` | `PlayerUtil.GetCurrentSpecID()` | Entirely different pattern |
| `GetSpecializationInfo(id)` | `C_SpecializationInfo.GetSpecializationInfo(id)` | — |
| `GetAchievementInfo(id)` | `C_AchievementInfo.GetAchievementInfo(id)` | — |
| `GetCurrencyInfo(id)` | `C_CurrencyInfo.GetCurrencyInfo(id)` | Returns a **table** |
| `UnitAura(unit, index)` | `C_UnitAuras.GetAuraDataByIndex(unit, index)` | Returns **AuraData** table |
| `UnitBuff(unit, index)` | `C_UnitAuras.GetBuffDataByIndex(unit, index)` | Returns **AuraData** table |
| `UnitDebuff(unit, index)` | `C_UnitAuras.GetDebuffDataByIndex(unit, index)` | Returns **AuraData** table |
| `CombatLogGetCurrentEventInfo()` | **REMOVED — no replacement** | CLEU is gone for addons in 12.0 |
| `GetTalentInfo(tier, col, group)` | `C_ClassTalents` + talent tree APIs | Completely reworked system |
| `SendChatMessage()` in instances | **Restricted** | Chat in instances is now opaque via Secret Values |

!!! tip "Pattern to remember"

    If a C_ version returns a **table** instead of multiple values, you must destructure it:

    ```lua
    -- OLD pattern (multiple return values)
    local name, rank, icon, castTime = GetSpellInfo(12345)

    -- NEW pattern (table return)
    local info = C_Spell.GetSpellInfo(12345)
    if info then
        local name = info.name
        local icon = info.iconID
        local castTime = info.castTime
    end
    ```

---

## The Midnight Paradigm Shift

!!! danger "12.0 changed everything"

    Midnight (Patch 12.0) made the largest addon API changes in WoW's 20-year history. Blizzard calls it "Addon Disarmament." The community calls it the "Addon Apocalypse."

**Secret Values** — Combat numbers (damage, healing, absorbs) are now opaque `SecretValue` objects. Addons can display them via Blizzard's UI, but cannot read, compare, or store the raw numbers. DPS meters now work fundamentally differently.

**CLEU Removed** — `COMBAT_LOG_EVENT_UNFILTERED`, the backbone of every combat addon for 15 years, no longer fires for addon code. Use specific unit events (`UNIT_HEALTH`, `UNIT_AURA`, `UNIT_SPELLCAST_*`) instead.

**Skin, Don't Replace** — Blizzard explicitly encourages addons to enhance existing UI rather than replacing it. Custom action bars, unit frames, and nameplates must hook into Blizzard's secure frames.

**Instance Communication Restricted** — Addon channel messages in instances are limited. Chat messages become Secret Values in dungeons and raids.

**The Official Statement:** [Combat Philosophy and Addon Disarmament in Midnight](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight)

---

## Guide Map

<div class="grid cards" markdown>

-   :material-file-document-outline:{ .lg .middle } **API Cheat Sheet**

    ---

    Correct function signatures for every commonly-used API call, verified against 12.0.1.

    [:octicons-arrow-right-24: API Cheat Sheet](api-cheatsheet.md)

-   :material-alert-outline:{ .lg .middle } **Common Pitfalls**

    ---

    What AI gets wrong and how to fix it — the 20 most common hallucinations.

    [:octicons-arrow-right-24: Common Pitfalls](pitfalls.md)

-   :material-puzzle-outline:{ .lg .middle } **Code Templates**

    ---

    Copy-paste working addons for every common pattern: options panel, minimap button, data broker, etc.

    [:octicons-arrow-right-24: Code Templates](code-templates.md)

-   :material-moon-waning-crescent:{ .lg .middle } **Midnight Patterns**

    ---

    The new coding paradigm — Secret Values, unit events, secure frame hooks.

    [:octicons-arrow-right-24: Midnight Patterns](midnight-patterns.md)

-   :material-robot-outline:{ .lg .middle } **AI Prompt Templates**

    ---

    Prompts that produce correct code — tested with Claude, GPT-4, and Gemini.

    [:octicons-arrow-right-24: AI Prompt Templates](ai-prompts.md)

-   :material-home-outline:{ .lg .middle } **Home**

    ---

    Back to the documentation homepage with quick start and full resource list.

    [:octicons-arrow-right-24: Home](index.md)

</div>

---

## The System Prompt

!!! tip "Copy this and paste it into ANY AI before asking it to write WoW addon code"

Copy the entire block below. It contains all constraints, correct function names, and Midnight rules.

```text
You are a World of Warcraft addon developer expert for Patch 12.0+ (Midnight).
Interface version: 120001. WoW uses Lua 5.1 (no goto, no bitwise operators,
no _ENV, no integer division //, no load() with env arg — use loadstring()).
Use bit.band(), bit.bor(), bit.lshift() for bitwise operations.

CRITICAL RULES:
1. Every .lua file starts with: local addonName, ns = ...
2. No global variables except SavedVariables declared in .toc
3. Use event-driven table dispatch pattern, never if/elseif chains for events
4. SavedVariables are ONLY safe after ADDON_LOADED fires for your addon
5. Always check InCombatLockdown() before modifying protected/secure frames
6. Queue combat-blocked operations for PLAYER_REGEN_ENABLED
7. Never use OnUpdate for what an event can do
8. Always generate complete .toc + .lua files, not snippets

MIDNIGHT (12.0) CHANGES — THESE ARE MANDATORY:
- COMBAT_LOG_EVENT_UNFILTERED does NOT fire for addons. It is removed.
- CombatLogGetCurrentEventInfo() is REMOVED with no replacement.
- Use UNIT_HEALTH, UNIT_AURA, UNIT_SPELLCAST_START/SUCCEEDED/FAILED instead.
- Combat numbers are Secret Values — opaque, cannot be read/compared/stored.
- Addon chat in instances is restricted. Messages become Secret Values.
- Skin and enhance Blizzard frames, do not replace them.

DEPRECATED FUNCTIONS — DO NOT USE THESE:
- GetSpellInfo() → C_Spell.GetSpellInfo() (returns TABLE: .name, .iconID, .castTime)
- GetSpellCooldown() → C_Spell.GetSpellCooldown() (returns TABLE)
- GetSpellCharges() → C_Spell.GetSpellCharges() (returns TABLE)
- GetSpellTexture() → C_Spell.GetSpellTexture()
- GetSpellDescription() → C_Spell.GetSpellDescription()
- GetItemInfo() → C_Item.GetItemInfo() (async, may return nil)
- GetItemIcon() → C_Item.GetItemIconByID()
- GetContainerNumSlots() → C_Container.GetContainerNumSlots()
- GetContainerItemInfo() → C_Container.GetContainerItemInfo() (returns TABLE)
- GetContainerItemLink() → C_Container.GetContainerItemLink()
- PickupContainerItem() → C_Container.PickupContainerItem()
- UseContainerItem() → C_Container.UseContainerItem()
- GetAddOnInfo() → C_AddOns.GetAddOnInfo()
- GetNumAddOns() → C_AddOns.GetNumAddOns()
- IsAddOnLoaded() → C_AddOns.IsAddOnLoaded()
- GetSpecialization() → PlayerUtil.GetCurrentSpecID()
- GetAchievementInfo() → C_AchievementInfo.GetAchievementInfo()
- GetCurrencyInfo() → C_CurrencyInfo.GetCurrencyInfo() (returns TABLE)
- UnitAura() → C_UnitAuras.GetAuraDataByIndex() (returns AuraData TABLE)
- UnitBuff() → C_UnitAuras.GetBuffDataByIndex() (returns AuraData TABLE)
- UnitDebuff() → C_UnitAuras.GetDebuffDataByIndex() (returns AuraData TABLE)

IMPORTANT: When C_ functions return a table, you must access fields:
  local info = C_Spell.GetSpellInfo(id)
  if info then print(info.name, info.iconID) end
Do NOT try: local name, _, icon = C_Spell.GetSpellInfo(id) -- THIS IS WRONG

UNAVAILABLE IN WOW LUA:
require(), dofile(), loadfile(), os.*, io.*, debug.* (mostly),
package.*, coroutine.* (limited), string.dump()

CORRECT .TOC FORMAT:
## Interface: 120001
## Title: AddonName
## Notes: Description
## Author: AuthorName
## Version: 1.0.0
## SavedVariables: AddonNameDB
## IconTexture: Interface\Icons\INV_Misc_QuestionMark
Core.lua
Modules/Module1.lua

VERIFY API CALLS: Before using any function, mentally verify it exists in
Patch 12.0.1. If unsure, note it needs verification against
warcraft.wiki.gg/wiki/API_FunctionName

COMMON PATTERNS:
- Slash commands: SLASH_NAME1 = "/cmd"; SlashCmdList["NAME"] = function(msg) end
- Timers: C_Timer.After(seconds, callback) — cannot be cancelled
- Cancellable timers: C_Timer.NewTimer(seconds, callback) — returns handle with :Cancel()
- Repeating timers: C_Timer.NewTicker(seconds, callback, iterations)
- Frame pools: CreateFramePool("Frame", parent, template)
- Secure hooks: hooksecurefunc("FunctionName", postHookFunc)
- Secure hooks on objects: hooksecurefunc(object, "MethodName", postHookFunc)
```

---

## Using Our Custom Agents

We provide three specialized agents that understand Midnight's API:

<div class="grid cards" markdown>

-   :material-magnify:{ .lg .middle } **wow-addon-researcher**

    ---

    Searches real documentation sources — warcraft.wiki.gg, Gethe/wow-ui-source, wago.tools — and returns verified API information. Never hallucinates.

    **Use when:** You need to verify an API function exists, find correct signatures, or understand how Blizzard's own UI uses an API.

-   :material-code-tags:{ .lg .middle } **wow-addon-coder**

    ---

    Writes correct addon code with all Midnight specs baked in. Automatically uses C_ namespaces, table dispatch, proper SavedVariables, and combat lockdown checks.

    **Use when:** You need to generate addon code, create a new addon from scratch, or convert deprecated code to 12.0+ patterns.

-   :material-newspaper-variant-outline:{ .lg .middle } **wow-addon-news-desk**

    ---

    Tracks the latest addon API changes, hotfixes, and community discoveries. Monitors Blizzard blue posts, wowhead news, and wiki changes.

    **Use when:** You need to check if an API has changed in a recent hotfix, or want the latest community findings on Secret Values behavior.

</div>

---

## Verification Checklist

!!! example "Use this checklist to review every piece of AI-generated WoW addon code"

    Print this out. Tape it to your monitor. Check every box before running `/reload`.

    **Structure**

    - [ ] `.toc` file has `## Interface: 120001`
    - [ ] `.toc` file has `## IconTexture:` set
    - [ ] Every `.lua` file starts with `local addonName, ns = ...`
    - [ ] All `.lua` files are listed in the `.toc` in correct load order
    - [ ] SavedVariables declared in `.toc` match variable names in code

    **API Correctness**

    - [ ] No deprecated global functions (check against [Deprecated Function Map](#the-deprecated-function-map))
    - [ ] C\_ function return values handled as **tables**, not multiple returns
    - [ ] Async API calls (like `C_Item.GetItemInfo`) handle `nil` returns
    - [ ] `C_Timer.After` is not assigned to a variable for cancellation (use `C_Timer.NewTimer` instead)
    - [ ] No `COMBAT_LOG_EVENT_UNFILTERED` usage anywhere
    - [ ] Every API function verified at `warcraft.wiki.gg/wiki/API_FunctionName`

    **Patterns**

    - [ ] Events use table dispatch pattern, not `if/elseif` chains
    - [ ] SavedVariables only accessed after `ADDON_LOADED` fires
    - [ ] `InCombatLockdown()` checked before any protected frame operations
    - [ ] Combat-blocked operations queued for `PLAYER_REGEN_ENABLED`
    - [ ] No `OnUpdate` scripts where an event would suffice
    - [ ] Hooks use `hooksecurefunc()`, not raw replacement

    **Lua 5.1 Compliance**

    - [ ] No `goto` statements
    - [ ] No bitwise operators (`&`, `|`, `~`, `<<`, `>>`) — use `bit.*` library
    - [ ] No `_ENV` — use `setfenv`/`getfenv` if needed
    - [ ] No `require()`, `dofile()`, `os.*`, `io.*`, `debug.*`
    - [ ] No integer division `//` — use `math.floor(a / b)`
    - [ ] No `load()` with env arg — use `loadstring()`

    **Namespace Hygiene**

    - [ ] No global variable pollution (only SavedVariables)
    - [ ] All module state stored on `ns` table
    - [ ] Slash command names are unique (`SLASH_MYADDON1`, not `SLASH_CMD1`)

---

## Essential Reference Links

| What | Where |
|------|-------|
| Verify any API function | [warcraft.wiki.gg/wiki/API_*](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) |
| Browse all events | [warcraft.wiki.gg/wiki/Events](https://warcraft.wiki.gg/wiki/Events) |
| See what changed in 12.0 | [Patch 12.0.0 API changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) |
| See what changed in 12.0.1 | [Patch 12.0.1 API changes](https://warcraft.wiki.gg/wiki/Patch_12.0.1/API_changes) |
| Read Blizzard's own UI code | [Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source) |
| Browse all C\_ namespaces | [Category:API namespaces](https://warcraft.wiki.gg/wiki/Category:API_namespaces) |
| Blizzard's Midnight addon philosophy | [Combat Philosophy and Addon Disarmament](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight) |
