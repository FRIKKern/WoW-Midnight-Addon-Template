---
title: "Agent Capabilities Showcase"
description: "Deep dive into the 9 specialized AI agents in the better-addons plugin — how they work, when to use each one, and how they collaborate to build, review, debug, and migrate WoW addons for Midnight 12.0+."
---

# Agent Capabilities Showcase

The better-addons plugin is not a single AI assistant that tries to do everything. It is a team of **nine specialized agents**, each with focused expertise, specific tool access, and deep domain knowledge. When you issue a command like `/wow-create` or `/wow-review`, the right agent activates — and it may spawn additional agents to verify facts, research APIs, or review its own output before presenting results.

This page covers what each agent does, when to use it, what tools it has access to, and how the agents collaborate on complex tasks.

---

## The Agent Architecture

### Route and Delegate

The agent system follows a **route and delegate** pattern. The Generalist agent acts as the front door — it receives all unstructured requests, answers simple questions directly, and delegates complex work to the appropriate specialist. Each specialist is an expert in a narrow domain: the Coder writes code, the Reviewer reads code, the Debugger diagnoses problems, and so on.

This architecture exists because WoW addon development is not one skill — it is many. Writing a complete addon requires knowledge of the TOC file format, the Lua 5.1 sandbox, the frame system, the event model, the taint system, Secret Values, combat lockdown rules, and hundreds of C_ namespace APIs. Reviewing an addon requires a different lens: security awareness, performance analysis, deprecated API detection, and Midnight compatibility checks. Debugging requires symptom categorization and root cause analysis. No single prompt can encode all of this well. Specialized agents can.

### Agent Spawning

Agents can spawn sub-agents during their work. When the Coder needs to verify an API signature it is not certain about, it spawns a Researcher sub-agent to look it up on warcraft.wiki.gg. When the Migrator needs to rewrite a file, it may spawn the Coder to generate the replacement. When the News Desk needs to cover five topics simultaneously, it spawns five parallel research agents. This delegation happens automatically — you do not need to orchestrate it.

### Tool Access Control

Each agent has a specific set of tools. The Reviewer has **no write access** — it cannot modify files, only read and report. The Researcher can search the web but cannot edit code. The Coder can write files but does not search the web directly (it delegates that to a Researcher). This separation enforces correctness: a review agent that could silently "fix" what it finds would defeat the purpose of a review.

### Mode Awareness

Every agent reads the active development mode before starting work. If the mode is set to `blizzard-faithful`, the Coder will refuse to use `hooksecurefunc` on Blizzard frames. If the mode is `performance-zealot`, the Reviewer will flag unthrottled OnUpdate handlers as critical issues. If the mode is `boundary-pusher`, the Scaffold will generate metatable hook boilerplate. The mode is a project-wide setting that shapes all AI behavior consistently.

---

## The Nine Agents

### :material-router: Generalist — The Router

The Generalist is the entry point for all unstructured questions and requests. It contains extensive inline knowledge about the WoW 12.0 addon landscape — the state of major addons, the Better addon ecosystem, Secret Values behavior, and common development patterns. For simple questions, it answers directly. For complex tasks, it identifies the right specialist and delegates.

!!! info "Tool Access"
    Read, Write, Edit, Glob, Grep, Bash, Agent, WebSearch, WebFetch — the Generalist has the broadest tool access of any agent because it needs to handle anything that does not fit a specialist's domain.

!!! tip "When to Use"
    Use the Generalist (or just ask a question without a slash command) when you want:

    - Quick answers about the addon ecosystem ("What happened to Details! in Midnight?")
    - Architecture advice ("Should I use SavedVariables or SavedVariablesPerCharacter?")
    - Guidance on which approach to take before committing to code
    - General WoW addon development questions

The Generalist knows the current state of major addons — which ones have migrated to 12.0, which are abandoned, which have forked. It knows the Better addon ecosystem (BetterBlizzFrames, BetterBlizzPlates, BetterAddonList) and can explain their patterns. It also understands development modes and will tailor architectural recommendations to the active mode.

