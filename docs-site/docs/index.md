---
title: WoW Addon Development Guide
description: The complete reference for building World of Warcraft addons for Patch 12.0+ (Midnight)
---

# WoW Addon Development Guide

**The complete reference for building World of Warcraft addons for Patch 12.0+ (Midnight)**

Interface version: `120001` · Lua 5.1 · Updated for Midnight launch

---

## :rocket: Quick Start

!!! success "Your first addon in 60 seconds"

    Every addon needs just two files in your `Interface/AddOns/` folder:

    ```
    MyAddon/
    ├── MyAddon.toc
    └── MyAddon.lua
    ```

    **MyAddon.toc** — tells WoW about your addon:

    ```toc
    ## Interface: 120001
    ## Title: My Addon
    ## Notes: My first addon for Midnight!
    MyAddon.lua
    ```

    **MyAddon.lua** — your addon code:

    ```lua
    local frame = CreateFrame("Frame")
    frame:RegisterEvent("PLAYER_LOGIN")
    frame:SetScript("OnEvent", function(self, event)
        print("|cff00ff00MyAddon|r loaded! Welcome to Midnight.")
    end)
    ```

    Type `/reload` in-game to load your addon. That's it!

---

## :books: Documentation

<div class="grid cards" markdown>

-   :material-file-document-outline:{ .lg .middle } **TOC Format**

    ---

    Define your addon's metadata, dependencies, saved variables, and load conditions using the `.toc` file format.

    [:octicons-arrow-right-24: TOC Reference](toc-format.md)

-   :material-code-braces:{ .lg .middle } **Lua API**

    ---

    Core functions, C\_ namespaces, and the Lua 5.1 environment available to addons — over 260 API namespaces.

    [:octicons-arrow-right-24: API Reference](lua-api.md)

-   :material-lightning-bolt:{ .lg .middle } **Events**

    ---

    Event-driven architecture — register, handle, and respond to game events like `PLAYER_LOGIN` and `UNIT_AURA`.

    [:octicons-arrow-right-24: Events Guide](events.md)

-   :material-widgets-outline:{ .lg .middle } **Frames & Widgets**

    ---

    Build UI with Frame, Button, Texture, FontString, and other widget types. XML and Lua creation patterns.

    [:octicons-arrow-right-24: Widget Guide](frames-widgets.md)

-   :material-shield-lock-outline:{ .lg .middle } **Security**

    ---

    Protected functions, secure templates, taint system, and combat lockdown — what you can and can't do.

    [:octicons-arrow-right-24: Security Guide](security.md)

-   :material-moon-waning-crescent:{ .lg .middle } **Midnight Changes**

    ---

    What's new in 12.0 — Secret Values, CLEU removal, the secure UI model, and how to migrate your addons.

    [:octicons-arrow-right-24: Midnight Guide](midnight.md)

</div>

---

## :bulb: Key Facts

!!! info "Things every addon developer should know"

    | Fact | Detail |
    |------|--------|
    | **Language** | WoW uses **Lua 5.1** — not 5.2+. No `goto`, no bitwise operators, no `_ENV`. |
    | **Official docs** | Blizzard does **not** publish addon API docs. The in-game `/api` command is the only official source. [warcraft.wiki.gg](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) is the community authority. |
    | **Midnight's secure model** | Patch 12.0 introduced **Secret Values** — opaque black boxes that replace raw combat numbers. `COMBAT_LOG_EVENT_UNFILTERED` no longer fires for addons. |
    | **API scope** | **260+ C\_ namespace APIs** available (`C_Item`, `C_Spell`, `C_Timer`, etc.) plus hundreds of global functions. |
    | **Interface number** | `120001` for Patch 12.0.1 (Midnight launch). Set this in your `.toc` file. |
    | **Blizzard UI source** | Blizzard's own UI code is mirrored at [Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source) — the best way to learn real patterns. |

---

## :mortar_board: Learn from the Best

These widely-used addons have open-source codebases — study them to learn real-world patterns:

<div class="grid cards" markdown>

-   **Deadly Boss Mods (DBM)**

    ---

    The most downloaded addon in WoW history. Boss encounter alerts and timers.

    :material-download: 595M downloads · [:octicons-mark-github-16: Source](https://github.com/DeadlyBossMods/DeadlyBossMods)

-   **Details! Damage Meter**

    ---

    Combat analysis and damage/healing meters. Complex data visualization patterns.

    :material-download: 330M downloads · [:octicons-mark-github-16: Source](https://github.com/Tercioo/Details-Damage-Meter)

-   **WeakAuras 2**

    ---

    The ultimate custom UI framework. Deep use of the event system and secure templates.

    :material-star: 1.4k stars · [:octicons-mark-github-16: Source](https://github.com/WeakAuras/WeakAuras2)

-   **BigWigs Bossmods**

    ---

    Alternative boss mod framework. Clean architecture and plugin-based design.

    :material-download: 183M downloads · [:octicons-mark-github-16: Source](https://github.com/BigWigsMods/BigWigs)

-   **Plater Nameplates**

    ---

    Nameplate customization. Great examples of frame manipulation and visual scripting.

    :material-download: 90M downloads · [:octicons-mark-github-16: Source](https://github.com/Tercioo/Plater-Nameplates)

</div>

---

## :link: Essential Resources

| Resource | URL | Description |
|----------|-----|-------------|
| **WoW API Reference** | [warcraft.wiki.gg/wiki/World_of_Warcraft_API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) | The primary community API reference |
| **Events List** | [warcraft.wiki.gg/wiki/Events](https://warcraft.wiki.gg/wiki/Events) | Complete categorized events reference |
| **API Changes (12.0)** | [warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) | What changed in Midnight |
| **Blizzard UI Source** | [github.com/Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source) | Official UI code mirror |
| **VS Code Extension** | [WoW API IntelliSense](https://marketplace.visualstudio.com/items?itemName=ketho.wow-api) | Autocomplete for WoW API |
| **WoWUIDev Discord** | [discord.gg/txUg39Vhc6](https://discord.com/invite/txUg39Vhc6) | Addon developer community chat |

---

<div class="grid" markdown>

!!! tip "Development Setup"

    Install these VS Code extensions for the best experience:

    - [WoW API](https://marketplace.visualstudio.com/items?itemName=ketho.wow-api) — IntelliSense
    - [WoW Bundle](https://marketplace.visualstudio.com/items?itemName=Septh.wow-bundle) — Lua syntax
    - [WoW TOC](https://marketplace.visualstudio.com/items?itemName=stanzilla.vscode-wow-toc) — TOC support

!!! warning "Debugging"

    Install these addons for in-game error reporting:

    - [BugGrabber](https://www.curseforge.com/wow/addons/bug-grabber) — Captures Lua errors
    - [BugSack](https://www.curseforge.com/wow/addons/bugsack) — Displays them nicely
    - [DevTool](https://www.curseforge.com/wow/addons/devtool) — Runtime inspector

</div>
