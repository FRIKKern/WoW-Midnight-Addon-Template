---
title: Command Reference
description: Complete reference for all ten better-addons slash commands — create addons, review code, debug issues, research APIs, and manage development modes from Claude Code.
---

# Command Reference

The better-addons plugin provides ten slash commands that turn Claude Code into a specialized WoW addon development environment. Each command invokes a purpose-built agent with deep knowledge of the Midnight 12.0+ API, Secret Values, taint model, and addon ecosystem.

---

## Using Commands

Commands are typed directly into Claude Code's prompt. They follow the format `/wow-<name>` and optionally accept arguments — a description, a file path, a question, or a mode name depending on the command.

```
/wow-create a housing decoration tracker with minimap button
/wow-review ./MyAddon/Core.lua
/wow-api C_Spell.GetSpellInfo
/wow-mode boundary
```

Each command invokes a specialized agent behind the scenes. The agent reads your project files, accesses the WoW API knowledge base, and may spawn sub-agents for parallel research. You interact with the results as normal Claude Code output — code is written to your project, reports appear in the conversation, and you can follow up with questions.

**Commands are mode-aware.** The active development mode (set with `/wow-mode`) changes how every command behaves. In `blizzard-faithful` mode, `/wow-create` generates conservative code using only official APIs. In `boundary-pusher` mode, the same command produces aggressive code with metatable hooks and frame-level modifications. `/wow-review` adjusts its severity thresholds based on the active mode — what's a critical violation in `blizzard-faithful` might be expected practice in `boundary-pusher`.

Some commands also accept inline mode overrides. Including keywords like "faithful", "boundary", "aggressive", or "performance" in your arguments temporarily overrides the active mode for that invocation without changing the project's mode setting.

!!! tip "Plugin prefix"
    If you have multiple plugins installed, you may need the fully qualified name: `/better-addons:wow-create`. If better-addons is your only plugin with `wow-` commands, the short form `/wow-create` works fine.

---

## All Commands

### /wow-init

:material-rocket-launch: **Initialize a project for better-addons development**

**Invoked agent:** Built-in initialization logic (no specialized agent)

Sets up a WoW addon project directory to work with the better-addons plugin. Creates the mode system, generates a starter `CLAUDE.md` with Midnight conventions, and displays available commands.

**Syntax:**

```
/wow-init
/wow-init boundary
/wow-init MyAddonProject
```

| Argument | Effect |
|----------|--------|
| *(none)* | Initialize with default mode (`enhancement-artist`) |
| Mode name or alias | Initialize with that mode active |
| Project name | Used in the generated `CLAUDE.md` header |

!!! example "Example"

    ```
    > /wow-init boundary

    better-addons initialized!

    Current mode: boundary-pusher

    Available commands:
      /wow-create   — Create a complete addon
      /wow-review   — Review addon code
      /wow-debug    — Debug addon issues
      /wow-mode     — Switch development philosophy
      /wow-migrate  — Migrate to Midnight 12.0
      /wow-api      — Look up WoW API docs
      /wow-news     — Latest addon ecosystem news
      /wow-research — Research & verify information
      /wow-verify   — Verify code or API claims

    Change mode: /wow-mode faithful|boundary|enhance|performance
    ```

**What it creates:**

