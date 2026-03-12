# WoW Addon Lifecycle, Security, and Best Practices Reference

> Compiled March 2026. Covers retail WoW through Patch 12.0 (Midnight).

---

## 1. Addon Loading Order and Initialization Sequence

### 1.1 Initial Scan

When the WoW client launches, it scans every subfolder of `Interface/AddOns/` looking for
a valid `.toc` file. The TOC's `## Dependencies` and `## OptionalDeps` directives determine
the order in which addons are loaded (dependencies first).

### 1.2 File Execution Order

Files listed in the `.toc` are executed **top-to-bottom** in the order they appear.
Files included via XML `<Include file="..."/>` or `<Script file="..."/>` are executed at the
point the tag is encountered during parsing.

### 1.3 Saved Variables Loading

By default, an addon's saved variables are loaded **after** all its files have been executed.
The TOC directive `## LoadSavedVariablesFirst: 1` reverses this so saved variables load
**before** any addon files.

### 1.4 Key Initialization Events (in order)

| Order | Event | When / Purpose |
|-------|-------|----------------|
| 1 | **`ADDON_LOADED`** | Fires once per addon after its files **and** saved variables are loaded. Check `arg1` (addon name) to identify your addon. This is the **earliest safe time** to read saved variables. |
| 2 | **`PLAYER_LOGIN`** | Fires once after all addons are loaded but before the loading screen disappears. Most character data (spells, talents, inventory) is now available. Ideal for one-time initialization that depends on character state. |
| 3 | **`PLAYER_ENTERING_WORLD`** | Fires when the player enters the world (initial login, reload, zone transitions). On initial login it fires after `PLAYER_LOGIN`. Most game-world information is available. Good for final frame placement. |
| 4 | **`VARIABLES_LOADED`** | Fires when Blizzard's own CVars/keybindings load. **Not reliable** for addon initialization -- it may fire before or after `PLAYER_ENTERING_WORLD`. Do not depend on it. |

### 1.5 Recommended Initialization Pattern

```lua
local addonName, ns = ...

local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:RegisterEvent("PLAYER_LOGIN")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "ADDON_LOADED" then
        local name = ...
        if name == addonName then
            -- Saved variables are now available; initialize defaults
            ns:InitSavedVars()
            self:UnregisterEvent("ADDON_LOADED")
        end
    elseif event == "PLAYER_LOGIN" then
        -- Character data is ready; finish setup
        ns:SetupUI()
        ns:RegisterSlashCommands()
    end
end)
```

---

## 2. SavedVariables

### 2.1 Declaration in .toc

```toc
## SavedVariables: MyAddonDB
## SavedVariablesPerCharacter: MyAddonCharDB
```

- **`SavedVariables`** -- Saved per-account. One copy shared by all characters on the account. Stored in `WTF/Account/ACCOUNTNAME/SavedVariables/AddonName.lua`.
- **`SavedVariablesPerCharacter`** -- Saved per-character. Each character gets its own copy. Stored in `WTF/Account/ACCOUNTNAME/RealmName/CharacterName/SavedVariables/AddonName.lua`.

### 2.2 When They Are Available

Saved variables are `nil` until `ADDON_LOADED` fires for your addon. Before that event, the
global variables declared in your TOC simply do not exist yet (unless `LoadSavedVariablesFirst`
is set).

### 2.3 Initializing Defaults

On first use, the saved variable will be `nil`. The standard pattern:

```lua
function ns:InitSavedVars()
    -- Account-wide
    if not MyAddonDB then
        MyAddonDB = {}
    end
    -- Merge defaults (handles new keys added in updates)
    for k, v in pairs(ns.defaults) do
        if MyAddonDB[k] == nil then
            MyAddonDB[k] = v
        end
    end

    -- Per-character
    if not MyAddonCharDB then
        MyAddonCharDB = {}
    end
    for k, v in pairs(ns.charDefaults) do
        if MyAddonCharDB[k] == nil then
            MyAddonCharDB[k] = v
        end
    end
end
```

Key points:
- Always check for `nil` (not falsiness) to preserve intentional `false` values.
- When adding new settings in an update, iterate your defaults table and fill in missing keys.
- Tables nested inside saved variables need recursive merging if you add sub-keys.
- Variables are written to disk on logout/reload; there is no manual "save" call.

### 2.4 SavedVariables vs SavedVariablesPerCharacter

