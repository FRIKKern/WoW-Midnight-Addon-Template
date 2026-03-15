---
title: Plugin Overview & Quickstart
description: Get started with better-addons, the Claude Code plugin for WoW addon development targeting Patch 12.0+ (Midnight). 9 AI agents, 10 commands, 4 development modes.
---

# Plugin Overview & Quickstart

## What is better-addons?

better-addons is a **Claude Code plugin purpose-built for World of Warcraft addon development** targeting Patch 12.0+ (Midnight). It gives you a team of 9 specialized AI agents, 10 slash commands, and 4 development modes — all engineered from the ground up to write correct, modern addon code that actually works in the post-Midnight world.

If you've tried using a general-purpose AI to write WoW addons, you've hit the wall: hallucinated APIs like `C_Spell.GetSpellName()` that don't exist, code that calls `CombatLogGetCurrentEventInfo()` even though CLEU was removed in 12.0, settings panels built with `InterfaceOptions_AddCategory()` which hasn't worked since Dragonflight. Generic AI assistants don't know what they don't know about WoW's Lua environment — and that's exactly the problem better-addons solves.

### :material-sitemap: Route and Delegate Architecture

better-addons doesn't use a single monolithic prompt. Instead, it operates as an **agent swarm** with a clear delegation chain:

1. The **Generalist** receives your request and determines what kind of work it is
2. It routes to a **specialist** — the Coder for writing code, the Researcher for API verification, the Reviewer for quality checks, the Debugger for diagnosing issues
3. Specialists can further delegate — the Coder calls the Researcher to verify an API signature before using it, the News Desk spawns parallel search agents to cross-reference sources

This means when you ask "build me a boss timer addon," the system doesn't just generate code. It researches which timer APIs exist in 12.0.1, verifies that `C_EncounterTimeline` is the right namespace, confirms the function signatures, generates code using only verified APIs, and then reviews that code for deprecated calls and taint risks.

### :material-shield-check: Anti-Hallucination Engineering

Every agent in the system carries extensive tables of **functions that do not exist** in WoW's Lua environment. This is the inverse of normal documentation — instead of listing what's available, better-addons explicitly tells the AI what to never generate:

- `C_Spell.GetSpellName()` — does not exist (use `C_Spell.GetSpellInfo().name`)
- `C_Timer.SetTimeout()` — does not exist (use `C_Timer.After()`)
- `RegisterEvent()` as a global — does not exist (use `frame:RegisterEvent()`)
- `frame:SetBackdrop()` without `BackdropTemplate` — broken since 9.0

Combined with **Secret Values awareness** baked into every agent, better-addons understands that `UnitHealth("target")` returns an opaque value during combat in 12.0 that cannot be compared, divided, or used as a table key — and generates the correct `issecretvalue()` guard pattern automatically.

---

## Installation

### Prerequisites

