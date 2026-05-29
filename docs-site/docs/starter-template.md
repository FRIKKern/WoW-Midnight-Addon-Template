---
title: "Starter Template & Pro Showcase"
description: "The perfect Midnight 12.0+ addon template, how the pros structure their addons, modern toolchain setup, and AI-assisted development workflow."
---

# Starter Template & Pro Showcase

Everything you need to go from zero to published addon. A production-ready template, real-world examples from top addons, modern tooling, and an AI-friendly workflow that lets you build faster than ever.

---

## 1. The Perfect Starter Template

No existing addon template targets Midnight 12.0+. Ours does. It ships with the correct Interface version, Secret Values awareness, CI/CD, linting, AI instructions, and every file a modern addon needs.

### What's Included (and Why)

| File | Purpose |
|------|---------|
| `MyAddon.toc` | Addon manifest — Interface `120001`, SavedVariables, load order |
| `Init.lua` | First-loaded file — namespace setup, table-dispatch event dispatcher, SavedVariables init, `ADDON_LOADED`/`PLAYER_LOGIN` handlers, slash commands |
| `Core.lua` | Feature logic, frame creation, combat queue, secret value helpers |
| `Config.lua` | Settings panel registration (modern Blizzard Settings API) |
| `Libs/` | Library loader (LibStub, CallbackHandler) |
| `.pkgmeta` | BigWigsMods/packager config — library externals, packaging rules |
| `.luacheckrc` | Luacheck config with WoW API globals |
| `.luarc.json` | VS Code LuaLS config for IntelliSense |
| `.editorconfig` | Consistent tabs, line endings, charset |
| `.github/workflows/release.yml` | Tag push auto-publishes to CurseForge + Wago + GitHub |
| `.github/workflows/lint.yml` | Luacheck runs on every push and PR |
| `CLAUDE.md` | AI instructions — 12.0 API rules, deprecated functions, patterns |
| `.cursorrules` | Cursor IDE instructions (mirrors CLAUDE.md) |

### Full File Tree

```
MyAddon/
├── .github/
│   └── workflows/
│       ├── release.yml          # Tag → package → upload everywhere
│       └── lint.yml             # Luacheck on push/PR
├── .vscode/
│   ├── extensions.json          # Recommends ketho.wow-api + StyLua
│   └── settings.json            # LuaLS Lua 5.1 config
├── Libs/                        # Populated by .pkgmeta at build time
│   ├── LibStub/                 # Library version registry
│   └── CallbackHandler-1.0/     # Event callback library
├── MyAddon.toc                  # Interface: 120001 (loads Init → Core → Config)
├── Init.lua                     # Namespace + event dispatch + ADDON_LOADED/PLAYER_LOGIN + slash commands
├── Core.lua                     # Feature logic + frame creation + combat queue
├── Config.lua                   # Settings panel registration
├── CLAUDE.md                    # AI coding instructions
├── .cursorrules                 # Cursor IDE instructions
├── .luacheckrc                  # Lua linter config
├── .luarc.json                  # VS Code language server config
├── .pkgmeta                     # Packager config + library externals
├── .editorconfig                # Editor settings
├── .gitignore                   # Ignores .release/, Libs/, *.zip
├── CHANGELOG.md
├── LICENSE
└── README.md
```

### File Walk-Through

#### MyAddon.toc

The manifest that tells WoW how to load your addon.

```toc
## Interface: 120001
## Title: MyAddon
## Notes: A brief description of what your addon does
## Author: YourName
## Version: @project-version@
## SavedVariables: MyAddonDB
## OptionalDeps: LibStub, CallbackHandler-1.0, Ace3
## IconTexture: Interface\AddOns\MyAddon\icon
## X-Curse-Project-ID: 000000
## X-Wago-ID: aBcDeFgH

#@no-lib-strip@
Libs\LibStub\LibStub.lua
Libs\CallbackHandler-1.0\CallbackHandler-1.0.lua
#@end-no-lib-strip@

Init.lua
Core.lua
Config.lua
```

!!! info "`@project-version@`"
    Replaced with your git tag at build time by BigWigsMods/packager. Tag `v1.0.0` → Version shows as `v1.0.0`.

!!! info "`#@no-lib-strip@`"
    Tells the packager to strip library loading lines from the `-nolib` zip variant. Users who have standalone library addons installed won't double-load.

