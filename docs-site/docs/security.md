# Security Model & Best Practices

## Security Model Overview

World of Warcraft employs a strict addon security model designed to prevent botting, unfair gameplay advantages, and exploitation. Addons run in a **sandboxed Lua environment** with no access to the filesystem, network sockets, or operating system commands. Blizzard carefully controls which API functions addons can call and under what circumstances, ensuring that addons enhance the user interface without automating gameplay.

Understanding these restrictions is essential for addon development — violating them results in blocked actions, tainted execution, or silent failures that can be difficult to debug.

## Protected Functions

Certain API functions directly affect gameplay and are classified as **protected**. These functions can only be called from secure (Blizzard) code, or from addon code in response to a hardware event (keypress or mouse click) while **not** in combat.

!!! danger "Protected Functions"
    The following functions are protected and cannot be called freely from addon code:

    | Function | Purpose |
    |---|---|
    | `CastSpellByName()` | Cast a spell by name |
    | `TargetUnit()` | Target a specific unit |
    | `UseAction()` | Use an action bar slot |
    | `JumpOrAscendStart()` | Make the character jump |
    | `ToggleAutoRun()` | Toggle auto-run |
    | `StartAttack()` | Begin auto-attack |
    | `PetAttack()` | Command pet to attack |
    | `RunMacroText()` | Execute macro text |
    | `SetBinding()` | Set a key binding |

    These require a **hardware event** (real user input) and are fully blocked during combat lockdown.

## Taint System

The taint system is Blizzard's mechanism for tracking which code and data originate from addons versus the default UI. This prevents addon code from interfering with secure Blizzard UI operations.