!!! example "Example: Ecosystem Question"
    **Input:**
    ```
    What's the current status of WeakAuras in Midnight?
    ```

    **Output:**
    The Generalist answers directly from its inline knowledge — no sub-agent needed:

    > WeakAuras is fully functional in Midnight 12.0.1. The team shipped a major update
    > that handles Secret Values gracefully — aura triggers that depend on health/power
    > values use the widget-safe display path when `issecretvalue()` returns true, and fall
    > back to full logic outside restricted contexts. Custom functions that do arithmetic
    > on combat data will show warnings in the WeakAuras editor if they don't include
    > Secret Values guards. The addon communication features queue messages during
    > `InChatMessagingLockdown()` and flush on encounter end.

---

### :material-hammer-wrench: Coder — The Builder

The Coder is the largest and most complex agent in the system. Its agent definition is **39KB** — a complete WoW addon development specification baked into a single prompt. It contains the full TOC file format, Lua environment reference, event system documentation, frame and widget API, C_ namespace specifications, Secret Values rules, Settings API patterns, Addon Compartment setup, ScrollBox/DataProvider usage, build and release workflows, and a 16-item pre-submission verification checklist.

The Coder writes **complete, production-ready addon code** — never snippets, never placeholders, never "TODO" comments. Every addon it generates includes a properly structured TOC file, namespace pattern in every Lua file, event dispatch tables, combat lockdown awareness, and Secret Values handling. The code can be dropped directly into `Interface/AddOns/` and loaded by the game client.

!!! info "Tool Access"
    Read, Write, Edit, Glob, Grep, Bash, Agent — the Coder can read, write, and edit files. It spawns Researcher sub-agents when it needs to verify API details beyond its built-in specification.

!!! tip "When to Use"
    Use the Coder (via `/wow-create`) when you want:

    - A complete new addon generated from a description
    - New features added to an existing addon
    - Code that follows mode-specific patterns (faithful, boundary, enhancement, performance)
    - Production-ready code with TOC, Lua, .pkgmeta, and CI/CD

#### Anti-Hallucination System

The Coder's most distinctive feature is its **anti-hallucination guide** — explicit tables of functions that do not exist in WoW's Lua environment but that AI models commonly generate:

| Function | Reality |
|----------|---------|
| `require()` | Does not exist. WoW has no module system. |
| `sleep()` | Does not exist. Use `C_Timer.After()`. |
| `io.open()` | Does not exist. No filesystem access. |
| `os.time()` | Does not exist. Use `GetServerTime()`. |

It also includes a deprecated function mapping — 17 functions that existed in previous expansions but were removed or replaced in Midnight, with their correct modern equivalents.

!!! example "Example: Creating an Addon"
    **Input:**
    ```
    /wow-create a poison reminder for rogues that warns when poisons are missing
    ```

    **Output:**
    The Coder generates a complete addon with multiple files:

    - `PoisonReminder.toc` — Interface 120001, SavedVariables, AddonCompartment, Category
    - `Core.lua` — Event dispatch table, UNIT_AURA tracking for poison buffs, periodic poison check using C_Timer.NewTicker, Secret Values guards
    - `Config.lua` — Settings API panel with warning threshold slider and sound toggle
    - `.pkgmeta` — Package configuration for CurseForge/Wago release
    - `.luacheckrc` — Lint configuration with WoW globals declared

    Every file starts with `local addonName, ns = ...` and includes the mode-appropriate header comment.

---

### :material-magnify: Reviewer — The Critic

The Reviewer is a **read-only** agent. It cannot modify files — it can only read code and produce structured reports. This constraint is intentional: a code review should identify problems, not silently fix them. The developer decides what to act on.

The Reviewer uses a structured severity system with three tiers:

- **CRITICAL** — Will cause errors, crashes, or security issues (deprecated APIs, missing combat lockdown checks, Secret Values violations)
- **WARNING** — Will cause problems in specific scenarios (missing event unregistration, potential taint, performance issues)
- **STYLE** — Code quality and convention issues (missing namespace pattern, global variables, non-standard event handling)