#### Init.lua — Namespace, Events & Slash Commands

`Init.lua` is loaded **first** (after libraries, per the `.toc`). It sets up the
shared namespace, a hand-rolled table-dispatch event dispatcher, SavedVariables
init, and the slash commands. No `EventUtil.ContinueOnAddOnLoaded` — the template
uses a plain `ADDON_LOADED` handler on a hidden event frame so the pattern stays
transparent and dependency-free.

```lua
local addonName, ns = ...

-- ─── Secret Value helper (Midnight 12.0) ─────────────────
ns.SECRETS_ENABLED = type(issecretvalue) == "function"
function ns.SafeValue(val, fallback)
    if ns.SECRETS_ENABLED and issecretvalue(val) then return fallback end
    return val
end

-- ─── Table-dispatch event dispatcher ─────────────────────
local eventFrame = CreateFrame("Frame")
local eventHandlers = {}
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = eventHandlers[event]
    if handler then handler(self, event, ...) end
end)
local function RegisterEvent(event, handler)
    eventHandlers[event] = handler
    eventFrame:RegisterEvent(event)
end
ns.RegisterEvent = RegisterEvent

-- ─── ADDON_LOADED: SavedVariables init + slash commands ──
RegisterEvent("ADDON_LOADED", function(self, event, loadedAddon)
    if loadedAddon ~= addonName then return end
    eventFrame:UnregisterEvent("ADDON_LOADED")

    MyAddonDB = MyAddonDB or {}
    for key, defaultValue in pairs(ns.defaults) do
        if MyAddonDB[key] == nil then MyAddonDB[key] = defaultValue end
    end
    ns.db = MyAddonDB

    SLASH_MYADDON1 = "/myaddon"
    SlashCmdList["MYADDON"] = function(input)
        local cmd = strlower(strtrim(input or ""))
        if cmd == "config" or cmd == "options" then
            if ns.settingsCategoryID then Settings.OpenToCategory(ns.settingsCategoryID) end
        elseif cmd == "toggle" then
            ns.db.enabled = not ns.db.enabled
        elseif cmd == "reset" then
            for key, value in pairs(ns.defaults) do ns.db[key] = value end
        else
            print("/myaddon config — Open settings")
            print("/myaddon toggle — Enable/disable")
            print("/myaddon reset  — Reset settings to defaults")
        end
    end

    if ns.RegisterSettings then ns:RegisterSettings() end
end)

-- ─── PLAYER_LOGIN: start features once the world is ready ─
RegisterEvent("PLAYER_LOGIN", function()
    if ns.db.enabled and ns.Enable then ns:Enable() end
end)
```

!!! info "Why `PLAYER_LOGIN`, not `PLAYER_ENTERING_WORLD`?"
    `PLAYER_LOGIN` fires exactly once per session. `PLAYER_ENTERING_WORLD` fires on
    every loading screen (portals, instances). Create UI in `PLAYER_LOGIN`.

#### Core.lua — Feature Logic & Combat Safety

`Core.lua` holds the addon's actual behaviour — frame creation, hooks, and a
combat-deferral queue. It reads `ns.db` (set up by `Init.lua`) and exposes
`ns:Enable()`/`ns:Disable()`.

```lua
local addonName, ns = ...

-- ─── Combat deferral ─────────────────────────────────────
ns.combatQueue = {}
function ns.AfterCombat(fn)
    if InCombatLockdown() then
        ns.combatQueue[#ns.combatQueue + 1] = fn
    else
        fn()
    end
end

ns.RegisterEvent("PLAYER_REGEN_ENABLED", function()
    for _, fn in ipairs(ns.combatQueue) do fn() end
    wipe(ns.combatQueue)
end)

function ns:Enable()
    -- Build frames, register gameplay events, start timers.
end

function ns:Disable()
    -- Tear down / hide what Enable created.
end
```

#### Config.lua — Settings Panel

`Config.lua` registers the addon's options page with the modern Blizzard Settings
API and wires up the Addon Compartment callbacks. SavedVariables defaults live in
`ns.defaults` (defined in `Init.lua`) — `Config.lua` only builds the UI.

```lua
local addonName, ns = ...

-- ─── Settings panel (Blizzard Settings API) ──────────────
function ns:RegisterSettings()
    local category = Settings.RegisterCanvasLayoutCategory(
        ns.CreateSettingsPanel(), addonName
    )
    Settings.RegisterAddOnCategory(category)
    ns.settingsCategoryID = category:GetID()
end
```

