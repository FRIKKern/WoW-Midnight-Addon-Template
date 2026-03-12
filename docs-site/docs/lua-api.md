# Lua API & C_ Namespaces

WoW's addon API is extensive — 260+ C_ namespaces plus hundreds of global functions. This page covers the essentials you'll use in most addons.

> **Canonical reference:** [warcraft.wiki.gg/wiki/World_of_Warcraft_API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API)

---

## Lua Sandbox

WoW embeds **Lua 5.1** in a sandboxed environment. Not all standard libraries are available.

### Available Libraries

| Library | Notes |
|---------|-------|
| **Base functions** | `pairs`, `ipairs`, `next`, `select`, `type`, `tostring`, `tonumber`, `unpack`, `pcall`, `xpcall`, `error`, `assert`, `rawget`, `rawset`, `rawequal`, `setmetatable`, `getmetatable` |
| **string** | All standard functions (`string.find`, `string.gsub`, `string.format`, etc.) |
| **table** | `table.insert`, `table.remove`, `table.sort`, `table.concat`, `table.getn` |
| **math** | All standard functions — **trig functions use degrees, not radians** |
| **bit** | `bit.band`, `bit.bor`, `bit.bxor`, `bit.bnot`, `bit.lshift`, `bit.rshift` |
| **coroutine** | Full coroutine support |

### Blocked / Restricted

| Item | Status |
|------|--------|
| `os`, `io`, `debug` | **Removed entirely** — no filesystem or OS access |
| `dofile`, `require` | **Removed** — use TOC file loading order instead |
| `loadstring` | **Returns tainted code** — the resulting function is insecure and cannot call protected APIs |
| `setfenv` / `getfenv` | **Taint risk** — using these on a function marks it (and its callers) as tainted |

---

## WoW Global Functions

These are the most commonly used global functions provided by the WoW client.

### Frame Creation

| Function | Description |
|----------|-------------|
| `CreateFrame(type [, name, parent, template, id])` | Create a UI frame. `type` is a string like `"Frame"`, `"Button"`, `"EditBox"`. Returns the new frame. |

```lua
-- Create a simple frame
local frame = CreateFrame("Frame", "MyAddonFrame", UIParent, "BackdropTemplate")
frame:SetSize(200, 100)
frame:SetPoint("CENTER")

-- Create a button from a template
local btn = CreateFrame("Button", nil, frame, "UIPanelButtonTemplate")
btn:SetSize(120, 25)
btn:SetText("Click Me")
btn:SetPoint("CENTER")
btn:SetScript("OnClick", function(self)
    print("Button clicked!")
end)
```

### Time

| Function | Description |
|----------|-------------|
| `GetTime()` | Returns game time in seconds as a float (high precision) |
| `time()` | Returns server time as a Unix timestamp (integer) |
| `date(fmt [, time])` | Format a time value (like `os.date` in standard Lua) |

### Output

| Function | Description |
|----------|-------------|
| `print(...)` | Print to the default chat frame. Concatenates args with spaces. |
| `format(fmt, ...)` | Alias for `string.format`. Format a string with substitutions. |

### String Utilities

| Function | Description |
|----------|-------------|
| `strsplit(delimiter, str [, pieces])` | Split a string. Returns multiple values (not a table). |
| `strjoin(delimiter, ...)` | Join values with a delimiter. |
| `strtrim(str [, chars])` | Trim whitespace (or specified chars) from both ends. |
| `strmatch(str, pattern)` | Alias for `string.match`. |
| `strfind(str, pattern [, init [, plain]])` | Alias for `string.find`. |

```lua
-- strsplit returns multiple values, not a table
local cmd, arg1, rest = strsplit(" ", msg, 3)

-- strjoin
local path = strjoin("/", "Interface", "AddOns", addonName)

-- strtrim
local cleaned = strtrim("  hello  ") -- "hello"
```

### Table Utilities

