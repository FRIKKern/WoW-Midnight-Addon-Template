---
title: "Build-Along: RepairBuddy — Smart Auto-Repair & Vendor Companion"
description: "A step-by-step tutorial for building a performance-optimized vendor companion addon using Performance Zealot mode. Covers auto-repair, auto-sell junk, durability warnings, and the /perf diagnostic — all with zero unnecessary memory allocations."
---

# Build-Along: RepairBuddy — Smart Auto-Repair & Vendor Companion

**Difficulty:** Intermediate | **Mode:** :material-speedometer: Performance Zealot | **Reading time:** ~25 minutes

---

## Step 1: What We're Building

RepairBuddy is a vendor companion addon that does four things:

1. **Auto-repairs** your gear when you open a vendor, using guild funds if available
2. **Auto-sells junk** (grey quality items) and reports the gold earned
3. **Tracks repair costs** over time in SavedVariables so you can see how much you spend on repairs per session, per day, or lifetime
4. **Warns about low durability** when any equipped item drops below a configurable threshold — before you get to a vendor

These are common features, but we are building them under **Performance Zealot** constraints. That means every line of code is written to minimize memory allocations, avoid garbage collection pressure, and stay purely event-driven. No OnUpdate polling. No table creation in hot paths. No uncached global lookups.

### Why Performance Zealot Mode?

RepairBuddy is a good candidate for performance optimization because:

- **Bag scanning touches every slot** — iterating 5 bags with up to 36 slots each means 180 potential iterations. If any of those iterations create garbage (temporary tables, intermediate strings), you pay for it on every vendor visit.
- **Durability checks run on equipment change events** — these fire frequently during gameplay (swapping gear, items breaking). Sloppy code here adds up.
- **The addon is always loaded** — even when you are not at a vendor, the durability warning system is active. Always-on addons must be lean.

### What You Will Learn

- Caching globals as locals for faster function lookup
- Pre-building format strings to avoid runtime string concatenation
- Table reuse with `wipe()` instead of creating new tables
- Purely event-driven architecture — no OnUpdate, no timers for polling
- The `/perf` diagnostic command that Performance Zealot mode requires in every addon
- Efficient SavedVariables initialization with defaults merging

---

## Step 2: Prerequisites and Mode Setup

Before starting, make sure you have a basic understanding of WoW addon structure (TOC files, Lua namespaces, events). If you are new to addon development, read the [Getting Started](../getting-started.md) guide first.

### Set the Development Mode

If you are using the better-addons plugin with Claude Code, set Performance Zealot mode:

```
/wow-mode zealot
```

This tells all AI agents to enforce performance constraints: every generated file will include local caching blocks, throttled handlers, table reuse patterns, and the `/perf` diagnostic. The Reviewer agent will flag unthrottled OnUpdate, uncached globals, and table creation in hot paths as **critical** issues.

### What Performance Zealot Means

The Performance Zealot philosophy is simple: **your addon should be invisible to the game client.** If a player opens `/eventtrace` or checks addon memory usage, RepairBuddy should be nearly undetectable. Every allocation is a future garbage collection pause. Every uncached global lookup is a hash table traversal that did not need to happen. Every unthrottled handler is CPU time stolen from rendering.

We are not optimizing prematurely — we are building with the right patterns from the start, so optimization is never needed later.

---

## Step 3: Project Setup

Create the addon directory structure. RepairBuddy uses four files:

```
Interface/AddOns/RepairBuddy/
  RepairBuddy.toc     -- Addon metadata and load order
  Data.lua            -- Constants, defaults, pre-built format strings
  Core.lua            -- Event handling, auto-repair, auto-sell, durability
  Config.lua          -- Settings API panel
```

### The TOC File

```toc title="RepairBuddy.toc"
## Interface: 120001
## Title: RepairBuddy
## Notes: Performance-optimized auto-repair, junk selling, and durability tracking
## Author: YourName
## Version: @project-version@
## Category: Inventory
## IconTexture: Interface\Icons\Trade_BlackSmithing
## SavedVariables: RepairBuddyDB
## AddonCompartmentFunc: RepairBuddy_OnAddonCompartmentClick
## AddonCompartmentFuncOnEnter: RepairBuddy_OnAddonCompartmentEnter
## AddonCompartmentFuncOnLeave: RepairBuddy_OnAddonCompartmentLeave

Data.lua
Core.lua
Config.lua
```

!!! tip "Load Order Matters"
    `Data.lua` loads first because it defines constants, defaults, and format strings that `Core.lua` and `Config.lua` depend on. In Performance Zealot mode, we centralize all constants and pre-built strings in a single file so they are allocated once at load time.

### File Responsibilities

| File | Purpose | Hot Path? |
|------|---------|-----------|
| `Data.lua` | Constants, defaults, format strings | No — runs once at load time |
| `Core.lua` | Event handlers, repair/sell/durability logic | Yes — runs on merchant open and equipment change |
| `Config.lua` | Settings API panel, slash commands, `/perf` | No — runs only when user opens settings |