#### .pkgmeta — Library Externals

```yaml
package-as: MyAddon
enable-nolib-creation: yes

externals:
  Libs/LibStub:
    url: https://repos.curseforge.com/wow/libstub/trunk
    tag: latest
  Libs/CallbackHandler-1.0:
    url: https://repos.curseforge.com/wow/callbackhandler/trunk/CallbackHandler-1.0
    tag: latest

ignore:
  - .github
  - .vscode
  - .luacheckrc
  - .luarc.json
  - .editorconfig
  - .cursorrules
  - CLAUDE.md
  - README.md
  - CHANGELOG.md
```

#### .github/workflows/release.yml

```yaml
name: Package and release

on:
  push:
    tags:
      - "v*"

jobs:
  release:
    runs-on: ubuntu-latest
    env:
      CF_API_KEY: ${{ secrets.CF_API_KEY }}
      WOWI_API_TOKEN: ${{ secrets.WOWI_API_TOKEN }}
      WAGO_API_TOKEN: ${{ secrets.WAGO_API_TOKEN }}
      GITHUB_OAUTH: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for changelog generation
      - uses: BigWigsMods/packager@v2
```

#### .github/workflows/lint.yml

```yaml
name: Lint

on:
  push:
    branches: [main]
  pull_request:

jobs:
  luacheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: nebularg/actions-luacheck@v1
        with:
          args: "--no-color -q"
          annotate: warning
```

#### .luacheckrc

```lua
std = "lua51"
max_line_length = false
exclude_files = { "Libs/", ".release/" }

ignore = {
    "11./SLASH_",     -- Setting slash command globals
    "212/self",       -- Unused self in mixin methods
    "212/event",      -- Unused event arg
}

globals = {
    "MyAddonDB",
    "SLASH_MYADDON1",
    "SlashCmdList",
}

read_globals = {
    "LibStub",
    "CreateFrame", "UIParent", "GameTooltip",
    "C_Timer", "C_AddOns", "C_EditMode",
    "Mixin", "CreateFromMixins", "BackdropTemplateMixin",
    "EventUtil", "EventRegistry", "Settings",
    "InCombatLockdown", "GetBuildInfo", "GetLocale",
    "hooksecurefunc", "issecretvalue",
    "WOW_PROJECT_ID", "WOW_PROJECT_MAINLINE",
    "strlower", "strtrim", "wipe", "CopyTable",
    "print", "format", "select", "type", "unpack",
}
```