Every finding includes a file path, line number, explanation of the problem, and a suggested fix. The final report includes an overall grade from A (no issues) to F (critical problems throughout).

!!! info "Tool Access"
    Read, Glob, Grep, Bash, Agent — **no Write or Edit access**. The Reviewer is strictly read-only. It can spawn sub-agents (typically a Researcher) to verify whether a flagged API is actually deprecated.

!!! tip "When to Use"
    Use the Reviewer (via `/wow-review`) when you want:

    - A comprehensive code review of your addon before release
    - Midnight compatibility validation
    - Deprecated API detection across your codebase
    - Performance analysis and optimization suggestions
    - Mode-specific compliance checking

!!! example "Example: Code Review"
    **Input:**
    ```
    /wow-review
    ```

    **Output (abbreviated):**
    ```
    ## Code Review: MyAddon

    ### CRITICAL
    - Core.lua:47 — Uses `GetSpellInfo()` (deprecated).
      Use `C_Spell.GetSpellInfo(spellID)` instead. Returns a
      SpellInfo table, not multiple values.
    - Core.lua:112 — Arithmetic on `UnitHealth("target")` without
      `issecretvalue()` check. Will error during M+/PvP/encounters.

    ### WARNING
    - Core.lua:89 — `ADDON_LOADED` handler does not call
      `self:UnregisterEvent("ADDON_LOADED")` after initialization.
    - UI.lua:34 — `frame:SetScript("OnUpdate", ...)` without
      elapsed throttle. Fires every frame (60-240+ Hz).

    ### STYLE
    - Config.lua:1 — Missing `local addonName, ns = ...` namespace
      pattern.

    Overall Grade: C
    3 files reviewed | 2 critical | 2 warnings | 1 style
    ```

---

### :material-bug: Debugger — The Detective

The Debugger diagnoses addon issues through **symptom categorization**. When you describe a problem, it classifies the symptom into one of nine categories and applies the corresponding diagnostic procedure. It does not guess — it reads your code, identifies the root cause, and explains both the problem and the fix.

The nine symptom categories are:

1. **Taint errors** — "Action blocked" messages, secure frame interference
2. **Secret Values problems** — Arithmetic errors that only appear in combat/M+/PvP
3. **Event failures** — Events not firing or firing with unexpected payloads
4. **Frame bugs** — Frames not appearing, incorrect positioning, visual glitches
5. **Combat lockdown violations** — Protected operations attempted during combat
6. **Communication failures** — Addon messages blocked in instances
7. **SavedVariables corruption** — Data not persisting or loading incorrectly
8. **Performance issues** — Memory leaks, FPS drops, excessive garbage collection
9. **Load failures** — Addon not appearing in the addon list, TOC errors

!!! info "Tool Access"
    Read, Glob, Grep, Bash, WebSearch, WebFetch, Agent — the Debugger can read code, search for patterns, look up documentation online, and spawn sub-agents for deep research. It cannot modify files.

!!! tip "When to Use"
    Use the Debugger (via `/wow-debug`) when you have:

    - An error message you do not understand
    - Behavior that differs between open world and instanced content
    - Intermittent failures that only appear in specific contexts
    - Performance problems you cannot isolate

!!! example "Example: Context-Dependent Bug"
    **Input:**
    ```
    /wow-debug my addon works fine in the open world but throws
    errors as soon as I enter a Mythic+ dungeon
    ```

    **Output:**
    The Debugger immediately categorizes this as a **Secret Values problem** — the defining symptom is "works in open world, breaks in restricted content." It reads your code, finds the line where you compare `UnitHealth("target") < threshold`, and explains:

    > In Mythic+ dungeons, `UnitHealth("target")` returns a **secret value** — an opaque
    > container that cannot be used in comparisons or arithmetic. Your code at `Core.lua:67`
    > performs `if health < warningThreshold then` which throws an error because the `<`
    > operator is not defined for secret values.
    >
    > **Fix:** Guard with `issecretvalue()`:
    > ```lua
    > local health = UnitHealth("target")
    > if issecretvalue(health) then
    >     healthBar:SetValue(UnitHealthPercent("target"))
    > else
    >     if health < warningThreshold then
    >         PlayWarningSound()
    >     end
    > end
    > ```

