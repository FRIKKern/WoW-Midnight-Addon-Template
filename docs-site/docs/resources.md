# Resources & References

A curated collection of documentation, tools, and community resources for World of Warcraft addon development.

---

## API Documentation

### Tier 1 — Essential References

These are the primary sources every addon developer should bookmark.

| Resource | Description |
|----------|-------------|
| [World of Warcraft API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) | Primary API reference — the definitive source for all WoW API functions |
| [Events Reference](https://warcraft.wiki.gg/wiki/Events) | Complete list of in-game events you can register and handle |
| [C_ Namespace Index](https://warcraft.wiki.gg/wiki/Category:API_namespaces) | Index of all C_ API namespaces (C_Map, C_Item, C_Timer, etc.) |
| [API Change Summaries](https://warcraft.wiki.gg/wiki/API_change_summaries) | Patch-by-patch tracking of API additions, removals, and changes |
| [XML/UI Reference](https://warcraft.wiki.gg/wiki/XML_user_interface) | XML schema and UI widget definitions |
| [Interface Customization Hub](https://warcraft.wiki.gg/wiki/Warcraft_Wiki:Interface_customization) | Central hub page linking to all UI/addon topics on the wiki |

### Tier 2 — Highly Useful

Supplementary references and beginner-friendly guides.

| Resource | Description |
|----------|-------------|
| [Wowpedia API Reference](https://wowpedia.fandom.com/wiki/World_of_Warcraft_API) | Backup API reference — sometimes has details or history not on warcraft.wiki.gg |
| [Widget API (Wowpedia)](https://wowpedia.fandom.com/wiki/Widget_API) | Comprehensive widget method reference (Frame, Button, FontString, etc.) |
| [Beginner's Guide to Addon Coding (Wowhead)](https://www.wowhead.com/guide/comprehensive-beginners-guide-for-wow-addon-coding-in-lua-5338) | Start-to-finish beginner tutorial covering Lua basics and addon structure |
| [Addon Writing by Example (Wowhead)](https://www.wowhead.com/guide/addon-writing-guide-a-basic-introduction-by-example-1949) | Hands-on tutorial building a simple addon step by step |
| [Create a WoW Addon in 15 Minutes](https://wowpedia.fandom.com/wiki/Create_a_WoW_AddOn_in_15_Minutes) | Quick-start guide for getting your first addon running fast |
| [Events A-Z Full List](https://addonstudio.org/wiki/WoW:Events_A-Z_(full_list)) | Single-page alphabetical listing of all events |

---

## Official Blizzard Sources

!!! note
    Blizzard does **not** provide standalone addon API documentation. The community wiki is the primary reference. The links below cover what Blizzard does publish.

| Resource | Description |
|----------|-------------|
| [Battle.net Developer Portal](https://community.developer.battle.net/documentation/world-of-warcraft) | Official Web APIs (game data, profiles) — **not** addon/in-game APIs |
| [10.0 UI Documentation Thread](https://us.forums.blizzard.com/en/wow/t/100-ui-documentation-addon-help/1380868) | Blizzard's forum post on the 10.0 UI overhaul with migration guidance |
| [Addon Releases with GitHub Actions](https://us.forums.blizzard.com/en/wow/t/creating-addon-releases-with-github-actions/613424) | Official forum guide on CI/CD pipelines for addon distribution |

---

## VS Code & Editor Tools

| Resource | Description |
|----------|-------------|
| [Lua (sumneko) Extension](https://marketplace.visualstudio.com/items?itemName=sumneko.lua) | Lua language server with IntelliSense, diagnostics, and formatting |
| [WoW API Extension](https://marketplace.visualstudio.com/items?itemName=Ketho.wow-api) | Autocomplete and type annotations for all WoW API functions and events |
| [WoW API Extension Source](https://github.com/Ketho/vscode-wow-api) | Source repository — useful for contributing or customizing definitions |
| [WoW UI Schema](https://github.com/Meorawr/wow-ui-schema) | XML schema definitions for WoW UI XML validation in your editor |

---

## Open Source Addon Examples

Study well-maintained addons to learn patterns and best practices.

| Addon | Description |
|-------|-------------|
| [OmniCC](https://github.com/tullamods/OmniCC) | Cooldown count addon — clean example of hooking into Blizzard frames |
| [Bagnon](https://github.com/tullamods/Bagnon) | Inventory addon — demonstrates complex UI with multiple frame types |
| [Total RP 3](https://github.com/Total-RP/Total-RP-3) | Roleplay addon — large-scale addon with modules, localization, and saved variables |

---

## Community Resources

| Resource | Description |
|----------|-------------|
| [awesome-wow](https://github.com/JuanjoSalvador/awesome-wow) | Curated list of addon development resources, libraries, and tools |
| [wowprogramming.com](https://wowprogramming.com) | Classic-era reference (WotLK/Cata) — still useful for fundamentals |
| [AddOn Studio 2022](https://addonstudio.org/wiki/AddOn_Studio_2022_for_World_of_Warcraft) | Full IDE for WoW addon development with visual designer |

---

## CI/CD & Distribution

### BigWigs Packager

The [BigWigs Packager](https://github.com/BigWigsMods/packager) is the community standard for automated addon releases. It handles:

- Packaging your addon with correct TOC and file structure
- Uploading to CurseForge, WoWInterface, and Wago
- Generating changelogs from git history
- Substituting version tags in your TOC file

### GitHub Actions Workflow

A typical release workflow using BigWigs Packager:

```yaml
name: Release
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: BigWigsMods/packager@v2
        env:
          CF_API_KEY: ${{ secrets.CF_API_KEY }}
          WOWI_API_TOKEN: ${{ secrets.WOWI_API_TOKEN }}
          WAGO_API_TOKEN: ${{ secrets.WAGO_API_TOKEN }}
```

Set your API tokens as repository secrets, then tag a release to trigger automated distribution.

---

## Quick Reference: Where to Look

| I need to... | Go to |
|--------------|-------|
| Look up an API function | [warcraft.wiki.gg API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) |
| Find an event name | [Events Reference](https://warcraft.wiki.gg/wiki/Events) |
| Check what changed in a patch | [API Change Summaries](https://warcraft.wiki.gg/wiki/API_change_summaries) |
| Understand a widget method | [Widget API](https://wowpedia.fandom.com/wiki/Widget_API) |
| Set up VS Code | [WoW API Extension](https://marketplace.visualstudio.com/items?itemName=Ketho.wow-api) + [Lua LSP](https://marketplace.visualstudio.com/items?itemName=sumneko.lua) |
| Automate releases | [BigWigs Packager](https://github.com/BigWigsMods/packager) |
| Learn from examples | [OmniCC](https://github.com/tullamods/OmniCC) / [Bagnon](https://github.com/tullamods/Bagnon) |