For full technical details, see the [Secure Execution and Tainting](https://warcraft.wiki.gg/wiki/Secure_Execution_and_Tainting) article on the Warcraft wiki.

### Secure vs. Tainted Code

- **Secure code** — Blizzard's default UI code. It can call protected functions and modify secure frames.
- **Tainted code** — Any code originating from an addon. It cannot call protected functions during combat.

### How Taint Propagates

Taint spreads through execution like a virus:

1. Reading a **tainted variable** taints the current execution path
2. A tainted execution path **taints any variables it writes**
3. Any function called from a tainted path becomes tainted

```lua
-- Example: taint propagation
local secureValue = SomeSecureFunction()  -- secure
local taintedValue = MyAddonVariable       -- tainted (addon-created)

-- Reading taintedValue taints this execution path
local result = secureValue + taintedValue  -- result is now TAINTED

-- This would fail if called during combat from a tainted path
-- TargetUnit("player")  -- ERROR: blocked by taint
```

!!! warning "Irrevocable Taint"
    Calling `loadstring()` or `setfenv()` **irrevocably taints** the resulting code. There is no way to "untaint" code produced by these functions. Avoid them if your code needs to interact with secure frames.

### Checking Taint Status

Use `issecurevariable()` to inspect whether a variable is tainted:

```lua
local isTainted = not issecurevariable("SomeGlobalVariable")
if isTainted then
    print("SomeGlobalVariable is tainted by addon code")
end

-- Check a table field
local isTainted = not issecurevariable(SomeTable, "key")
```

## Combat Lockdown

During combat, Blizzard locks down the secure UI environment. This is the most common source of addon errors for new developers.

### Detecting Combat Lockdown

```lua
if InCombatLockdown() then
    print("In combat — protected actions are blocked!")
end
```

!!! danger "During Combat Lockdown, You Cannot:"
    - Call **any** protected function
    - Create new **secure frames** (frames using secure templates)
    - Modify **secure frame attributes** (e.g., changing a SecureActionButton's action)
    - Show or hide **secure frames**

### Pattern: Queue Actions for After Combat

The standard pattern is to queue desired actions during combat and execute them when combat ends via the `PLAYER_REGEN_ENABLED` event:

```lua
local addonName, ns = ...

ns.pendingActions = {}

local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_REGEN_ENABLED")

frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_ENABLED" then
        for _, action in ipairs(ns.pendingActions) do
            action()
        end
        wipe(ns.pendingActions)
    end
end)

-- Helper to run an action now, or queue it for after combat
function ns:RunOrQueue(action)
    if InCombatLockdown() then
        table.insert(self.pendingActions, action)
        print("Action queued — will execute after combat.")
    else
        action()
    end
end

-- Usage example: safely show a secure frame
ns:RunOrQueue(function()
    MySecureFrame:Show()
end)
```

!!! tip
    Always check `InCombatLockdown()` before performing any protected operation. The queue-and-execute pattern above is the idiomatic way to handle this.

## Hardware Event Requirements

Many gameplay-affecting actions require a **real hardware event** — an actual keypress or mouse click from the user. Synthetic events generated by code do not satisfy this requirement.

This means you **cannot** programmatically:

- Cast spells
- Target units
- Use items
- Interact with the world

To create buttons that perform these actions, use `SecureActionButtonTemplate`:

```lua
local btn = CreateFrame("Button", "MyAddonCastButton", UIParent, "SecureActionButtonTemplate")
btn:SetAttribute("type", "spell")
btn:SetAttribute("spell", "Fireball")
btn:SetSize(40, 40)
btn:SetPoint("CENTER")

-- The button's texture/visual setup
local tex = btn:CreateTexture(nil, "BACKGROUND")
tex:SetAllPoints()
tex:SetTexture(133230)  -- Fireball icon

-- When the user CLICKS this button, it casts Fireball
-- This works because it's a real hardware event routed through secure code
```

!!! warning
    You **must** set secure button attributes **outside** of combat. Attempting to call `SetAttribute()` on a secure frame during `InCombatLockdown()` will silently fail.

## What Addons Cannot Do

Understanding the boundaries of the sandbox helps avoid wasted effort and security violations.

!!! danger "Addon Restrictions"
    - **No filesystem access** — Cannot read, write, or list files (SavedVariables are the only persistence mechanism, managed by the game client)
    - **No network access** — Cannot open sockets, make HTTP requests, or communicate outside the game (addon messages are the only inter-player communication)
    - **No system commands** — Cannot execute OS commands, launch processes, or interact with the operating system
    - **No gameplay automation** — Cannot cast spells, target units, use items, or move the character without a real hardware event
    - **No secure frame modification in combat** — Cannot show, hide, or change attributes on secure frames during combat lockdown
    - **No cross-addon SavedVariables** — Cannot directly read another addon's saved data (use addon messages or public APIs instead)
    - **No raw memory access** — Cannot read or modify game memory; only the exposed Lua API is available

## Best Practices

### Performance

#### Avoid Table Creation in Frequently Called Code

Creating tables in `OnUpdate` or other high-frequency handlers generates garbage that triggers expensive garbage collection pauses.

```lua
-- BAD: creates a new table every frame
frame:SetScript("OnUpdate", function(self, elapsed)
    local data = { x = 0, y = 0, z = 0 }
    -- ... use data ...
end)

-- GOOD: reuse a table with wipe()
local data = { x = 0, y = 0, z = 0 }
frame:SetScript("OnUpdate", function(self, elapsed)
    wipe(data)
    data.x, data.y, data.z = 0, 0, 0
    -- ... use data ...
end)
```

#### Throttle OnUpdate Handlers

`OnUpdate` fires every frame (potentially 60–240+ times per second). Throttle it to avoid wasting CPU:

```lua
local THROTTLE_INTERVAL = 0.1  -- 10 updates per second max
local elapsed_acc = 0

frame:SetScript("OnUpdate", function(self, elapsed)
    elapsed_acc = elapsed_acc + elapsed
    if elapsed_acc < THROTTLE_INTERVAL then
        return
    end
    elapsed_acc = elapsed_acc - THROTTLE_INTERVAL

    -- Your actual update logic here
end)
```

!!! tip
    If you only need a one-time delayed action, use `C_Timer.After(seconds, callback)` instead of an `OnUpdate` handler. It's cleaner and automatically cleans up.

#### Cache Globals as Locals

Lua looks up global variables through a table hash every access. Caching frequently used globals as local upvalues is measurably faster in hot paths:

```lua
-- Cache at file scope (top of your .lua file)
local pairs = pairs
local ipairs = ipairs
local format = string.format
local tinsert = table.insert
local GetTime = GetTime
local UnitHealth = UnitHealth
local UnitHealthMax = UnitHealthMax
```

#### Prefer Events Over Polling

Events are always more efficient than polling with `OnUpdate`:

```lua
-- BAD: polling every frame
frame:SetScript("OnUpdate", function()
    local hp = UnitHealth("player")
    -- react to hp changes
end)

-- GOOD: event-driven
frame:RegisterEvent("UNIT_HEALTH")
frame:SetScript("OnEvent", function(self, event, unit)
    if unit == "player" then
        local hp = UnitHealth("player")
        -- react to hp changes
    end
end)
```

#### Unregister One-Time Events

If you only need an event once (e.g., initialization), unregister it immediately:

```lua
frame:RegisterEvent("PLAYER_LOGIN")
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_LOGIN" then
        self:UnregisterEvent("PLAYER_LOGIN")
        -- One-time initialization logic
    end
end)
```

### Code Organization

#### Event Dispatch Table

Use a dispatch table instead of long `if/elseif` chains for event handling:

```lua
local addonName, ns = ...

local frame = CreateFrame("Frame")

local events = {}

function events:PLAYER_LOGIN()
    print(addonName .. " loaded!")
end

function events:PLAYER_ENTERING_WORLD(isInitialLogin, isReloadingUI)
    if isInitialLogin then
        print("Welcome!")
    end
end

function events:ADDON_LOADED(loadedAddon)
    if loadedAddon == addonName then
        -- Initialize SavedVariables, etc.
    end
end

for event in pairs(events) do
    frame:RegisterEvent(event)
end

frame:SetScript("OnEvent", function(self, event, ...)
    events[event](self, ...)
end)
```

#### Mixin Pattern for Composition

Use mixins to share behavior across objects without deep inheritance chains:

```lua
-- Define a mixin
local HealthBarMixin = {}

function HealthBarMixin:UpdateHealth(unit)
    local hp = UnitHealth(unit)
    local maxHp = UnitHealthMax(unit)
    self.healthBar:SetValue(hp / maxHp)
end

function HealthBarMixin:SetHealthColor(r, g, b)
    self.healthBar:SetStatusBarColor(r, g, b)
end

-- Apply the mixin to a frame
local myFrame = CreateFrame("Frame", nil, UIParent)
Mixin(myFrame, HealthBarMixin)

-- Now myFrame has :UpdateHealth() and :SetHealthColor()
```

#### Namespace Encapsulation

Always use the addon namespace to encapsulate your code and avoid global pollution:

```lua
-- At the top of every file
local addonName, ns = ...

-- Everything goes in the namespace
ns.CONSTANTS = {
    MAX_ITEMS = 50,
    DEFAULT_COLOR = { r = 1, g = 1, b = 1 },
}

function ns:Initialize()
    -- Addon initialization
end

-- Local upvalues for performance-critical code
local MAX_ITEMS = ns.CONSTANTS.MAX_ITEMS
```

## Addon Communication Best Practices

Addons can communicate with other instances of the same addon (or cooperating addons) across players using addon messages.

### Registering a Prefix

Every addon message channel requires a prefix (max **16 characters**):

```lua
local PREFIX = "MyAddon"
C_ChatInfo.RegisterAddonMessagePrefix(PREFIX)
```

### Sending Messages

```lua
-- Send to party/raid/guild
C_ChatInfo.SendAddonMessage(PREFIX, "hello", "PARTY")
C_ChatInfo.SendAddonMessage(PREFIX, "sync:data", "RAID")
C_ChatInfo.SendAddonMessage(PREFIX, "version:1.2.3", "GUILD")

-- Send to a specific player (whisper)
C_ChatInfo.SendAddonMessage(PREFIX, "request:config", "WHISPER", "PlayerName")
```

!!! warning "Message Limits"
    - Maximum message size: **255 bytes** per message
    - Throttle: approximately **10 messages per second** before the server starts dropping them
    - Prefix: maximum **16 characters**

    Exceeding these limits causes messages to be silently dropped.

### Receiving Messages

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("CHAT_MSG_ADDON")
frame:SetScript("OnEvent", function(self, event, prefix, message, channel, sender)
    if prefix ~= PREFIX then return end

    -- Parse and handle the message
    local command, data = strsplit(":", message, 2)
    if command == "sync" then
        -- Handle sync data
    elseif command == "version" then
        -- Handle version check
    end
end)
```

### Batching and Compression for Heavy Communication

For syncing large datasets, batch your data and consider compression:

```lua
local addonName, ns = ...

-- Simple message batching
function ns:SendLargeData(data, channel)
    local serialized = ns:Serialize(data)  -- your serialization logic
    local chunks = {}

    -- Split into 250-byte chunks (leave room for header)
    for i = 1, #serialized, 250 do
        table.insert(chunks, serialized:sub(i, i + 249))
    end

    -- Send with a small delay between chunks to avoid throttle
    for i, chunk in ipairs(chunks) do
        C_Timer.After((i - 1) * 0.12, function()
            local header = format("CHUNK:%d:%d:", i, #chunks)
            C_ChatInfo.SendAddonMessage(PREFIX, header .. chunk, channel)
        end)
    end
end
```

!!! tip
    For serious data synchronization needs, consider using the **LibSerialize** and **LibDeflate** libraries available through the addon ecosystem, rather than writing your own serialization and compression.