---

### :material-transfer: Migrator — The Translator

The Migrator converts pre-12.0 addons to Midnight compatibility through a **six-phase migration pipeline**. It does not just find-and-replace deprecated functions — it understands the semantic differences between old and new APIs and rewrites code to handle the structural changes (functions that returned multiple values now return tables, events that carried payloads now carry secret values, etc.).

The six phases are:

1. **TOC Update** — Interface number to 120001, add Category/Group fields, add AddonCompartment declarations
2. **API Replacements** — Deprecated function calls replaced with C_ namespace equivalents, with return value destructuring updated
3. **CLEU Migration** — `COMBAT_LOG_EVENT_UNFILTERED` handlers rewritten to use unit events (`UNIT_HEALTH`, `UNIT_AURA`, `UNIT_SPELLCAST_SUCCEEDED`)
4. **Secret Values** — All combat data access paths guarded with `issecretvalue()` checks and widget-safe fallbacks
5. **Communication** — Addon message sends wrapped with `InChatMessagingLockdown()` checks, queue-and-flush pattern added
6. **Validation** — Final pass checking for remaining deprecated calls, missing guards, and Midnight anti-patterns

!!! info "Tool Access"
    Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch, Agent — the Migrator has full read/write access because it needs to modify existing code files. It spawns Researcher agents to verify API replacements and may spawn a Coder for complex rewrites.

!!! tip "When to Use"
    Use the Migrator (via `/wow-migrate`) when you have:

    - An addon written for WoW 11.x or earlier that needs to work in 12.0+
    - Code that uses `CombatLogGetCurrentEventInfo()` or `COMBAT_LOG_EVENT_UNFILTERED` for logic
    - Code that uses deprecated functions like `GetSpellInfo()`, `EasyMenu()`, or `InterfaceOptions_AddCategory()`
    - An addon that works in the open world but breaks in instanced content

!!! example "Example: Migration"
    **Input:**
    ```
    /wow-migrate
    ```

    **Output (abbreviated):**
    ```
    ## Migration Report: OldAddon

    ### Phase 1: TOC Update
    - Updated Interface: 110002 -> 120001
    - Added: ## Category: Combat
    - Added: ## AddonCompartmentFunc: OldAddon_OnAddonCompartmentClick

    ### Phase 2: API Replacements (7 found)
    - Core.lua:23 — GetSpellInfo(id) -> C_Spell.GetSpellInfo(id)
      NOTE: Return value changed from multiple values to SpellInfo table.
      Updated destructuring: name = spellInfo.name, icon = spellInfo.iconID
    - Core.lua:89 — EasyMenu(menuList, ...) -> MenuUtil.CreateContextMenu(...)
      NOTE: Complete rewrite required. Menu structure converted.
    ...

    ### Phase 3: CLEU Migration
    - Removed COMBAT_LOG_EVENT_UNFILTERED handler (47 lines)
    - Added UNIT_HEALTH handler for health tracking
    - Added UNIT_AURA handler for debuff tracking
    - Added UNIT_SPELLCAST_SUCCEEDED handler for cast detection

    ### Phase 4: Secret Values Guards
    - Added issecretvalue() checks at 4 locations
    - Added widget-safe fallback paths for StatusBar updates

    6 phases complete | 23 changes | 0 manual review items
    ```

---

### :material-book-search: Researcher — The Verifier

The Researcher finds and verifies WoW API documentation. It does not trust its own training data — it actively searches warcraft.wiki.gg, GitHub repositories, and Blizzard blog posts to confirm API signatures, return values, event payloads, and behavioral details. Every piece of information it returns carries a **confidence label**.