| Aspect | SavedVariables | SavedVariablesPerCharacter |
|--------|---------------|---------------------------|
| Scope | All characters on the account | One character only |
| Use case | Global settings, shared profiles | Character-specific state, per-char toggles |
| File location | `WTF/Account/NAME/SavedVariables/` | `WTF/Account/NAME/Realm/Char/SavedVariables/` |
| Common pattern | Store a profiles table with a key per character | Store only character-specific overrides |

---

## 3. Slash Command Registration

### 3.1 The Standard Pattern

```lua
SLASH_MYADDON1 = "/myaddon"
SLASH_MYADDON2 = "/ma"
SlashCmdList["MYADDON"] = function(msg)
    local command, rest = msg:match("^(%S*)%s*(.-)$")
    if command == "config" then
        ns:OpenConfig()
    elseif command == "reset" then
        ns:ResetDefaults()
    else
        ns:PrintHelp()
    end
end
```

### 3.2 How It Works

1. WoW reads `SlashCmdList["KEY"]` and looks for globals named `SLASH_KEY1`, `SLASH_KEY2`, etc.
2. The numbered globals **must be consecutive** starting at 1 (no gaps).
3. Each global's value is the actual slash command string (e.g., `"/myaddon"`).
4. The handler function receives the text typed **after** the slash command as `msg`.

### 3.3 Helper Function

```lua
SlashCmdList_AddSlashCommand("MYADDON", handlerFunc, "myaddon", "ma")
```

This utility creates the `SLASH_MYADDON1`, `SLASH_MYADDON2` globals and assigns the handler
in one call.

---

## 4. Localization / Internationalization

### 4.1 GetLocale() and the L Table Pattern

```lua
-- Locales/enUS.lua (default / fallback)
local addonName, ns = ...
local L = setmetatable({}, {
    __index = function(t, key)
        return key  -- missing keys return themselves
    end,
})
ns.L = L

-- English strings (optional if keys are English)
L["Options"] = "Options"
L["Enabled"] = "Enabled"
```

```lua
-- Locales/deDE.lua
local addonName, ns = ...
if GetLocale() ~= "deDE" then return end
local L = ns.L

L["Options"] = "Optionen"
L["Enabled"] = "Aktiviert"
```

### 4.2 Supported Locales

`enUS`, `enGB`, `deDE`, `esES`, `esMX`, `frFR`, `itIT`, `ptBR`, `ruRU`, `koKR`, `zhCN`, `zhTW`

### 4.3 Best Practices

- Load locale files in the TOC **before** any file that references `L`.
- Load the default (English) file first, then all other locale files.
- Use the metatable `__index` fallback so untranslated strings display the English key.
- Keep all localizable strings in dedicated locale files, not scattered through code.
- For library-based localization, **AceLocale-3.0** provides a robust framework with
  `NewLocale`, `GetLocale`, and automatic fallback to the base locale.

---

## 5. Security Model

### 5.1 Protected Functions

Protected functions are API calls that can only succeed from a **secure execution path**.
They exist to prevent addons from automating gameplay decisions, especially combat actions.

Categories of restricted functions:

| Category | Trigger on Violation | Examples |
|----------|---------------------|----------|
| **Always forbidden** to insecure code | `ADDON_ACTION_FORBIDDEN` | Movement functions, direct targeting |
| **Require hardware event** (user click/keypress) | `ADDON_ACTION_BLOCKED` | `UseAction()`, `CastSpellByName()`, `TargetUnit()`, `SetTitle()` |
| **Forbidden in combat** | `ADDON_ACTION_BLOCKED` | `CreateMacro()`, `EditMacro()`, `SetBinding()` |
| **Forbidden on secure frames in combat** | `ADDON_ACTION_BLOCKED` | `frame:Hide()`, `frame:Show()`, `frame:SetPoint()`, `frame:SetAttribute()` on secure frames |
| **Forbidden on restricted frames** (e.g. nameplates) | Lua error | Certain widget methods |
| **Forbidden from /run** | `ADDON_ACTION_BLOCKED` or silent fail | Some APIs cannot be called from `/run` or WeakAura scripts |

### 5.2 Taint System (Secure vs. Insecure Execution)

**Core concept:** When WoW starts executing Lua, execution begins in a **secure** state.
It becomes **tainted** the moment it reads a value or calls a function that originated from
addon code. Once tainted, any value the code writes also becomes tainted.

