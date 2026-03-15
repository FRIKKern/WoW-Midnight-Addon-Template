# AI Prompt Templates for WoW Addon Development

Ready-to-use prompts for ChatGPT, Claude, Copilot, or any AI assistant. Each prompt includes the constraints needed to produce **correct** WoW addon code and avoid the mistakes AI models commonly make.

!!! tip "How to use these prompts"
    1. Copy the prompt from the code block
    2. Paste it into your AI assistant
    3. The prompt already includes the technical constraints — just fill in the `[bracketed]` placeholders
    4. Check the "AI mistakes this prevents" section to know what to verify

---

## The Master System Prompt

Paste this at the **start of any conversation** about WoW addon development. It establishes the ground rules that prevent the most common AI mistakes.

```text
You are an expert World of Warcraft addon developer. Follow these rules strictly:

ENVIRONMENT:
- WoW uses Lua 5.1 (NOT 5.2, 5.3, or 5.4). No goto, no // operator, no bitwise
  operators from 5.3+. Use bit.band/bit.bor/bit.bxor for bitwise operations.
- Interface number: 120001 (Patch 12.0.x Midnight)
- No require(), dofile(), loadfile(), os, io, or debug libraries. These do not
  exist in WoW's sandbox.
- Math trig functions use DEGREES, not radians.
- Strings are 1-indexed (standard Lua).

ARCHITECTURE:
- Every file receives two varargs: local addonName, ns = ...
  Use ns (namespace table) to share data between files. NEVER use globals for
  addon state.
- Use the table dispatch pattern for events (define functions on a table keyed
  by event name, iterate to register, dispatch in OnEvent).
- Check InCombatLockdown() before any frame Show/Hide/SetPoint/SetAttribute
  operations on secure frames. Queue changes for PLAYER_REGEN_ENABLED.
- Initialize SavedVariables in ADDON_LOADED, not at file load time. They are
  nil until that event fires for your addon.

API RULES (Midnight 12.0):
- Use C_ namespace functions, NOT deprecated globals:
  C_Spell.GetSpellInfo(id) NOT GetSpellInfo(id)
  C_Spell.GetSpellCooldown(id) NOT GetSpellCooldown(id)
  C_AddOns.GetAddOnMetadata() NOT GetAddOnMetadata()
  C_Timer.After(seconds, callback) for delayed execution
- COMBAT_LOG_EVENT_UNFILTERED does NOT exist for addons in Midnight.
  Do not register for it or call CombatLogGetCurrentEventInfo().
- UnitHealth(), UnitPower(), UnitName() return SECRET VALUES in instanced
  combat. Addons can bind them to display widgets but cannot read, compare,
  or branch on them in Lua logic.
- Addon messaging (C_ChatInfo.SendAddonMessage) is restricted during active
  encounters. Guard sends with encounter state checks.
- Use Settings.RegisterAddOnCategory() for the options panel, not the
  removed InterfaceOptions_AddCategory().

CODE STYLE:
- Use local variables everywhere. Minimize globals (only SLASH_ and
  SlashCmdList entries).
- Use wipe(table) to clear tables, not table = {}.
- Use string.format() or format() for string building, not .. chains in loops.
- Always nil-check API returns before indexing into them.
```

??? example "What this prevents"
    | AI Mistake | What Happens In-Game |
    |---|---|
    | Using `require("module")` | Immediate Lua error — `require` doesn't exist |
    | Using Lua 5.3 `//` integer division | Syntax error at load time |
    | Using `goto` statements (Lua 5.2+) | Syntax error at load time |
    | Calling `GetSpellInfo()` directly | Function removed in 12.0, nil error |
    | Registering `COMBAT_LOG_EVENT_UNFILTERED` | Event doesn't exist, silent failure |
    | Branching on `UnitHealth()` in combat | Secret value — comparison always fails |
    | Using `InterfaceOptions_AddCategory()` | Removed function, nil error |
    | Global variable pollution | Silently overwrites other addons' state |
    | Accessing SavedVariables at load time | nil error — data not loaded yet |
    | Using radians in trig functions | Wrong angles — WoW uses degrees |

---