This separation keeps hot-path code in `Core.lua` clean and focused. Configuration and constants never pollute the event handler file.

---

## Step 4: Performance Zealot Foundations

Before writing any feature code, we establish the performance foundations that every file in a Zealot-mode addon must have.

### Data.lua — Constants and Format Strings

```lua title="Data.lua"
-- Mode: Performance Zealot | Zero-waste addon
local addonName, ns = ...

-- PERF: Pre-built format strings — never concatenate at runtime
ns.FMT = {
    REPAIR_COST   = "|cff00ff00RepairBuddy|r: Repaired for %s",
    REPAIR_GUILD  = "|cff00ff00RepairBuddy|r: Repaired for %s (guild funds)",
    REPAIR_BROKE  = "|cff00ff00RepairBuddy|r: Not enough gold to repair (%s needed)",
    JUNK_SOLD     = "|cff00ff00RepairBuddy|r: Sold %d junk item(s) for %s",
    JUNK_NONE     = "|cff00ff00RepairBuddy|r: No junk to sell",
    DURABILITY    = "|cffff6600RepairBuddy|r: Warning — %s is at %d%% durability",
    PERF_REPORT   = "|cff00ff00RepairBuddy|r: %.1f KB | %d events handled",
}

-- Defaults for SavedVariables
ns.defaults = {
    autoRepair = true,
    useGuildFunds = true,
    autoSellJunk = true,
    durabilityWarning = true,
    durabilityThreshold = 20,
    totalRepairCost = 0,
    sessionRepairCost = 0,
    totalJunkValue = 0,
    repairCount = 0,
    eventCount = 0,
}

-- PERF: Equipment slot IDs — static, never changes
ns.EQUIPMENT_SLOTS = {
    1, 2, 3, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 19
}
ns.NUM_EQUIPMENT_SLOTS = #ns.EQUIPMENT_SLOTS

-- PERF: Bag IDs — static array
ns.BAG_IDS = { 0, 1, 2, 3, 4 }
ns.NUM_BAGS = #ns.BAG_IDS
```

!!! success "Performance Win: Pre-built Format Strings"
    Every `format()` call in RepairBuddy uses a pre-built string from `ns.FMT`. Compare:

    === "Slow: Runtime Concatenation"
        ```lua
        -- BAD: creates 4 intermediate strings every call
        print("|cff00ff00RepairBuddy|r: Repaired for " .. GetCoinTextureString(cost))
        ```

    === "Fast: Pre-built Format"
        ```lua
        -- GOOD: one format() call, no intermediate strings
        print(format(ns.FMT.REPAIR_COST, GetCoinTextureString(cost)))
        ```

    The slow version creates a new string for each `..` concatenation — four string objects that become garbage immediately. The fast version creates one string via `format()` using a pre-existing template.

### Local Caching Block

Every `.lua` file in a Performance Zealot addon starts with a **local caching block** immediately after the namespace line. This caches every WoW API function and Lua global used in the file as a local upvalue.

Why? Lua resolves global variables by walking up the environment chain and performing a hash table lookup. Local variables are resolved by index — a direct array access. In code that runs on every event or every frame, this difference is measurable.

```lua title="Core.lua — caching block (top of file)"
-- Mode: Performance Zealot | Zero-waste addon
local addonName, ns = ...

-- PERF: Cached globals — local lookup is O(1), global is hash lookup
local pairs, ipairs, next = pairs, ipairs, next
local format = string.format
local wipe = wipe
local print = print

-- PERF: Cached WoW API functions
local GetRepairAllCost = GetRepairAllCost
local RepairAllItems = RepairAllItems
local CanMerchantRepair = CanMerchantRepair
local GetMoney = GetMoney
local GetCoinTextureString = GetCoinTextureString
local GetInventoryItemDurability = GetInventoryItemDurability
local GetInventoryItemLink = GetInventoryItemLink
local InCombatLockdown = InCombatLockdown
local IsInGuild = IsInGuild
local C_Container = C_Container
local UpdateAddOnMemoryUsage = UpdateAddOnMemoryUsage
local GetAddOnMemoryUsage = GetAddOnMemoryUsage
```

!!! danger "Performance Anti-Pattern: Uncached Globals in Loops"
    ```lua
    -- BAD: GetContainerItemInfo is resolved via hash lookup 180 times
    for bag = 0, 4 do
        for slot = 1, C_Container.GetContainerNumSlots(bag) do
            local info = C_Container.GetContainerItemInfo(bag, slot)
        end
    end
    ```
    With the caching block above, `C_Container` is already a local — so `C_Container.GetContainerItemInfo` is a local table access followed by a direct function call. Without it, every iteration walks the global environment chain.

### Reusable Tables

Performance Zealot mode forbids creating tables inside functions that run more than once. Instead, allocate tables at file scope and reuse them with `wipe()`:

```lua
-- PERF: Pre-allocated scan results table — reused on every merchant visit
local scanResults = {}

-- PERF: Pre-allocated durability results — reused on every equipment change
local durabilityResults = {}
```