The confidence labels are:

| Label | Meaning |
|-------|---------|
| **CONFIRMED** | Verified against warcraft.wiki.gg documentation |
| **CORROBORATED** | Found in multiple addon source repositories |
| **UNVERIFIED** | Mentioned in community sources but not officially documented |
| **LIKELY OUTDATED** | Documented for a previous expansion, may have changed |
| **DEPRECATED** | Confirmed removed or replaced in 12.0 |

The Researcher uses a **swarm pattern** — when researching a complex topic, it spawns 3-4 parallel sub-agents that each search different sources simultaneously, then consolidates their findings with cross-referenced confidence scores.

!!! info "Tool Access"
    WebSearch, WebFetch, Read, Glob, Grep, Agent — the Researcher can search the web and read local files, but **cannot write or edit code**. It is a pure research agent.

!!! tip "When to Use"
    Use the Researcher (via `/wow-research`) when you need:

    - Verified API documentation with exact function signatures and return types
    - Confirmation that an API exists and works as expected in 12.0
    - Cross-referenced information from multiple sources
    - Historical context on when an API was added, changed, or deprecated

!!! example "Example: API Research"
    **Input:**
    ```
    /wow-research C_EncounterEvents namespace — what functions exist and how do they work?
    ```

    **Output:**
    The Researcher spawns parallel sub-agents searching warcraft.wiki.gg, GitHub addon repos, and Blizzard developer posts. It returns:

    > **C_EncounterEvents** (added 12.0.1) — [CONFIRMED via warcraft.wiki.gg]
    >
    > Provides programmatic access to the Boss Timeline HUD data. Boss mods use this
    > to overlay custom visuals on Blizzard's native encounter timeline.
    >
    > **Functions:**
    >
    > - `C_EncounterEvents.GetEncounterEventInfo(eventID)` — [CONFIRMED]
    >   Returns: EncounterEventInfo table { name, iconID, duration, eventType }
    > - `C_EncounterEvents.GetActiveEncounterEvents()` — [CORROBORATED]
    >   Returns: array of active event IDs. Found in DBM and BigWigs source.
    >
    > **Used by:** DeadlyBossMods (confirmed in source), BigWigs (confirmed in source)

---

### :material-scaffold: Scaffold — The Architect

The Scaffold creates complete project structures from scratch. It generates not just code files but the entire development environment — TOC file, Lua source files, .pkgmeta for CurseForge/Wago packaging, .luacheckrc for linting, GitHub Actions release workflow, and optionally a CLAUDE.md file for AI-assisted development.

The Scaffold offers three tiers of project complexity:

| Tier | Files Generated | Best For |
|------|----------------|----------|
| **Minimal** | TOC + single Lua file | Simple utilities, learning projects |
| **Standard** | TOC + Init/Core/UI/Config Lua files + .pkgmeta + .luacheckrc | Most addons |
| **Pro** | Standard + embeds.xml + lib stubs + GitHub Actions CI/CD + .editorconfig | Addons intended for public release |

The Scaffold is **mode-aware**. In `blizzard-faithful` mode, it generates conservative boilerplate with Settings API and Addon Compartment. In `boundary-pusher` mode, it includes metatable hook boilerplate and taint isolation patterns. In `performance-zealot` mode, it generates local caching blocks, object pool boilerplate, and a `/perf` diagnostic slash command.

!!! info "Tool Access"
    Read, Write, Glob, Grep, Bash — the Scaffold can create files and directories but does not search the web or spawn sub-agents. It works from its built-in templates.

!!! tip "When to Use"
    Use the Scaffold (via `/wow-create` with a tier specification) when you want:

    - A clean project structure to start a new addon
    - Correct boilerplate for your chosen development mode
    - Build and release infrastructure (CI/CD, packaging, linting)