## Prompt 1: Create a New Addon from Scratch

```text
Create a complete WoW addon called [ADDON_NAME] that [DESCRIPTION].

Requirements:
- Generate the .toc file (Interface: 120001) and all .lua files
- Use the namespace pattern: local addonName, ns = ...
- Use table dispatch for event handling
- Include SavedVariables with defaults initialized in ADDON_LOADED
- Add slash command /[COMMAND] with help text
- Add Addon Compartment support (minimap button via TOC fields)
- Use Settings.RegisterAddOnCategory() for an options panel
- All frame operations must check InCombatLockdown()

Show me the complete file structure and contents of each file.
```

??? example "Example: what this produces"
    Fill in `[ADDON_NAME]` = `GoldTracker`, `[DESCRIPTION]` = `tracks gold earned per session and per day`, `[COMMAND]` = `gt`

    The AI should generate:

    **GoldTracker.toc**
    ```toc
    ## Interface: 120001
    ## Title: Gold Tracker
    ## Notes: Tracks gold earned per session and per day.
    ## Author: YourName
    ## Version: 1.0.0
    ## SavedVariables: GoldTrackerDB
    ## SavedVariablesPerCharacter: GoldTrackerCharDB
    ## IconTexture: Interface\Icons\INV_Misc_Coin_01
    ## AddonCompartmentFunc: GoldTracker_OnAddonCompartmentClick

    Core.lua
    Options.lua
    ```

    **Core.lua** should contain:
    ```lua
    local addonName, ns = ...

    -- Namespace utilities
    function ns:Print(msg)
        print("|cffffd700" .. addonName .. ":|r " .. msg)
    end

    -- Defaults
    local DEFAULTS = {
        dailyGold = {},
        showOnLogin = true,
    }

    -- Event handling with table dispatch
    local frame = CreateFrame("Frame")
    local events = {}

    function events:ADDON_LOADED(loadedAddonName)
        if loadedAddonName ~= addonName then return end
        GoldTrackerDB = GoldTrackerDB or {}
        -- Initialize defaults...
        frame:UnregisterEvent("ADDON_LOADED")
    end

    function events:PLAYER_LOGIN()
        -- Record starting gold...
    end

    -- Dispatcher
    frame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then
            events[event](self, ...)
        end
    end)

    for event in pairs(events) do
        frame:RegisterEvent(event)
    end

    -- Slash command
    SLASH_GOLDTRACKER1 = "/gt"
    SlashCmdList["GOLDTRACKER"] = function(msg)
        -- Handle commands...
    end

    -- Addon Compartment click handler
    function GoldTracker_OnAddonCompartmentClick(addonName, button)
        -- Toggle display...
    end
    ```

!!! warning "Verify these details in the output"
    - TOC has `## Interface: 120001` (not 110000 or other old values)
    - Uses `local addonName, ns = ...` at the top of every file
    - SavedVariables initialized inside `ADDON_LOADED`, not at file scope
    - No `require()` or `import` statements
    - Slash command uses the `SLASH_NAME1` / `SlashCmdList["NAME"]` pattern

---

## Prompt 2: Add a Settings Panel

```text
Add a settings/options panel to my WoW addon. My addon's namespace table is
called ns, addon name is [ADDON_NAME], and my SavedVariables table is
[SAVED_VAR_NAME] with these settings:

[LIST YOUR SETTINGS, e.g.:
- enabled (boolean, default true)
- scale (number, 0.5-2.0, default 1.0)
- soundEnabled (boolean, default true)
- displayMode (dropdown: "minimal", "detailed", "full", default "detailed")
]

Requirements:
- Use Settings.RegisterAddOnCategory() (NOT InterfaceOptions_AddCategory)
- Use Settings.RegisterProxySetting for each option
- Include proper category name and layout
- Settings must read from and write to the SavedVariables table
- Include a "Defaults" button that resets all settings
- Panel must be accessible via /[COMMAND] config and the game's addon settings

WoW Lua 5.1, Interface 120001 (Midnight). Use C_ namespace APIs.
```