These tables are allocated once when the file loads and never again. Every time we need to scan bags or check durability, we `wipe()` them and refill.

---

## Step 5: Core Logic — Auto-Repair

The auto-repair system activates on the `MERCHANT_SHOW` event — fired every time the player opens a vendor window. It checks whether the vendor offers repairs, calculates the cost, and repairs if the player can afford it.

### Event Architecture

```lua title="Core.lua — event dispatch"
local events = {}
local db  -- PERF: local reference to SavedVariables, set in ADDON_LOADED

function events:ADDON_LOADED(loadedAddon)
    if loadedAddon ~= addonName then return end

    -- Initialize SavedVariables with defaults
    if not RepairBuddyDB then
        RepairBuddyDB = {}
    end
    for k, v in pairs(ns.defaults) do
        if RepairBuddyDB[k] == nil then
            RepairBuddyDB[k] = v
        end
    end

    -- PERF: Local reference avoids global lookup on every access
    db = RepairBuddyDB
    ns.db = db

    -- Reset session counter
    db.sessionRepairCost = 0

    self:UnregisterEvent("ADDON_LOADED")
end
```

!!! tip "Local Reference to SavedVariables"
    Notice `db = RepairBuddyDB` — we store a local reference to the SavedVariables table. Every subsequent access uses `db.autoRepair` instead of `RepairBuddyDB.autoRepair`, avoiding a global table lookup each time.

### The Repair Handler

```lua title="Core.lua — auto-repair"
function events:MERCHANT_SHOW()
    db.eventCount = db.eventCount + 1

    -- Auto-repair
    if db.autoRepair and CanMerchantRepair() then
        local cost, canRepair = GetRepairAllCost()

        if canRepair and cost > 0 then
            local playerMoney = GetMoney()

            -- Try guild funds first if enabled and in a guild
            if db.useGuildFunds and IsInGuild() then
                RepairAllItems(true)  -- true = use guild bank
                print(format(ns.FMT.REPAIR_GUILD, GetCoinTextureString(cost)))
            elseif playerMoney >= cost then
                RepairAllItems(false)
                print(format(ns.FMT.REPAIR_COST, GetCoinTextureString(cost)))
            else
                print(format(ns.FMT.REPAIR_BROKE, GetCoinTextureString(cost)))
                return  -- Can't afford repair, skip cost tracking
            end

            -- Track repair costs
            db.totalRepairCost = db.totalRepairCost + cost
            db.sessionRepairCost = db.sessionRepairCost + cost
            db.repairCount = db.repairCount + 1
        end
    end

    -- Auto-sell junk (runs after repair)
    if db.autoSellJunk then
        SellJunk()
    end
end
```

!!! success "Performance Win: Event-Driven, Not Polled"
    The repair logic runs **only** when `MERCHANT_SHOW` fires — once per vendor visit. There is no timer checking "are we at a vendor?" every few seconds. Event-driven architecture means zero CPU cost when the feature is not active.

---

## Step 6: Auto-Sell Junk — The Performance Challenge

Selling junk items requires scanning every bag slot. With 5 bags of up to 36 slots each, that is up to 180 slots to check. This is the most performance-sensitive part of RepairBuddy because a naive implementation creates garbage on every iteration.

### The Naive Approach (Do Not Use)

!!! danger "Performance Anti-Pattern: Table Creation in Scan Loop"
    ```lua
    -- BAD: GetContainerItemInfo returns a NEW table every call
    -- BAD: String concatenation in the loop body
    local function SellJunkBad()
        local totalValue = 0
        local count = 0
        for bag = 0, 4 do
            for slot = 1, C_Container.GetContainerNumSlots(bag) do
                local info = C_Container.GetContainerItemInfo(bag, slot)
                if info and info.quality == Enum.ItemQuality.Poor then
                    C_Container.UseContainerItem(bag, slot)
                    totalValue = totalValue + (info.stackCount * info.itemID)
                    count = count + 1
                    -- BAD: string concat in loop
                    print("Sold: " .. info.hyperlink)
                end
            end
        end
    end
    ```
    This creates a new `info` table on every non-empty slot — potentially 100+ table allocations per vendor visit. Each one becomes garbage immediately after the `if` check.

### The Performance Zealot Approach

```lua title="Core.lua — junk selling"
-- PERF: Reusable variables — no allocations in the scan loop
local junkCount = 0
local junkValue = 0

local function SellJunk()
    junkCount = 0
    junkValue = 0

    -- PERF: Use cached C_Container, iterate with numeric for
    for i = 1, ns.NUM_BAGS do
        local bagID = ns.BAG_IDS[i]
        local numSlots = C_Container.GetContainerNumSlots(bagID)

        for slot = 1, numSlots do
            local info = C_Container.GetContainerItemInfo(bagID, slot)

            -- PERF: Early return chain — skip as fast as possible
            if info and info.quality == Enum.ItemQuality.Poor and not info.hasNoValue then
                C_Container.UseContainerItem(bagID, slot)
                junkValue = junkValue + (info.sellPrice * info.stackCount)
                junkCount = junkCount + 1
            end
        end
    end

    if junkCount > 0 then
        print(format(ns.FMT.JUNK_SOLD, junkCount, GetCoinTextureString(junkValue)))
        db.totalJunkValue = db.totalJunkValue + junkValue
    end
end
```

