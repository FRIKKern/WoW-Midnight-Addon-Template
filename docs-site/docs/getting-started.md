# Getting Started with WoW Addon Development

This guide walks you through creating your first World of Warcraft addon from scratch, covering every fundamental concept you need to build real addons for Patch 12.0+ (Midnight).

---

## Prerequisites

Before writing your first addon, make sure you have:

### World of Warcraft Installed

You need a WoW installation (Retail, PTR, or Beta). Your addon files go in:

| Platform | Path |
|----------|------|
| **Windows** | `C:\Program Files (x86)\World of Warcraft\_retail_\Interface\AddOns\` |
| **macOS** | `/Applications/World of Warcraft/_retail_/Interface/AddOns/` |

!!! tip "Finding your WoW folder"
    Open the Battle.net launcher, click the gear icon next to WoW, and select **Show in Explorer/Finder** to locate your install directory.

### VS Code + Extensions

[Visual Studio Code](https://code.visualstudio.com/) is the recommended editor. Install these extensions for WoW addon development:

| Extension | ID | Purpose |
|---|---|---|
| **WoW API** | `ketho.wow-api` | Autocomplete and type annotations for the WoW API |
| **WoW Bundle** | `Septh.wow-bundle` | Syntax highlighting for TOC, XML, and WoW Lua files |
| **WoW TOC Language** | `stanzilla.vscode-wow-toc` | TOC file syntax support and validation |

!!! note "Lua Language Server"
    The `ketho.wow-api` extension works with [sumneko's Lua Language Server](https://marketplace.visualstudio.com/items?itemName=sumneko.lua) (`sumneko.lua`). Install it alongside for full IntelliSense, diagnostics, and go-to-definition support.

---

## Your First Addon

Let's create a minimal addon called **MyFirstAddon** that prints a message when you log in.

### Step 1: Create the Addon Folder

Create a new folder inside your `Interface/AddOns/` directory:

```
Interface/AddOns/MyFirstAddon/
```

!!! warning "Folder name rules"
    - No spaces or special characters (use letters, numbers, underscores)
    - The folder name **is** your addon's identity — it must match the TOC filename exactly

### Step 2: Create the TOC File

The **Table of Contents** file tells WoW about your addon. Create `MyFirstAddon.toc`:

```toc
## Interface: 120001
## Title: My First Addon
## Notes: A simple hello world addon
## Author: YourName
## Version: 1.0.0

MyFirstAddon.lua
```

- `Interface: 120001` — targets Patch 12.0.x (Midnight). See the [TOC Format](toc-format.md) reference for details.
- The last section lists files to load, **in order**, one per line.

### Step 3: Create the Lua File

Create `MyFirstAddon.lua`:

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:SetScript("OnEvent", function(self, event)
    print("Hello World! MyFirstAddon is loaded.")
end)
```

### Step 4: Load It In-Game

1. Launch WoW (or type `/reload` if already in-game)
2. Open **AddOns** from the character select screen and verify **My First Addon** is listed and enabled
3. Log in — you should see the message in your chat frame

!!! success "Congratulations!"
    You've built your first working WoW addon. The rest of this guide covers the patterns you'll use in every real addon.

---

## The Namespace Pattern

Every Lua file in an addon receives two implicit arguments via the `...` vararg:

```lua
local addonName, ns = ...
```

| Variable | Type | Description |
|----------|------|-------------|
| `addonName` | `string` | The folder name of your addon (e.g., `"MyFirstAddon"`) |
| `ns` | `table` | A **shared private table** — the same table is passed to every file in your addon |

The namespace table (`ns`) is the standard way to share data between your addon's files without polluting the global scope:

```lua
-- In Core.lua
local addonName, ns = ...

ns.VERSION = "1.0.0"
ns.DEBUG = false

function ns:Print(msg)
    print("|cff00ccff" .. addonName .. ":|r " .. msg)
end
```

```lua
-- In Events.lua (loaded after Core.lua in the TOC)
local addonName, ns = ...

-- Access shared data from Core.lua
ns:Print("Events module loaded! Version: " .. ns.VERSION)
```

