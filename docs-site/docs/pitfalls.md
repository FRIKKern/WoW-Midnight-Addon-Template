# Pitfalls & Anti-Patterns

AI code generators (ChatGPT, Copilot, Claude, etc.) frequently produce WoW addon code that **looks correct but fails at runtime**. These models were trained on outdated documentation, deprecated API examples, and standard Lua patterns that don't apply to WoW's sandboxed environment.

This page catalogs the most common mistakes so you can catch them in code review — whether the code was written by AI or by hand.

---

## 1. Using Deprecated GetSpellInfo()

The standalone `GetSpellInfo()` was removed in Dragonflight (10.x). The replacement `C_Spell.GetSpellInfo()` returns a **table**, not multiple return values — this trips up almost every AI model.

!!! danger "Wrong: Deprecated API with wrong return handling"
    ```lua
    -- AI generates this constantly — double wrong
    local name, rank, icon, castTime, minRange, maxRange, spellID = GetSpellInfo(116)
    print("Spell:", name, "Icon:", icon)
    ```

!!! tip "Correct: C_Spell namespace returns a table"
    ```lua
    local spellInfo = C_Spell.GetSpellInfo(116)  -- Returns a SpellInfo table or nil
    if spellInfo then
        print("Spell:", spellInfo.name, "Icon:", spellInfo.iconID)
        -- Available fields: name, iconID, originalIconID, castTime, minRange, maxRange, spellID, spellSubtext
    end
    ```

!!! warning
    Many `C_Spell` functions are **asynchronous**. If `C_Spell.GetSpellInfo()` returns `nil`, the data may not be cached yet. Use `C_Spell.RequestLoadSpellData(spellID)` and listen for `SPELL_DATA_LOAD_RESULT` to be notified when it's available.

---

## 2. Using Deprecated GetItemInfo()

Same story as `GetSpellInfo()` — the standalone `GetItemInfo()` was deprecated and `C_Item.GetItemInfo()` returns a **table**.

!!! danger "Wrong: Deprecated multi-return API"
    ```lua
    local itemName, itemLink, itemQuality, itemLevel = GetItemInfo(19019)
    if itemName then
        print(itemName .. " is quality " .. itemQuality)
    end
    ```

!!! tip "Correct: C_Item namespace returns a table"
    ```lua
    local itemInfo = C_Item.GetItemInfo(19019)  -- Returns an ItemInfo table or nil
    if itemInfo then
        print(itemInfo.itemName .. " is quality " .. itemInfo.itemQuality)
    end
    ```

Item data is also asynchronous. If the item isn't cached, use the `ITEM_DATA_LOAD_RESULT` event or `C_Item.RequestLoadItemData(itemID)`:

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("ITEM_DATA_LOAD_RESULT")
frame:SetScript("OnEvent", function(self, event, itemID, success)
    if success then
        local itemInfo = C_Item.GetItemInfo(itemID)
        -- Now guaranteed to have data
    end
end)