- **Taint spreads**: If tainted code writes to a table used by secure Blizzard code, that
  table entry becomes tainted, and the secure code may refuse to operate.
- **Taint persists**: Once spread, taint remains until `/reload` or relog.
- **Consequences**: Protected functions called from a tainted execution path will be blocked,
  especially during combat. This can break Blizzard UI elements.

**Avoiding taint:**

| Technique | How It Helps |
|-----------|-------------|
| `hooksecurefunc("FuncName", myPostHook)` | Post-hooks a secure function without tainting it. Your hook runs **after** the original. |
| `securecall(func, ...)` | Calls a function in a way that prevents taint from propagating back to the caller. |
| `issecurevariable(table, "key")` | Check if a variable is tainted before interacting with it. |
| Avoid writing to Blizzard global tables | The #1 cause of taint is addons modifying global Blizzard tables or frame attributes. |
| Use the addon namespace (`ns`) | Keep your data off the global table entirely. |

### 5.3 Combat Lockdown

**`InCombatLockdown()`** returns `true` when combat restrictions are active.

- Combat lockdown **begins** when `PLAYER_REGEN_DISABLED` fires.
- Combat lockdown **ends** when `PLAYER_REGEN_ENABLED` fires.

During combat lockdown you **cannot**:
- Show/hide/move secure frames (action bars, unit frames created with secure templates)
- Set attributes on secure frames (`frame:SetAttribute()`)
- Create or modify macros or keybindings
- Create new secure frames

**Pattern for deferred combat actions:**

```lua
local pendingActions = {}

local function ProcessPending()
    if InCombatLockdown() then return end
    for _, action in ipairs(pendingActions) do
        action()
    end
    wipe(pendingActions)
end

frame:RegisterEvent("PLAYER_REGEN_ENABLED")
frame:SetScript("OnEvent", function(self, event)
    if event == "PLAYER_REGEN_ENABLED" then
        ProcessPending()
    end
end)

function ns:QueueSecureAction(fn)
    if InCombatLockdown() then
        table.insert(pendingActions, fn)
    else
        fn()
    end
end
```

### 5.4 Hardware Event Requirements

Certain protected functions (marked "HW" in API documentation) can **only** be called in
direct response to a user hardware event (mouse click or key press). This means:

- They work inside `OnClick` handlers of buttons.
- They do **not** work from `OnUpdate`, `OnEvent`, timers, or C_Timer callbacks.
- The hardware event "propagates" through the call chain of the click handler -- you can call
  sub-functions from `OnClick` and they still have hardware-event privilege.

Examples requiring hardware events: `UseAction()`, `CastSpellByName()`, `CastSpellByID()`,
`TargetUnit("player")`, `PickupAction()`, most item usage functions.

### 5.5 Midnight (Patch 12.0) Combat Addon Restrictions

Patch 12.0 introduced significant changes to combat addon capabilities:

- **Philosophy**: Addons should no longer offer competitive advantages in combat. They remain
  tools for aesthetic customization and personalized information presentation.
- **Real-time combat data** is now a "black box" -- addons cannot interpret it for automated
  decision-making (rotation helpers, boss alerts with actionable callouts, etc.).
- **WeakAuras** and similar addons (DBM, BigWigs, Plater triggers) are heavily restricted
  in their combat functionality in retail.
- **Relaxations**: Blizzard acknowledged the initial restrictions were too broad and has
  eased some combat log interaction restrictions so benign addons (damage meters, log
  recorders, information displays) can still function.
- **What still works**: UI customization (raid frames, nameplates, action bars, fonts, art),
  presenting combat information in different visual layouts, out-of-combat functionality.

---

## 6. Best Practices

### 6.1 Performance

**OnUpdate handlers:**
- `OnUpdate` fires every rendered frame (potentially 60-240+ times/second).
- **Never** create tables, strings, or closures inside an OnUpdate handler.
- **Throttle** updates when you do not need per-frame granularity:

```lua
local elapsed_total = 0
local UPDATE_INTERVAL = 0.1 -- 10 times per second

frame:SetScript("OnUpdate", function(self, elapsed)
    elapsed_total = elapsed_total + elapsed
    if elapsed_total < UPDATE_INTERVAL then return end
    elapsed_total = 0
    -- Do actual work here
end)
```

**Local variable caching (upvalues):**
- Global lookups go through `_G` every time. Locals are stored on the Lua stack and are
  significantly faster to access.
- Cache frequently-used globals at file scope:

