# Events System

## Event-Driven Architecture

World of Warcraft addons are entirely **event-driven**. There is no main loop you control. Instead, you register frames to listen for specific events, and the game calls your handler when those events fire. This is the fundamental pattern behind every addon.

The game client dispatches hundreds of events covering combat, UI changes, chat messages, inventory updates, zone transitions, and more. Your addon picks the events it cares about and ignores the rest.

## Core Mechanism

Every event handler is attached to a `Frame`:

```lua
-- Create a frame to listen for events
local frame = CreateFrame("Frame")

-- Register for specific events
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("PLAYER_ENTERING_WORLD")

-- Set the event handler
frame:SetScript("OnEvent", function(self, event, ...)
    -- event is the event name (string)
    -- ... contains event-specific arguments
    print("Event fired:", event)
end)

-- Stop listening for an event
frame:UnregisterEvent("PLAYER_LOGIN")

-- For unit-specific events, filter to specific units
frame:RegisterUnitEvent("UNIT_HEALTH", "player")
frame:RegisterUnitEvent("UNIT_AURA", "target", "focus")
```

:::tip
`RegisterUnitEvent` is more efficient than `RegisterEvent` for unit events because the client only fires the handler for the units you specify, rather than every unit in range.
:::

## Table Dispatch Pattern

For addons that handle multiple events, use a **table dispatch** pattern instead of long `if/elseif` chains. This gives O(1) lookup and keeps your code clean:

```lua
local MyAddon = {}
local frame = CreateFrame("Frame")

-- Define handlers in a table
local eventHandlers = {
    ADDON_LOADED = function(self, addonName)
        if addonName ~= "MyAddon" then return end
        MyAddon:Initialize()
        frame:UnregisterEvent("ADDON_LOADED")
    end,

    PLAYER_LOGIN = function(self)
        MyAddon:SetupUI()
    end,

    PLAYER_ENTERING_WORLD = function(self, isInitialLogin, isReloadingUI)
        if isInitialLogin or isReloadingUI then
            MyAddon:RefreshState()
        end
    end,

    COMBAT_LOG_EVENT_UNFILTERED = function(self)
        local timestamp, subevent, _, sourceGUID = CombatLogGetCurrentEventInfo()
        if sourceGUID == UnitGUID("player") then
            MyAddon:HandleCombatEvent(subevent)
        end
    end,

    PLAYER_REGEN_ENABLED = function(self)
        MyAddon:ProcessCombatQueue()
    end,
}

-- Single dispatcher
frame:SetScript("OnEvent", function(self, event, ...)
    local handler = eventHandlers[event]
    if handler then
        handler(self, ...)
    end
end)

-- Register all events from the table
for event in pairs(eventHandlers) do
    frame:RegisterEvent(event)
end
```

:::note
This pattern scales well. Adding a new event means adding one table entry — no cascading `elseif` blocks to maintain.
:::

## Events vs OnUpdate

| | Events | OnUpdate |
|---|---|---|
| **When it fires** | On specific game occurrences | Every rendered frame (~60+ times/sec) |
| **Performance** | Efficient — only fires when needed | Expensive — runs constantly |
| **Use case** | Reacting to game state changes | Animations, continuous polling |
| **Typical overhead** | Near zero when idle | Always consuming CPU |

**Prefer events whenever possible.** If you need a delayed action, use `C_Timer.After` or `C_Timer.NewTicker` instead of polling with `OnUpdate`:

```lua
-- Bad: polling with OnUpdate
local elapsed = 0
frame:SetScript("OnUpdate", function(self, delta)
    elapsed = elapsed + delta
    if elapsed >= 2.0 then
        DoSomething()
        self:SetScript("OnUpdate", nil)
    end
end)

-- Good: use C_Timer
C_Timer.After(2.0, function()
    DoSomething()
end)
```

## Complete Event Reference

### Addon Lifecycle

These events fire in a specific order during login and reload. Understanding this sequence is critical for proper addon initialization.