!!! tip "Auto-generated luacheckrc"
    [Jayrgo/wow-luacheckrc](https://github.com/Jayrgo/wow-luacheckrc) auto-generates a complete `.luacheckrc` from Blizzard's own interface resources, covering every known WoW global. Far more comprehensive than hand-maintained configs.

---

## 2. Showcase: How The Pros Do It

### BetterBags — Modern Clean Architecture

[:material-github: Cidan/BetterBags](https://github.com/Cidan/BetterBags) | MIT License | Midnight Ready

```
BetterBags/
├── .github/workflows/          # release + alpha + luacheck + busted tests + claude
├── CLAUDE.md, AGENTS.md        # AI context files
├── BetterBags.toc              # Interface: 120000, 120001
├── BetterBags_Vanilla.toc, _TBC.toc, _Mists.toc
├── annotations.lua             # LuaCATS type annotations throughout
├── core/
│   ├── boot.lua                # FIRST loaded — AceAddon:NewAddon + SetDefaultModuleState(false)
│   ├── init.lua                # LAST loaded — OnInitialize/OnEnable
│   ├── async.lua               # Coroutine batch rendering (frame-rate aware)
│   ├── context.lua             # Go-style context system
│   ├── database.lua            # AceDB wrapper with migration
│   ├── events.lua              # Custom event layer (bucketing, deferred send)
│   └── pool.lua                # Object pooling
├── data/                       # Data layer (items, categories, refresh)
├── frames/                     # UI components (~25 files)
├── views/                      # View layer (bag, grid)
├── themes/                     # Pluggable themes (default, simpledark, gw2, elvui)
├── integrations/               # Masque, ConsolePort, Pawn
├── config/                     # Custom form framework (NOT AceConfig)
├── spec/                       # Unit tests with busted
└── examples/plugin/            # Plugin example for third-party devs
```

**What makes it great:**

- **Modules start disabled** — `SetDefaultModuleState(false)`, enabled selectively in `OnEnable`
- **Coroutine batch rendering** — adapts batch size to framerate, yields between batches
- **Go-style contexts** — passed as first arg to every callback for cancellation/timeout
- **AI-ready** — ships CLAUDE.md and AGENTS.md, has a `claude.yml` workflow
- **Theme registry** — pluggable themes let users completely restyle the UI
- **Unit tests** — busted test framework runs in CI

**Key patterns to steal:** Boot/Init split, debounced refresh with combat deferral, hidden parent taint isolation, object pooling.

---

### Cell — Gold Standard Raid Frames

[:material-github: enderneko/Cell](https://github.com/enderneko/Cell) | 186 stars | Interface `120001`

```
Cell/
├── Cell.toc + 5 flavor TOCs
├── Core.lua                    # Single global: _G.Cell = select(2, ...)
├── Utils.lua, Revise.lua       # Utilities + DB migration
├── HideBlizzard.lua            # Hides native raid frames
├── Libs/
│   ├── CallbackHandler.lua     # Cell's OWN lightweight callback (not Ace3)
│   ├── PixelPerfect.lua        # Own pixel-perfect helper
│   └── [External libs via .pkgmeta]
├── Defaults/                   # Settings defaults per category, per flavor
├── Indicators/                 # Visual indicators (9 files)
├── RaidFrames/                 # Core: MainFrame, UnitButton, Groups/
├── Modules/                    # Options UI
├── RaidDebuffs/                # Per-expansion debuff data
├── Utilities/                  # BattleRes, BuffTracker, DeathReport, Marks
├── Widgets/                    # Animation, ColorPicker, Tooltip
├── Locales/                    # 11 languages
└── .snippets/                  # 30+ user code snippet examples
```

**What makes it great:**

- **Zero framework dependency** — custom namespace, custom callbacks, custom SavedVariables. No Ace3.
- **Self-dispatching events** — `eventFrame[event](self, ...)` pattern for zero-allocation dispatch
- **Custom callback system** — `Cell.Fire("UpdateLayout")` pub/sub without library overhead
- **Per-expansion data** — RaidDebuffs split by expansion for easy maintenance
- **6 flavor TOCs** with flavor-specific Lua files

**Key patterns to steal:** Custom lightweight callback system, self-dispatching event frame, SavedVariables validation without AceDB.

---

### OmniCC — Minimal & Modern

[:material-github: tullamods/OmniCC](https://github.com/tullamods/OmniCC) | Secret Values handling

```
OmniCC/
├── OmniCC/                     # Main addon (monorepo)
│   ├── OmniCC.toc              # Interface: 110207, 50503, 38000, 20505, 11508
│   ├── main.lua                # Entry point — EventUtil, no ADDON_LOADED frame!
│   ├── core/
│   │   ├── cooldown.lua        # Hooks Cooldown widget METATABLE globally
│   │   ├── timer.lua           # Timer pool with subscriber pattern
│   │   ├── display.lua         # FontString displays
│   │   └── types.lua           # LuaCATS annotations
│   ├── effects/                # Finish effects (alert, flare, pulse, shine)
│   ├── settings/               # Themes + pattern-matching rules
│   └── localization/
├── OmniCC_Config/              # Separate LoadOnDemand config addon
│   ├── OmniCC_Config.toc       # LoadOnDemand: 1
│   └── options.lua, preview.lua, rules.lua, themes.lua
└── .pkgmeta                    # move-folders separates addons at build time
```

**What makes it great:**

- **Modern EventUtil init** — `EventUtil.ContinueOnAddOnLoaded()` instead of the traditional ADDON_LOADED frame
- **Metatable-level hooks** — hooks ALL cooldown widgets by hooking the Cooldown widget metatable
- **Secret Values handling** — detects `SECRETS_ENABLED`, uses proxy frames to reverse-engineer durations
- **LoadOnDemand config** — settings UI only loads when opened, reducing memory
- **Monorepo with `move-folders`** — two addons in one repo, separated at build time

**Key patterns to steal:** EventUtil initialization, metatable hooking, LoadOnDemand config addon, Secret Values detection.

---

### EditModeExpanded — Edit Mode Library

[:material-github: teelolws/EditModeExpanded](https://github.com/teelolws/EditModeExpanded) | Interface `120001`

```
EditModeExpanded/
├── Source/                     # Moved to root at build time
│   ├── EditModeExpanded.toc
│   ├── EditModeExpanded.lua    # Main init — EventUtil throughout
│   ├── FrameHandlers/          # One file per frame (35 handlers!)
│   │   ├── ActionBars.lua, Buffs.lua, ChatFrame.lua, Housing.lua
│   │   ├── Minimap.lua, ObjectiveTracker.lua, Pet.lua, Tooltip.lua, ...
│   ├── RegisterFrame.lua       # Core frame registration API
│   ├── CombatManager.lua       # continueAfterCombatEnds() utility
│   ├── HookOnce.lua            # One-time hook utilities
│   └── libs/
│       └── EditModeExpanded-1.0/  # Reusable library for other addons
```

**What makes it great:**

- **One handler per frame** — 35 clean files in `FrameHandlers/`, easy to find and modify
- **Reusable library** — `EditModeExpanded-1.0` via LibStub, other addons can register frames
- **EventUtil everywhere** — zero manual frame event registration
- **Lazy Blizzard addon loading** — `EventUtil.ContinueOnAddOnLoaded("Blizzard_AuctionHouseUI", ...)`
- **`hookScriptOnce` / `hookFuncOnce`** — clean one-time hooks

**Key patterns to steal:** File-per-frame organization, reusable library pattern, one-time hook utilities.

---

### AdiBags — Modular Bag Addon

[:material-github: AdiAddons/AdiBags](https://github.com/AdiAddons/AdiBags) | Predecessor to BetterBags

```
AdiBags/
├── AdiBags.toc + 4 flavor TOCs
├── core/
│   ├── EventHandlers.lua       # Loaded FIRST — creates custom event libraries
│   ├── Boot.lua                # AceAddon:NewAddon — starts DISABLED
│   ├── Core.lua, Hooks.lua, Layout.lua, Filters.lua
│   ├── ItemDatabase.lua, Theme.lua, OO.lua
├── modules/                    # 12 feature modules (Masque, Junk, ItemLevel, ...)
├── widgets/                    # UI: ItemButton, Section, ContainerFrame, grid/
├── config/                     # Options
├── AdiBags_Config/             # Separate LoadOnDemand config addon
└── tests/
```

**What makes it great:**

- **Custom event libraries** — `ABEvent-1.0` and `ABBucket-1.0` via LibStub, reusable by any addon
- **Starts disabled** — `SetEnabledState(false)`, enables on `PLAYER_ENTERING_WORLD`, disables on `PLAYER_LEAVING_WORLD`
- **Plugin system** — filter plugins register categorization rules: `AdiBags:RegisterFilter("MyFilter", priority)`
- **Event buckets** — `self:RegisterBucketEvent("BAG_UPDATE", 0.01, "BagUpdated")` for debouncing

**Key patterns to steal:** Custom event libraries via LibStub, delayed enablement, plugin/filter system.

---

### Plater — Nameplate Powerhouse

[:material-github: Tercioo/Plater-Nameplates](https://github.com/Tercioo/Plater-Nameplates) | Interface `120001`

```
Plater/
├── .github/workflows/
│   ├── CurseForge-Deployment.yml
│   └── TOCBump.yml             # Daily cron → auto-updates Interface versions!
├── Plater.lua                  # Main addon (~5000 lines, event dispatch table)
├── Plater_DefaultSettings.lua  # Massive defaults table
├── Plater_API.lua              # Public API
├── Plater_ScriptLibrary.lua    # User script sandbox
├── Plater_LoadFinished.lua     # LAST loaded — sets up load-on-demand subsystems
├── libs/
│   └── DF/                     # Details Framework — massive UI library
├── options/                    # Load-on-demand option panels
├── locales/, fonts/, images/, masks/, media/, sounds/
```

**What makes it great:**

- **Auto TOC bump** — daily cron workflow auto-updates Interface versions
- **Event dispatch table** — `handlers["NAME_PLATE_CREATED"] = function(...) end` with per-handler CPU profiling
- **User script sandbox** — lets users write custom Lua scripts that run in a controlled environment
- **Dual namespace** — `platerInternal` (private) + `Plater` (global via Details Framework)

**Key patterns to steal:** Auto TOC bump workflow, event dispatch table with profiling, user script system.

---

## 3. Template Comparison Matrix

| Feature | **Ours** | kapresoft | layday | Kkthnx | Ketho HelloWorld |
|---------|:--------:|:---------:|:------:|:------:|:----------------:|
| Interface `120001` | :white_check_mark: | :x: | :x: | :x: | :x: |
| Secret Values awareness | :white_check_mark: | :x: | :x: | :x: | :x: |
| Table-dispatch event pattern | :white_check_mark: | :x: | :x: | :x: | :x: |
| BigWigsMods/packager CI | :white_check_mark: | :white_check_mark: | :white_check_mark: | :x: | :x: |
| Luacheck linting | :white_check_mark: | :x: | :white_check_mark: | :x: | :x: |
| `.luarc.json` (VS Code) | :white_check_mark: | :x: | :x: | :x: | :x: |
| `.pkgmeta` externals | :white_check_mark: | :white_check_mark: | :white_check_mark: | :x: | :x: |
| SavedVariables + migration | :white_check_mark: | :x: | :x: | :white_check_mark: | :white_check_mark: |
| Combat deferral utilities | :white_check_mark: | :x: | :x: | :x: | :x: |
| CLAUDE.md (AI instructions) | :white_check_mark: | :x: | :x: | :x: | :x: |
| .cursorrules | :white_check_mark: | :x: | :x: | :x: | :x: |
| Multi-flavor TOC | :x: | :white_check_mark: | :x: | :x: | :x: |
| Ace3 integration | Optional | :white_check_mark: | :x: | :white_check_mark: | :x: |
| Last updated | **2026** | 2024 | 2020 | 2023 (archived) | 2025 |

**Bottom line:** Existing templates are outdated. None target Midnight, none handle Secret Values, and none include AI tooling. Our template fills every gap.

---

## 4. The AI-Friendly Addon Project

### Why Structure Matters for AI

AI tools generate correct code when they have the right context upfront. Every file in this template serves double duty:

| For Humans | For AI |
|-----------|--------|
| Documentation | Context that prevents hallucination |
| Code examples | Patterns to extrapolate from |
| Linting rules | Guardrails against deprecated APIs |
| File structure | Signals for where to put new code |

### CLAUDE.md Best Practices

The `CLAUDE.md` file is your **behavioral specification** for AI. It's the difference between Claude hallucinating removed APIs and generating working 12.0 code on the first try.

```markdown
# MyAddon

## Critical: WoW 12.0 Midnight API Rules

### DO NOT USE (Removed/Broken):
- COMBAT_LOG_EVENT_UNFILTERED — removed in 12.0
- CombatLogGetCurrentEventInfo() — removed
- UnitHealth() for math — returns secret values during combat

### Secret Values (CRITICAL):
- During M+/PvP/encounters, combat data returns as opaque tokens
- Always check: if issecretvalue(val) then return end
- Use CurveObject/DurationObject for visual display

### Interface Version:
- Current: 120001 (Patch 12.0.1)
- Addons below 120000 will NOT load

## Architecture

### File Load Order (defined in .toc):
1. Libs/ — External libraries (LibStub, CallbackHandler)
2. Init.lua — Namespace + event dispatcher + ADDON_LOADED/PLAYER_LOGIN + slash commands
3. Core.lua — Feature logic + frame creation + combat queue
4. Config.lua — Settings panel registration

### Namespace Pattern:
local addonName, ns = ...
-- All code uses ns.* to share between files
-- NEVER create globals except slash commands

## Coding Conventions

### Always:
- Use `local` for everything
- Check issecretvalue() before comparing combat values
- Check InCombatLockdown() before modifying secure frames
- Use hooksecurefunc() for post-hooks (never wrap secure functions)
- Use BAG_UPDATE_DELAYED (not BAG_UPDATE)

### Never:
- Generate CLEU handlers
- Call functions listed as removed in 12.0
- Create globals without explicit declaration in .luacheckrc
- Call SetScript() on Blizzard frames (use HookScript)
```

### Claude Agents for Addon Dev

Place these in `.claude/agents/` for specialized AI assistance:

| Agent File | Role |
|-----------|------|
| `wow-addon-coder.md` | Writes Lua code following verified 12.0 patterns. Knows Secret Values. Uses `local` everywhere. |
| `wow-addon-researcher.md` | Searches warcraft.wiki.gg for API docs. Verifies function signatures exist in 12.0. |

### The AI Workflow

```
You: "Add a minimap button that toggles the config panel"

Claude reads CLAUDE.md → knows the namespace pattern, load order, 12.0 APIs
     ↓
Claude reads Config.lua → sees existing settings panel registration
     ↓
Claude adds LibDBIcon to .pkgmeta externals
Claude adds the LibDBIcon load line to MyAddon.toc
Claude adds minimap button code to Core.lua using ns.db for toggle state
Claude updates .luacheckrc with new globals
     ↓
You: /reload → it works
```

The key insight: AI tools **extrapolate from your existing patterns**. One well-structured file teaches it how to write the next ten. A messy codebase produces messy AI output.

---

## 5. Modern Toolchain Setup

### IDE Setup (VS Code)

Install three extensions:

```
ext install sumneko.lua          # LuaLS language server
ext install ketho.wow-api        # WoW API autocomplete (8,000+ functions)
ext install stanzilla.vscode-wow-toc  # TOC file syntax highlighting
```

`ketho.wow-api` auto-activates when it detects a `.toc` file. Covers all 12.0.1 APIs with signatures, parameter types, and wiki links.

**`.vscode/settings.json`:**

```json
{
    "Lua.runtime.version": "Lua 5.1",
    "Lua.runtime.builtin": {
        "io": "disable",
        "os": "disable",
        "package": "disable",
        "debug": "disable"
    },
    "Lua.workspace.ignoreDir": ["Libs", ".release", ".github"],
    "Lua.diagnostics.disable": ["lowercase-global"],
    "Lua.workspace.checkThirdParty": true
}
```

**`.vscode/extensions.json`:**

```json
{
    "recommendations": [
        "sumneko.lua",
        "ketho.wow-api",
        "stanzilla.vscode-wow-toc",
        "johnnymorganz.stylua"
    ]
}
```

### Linting with Luacheck

```bash
# Install
brew install luacheck    # macOS
luarocks install luacheck # via luarocks

# Run
luacheck .

# In CI: nebularg/actions-luacheck@v1 (see lint.yml above)
```

!!! tip "Auto-generated configs"
    [Jayrgo/wow-luacheckrc](https://github.com/Jayrgo/wow-luacheckrc) parses Blizzard's own interface resources to generate a `.luacheckrc` covering every known WoW global. Alternatively, [nebularg/wow-selene-parser](https://github.com/nebularg/wow-selene-parser) generates configs for the Selene linter.

### Packaging with BigWigsMods/packager

The industry standard. Used by **every major addon** — BetterBags, Cell, Plater, OmniCC, WeakAuras, ElvUI, Bartender4, BigWigs, DBM.

**How it works:**

1. You push a git tag: `git tag v1.0.0 && git push --tags`
2. GitHub Actions triggers `BigWigsMods/packager@v2`
3. Packager checks out library externals from `.pkgmeta`
4. Replaces `@project-version@` tokens in your code
5. Strips `@debug@` blocks from release builds
6. Creates zip files (including `-nolib` variant)
7. Uploads to CurseForge, WoWInterface, Wago, and GitHub Releases

**Keyword substitutions** (replaced at build time):

| Keyword | Becomes |
|---------|---------|
| `@project-version@` | Your git tag (e.g., `v1.0.0`) |
| `@project-hash@` | Full git commit hash |
| `@project-date-iso@` | ISO date of last commit |
| `@debug@ ... @end-debug@` | Stripped from release builds |
| `@alpha@ ... @end-alpha@` | Active only in untagged builds |

!!! tip "Full CI/CD Guide"
    For the complete walkthrough — from creating your CurseForge project to troubleshooting failed uploads — see the [Publishing & CI/CD guide](publishing.md).

### CI/CD with GitHub Actions

**Secrets to configure** (Settings > Secrets and variables > Actions):

| Secret | Platform | Where to Generate |
|--------|----------|-------------------|
| `CF_API_KEY` | CurseForge | Author dashboard > API tokens |
| `WOWI_API_TOKEN` | WoWInterface | File management > API tokens |
| `WAGO_API_TOKEN` | Wago Addons | `addons.wago.io/account/apikeys` |
| `GITHUB_TOKEN` | GitHub Releases | Auto-provided (enable write permissions) |

!!! warning "Enable workflow write permissions"
    Settings > Actions > General > Workflow permissions > **Read and write permissions**. Required for GitHub Releases.

### Publishing to CurseForge / Wago

1. **Create the project** on CurseForge and/or Wago
2. **Get your project IDs** and add them to your TOC:
    ```toc
    ## X-Curse-Project-ID: 123456
    ## X-Wago-ID: aBcDeFgH
    ```
3. **Add API key secrets** to your GitHub repo
4. **Push a tag** — the workflow handles everything:
    ```bash
    git tag v1.0.0
    git push --tags
    ```

---

## 6. Quick Start Guide

### Step 1: Fork & Clone

```bash
# Clone the template
git clone https://github.com/FRIKKern/Better-Addons-WoW-Template.git MyAddon
cd MyAddon

# Remove template git history, start fresh
rm -rf .git
git init
git add -A
git commit -m "Initial commit from Midnight addon template"
```

### Step 2: Rename Everything

Find and replace these strings across all files:

| Find | Replace With |
|------|-------------|
| `MyAddon` | Your addon name (e.g., `CoolBags`) |
| `MYADDON` | Uppercase version (e.g., `COOLBAGS`) |
| `MyAddonDB` | Your SavedVariables name (e.g., `CoolBagsDB`) |
| `YourName` | Your author name |

**Files to rename:**

- `MyAddon.toc` → `CoolBags.toc`

### Step 3: Customize

1. **Edit the TOC** — update Title, Notes, Author
2. **Edit Init.lua** — change the slash commands and edit `ns.defaults` for your settings
3. **Edit Core.lua** — add your feature logic and build your frames in `ns:Enable()`
4. **Edit Config.lua** — wire up your settings panel
5. **Update `.luacheckrc`** — add your global names

### Step 4: Set Up Dev Environment

```bash
# macOS: symlink into WoW's AddOns folder
ln -s "$(pwd)" \
  "/Applications/World of Warcraft/_retail_/Interface/AddOns/CoolBags"
```

```powershell
# Windows (admin PowerShell):
New-Item -ItemType SymbolicLink `
  -Path "D:\Games\World of Warcraft\_retail_\Interface\AddOns\CoolBags" `
  -Value "D:\repos\CoolBags"
```

### Step 5: Test In-Game

1. Launch WoW
2. Type `/reload` to reload the UI
3. Your addon loads — check for errors with `/console scriptErrors 1`
4. Install **BugSack + BugGrabber** for proper error capture with stack traces

### Step 6: Iterate

```
Edit code → Save → Alt-Tab to WoW → /reload → Test → Repeat
```

No hot-reload exists. `/reload` restarts the full addon lifecycle.

### Step 7: Release

```bash
# Create GitHub repo
gh repo create CoolBags --public --source=. --push

# Configure secrets (CF_API_KEY, WAGO_API_TOKEN, etc.)
# Then tag and push:
git tag v1.0.0
git push --tags
```

The CI pipeline packages your addon and uploads it to CurseForge, Wago, and GitHub Releases automatically.

!!! success "You did it"
    Your addon is live. Users can install it from CurseForge or Wago. Push a new tag whenever you want to release an update.

---

## Essential Resources

| Resource | URL |
|----------|-----|
| **Warcraft Wiki API** | [warcraft.wiki.gg/wiki/World_of_Warcraft_API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) |
| **12.0 API Changes** | [warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) |
| **Blizzard FrameXML Source** | [github.com/Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source) |
| **Ketho's API Resources** | [github.com/Ketho/BlizzardInterfaceResources](https://github.com/Ketho/BlizzardInterfaceResources) |
| **VS Code WoW API Extension** | [ketho.wow-api](https://marketplace.visualstudio.com/items?itemName=ketho.wow-api) |
| **BigWigsMods Packager** | [github.com/BigWigsMods/packager](https://github.com/BigWigsMods/packager) |
| **Packager Wiki** | [github.com/BigWigsMods/packager/wiki](https://github.com/BigWigsMods/packager/wiki) |
| **WoW Addon Dev Guide (AI)** | [github.com/Amadeus-/WoWAddonDevGuide](https://github.com/Amadeus-/WoWAddonDevGuide) |
| **Copilot Agents for WoW** | [github.com/JBurlison/WoWAddonAPIAgents](https://github.com/JBurlison/WoWAddonAPIAgents) |
| **TOC Format Spec** | [warcraft.wiki.gg/wiki/TOC_format](https://warcraft.wiki.gg/wiki/TOC_format) |