!!! success "Performance Win: Minimize Work Per Iteration"
    The key optimizations:

    1. **Counter variables are file-scope locals** — `junkCount` and `junkValue` are reset to 0 at the start, not allocated as new variables. No table wrapping, no closure capture.
    2. **Early return chain** — the `if` condition checks `info` existence first (cheapest check), then quality (integer comparison), then `hasNoValue`. Most slots either are empty or are not grey quality, so we skip them in the first or second check.
    3. **Single print at the end** — we do not print per item. One `format()` call with the pre-built template produces one output string.
    4. **Numeric `for` over bag IDs** — using `ns.BAG_IDS[i]` with a numeric loop avoids the overhead of `pairs()` hash traversal on a sequential table.

### Understanding GetContainerItemInfo

`C_Container.GetContainerItemInfo(bagID, slot)` returns a `ContainerItemInfo` table or `nil` for empty slots. The table contains:

| Field | Type | Description |
|-------|------|-------------|
| `iconFileID` | number | Item icon texture ID |
| `stackCount` | number | Number of items in the stack |
| `isLocked` | boolean | Is the item locked (e.g., during trade) |
| `quality` | number | Item quality (0=Poor/grey, 1=Common, ...) |
| `isReadable` | boolean | Can the item be read (books, letters) |
| `hasLoot` | boolean | Does the item contain loot (lockboxes) |
| `hyperlink` | string | Item link for chat/tooltips |
| `hasNoValue` | boolean | Item cannot be sold to a vendor |
| `itemID` | number | The item's ID |
| `sellPrice` | number | Vendor sell price per unit |

!!! tip "Sell Price Calculation"
    The `sellPrice` field is the price **per single item**, not per stack. Multiply by `stackCount` to get the total value: `info.sellPrice * info.stackCount`.

Note that `C_Container.GetContainerItemInfo()` returns a new table on every call — this is a Blizzard API behavior we cannot avoid. The optimization focus is on everything else around it: no additional allocations, no string work, and fast early returns to minimize how many of these tables we actually inspect.

---

## Step 7: Durability Warning System

The durability system warns the player when any equipped item drops below a configurable threshold (default: 20%). It runs on equipment change events — never on a timer.

### When to Scan

Two events trigger a durability scan:

- `PLAYER_EQUIPMENT_CHANGED` — fires when any equipment slot changes (gear swap, item breaks further)
- `PLAYER_ENTERING_WORLD` — fires on login, reload, and zone transitions. We scan here to catch durability changes that happened while logged out.

We do **not** use `UPDATE_INVENTORY_DURABILITY` here because it fires during combat (when items take durability damage from deaths or hits). Printing warnings during an encounter is disruptive. Instead, we scan on equipment change and world entry — moments when the player can actually act on the information.

### The Durability Scanner

```lua title="Core.lua — durability warning"
-- PERF: Pre-allocated results table, reused every scan
local durSlot = 0
local durCurrent = 0
local durMax = 0

local function CheckDurability()
    if not db.durabilityWarning then return end

    local threshold = db.durabilityThreshold

    -- PERF: Numeric for over static slot array — no pairs(), no allocation
    for i = 1, ns.NUM_EQUIPMENT_SLOTS do
        durSlot = ns.EQUIPMENT_SLOTS[i]
        durCurrent, durMax = GetInventoryItemDurability(durSlot)

        if durCurrent and durMax and durMax > 0 then
            local pct = (durCurrent / durMax) * 100

            if pct <= threshold and pct > 0 then
                local link = GetInventoryItemLink("player", durSlot)
                if link then
                    print(format(ns.FMT.DURABILITY, link, pct))
                end
            end
        end
    end
end

function events:PLAYER_EQUIPMENT_CHANGED()
    db.eventCount = db.eventCount + 1
    CheckDurability()
end

function events:PLAYER_ENTERING_WORLD()
    db.eventCount = db.eventCount + 1
    CheckDurability()
end
```

!!! success "Performance Win: Scan Only on Change Events"
    Compare these two approaches:

    === "Bad: Timer-Based Polling"
        ```lua
        -- BAD: Burns CPU every 5 seconds whether anything changed or not
        C_Timer.NewTicker(5, function()
            CheckDurability()
        end)
        ```

    === "Good: Event-Driven"
        ```lua
        -- GOOD: Runs ONLY when equipment actually changes
        function events:PLAYER_EQUIPMENT_CHANGED()
            CheckDurability()
        end
        ```

    The timer version runs 12 times per minute, 720 times per hour — the vast majority of which find nothing changed. The event version runs only when the game engine tells us something changed. Over a 4-hour raid session, that could be the difference between 2,880 unnecessary scans and perhaps 10 meaningful ones.

