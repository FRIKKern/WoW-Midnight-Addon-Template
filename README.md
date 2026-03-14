# WoW Midnight Addon Template (Patch 12.0+)

A production-ready starter template and comprehensive documentation site for World of Warcraft addon development targeting Patch 12.0 (Midnight) and beyond. Includes a complete addon template, a 26-page docs site, and AI-powered development tools.

**Interface version:** `120001` | **Lua:** 5.1 | **Target:** Midnight (12.0+)

## Features

- Complete addon template with modern Blizzard API patterns (no Ace3 dependency)
- Secret Values handling, combat lockdown patterns, and Addon Compartment integration
- Settings API using the modern `Settings.RegisterAddOnSetting()` approach
- CI/CD via GitHub Actions with BigWigsMods/packager for CurseForge, Wago, and WoWInterface
- Full MkDocs Material documentation site covering the entire addon development lifecycle
- Claude Code agents and skills for AI-assisted WoW addon development

## Quick Start

1. Copy the `template/` directory to your WoW `Interface/AddOns/` folder
2. Rename the folder and files to your addon name
3. Update `MyAddon.toc` with your addon's metadata
4. Edit `Init.lua`, `Core.lua`, and `Config.lua` with your logic
5. Type `/reload` in-game to load

For full instructions, see the [Getting Started](docs-site/docs/getting-started.md) guide.

## Template Contents

```
template/
├── MyAddon.toc          # TOC with full metadata reference
├── Init.lua             # Namespace setup, event dispatcher, slash commands
├── Core.lua             # Feature logic, frame creation, combat queue
├── Config.lua           # Modern Settings panel registration
├── .pkgmeta            # BigWigsMods/packager config
├── .luacheckrc         # Luacheck linting rules
├── .github/workflows/  # CI/CD release pipeline
└── Libs/               # LibStub, CallbackHandler
```

## Documentation

The `docs-site/` directory contains a full MkDocs Material site with 26 pages:

- **Reference** -- TOC format, Lua API, Events, Frames & Widgets, Security model
- **Midnight 12.0** -- What changed, coding patterns, Secret Values, CLEU removal
- **Better Addons** -- Building enhancements, Blizzard systems, code patterns, tutorials
- **AI Coding Guide** -- Overview, API cheat sheet, common pitfalls, code templates, prompt templates

Build the docs locally:

```bash
cd docs-site && pip install -r requirements.txt && mkdocs serve
```

## AI Integration

This project includes specialized Claude Code agents and skills for WoW addon development:

- `/wow-create` -- Scaffold a new addon
- `/wow-migrate` -- Migrate pre-12.0 code to Midnight
- `/wow-review` -- Review addon code for issues
- `/wow-debug` -- Diagnose addon problems
- `/wow-api` -- Look up WoW API documentation
- `/wow-research` -- Verify API claims against real sources
- `/wow-mode` -- Switch development philosophy (faithful, boundary, enhance, performance)

## Resources

- [WoW API Reference](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API)
- [Patch 12.0 API Changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes)
- [Blizzard UI Source](https://github.com/Gethe/wow-ui-source)
- [WoWUIDev Discord](https://discord.com/invite/txUg39Vhc6)

## License

MIT