!!! example "Example: Scaffolding a Standard Addon"
    **Input:**
    ```
    /wow-create standard tier addon called "RaidCallouts" for encounter alerts
    ```

    **Output:**
    The Scaffold generates a complete directory structure:

    ```
    RaidCallouts/
      RaidCallouts.toc          -- Interface: 120001, Category: Boss Encounters
      Init.lua                  -- Namespace setup, defaults table, constants
      Core.lua                  -- Event dispatch, ENCOUNTER_START/END handling
      UI.lua                    -- Alert frame creation, positioning, animations
      Config.lua                -- Settings API panel, sound/visual toggles
      .pkgmeta                  -- CurseForge/Wago packaging config
      .luacheckrc               -- Lint config with WoW globals
      .github/workflows/release.yml  -- Tag-triggered release pipeline
    ```

---

### :material-palette: Skin Designer — The Artist

The Skin Designer specializes in **UI enhancement addons** — addons that modify the visual appearance of Blizzard's existing frames without replacing them. It is the agent that best understands the "enhance don't replace" philosophy and the specific techniques required to skin frames safely in Midnight.

The Skin Designer knows eight core skinning techniques:

1. **StripTextures** — Iterate `GetRegions()`, reparent textures to a hidden frame
2. **ApplyBackdrop** — Mixin `BackdropTemplateMixin`, apply flat backdrop with pixel borders
3. **hooksecurefunc post-hooks** — React to Blizzard function calls without overriding them
4. **HookScript** — Add behavior to Blizzard frame scripts without replacing handlers
5. **Mixin extension** — Add custom methods to Blizzard frame instances
6. **Edit Mode integration** — Hook `EditModeManagerFrame` enter/exit events
7. **ScrollBox skinning** — Hook `OnDataRangeChanged` to skin dynamically created list elements
8. **Secret Values-safe StatusBar hooks** — Apply visual modifications (class colors, gradients) without reading the bar's value

!!! info "Tool Access"
    Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch, Agent — the Skin Designer has full tool access because skinning addons often require research into Blizzard frame hierarchies and reference implementations.

!!! tip "When to Use"
    Use the Skin Designer when you want:

    - A "BetterX" style addon that enhances a specific Blizzard frame
    - Frame skinning with proper taint avoidance
    - Nameplate, tooltip, or unit frame visual modifications
    - Masque-compatible button skinning
    - Edit Mode-aware layout modifications

!!! example "Example: Skinning Request"
    **Input:**
    ```
    Make the player frame look modern — flat dark background, class-colored health bar,
    remove the portrait and dragon frame decorations
    ```

    **Output:**
    The Skin Designer generates a complete addon that:

    - Hooks `PlayerFrame` after Blizzard initializes it via `hooksecurefunc`
    - Strips decorative textures (portrait ring, dragon overlay) by reparenting to a hidden frame
    - Applies a flat dark backdrop with 1px border using `BackdropTemplateMixin`
    - Hooks the health bar's `SetStatusBarColor` to apply class color on every update
    - Checks `IsForbidden()` in every hook to avoid errors on protected frames
    - Includes Edit Mode awareness to disable position overrides during layout editing
    - Queues all frame modifications through a combat lockdown check

---

### :material-newspaper: News Desk — The Reporter

The News Desk provides **real-time addon ecosystem intelligence**. It searches the web for the latest news about WoW addon development — Blizzard API announcements, addon updates, community controversies, new addon releases, and ecosystem trends. It uses a **swarm pattern**, spawning up to five parallel research agents that each cover a different angle of the requested topic.

The five swarm agents typically cover:

1. **Official sources** — Blizzard blog posts, blue tracker, hotfix notes
2. **Addon platforms** — CurseForge, Wago, WoWInterface trending/updated
3. **Community discussion** — Reddit r/wow, r/wowaddons, WoW forums
4. **Developer activity** — GitHub commits, pull requests, release tags on major addon repos
5. **Content creator coverage** — YouTube, Twitch clips, blog posts about addon changes

!!! info "Tool Access"
    WebSearch, WebFetch, Read, Write, Agent — the News Desk focuses on web research and can write summary reports. It spawns multiple parallel sub-agents for comprehensive coverage.

