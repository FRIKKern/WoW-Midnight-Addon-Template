# TOC File Format Reference

## Overview

Every WoW addon requires a **Table of Contents** (`.toc`) file. This plain-text file tells the game client what your addon is called, which files to load, and how it should behave.

### Naming Rules

The `.toc` file **must** match its parent folder name exactly:

```
MyAddon/
├── MyAddon.toc        ✓  Folder name matches file name
├── Core.lua
└── Config.lua
```

If the names don't match, the game will not recognize the addon.

### Line Types

A `.toc` file contains three types of lines:

| Prefix | Type | Example |
|--------|------|---------|
| `##` | Metadata tag | `## Title: My Addon` |
| `#` | Comment (ignored) | `# This is a comment` |
| *(none)* | File path to load | `Core.lua` |

- Blank lines are ignored.
- File paths are relative to the addon folder and use forward slashes (`Libs/LibStub/LibStub.lua`).
- Files are loaded **in the order listed**.

---

## Interface Number

The `## Interface:` tag declares which game version your addon targets. The client uses this to flag outdated addons.

**Current value for WoW 12.0.1 (Midnight):** `120001`

### Formula

Remove the periods from the version number, padding each segment to the appropriate width:

```
Major (1 digit) . Minor (2 digits) . Patch (2 digits)
12              .  00               .  01
→ 120001
```

More examples:

| Game Version | Interface Number |
|-------------|-----------------|
| 12.0.1 (Midnight) | `120001` |
| 11.1.5 | `111500` |
| 11.0.0 (TWW) | `110000` |
| 1.15.6 (Classic Era) | `11506` |
| 4.4.1 (Cataclysm Classic) | `40401` |

### Multi-Flavor Support

You can list multiple interface numbers separated by commas to support several game flavors from a single TOC:

```
## Interface: 120001, 11506, 40401
```

---

## Metadata Fields

### Display Fields

These control how your addon appears in the in-game Addon List.

| Field | Description | Example |
|-------|-------------|---------|
| `Title` | Display name in the addon list. Supports UI escape sequences for color. | `## Title: \|cff00ff00My Addon\|r` |
| `Title-xxXX` | Localized title for a specific locale (e.g., `deDE`, `frFR`, `zhCN`). | `## Title-deDE: Mein Addon` |
| `Notes` | Short description shown in the addon list tooltip. | `## Notes: Does cool things` |
| `Notes-xxXX` | Localized notes. | `## Notes-deDE: Macht coole Sachen` |
| `Category` | Category grouping in the addon list. *(Added in 11.1.0)* | `## Category: Bags & Inventory` |
| `Group` | Groups related addons together in the list. *(Added in 11.1.0)* | `## Group: MyAddon` |
| `IconTexture` | Texture path or fileID for the addon icon. *(Added in 10.1.0)* | `## IconTexture: Interface\Icons\INV_Misc_Gear_01` |
| `IconAtlas` | Atlas name for the addon icon. *(Added in 10.1.0)* | `## IconAtlas: Warlock-ReadyShard` |

:::tip
Use `Group` to visually cluster multi-module addons (e.g., `MyAddon_Core`, `MyAddon_Config`) under one collapsible header in the addon list.
:::

### Version & Compatibility

| Field | Description | Example |
|-------|-------------|---------|
| `Interface` | Target game client version number. Required. | `## Interface: 120001` |
| `AllowLoadGameType` | Restrict loading to specific game flavors. Comma-separated. *(Added in 10.2.7)* | `## AllowLoadGameType: mainline` |
| `OnlyBetaAndPTR` | If `1`, addon only loads on Beta/PTR realms. | `## OnlyBetaAndPTR: 1` |
| `DefaultState` | Whether the addon is enabled by default. `enabled` or `disabled`. | `## DefaultState: enabled` |

**Valid values for `AllowLoadGameType`:**

| Value | Flavor |
|-------|--------|
| `mainline` | Retail / Live (including Midnight) |
| `standard` | Alias for `mainline` |
| `classic` | Classic Era (Vanilla) |
| `cata` | Cataclysm Classic |

### Loading Control

| Field | Description | Example |
|-------|-------------|---------|
| `LoadOnDemand` | If `1`, the addon is not loaded automatically at login. Must be loaded programmatically. | `## LoadOnDemand: 1` |
| `Dependencies` | Comma-separated list of addons that **must** be loaded first. Aliases: `RequiredDeps`, `Dep1`…`DepN`. | `## Dependencies: LibStub, Ace3` |
| `OptionalDeps` | Comma-separated addons to load before this one **if available**. Does not require them. | `## OptionalDeps: LibSharedMedia-3.0` |
| `LoadWith` | For LoadOnDemand addons: auto-load when **any** listed addon loads. | `## LoadWith: Blizzard_CombatLog` |
| `LoadManagers` | Names addons that manage loading this LoadOnDemand addon. | `## LoadManagers: AddonLoader` |