### Equipment Slot IDs

The `ns.EQUIPMENT_SLOTS` table contains the inventory slot IDs for all equipment positions that have durability. We exclude slots that never have durability (trinkets, rings, neck, cloak slot 15 has durability on cloaks). The slot IDs are:

| Slot ID | Equipment |
|---------|-----------|
| 1 | Head |
| 2 | Neck |
| 3 | Shoulder |
| 5 | Chest |
| 6 | Waist |
| 7 | Legs |
| 8 | Feet |
| 9 | Wrist |
| 10 | Hands |
| 11-12 | Rings |
| 13-14 | Trinkets |
| 15 | Cloak |
| 16-17 | Weapons |
| 19 | Tabard |

`GetInventoryItemDurability(slot)` returns `nil, nil` for slots without durability, so the `if durCurrent and durMax` guard handles non-durability slots gracefully.

---

## Step 8: The /perf Diagnostic Command

Performance Zealot mode **requires** every addon to include a `/perf` diagnostic command. This is not optional — it is how you verify that your addon is actually performing well, and it is how other developers can audit your addon's resource usage.

### What to Measure

The diagnostic should report:

1. **Memory usage** — how many kilobytes your addon occupies
2. **Event count** — how many events your addon has processed since login
3. **Feature status** — which features are active (helps correlate memory with behavior)

### Implementation

```lua title="Config.lua — /perf command"
-- Mode: Performance Zealot | Zero-waste addon
local addonName, ns = ...

-- PERF: Cached globals
local format = string.format
local print = print
local collectgarbage = collectgarbage
local UpdateAddOnMemoryUsage = UpdateAddOnMemoryUsage
local GetAddOnMemoryUsage = GetAddOnMemoryUsage
local GetCoinTextureString = GetCoinTextureString

-- /perf diagnostic
SLASH_REPAIRBUDDYPERF1 = "/repairbuddyperf"
SLASH_REPAIRBUDDYPERF2 = "/rbperf"
SlashCmdList["REPAIRBUDDYPERF"] = function()
    UpdateAddOnMemoryUsage()
    local mem = GetAddOnMemoryUsage(addonName)
    local db = ns.db

    print(format(ns.FMT.PERF_REPORT, mem, db.eventCount))
    print(format("|cff00ff00RepairBuddy|r: Repairs: %d | Cost: %s | Junk: %s",
        db.repairCount,
        GetCoinTextureString(db.totalRepairCost),
        GetCoinTextureString(db.totalJunkValue)))
end
```

!!! tip "Reading the Memory Number"
    `GetAddOnMemoryUsage()` returns kilobytes. A well-written utility addon like RepairBuddy should use **under 20 KB**. If you see numbers climbing over time (memory leak), you are creating tables or strings that are not being collected. Run `collectgarbage("collect")` before measuring to get a clean reading of retained memory.

### Main Slash Command

```lua title="Config.lua — main slash command"
SLASH_REPAIRBUDDY1 = "/repairbuddy"
SLASH_REPAIRBUDDY2 = "/rb"
SlashCmdList["REPAIRBUDDY"] = function(msg)
    local cmd = (msg or ""):lower()
    if cmd == "config" or cmd == "settings" then
        Settings.OpenToCategory(ns.categoryID)
    elseif cmd == "perf" then
        SlashCmdList["REPAIRBUDDYPERF"]()
    elseif cmd == "stats" then
        local db = ns.db
        print(format("|cff00ff00RepairBuddy|r Stats:"))
        print(format("  Total repairs: %d", db.repairCount))
        print(format("  Total repair cost: %s", GetCoinTextureString(db.totalRepairCost)))
        print(format("  Session repair cost: %s", GetCoinTextureString(db.sessionRepairCost)))
        print(format("  Total junk sold: %s", GetCoinTextureString(db.totalJunkValue)))
    else
        print("|cff00ff00RepairBuddy|r: /rb config | stats | perf")
    end
end
```

---

## Step 9: Settings and Data Visualization

### Settings API Panel

RepairBuddy uses the modern Settings API for its configuration panel — no legacy `InterfaceOptions` calls.