!!! tip "When to Use"
    Use the News Desk (via `/wow-news`) when you want:

    - The latest state of a specific addon or addon category
    - Breaking news about API changes or Blizzard policy updates
    - Competitive landscape analysis ("What are the top nameplate addons right now?")
    - Trend reports on the addon ecosystem

!!! example "Example: News Report"
    **Input:**
    ```
    /wow-news what's happening with bag addons in Midnight?
    ```

    **Output:**
    The News Desk spawns five parallel agents and consolidates their findings:

    > **Bag Addons in Midnight — Ecosystem Report**
    >
    > **BetterBags** remains the dominant bag addon with active Midnight support. The
    > developer shipped a taint isolation fix (parent frame trick) that resolved the
    > last remaining combat-related errors. Currently at 2.1M downloads on CurseForge.
    >
    > **AdiBags** has a community fork maintaining Midnight compatibility after the
    > original author stepped back. The fork uses the new `C_Container` namespace
    > throughout.
    >
    > **Bagnon** updated to 12.0 with minimal changes — bag addons were largely
    > unaffected by Secret Values since inventory data is not restricted.
    >
    > **Trend:** Bag addons are one of the least disrupted categories in Midnight.
    > The main migration work was replacing deprecated container APIs with
    > `C_Container.*` equivalents.

---

## Agent Interactions

The agents do not work in isolation. Complex tasks trigger multi-agent workflows where specialists collaborate through spawning and delegation.

### The Verification Loop

When the Coder writes addon code, it may encounter an API it is not fully certain about — perhaps a newer function added in 12.0.1 that was not in its training data, or a function whose return value structure changed between expansions. Rather than guessing, the Coder spawns a **Researcher** sub-agent to verify the API against warcraft.wiki.gg and real addon source code. The Researcher returns a confidence-labeled answer, and the Coder uses only CONFIRMED or CORROBORATED information in the generated code. This verification loop is why the Coder rarely hallucinates API calls — it checks its work automatically.

### The Review Cycle

A natural development cycle emerges across three agents:

1. **Coder** generates the addon code
2. **Reviewer** examines the code and produces a structured report with severity-graded findings
3. **Debugger** investigates any runtime issues discovered during testing
4. **Coder** applies fixes based on the Reviewer's and Debugger's findings

You can run this cycle manually by issuing `/wow-create`, then `/wow-review`, then `/wow-debug` if issues arise. Each agent operates independently — the Reviewer does not know what the Coder intended, it only evaluates what the Coder produced. This independence prevents confirmation bias.

### The Migration Pipeline

Migrating a pre-12.0 addon to Midnight is the most agent-intensive workflow:

1. **Migrator** analyzes the existing codebase and identifies all deprecated APIs, CLEU usage, missing Secret Values guards, and communication patterns
2. **Researcher** verifies the correct replacement APIs for each deprecated function, confirming they exist and work as documented in 12.0
3. **Coder** rewrites complex sections that require structural changes (CLEU handlers converted to unit event handlers, EasyMenu converted to MenuUtil, InterfaceOptions converted to Settings API)
4. **Reviewer** validates the migrated code against Midnight compatibility requirements

This pipeline ensures that migrations are not just mechanical find-and-replace operations but semantic transformations that account for the structural differences between old and new APIs.

### Agent Comparison Table

| Agent | Writes Code | Searches Web | Spawns Agents | Read-Only |
|-------|:-----------:|:------------:|:-------------:|:---------:|
| Generalist | Yes | Yes | Yes | No |
| Coder | Yes | No | Yes | No |
| Reviewer | No | No | Yes | **Yes** |
| Debugger | No | Yes | Yes | **Yes** |
| Migrator | Yes | Yes | Yes | No |
| Researcher | No | Yes | Yes | **Yes** |
| Scaffold | Yes | No | No | No |
| Skin Designer | Yes | Yes | Yes | No |
| News Desk | Yes | Yes | Yes | No |