:::note
`Dependencies`, `RequiredDeps`, and `Dep1`/`Dep2`/etc. are all aliases. Use whichever style you prefer, but `Dependencies` is the most common modern convention.
:::

### Saved Variables

| Field | Description | Storage Location |
|-------|-------------|-----------------|
| `SavedVariables` | Account-wide saved variables (global tables persisted between sessions). | `WTF/<account>/SavedVariables/<AddonName>.lua` |
| `SavedVariablesPerCharacter` | Per-character saved variables. | `WTF/<account>/<realm>/<character>/SavedVariables/<AddonName>.lua` |
| `LoadSavedVariablesFirst` | If `1`, saved variables are loaded before the addon's Lua files execute. *(Added in 11.1.5)* | — |

```
## SavedVariables: MyAddonDB
## SavedVariablesPerCharacter: MyAddonCharDB
```

:::warning
Variable names listed here become **global Lua variables**. Choose unique, descriptive names to avoid collisions (e.g., `MyAddonDB` not `db`).
:::

:::tip
`LoadSavedVariablesFirst` (11.1.5+) lets you read saved data immediately in your top-level Lua code instead of waiting for the `ADDON_LOADED` event. This simplifies initialization significantly.
:::

### Addon Compartment (Minimap Button)

*(Added in 10.1.0)* — These fields register your addon in the **Addon Compartment** dropdown attached to the minimap, without needing a LibDataBroker launcher or custom minimap button.

| Field | Description | Example |
|-------|-------------|---------|
| `AddonCompartmentFunc` | Global function name called on click. | `## AddonCompartmentFunc: MyAddon_OnClick` |
| `AddonCompartmentFuncOnEnter` | Global function name called on mouse-enter (tooltip). | `## AddonCompartmentFuncOnEnter: MyAddon_OnEnter` |
| `AddonCompartmentFuncOnLeave` | Global function name called on mouse-leave. | `## AddonCompartmentFuncOnLeave: MyAddon_OnLeave` |

```lua
-- These must be global functions
function MyAddon_OnClick(addonName, mouseButton)
    Settings.OpenToCategory("MyAddon")
end

function MyAddon_OnEnter(addonName, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("My Addon")
    GameTooltip:AddLine("Click to open settings", 1, 1, 1)
    GameTooltip:Show()
end

function MyAddon_OnLeave(addonName, menuButtonFrame)
    GameTooltip:Hide()
end
```

### Informational Fields

| Field | Description | Example |
|-------|-------------|---------|
| `Author` | Addon author name. | `## Author: YourName` |
| `Version` | Addon version string. | `## Version: 1.0.0` |
| `X-*` | Custom metadata. Any `X-` prefixed tag is stored and queryable at runtime. | `## X-Website: https://example.com` |

Custom `X-*` metadata can be retrieved in Lua:

```lua
local website = C_AddOns.GetAddOnMetadata("MyAddon", "X-Website")
-- Returns: "https://example.com"
```

---

## Per-File Directives

*(Added in 11.1.5)*

Individual file entries can include **inline directives** in square brackets to control when that file is loaded:

```
Core.lua
Retail.lua [AllowLoadGameType mainline]
Classic.lua [AllowLoadGameType classic]
Locale_deDE.lua [AllowLoadTextLocale deDE]
Locale_frFR.lua [AllowLoadTextLocale frFR]
```

### Available Directives

| Directive | Description | Example |
|-----------|-------------|---------|
| `AllowLoadGameType` | Only load this file on the specified game flavor. | `[AllowLoadGameType mainline]` |
| `AllowLoadTextLocale` | Only load this file for the specified text locale. | `[AllowLoadTextLocale deDE]` |

### Variable Expansions

File paths support variable expansion for dynamic file loading:

| Variable | Expands To | Example |
|----------|-----------|---------|
| `[Family]` | Game family (`Mainline`, `Classic`, etc.) | `Config_[Family].lua` |
| `[Game]` | Specific game type | `Compat_[Game].lua` |
| `[TextLocale]` | Current client locale (`enUS`, `deDE`, etc.) | `Locale_[TextLocale].lua` |

```
# Automatically loads the right locale file
Locales/Locale_[TextLocale].lua
```

---

## Client-Specific TOC Suffixes

