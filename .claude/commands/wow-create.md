Create a WoW addon: $ARGUMENTS

**Development Mode:** Before creating the addon, read `.claude/modes/active-mode.md` to determine the active mode (default: `enhancement-artist`), then read `.claude/modes/{mode-name}.md` for mode-specific rules. Pass the mode context to the Coder agent. The mode affects which patterns, APIs, and techniques are used in the generated code.

If the user includes a mode keyword in $ARGUMENTS (e.g., "create a **faithful** damage meter" or "create a **boundary** nameplate addon"), use that mode instead of the active mode. Keywords: faithful/blizzard/safe, boundary/aggressive/advanced, enhance/better/skin, performance/perf/fast.

You are a WoW Midnight (12.0.1) addon generator. Create a complete, ready-to-install addon based on the user's description.

## Process

### Step 1: Understand the Request

Parse `$ARGUMENTS` for:
- **Core functionality** — What does the addon do?
- **Addon name** — Extract or generate a PascalCase name (e.g., "RareTracker", "HousingHelper")
- **UI needs** — Does it need frames, buttons, minimap icons, slash commands?
- **Data needs** — Does it need SavedVariables, combat data, unit info, housing APIs?
- **Combat relevance** — Will it operate during encounters? (Secret Values implications)

### Step 2: Read the Template

Read these template files for patterns:
- `template/MyAddon.toc` — TOC format, metadata, Addon Compartment, Category
- `template/Init.lua` — Namespace, event dispatch, Secret Values helpers, Debounce, SV migration
- `template/Core.lua` — Combat queue, draggable frames, hooksecurefunc, issecretvalue guards
- `template/Config.lua` — Settings API panel with checkbox, slider, dropdown examples
- `template/CLAUDE.md` — AI coding instructions, deprecated API table, Secret Values rules
- `template/.pkgmeta` — Packager config with externals and nolib support
- `template/.luacheckrc` — Linter config with WoW globals
- `template/.github/workflows/release.yml` — CI/CD release pipeline
- `template/.github/workflows/lint.yml` — CI lint on push/PR
- `template/Libs/embeds.xml` — Library loader XML

### Step 3: Research Required APIs

Use the WoW Addon Researcher agent to verify every API you plan to use:
- Confirm functions exist in 12.0.1
- Get correct signatures
- Check for Secret Values implications
- Find real-world usage patterns

### Step 4: Generate the Addon

Create all files in a new directory named after the addon. Follow these MANDATORY patterns:

**TOC File Requirements:**
```toc
## Interface: 120001
## Title: AddonName
## Notes: Description
## Author: [user]
## Version: @project-version@
## SavedVariables: AddonNameDB
## IconTexture: Interface\Icons\[appropriate-icon]
## AddonCompartmentFunc: AddonName_OnCompartmentClick
## AddonCompartmentFuncOnEnter: AddonName_OnCompartmentEnter
## AddonCompartmentFuncOnLeave: AddonName_OnCompartmentLeave
## OptionalDeps: LibStub, CallbackHandler-1.0
## X-Curse-Project-ID: 0
## X-Wago-ID: CHANGEME
## Category: [appropriate-category]

#@no-lib-strip@
Libs\LibStub\LibStub.lua
Libs\CallbackHandler-1.0\CallbackHandler-1.0.lua
#@end-no-lib-strip@

Init.lua
Core.lua
Config.lua
```

**Init.lua — Namespace Pattern (MANDATORY):**
```lua
local addonName, ns = ...
ns.addonName = addonName

-- Shared state
ns.db = {}
ns.isLoaded = false

-- Secret Values (Midnight 12.0+)
ns.SECRETS_ENABLED = type(issecretvalue) == "function"
function ns.SafeValue(val, fallback)
    if ns.SECRETS_ENABLED and issecretvalue(val) then
        return fallback
    end
    return val
end

-- Debounce utility
function ns.Debounce(delay, fn)
    local timer
    return function(...)
        if timer then timer:Cancel() end
        local args = { ... }
        timer = C_Timer.NewTimer(delay, function()
            timer = nil
            fn(unpack(args))
        end)
    end
end
```