| Function | Description |
|----------|-------------|
| `tinsert(t, [pos,] value)` | Alias for `table.insert` |
| `tremove(t [, pos])` | Alias for `table.remove` |
| `wipe(t)` | Remove all keys from a table (reuses memory — faster than `t = {}`) |
| `tContains(t, value)` | Returns `true` if `value` is in array `t` |
| `tInvert(t)` | Returns a new table with keys and values swapped |
| `CopyTable(t)` | Deep copy a table (recursively copies subtables) |

```lua
-- wipe reuses the table allocation
local cache = {}
-- ... fill cache ...
wipe(cache) -- clears it without creating garbage

-- tContains for membership checks
local validTypes = {"warrior", "mage", "priest"}
if tContains(validTypes, playerClass) then
    -- handle
end

-- tInvert for reverse lookups
local lookup = tInvert(validTypes)
-- lookup = { warrior = 1, mage = 2, priest = 3 }

-- CopyTable for safe defaults
local db = CopyTable(defaults)
```

### OOP Utilities

| Function | Description |
|----------|-------------|
| `Mixin(target, ...)` | Copy all fields from mixin tables into `target`. Returns `target`. |
| `CreateFromMixins(...)` | Create a new table with fields from all provided mixins. |

```lua
-- Define a mixin
local HealthBarMixin = {}

function HealthBarMixin:Init(unit)
    self.unit = unit
    self:Update()
end

function HealthBarMixin:Update()
    local hp = UnitHealth(self.unit)
    local max = UnitHealthMax(self.unit)
    self.bar:SetValue(hp / max)
end

-- Create an instance
local healthBar = CreateFromMixins(HealthBarMixin)
healthBar.bar = CreateFrame("StatusBar", nil, UIParent)
healthBar:Init("player")

-- Or mixin into an existing frame
local frame = CreateFrame("Frame")
Mixin(frame, HealthBarMixin)
frame.bar = frame:CreateTexture()
frame:Init("target")
```

### Hooking & Security

| Function | Description |
|----------|-------------|
| `hooksecurefunc([table,] "name", hook)` | Post-hook a function without causing taint. Hook runs after the original. |
| `securecall(func, ...)` | Call a function in a secure context (rarely needed by addons). |
| `issecurevariable([table,] "name")` | Check if a variable is secure (untainted). Returns `isSecure, taintedBy`. |
| `InCombatLockdown()` | Returns `true` if the player is in combat lockdown (protected actions blocked). |

### Debug

| Function | Description |
|----------|-------------|
| `debugstack([thread,] [start [, count1 [, count2]]])` | Returns a stack trace string. Useful for debugging. |
| `debugprofilestart()` | Start a high-precision timer. |
| `debugprofilestop()` | Returns milliseconds elapsed since `debugprofilestart()`. |

```lua
-- Profile a function
debugprofilestart()
MyExpensiveFunction()
local elapsed = debugprofilestop()
print(format("Took %.2f ms", elapsed))
```

---

## C_ Namespace APIs

WoW's modern API is organized into 260+ `C_` namespaces. These are the ones you'll use most often.

### C_Timer

Non-blocking timers. **Always prefer these over `OnUpdate` for simple delays.**

| Function | Description |
|----------|-------------|
| `C_Timer.After(seconds, callback)` | One-shot timer. No cancel handle. |
| `C_Timer.NewTimer(seconds, callback)` | One-shot timer. Returns a handle with `:Cancel()`. |
| `C_Timer.NewTicker(seconds, callback [, iterations])` | Repeating timer. Returns a handle with `:Cancel()`. |

```lua
-- Simple delay
C_Timer.After(2, function()
    print("2 seconds later!")
end)

-- Cancellable one-shot
local timer = C_Timer.NewTimer(5, function()
    print("This may never fire")
end)
timer:Cancel() -- cancel before it fires

-- Repeating ticker
local ticker = C_Timer.NewTicker(1, function()
    print("Every second")
end)

-- Stop it later
C_Timer.After(10, function()
    ticker:Cancel()
end)

-- Limited iterations (fires 5 times then stops)
C_Timer.NewTicker(0.5, function(self)
    print("Tick", self._remainingIterations)
end, 5)
```