??? example "What this produces"
    A settings panel using the modern Settings API:
    ```lua
    local addonName, ns = ...

    function ns:InitializeOptions()
        local category = Settings.RegisterVerticalLayoutCategory(addonName)

        -- Boolean setting
        do
            local variable = Settings.RegisterProxySetting(category,
                addonName .. "_Enabled", type(true),
                "Enable Addon", ns.db.enabled,
                function() return ns.db.enabled end,
                function(value) ns.db.enabled = value end
            )
            Settings.CreateCheckbox(category, variable, "Enable or disable the addon")
        end

        -- Slider setting
        do
            local variable = Settings.RegisterProxySetting(category,
                addonName .. "_Scale", type(1.0),
                "UI Scale", ns.db.scale,
                function() return ns.db.scale end,
                function(value)
                    ns.db.scale = value
                    ns:ApplyScale()
                end
            )
            local options = Settings.CreateSliderOptions(0.5, 2.0, 0.1)
            Settings.CreateSlider(category, variable, options, "Adjust the display scale")
        end

        Settings.RegisterAddOnCategory(category)
        ns.settingsCategory = category
    end
    ```

!!! warning "Verify these details in the output"
    - Uses `Settings.RegisterAddOnCategory()`, not the removed `InterfaceOptions_AddCategory()`
    - Does not reference `InterfaceOptionsFramePanelContainer` (removed)
    - Settings read from and write to your SavedVariables, not temporary variables
    - No `CreateFrame("Frame", nil, InterfaceOptionsFramePanelContainer)` pattern (that's the old way)

---

## Prompt 3: Minimap Button / Addon Compartment

```text
Add both an Addon Compartment entry (Midnight's native minimap menu) AND a
traditional LibDataBroker minimap button to my WoW addon [ADDON_NAME].

The addon compartment entry should:
- Show a tooltip with addon name, version, and usage hints
- Left-click toggles the main window
- Right-click opens settings

The LibDataBroker minimap button should:
- Use LibDBIcon-1.0 for positioning/dragging
- Persist position in SavedVariables
- Show the same tooltip and click behavior
- Be hideable (some users prefer Compartment only)

My namespace is ns, saved variables table is [SAVED_VAR_NAME].
WoW Lua 5.1, Interface 120001. Show complete code.
```

??? example "What this produces"
    **TOC additions:**
    ```toc
    ## AddonCompartmentFunc: MyAddon_OnAddonCompartmentClick
    ## AddonCompartmentFuncOnEnter: MyAddon_OnAddonCompartmentEnter
    ## AddonCompartmentFuncOnLeave: MyAddon_OnAddonCompartmentLeave
    ## IconTexture: Interface\Icons\INV_Misc_QuestionMark
    ```

    **Compartment handler:**
    ```lua
    function MyAddon_OnAddonCompartmentClick(addonName, button)
        if button == "LeftButton" then
            ns:ToggleMainWindow()
        elseif button == "RightButton" then
            Settings.OpenToCategory(ns.settingsCategory)
        end
    end

    function MyAddon_OnAddonCompartmentEnter(_, menuButtonFrame)
        GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
        GameTooltip:AddLine(addonName)
        GameTooltip:AddLine("Left-click: Toggle window", 1, 1, 1)
        GameTooltip:AddLine("Right-click: Settings", 1, 1, 1)
        GameTooltip:Show()
    end

    function MyAddon_OnAddonCompartmentLeave()
        GameTooltip:Hide()
    end
    ```

!!! warning "Verify these details in the output"
    - Compartment functions are **global** (not local) — they must be, since the TOC references them by name
    - TOC fields use `##` prefix with correct field names
    - `GameTooltip:SetOwner()` is called before adding lines
    - LibDBIcon usage stores minimap position in SavedVariables (not hardcoded)

---

## Prompt 4: Build a Unit Frame Addon

```text
Create a custom unit frame addon for WoW Midnight (12.0, Interface 120001)
that displays [player/target/party/raid] frames with:

- Health bar (using StatusBar widget with proper Secret Values handling)
- Power bar (mana/rage/energy)
- Name text
- Level and class color
- Buff/debuff icons
- Cast bar (if target)

CRITICAL Midnight constraints:
- UnitHealth(), UnitPower() return SECRET VALUES in instanced combat.
  You can bind them to StatusBar:SetValue() for display, but you CANNOT
  read the values, compare them, or use them in if/then logic during combat.
- Use UNIT_HEALTH, UNIT_POWER_UPDATE, UNIT_AURA events for updates.
- Do NOT use COMBAT_LOG_EVENT_UNFILTERED — it does not exist in 12.0.
- Check InCombatLockdown() before frame manipulation.
- For heal prediction, use CreateUnitHealPredictionCalculator() with
  SetIncomingHealClampMode().
- Focus on SKINNING Blizzard's data pipeline, not replacing it.

Lua 5.1, no require/dofile. Show complete addon with TOC and all files.
```

??? example "AI mistakes this prevents"
    | Mistake | Why It's Wrong |
    |---|---|
    | `if UnitHealth("target") < 1000 then` | Cannot branch on secret values in combat |
    | `local healthPercent = UnitHealth(unit) / UnitHealthMax(unit)` | Cannot do arithmetic on secret values |
    | Registering for `COMBAT_LOG_EVENT_UNFILTERED` | Event removed in 12.0 |
    | Creating frames without `InCombatLockdown()` check | Will error or silently fail in combat |
    | Using `UnitBuff()` / `UnitDebuff()` directly | Replaced by `C_UnitAuras.GetAuraDataByIndex()` in modern API |
    | Hardcoding power bar colors | Should use `PowerBarColor[powerType]` or class-appropriate colors |

---

## Prompt 5: Create a Boss Encounter Helper

```text
Create a WoW addon that enhances the boss encounter experience in Midnight
(12.0, Interface 120001). The addon should:

- Skin/restyle Blizzard's built-in Boss Ability Timeline
- Add custom audio alerts layered on Blizzard's alert severity pools
  (minor, medium, critical)
- Display custom timer bars using the reminder system
- Show a pull timer with countdown
- Track encounter duration
- Display post-wipe summary (boss health %, deaths, duration)

CRITICAL Midnight constraints:
- COMBAT_LOG_EVENT_UNFILTERED does NOT exist. Do not register for it.
- Addon messaging (C_ChatInfo.SendAddonMessage) is blocked during
  active encounters. Queue messages for ENCOUNTER_END.
- You cannot independently detect boss abilities — work WITH Blizzard's
  native Boss Timeline and alert system, not around it.
- Boss health is a secret value during encounters. You can display it
  but not read it programmatically.
- Use ENCOUNTER_START, ENCOUNTER_END events for encounter boundaries.
- Post-combat data is available between pulls.

Design philosophy: "Skin, don't replace." Enhance what Blizzard provides.
Lua 5.1, no require/dofile. Show complete addon.
```

!!! warning "Verify these details in the output"
    - No references to `COMBAT_LOG_EVENT_UNFILTERED` or `CombatLogGetCurrentEventInfo()`
    - No attempts to parse combat log events
    - Uses `ENCOUNTER_START` / `ENCOUNTER_END` for encounter boundaries
    - Addon messaging is guarded with encounter state checks
    - Post-wipe data uses only APIs that are accessible between pulls
    - Audio alerts use `PlaySound()` or `PlaySoundFile()`, not blocked APIs

---

## Prompt 6: Build an Auction House / Economy Addon

```text
Create a WoW addon for auction house and economy tracking in Midnight
(12.0, Interface 120001). Features:

- Scan the AH and store price history in SavedVariables
- Show tooltip price information when hovering items
- Track daily gold income/expenses per character
- Display a price history graph for items
- Batch posting interface for common items

Technical requirements:
- Use C_AuctionHouse namespace (NOT the old AuctionFrame functions)
- Key functions: C_AuctionHouse.GetItemSearchResultInfo(),
  C_AuctionHouse.SearchForItemKeys(), C_AuctionHouse.PostCommodity(),
  C_AuctionHouse.PostItem()
- Use AUCTION_HOUSE_SHOW, AUCTION_HOUSE_CLOSED,
  COMMODITY_SEARCH_RESULTS_UPDATED, ITEM_SEARCH_RESULTS_UPDATED events
- Store price data efficiently (average, min, max, sample count per day)
- Tooltip hooks via GameTooltip:HookScript("OnTooltipSetItem", ...)
- This is a non-combat addon — Secret Values are NOT relevant here.

Lua 5.1, Interface 120001. Use local + namespace pattern. Show complete
addon with TOC and all files.
```

??? example "AI mistakes this prevents"
    | Mistake | Why It's Wrong |
    |---|---|
    | Using `AuctionFrameBrowse` globals | Old AH API, replaced by `C_AuctionHouse` |
    | Calling `QueryAuctionItems()` | Removed — use `C_AuctionHouse.SearchForItemKeys()` |
    | Using `GetAuctionItemInfo()` | Removed — use `C_AuctionHouse.GetItemSearchResultInfo()` |
    | Storing raw price tables without compression | SavedVariables bloat — will cause login lag |
    | Using `GameTooltip:AddLine()` without checking `GameTooltip:GetItem()` | May add lines to wrong tooltip |

---

## Prompt 7: Debug My Existing Addon

```text
I'm getting this error in my WoW addon:

```
[ERROR TEXT - paste your error here]
```

My addon code is:

```lua
[PASTE RELEVANT CODE HERE]
```

Environment: WoW Midnight (Patch 12.0, Interface 120001), Lua 5.1.

Please:
1. Explain what's causing the error
2. Check if I'm using any APIs removed or changed in Midnight 12.0:
   - GetSpellInfo() → C_Spell.GetSpellInfo()
   - GetSpellCooldown() → C_Spell.GetSpellCooldown()
   - CombatLogGetCurrentEventInfo() → REMOVED, no addon replacement
   - GetAddOnMetadata() → C_AddOns.GetAddOnMetadata()
   - InterfaceOptions_AddCategory() → Settings.RegisterAddOnCategory()
3. Check if I'm branching on Secret Values (UnitHealth, UnitPower, etc.)
   in combat contexts
4. Check for Lua 5.1 compatibility issues (no goto, no //, no 5.3 bitwise)
5. Show the corrected code
```

!!! tip "Getting the error text"
    Install [BugSack](https://www.curseforge.com/wow/addons/bugsack) + [BugGrabber](https://www.curseforge.com/wow/addons/bug-grabber) for clean error capture. Or enable the built-in error display: `/console scriptErrors 1`

---

## Prompt 8: Migrate My Addon to Midnight 12.0

```text
I need to update my WoW addon from [PREVIOUS_EXPANSION] to Midnight
(Patch 12.0, Interface 120001). Here is my addon code:

[PASTE YOUR ADDON FILES HERE]

Please perform a complete migration:

1. UPDATE TOC: Change Interface to 120001

2. REPLACE REMOVED APIs:
   - GetSpellInfo(id) → C_Spell.GetSpellInfo(id) (returns a table, not
     multiple values)
   - GetSpellCooldown(id) → C_Spell.GetSpellCooldown(id) (returns a table)
   - GetAddOnMetadata() → C_AddOns.GetAddOnMetadata()
   - UnitBuff/UnitDebuff → C_UnitAuras.GetAuraDataByIndex() or
     AuraUtil.ForEachAura()
   - InterfaceOptions_AddCategory() → Settings.RegisterAddOnCategory()

3. REMOVE CLEU DEPENDENCIES:
   - Remove any COMBAT_LOG_EVENT_UNFILTERED handlers
   - Remove calls to CombatLogGetCurrentEventInfo()
   - Replace with appropriate alternatives or remove combat log features

4. HANDLE SECRET VALUES:
   - Find any code that reads UnitHealth/UnitPower/combat data and
     uses it in conditional logic
   - Convert to display-only patterns where possible
   - Mark features that are no longer possible under Midnight

5. GUARD ADDON MESSAGING:
   - Wrap C_ChatInfo.SendAddonMessage() calls with encounter state checks
   - Queue messages during encounters, flush on ENCOUNTER_END

6. Show me a complete diff of all changes with explanations.
```

!!! warning "Important: C_Spell.GetSpellInfo() returns differently"
    The most common migration mistake: `GetSpellInfo()` returned multiple values (`name, rank, icon, ...`). The replacement `C_Spell.GetSpellInfo()` returns a **single table**:
    ```lua
    -- OLD (removed):
    local name, rank, icon = GetSpellInfo(spellID)

    -- NEW:
    local info = C_Spell.GetSpellInfo(spellID)
    if info then
        local name = info.name
        local icon = info.iconID  -- note: iconID, not icon
    end
    ```

---

## Prompt 9: Create a Custom Display (WeakAura Replacement)

```text
Create a WoW addon that provides a custom visual overlay for [DESCRIBE
WHAT YOU WANT TO TRACK]. This is for Midnight (12.0, Interface 120001)
where WeakAuras combat triggers no longer work.

The display should show:
- [Describe elements: progress bars, icons, text, etc.]
- [Position: center screen, near player frame, etc.]
- [Behavior: when to show/hide, animations, etc.]

CRITICAL Midnight constraints:
- This is a VISUAL DISPLAY addon, not a decision-making addon.
- For whitelisted spells (class resources like Holy Power, Combo Points,
  Maelstrom Weapon, Soul Fragments), you CAN read cooldown and charge
  data via C_Spell.GetSpellCooldown().
- For non-whitelisted combat data, you can only DISPLAY values through
  StatusBar/FontString binding — not read them.
- Use UNIT_AURA, UNIT_POWER_UPDATE, SPELL_UPDATE_COOLDOWN events.
- Do NOT use COMBAT_LOG_EVENT_UNFILTERED.
- Out-of-combat tracking (profession cooldowns, world timers, etc.) has
  NO restrictions — full API access.

Lua 5.1, namespace pattern, table dispatch events. Show complete addon.
```

??? example "What works vs. what doesn't in Midnight"
    | Use Case | Possible? | How |
    |---|---|---|
    | Cooldown spinner for whitelisted spells | :material-check: Yes | `C_Spell.GetSpellCooldown()` on whitelisted spell IDs |
    | Class resource bar (Holy Power, etc.) | :material-check: Yes | Class resources are explicitly non-secret |
    | Health bar that changes color at low HP | :material-close: No | Cannot branch on health value in combat |
    | Proc glow when buff appears | :material-check: Partial | Can detect some auras via `UNIT_AURA`, but many are secret |
    | Profession cooldown tracker | :material-check: Yes | No restrictions outside combat/instances |
    | Boss ability countdown | :material-close: No | Cannot independently detect boss abilities |
    | Out-of-combat reminder timers | :material-check: Yes | Full API access outside combat |

---

## Prompt 10: Add Localization Support

```text
Add multi-language localization support to my WoW addon [ADDON_NAME].
Current English strings are hardcoded. Here is my addon code:

[PASTE YOUR CODE OR LIST YOUR USER-FACING STRINGS]

Requirements:
- Create a Locales/ folder with one file per language
- Use the standard GetLocale() pattern to load the right language
- Support at least: enUS, deDE, frFR, esES, ptBR, zhCN, zhTW, koKR, ruRU
- English (enUS) is the fallback for any missing translations
- Non-English files should have empty strings as placeholders for
  translators to fill in
- Use a metatable with __index fallback so missing translations
  show the English text
- Strings must be accessible via ns.L["key"] from any file

Lua 5.1. File loading order matters — locale files must load before
core files in the TOC. Show the TOC changes and all locale files.
```

??? example "What this produces"
    **TOC file order:**
    ```toc
    ## Interface: 120001
    ## Title: My Addon

    Locales\enUS.lua
    Locales\deDE.lua
    Locales\frFR.lua
    Locales\esES.lua
    Locales\ptBR.lua
    Locales\zhCN.lua
    Locales\zhTW.lua
    Locales\koKR.lua
    Locales\ruRU.lua
    Core.lua
    Options.lua
    ```

    **Locales/enUS.lua:**
    ```lua
    local addonName, ns = ...

    local L = setmetatable({}, {
        __index = function(self, key)
            return key  -- fallback: return the key itself
        end,
    })
    ns.L = L

    -- English strings (always loaded, serves as fallback)
    L["WELCOME_MESSAGE"] = "Welcome! Type /%s for options."
    L["SETTINGS_TITLE"] = "Settings"
    L["ENABLE_ADDON"] = "Enable Addon"
    L["SCALE_LABEL"] = "UI Scale"
    L["RESET_CONFIRM"] = "Are you sure you want to reset all settings?"
    ```

    **Locales/deDE.lua:**
    ```lua
    local addonName, ns = ...
    if GetLocale() ~= "deDE" then return end

    local L = ns.L
    L["WELCOME_MESSAGE"] = "Willkommen! Tippe /%s für Optionen."
    L["SETTINGS_TITLE"] = "Einstellungen"
    -- Untranslated strings fall back to English automatically
    ```

!!! warning "Verify these details in the output"
    - Locale files load **before** core files in the TOC
    - English file creates the table with `__index` fallback
    - Non-English files `return` early if locale doesn't match
    - All user-facing strings go through `ns.L["KEY"]`, not hardcoded

---

## Bonus: The Quick Fix Prompts

Short prompts for common one-off tasks:

### Fix a Taint Error

```text
My WoW addon is causing a taint error:
[PASTE ERROR]

This addon is for Midnight 12.0 (Interface 120001). Explain what's
causing the taint, and show me how to fix it. Remember:
- Setting attributes on secure frames in combat causes taint
- Reading tainted globals propagates taint
- hooksecurefunc() is safe; directly replacing Blizzard functions is not
```

### Add a Movable Frame

```text
Make this WoW frame movable by the user (drag to reposition) and save
its position in SavedVariables. Must check InCombatLockdown() before
restoring position. Lua 5.1, Midnight 12.0.

Frame variable name: [FRAME_NAME]
SavedVariables table: [SAVED_VAR_NAME]
```

### Add Tooltip to Any Frame

```text
Add a tooltip to my WoW frame that shows on mouseover. The tooltip
should display:
[DESCRIBE TOOLTIP CONTENT]

Use GameTooltip:SetOwner() with proper anchoring. Must call
GameTooltip:Hide() on leave. Lua 5.1, Midnight 12.0.
```

### Create a Scrollable List

```text
Create a scrollable list frame for my WoW addon that displays [DESCRIBE
DATA]. Use the modern ScrollBox API (NOT the old FauxScrollFrame or
HybridScrollFrame which are deprecated).

Data source: [DESCRIBE WHERE DATA COMES FROM]
Lua 5.1, Midnight 12.0. Show complete code.
```

---

## Prompt Engineering Tips

### Always Include These in Your Prompts

!!! success "The non-negotiable context"
    Every WoW addon prompt should specify:

    1. **"Lua 5.1"** — Without this, AI assumes modern Lua and generates incompatible code
    2. **"Interface 120001"** or **"Midnight 12.0"** — Prevents outdated API usage
    3. **"No require/dofile"** — Stops the AI from using standard Lua module patterns
    4. **"Use C_ namespace APIs"** — Prevents deprecated function calls
    5. **"COMBAT_LOG_EVENT_UNFILTERED does not exist"** — Critical for any combat-adjacent addon

### Structure Your Requests

```text
BAD:  "Make me a damage meter addon"
GOOD: "Create a WoW addon (Lua 5.1, Interface 120001, Midnight) that
       displays post-combat damage breakdowns. CLEU does not exist in
       12.0. Use between-pull data access. Show TOC and all files."
```

The more constraints you give, the less you have to fix afterward.

### How to Verify AI-Generated WoW Code

Run through this checklist before using any AI-generated addon code:

- [ ] **TOC file** has `## Interface: 120001`
- [ ] **No `require()`, `dofile()`, `loadfile()`** anywhere
- [ ] **No `os.*`, `io.*`, `debug.*`** calls
- [ ] **No Lua 5.2+ syntax**: `goto`, `//`, bitwise operators (`&`, `|`, `~`, `<<`, `>>`)
- [ ] **Uses `local addonName, ns = ...`** at the top of every file
- [ ] **SavedVariables** initialized in `ADDON_LOADED`, not at file scope
- [ ] **No `GetSpellInfo()`** — uses `C_Spell.GetSpellInfo()` instead
- [ ] **No `COMBAT_LOG_EVENT_UNFILTERED`** registration
- [ ] **No branching on Secret Values** (UnitHealth, UnitPower) in combat code
- [ ] **`InCombatLockdown()` check** before frame Show/Hide/SetPoint on secure frames
- [ ] **No `InterfaceOptions_AddCategory()`** — uses `Settings.RegisterAddOnCategory()`
- [ ] **Slash commands** use `SLASH_NAME1` + `SlashCmdList["NAME"]` pattern (globals are correct here)
- [ ] **File loading order** in TOC makes sense (locales before core, core before UI)

### Red Flags in AI Output

!!! danger "Stop and fix if you see any of these"

    | Red Flag | What's Wrong |
    |---|---|
    | `require("something")` | Doesn't exist in WoW |
    | `local x = import("module")` | Not a thing in WoW Lua |
    | `goto label` | Lua 5.2+ syntax, won't parse |
    | `x // 2` | Lua 5.3 integer division, syntax error |
    | `x & 0xFF` | Lua 5.3 bitwise, use `bit.band(x, 0xFF)` |
    | `GetSpellInfo(id)` returning multiple values | Removed in 12.0 |
    | `CombatLogGetCurrentEventInfo()` | Removed in 12.0 |
    | `COMBAT_LOG_EVENT_UNFILTERED` | Doesn't exist for addons |
    | `if UnitHealth("target") < threshold then` | Secret value in combat |
    | `InterfaceOptions_AddCategory(panel)` | Removed, use Settings API |
    | `UIDropDownMenu_CreateInfo()` | Old dropdown API, use new Menu system |
    | `getglobal("frameName")` | Ancient pattern, use `_G["frameName"]` or direct reference |
    | `this` as implicit frame reference | Removed years ago, use explicit `self` |
    | `arg1, arg2, ...` as implicit event args | Removed years ago, use `function(self, event, ...)` |

### Iterating with AI

When the first output has issues:

```text
The code you generated has these problems:
1. Line X uses GetSpellInfo() which was removed in 12.0 — replace with
   C_Spell.GetSpellInfo() which returns a table
2. Line Y branches on UnitHealth() which returns a secret value in
   Midnight combat — convert to display-only
3. Line Z uses InterfaceOptions_AddCategory which was removed — use
   Settings.RegisterAddOnCategory()

Please fix all three issues and show the corrected code.
```

Be specific about what's wrong and what the replacement should be. AI models correct errors much better when you name the exact function and its replacement.

---

## Quick Reference: Removed → Replacement

Keep this handy when reviewing AI output:

| Removed Function | Replacement | Notes |
|---|---|---|
| `GetSpellInfo(id)` | `C_Spell.GetSpellInfo(id)` | Returns a table, not multiple values |
| `GetSpellCooldown(id)` | `C_Spell.GetSpellCooldown(id)` | Returns a table with `.startTime`, `.duration` |
| `GetAddOnMetadata(name, field)` | `C_AddOns.GetAddOnMetadata(name, field)` | Same signature |
| `CombatLogGetCurrentEventInfo()` | *None* | Removed entirely for addons |
| `InterfaceOptions_AddCategory()` | `Settings.RegisterAddOnCategory()` | Completely different API |
| `UnitBuff(unit, index)` | `C_UnitAuras.GetAuraDataByIndex(unit, index, "HELPFUL")` | Returns a table |
| `UnitDebuff(unit, index)` | `C_UnitAuras.GetAuraDataByIndex(unit, index, "HARMFUL")` | Returns a table |
| `GetContainerItemInfo(bag, slot)` | `C_Container.GetContainerItemInfo(bag, slot)` | Returns a table |
| `GetContainerNumSlots(bag)` | `C_Container.GetContainerNumSlots(bag)` | Same signature |
| `UIDropDownMenu_CreateInfo()` | `MenuUtil.CreateContextMenu()` | Completely different API |

---

!!! tip "Start here"
    New to WoW addon development? Read the [Getting Started](getting-started.md) guide first, then come back here with context about what addons *are* before asking an AI to build one. Understanding the [Midnight restrictions](midnight.md) is essential for verifying AI output.