!!! info "Why not globals?"
    Global variables are shared across **all** addons. Using them risks name collisions — if two addons both define a global `function Init()`, one silently overwrites the other. The namespace table is private to your addon and collision-free.

---

## The Event System

The WoW API is **event-driven**. The game fires events when things happen (player logs in, health changes, bags update), and your addon registers to listen for the ones it cares about.

### Basic Pattern

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("PLAYER_LOGOUT")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_LOGIN" then
        print("Logged in!")
    elseif event == "PLAYER_LOGOUT" then
        print("Logging out!")
    end
end)
```

### Table Dispatch Pattern

The `if/elseif` chain gets unwieldy fast. The **table dispatch** pattern is the idiomatic solution:

```lua
local addonName, ns = ...

local frame = CreateFrame("Frame")
local events = {}

function events:PLAYER_LOGIN()
    ns:Print("Welcome back!")
end

function events:PLAYER_ENTERING_WORLD(isInitialLogin, isReloadingUi)
    if isInitialLogin then
        ns:Print("First login this session.")
    elseif isReloadingUi then
        ns:Print("UI reloaded.")
    end
end

function events:ADDON_LOADED(loadedAddonName)
    if loadedAddonName == addonName then
        ns:Print("Addon loaded successfully.")
        frame:UnregisterEvent("ADDON_LOADED")
    end
end

frame:SetScript("OnEvent", function(self, event, ...)
    if events[event] then
        events[event](self, ...)
    end
end)

-- Register all events defined in the table
for event in pairs(events) do
    frame:RegisterEvent(event)
end
```

!!! tip "Why table dispatch?"
    - Each event handler is a clean, separate function
    - Adding a new event is just defining a new function — no touching the dispatcher
    - Events auto-register from the table keys — you can't forget to register one

---

## Slash Commands

Slash commands let players interact with your addon from the chat box.

```lua
local addonName, ns = ...

-- Define one or more slash prefixes
SLASH_MYFIRSTADDON1 = "/mfa"
SLASH_MYFIRSTADDON2 = "/myfirstaddon"

SlashCmdList["MYFIRSTADDON"] = function(msg)
    local command, rest = msg:match("^(%S*)%s*(.-)$")
    command = command:lower()

    if command == "config" then
        ns:Print("Opening config...")
    elseif command == "reset" then
        ns:Print("Settings reset to defaults.")
    elseif command == "" then
        ns:Print("Usage: /mfa config | reset")
    else
        ns:Print("Unknown command: " .. command)
    end
end
```

!!! warning "Naming convention"
    The global `SLASH_MYFIRSTADDON1` and the key in `SlashCmdList["MYFIRSTADDON"]` must use the **same identifier** (here `MYFIRSTADDON`). The number suffix (`1`, `2`, ...) defines alternate prefixes for the same command.

---

## Saved Variables

Saved variables let your addon persist data between sessions. WoW saves them to disk when the player logs out and loads them back when the addon loads.

### Step 1: Declare in the TOC

```toc
## Interface: 120001
## Title: My First Addon
## SavedVariables: MyFirstAddonDB
## SavedVariablesPerCharacter: MyFirstAddonCharDB

MyFirstAddon.lua
```

| Directive | Scope |
|-----------|-------|
| `SavedVariables` | Shared across **all** characters on the account |
| `SavedVariablesPerCharacter` | Unique to **each** character |

### Step 2: Initialize with Defaults on ADDON_LOADED

The saved variable globals don't exist until WoW loads them. Initialize them with defaults in `ADDON_LOADED`:

```lua
local addonName, ns = ...

local DEFAULTS = {
    welcomeMessage = true,
    scale = 1.0,
    position = { x = 0, y = 0 },
}

local CHAR_DEFAULTS = {
    enabled = true,
}

local function InitializeDefaults(saved, defaults)
    for key, value in pairs(defaults) do
        if saved[key] == nil then
            if type(value) == "table" then
                saved[key] = CopyTable(value)
            else
                saved[key] = value
            end
        end
    end
end