### C_ChatInfo — Addon Communication

Send messages between addon instances across the network.

| Function | Description |
|----------|-------------|
| `C_ChatInfo.RegisterAddonMessagePrefix(prefix)` | Register a prefix (max 16 chars). Must register before receiving. |
| `C_ChatInfo.SendAddonMessage(prefix, message, chatType [, target])` | Send a message. `chatType`: `"PARTY"`, `"RAID"`, `"GUILD"`, `"WHISPER"`. |

**Limits:** Max 255 bytes per message. Throttled to ~10 messages/second. Exceeding the throttle queues messages.

```lua
local PREFIX = "MyAddon"
C_ChatInfo.RegisterAddonMessagePrefix(PREFIX)

-- Send a message to the raid
C_ChatInfo.SendAddonMessage(PREFIX, "PULL:10", "RAID")

-- Receive messages via event
function eventHandlers:CHAT_MSG_ADDON(prefix, message, channel, sender)
    if prefix ~= PREFIX then return end

    local cmd, data = strsplit(":", message, 2)
    if cmd == "PULL" then
        local countdown = tonumber(data)
        print(sender .. " is pulling in " .. countdown .. " seconds!")
    end
end
```

### C_Map

Map and zone information.

| Function | Description |
|----------|-------------|
| `C_Map.GetBestMapForUnit(unitToken)` | Returns the most specific mapID for a unit's location. |
| `C_Map.GetMapInfo(mapID)` | Returns map info table: `{ mapID, name, mapType, parentMapID }`. |
| `C_Map.GetPlayerMapPosition(mapID, unitToken)` | Returns a `Vector2D` with `:GetXY()` for player position (0–1 range). |
| `C_Map.GetAreaInfo(areaID)` | Returns the name of a map area. |

```lua
local mapID = C_Map.GetBestMapForUnit("player")
if mapID then
    local info = C_Map.GetMapInfo(mapID)
    print("Current zone:", info.name)

    local pos = C_Map.GetPlayerMapPosition(mapID, "player")
    if pos then
        local x, y = pos:GetXY()
        print(format("Position: %.1f%%, %.1f%%", x * 100, y * 100))
    end
end
```

### C_Item

Item data queries. Many functions are async — data may not be available immediately.

| Function | Description |
|----------|-------------|
| `C_Item.GetItemInfo(itemID)` | Returns item info (name, link, quality, etc.). May return nil if not cached. |
| `C_Item.GetItemNameByID(itemID)` | Returns item name. May require loading. |
| `C_Item.GetItemIconByID(itemID)` | Returns the icon texture path/ID. |
| `C_Item.RequestLoadItemDataByID(itemID)` | Request the server to load item data. Listen for `ITEM_DATA_LOAD_RESULT`. |
| `C_Item.IsItemDataCachedByID(itemID)` | Check if item data is already cached locally. |

```lua
-- Safe item info retrieval with async fallback
local function GetItemInfoAsync(itemID, callback)
    if C_Item.IsItemDataCachedByID(itemID) then
        callback(C_Item.GetItemInfo(itemID))
        return
    end

    -- Request load and wait for event
    local item = Item:CreateFromItemID(itemID)
    item:ContinueOnItemLoad(function()
        callback(C_Item.GetItemInfo(itemID))
    end)
end
```

### C_Spell

Spell information queries.

| Function | Description |
|----------|-------------|
| `C_Spell.GetSpellInfo(spellID)` | Returns spell info table with `name`, `iconID`, `castTime`, etc. |
| `C_Spell.GetSpellName(spellID)` | Returns the localized spell name. |
| `C_Spell.GetSpellCooldown(spellID)` | Returns cooldown info: `startTime`, `duration`, `isEnabled`, `modRate`. |
| `C_Spell.GetSpellTexture(spellID)` | Returns the spell's icon texture. |

