---
title: "Publishing & CI/CD"
description: "Zero to automated CurseForge releases — GitHub Actions, BigWigsMods/packager, .pkgmeta, and multi-platform deployment for WoW addons."
---

# Publishing & CI/CD

The template ships with everything you need for one-tag releases. Push a git tag, and GitHub Actions packages your addon and uploads it to CurseForge, Wago, WoWInterface, and GitHub Releases — automatically.

This guide walks you through the full setup from zero to automated releases.

---

## How It Works

```
git tag v1.0.0 && git push --tags
         │
         ▼
   GitHub Actions
         │
         ├── Luacheck lint (catches issues early)
         │
         ▼
   BigWigsMods/packager@v2
         │
         ├── Fetches external libraries from .pkgmeta
         ├── Replaces @project-version@ tokens
         ├── Strips @debug@ blocks
         ├── Generates changelog from git history
         ├── Creates zip + nolib zip
         │
         ▼
   Uploads to all platforms
         │
         ├── CurseForge  (CF_API_KEY)
         ├── Wago Addons  (WAGO_API_TOKEN)
         ├── WoWInterface (WOWI_API_TOKEN)
         └── GitHub Releases (GITHUB_TOKEN)
```

**The 30-second version:** Tag your commit, push, wait ~2 minutes, your addon appears on CurseForge. That's it.