```lua
local pairs = pairs
local tinsert = table.insert
local GetTime = GetTime
local UnitHealth = UnitHealth
```

**Table reuse:**
- Pre-allocate working tables and `wipe()` them instead of creating new ones:

```lua
local sortBuffer = {}
local function DoSort(data)
    wipe(sortBuffer)
    for i, v in ipairs(data) do
        sortBuffer[i] = v
    end
    table.sort(sortBuffer, comparator)
    return sortBuffer
end
```

**String concatenation:**
- Avoid repeated `..` in hot paths; use `table.concat()` or `string.format()`.

### 6.2 Memory Management

- Memory usage is generally **less important** than CPU usage. Everything addons do happens
  between rendered frames; more work = longer frame times = lower FPS.
- Lua's garbage collector in WoW is incremental and low-impact for gradual allocation.
- The problem is **bursty garbage creation** (allocating many short-lived tables/strings in
  a single frame), which forces expensive GC pauses.
- Use `UpdateAddOnMemoryUsage()` and `GetAddOnMemoryUsage("AddonName")` to monitor.

### 6.3 Event Handler Dispatch Table

Instead of a long if/elseif chain, use a table of handler functions:

```lua
local addonName, ns = ...
local frame = CreateFrame("Frame")

local events = {}

function events:ADDON_LOADED(name)
    if name ~= addonName then return end
    ns:InitSavedVars()
    frame:UnregisterEvent("ADDON_LOADED")
end

function events:PLAYER_LOGIN()
    ns:SetupUI()
end

function events:PLAYER_ENTERING_WORLD(isInitialLogin, isReloadingUi)
    if isInitialLogin or isReloadingUi then
        ns:RefreshState()
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

### 6.4 Mixin Pattern

WoW provides `Mixin()` and `CreateFromMixins()` for composition-based OOP:

```lua
-- Define a mixin (just a table of methods/data)
MyAddonButtonMixin = {}

function MyAddonButtonMixin:OnLoad()
    self:RegisterForClicks("AnyUp")
end

function MyAddonButtonMixin:OnClick(button)
    if button == "LeftButton" then
        self:DoAction()
    end
end

function MyAddonButtonMixin:DoAction()
    print("Clicked!")
end

-- In XML: <Frame mixin="MyAddonButtonMixin">
-- Or in Lua:
local obj = CreateFromMixins(MyAddonButtonMixin)
-- For existing frames:
Mixin(existingFrame, MyAddonButtonMixin)
```

When multiple mixins are passed to `CreateFromMixins()`, later mixins override earlier ones.

### 6.5 Namespace / Encapsulation Patterns

**The addon namespace table:**

Every Lua file in an addon receives two arguments via `...`:

```lua
local addonName, ns = ...
```

- `addonName` -- string name of the addon (matches folder name).
- `ns` -- a private table shared across all Lua files in the addon. Use it to pass data
  between files without polluting `_G`.

```lua
-- Core.lua
local addonName, ns = ...
ns.version = "1.0.0"
ns.eventFrame = CreateFrame("Frame")

-- Config.lua
local addonName, ns = ...
-- ns.version and ns.eventFrame are accessible here
```

**Minimize globals:**
- Only expose globals that other addons need to interact with (a public API table).
- Use `local` for everything else.
- A common pattern: one global table named after the addon, everything else local or on `ns`.

```lua
-- One global entry point
MyAddon = {}
MyAddon.version = ns.version
function MyAddon:GetSetting(key)
    return MyAddonDB[key]
end
```

### 6.6 Addon Communication Channel

**Registering a prefix:**

```lua
C_ChatInfo.RegisterAddonMessagePrefix("MyAddon")
```

- Prefix is at most **16 characters**. Use your addon name if possible.
- You must register before you can receive messages for that prefix.

**Sending messages:**

```lua
C_ChatInfo.SendAddonMessage("MyAddon", dataString, "PARTY")
C_ChatInfo.SendAddonMessage("MyAddon", dataString, "RAID")
C_ChatInfo.SendAddonMessage("MyAddon", dataString, "GUILD")
C_ChatInfo.SendAddonMessage("MyAddon", dataString, "WHISPER", "PlayerName")
```

- Message body is at most **255 characters**.
- Chat types: `"PARTY"`, `"RAID"`, `"GUILD"`, `"WHISPER"`, `"CHANNEL"`.

**Receiving messages:**

```lua
frame:RegisterEvent("CHAT_MSG_ADDON")