```lua
local info = C_Spell.GetSpellInfo(12345)
if info then
    print(info.name, "cast time:", info.castTime / 1000, "sec")
end
```

### C_AddOns

Addon management functions.

| Function | Description |
|----------|-------------|
| `C_AddOns.GetNumAddOns()` | Returns the total number of addons. |
| `C_AddOns.GetAddOnInfo(index)` | Returns `name, title, notes, loadable, reason, security, newVersion`. |
| `C_AddOns.IsAddOnLoaded(indexOrName)` | Returns `loaded, finished`. |
| `C_AddOns.GetAddOnMetadata(addon, field)` | Read a TOC field value (e.g., `"Version"`, `"X-Category"`). |
| `C_AddOns.LoadAddOn(indexOrName)` | Load a LoadOnDemand addon at runtime. |

```lua
-- Check if another addon is loaded
local loaded = C_AddOns.IsAddOnLoaded("Details")
if loaded then
    print("Details is loaded, enabling integration")
end

-- Read your own version from TOC
local version = C_AddOns.GetAddOnMetadata(addonName, "Version")
print(addonName, "v" .. version)
```

### C_Container

Bag and inventory operations.

| Function | Description |
|----------|-------------|
| `C_Container.GetContainerNumSlots(bagID)` | Number of slots in a bag (0 = backpack, 1–4 = bags). |
| `C_Container.GetContainerItemInfo(bagID, slot)` | Returns item info table for a bag slot. |
| `C_Container.GetContainerItemID(bagID, slot)` | Returns the itemID in a bag slot. |
| `C_Container.GetContainerNumFreeSlots(bagID)` | Returns free slots and bag type. |
| `C_Container.UseContainerItem(bagID, slot)` | Use/equip an item (protected in combat). |

```lua
-- Count free bag slots
local freeSlots = 0
for bag = 0, 4 do
    local free = C_Container.GetContainerNumFreeSlots(bag)
    freeSlots = freeSlots + free
end
print("Free bag slots:", freeSlots)

-- Find an item in bags
local function FindItemInBags(targetItemID)
    for bag = 0, 4 do
        for slot = 1, C_Container.GetContainerNumSlots(bag) do
            local itemID = C_Container.GetContainerItemID(bag, slot)
            if itemID == targetItemID then
                return bag, slot
            end
        end
    end
    return nil
end
```

### C_UnitAuras

Buff and debuff queries (modern replacement for UnitAura).

| Function | Description |
|----------|-------------|
| `C_UnitAuras.GetAuraDataByIndex(unit, index [, filter])` | Get aura data by index. Returns aura info table or nil. |
| `C_UnitAuras.GetAuraDataByAuraInstanceID(unit, auraInstanceID)` | Get aura data by its unique instance ID (preferred). |
| `C_UnitAuras.GetAuraDataBySpellName(unit, spellName [, filter])` | Get aura data by spell name. |
| `C_UnitAuras.GetPlayerAuraBySpellID(spellID)` | Shortcut for checking player buffs/debuffs. |

Filters: `"HELPFUL"` (buffs), `"HARMFUL"` (debuffs), `"PLAYER"` (cast by player), `"CANCELABLE"`. Combine with space: `"HARMFUL|PLAYER"`.

```lua
-- Check if player has a specific buff
local aura = C_UnitAuras.GetPlayerAuraBySpellID(1459) -- Arcane Intellect
if aura then
    print("Arcane Intellect active, expires in", aura.expirationTime - GetTime(), "sec")
end

-- Iterate all debuffs on target
local i = 1
while true do
    local aura = C_UnitAuras.GetAuraDataByIndex("target", i, "HARMFUL")
    if not aura then break end
    print(aura.name, "stacks:", aura.applications, "from:", aura.sourceUnit or "unknown")
    i = i + 1
end
```