You can ship **multiple TOC files** with client-specific suffixes. The game loads the most specific matching TOC, falling back to the base TOC if no suffix matches.

| Suffix | Game Flavor |
|--------|-------------|
| `_Mainline.toc` | Retail / Midnight |
| `_Classic.toc` | Classic Era |
| `_Cata.toc` | Cataclysm Classic |

```
MyAddon/
├── MyAddon.toc              # Fallback / shared
├── MyAddon_Mainline.toc     # Retail-specific
├── MyAddon_Classic.toc      # Classic Era-specific
├── Core.lua
└── RetailFeatures.lua
```

The client selects the most specific TOC available. If `MyAddon_Mainline.toc` exists and you're on retail, it is used instead of `MyAddon.toc`.

---

## Complete Example TOC

```toc
## Interface: 120001
## Title: My Awesome Addon
## Title-deDE: Mein Tolles Addon
## Notes: A feature-rich addon for WoW Midnight.
## Notes-deDE: Ein funktionsreiches Addon für WoW Midnight.
## Author: YourName
## Version: 1.0.0
## Category: Combat
## Group: MyAwesomeAddon
## IconTexture: Interface\Icons\INV_Misc_Gear_01

## DefaultState: enabled
## AllowLoadGameType: mainline

## SavedVariables: MyAwesomeAddonDB
## SavedVariablesPerCharacter: MyAwesomeAddonCharDB

## Dependencies: LibStub
## OptionalDeps: LibSharedMedia-3.0, Ace3

## AddonCompartmentFunc: MyAwesomeAddon_OnCompartmentClick
## AddonCompartmentFuncOnEnter: MyAwesomeAddon_OnCompartmentEnter
## AddonCompartmentFuncOnLeave: MyAwesomeAddon_OnCompartmentLeave

## X-Website: https://www.curseforge.com/wow/addons/my-awesome-addon
## X-License: MIT
## X-Category: Combat

# Libraries
Libs/LibStub/LibStub.lua
Libs/CallbackHandler-1.0/CallbackHandler-1.0.lua

# Localization
Locales/enUS.lua
Locales/deDE.lua [AllowLoadTextLocale deDE]
Locales/frFR.lua [AllowLoadTextLocale frFR]

# Core
Core.lua
Database.lua
Events.lua

# Modules
Modules/CombatTracker.lua
Modules/Alerts.lua

# UI
UI/MainFrame.lua
UI/MainFrame.xml
UI/Settings.lua
```

---

## Common Patterns

### Multi-Module Addon

Large addons like **Deadly Boss Mods (DBM)** split into a core addon and many LoadOnDemand encounter packs:

**Core addon** (`DBM-Core/DBM-Core.toc`) — always loaded:

```toc
## Interface: 120001
## Title: DBM - Core
## SavedVariables: DBM_AllSavedOptions

Core.lua
Timers.lua
```

**Encounter pack** (`DBM-Raid-Midnight/DBM-Raid-Midnight.toc`) — loaded on demand:

```toc
## Interface: 120001
## Title: DBM - Midnight Raids
## LoadOnDemand: 1
## Dependencies: DBM-Core
## LoadWith: Blizzard_EncounterJournal

MidnightRaid/Boss1.lua
MidnightRaid/Boss2.lua
```

The core addon calls `C_AddOns.LoadAddOn("DBM-Raid-Midnight")` when the player enters the relevant raid, or the `LoadWith` tag triggers it automatically when the Encounter Journal opens.

### LoadOnDemand Config Panel

Keep your configuration UI in a separate addon that loads only when the player opens settings:

```toc
## Interface: 120001
## Title: MyAddon - Options
## LoadOnDemand: 1
## Dependencies: MyAddon
## LoadWith: MyAddon

OptionsPanel.lua
OptionsPanel.xml
```

In your main addon, register the options panel as loadable:

```lua
-- In MyAddon's Core.lua
Settings.RegisterAddOnCategory(Settings.CreateCategory("MyAddon"))
```

### Library Embedding with OptionalDeps

When embedding libraries (bundling them inside your addon), use `OptionalDeps` to ensure proper load order if the library is also standalone:

```toc
## OptionalDeps: LibStub, CallbackHandler-1.0, LibSharedMedia-3.0

# Embedded libraries (loaded from your addon folder)
Libs/LibStub/LibStub.lua
Libs/CallbackHandler-1.0/CallbackHandler-1.0.lua
Libs/LibSharedMedia-3.0/lib.xml
```

`OptionalDeps` ensures that if a standalone version of the library exists and is enabled, it loads **before** your addon — so its newer version takes precedence over your embedded copy (when using LibStub's version negotiation).