| Event | Description | Arguments |
|---|---|---|
| `ADDON_LOADED` | An addon's saved variables have been loaded | `addonName` |
| `VARIABLES_LOADED` | All account-wide saved variables are available | — |
| `PLAYER_LOGIN` | Player has entered the world for the first time this session | — |
| `PLAYER_ENTERING_WORLD` | Player enters the world, changes zones via portals, or reloads UI | `isInitialLogin`, `isReloadingUI` |
| `PLAYER_LOGOUT` | Player is logging out or quitting | — |
| `ADDONS_UNLOADING` | Addons are about to be unloaded (12.0+, use for final saves) | — |
| `SPELLS_CHANGED` | Player's spell book has been updated | — |

#### Loading Sequence

```
ADDON_LOADED (fires per addon)
    → VARIABLES_LOADED
        → PLAYER_LOGIN
            → PLAYER_ENTERING_WORLD (isInitialLogin = true)
                → SPELLS_CHANGED
```

:::warning
Do not access spell or talent data before `SPELLS_CHANGED` fires. APIs like `GetSpellInfo` may return nil during early loading events.
:::

### Combat

| Event | Description | Arguments |
|---|---|---|
| `COMBAT_LOG_EVENT_UNFILTERED` | Any combat log entry (use `CombatLogGetCurrentEventInfo()` to read) | — |
| `PLAYER_REGEN_DISABLED` | Player entered combat | — |
| `PLAYER_REGEN_ENABLED` | Player left combat | — |
| `ENCOUNTER_START` | Boss encounter started | `encounterID`, `name`, `difficultyID`, `groupSize` |
| `ENCOUNTER_END` | Boss encounter ended | `encounterID`, `name`, `difficultyID`, `groupSize`, `success` |
| `PLAYER_DEAD` | Player has died | — |
| `PLAYER_ALIVE` | Player is alive (after releasing or resurrecting) | — |

:::tip
`COMBAT_LOG_EVENT_UNFILTERED` passes no arguments directly. Always call `CombatLogGetCurrentEventInfo()` inside the handler to get the combat log data.
:::

### Units

| Event | Description | Arguments |
|---|---|---|
| `UNIT_HEALTH` | A unit's health changed | `unitTarget` |
| `UNIT_AURA` | Buffs/debuffs on a unit changed | `unitTarget`, `updateInfo` |
| `UNIT_POWER_UPDATE` | A unit's power (mana, rage, energy, etc.) changed | `unitTarget`, `powerType` |
| `PLAYER_TARGET_CHANGED` | Player's target changed | — |
| `UPDATE_MOUSEOVER_UNIT` | Mouseover unit changed | — |

### Spells & Casting

| Event | Description | Arguments |
|---|---|---|
| `UNIT_SPELLCAST_START` | A unit began casting | `unitTarget`, `castGUID`, `spellID` |
| `UNIT_SPELLCAST_STOP` | A unit's cast was removed (any reason) | `unitTarget`, `castGUID`, `spellID` |
| `UNIT_SPELLCAST_SUCCEEDED` | A unit's cast completed successfully | `unitTarget`, `castGUID`, `spellID` |
| `UNIT_SPELLCAST_FAILED` | A unit's cast failed | `unitTarget`, `castGUID`, `spellID` |
| `UNIT_SPELLCAST_INTERRUPTED` | A unit's cast was interrupted | `unitTarget`, `castGUID`, `spellID` |
| `UNIT_SPELLCAST_CHANNEL_START` | A unit began channeling | `unitTarget`, `castGUID`, `spellID` |
| `UNIT_SPELLCAST_CHANNEL_STOP` | A unit stopped channeling | `unitTarget`, `castGUID`, `spellID` |
| `SPELL_UPDATE_COOLDOWN` | A spell cooldown changed | — |
| `SPELLS_CHANGED` | The player's spellbook was updated | — |

### Chat Messages