### C_EncodingUtil

Data encoding and compression utilities (added in 11.1.5).

| Function | Description |
|----------|-------------|
| `C_EncodingUtil.MakeHardenedTable(t)` | Creates a tamper-resistant table. |

> **Note:** This namespace is relatively new. Check the [wiki](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) for the latest available functions.

---

## Slash Command Registration

Register custom slash commands for your addon.

```lua
local addonName, ns = ...

-- Register one or more slash command aliases
SLASH_MYADDON1 = "/myaddon"
SLASH_MYADDON2 = "/ma"

SlashCmdList["MYADDON"] = function(msg)
    local cmd, rest = strsplit(" ", msg, 2)
    cmd = (cmd or ""):lower()

    if cmd == "config" or cmd == "options" then
        ns:OpenConfig()
    elseif cmd == "reset" then
        ns:ResetPosition()
        print("|cff00ff00MyAddon:|r Position reset.")
    elseif cmd == "toggle" then
        ns.db.enabled = not ns.db.enabled
        print("|cff00ff00MyAddon:|r " .. (ns.db.enabled and "Enabled" or "Disabled"))
    elseif cmd == "debug" then
        ns.db.debug = not ns.db.debug
        print("|cff00ff00MyAddon:|r Debug " .. (ns.db.debug and "ON" or "OFF"))
    else
        print("|cff00ff00MyAddon|r commands:")
        print("  /ma config  — Open settings")
        print("  /ma toggle  — Enable/disable")
        print("  /ma reset   — Reset position")
        print("  /ma debug   — Toggle debug mode")
    end
end
```

**Rules:**
- The global name must be `SLASH_COMMANDNAME1` (the number is the alias index)
- The key in `SlashCmdList` must match the `COMMANDNAME` part (without `SLASH_` or the number)
- Slash commands are case-insensitive by the game client
- The `msg` argument is everything after the slash command itself

---

## hooksecurefunc Pattern

Post-hook any function without causing taint. The hook runs **after** the original function.

```lua
-- Hook a global function
hooksecurefunc("TargetFrame_Update", function(self)
    -- Runs after every TargetFrame_Update call
    -- self is the same first argument passed to the original
    print("Target frame updated")
end)

-- Hook a method on a specific object
hooksecurefunc(GameTooltip, "SetUnitBuff", function(self, unit, index)
    -- Runs after GameTooltip:SetUnitBuff()
    -- Add extra info to tooltip
    self:AddLine("Custom addon info here", 0.5, 0.8, 1.0)
    self:Show() -- refresh tooltip height
end)

-- Hook ChatFrame to intercept messages
hooksecurefunc(ChatFrame1, "AddMessage", function(self, text, ...)
    -- Runs after every chat message is added
    if text and text:find("pattern") then
        -- do something
    end
end)
```

**Limitations:**

- Cannot **prevent** the original function from running
- Cannot **modify** the return value of the original
- Cannot **unhook** — once hooked, it's hooked for the session
- Cannot change the **arguments** the original receives
- The hook itself is tainted (addon code), but the original remains secure

---

## Addon Communication

Send and receive messages between addon instances on different clients.

### Setup

```lua
local addonName, ns = ...
local PREFIX = "MyAddon" -- max 16 characters

-- Must register before you can receive
C_ChatInfo.RegisterAddonMessagePrefix(PREFIX)
```

### Sending

```lua
-- chatType: "PARTY", "RAID", "GUILD", "WHISPER", "CHANNEL"
C_ChatInfo.SendAddonMessage(PREFIX, "HELLO:1.0", "PARTY")

-- Whisper to a specific player
C_ChatInfo.SendAddonMessage(PREFIX, "VERSION:1.0", "WHISPER", "PlayerName")

-- Channel (must know the channel number)
C_ChatInfo.SendAddonMessage(PREFIX, data, "CHANNEL", channelNumber)
```