!!! tip "Plugin Shortcut"
    If you scaffolded your addon with `/wow-create`, the release workflow, lint workflow, `.pkgmeta`, and `.luacheckrc` are already in place. Skip to [Step 1: Create Your CurseForge Project](#step-1-create-your-curseforge-project).

---

## Step 1: Create Your CurseForge Project

Before your CI pipeline can upload anything, you need a project on the platform.

1. Go to [CurseForge Author Dashboard](https://authors.curseforge.com/)
2. Click **Create Project**
3. Fill in:
    - **Game:** World of Warcraft
    - **Category:** Addons (pick the best sub-category)
    - **Name:** Your addon name
    - **Summary:** One-liner description
4. Submit and wait for approval (usually within 24 hours)

### Finding Your Project ID

Once approved, your project has a **numeric ID**. This is NOT the URL slug.

1. Open your project on CurseForge
2. Look in the **right sidebar** under "About Project"
3. The **Project ID** is a number like `123456`

Add these headers to your `.toc` file:

```toc
## X-Curse-Project-ID: 123456
## X-Wago-ID: aBcDeFgH
## X-WoWI-ID: 98765
```

!!! warning "Use the numeric ID"
    The CurseForge URL might show `/addons/cool-bags`, but the packager needs the **numeric** Project ID from the sidebar. Using the slug will fail silently.

---

## Step 2: Configure GitHub Secrets

The packager uses environment variables to authenticate with each platform. You store these as GitHub repository secrets.

### Tokens You Need

| Secret | Platform | Where to Get It |
|--------|----------|----------------|
| `CF_API_KEY` | CurseForge | [Author Dashboard → API Tokens](https://authors.curseforge.com/) |
| `WAGO_API_TOKEN` | Wago Addons | [addons.wago.io/account/apikeys](https://addons.wago.io/account/apikeys) |
| `WOWI_API_TOKEN` | WoWInterface | File Management → API Tokens |
| `GITHUB_TOKEN` | GitHub Releases | Auto-provided by GitHub Actions (no setup needed) |

### Adding Secrets to GitHub

1. Open your repo on GitHub
2. Go to **Settings → Secrets and variables → Actions**
3. Click **New repository secret**
4. Add each token with its exact name (e.g., `CF_API_KEY`)

!!! info "Missing secrets are fine"
    If a secret isn't set, the packager simply skips that platform. You can start with just `CF_API_KEY` and add others later. No errors, no failures — it just won't upload where it can't authenticate.

---

## Step 3: Enable Workflow Permissions

This is the **#1 setup mistake** people hit. GitHub Actions needs write permission to create GitHub Releases.

1. Go to your repo's **Settings → Actions → General**
2. Scroll to **Workflow permissions**
3. Select **Read and write permissions**
4. Save

Without this, the `GITHUB_TOKEN` has read-only access and the release upload silently fails.

---

## Step 4: The Release Workflow

Here's the full `release.yml` from the template, annotated:

```yaml title=".github/workflows/release.yml"
name: Package and Release

# Trigger: any tag push (v1.0.0, v2.1.0-beta, etc.)
on:
  push:
    tags:
      - "**"

jobs:
  # ---- Step 1: Lint ----
  # Catches Lua issues before packaging.
  # If lint fails, the release still proceeds (see the 'if' below).
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Luacheck
        uses: nebularg/actions-luacheck@v1
        with:
          args: "--no-color -q"
          annotate: warning

  # ---- Step 2: Package and Upload ----
  release:
    runs-on: ubuntu-latest
    needs: lint
    # Run even if lint fails — don't block a release on warnings.
    # But DO skip if the workflow was cancelled.
    if: always() && needs.lint.result != 'cancelled'

    env:
      # Each secret maps to a platform. Missing = skipped.
      CF_API_KEY: ${{ secrets.CF_API_KEY }}
      WOWI_API_TOKEN: ${{ secrets.WOWI_API_TOKEN }}
      WAGO_API_TOKEN: ${{ secrets.WAGO_API_TOKEN }}
      GITHUB_OAUTH: ${{ secrets.GITHUB_TOKEN }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          # Full git history — the packager needs this for changelogs.
          # Without it, you get an empty changelog on CurseForge.
          fetch-depth: 0

      - name: Package and Release
        uses: BigWigsMods/packager@v2
        # Auto-detects game version from your TOC's ## Interface: line.
        # For multi-game addons, see Step 10.
```

!!! info "Why `GITHUB_OAUTH` and not `GITHUB_TOKEN`?"
    The packager expects `GITHUB_OAUTH` as its environment variable name. The `${{ secrets.GITHUB_TOKEN }}` is the built-in token provided by GitHub Actions. The naming mismatch is intentional — it's how the packager's API works.

---

## Step 5: The Lint Workflow

A separate workflow runs Luacheck on every push and pull request — not just tag pushes.

```yaml title=".github/workflows/lint.yml"
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

### Your `.luacheckrc`

Luacheck needs to know about WoW-specific globals. The template includes a starter config:

```lua title=".luacheckrc"
std = "lua51"
max_line_length = false
exclude_files = { "Libs/", ".release/" }

ignore = {
    "11./SLASH_",     -- Setting slash command globals
    "212/self",       -- Unused self in mixin methods
    "212/event",      -- Unused event arg
}

globals = {
    "MyAddonDB",          -- Your SavedVariables
    "SLASH_MYADDON1",
    "SlashCmdList",
}

read_globals = {
    "LibStub",
    "CreateFrame", "UIParent", "GameTooltip",
    "C_Timer", "C_AddOns", "C_EditMode",
    "Mixin", "CreateFromMixins",
    "EventUtil", "EventRegistry", "Settings",
    "InCombatLockdown", "GetBuildInfo", "GetLocale",
    "hooksecurefunc", "issecretvalue",
    -- Add your own as needed
}
```

To add your own globals: if Luacheck reports `accessing undefined global 'C_Housing'`, add `"C_Housing"` to the `read_globals` table.

!!! tip "Auto-generated configs"
    [Jayrgo/wow-luacheckrc](https://github.com/Jayrgo/wow-luacheckrc) parses Blizzard's interface resources to generate a complete `.luacheckrc` covering every known WoW global. Far more comprehensive than maintaining one by hand.

---

## Step 6: .pkgmeta Deep Dive

The `.pkgmeta` file controls how BigWigsMods/packager builds your addon. It's YAML — indentation matters.

### Full Annotated Reference

```yaml title=".pkgmeta"
# ---- Package Name ----
# The folder name inside the zip. Must match your .toc filename.
package-as: MyAddon

# ---- External Libraries ----
# Downloaded fresh at build time. NOT committed to your repo.
# Add Libs/ to .gitignore.
externals:
  # SVN sources (CurseForge repositories)
  Libs/LibStub:
    url: https://repos.curseforge.com/wow/libstub/trunk
    tag: latest
  Libs/CallbackHandler-1.0:
    url: https://repos.curseforge.com/wow/callbackhandler/trunk/CallbackHandler-1.0
    tag: latest

  # Git sources (GitHub repositories)
  # Libs/LibDataBroker-1.1:
  #   url: https://github.com/tekkub/libdatabroker-1-1
  #   tag: latest

  # Pin to a specific version (recommended for stability)
  # Libs/AceDB-3.0:
  #   url: https://repos.curseforge.com/wow/ace3/trunk/AceDB-3.0
  #   tag: v3.0.12

# ---- Nolib Package ----
# Creates a second zip WITHOUT embedded libraries.
# Players who already have the libs from other addons use this.
enable-nolib-creation: yes

# ---- Files to Exclude ----
# These exist in git but should NOT ship to players.
ignore:
  - .github
  - .vscode
  - .luacheckrc
  - .luarc.json
  - .editorconfig
  - .stylua.toml
  - .gitignore
  - .cursorrules
  - CLAUDE.md
  - README.md
  - CHANGELOG.md
  - tests/

# ---- Manual Changelog ----
# Use your own CHANGELOG.md instead of auto-generating from commits.
# manual-changelog:
#   filename: CHANGELOG.md
#   markup-type: markdown

# ---- Move Folders (Monorepo) ----
# For repos with multiple addons (like OmniCC + OmniCC_Config):
# move-folders:
#   MyAddon/MyAddon_Config: MyAddon_Config

# ---- CurseForge Metadata ----
# Tells CurseForge about library dependencies.
# embedded-libraries:
#   - libstub
#   - callbackhandler
# optional-dependencies:
#   - masque
#   - elvui
```

### Common External Libraries

| Library | URL | Purpose |
|---------|-----|---------|
| LibStub | `repos.curseforge.com/wow/libstub/trunk` | Library version management |
| CallbackHandler-1.0 | `repos.curseforge.com/wow/callbackhandler/trunk/CallbackHandler-1.0` | Event callbacks for libraries |
| AceDB-3.0 | `repos.curseforge.com/wow/ace3/trunk/AceDB-3.0` | Saved variable management |
| LibDBIcon-1.0 | `repos.curseforge.com/wow/libdbicon-1-0/trunk/LibDBIcon-1.0` | Minimap button |
| LibDataBroker-1.1 | `github.com/tekkub/libdatabroker-1-1` | Data broker framework |

!!! danger "YAML indentation"
    `.pkgmeta` is YAML — use **spaces, not tabs**. Incorrect indentation silently breaks external library resolution. If your `Libs/` folder is empty after a build, check your indentation first.

---

## Step 7: Version & Keyword Substitution

The packager replaces special `@keywords@` in your source files at build time.

### Available Keywords

| Keyword | Replaced With | Example |
|---------|--------------|---------|
| `@project-version@` | Git tag name | `v1.0.0` |
| `@project-hash@` | Full commit SHA | `a1b2c3d4e5f6...` |
| `@project-abbreviated-hash@` | Short commit SHA | `a1b2c3d` |
| `@project-author@` | Last commit author | `YourName` |
| `@project-date-iso@` | ISO date of last commit | `2026-03-15T10:30:00Z` |
| `@project-date-integer@` | Date as integer | `20260315103000` |
| `@project-timestamp@` | Unix timestamp | `1773756600` |
| `@project-revision@` | Number of commits | `42` |

### Conditional Blocks

| Block | When It's Included |
|-------|--------------------|
| `@debug@ ... @end-debug@` | Development only — **stripped** from release builds |
| `@alpha@ ... @end-alpha@` | Only in **untagged** (alpha) builds |
| `@do-not-package@ ... @end-do-not-package@` | **Never** included in packages |
| `#@no-lib-strip@ ... #@end-no-lib-strip@` | Stripped from `-nolib` package variant |

### Best Practice

Always use `@project-version@` in your TOC instead of a hardcoded version:

```toc
## Version: @project-version@
```

On tagged builds, this becomes `v1.0.0`. On untagged builds, it becomes a commit hash — making it easy to identify development builds.

In Lua, you can use conditional blocks for debug code:

```lua
--@debug@
print("DEBUG: This line only exists in development")
--@end-debug@
```

The `--` Lua comment prefix ensures the code runs during development (it's just a comment followed by code). The packager strips the entire block — comments and all — from release builds.

---

## Step 8: Your First Release

Everything configured? Let's ship it.

### 1. Commit Everything

```bash
git add -A
git commit -m "Ready for first release"
```

### 2. Create an Annotated Tag

```bash
git tag -a v1.0.0 -m "Initial release"
```

!!! info "Annotated vs. Lightweight Tags"
    Use `-a` for annotated tags. They include a message and author, and the packager uses the tag message as the release title on GitHub. Lightweight tags (`git tag v1.0.0` without `-a`) work too, but annotated is the convention.

### 3. Push the Tag

```bash
git push origin v1.0.0
```

Or push all tags at once:

```bash
git push --tags
```

### 4. Watch the Actions Tab

Open your repo on GitHub and click the **Actions** tab. You'll see "Package and Release" running. It typically takes 1-2 minutes.

### 5. Verify

- **GitHub:** Check the Releases page — you should see `v1.0.0` with a zip attached
- **CurseForge:** Check your project page — the new file should appear within a few minutes
- **Wago/WoWInterface:** Same — check your project pages

!!! success "Congratulations"
    Your addon is live. Every future release is just `git tag` + `git push`.

---

## Step 9: Alpha & Beta Channels

CurseForge supports three release channels: **release**, **beta**, and **alpha**. The packager determines the channel from your tag name.

### Channel Detection

| Tag Pattern | Channel |
|-------------|---------|
| `v1.0.0` | Release |
| `v1.0.0-beta` or `v1.0.0-beta.1` | Beta |
| `v1.0.0-alpha` or `v1.0.0-alpha.3` | Alpha |
| Untagged push (no tag) | Alpha |

### Auto-Alpha on Every Push

Want every push to `main` to create an alpha build on CurseForge? Add this trigger to your `release.yml`:

```yaml
on:
  push:
    tags:
      - "**"
    branches:
      - main
```

Now tagged pushes create releases, and untagged pushes to `main` create alpha builds. Alpha users who follow your addon on CurseForge get automatic updates between releases.

!!! warning "Alpha builds are public"
    Alpha builds appear on your CurseForge project page. They're installable by anyone who opts into alpha releases in their addon manager. Make sure `main` is always in a working state.

---

## Step 10: Multi-Platform Publishing

### Adding WoWInterface

1. Create your addon project on [WoWInterface](https://www.wowinterface.com/)
2. Get your numeric project ID from the file management page
3. Add to your TOC: `## X-WoWI-ID: 98765`
4. Add `WOWI_API_TOKEN` to your GitHub secrets
5. Push a tag — done

### Adding Wago Addons

1. Create your addon on [addons.wago.io](https://addons.wago.io/)
2. Get your project ID (an alphanumeric string like `aBcDeFgH`)
3. Add to your TOC: `## X-Wago-ID: aBcDeFgH`
4. Add `WAGO_API_TOKEN` to your GitHub secrets
5. Push a tag — done

### Skipping a Platform

Set the project ID to `0` in your TOC to explicitly skip a platform:

```toc
## X-Curse-Project-ID: 123456
## X-Wago-ID: aBcDeFgH
## X-WoWI-ID: 0
```

Or simply don't set the secret — the packager skips platforms with no authentication token.

### Multi-Game Builds

If your addon supports both Retail and Classic, pass the `-g` flag to the packager:

```yaml
- name: Package and Release
  uses: BigWigsMods/packager@v2
  with:
    args: -g retail -g classic
```

This builds separate packages for each game version, reading the appropriate `.toc` file for each (e.g., `MyAddon_Vanilla.toc`, `MyAddon_TBC.toc`).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Empty changelog on CurseForge | Missing `fetch-depth: 0` in checkout step | Add `fetch-depth: 0` to the `actions/checkout` step |
| GitHub Release not created | Workflow lacks write permissions | Settings → Actions → General → **Read and write permissions** |
| CurseForge upload fails | Wrong project ID or expired API token | Verify the **numeric** project ID from the sidebar; regenerate your `CF_API_KEY` |
| "No upload targets" in packager log | No secrets configured AND no project IDs in TOC | Add at least one `X-Curse-Project-ID` / `X-Wago-ID` header and its matching secret |
| `Libs/` folder empty after build | `.pkgmeta` YAML indentation error | Use spaces (not tabs), check externals nesting is correct |
| Tag triggers unrelated workflows | Tag pattern `**` catches non-release tags | Use `v*` instead of `**` if you have non-release tags |
| Nolib package missing libraries user expects | Nolib packages strip ALL embedded libs | This is intentional — nolib is for users who install libs separately |
| `@project-version@` not replaced | File is in the `ignore` list or outside `package-as` folder | Ensure the file is included in the package and not in `.pkgmeta` ignore |
| Lint blocks the release | Default GitHub Actions behavior | The template uses `if: always()` so lint warnings don't block releases |
| Packager can't detect game version | Missing or malformed `## Interface:` in TOC | Ensure `## Interface: 120001` (no extra spaces, correct format) |

---

## Quick Reference

Everything you need for automated releases, in one checklist:

### The 5 Things

1. **TOC headers** — Add platform project IDs:
    ```toc
    ## X-Curse-Project-ID: 123456
    ## X-Wago-ID: aBcDeFgH
    ## Version: @project-version@
    ```

2. **GitHub Secrets** — Add API tokens:
    ```
    CF_API_KEY, WAGO_API_TOKEN, WOWI_API_TOKEN
    ```

3. **Workflow permissions** — Enable write access:
    ```
    Settings → Actions → General → Read and write permissions
    ```

4. **Tag your release:**
    ```bash
    git tag -a v1.0.0 -m "Initial release"
    ```

5. **Push:**
    ```bash
    git push origin v1.0.0
    ```

### Common Tag Commands

```bash
# Create annotated tag
git tag -a v1.0.0 -m "Initial release"

# Push a specific tag
git push origin v1.0.0

# Push all tags
git push --tags

# List existing tags
git tag -l

# Delete a local tag (if you made a mistake)
git tag -d v1.0.0

# Delete a remote tag (use with caution)
git push origin --delete v1.0.0
```

---

## Further Reading

- **[Starter Template](starter-template.md)** — Full template walkthrough, pro showcase, and AI workflow
- **[BigWigsMods/packager wiki](https://github.com/BigWigsMods/packager/wiki)** — Complete packager documentation
- **[GitHub Actions workflow reference](https://github.com/BigWigsMods/packager/wiki/GitHub-Actions-workflow)** — Advanced workflow configurations
- **[Getting Started](getting-started.md)** — If you haven't built your first addon yet, start here