| Event | Description | Key Arguments |
|---|---|---|
| `CHAT_MSG_SAY` | /say message | `text`, `playerName`, `languageName`, ... |
| `CHAT_MSG_WHISPER` | Incoming whisper | `text`, `playerName`, ... |
| `CHAT_MSG_PARTY` | Party chat | `text`, `playerName`, ... |
| `CHAT_MSG_RAID` | Raid chat | `text`, `playerName`, ... |
| `CHAT_MSG_GUILD` | Guild chat | `text`, `playerName`, ... |
| `CHAT_MSG_CHANNEL` | Custom channel message | `text`, `playerName`, ..., `channelNumber`, `channelName` |
| `CHAT_MSG_SYSTEM` | System messages | `text` |
| `CHAT_MSG_ADDON` | Hidden addon-to-addon communication | `prefix`, `text`, `channel`, `sender` |

### Group & Raid

| Event | Description | Arguments |
|---|---|---|
| `GROUP_ROSTER_UPDATE` | Party/raid composition changed | — |
| `READY_CHECK` | A ready check was initiated | `initiatorName`, `readyCheckTimeLeft` |
| `RAID_TARGET_UPDATE` | A raid marker was set or removed | — |

### Bags & Inventory

| Event | Description | Arguments |
|---|---|---|
| `BAG_UPDATE` | Contents of a bag changed | `bagID` |
| `ITEM_LOCK_CHANGED` | An item's locked state changed (during moves) | `bagID`, `slotIndex` |
| `LOOT_READY` | Loot window is ready to be read | `autoLoot` |

### Quests

| Event | Description | Arguments |
|---|---|---|
| `QUEST_ACCEPTED` | A quest was accepted | `questID` |
| `QUEST_TURNED_IN` | A quest was turned in | `questID`, `xpReward`, `moneyReward` |
| `QUEST_LOG_UPDATE` | The quest log was updated | — |

### Zone & Map

| Event | Description | Arguments |
|---|---|---|
| `ZONE_CHANGED` | Subzone changed (within same area) | — |
| `ZONE_CHANGED_NEW_AREA` | Major zone change (new area/continent) | — |

### Talents & Specialization

| Event | Description | Arguments |
|---|---|---|
| `PLAYER_TALENT_UPDATE` | Talents were changed | — |
| `ACTIVE_TALENT_GROUP_CHANGED` | Active spec was switched | `newSpecIndex` |

## Common Patterns

### One-Time Initialization

Register for `ADDON_LOADED`, do your setup, then unregister:

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, addonName)
    if addonName ~= "MyAddon" then return end

    -- Initialize saved variables, build UI, etc.
    MyAddonDB = MyAddonDB or {}
    print("MyAddon loaded!")

    self:UnregisterEvent("ADDON_LOADED")
end)
```

### Combat Lockdown Queue

Many UI operations (moving frames, showing protected elements) are forbidden during combat. Queue them and execute when combat ends:

```lua
local actionQueue = {}

local function QueueAction(action)
    if InCombatLockdown() then
        table.insert(actionQueue, action)
    else
        action()
    end
end

local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_REGEN_ENABLED")
frame:SetScript("OnEvent", function(self, event)
    for _, action in ipairs(actionQueue) do
        action()
    end
    wipe(actionQueue)
end)

-- Usage
QueueAction(function()
    myFrame:SetPoint("CENTER")
    myFrame:Show()
end)
```

### Unit Event Filtering

Use `RegisterUnitEvent` to only receive events for units you care about, reducing unnecessary handler calls:

```lua
local frame = CreateFrame("Frame")

-- Only track health for player and target
frame:RegisterUnitEvent("UNIT_HEALTH", "player", "target")
frame:RegisterUnitEvent("UNIT_POWER_UPDATE", "player")
frame:RegisterUnitEvent("UNIT_AURA", "player")

frame:SetScript("OnEvent", function(self, event, unit, ...)
    if event == "UNIT_HEALTH" then
        local health = UnitHealth(unit)
        local maxHealth = UnitHealthMax(unit)
        print(unit, "health:", health .. "/" .. maxHealth)
    end
end)
```

## Further Reading

- [Warcraft Wiki — Events](https://warcraft.wiki.gg/wiki/Events) — Complete event listing with all arguments
- [Warcraft Wiki — Widget Handlers](https://warcraft.wiki.gg/wiki/Widget_handlers) — Frame script handlers including OnEvent and OnUpdate
- [Warcraft Wiki — API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) — Full API reference for functions used in event handlers