### Receiving

```lua
function eventHandlers:CHAT_MSG_ADDON(prefix, message, channel, sender)
    if prefix ~= PREFIX then return end

    -- Ignore messages from self
    local playerName = UnitName("player")
    if sender == playerName or sender:find(playerName) then return end

    local cmd, data = strsplit(":", message, 2)
    if cmd == "HELLO" then
        -- Respond with our version
        C_ChatInfo.SendAddonMessage(PREFIX, "HELLO:" .. ns.version, channel)
    end
end
```

### Limits & Best Practices

| Constraint | Value |
|-----------|-------|
| Max prefix length | 16 characters |
| Max message length | 255 bytes |
| Send throttle | ~10 messages/second |
| Max registered prefixes (all addons) | 128 |

- **Serialize long data** by splitting across multiple messages with sequence numbers
- **Throttle outgoing messages** — exceeding the limit queues them (causes delay, not errors)
- **Use compact encodings** — every byte counts in the 255-byte limit
- **Prefix with addon version** so you can handle protocol changes

---

## Useful Utility Patterns

### Throttle Function

Prevent a function from running more than once per interval.

```lua
local function CreateThrottle(interval)
    local lastCall = 0
    return function(func, ...)
        local now = GetTime()
        if now - lastCall >= interval then
            lastCall = now
            return func(...)
        end
    end
end

-- Usage
local throttledUpdate = CreateThrottle(0.1) -- max 10/sec
frame:SetScript("OnUpdate", function(self, elapsed)
    throttledUpdate(function()
        -- expensive work here
    end)
end)
```

### Debounce Function

Delay execution until calls stop arriving for a specified duration.

```lua
local function CreateDebounce(delay)
    local timer = nil
    return function(func)
        if timer then
            timer:Cancel()
        end
        timer = C_Timer.NewTimer(delay, function()
            timer = nil
            func()
        end)
    end
end

-- Usage: only update after user stops typing for 0.3s
local debouncedSearch = CreateDebounce(0.3)
editBox:SetScript("OnTextChanged", function(self)
    local text = self:GetText()
    debouncedSearch(function()
        ns:PerformSearch(text)
    end)
end)
```

### Deep Table Comparison

```lua
local function DeepEqual(a, b)
    if type(a) ~= type(b) then return false end
    if type(a) ~= "table" then return a == b end

    for k, v in pairs(a) do
        if not DeepEqual(v, b[k]) then return false end
    end
    for k in pairs(b) do
        if a[k] == nil then return false end
    end
    return true
end
```

### Safe Callback Queue (Combat-Aware)

```lua
local pendingActions = {}

local function ExecuteOrQueue(action)
    if InCombatLockdown() then
        table.insert(pendingActions, action)
    else
        action()
    end
end

function eventHandlers:PLAYER_REGEN_ENABLED()
    for _, action in ipairs(pendingActions) do
        action()
    end
    wipe(pendingActions)
end

-- Usage
ExecuteOrQueue(function()
    -- This involves protected operations (e.g., SetAttribute)
    mySecureButton:SetAttribute("macrotext", "/cast Fireball")
end)
```

### Colored Print Helper

```lua
local function AddonPrint(...)
    local prefix = "|cff00ccff[MyAddon]|r "
    print(prefix, ...)
end

-- Color codes: |cffRRGGBB starts color, |r resets
-- Common colors:
--   |cffff0000  red         |cff00ff00  green
--   |cff00ccff  light blue  |cffffff00  yellow
--   |cffff8800  orange      |cffff00ff  pink
```

---

## Next Steps

- [Event System](events.md) — Deep dive into WoW's event-driven architecture
- [Frame & Widget System](frames-widgets.md) — Building UI with frames, textures, and font strings
- [Security Model](security.md) — Understanding taint, combat lockdown, and protected functions
- [warcraft.wiki.gg API Reference](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) — Complete API listing