**Core.lua — Event Dispatch (MANDATORY):**
```lua
local addonName, ns = ...

local frame = CreateFrame("Frame")
local events = {}

ns.defaults = {
    _version = 1,
    enabled = true,
    -- addon-specific defaults here
}

function events:ADDON_LOADED(loadedName)
    if loadedName ~= addonName then return end
    -- Initialize SavedVariables with defaults merge
    AddonNameDB = AddonNameDB or {}
    for key, default in pairs(ns.defaults) do
        if AddonNameDB[key] == nil then
            AddonNameDB[key] = default
        end
    end
    ns.db = AddonNameDB
    ns.db._version = ns.defaults._version  -- migration stamp
    ns.isLoaded = true
    frame:UnregisterEvent("ADDON_LOADED")
end

-- Register and dispatch
for event in pairs(events) do
    frame:RegisterEvent(event)
end
frame:SetScript("OnEvent", function(self, event, ...)
    if events[event] then
        events[event](self, ...)
    end
end)
```

**Combat Data — Secret Values Guard (if applicable):**
```lua
local function SafeGetHealth(unit)
    local health = UnitHealth(unit)
    if issecretvalue(health) then
        return nil -- Cannot process, degrade gracefully
    end
    return health
end
```

**Communication Lockdown (if using addon comms):**
```lua
local pendingMessages = {}

local function SafeSendMessage(prefix, text, channel, target)
    if C_ChatInfo.InChatMessagingLockdown() then
        table.insert(pendingMessages, {prefix, text, channel, target})
        return false
    end
    C_ChatInfo.SendAddonMessage(prefix, text, channel, target)
    return true
end

-- Flush on encounter end
function events:ENCOUNTER_END()
    for _, msg in ipairs(pendingMessages) do
        C_ChatInfo.SendAddonMessage(unpack(msg))
    end
    wipe(pendingMessages)
end
```

### Step 5: Verify Generated Code

After generating all files, run verification:
1. Check every API call against the deprecated list
2. Ensure no hallucinated functions (C_Spell.GetSpellName, C_Timer.SetTimeout, etc.)
3. Verify Secret Values handling if combat-relevant
4. Confirm Interface version is 120001
5. Check namespace pattern is used (no bare globals)
6. Verify event dispatch table pattern

### Step 6: Output

Write all files to a directory. Provide a summary:

```markdown
## Addon Created: [Name]

**Files:**
- `[Name]/[Name].toc` — Interface 120001, Addon Compartment, Category
- `[Name]/Init.lua` — Namespace, Secret Values, Debounce, SV migration, slash commands
- `[Name]/Core.lua` — Main logic, event handlers, combat queue
- `[Name]/Config.lua` — Settings panel (if needed)
- `[Name]/CLAUDE.md` — AI coding instructions
- `[Name]/.pkgmeta` — Packager config with externals
- `[Name]/.luacheckrc` — Linter config with addon globals
- `[Name]/.github/workflows/release.yml` — Tag → package → upload
- `[Name]/.github/workflows/lint.yml` — Luacheck on push/PR
- `[Name]/.gitignore` — Ignores Libs/, .release/, *.zip
- `[Name]/.editorconfig` — Consistent formatting

**Features:**
- [bullet list of what the addon does]

**APIs Used:**
- [list of verified WoW APIs with wiki links]

**Installation:**
Copy the `[Name]` folder to `World of Warcraft/_retail_/Interface/AddOns/`

**Secret Values:** [Yes/No — does it handle combat data?]
**Addon Comms:** [Yes/No — does it communicate?]
```

## CRITICAL RULES

- NEVER use deprecated functions. Always use the C_ namespace replacements.
- NEVER hallucinate API functions. If unsure, research first.
- ALWAYS use the namespace pattern (`local addonName, ns = ...`)
- ALWAYS use the event dispatch table pattern
- ALWAYS guard combat data with `issecretvalue()` if the addon operates during encounters
- ALWAYS set Interface to `120001`
- NEVER use globals without declaring them in the TOC or a shared namespace
- PREFER `RegisterEventCallback()` for simple event handling (new in 12.0)