- **Claude Code CLI** installed and authenticated ([install guide](https://docs.anthropic.com/en/docs/claude-code/overview))
- **A WoW addon project** or an empty directory where you want to create one
- WoW 12.0+ (Midnight) installed for testing (recommended, not required for code generation)

### Install the Plugin

```bash
claude install better-addons
```

### Initialize a New Project

Navigate to your addon directory and run:

```bash
claude
```

Then inside Claude Code:

```
/wow-create "MyFirstAddon - a simple addon that tracks your gold"
```

This scaffolds a complete addon project with all the files you need.

### Verify Installation

Inside Claude Code, run:

```
/wow-mode status
```

You should see the active development mode (defaults to `enhancement-artist`) and a list of all four available modes. If this works, better-addons is installed and ready.

!!! tip "Already have an addon project?"
    You don't need to start fresh. Navigate to your existing addon directory, launch Claude Code, and start using any `/wow-*` command. The plugin works with any WoW addon project structure.

---

## Your First Addon in 5 Minutes

Let's build a simple addon from scratch using `/wow-create`. This walkthrough shows you the full flow — from a one-line description to a working addon you can load in WoW.

### Step 1: Describe What You Want

```
/wow-create "WelcomeBack - shows a welcome message with your character name and
current gold when you log in, with a slash command to show it again"
```

The Coder agent reads the starter template, researches the required APIs (`GetMoney()`, `GetCoinTextureString()`, `UnitName("player")`), and generates a complete addon.

### Step 2: What Gets Generated

The plugin creates these files in your working directory:

=== "WelcomeBack.toc"

    ```toc
    ## Interface: 120001
    ## Title: WelcomeBack
    ## Notes: Shows a welcome message with your character name and gold on login.
    ## Author: YourName
    ## Version: @project-version@
    ## SavedVariables: WelcomeBackDB
    ## AddonCompartmentFunc: WelcomeBack_OnCompartmentClick
    ## IconTexture: Interface\Icons\Spell_Holy_BlessingOfStrength

    Init.lua
    Core.lua
    ```

=== "Init.lua"

    ```lua
    local addonName, ns = ...

    ns.addonName = addonName

    ns.defaults = {
        enabled = true,
        showOnLogin = true,
    }

    -- Event dispatcher
    local eventHandlers = {}
    local eventFrame = CreateFrame("Frame")
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        local handler = eventHandlers[event]
        if handler then handler(self, event, ...) end
    end)

    local function RegisterEvent(event, handler)
        eventHandlers[event] = handler
        eventFrame:RegisterEvent(event)
    end

    -- ADDON_LOADED: initialize SavedVariables
    RegisterEvent("ADDON_LOADED", function(self, event, loadedAddon)
        if loadedAddon ~= addonName then return end
        eventFrame:UnregisterEvent("ADDON_LOADED")

        if not WelcomeBackDB then
            WelcomeBackDB = {}
        end
        ns.db = WelcomeBackDB

        -- Merge defaults for any missing keys
        for key, value in pairs(ns.defaults) do
            if ns.db[key] == nil then
                ns.db[key] = value
            end
        end

        -- Register slash command
        SLASH_WELCOMEBACK1 = "/welcomeback"
        SLASH_WELCOMEBACK2 = "/wb"
        SlashCmdList["WELCOMEBACK"] = function(msg)
            ns:ShowWelcome()
        end
    end)

    -- PLAYER_LOGIN: show the welcome message
    RegisterEvent("PLAYER_LOGIN", function()
        if ns.db.showOnLogin then
            -- Delay slightly so chat frame is ready
            C_Timer.After(1, function()
                ns:ShowWelcome()
            end)
        end
    end)

    -- Addon Compartment click handler
    function WelcomeBack_OnCompartmentClick()
        ns:ShowWelcome()
    end
    ```

=== "Core.lua"

    ```lua
    local addonName, ns = ...

    function ns:ShowWelcome()
        local playerName = UnitName("player")
        local money = GetMoney()
        local goldText = GetCoinTextureString(money)

        print("|cff00ccff[WelcomeBack]|r Welcome back, " .. playerName .. "!")
        print("|cff00ccff[WelcomeBack]|r Current gold: " .. goldText)
    end
    ```

### Step 3: Understanding the Generated Code

Every generated addon follows the same battle-tested structure:

| File | Role | Loads |
|------|------|-------|
| `WelcomeBack.toc` | Manifest — tells WoW what to load, declares SavedVariables | First (by WoW client) |
| `Init.lua` | Namespace, event dispatch, SavedVariables init, slash commands | Second |
| `Core.lua` | Feature logic — the actual addon behavior | Third |

The code uses three foundational patterns:

**Namespace pattern** — `local addonName, ns = ...` gives every file access to a shared private table. No globals needed (except SavedVariables and slash commands, which WoW requires to be global).

**Table-dispatch events** — Instead of `if event == "X" then ... elseif event == "Y" then ...`, handlers are stored in a table and dispatched by key lookup. Clean, fast, and easy to extend.

**Proper initialization order** — `ADDON_LOADED` fires when your addon's files are loaded (where SavedVariables become available). `PLAYER_LOGIN` fires when the player enters the world (where you can safely access character data and show UI).

### Step 4: Install and Test

Copy the generated folder to your WoW addons directory:

```bash
cp -r WelcomeBack "/path/to/World of Warcraft/_retail_/Interface/AddOns/"
```

Launch WoW, log in, and you should see the welcome message in chat. Type `/wb` to show it again.

!!! success "That's it"
    You have a working addon. It handles SavedVariables correctly, follows the namespace pattern, uses the Addon Compartment for minimap access, and targets Interface 120001. No deprecated APIs, no global pollution, no taint risk.

---

## The Workflow

Real addon development is rarely "generate once and ship." better-addons supports a full **Research :material-arrow-right: Code :material-arrow-right: Review :material-arrow-right: Debug** cycle where agents hand off context to each other.

### The Agent Handoff Chain

Here's how agents collaborate on a real task. Say you want to build a **Poison Reminder** addon — something that checks if your Rogue has poisons applied and warns you before a dungeon.

#### :material-magnify: Phase 1 — Research

You ask: *"What APIs can I use to check if the player has poisons applied?"*

The Generalist recognizes this as an API research question and delegates to the **Researcher**. The Researcher launches parallel searches across the WoW wiki, addon source code on GitHub, and community resources. It returns verified findings:

- `C_UnitAuras.GetAuraDataByIndex()` — iterate all buffs, check for poison buff names
- `C_Spell.GetSpellInfo(spellID)` — get poison spell details by ID
- `C_SpellBook.HasSpell(spellID)` — check if the player knows a poison spell
- Secret Values note: aura data is readable out of combat, may be restricted in instanced PvP

Each finding carries a confidence label: **CONFIRMED** (verified in wiki + source), **CORROBORATED** (found in multiple real addons), or **UNVERIFIED** (single source only).

#### :material-code-braces: Phase 2 — Code

You say: *"Build the addon."*

The Generalist routes to the **Coder**, passing along the Researcher's verified API list. The Coder generates a complete addon using only the confirmed APIs — never inventing functions, never using deprecated calls. The generated code includes:

- Aura scanning on `UNIT_AURA` events (filtered to `"player"`)
- Instance detection via `IsInInstance()` before showing warnings
- `issecretvalue()` guards on any value that might be secret in combat
- Settings panel for configuring which poisons to track

#### :material-check-decagram: Phase 3 — Review

The Coder can optionally delegate to the **Reviewer** for a quality pass, or you can trigger it yourself:

```
/wow-review
```

The Reviewer scans for 7 categories of issues: deprecated APIs, taint risks, Secret Values violations, Lua 5.1 compatibility, combat lockdown violations, performance problems, and TOC correctness. You get a graded report (A through F) with severity-tagged findings.

#### :material-bug: Phase 4 — Debug

If something doesn't work in-game, paste the error:

```
/wow-debug "attempt to compare two secret values on PoisonReminder Core.lua:47"
```

The **Debugger** recognizes this as a Secret Values error, reads your code, identifies the line doing `if aura.duration > 0 then`, and rewrites it with the correct `issecretvalue()` guard pattern.

### Why This Matters

Each agent is a specialist with deep context about its domain. The Coder doesn't need to be an API researcher — it receives verified data. The Reviewer doesn't need to generate code — it only reads and critiques. This separation means each agent can be highly focused and carry extensive domain-specific anti-hallucination rules without hitting context limits.

---

## Quick Reference

### Commands

All commands are available as `/wow-*` slash commands inside Claude Code.

| Command | Description |
|---------|-------------|
| `/wow-create` | Generate a complete addon from a description |
| `/wow-api` | Look up a WoW API function, event, or namespace |
| `/wow-review` | Review addon code for bugs, deprecated APIs, and taint risks |
| `/wow-debug` | Diagnose an addon error or unexpected behavior |
| `/wow-migrate` | Migrate a pre-12.0 addon to Midnight compatibility |
| `/wow-research` | Deep research on a WoW API topic with multi-source verification |
| `/wow-verify` | Verify whether a code snippet or API claim is valid for 12.0.1 |
| `/wow-news` | Get the latest WoW addon ecosystem news |
| `/wow-mode` | Switch development mode or check current mode |
| `/wow-init` | Initialize a new addon project from the starter template |

For detailed usage of each command, see the [Commands Reference](commands.md).

### Agents

These agents work behind the scenes. You don't invoke them directly — commands and the Generalist route to them automatically.

| Agent | Role |
|-------|------|
| :material-robot: **Generalist** | Entry point. Routes requests to specialists. Handles simple questions directly. |
| :material-code-braces: **Coder** | Writes production-ready addon code. Carries the full anti-hallucination guide. |
| :material-magnify: **Researcher** | Verifies API signatures from multiple sources. Returns confidence-tagged findings. |
| :material-check-decagram: **Reviewer** | Read-only code review. Checks 7 categories, grades A-F. |
| :material-bug: **Debugger** | Diagnoses errors: taint, Secret Values, events, frames, init order. |
| :material-transfer: **Migrator** | Converts pre-12.0 addons to Midnight. 6-phase migration process. |
| :material-hammer-wrench: **Scaffold** | Generates project structure (TOC, files, CI/CD, linting config). |
| :material-newspaper: **News Desk** | Tracks addon ecosystem news with swarm research and source verification. |
| :material-palette: **Skin Designer** | UI enhancement specialist. hooksecurefunc, Mixin, frame skinning patterns. |

For a deep dive into each agent's capabilities, see the [Agents Reference](agents.md).

### Development Modes

Modes change how every agent writes, reviews, and scaffolds code. Set them with `/wow-mode`.

| Mode | Philosophy | Default? |
|------|-----------|----------|
| :material-shield-check: **blizzard-faithful** | Official APIs only. No Blizzard frame hooks. Patch-proof. | |
| :material-lightning-bolt: **boundary-pusher** | Undocumented APIs, metatable hooks, creative workarounds. ElvUI-class. | |
| :material-palette: **enhancement-artist** | hooksecurefunc, Mixin, frame skinning. "Enhance don't replace." | :material-check: |
| :material-speedometer: **performance-zealot** | Minimal memory, throttled updates, object pooling, event-driven only. | |

For detailed mode philosophy and code examples, see the [Modes Reference](modes.md).

---

## What Makes It Different

### The Problem with Generic AI

When you ask a general-purpose AI to write a WoW addon, it draws on training data that spans every version of WoW, every community post, and every deprecated tutorial ever published. The result is a confident blend of current and outdated code — and you won't know which is which until your addon throws errors in-game.

Common hallucinations from generic AI:

```lua
-- None of these work in WoW 12.0.1

local name = C_Spell.GetSpellName(spellID)       -- DOES NOT EXIST
C_Timer.SetTimeout(2, callback)                   -- DOES NOT EXIST
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED") -- EVENT REMOVED IN 12.0
RegisterEvent("PLAYER_LOGIN", handler)             -- NOT A GLOBAL FUNCTION
InterfaceOptions_AddCategory(panel)                -- REMOVED IN 11.0
local _, _, _, _, _, _, _, _, _, spellID = CombatLogGetCurrentEventInfo() -- REMOVED IN 12.0
```

These all look reasonable. An AI trained on pre-Midnight data will generate them with full confidence. Your addon loads, appears to work, then breaks the moment you enter combat or open settings.

### How better-addons Solves This

**Anti-hallucination tables** — Every code-generating agent carries an explicit blocklist of functions that don't exist or were removed. The AI is told *"do not generate these"* before it writes a single line.

**Mode-aware code generation** — A `blizzard-faithful` addon and a `boundary-pusher` addon solving the same problem look completely different. The mode system ensures the generated code matches your philosophy, not a one-size-fits-all approach.

**Swarm research** — The Researcher doesn't check one source. It launches parallel agents searching the WoW wiki, real addon source code on GitHub, and community resources simultaneously. Findings are cross-referenced and tagged with confidence levels before being passed to the Coder.

**12.0-native** — better-addons was built for Midnight. It doesn't treat Secret Values as an edge case or CLEU removal as a footnote — these are first-class concerns woven into every agent's instructions. When your addon needs to display health bars, it generates the `issecretvalue()` guard pattern by default, not as an afterthought.

---

## Next Steps

You're set up and you've built your first addon. Here's where to go from here:

- **[Development Modes](modes.md)** — Understand the four modes and how they change code generation, review criteria, and architectural recommendations
- **[Agents Reference](agents.md)** — Deep dive into each agent's capabilities, delegation rules, and specializations
- **[Commands Reference](commands.md)** — Detailed usage for all 10 commands with examples and options
- **[API Cheat Sheet](../api-cheatsheet.md)** — Verified 12.0.1 API signatures for quick reference
- **[Common Pitfalls](../pitfalls.md)** — 17 catalogued AI mistakes with wrong/right code comparisons
- **[Starter Template](../starter-template.md)** — The full template that powers `/wow-create`, explained line by line