function events:CHAT_MSG_ADDON(prefix, message, channel, sender)
    if prefix ~= "MyAddon" then return end
    ns:HandleAddonMessage(message, sender, channel)
end
```

**Throttling:**
- Server-side throttle: each prefix gets an allowance of **10 messages**, regenerating at
  **1 message per second** up to the cap of 10.
- For large data payloads, serialize, compress (LibDeflate), and encode (base64) before
  sending, then chunk into 255-byte segments with sequence numbers.

**`CHAT_MSG_ADDON_LOGGED`**: A variant event for messages that should also be recorded in
the combat log for Warcraft Logs and similar tools.

---

## 7. Quick Reference: Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Reading saved variables before `ADDON_LOADED` | Wait for `ADDON_LOADED` with `arg1 == addonName` |
| Modifying secure frames in combat | Check `InCombatLockdown()`, queue changes for `PLAYER_REGEN_ENABLED` |
| Writing to Blizzard global tables | Use `hooksecurefunc()` for post-hooks; never overwrite originals |
| Creating tables in OnUpdate | Pre-allocate and `wipe()` reusable tables |
| Long if/elseif event chains | Use a dispatch table keyed by event name |
| Global namespace pollution | Use `local addonName, ns = ...` and keep everything on `ns` |
| Not merging new default settings | Iterate defaults and fill `nil` keys on each load |
| Sending too many addon messages | Respect the 10-message throttle; batch and compress data |
| Calling protected functions from timers | Only call HW-required functions from `OnClick` handlers |
| Not throttling OnUpdate | Accumulate elapsed time and skip frames below threshold |

---

## Sources

- [AddOn loading process - Warcraft Wiki](https://warcraft.wiki.gg/wiki/AddOn_loading_process)
- [Saving variables between game sessions - Wowpedia](https://wowpedia.fandom.com/wiki/Saving_variables_between_game_sessions)
- [Creating a slash command - Warcraft Wiki](https://warcraft.wiki.gg/wiki/Creating_a_slash_command)
- [Localizing an addon - Warcraft Wiki](https://warcraft.wiki.gg/wiki/Localizing_an_addon)
- [Localizing an addon - Phanx](https://phanx.net/addons/tutorials/localize)
- [Secure Execution and Tainting - Warcraft Wiki](https://warcraft.wiki.gg/wiki/Secure_Execution_and_Tainting)
- [Restricted API functions - Warcraft Wiki](https://warcraft.wiki.gg/wiki/Category:API_functions/restricted)
- [InCombatLockdown - Warcraft Wiki](https://warcraft.wiki.gg/wiki/API_InCombatLockdown)
- [UI best practices - Wowpedia](https://wow.gamepedia.com/UI_best_practices)
- [Using OnUpdate correctly - WoWWiki](https://wowwiki-archive.fandom.com/wiki/Using_OnUpdate_correctly)
- [Use Tables Without Generating Extra Garbage - AddOn Studio](https://addonstudio.org/wiki/WoW:HOWTO:_Use_Tables_Without_Generating_Extra_Garbage)
- [Using the AddOn namespace - Wowpedia](https://wowpedia.fandom.com/wiki/Using_the_AddOn_namespace)
- [C_ChatInfo.SendAddonMessage - Warcraft Wiki](https://warcraft.wiki.gg/wiki/API_C_ChatInfo.SendAddonMessage)
- [C_ChatInfo.RegisterAddonMessagePrefix - Wowpedia](https://wowpedia.fandom.com/wiki/API_C_ChatInfo.RegisterAddonMessagePrefix)
- [Combat Philosophy and Addon Disarmament in Midnight - Blizzard](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight)
- [Combat Addon Restrictions Eased in Midnight - Icy Veins](https://www.icy-veins.com/wow/news/combat-addon-restrictions-eased-in-midnight/)
- [Handling events - Warcraft Wiki](https://warcraft.wiki.gg/wiki/Handling_events)
- [hooksecurefunc - Warcraft Wiki](https://warcraft.wiki.gg/wiki/API_hooksecurefunc)
- [PLAYER_ENTERING_WORLD - Wowpedia](https://wowpedia.fandom.com/wiki/PLAYER_ENTERING_WORLD)
- [Good Design in Warcraft Addons - Andy Dote](https://andydote.co.uk/2014/11/23/good-design-in-warcraft-addons/)