- `.claude/modes/active-mode.md` — containing the active mode name
- `CLAUDE.md` — with WoW 12.0+ conventions, Secret Values guidance, and a command reference (only if one doesn't already exist)

!!! tip
    Run `/wow-init` first in any new project. It creates the `.claude/modes/` directory that all other commands depend on for mode-aware behavior.

---

### /wow-create

:material-plus-box: **Create a complete addon from a natural language description**

**Invoked agent:** WoW Addon Coder (with WoW Addon Researcher sub-agent for API verification)

Takes a plain English description of what your addon should do and generates a complete, ready-to-install addon — TOC file, Lua files, Settings panel, slash commands, and all. Every API call is verified against the 12.0.1 specification before code is written.

**Syntax:**

```
/wow-create a poison reminder addon that alerts when lethal poison is missing
/wow-create faithful housing decoration tracker with minimap button
/wow-create boundary nameplate addon with classification-based coloring
```

| Argument | Effect |
|----------|--------|
| Description | Natural language description of the addon |
| Mode keyword | Optional inline override: `faithful`, `boundary`, `enhance`, `performance` |

!!! example "Example"

    ```
    > /wow-create a buff tracker that shows my active buffs near the minimap

    ## Addon Created: MiniBuffTracker

    **Files:**
    - MiniBuffTracker/MiniBuffTracker.toc — Interface 120001
    - MiniBuffTracker/Init.lua — Namespace setup
    - MiniBuffTracker/Core.lua — Buff tracking logic
    - MiniBuffTracker/Config.lua — Settings panel

    **Features:**
    - Compact buff display anchored to minimap
    - Tracks player buffs using C_UnitAuras
    - Secret Values safe — degrades gracefully in instances
    - Configurable icon size and layout via Settings panel

    **APIs Used:**
    - C_UnitAuras.GetBuffDataByIndex()
    - C_UnitAuras.GetPlayerAuraBySpellID()
    - Settings.RegisterVerticalLayoutCategory()

    **Installation:**
    Copy the `MiniBuffTracker` folder to Interface/AddOns/
    ```

The agent follows mandatory patterns: namespace pattern (`local ADDON_NAME, ns = ...`), event dispatch table, `issecretvalue()` guards on combat data, and `InCombatLockdown()` checks before frame modifications. It never generates placeholder code or TODO comments.

!!! warning "Common mistake"
    Don't ask for addons that fundamentally require Secret Values access (rotation helpers, detailed damage breakdowns, conditional ability triggers). The agent will explain why these are impossible under Midnight restrictions and suggest viable alternatives.

---

### /wow-review

:material-magnify: **Review addon code for bugs, deprecated APIs, and taint risks**

**Invoked agent:** WoW Addon Reviewer

Performs a thorough code review across seven categories: API correctness, taint risks, Secret Values compliance, pattern violations, Lua 5.1 compliance, performance, and TOC file validation. Produces a graded report with specific file/line references and fix suggestions.

**Syntax:**

```
/wow-review ./MyAddon/
/wow-review ./MyAddon/Core.lua
/wow-review as boundary-pusher ./MyAddon/
```

| Argument | Effect |
|----------|--------|
| File path | Review a single file |
| Directory path | Review all `.lua`, `.toc`, and `.xml` files recursively |
| Mode keyword | Override mode for this review (e.g., `as boundary-pusher`) |

!!! example "Example output (abbreviated)"

    ```markdown
    ## Code Review: MyAddon/

    **Reviewer:** WoW Addon Code Review (12.0.1)
    **Files Reviewed:** 4
    **Overall Grade:** C

    ### Summary
    | Severity | Count |
    |----------|-------|
    | CRITICAL | 2     |
    | HIGH     | 1     |
    | MEDIUM   | 3     |
    | LOW      | 1     |
    | PASS     | 18    |

    ### Findings

    #### CRITICAL

    **[C1] Deprecated API Usage** — Core.lua:42
    GetSpellInfo() removed in 11.0. Use C_Spell.GetSpellInfo().

    **[C2] Unguarded Secret Value** — Health.lua:88
    UnitHealth("target") compared without issecretvalue() guard.
    ```

Review severity adjusts by mode. In `blizzard-faithful`, any `hooksecurefunc` on Blizzard frames is flagged as critical. In `boundary-pusher`, missing `-- BOUNDARY` comments and `pcall` wrappers around metatable access are flagged instead.

!!! tip
    Review early and often. Run `/wow-review` after `/wow-create` to catch any edge cases the generator missed, and after `/wow-migrate` to verify all deprecated APIs were replaced correctly.

---

### /wow-debug

:material-bug: **Diagnose addon errors from symptoms or error messages**

**Invoked agent:** WoW Addon Debugger

Takes an error message, symptom description, or file path with an error and diagnoses the root cause. Categorizes issues into nine categories (taint, code errors, silent failures, frame issues, data issues, Secret Values, taint spread, persistence, and initialization order) and provides targeted fixes with before/after code.

**Syntax:**

```
/wow-debug attempt to perform arithmetic on a secret value
/wow-debug "addon works in open world but breaks in dungeons"
/wow-debug Core.lua:42 attempt to index a nil value
/wow-debug "health bars show 0 during boss fights"
```

| Argument | Effect |
|----------|--------|
| Error message | Diagnose that specific Lua error |
| Symptom description | Identify the most likely cause |
| File:line + error | Read the file, find the location, diagnose in context |

!!! example "Example"

    ```
    > /wow-debug attempt to perform arithmetic on a secret value

    ## Diagnosis: Secret Values Arithmetic Error

    **Category:** A (Secret Values)
    **Severity:** CRITICAL
    **Context:** Error occurs inside instances, M+, PvP, or encounters

    ### Root Cause
    Code is performing math (addition, division, comparison) on a value
    returned by UnitHealth(), UnitPower(), or similar API that returns
    Secret Values in restricted contexts.

    ### Fix

    **Before (broken):**
    local percent = UnitHealth("target") / UnitHealthMax("target") * 100

    **After (fixed):**
    local health = UnitHealth("target")
    local maxHealth = UnitHealthMax("target")
    if issecretvalue(health) or issecretvalue(maxHealth) then
        myBar:SetValue(health)  -- widget setters accept secrets
        return
    end
    local percent = health / maxHealth * 100

    ### Testing
    Force Secret Values in open world for testing:
    /run SetCVar("secretUnitPowerForced", 1)
    ```

!!! tip
    The most common Midnight debugging scenario is "works in open world, breaks in dungeons." This is almost always a Secret Values issue. The debugger knows this pattern and will suggest `issecretvalue()` guards immediately.

---

### /wow-migrate

:material-transfer: **Scan and migrate pre-12.0 addon code to Midnight**

**Invoked agent:** WoW Addon Migrator

Scans an entire addon for deprecated APIs, CLEU dependencies, Secret Values readiness, interface version issues, and pattern modernization opportunities. Produces a structured migration report and can optionally apply fixes automatically.

**Syntax:**

```
/wow-migrate ./MyOldAddon/
/wow-migrate ./MyOldAddon/Core.lua
```

| Argument | Effect |
|----------|--------|
| Directory path | Scan all `.lua` and `.toc` files recursively |
| File path | Scan a single file |

The migration scan runs in five phases:

1. **Deprecated API detection** — flags all 50+ removed functions with their replacements
2. **CLEU dependency analysis** — finds `COMBAT_LOG_EVENT_UNFILTERED` and `CombatLogGetCurrentEventInfo()` usage
3. **Secret Values readiness** — finds unguarded arithmetic on `UnitHealth()`, `UnitPower()`, etc.
4. **Interface version** — checks TOC for `120001`
5. **Pattern modernization** — suggests `RegisterEventCallback()`, namespace pattern, and other modern patterns

!!! example "Example output (abbreviated)"

    ```markdown
    ## Migration Report: MyOldAddon

    **Scanned:** 6 files
    **Patch Target:** 12.0.1 (Interface 120001)
    **Overall Status:** CRITICAL ISSUES

    ### Summary
    - Critical: 4 (blocks loading or causes errors)
    - Warning: 7 (degraded functionality)
    - Info: 3 (modernization opportunities)

    ### Critical Issues
    | # | File:Line        | Issue                    | Fix                           |
    |---|------------------|--------------------------|-------------------------------|
    | 1 | Core.lua:12      | GetSpellInfo() removed   | C_Spell.GetSpellInfo()        |
    | 2 | Combat.lua:45    | CLEU parsing in instance | Migrate to unit events        |
    | 3 | Health.lua:88    | UnitHealth() unguarded   | Add issecretvalue() check     |
    | 4 | MyAddon.toc:1    | Interface: 110002        | Change to 120001              |
    ```

After generating the report, the migrator asks whether to apply critical fixes automatically using the Edit tool.

!!! warning
    CLEU-dependent code (combat log parsing) cannot be mechanically migrated. The migrator will flag it and suggest replacement strategies (unit events, `C_DamageMeter`, `UNIT_SPELLCAST_SUCCEEDED`), but this requires architectural decisions that need human judgment.

---

### /wow-api

:material-book-open-variant: **Look up any WoW API function, event, or widget method**

**Invoked agent:** WoW Addon Researcher

Quick API reference lookup. Returns the current 12.0.1 status, signature, parameters, return values, Secret Values behavior, and a usage example. For deprecated functions, immediately shows the replacement with a migration code sample.

**Syntax:**

```
/wow-api C_Spell.GetSpellInfo
/wow-api UnitHealth
/wow-api UNIT_AURA
/wow-api Frame:SetFrameStrata
/wow-api C_Housing
/wow-api Secret Values
```

| Argument | Effect |
|----------|--------|
| Function name | Look up a global or namespaced function |
| Event name | Look up an event's payload and Secret Values status |
| Widget method | Look up a frame/widget method |
| C_ namespace | List all functions in a namespace |
| Concept | Explain a WoW addon concept |

!!! example "Example"

    ```
    > /wow-api C_Spell.GetSpellInfo

    ## API Reference: C_Spell.GetSpellInfo

    **Status:** CURRENT (changed in 11.0)
    **Patch:** 12.0.1 (Interface 120001)
    **Source:** https://warcraft.wiki.gg/wiki/API_C_Spell.GetSpellInfo

    ### Signature
    spellInfo = C_Spell.GetSpellInfo(spellIdentifier)

    ### Parameters
    | Name            | Type          | Description          |
    |-----------------|---------------|----------------------|
    | spellIdentifier | number/string | Spell ID or name     |

    ### Returns
    | Name      | Type  | Description                                    |
    |-----------|-------|------------------------------------------------|
    | spellInfo | table | { name, iconID, castTime, minRange, maxRange } |

    ### Example
    local info = C_Spell.GetSpellInfo(spellId)
    if info then
        print(info.name, info.iconID)
    end

    ### Notes
    - Replaces the old GetSpellInfo() which returned multiple values
    - Returns a TABLE, not multiple values — do not destructure
    ```

!!! tip
    Use `/wow-api` for quick lookups during coding. Use `/wow-research` when you need deep investigation with cross-referenced sources and community context.

---

### /wow-research

:material-flask: **Deep research with parallel verification from multiple sources**

**Invoked agent:** WoW Addon Researcher (coordinator), plus three parallel sub-agents: Wiki Specialist, Community Specialist, and Code Specialist

Launches a three-agent research swarm that simultaneously searches warcraft.wiki.gg, community sites (Reddit, Wowhead, Icy Veins, Blizzard forums), and GitHub repositories. Results are cross-referenced and tagged with confidence levels.

**Syntax:**

```
/wow-research how does Cell handle Secret Values in raid frames
/wow-research C_EncounterEvents namespace usage patterns
/wow-research ScrollBox element factory hooks
```

| Argument | Effect |
|----------|--------|
| Topic | Any WoW addon development topic to investigate |

!!! example "Example"

    ```
    > /wow-research Private Aura integration for audio alerts

    # Research Report: Private Aura Audio Integration

    **Date:** 2026-03-15
    **Patch:** 12.0.1 (Midnight)
    **Confidence:** CONFIRMED

    ## Summary
    Private Auras in 12.0 allow addons to attach custom audio to
    encounter-specific debuffs. DBM and BigWigs both use this pattern
    for boss mod alerts.

    ## Findings

    ### From warcraft.wiki.gg
    [API signatures, event documentation]

    ### From Community
    [Wowhead coverage, developer statements from MysticalOS]

    ### From GitHub
    [DBM source code showing Private Aura audio integration]

    ## Conflicts / Gaps
    None — all three sources agree on the implementation approach.
    ```

Each finding is tagged: **CONFIRMED** (multiple sources agree), **LIKELY** (single credible source), or **UNVERIFIED** (speculation or unverifiable).

!!! tip
    `/wow-research` is significantly slower than `/wow-api` because it launches parallel agents. Use it when you need community context, real-world code examples, or answers to questions that go beyond API signatures.

---

### /wow-verify

:material-check-decagram: **Fact-check code snippets or API claims**

**Invoked agent:** WoW Addon Researcher (verification mode)

Takes a code snippet, file, or textual claim and verifies every API call, pattern, and assumption against the 12.0.1 specification. Catches hallucinated functions, deprecated calls, and Secret Values violations that might have slipped through.

**Syntax:**

```
/wow-verify GetSpellInfo still works in 12.0
/wow-verify ./MyAddon/Core.lua
/wow-verify "C_Spell.GetSpellName(spellId) returns the spell name"
```

| Argument | Effect |
|----------|--------|
| Claim | Verify a factual statement about the WoW API |
| Code snippet | Check all API calls in the snippet |
| File path | Read the file and verify all API usage |

!!! example "Example"

    ```
    > /wow-verify "C_Spell.GetSpellName(spellId) returns the spell name"

    ## Verification Report

    **Target:** C_Spell.GetSpellName()
    **Patch:** 12.0.1 (Interface 120001)
    **Overall:** FAIL

    ### Results
    | # | Check                     | Status | Details                              |
    |---|---------------------------|--------|--------------------------------------|
    | 1 | C_Spell.GetSpellName()    | FAIL   | Function does NOT exist in WoW API   |

    ### Required Fixes
    1. C_Spell.GetSpellName() is a common AI hallucination.
       Use instead: C_Spell.GetSpellInfo(spellId).name
    ```

!!! warning
    AI-generated WoW addon code frequently contains hallucinated functions. The most common offenders: `C_Spell.GetSpellName()`, `C_Spell.GetSpellIcon()`, `C_UnitAuras.GetAuraBySpellID()`, `C_Item.GetItemName()`, and `C_Timer.SetTimeout()`. None of these exist. Always verify.

---

### /wow-news

:material-newspaper: **Latest WoW addon ecosystem intelligence**

**Invoked agent:** WoW Addon News Desk

Searches official Blizzard sources, community news sites, developer blogs, and addon platforms for the latest WoW addon ecosystem developments. Every claim is verified against the original source with confidence tags.

**Syntax:**

```
/wow-news
/wow-news latest
/wow-news WeakAuras
/wow-news housing addons
/wow-news API changes
```

| Argument | Effect |
|----------|--------|
| *(none)* or `latest` | General ecosystem news roundup |
| Topic | News filtered to a specific topic |

!!! example "Example"

    ```
    > /wow-news housing addons

    ## WoW Addon News Brief

    **Date:** 2026-03-15
    **Topic:** Housing Addons
    **Patch:** 12.0.1 (Midnight, launched March 2, 2026)

    ### Headlines

    1. **HomeBound hits 3M downloads** — The essential housing addon
       adds 3D preview images and organized checklists for all
       housing achievements.
       Source: curseforge.com (March 2026) [CONFIRMED]

    2. **C_HousingPhotoSharing API added in 12.0.1** — New namespace
       with authorization flows, screenshot handling, and uploads.
       Source: warcraft.wiki.gg [CONFIRMED]

    ### What This Means for Addon Developers
    Housing APIs are unrestricted — no Secret Values, no combat
    lockdown. This is a greenfield opportunity with a hungry user base.
    ```

News is prioritized by impact: API changes that break addons come first, then Blizzard policy statements, major addon deaths/comebacks, new addons filling gaps, and community drama.

!!! tip
    Run `/wow-news` before starting a new addon project. The landscape is evolving rapidly post-Midnight — an addon concept that was impossible two weeks ago might be viable after a whitelist expansion, and vice versa.

---

### /wow-mode

:material-palette: **Switch the active development philosophy mode**

**Invoked agent:** Built-in mode manager (no specialized agent)

Changes the development mode that governs how all other commands behave. The mode is persisted in your project's `.claude/modes/active-mode.md` and affects code generation, review severity, scaffold structure, and architectural recommendations.

**Syntax:**

```
/wow-mode
/wow-mode status
/wow-mode boundary
/wow-mode faithful
/wow-mode enhance
/wow-mode performance
```

| Argument | Effect |
|----------|--------|
| *(none)* or `status` | Display current mode and all options |
| Mode name or alias | Switch to that mode |

**Available modes:**

| Mode | Aliases | Philosophy |
|------|---------|-----------|
| `blizzard-faithful` | faithful, blizzard, safe, conservative | Official APIs only. No Blizzard frame hooks. Patch-proof. |
| `boundary-pusher` | boundary, pusher, aggressive, advanced, elvui | Metatable hooks, creative workarounds. ElvUI-class. |
| `enhancement-artist` | enhance, artist, better, skin, hook | hooksecurefunc, Mixin, frame skinning. **DEFAULT.** |
| `performance-zealot` | performance, perf, zealot, fast, lean | Minimal memory, throttled updates, object pooling. |

!!! example "Example"

    ```
    > /wow-mode boundary

    Mode: boundary-pusher

    Philosophy: "Push WoW's addon API to its absolute limits."

    Key rules:
    - getmetatable() + __index hooks on Blizzard frame types
    - hooksecurefunc on any Blizzard global function
    - Noop pattern to block texture re-application
    - Every aggressive technique requires pcall() and fallback path
    - BOUNDARY comments marking fragile code

    All /wow-create, /wow-review, and agent interactions will now
    follow boundary-pusher rules.
    ```

!!! tip
    You can override the mode for a single command without switching globally. Include a mode keyword in your arguments: `/wow-create faithful housing tracker` uses `blizzard-faithful` rules for that one addon without changing your project's active mode.

---

## Command Cheat Sheet

### Quick Reference

| Command | What It Does | When to Use It |
|---------|-------------|----------------|
| `/wow-init` | Set up project for better-addons | First time in a new project |
| `/wow-create` | Generate a complete addon | Starting a new addon from scratch |
| `/wow-review` | Review code for issues | After writing or generating code |
| `/wow-debug` | Diagnose error messages | When your addon throws errors |
| `/wow-migrate` | Scan for deprecated APIs | Updating a pre-12.0 addon |
| `/wow-api` | Quick API lookup | Need a function signature fast |
| `/wow-research` | Deep multi-source research | Complex questions needing evidence |
| `/wow-verify` | Fact-check code or claims | Validating AI-generated code |
| `/wow-news` | Latest ecosystem updates | Before starting new work |
| `/wow-mode` | Switch development philosophy | Changing your coding approach |

### Recommended Workflows

**:material-plus-circle: New addon from scratch:**

```
/wow-init              -- set up the project
/wow-mode enhance      -- choose your philosophy
/wow-create "a buff tracker near the minimap"
/wow-review ./BuffTracker/
```

Initialize, set your mode, generate the addon, then review the output. The review catches edge cases the generator might miss.

**:material-transfer: Migrating a legacy addon:**

```
/wow-migrate ./MyOldAddon/
/wow-review ./MyOldAddon/
/wow-debug "attempt to call a nil value at Core.lua:42"
```

Scan for deprecated APIs, review the migrated code, then debug any remaining issues. The migrator handles mechanical replacements; the debugger handles the subtle breakages.

**:material-book-search: Research and verification:**

```
/wow-api C_EncounterEvents        -- quick signature lookup
/wow-research Private Aura audio  -- deep investigation
/wow-verify "code snippet here"   -- fact-check before shipping
```

Start with `/wow-api` for fast lookups. Escalate to `/wow-research` when you need community context and real-world examples. Always `/wow-verify` before shipping AI-generated code.

**:material-wrench: Ongoing maintenance:**

```
/wow-news API changes    -- check for breaking changes
/wow-review ./MyAddon/   -- re-review after patch notes
/wow-migrate ./MyAddon/  -- scan for newly deprecated APIs
```

After each WoW patch, check the news for API changes, review your addon against the new rules, and run a migration scan to catch any newly deprecated functions.

!!! tip "Chaining commands"
    Commands don't need to be run in isolation. You can ask Claude Code to run a sequence: "Create a housing addon, then review it, then fix any issues." Claude will invoke `/wow-create`, pass the output to `/wow-review`, and apply fixes — all in one conversation.