function events:ADDON_LOADED(loadedAddonName)
    if loadedAddonName ~= addonName then return end

    -- Initialize account-wide saved variables
    MyFirstAddonDB = MyFirstAddonDB or {}
    InitializeDefaults(MyFirstAddonDB, DEFAULTS)
    ns.db = MyFirstAddonDB

    -- Initialize per-character saved variables
    MyFirstAddonCharDB = MyFirstAddonCharDB or {}
    InitializeDefaults(MyFirstAddonCharDB, CHAR_DEFAULTS)
    ns.charDB = MyFirstAddonCharDB

    ns:Print("Settings loaded.")
    frame:UnregisterEvent("ADDON_LOADED")
end
```

!!! danger "Common mistake: accessing saved variables too early"
    The globals (`MyFirstAddonDB`) are `nil` until `ADDON_LOADED` fires for your addon. Never access them at file load time — always wait for the event.

---

## Debugging

### Built-in Tools

| Tool | Usage | Description |
|------|-------|-------------|
| `/reload` | Chat command | Reloads the entire UI — essential during development |
| `/dump expr` | Chat command | Evaluates and prints any Lua expression (e.g., `/dump GetPlayerMapPosition()`) |
| `print()` | Lua function | Outputs to the default chat frame |
| `/console scriptErrors 1` | Chat command | Enables the built-in Lua error popup |

### Helpful Addons

!!! tip "BugSack + BugGrabber"
    Install [BugSack](https://www.curseforge.com/wow/addons/bugsack) and [BugGrabber](https://www.curseforge.com/wow/addons/bug-grabber) — they capture all Lua errors into a browsable list instead of modal popups. Essential for addon development.

### Debug Print Helper

Add a conditional debug print to your namespace:

```lua
function ns:Debug(...)
    if ns.db and ns.db.debug then
        local msg = ""
        for i = 1, select("#", ...) do
            msg = msg .. tostring(select(i, ...)) .. " "
        end
        print("|cffff6600[" .. addonName .. " Debug]|r " .. msg)
    end
end
```

### Inspecting Tables

```lua
-- Quick table dump for debugging
function ns:DumpTable(tbl, indent)
    indent = indent or ""
    for k, v in pairs(tbl) do
        if type(v) == "table" then
            print(indent .. tostring(k) .. " = {")
            ns:DumpTable(v, indent .. "  ")
            print(indent .. "}")
        else
            print(indent .. tostring(k) .. " = " .. tostring(v))
        end
    end
end
```

!!! tip "DevTool addon"
    For complex table inspection, install [DevTool](https://www.curseforge.com/wow/addons/devtool) — it provides a GUI table browser you can feed any value to with `/dev yourVariable`.

---

## Complete Working Example

Here's a full, working addon that ties all the concepts together.

### File Structure

```
Interface/AddOns/MyFirstAddon/
├── MyFirstAddon.toc
└── MyFirstAddon.lua
```

### MyFirstAddon.toc

```toc
## Interface: 120001
## Title: My First Addon
## Notes: A complete starter addon demonstrating core WoW addon patterns.
## Author: YourName
## Version: 1.0.0
## SavedVariables: MyFirstAddonDB
## SavedVariablesPerCharacter: MyFirstAddonCharDB

MyFirstAddon.lua
```

### MyFirstAddon.lua

```lua
local addonName, ns = ...

-- ---------------------------------------------------------------------------
-- Namespace utilities
-- ---------------------------------------------------------------------------

function ns:Print(msg)
    print("|cff00ccff" .. addonName .. ":|r " .. msg)
end

