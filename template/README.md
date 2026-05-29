# MyAddon

> A WoW addon built with the [Midnight Addon Template](https://github.com/FRIKKern/WoW-Midnight-Addon-Template) for Patch 12.0+.

## Features

- Feature 1
- Feature 2

## Installation

1. Download the latest release from [CurseForge](https://www.curseforge.com/wow/addons/myaddon) or [GitHub Releases](https://github.com/YourName/MyAddon/releases)
2. Extract `MyAddon/` into `World of Warcraft/_retail_/Interface/AddOns/`
3. Restart WoW or type `/reload`

## Usage

- `/myaddon` — show help
- `/myaddon config` — open settings
- `/myaddon toggle` — enable/disable
- `/myaddon reset` — reset settings to defaults
- Find **MyAddon** in the minimap Addon Compartment dropdown

## Configuration

Open the settings panel via `/myaddon config` or Game Menu → Options → AddOns → MyAddon.

## Development

### Prerequisites

- Git
- [VS Code](https://code.visualstudio.com/) with recommended extensions (auto-prompted on open)

### Setup

1. Clone this repo
2. Symlink or copy to your AddOns folder:
   ```bash
   ln -s "$(pwd)" "/path/to/World of Warcraft/_retail_/Interface/AddOns/MyAddon"
   ```
3. Edit, `/reload` in-game, repeat

### Linting

```bash
luacheck .
```

### Releasing

1. Commit your changes
2. Tag: `git tag -a v1.0.0 -m "Release v1.0.0"`
3. Push: `git push origin v1.0.0`
4. GitHub Actions packages and uploads to CurseForge/Wago/WoWInterface automatically

See [Publishing & CI/CD](https://frikk.dev/WoW-Midnight-Addon-Template/publishing/) for the full guide.

## Built With

- [Midnight Addon Template](https://github.com/FRIKKern/WoW-Midnight-Addon-Template) — Blizzard-faithful patterns for WoW 12.0+
- [BigWigsMods/packager](https://github.com/BigWigsMods/packager) — automated addon packaging and distribution
- [Claude Code](https://claude.com/claude-code) — AI-assisted addon development with [better-addons](https://github.com/FRIKKern/better-addons) plugin

## License

MIT — see [LICENSE](LICENSE).