```lua title="Config.lua — settings panel"
local function RegisterSettings()
    local db = ns.db
    local category, layout = Settings.RegisterVerticalLayoutCategory(addonName)
    ns.categoryID = category:GetID()

    -- Auto-repair toggle
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "autoRepair", "autoRepair", db, type(true),
            "Auto-Repair", true)
        Settings.CreateCheckbox(category, setting,
            "Automatically repair all equipment when visiting a repair vendor.")
    end

    -- Guild funds toggle
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "useGuildFunds", "useGuildFunds", db, type(true),
            "Use Guild Funds", true)
        Settings.CreateCheckbox(category, setting,
            "Use guild bank funds for repairs when available.")
    end

    -- Auto-sell junk toggle
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "autoSellJunk", "autoSellJunk", db, type(true),
            "Auto-Sell Junk", true)
        Settings.CreateCheckbox(category, setting,
            "Automatically sell grey-quality items when visiting a vendor.")
    end

    -- Durability warning toggle
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "durabilityWarning", "durabilityWarning", db, type(true),
            "Durability Warnings", true)
        Settings.CreateCheckbox(category, setting,
            "Show a warning when any equipped item falls below the durability threshold.")
    end

    -- Durability threshold slider
    do
        local setting = Settings.RegisterAddOnSetting(category,
            "durabilityThreshold", "durabilityThreshold", db, type(1),
            "Warning Threshold (%)", 20)
        local options = Settings.CreateSliderOptions(5, 50, 5)
        Settings.CreateSlider(category, setting, options,
            "Warn when any item drops below this durability percentage.")
    end

    Settings.RegisterAddOnCategory(category)
end

EventUtil.ContinueOnAddOnLoaded(addonName, RegisterSettings)
```

### Addon Compartment

```lua title="Config.lua — addon compartment handlers"
function RepairBuddy_OnAddonCompartmentClick()
    Settings.OpenToCategory(ns.categoryID)
end

function RepairBuddy_OnAddonCompartmentEnter(_, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("RepairBuddy")
    if ns.db then
        GameTooltip:AddLine(format("Total repairs: %s",
            GetCoinTextureString(ns.db.totalRepairCost)), 1, 1, 1)
        GameTooltip:AddLine(format("Session: %s",
            GetCoinTextureString(ns.db.sessionRepairCost)), 1, 1, 1)
    end
    GameTooltip:AddLine("Click to open settings", 0.7, 0.7, 0.7)
    GameTooltip:Show()
end

function RepairBuddy_OnAddonCompartmentLeave()
    GameTooltip:Hide()
end
```

---

## Step 10: Complete Code

Here is the complete, copy-pasteable code for all three files. Drop the `RepairBuddy` directory into `Interface/AddOns/` and reload your UI.

### Data.lua

```lua title="Data.lua"
-- Mode: Performance Zealot | Zero-waste addon
local addonName, ns = ...

-- PERF: Pre-built format strings — allocated once at load time
ns.FMT = {
    REPAIR_COST   = "|cff00ff00RepairBuddy|r: Repaired for %s",
    REPAIR_GUILD  = "|cff00ff00RepairBuddy|r: Repaired for %s (guild funds)",
    REPAIR_BROKE  = "|cff00ff00RepairBuddy|r: Not enough gold to repair (%s needed)",
    JUNK_SOLD     = "|cff00ff00RepairBuddy|r: Sold %d junk item(s) for %s",
    JUNK_NONE     = "|cff00ff00RepairBuddy|r: No junk to sell",
    DURABILITY    = "|cffff6600RepairBuddy|r: Warning — %s is at %d%% durability",
    PERF_REPORT   = "|cff00ff00RepairBuddy|r: %.1f KB | %d events handled",
}

-- Defaults for SavedVariables
ns.defaults = {
    autoRepair = true,
    useGuildFunds = true,
    autoSellJunk = true,
    durabilityWarning = true,
    durabilityThreshold = 20,
    totalRepairCost = 0,
    sessionRepairCost = 0,
    totalJunkValue = 0,
    repairCount = 0,
    eventCount = 0,
}

-- PERF: Static equipment slot IDs (slots that can have durability)
ns.EQUIPMENT_SLOTS = {
    1, 2, 3, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 19
}
ns.NUM_EQUIPMENT_SLOTS = #ns.EQUIPMENT_SLOTS

-- PERF: Static bag IDs
ns.BAG_IDS = { 0, 1, 2, 3, 4 }
ns.NUM_BAGS = #ns.BAG_IDS
```

### Core.lua