C_Item.RequestLoadItemData(19019)
```

---

## 3. Accessing SavedVariables Before ADDON_LOADED

SavedVariables are `nil` until the game loads them. Accessing them at file parse time is a guaranteed `nil` read.

!!! danger "Wrong: Accessing SavedVariables at file scope"
    ```lua
    local addonName, ns = ...

    -- This runs at FILE LOAD TIME — SavedVariables haven't been deserialized yet!
    MyAddonDB = MyAddonDB or {}
    MyAddonDB.settings = MyAddonDB.settings or { enabled = true }

    local frame = CreateFrame("Frame")
    frame:RegisterEvent("PLAYER_LOGIN")
    frame:SetScript("OnEvent", function()
        print("Settings:", MyAddonDB.settings.enabled)  -- Overwrote saved data!
    end)
    ```

!!! tip "Correct: Initialize SavedVariables inside ADDON_LOADED"
    ```lua
    local addonName, ns = ...

    local frame = CreateFrame("Frame")
    frame:RegisterEvent("ADDON_LOADED")
    frame:SetScript("OnEvent", function(self, event, loadedAddon)
        if loadedAddon ~= addonName then return end
        self:UnregisterEvent("ADDON_LOADED")

        -- NOW SavedVariables are available
        MyAddonDB = MyAddonDB or {}
        MyAddonDB.settings = MyAddonDB.settings or { enabled = true }

        ns.db = MyAddonDB  -- Store reference in namespace for other files
    end)
    ```

!!! warning
    The `ADDON_LOADED` event fires once per addon. Always check `loadedAddon == addonName` — you'll receive this event for **every** addon that loads, not just yours.

---

## 4. Global Namespace Pollution

Every WoW addon shares a single Lua global table. AI loves to create globals everywhere. This causes mysterious cross-addon conflicts.

!!! danger "Wrong: Globals everywhere"
    ```lua
    -- No local keyword — these are ALL global!
    frame = CreateFrame("Frame")
    db = {}
    settings = { scale = 1.0 }
    ADDON_VERSION = "1.0.0"

    function Initialize()
        -- Overwrites any other addon's Initialize function!
    end

    function OnUpdate(self, elapsed)
        -- Same problem
    end
    ```

!!! tip "Correct: Use the addon namespace and locals"
    ```lua
    local addonName, ns = ...

    local frame = CreateFrame("Frame")

    ns.db = {}
    ns.settings = { scale = 1.0 }
    ns.ADDON_VERSION = "1.0.0"

    local function Initialize()
        -- File-local, no conflict possible
    end

    local function OnUpdate(self, elapsed)
        -- File-local
    end

    frame:SetScript("OnUpdate", OnUpdate)
    ```

The `local addonName, ns = ...` pattern is provided by the WoW loader. The `ns` table is shared across all files listed in your `.toc` — use it instead of globals.

---

## 5. Using require(), dofile(), or Standard Lua I/O

WoW's Lua sandbox removes most of the standard library. AI models trained on general Lua constantly reach for these.

!!! danger "Wrong: Standard Lua patterns that don't exist in WoW"
    ```lua
    -- NONE of these exist in WoW's Lua environment
    local json = require("json")
    dofile("config.lua")
    loadfile("data.lua")

    local f = io.open("settings.txt", "r")
    local data = f:read("*all")

    os.execute("echo hello")
    print(os.time())
    ```

!!! tip "Correct: Use WoW's module system"
    ```lua
    local addonName, ns = ...

    -- Multi-file sharing: all .lua files in your .toc share the ns table
    -- File: Core.lua
    ns.Core = {}
    function ns.Core:Init()
        -- ...
    end

    -- File: Config.lua (loaded after Core.lua via .toc order)
    ns.Config = {}
    function ns.Config:Load()
        ns.Core:Init()  -- Access other files through the namespace
    end

    -- For time, use WoW APIs:
    local timestamp = GetTime()          -- Game time in seconds (float)
    local serverTime = GetServerTime()   -- Server epoch time
    local dateInfo = C_DateAndTime.GetCurrentCalendarTime()
    ```

Your `.toc` file controls load order. Files are executed top-to-bottom:

```toc
## Title: MyAddon
Core.lua
Config.lua
UI.lua
```

---

## 6. Trying to Cancel C_Timer.After()

`C_Timer.After()` returns **nil**. It cannot be cancelled. AI models constantly try to store and cancel its "handle."

!!! danger "Wrong: Assuming C_Timer.After returns a handle"
    ```lua
    -- C_Timer.After returns NIL — this stores nothing!
    local timer = C_Timer.After(5, function()
        print("Delayed action")
    end)

    -- This crashes: attempt to index a nil value
    timer:Cancel()
    ```

!!! tip "Correct: Use C_Timer.NewTimer() for cancellable timers"
    ```lua
    -- C_Timer.NewTimer returns a ticker object with a Cancel method
    local timer = C_Timer.NewTimer(5, function()
        print("Delayed action")
    end)

    -- This works!
    timer:Cancel()

    -- For repeating timers, use C_Timer.NewTicker
    local ticker = C_Timer.NewTicker(1, function()
        print("Every second")
    end)

    -- Stop repeating
    ticker:Cancel()
    ```

Quick reference:

| Function | Returns | Cancellable | Repeats |
|---|---|---|---|
| `C_Timer.After(sec, fn)` | `nil` | No | No |
| `C_Timer.NewTimer(sec, fn)` | Timer object | Yes | No |
| `C_Timer.NewTicker(sec, fn)` | Ticker object | Yes | Yes |

---

## 7. Creating Frames During Combat

Creating frames that use secure templates during combat lockdown silently fails or taints the UI. This is one of the hardest bugs to track down because it only manifests in combat.

!!! danger "Wrong: No combat check before frame creation"
    ```lua
    function ns:ShowLootFrame(items)
        -- If called during combat, this can fail or taint the UI!
        local frame = CreateFrame("Frame", "MyLootFrame", UIParent, "SecureHandlerBaseTemplate")
        frame:SetSize(300, 400)
        frame:Show()

        for i, item in ipairs(items) do
            local btn = CreateFrame("Button", nil, frame, "SecureActionButtonTemplate")
            btn:SetAttribute("type", "item")
            btn:SetAttribute("item", item)
        end
    end
    ```

!!! tip "Correct: Check combat lockdown and queue if needed"
    ```lua
    local pendingActions = {}

    local combatFrame = CreateFrame("Frame")
    combatFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
    combatFrame:SetScript("OnEvent", function()
        for _, action in ipairs(pendingActions) do
            action()
        end
        wipe(pendingActions)
    end)

    function ns:ShowLootFrame(items)
        if InCombatLockdown() then
            table.insert(pendingActions, function() ns:ShowLootFrame(items) end)
            print("Will show after combat ends.")
            return
        end

        local frame = CreateFrame("Frame", "MyLootFrame", UIParent, "SecureHandlerBaseTemplate")
        frame:SetSize(300, 400)
        frame:Show()

        for i, item in ipairs(items) do
            local btn = CreateFrame("Button", nil, frame, "SecureActionButtonTemplate")
            btn:SetAttribute("type", "item")
            btn:SetAttribute("item", item)
        end
    end
    ```

!!! tip
    Non-secure frames (plain `"Frame"`, `"Button"` without secure templates) **can** be created during combat. The restriction only applies to frames using secure templates or calling `SetAttribute()` on secure frames.

---

## 8. If/Elseif Event Handling

AI almost always generates long `if/elseif` chains for event handling. This is slow, hard to maintain, and doesn't scale.

!!! danger "Wrong: if/elseif chain for events"
    ```lua
    local frame = CreateFrame("Frame")
    frame:RegisterEvent("PLAYER_LOGIN")
    frame:RegisterEvent("PLAYER_ENTERING_WORLD")
    frame:RegisterEvent("ADDON_LOADED")
    frame:RegisterEvent("UNIT_HEALTH")
    frame:RegisterEvent("PLAYER_REGEN_ENABLED")
    frame:RegisterEvent("PLAYER_REGEN_DISABLED")

    frame:SetScript("OnEvent", function(self, event, ...)
        if event == "PLAYER_LOGIN" then
            -- handle login
        elseif event == "PLAYER_ENTERING_WORLD" then
            -- handle entering world
        elseif event == "ADDON_LOADED" then
            -- handle addon loaded
        elseif event == "UNIT_HEALTH" then
            -- handle health (fires VERY frequently!)
        elseif event == "PLAYER_REGEN_ENABLED" then
            -- handle leaving combat
        elseif event == "PLAYER_REGEN_DISABLED" then
            -- handle entering combat
        end
    end)
    ```

!!! tip "Correct: Table dispatch pattern"
    ```lua
    local addonName, ns = ...

    local frame = CreateFrame("Frame")
    local events = {}

    function events:PLAYER_LOGIN()
        -- handle login
    end

    function events:PLAYER_ENTERING_WORLD(isInitialLogin, isReloadingUI)
        -- handle entering world
    end

    function events:ADDON_LOADED(loadedAddon)
        if loadedAddon ~= addonName then return end
        -- handle addon loaded
    end

    function events:UNIT_HEALTH(unit)
        -- handle health — O(1) dispatch, no chain of comparisons
    end

    function events:PLAYER_REGEN_ENABLED()
        -- handle leaving combat
    end

    function events:PLAYER_REGEN_DISABLED()
        -- handle entering combat
    end

    -- Auto-register all events in the table
    for event in pairs(events) do
        frame:RegisterEvent(event)
    end

    frame:SetScript("OnEvent", function(self, event, ...)
        events[event](self, ...)
    end)
    ```

Table dispatch is O(1) per event. An `if/elseif` chain is O(n) and gets worse with every event you add. For high-frequency events like `UNIT_HEALTH`, this matters.

---

## 9. Table Creation in OnUpdate Handlers

`OnUpdate` fires every rendered frame (60–240+ FPS). Creating tables inside it generates massive garbage collection pressure.

!!! danger "Wrong: New tables every frame"
    ```lua
    frame:SetScript("OnUpdate", function(self, elapsed)
        local pos = { x = 0, y = 0 }           -- NEW table every frame!
        local colors = { 1.0, 0.5, 0.0 }       -- Another one!
        local result = format("(%s, %s)", tostring(pos.x), tostring(pos.y))

        for _, unit in pairs({ "player", "target", "focus" }) do  -- Yet another!
            -- process units
        end
    end)
    ```

!!! tip "Correct: Pre-allocate and reuse tables"
    ```lua
    local pos = { x = 0, y = 0 }
    local colors = { 1.0, 0.5, 0.0 }
    local units = { "player", "target", "focus" }

    frame:SetScript("OnUpdate", function(self, elapsed)
        pos.x, pos.y = 0, 0          -- Reuse existing table
        colors[1], colors[2], colors[3] = 1.0, 0.5, 0.0

        for _, unit in ipairs(units) do  -- Reuse existing table
            -- process units
        end
    end)
    ```

Also throttle your `OnUpdate` if you don't need per-frame updates:

```lua
local INTERVAL = 0.1
local elapsed_acc = 0