function ns:Debug(...)
    if not (ns.db and ns.db.debug) then return end
    local parts = {}
    for i = 1, select("#", ...) do
        parts[#parts + 1] = tostring(select(i, ...))
    end
    print("|cffff6600[" .. addonName .. " Debug]|r " .. table.concat(parts, " "))
end

-- ---------------------------------------------------------------------------
-- Defaults
-- ---------------------------------------------------------------------------

local DEFAULTS = {
    welcomeMessage = true,
    debug = false,
}

local CHAR_DEFAULTS = {
    enabled = true,
}

local function InitializeDefaults(saved, defaults)
    for key, value in pairs(defaults) do
        if saved[key] == nil then
            if type(value) == "table" then
                saved[key] = CopyTable(value)
            else
                saved[key] = value
            end
        end
    end
end

-- ---------------------------------------------------------------------------
-- Event handling (table dispatch)
-- ---------------------------------------------------------------------------

local frame = CreateFrame("Frame")
local events = {}

function events:ADDON_LOADED(loadedAddonName)
    if loadedAddonName ~= addonName then return end

    -- Initialize saved variables with defaults
    MyFirstAddonDB = MyFirstAddonDB or {}
    InitializeDefaults(MyFirstAddonDB, DEFAULTS)
    ns.db = MyFirstAddonDB

    MyFirstAddonCharDB = MyFirstAddonCharDB or {}
    InitializeDefaults(MyFirstAddonCharDB, CHAR_DEFAULTS)
    ns.charDB = MyFirstAddonCharDB

    ns:Debug("Saved variables initialized.")
    frame:UnregisterEvent("ADDON_LOADED")
end

function events:PLAYER_LOGIN()
    if ns.db.welcomeMessage then
        local name = UnitName("player")
        ns:Print("Welcome, " .. name .. "! Type /mfa for options.")
    end
end

function events:PLAYER_ENTERING_WORLD(isInitialLogin, isReloadingUi)
    if isInitialLogin then
        ns:Debug("Initial login detected.")
    elseif isReloadingUi then
        ns:Debug("UI reload detected.")
    end
end

-- Register all events and wire up the dispatcher
frame:SetScript("OnEvent", function(self, event, ...)
    if events[event] then
        events[event](self, ...)
    end
end)

for event in pairs(events) do
    frame:RegisterEvent(event)
end

-- ---------------------------------------------------------------------------
-- Slash commands
-- ---------------------------------------------------------------------------

SLASH_MYFIRSTADDON1 = "/mfa"
SLASH_MYFIRSTADDON2 = "/myfirstaddon"

SlashCmdList["MYFIRSTADDON"] = function(msg)
    local command = msg:match("^(%S*)"):lower()

    if command == "debug" then
        ns.db.debug = not ns.db.debug
        ns:Print("Debug mode: " .. (ns.db.debug and "|cff00ff00ON" or "|cffff0000OFF") .. "|r")

    elseif command == "welcome" then
        ns.db.welcomeMessage = not ns.db.welcomeMessage
        ns:Print("Welcome message: " .. (ns.db.welcomeMessage and "|cff00ff00ON" or "|cffff0000OFF") .. "|r")

    elseif command == "status" then
        ns:Print("--- Status ---")
        ns:Print("Version: 1.0.0")
        ns:Print("Debug: " .. tostring(ns.db.debug))
        ns:Print("Welcome: " .. tostring(ns.db.welcomeMessage))
        ns:Print("Character enabled: " .. tostring(ns.charDB.enabled))

    elseif command == "reset" then
        wipe(MyFirstAddonDB)
        InitializeDefaults(MyFirstAddonDB, DEFAULTS)
        ns.db = MyFirstAddonDB
        ns:Print("Settings reset to defaults. /reload to fully apply.")

    else
        ns:Print("Commands:")
        ns:Print("  /mfa debug    - Toggle debug mode")
        ns:Print("  /mfa welcome  - Toggle login welcome message")
        ns:Print("  /mfa status   - Show current settings")
        ns:Print("  /mfa reset    - Reset all settings to defaults")
    end
end
```

---

## Next Steps

You now have all the building blocks for a solid addon. From here, explore:

- [**TOC File Format**](toc-format.md) — all directives, multi-TOC for Retail/Classic, and file loading order
- [**Lua API Reference**](lua-api.md) — the functions WoW exposes to addons
- [**Events System**](events.md) — full event reference and payload documentation
- [**Frames & Widgets**](frames-widgets.md) — building UI with frames, buttons, textures, and more
- [**Security Model**](security.md) — understanding protected functions and hardware event requirements
- [**Midnight 12.0 Changes**](midnight.md) — what's new and what broke in the latest expansion
- [**Publishing & CI/CD**](publishing.md) — one-tag deployment to CurseForge, Wago, and WoWInterface