```lua title="Core.lua"
-- Mode: Performance Zealot | Zero-waste addon
local addonName, ns = ...

-- PERF: Cached Lua globals
local pairs = pairs
local format = string.format
local print = print

-- PERF: Cached WoW API
local GetRepairAllCost = GetRepairAllCost
local RepairAllItems = RepairAllItems
local CanMerchantRepair = CanMerchantRepair
local GetMoney = GetMoney
local GetCoinTextureString = GetCoinTextureString
local GetInventoryItemDurability = GetInventoryItemDurability
local GetInventoryItemLink = GetInventoryItemLink
local IsInGuild = IsInGuild
local C_Container = C_Container

-- PERF: Reusable counters — no allocations in hot paths
local junkCount = 0
local junkValue = 0
local durSlot = 0
local durCurrent = 0
local durMax = 0

-- Local reference to SavedVariables (set in ADDON_LOADED)
local db

----------------------------------------------------------------
-- Junk Selling
----------------------------------------------------------------
local function SellJunk()
    junkCount = 0
    junkValue = 0

    for i = 1, ns.NUM_BAGS do
        local bagID = ns.BAG_IDS[i]
        local numSlots = C_Container.GetContainerNumSlots(bagID)

        for slot = 1, numSlots do
            local info = C_Container.GetContainerItemInfo(bagID, slot)

            if info and info.quality == Enum.ItemQuality.Poor and not info.hasNoValue then
                C_Container.UseContainerItem(bagID, slot)
                junkValue = junkValue + (info.sellPrice * info.stackCount)
                junkCount = junkCount + 1
            end
        end
    end

    if junkCount > 0 then
        print(format(ns.FMT.JUNK_SOLD, junkCount, GetCoinTextureString(junkValue)))
        db.totalJunkValue = db.totalJunkValue + junkValue
    end
end

----------------------------------------------------------------
-- Durability Check
----------------------------------------------------------------
local function CheckDurability()
    if not db.durabilityWarning then return end

    local threshold = db.durabilityThreshold

    for i = 1, ns.NUM_EQUIPMENT_SLOTS do
        durSlot = ns.EQUIPMENT_SLOTS[i]
        durCurrent, durMax = GetInventoryItemDurability(durSlot)

        if durCurrent and durMax and durMax > 0 then
            local pct = (durCurrent / durMax) * 100

            if pct <= threshold and pct > 0 then
                local link = GetInventoryItemLink("player", durSlot)
                if link then
                    print(format(ns.FMT.DURABILITY, link, pct))
                end
            end
        end
    end
end

----------------------------------------------------------------
-- Event Handlers
----------------------------------------------------------------
local events = {}

function events:ADDON_LOADED(loadedAddon)
    if loadedAddon ~= addonName then return end

    if not RepairBuddyDB then
        RepairBuddyDB = {}
    end
    for k, v in pairs(ns.defaults) do
        if RepairBuddyDB[k] == nil then
            RepairBuddyDB[k] = v
        end
    end

    db = RepairBuddyDB
    ns.db = db
    db.sessionRepairCost = 0

    self:UnregisterEvent("ADDON_LOADED")
end

function events:MERCHANT_SHOW()
    db.eventCount = db.eventCount + 1

    if db.autoRepair and CanMerchantRepair() then
        local cost, canRepair = GetRepairAllCost()

        if canRepair and cost > 0 then
            local playerMoney = GetMoney()

            if db.useGuildFunds and IsInGuild() then
                RepairAllItems(true)
                print(format(ns.FMT.REPAIR_GUILD, GetCoinTextureString(cost)))
            elseif playerMoney >= cost then
                RepairAllItems(false)
                print(format(ns.FMT.REPAIR_COST, GetCoinTextureString(cost)))
            else
                print(format(ns.FMT.REPAIR_BROKE, GetCoinTextureString(cost)))
                return
            end

            db.totalRepairCost = db.totalRepairCost + cost
            db.sessionRepairCost = db.sessionRepairCost + cost
            db.repairCount = db.repairCount + 1
        end
    end

    if db.autoSellJunk then
        SellJunk()
    end
end

function events:PLAYER_EQUIPMENT_CHANGED()
    db.eventCount = db.eventCount + 1
    CheckDurability()
end

function events:PLAYER_ENTERING_WORLD()
    db.eventCount = db.eventCount + 1
    CheckDurability()
end

----------------------------------------------------------------
-- Event Frame
----------------------------------------------------------------
local eventFrame = CreateFrame("Frame")
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = events[event]
    if handler then
        handler(self, ...)
    end
end)

for event in pairs(events) do
    eventFrame:RegisterEvent(event)
end
```

### Config.lua