frame:SetScript("OnUpdate", function(self, elapsed)
    elapsed_acc = elapsed_acc + elapsed
    if elapsed_acc < INTERVAL then return end
    elapsed_acc = elapsed_acc - INTERVAL

    -- Actual work here, runs ~10 times per second
end)
```

---

## 10. Using Lua 5.2+ Features

WoW uses **Lua 5.1** (with some custom extensions). AI models often generate Lua 5.2, 5.3, or 5.4 syntax that fails immediately.

!!! danger "Wrong: Lua 5.2+ features that don't exist in WoW"
    ```lua
    -- goto (Lua 5.2+) — syntax error in WoW
    for i = 1, 10 do
        if i == 5 then goto continue end
        print(i)
        ::continue::
    end

    -- Bitwise operators (Lua 5.3+) — syntax error in WoW
    local flags = 0xFF & 0x0F
    local shifted = flags << 4

    -- Integer division (Lua 5.3+) — syntax error in WoW
    local half = 10 // 2

    -- _ENV manipulation (Lua 5.2+) — doesn't exist
    _ENV = setmetatable({}, { __index = _G })

    -- utf8 library (Lua 5.3+) — doesn't exist
    local len = utf8.len("hello")
    ```

!!! tip "Correct: Lua 5.1 equivalents"
    ```lua
    -- Skip pattern instead of goto
    for i = 1, 10 do
        if i ~= 5 then
            print(i)
        end
    end

    -- Use bit library (WoW provides bit.band, bit.bor, bit.lshift, etc.)
    local flags = bit.band(0xFF, 0x0F)
    local shifted = bit.lshift(flags, 4)

    -- Use math.floor for integer division
    local half = math.floor(10 / 2)

    -- Use setfenv for environment manipulation (Lua 5.1)
    -- But note: setfenv taints execution, avoid if possible

    -- Use string.len for byte length (no native UTF-8 support)
    local len = string.len("hello")
    ```

WoW's Lua 5.1 **does** include some extensions: `...` varargs in file scope, `unpack()` as a global, and the `bit` library. But core syntax is strictly 5.1.

---

## 11. Wrong SetTexCoord Parameter Order

`SetTexCoord` crops a texture atlas. Getting the parameter order wrong results in distorted or invisible textures.

!!! danger "Wrong: Incorrect parameter order or values"
    ```lua
    -- Wrong: using pixel coordinates instead of 0-1 normalized
    texture:SetTexCoord(64, 128, 0, 64)

    -- Wrong: common AI mistake — swapping rows
    texture:SetTexCoord(left, top, right, bottom)  -- WRONG order for 4-arg!
    ```

!!! tip "Correct: Normalized coordinates in the right order"
    ```lua
    -- 4-argument form: left, right, top, bottom (NOT left, top, right, bottom!)
    texture:SetTexCoord(0.25, 0.50, 0.0, 0.5)
    --                  left  right top  bottom

    -- 8-argument form for arbitrary quads (ULx, ULy, LLx, LLy, URx, URy, LRx, LRy)
    -- Upper-Left, Lower-Left, Upper-Right, Lower-Right
    texture:SetTexCoord(
        0.25, 0.0,   -- Upper-left
        0.25, 0.5,   -- Lower-left
        0.50, 0.0,   -- Upper-right
        0.50, 0.5    -- Lower-right
    )
    ```

The 4-argument order is `left, right, top, bottom` — **not** `left, top, right, bottom`. This is the single most common parameter-order mistake in WoW texture code.

---

## 12. Assuming COMBAT_LOG_EVENT_UNFILTERED Still Works

`COMBAT_LOG_EVENT_UNFILTERED` (CLEU) was **removed entirely in Patch 12.0 (Midnight)**. Code that registers for it will silently receive nothing — no error, no warning, just silence. Both `CombatLogGetCurrentEventInfo()` and the event itself are gone.

!!! danger "Wrong: Any CLEU-based code (completely broken in 12.0+)"
    ```lua
    -- This event no longer exists in Midnight — handler never fires
    frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
    frame:SetScript("OnEvent", function(self, event)
        local _, subevent = CombatLogGetCurrentEventInfo() -- API removed
        if subevent == "SPELL_DAMAGE" then
            print("damage!")
        end
    end)
    ```

!!! tip "Correct: Use unit-specific events instead"
    ```lua
    frame:RegisterUnitEvent("UNIT_HEALTH", "player", "target")
    frame:RegisterUnitEvent("UNIT_AURA", "player")
    frame:RegisterEvent("UNIT_POWER_UPDATE")
    frame:RegisterEvent("UNIT_COMBAT")

    frame:SetScript("OnEvent", function(self, event, unit, ...)
        if event == "UNIT_HEALTH" then
            local hp = UnitHealth(unit)
            local max = UnitHealthMax(unit)
            print(unit .. " health: " .. hp .. "/" .. max)
        elseif event == "UNIT_AURA" then
            -- Use C_UnitAuras.GetAuraDataByIndex() to inspect auras
        end
    end)
    ```

!!! warning
    There is no 1:1 replacement for CLEU's full combat log stream. Migrate to the specific unit events that cover your actual use case: `UNIT_HEALTH`, `UNIT_AURA`, `UNIT_POWER_UPDATE`, `UNIT_COMBAT`, etc.

---

## 13. Not Handling nil Returns from Async APIs

Many modern WoW APIs are asynchronous — they return `nil` on first call and require an event callback. AI rarely generates the nil-check + event pattern.

!!! danger "Wrong: Assuming API always returns data"
    ```lua
    function ns:ShowPlayerItemLevel()
        local avgItemLevel, avgItemLevelEquipped = GetAverageItemLevel()
        -- Might work... but what about these?

        local specInfo = C_Spell.GetSpellInfo(12345)
        print("Spell: " .. specInfo.name)  -- CRASH if specInfo is nil!

        local itemInfo = C_Item.GetItemInfo(19019)
        local tex = itemInfo.iconFileID  -- CRASH if itemInfo is nil!
    end
    ```

!!! tip "Correct: Always guard against nil and use request/event pattern"
    ```lua
    function ns:GetSpellInfoSafe(spellID, callback)
        local info = C_Spell.GetSpellInfo(spellID)
        if info then
            callback(info)
            return
        end

        -- Data not cached yet — request it and wait
        C_Spell.RequestLoadSpellData(spellID)

        local frame = CreateFrame("Frame")
        frame:RegisterEvent("SPELL_DATA_LOAD_RESULT")
        frame:SetScript("OnEvent", function(self, event, loadedSpellID, success)
            if loadedSpellID == spellID then
                self:UnregisterEvent("SPELL_DATA_LOAD_RESULT")
                if success then
                    callback(C_Spell.GetSpellInfo(spellID))
                end
            end
        end)
    end

    -- Usage
    ns:GetSpellInfoSafe(12345, function(info)
        print("Spell: " .. info.name)
    end)
    ```

APIs that commonly return nil on first call:

| API | Request Function | Callback Event |
|---|---|---|
| `C_Spell.GetSpellInfo()` | `C_Spell.RequestLoadSpellData()` | `SPELL_DATA_LOAD_RESULT` |
| `C_Item.GetItemInfo()` | `C_Item.RequestLoadItemData()` | `ITEM_DATA_LOAD_RESULT` |
| `C_Item.GetItemIconByID()` | `C_Item.RequestLoadItemDataByID()` | `ITEM_DATA_LOAD_RESULT` |

---

## 14. Using pairs() for Ordered Iteration

`pairs()` iterates in **undefined order** — it can differ between calls, between sessions, and between Lua versions. AI models use it everywhere, even when order matters.

!!! danger "Wrong: pairs() when order matters"
    ```lua
    local tabs = {
        [1] = { name = "General", icon = "inv_misc_gear" },
        [2] = { name = "Combat", icon = "ability_warrior_charge" },
        [3] = { name = "Loot", icon = "inv_misc_coin_01" },
    }

    -- pairs() may iterate as 2, 1, 3 or 3, 1, 2 — order is NOT guaranteed!
    for index, tab in pairs(tabs) do
        CreateTab(index, tab.name, tab.icon)  -- Tabs appear in random order!
    end
    ```

!!! tip "Correct: ipairs() for sequential integer keys"
    ```lua
    local tabs = {
        { name = "General", icon = "inv_misc_gear" },
        { name = "Combat", icon = "ability_warrior_charge" },
        { name = "Loot", icon = "inv_misc_coin_01" },
    }

    -- ipairs() guarantees 1, 2, 3 order
    for index, tab in ipairs(tabs) do
        CreateTab(index, tab.name, tab.icon)  -- Always correct order
    end
    ```

Quick rule:

- **`ipairs()`** — Sequential integer keys starting at 1. Guaranteed order. Stops at first `nil` gap.
- **`pairs()`** — All keys (string, number, mixed). **No order guarantee.** Use only when order doesn't matter (e.g., lookup tables, deduplication).

---

## 15. Frame Naming Conflicts

Named frames become **globals**. Using common names like `"MainFrame"` or `"SettingsPanel"` will collide with other addons.

!!! danger "Wrong: Generic global frame names"
    ```lua
    -- These become _G["MainFrame"], _G["SettingsPanel"], etc.
    -- Almost guaranteed to conflict with another addon
    local frame = CreateFrame("Frame", "MainFrame", UIParent)
    local settings = CreateFrame("Frame", "SettingsPanel", UIParent)
    local button = CreateFrame("Button", "CloseButton", UIParent)
    local tooltip = CreateFrame("GameTooltip", "Tooltip", UIParent)
    ```

!!! tip "Correct: Prefix with your addon name, or use nil"
    ```lua
    -- Option 1: Prefix frame names with your addon name
    local frame = CreateFrame("Frame", "MyAddonMainFrame", UIParent)
    local settings = CreateFrame("Frame", "MyAddonSettingsPanel", UIParent)

    -- Option 2: Use nil for frames that don't need global names
    -- (Most frames don't! Only name them if other addons or macros need access)
    local frame = CreateFrame("Frame", nil, UIParent)
    local button = CreateFrame("Button", nil, frame)

    -- Option 3: If you need the frame in XML or macro /click, still prefix
    local btn = CreateFrame("Button", "MyAddonToggleButton", UIParent, "SecureActionButtonTemplate")
    ```

!!! tip
    Ask yourself: "Does anything outside my addon need to find this frame by name?" If no, pass `nil` as the name. This avoids global pollution entirely and is the preferred modern approach.

---

## 16. Forgetting the Combat Lockdown Queue Pattern

This is so common it deserves its own section beyond pitfall #7. AI generates code that modifies secure frames in response to events that can fire during combat, with no lockdown check.

!!! danger "Wrong: Modifying secure frames in event handlers without combat check"
    ```lua
    function events:BAG_UPDATE()
        -- BAG_UPDATE fires during combat!
        MyAddonSecureButton:SetAttribute("macrotext", "/use " .. ns:GetBestPotion())
        MyAddonSecureButton:Show()
    end

    function events:PLAYER_TARGET_CHANGED()
        -- Also fires during combat!
        if UnitExists("target") then
            MyAddonTargetFrame:Show()     -- Fails silently during combat
        else
            MyAddonTargetFrame:Hide()     -- Fails silently during combat
        end
    end
    ```

!!! tip "Correct: Queue pattern for any code path that touches secure frames"
    ```lua
    local addonName, ns = ...

    ns.combatQueue = {}

    local combatFrame = CreateFrame("Frame")
    combatFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
    combatFrame:SetScript("OnEvent", function()
        for _, fn in ipairs(ns.combatQueue) do
            fn()
        end
        wipe(ns.combatQueue)
    end)

    function ns:RunSecure(fn)
        if InCombatLockdown() then
            table.insert(ns.combatQueue, fn)
        else
            fn()
        end
    end

    -- Now use it everywhere you touch secure frames
    function events:BAG_UPDATE()
        ns:RunSecure(function()
            MyAddonSecureButton:SetAttribute("macrotext", "/use " .. ns:GetBestPotion())
            MyAddonSecureButton:Show()
        end)
    end

    function events:PLAYER_TARGET_CHANGED()
        ns:RunSecure(function()
            if UnitExists("target") then
                MyAddonTargetFrame:Show()
            else
                MyAddonTargetFrame:Hide()
            end
        end)
    end
    ```

---

## 17. Wrong Slash Command Registration

Slash commands require a specific naming convention. The global variable name must match the `SlashCmdList` key exactly. AI frequently gets this wrong.

!!! danger "Wrong: Mismatched names and missing globals"
    ```lua
    -- Wrong: SLASH_ global doesn't match SlashCmdList key
    SLASH_MYADDON1 = "/myaddon"
    SlashCmdList["MyAddon"] = function(msg)  -- Key must be "MYADDON" not "MyAddon"!
        print("Hello!")
    end

    -- Wrong: Using local for SLASH_ variables (they MUST be global)
    local SLASH_TEST1 = "/test"
    SlashCmdList["TEST"] = function(msg) end  -- Won't work, local isn't in _G

    -- Wrong: Registering in ADDON_LOADED (can fail if SlashCmdList isn't ready)
    function events:ADDON_LOADED(addon)
        if addon ~= addonName then return end
        SLASH_MA1 = "/ma"
        SlashCmdList["MA"] = function(msg) end
    end
    ```

!!! tip "Correct: Matching names at file scope"
    ```lua
    local addonName, ns = ...

    -- The SLASH_ variable suffix must EXACTLY match the SlashCmdList key
    -- The key must be UPPERCASE
    SLASH_MYADDON1 = "/myaddon"
    SLASH_MYADDON2 = "/ma"  -- Optional: multiple aliases

    SlashCmdList["MYADDON"] = function(msg)
        local cmd, rest = strsplit(" ", msg, 2)
        cmd = strlower(cmd or "")

        if cmd == "config" or cmd == "options" then
            ns:OpenOptions()
        elseif cmd == "reset" then
            ns:ResetSettings()
        else
            print("|cff00ff00MyAddon|r commands: /myaddon config | reset")
        end
    end
    ```

The rules are:

1. `SLASH_XXXN` must be a **global** variable (no `local`)
2. `XXX` must **exactly match** the key in `SlashCmdList["XXX"]`
3. `XXX` must be **UPPERCASE**
4. `N` is a number (1, 2, 3...) for multiple aliases
5. Register at **file scope**, not inside an event handler

---

## Summary Cheat Sheet

| Pitfall | What AI Generates | What You Should Use |
|---|---|---|
| Spell info | `GetSpellInfo()` → multi-return | `C_Spell.GetSpellInfo()` → table |
| Item info | `GetItemInfo()` → multi-return | `C_Item.GetItemInfo()` → table |
| SavedVariables | Init at file scope | Init inside `ADDON_LOADED` |
| Namespace | Bare globals | `local addonName, ns = ...` |
| Module loading | `require()` / `dofile()` | `.toc` file ordering + `ns` |
| Cancellable timer | `C_Timer.After()` | `C_Timer.NewTimer()` |
| Combat frames | No lockdown check | `InCombatLockdown()` + queue |
| Event dispatch | `if/elseif` chain | Table dispatch |
| OnUpdate tables | `local t = {}` per frame | Pre-allocate, reuse with `wipe()` |
| Lua version | 5.2+ syntax (`goto`, `//`, `&`) | Lua 5.1 + `bit` library |
| SetTexCoord | `left, top, right, bottom` | `left, right, top, bottom` |
| Combat log | CLEU / `CombatLogGetCurrentEventInfo()` | Removed in 12.0 — use `UNIT_HEALTH`, `UNIT_AURA`, etc. |
| Async APIs | Assume non-nil return | Nil-check + request/event |
| Ordered iteration | `pairs()` | `ipairs()` |
| Frame names | `"MainFrame"` | `"MyAddonMainFrame"` or `nil` |
| Combat queue | Direct secure frame ops | `RunSecure()` wrapper |
| Slash commands | Mismatched names | `SLASH_XXX1` matches `SlashCmdList["XXX"]` |