```lua title="Config.lua"
-- Mode: Performance Zealot | Zero-waste addon
local addonName, ns = ...

-- PERF: Cached globals
local format = string.format
local print = print
local GetCoinTextureString = GetCoinTextureString
local UpdateAddOnMemoryUsage = UpdateAddOnMemoryUsage
local GetAddOnMemoryUsage = GetAddOnMemoryUsage

----------------------------------------------------------------
-- Settings Panel
----------------------------------------------------------------
local function RegisterSettings()
    local db = ns.db
    local category, layout = Settings.RegisterVerticalLayoutCategory(addonName)
    ns.categoryID = category:GetID()

    do
        local setting = Settings.RegisterAddOnSetting(category,
            "autoRepair", "autoRepair", db, type(true),
            "Auto-Repair", true)
        Settings.CreateCheckbox(category, setting,
            "Automatically repair all equipment when visiting a repair vendor.")
    end

    do
        local setting = Settings.RegisterAddOnSetting(category,
            "useGuildFunds", "useGuildFunds", db, type(true),
            "Use Guild Funds", true)
        Settings.CreateCheckbox(category, setting,
            "Use guild bank funds for repairs when available.")
    end

    do
        local setting = Settings.RegisterAddOnSetting(category,
            "autoSellJunk", "autoSellJunk", db, type(true),
            "Auto-Sell Junk", true)
        Settings.CreateCheckbox(category, setting,
            "Automatically sell grey-quality items when visiting a vendor.")
    end

    do
        local setting = Settings.RegisterAddOnSetting(category,
            "durabilityWarning", "durabilityWarning", db, type(true),
            "Durability Warnings", true)
        Settings.CreateCheckbox(category, setting,
            "Show a warning when any equipped item falls below the durability threshold.")
    end

    do
        local setting = Settings.RegisterAddOnSetting(category,
            "durabilityThreshold", "durabilityThreshold", db, type(1),
            "Warning Threshold (%)", 20)
        local options = Settings.CreateSliderOptions(5, 50, 5)
        Settings.CreateSlider(category, setting, options,
            "Warn when any item drops below this durability percentage.")
    end

    Settings.RegisterAddOnCategory(category)
end

EventUtil.ContinueOnAddOnLoaded(addonName, RegisterSettings)

----------------------------------------------------------------
-- Slash Commands
----------------------------------------------------------------
SLASH_REPAIRBUDDYPERF1 = "/repairbuddyperf"
SLASH_REPAIRBUDDYPERF2 = "/rbperf"
SlashCmdList["REPAIRBUDDYPERF"] = function()
    UpdateAddOnMemoryUsage()
    local mem = GetAddOnMemoryUsage(addonName)
    local db = ns.db

    print(format(ns.FMT.PERF_REPORT, mem, db.eventCount))
    print(format("|cff00ff00RepairBuddy|r: Repairs: %d | Cost: %s | Junk: %s",
        db.repairCount,
        GetCoinTextureString(db.totalRepairCost),
        GetCoinTextureString(db.totalJunkValue)))
end

SLASH_REPAIRBUDDY1 = "/repairbuddy"
SLASH_REPAIRBUDDY2 = "/rb"
SlashCmdList["REPAIRBUDDY"] = function(msg)
    local cmd = (msg or ""):lower()
    if cmd == "config" or cmd == "settings" then
        Settings.OpenToCategory(ns.categoryID)
    elseif cmd == "perf" then
        SlashCmdList["REPAIRBUDDYPERF"]()
    elseif cmd == "stats" then
        local db = ns.db
        print("|cff00ff00RepairBuddy|r Stats:")
        print(format("  Total repairs: %d", db.repairCount))
        print(format("  Total repair cost: %s", GetCoinTextureString(db.totalRepairCost)))
        print(format("  Session repair cost: %s", GetCoinTextureString(db.sessionRepairCost)))
        print(format("  Total junk sold: %s", GetCoinTextureString(db.totalJunkValue)))
    else
        print("|cff00ff00RepairBuddy|r: /rb config | stats | perf")
    end
end

----------------------------------------------------------------
-- Addon Compartment
----------------------------------------------------------------
function RepairBuddy_OnAddonCompartmentClick()
    Settings.OpenToCategory(ns.categoryID)
end

function RepairBuddy_OnAddonCompartmentEnter(_, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("RepairBuddy")
    if ns.db then
        GameTooltip:AddLine(format("Total repairs: %s",
            GetCoinTextureString(ns.db.totalRepairCost)), 1, 1, 1)
        GameTooltip:AddLine(format("Session: %s",
            GetCoinTextureString(ns.db.sessionRepairCost)), 1, 1, 1)
    end
    GameTooltip:AddLine("Click to open settings", 0.7, 0.7, 0.7)
    GameTooltip:Show()
end

function RepairBuddy_OnAddonCompartmentLeave()
    GameTooltip:Hide()
end
```

---

## Performance Checklist

Before shipping, verify RepairBuddy against the Performance Zealot checklist:

| Check | Status |
|-------|--------|
| All globals cached as locals | :material-check: Every file has a caching block |
| No table creation in hot paths | :material-check: Counters are file-scope locals, reset with assignment |
| No string concatenation in loops | :material-check: All output uses pre-built `ns.FMT` templates |
| No OnUpdate handlers | :material-check: Purely event-driven architecture |
| No timers for polling | :material-check: Durability checks run on change events only |
| Events unregistered when done | :material-check: `ADDON_LOADED` unregistered after init |
| `/perf` diagnostic included | :material-check: `/rbperf` reports memory and event count |
| `wipe()` used over `t = {}` | :material-check: No runtime table allocation in any feature |
| Numeric for over sequential data | :material-check: Bag and slot iteration uses numeric loops |
| Single event frame for entire addon | :material-check: One `eventFrame` handles all events |

Run `/rbperf` after a play session. RepairBuddy should report under 15 KB of memory usage. If you see it climbing over time, you have a leak — check for tables or strings being created and never released.

!!! tip "Next Steps"
    Want to extend RepairBuddy? Here are some ideas that stay within Performance Zealot constraints:

    - **Repair cost graph** — store daily totals in SavedVariables, render with a simple StatusBar-based chart
    - **Auto-delete specific items** — maintain a whitelist in SavedVariables, check against it during the junk scan
    - **Durability percentage on character frame** — hook the character panel to overlay a FontString (use Enhancement Artist mode for this, since it requires `hooksecurefunc` on Blizzard frames)
