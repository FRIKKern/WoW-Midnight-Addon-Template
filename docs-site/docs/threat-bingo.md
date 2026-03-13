---
title: "Threat Bingo: The Midnight Reality Check"
description: "11 days into WoW Midnight 12.0 — verified addon casualties, what actually still works, the next bingo card, and the March 24 Mythic question. Every claim sourced and fact-checked."
---

# :game_die: Threat Bingo: The Midnight Reality Check

!!! danger "This is NOT speculation. We are 11 days into Patch 12.0."
    **WoW: Midnight launched March 2, 2026.** The Addonpocalypse already happened. The bodies are on the ground. This page tracks what's confirmed dead, what survived, what's coming next, and which addons are living on borrowed time.

    *Last updated: March 13, 2026 — Day 11 of Midnight*

---

## :chart_with_upwards_trend: The Verified Scorecard

Every addon below has been checked against CurseForge release dates, GitHub commits, developer statements, and community reports. No rumors. No cope.

| Addon | Status | Evidence | Sources |
|-------|--------|----------|---------|
| **WeakAuras** | :skull: **DEAD** | Team officially refused to ship Midnight version. *"The core value proposition of WeakAuras isn't compatible with the direction Blizzard is taking."* — Stanzilla | [Wowhead](https://www.wowhead.com/news/no-weakauras-addon-for-midnight-378740), [Icy Veins](https://www.icy-veins.com/wow/news/weakauras-responds-to-addon-limitation-loosening-in-midnight/) |
| **Hekili** | :skull: **DEAD** | Rotation helpers are exactly what Secret Values targeted. Cannot function. | [Blizzard Forums](https://us.forums.blizzard.com/en/wow/t/list-of-addons-going-away-in-midnight/2214572) |
| **GTFO** | :skull: **DEAD** | Cannot detect avoidable damage — combat data is secrets | [Blizzard Forums](https://us.forums.blizzard.com/en/wow/t/list-of-addons-going-away-in-midnight/2214572) |
| **Shadowed Unit Frames** | :skull: **DEAD** | Developer statement: *"Due to the strict limitations on addons introduced by Blizzard in Midnight, SUF will not be updated."* | [EU Forums](https://eu.forums.blizzard.com/en/wow/t/add-on-devs-refused-to-continue-in-the-midnight-era-due-to-restrictions-imposed-by-blizz/602690) |
| **Details!** | :red_circle: **GUTTED** | Now a skin over Blizzard's `C_DamageMeter`. Lost: overhealing, CC breaks, buff uptime, pet damage segmentation, chat reporting | [Wowhead](https://www.wowhead.com/news/blizzards-damage-meter-shortcomings-in-midnight-pre-patch-and-addon-alternatives-379992), [CurseForge](https://www.curseforge.com/wow/addons/details) |
| **DBM** | :red_circle: **GUTTED** | v12.0.30 released Mar 12. Reformats Blizzard's Boss Timeline. Lost: independent CLEU parsing, custom ability names, external audio countdowns | [CurseForge](https://www.curseforge.com/wow/addons/deadly-boss-mods/files/7745941), [Wowhead](https://www.wowhead.com/news/boss-mod-addons-in-midnight-teaching-old-mods-new-tricks-380024) |
| **BigWigs** | :red_circle: **GUTTED** | 183M+ downloads. Same architectural pivot as DBM — visual layer over native encounter system | [CurseForge](https://www.curseforge.com/wow/addons/bigwigs), [Wowhead](https://www.wowhead.com/news/boss-mod-addons-in-midnight-teaching-old-mods-new-tricks-380024) |
| **Plater** | :red_circle: **GUTTED** | v635 Mar 4. Lost NPC-specific coloring, aura-based scaling, interrupt tracking, fixate, time-to-death in instances | [Wowhead](https://www.wowhead.com/news/nameplates-in-midnight-whats-changing-and-what-add-ons-can-i-use-379924), [Xepheris](https://gerritalex.de/blog/nameplates-in-midnight) |
| **ElvUI** | :yellow_circle: **ADAPTED** | v15.0.0 shipped. Initially quit (Oct 2025), came back (Dec 2025). Lost Style Filters, Cutaway Bars, Portraits | [Wowhead](https://www.wowhead.com/news/elvui-now-updated-for-midnight-pre-patch-380096), [CurseForge](https://www.curseforge.com/wow/addons/elvui) |
| **Cell** | :yellow_circle: **ADAPTED** | v274 retail. Best-documented adaptation (PR #457). Per-aura secret detection, CLEU-to-unit-event migration, message queuing | [GitHub PR #457](https://github.com/enderneko/Cell/pull/457), [Icy Veins](https://www.icy-veins.com/wow/news/cell-confirms-a-stripped-down-version-for-midnight/) |
| **MRT** | :green_circle: **THRIVING** | v5260 for 12.0.1. Raid cooldowns, external buff assignments, planning notes all working | [CurseForge](https://www.curseforge.com/wow/addons/method-raid-tools) |
| **Northern Sky** | :green_circle: **THRIVING** | 3.2M+ downloads. WeakAura raid pack replacement built for 12.0 constraints | [CurseForge](https://www.curseforge.com/wow/addons/northern-sky-raid-tools) |
| **Warcraft Logs** | :green_circle: **THRIVING** | `/combatlog` disk file is **completely unaffected** by addon restrictions. Full analysis intact | Community consensus |
| **HomeBound** | :green_circle: **THRIVING** | 3M+ downloads. Housing is unrestricted greenfield for addon devs | [CurseForge](https://www.curseforge.com/wow/addons/home-bound) |
| **BetterBlizzFrames** | :green_circle: **THRIVING** | 3.9M+ downloads. "Enhance, don't replace" — the winning pattern | [CurseForge](https://www.curseforge.com/wow/addons/betterblizzframes) |

!!! info "Legend"
    :skull: **DEAD** = Not shipping for Midnight, developer confirmed |
    :red_circle: **GUTTED** = Shipping but core features lost, reduced to Blizzard skin |
    :yellow_circle: **ADAPTED** = Shipping with significant changes, functionality preserved where possible |
    :green_circle: **THRIVING** = Fully functional or grew post-Midnight

---

## :crystal_ball: Predictions vs Reality

The community had 6 months of beta to predict the Addonpocalypse. How'd they do?

| Prediction | What Actually Happened | Verdict |
|------------|----------------------|---------|
| *"WeakAuras will find a workaround"* | Team officially quit. *"The changes they have made don't really pass muster."* — WeakAuras team | :skull: **Dead wrong** |
| *"ElvUI is gone forever"* | Quit in October 2025, came back in December 2025, shipped v15.0.0 | :yellow_circle: **Wrong** |
| *"DBM/BigWigs will die"* | MysticalOS met with Ion personally. Boss mods survived as reskins of native systems | :yellow_circle: **Wrong** |
| *"Damage meters are dead"* | Details! is a skin over `C_DamageMeter`. Numbers are real, features are gutted | :red_circle: **Half right** |
| *"CLEU is completely removed"* | Still fires, but payload is Secret Values in instances. Effectively useless for addon logic | :red_circle: **Half right** |
| *"Blizzard will back down"* | Made concessions (healer spells, whitelisted auras) but core restrictions held | :red_circle: **Mostly wrong** |
| *"This will kill WoW"* | Launch described as "broadly positive." PC Gamer: *"I don't miss combat addons"* | :skull: **Dead wrong** |
| *"Healers are screwed"* | Scrambled hard. Cell, Grid2, Clique adapted. VuhDo, Healbot being updated. Danders Frames emerging | :yellow_circle: **Overstated** |
| *"Housing addons will boom"* | C_Housing APIs unrestricted. HomeBound 3M+, ADT, HomeDecor all thriving | :green_circle: **Correct** |
| *"AI will write the replacement addons"* | NephUI ("vibecoded"), QuaziiUI (alleged code theft) — both imploded spectacularly | :fire: **Cursed prophecy** |

---

## :slot_machine: The NEXT Bingo Card

What happens in the next 90 days? Mark your card.

### 12.0.5 PTR (Active Now)

| | B | I | N | G | O |
|---|---|---|---|---|---|
| **1** | Whitelist expands to 20+ spells | `C_DamageMeter` gets overhealing | Details! regains chat reporting | Plater gets NPC coloring back (open world) | New `C_Encounter` APIs |
| **2** | Cell becomes default healer rec | VuhDo ships stable 12.0 | Housing addons hit 10M combined downloads | "Better" addon prefix trend continues | Blizzard absorbs another addon feature |
| **3** | WeakAuras team reconsiders | Ion posts another blog | **FREE SPACE: Macro exploit patched** | First "Midnight-native" WA alternative hits 1M downloads | Vibecoded addon drama round 2 |
| **4** | Secret Values relaxed for LFR | Addon comms unlocked between pulls | Healer spell whitelist made permanent | New `C_CombatLog` APIs | Nameplate addon renaissance |
| **5** | Console port announced | 12.0.5 breaks 50+ addons again | Community fork of dead addon goes viral | Blizzard hires addon dev | Wago.io competitor launches |

??? tip "B1: Whitelist expands to 20+ spells"
    Currently 8 cooldown-only and 8 full aura spells whitelisted. Blizzard has been progressively adding spells since beta. The 12.0.5 cycle is the most likely time for a major expansion.

??? tip "I1: `C_DamageMeter` gets overhealing"
    The #1 complaint about the native meter. [Wowhead documented the shortcomings](https://www.wowhead.com/news/blizzards-damage-meter-shortcomings-in-midnight-pre-patch-and-addon-alternatives-379992). Blizzard is actively improving it (minimize button coming in 12.0.5).

??? tip "N1: Details! regains chat reporting"
    Addon comms are locked during encounters but the chat report feature was a social function. May return if Blizzard relaxes post-encounter reporting.

??? tip "G1: Plater gets NPC coloring back (open world)"
    Nameplate NPC identification works outside instances. The question is whether Blizzard relaxes instance restrictions for non-boss mobs.

??? tip "O1: New `C_Encounter` APIs"
    `C_EncounterTimeline` (29 functions) and `C_EncounterWarnings` (5 functions) were just the start. More encounter data exposure would help boss mods.

??? tip "B3: WeakAuras team reconsiders"
    Their Patreon statement was firm: *"Purely technical issue."* But if Blizzard significantly expands whitelists, the calculus could change. Currently rated :question: UNLIKELY.

??? tip "N3: FREE SPACE"
    Macro exploits for addon workarounds are being patched as fast as they're found. [March 11 hotfix](https://news.blizzard.com/en-us/article/24266320/hotfixes-march-11-2026) already hit target marker spam and encounter chat macros.

??? tip "B5: Console port announced"
    [Widely speculated](https://blizzardwatch.com/2025/11/03/heck-happening-wow-addons-midnight/) as the hidden motivation behind addon restrictions. Console platforms can't support the addon ecosystem. :question: UNVERIFIED but the theory keeps gaining traction.

### :calendar: March 24: Mythic Raids Unlock

| | B | I | N | G | O |
|---|---|---|---|---|---|
| **1** | World First race completes without WeakAuras | Guild uses macro exploits, gets banned | DBM proves sufficient for Mythic | Healers demand rollback | Boss mods get emergency whitelist |
| **2** | WarcraftLogs traffic hits all-time high | Manual callouts replace automated alerts | First Mythic kill takes 200+ pulls | Guild develops internal addon | Stream overlay replaces in-game data |
| **3** | Ion comments on race difficulty | **FREE SPACE: Someone blames wipe on no addons** | Community praises "skill matters" | Competitive players quit en masse | New boss mod emerges from race |
| **4** | Echo/Liquid publish addon-free guide | Secret Values cause raid-wide Lua errors | Blizzard hotfixes mid-race | MRT becomes MVP | Private Aura audio bugs discovered |
| **5** | Race is faster than predicted | Race is slower than predicted | Both guilds use identical addon setups | Viewer experience improves (less addon clutter) | Blizzard declares addon policy success |

??? warning "The Big Question: World First Without WeakAuras"
    For the first time since WeakAuras became dominant, the World First race will happen without it. DBM and BigWigs reformatted Blizzard's Boss Timeline. MRT handles coordination. Northern Sky covers raid packs. But **zero custom per-mechanic triggers** means every mechanic response is manual or relies on Blizzard's native alerts.

### :telescope: 12.1 Horizon

| | B | I | N | G | O |
|---|---|---|---|---|---|
| **1** | Secret Values v2 (more relaxed) | New addon framework announced | WeakAuras 2.0 by new team | Housing API explosion | Addon certification program |
| **2** | CLEU restored for open world | Rotation helpers return (limited) | Blizzard open-sources native addons | Community-driven whitelist voting | Addon Disarmament declared success |
| **3** | 50% of top addons are "Midnight-native" | **FREE SPACE: More API removals** | Old addon devs return | "Enhance, don't replace" codified | Competitive scene stabilizes |
| **4** | `C_DamageMeter` reaches feature parity | Console beta begins | Addon comms fully restored | New taint system iteration | Player housing goes competitive |
| **5** | 12.1 breaks everything again | Community builds addon SDK | Blizzard addon store launches | Vibecoded addons mature into real projects | Ion does addon dev AMA |

---

## :trophy: API Exploit Tier List

What ACTUALLY still works for addon developers in 12.0.1? Tested and verified.

### :green_circle: SAFE — Works, Blizzard-sanctioned, use freely

| Technique | What It Does | Example |
|-----------|-------------|---------|
| `hooksecurefunc()` | Post-hook any non-forbidden function | `hooksecurefunc("CastSpellByName", myHook)` |
| `C_Timer.After/NewTicker` | Timers with no restrictions | `C_Timer.After(1, callback)` |
| Frame strata/level (own frames) | Full control of addon UI layers | `myFrame:SetFrameStrata("HIGH")` |
| `getmetatable()` / `GetFrameMetatable()` | Read/extend frame metatables | `GetFrameMetatable().__index.MyMethod = func` |
| `RegisterEventCallback()` | **NEW 12.0** Frame-free event registration | `RegisterEventCallback("PLAYER_LOGIN", func)` |
| `issecretvalue()` | Detect secrets, degrade gracefully | `if issecretvalue(hp) then showBar() else showNumber() end` |
| `C_DamageMeter.*` | Access native meter data for reskinning | `C_DamageMeter.GetCombatSessionSourceFromID(id, type)` |
| Color Curves / `SetAlphaFromBoolean` | Visualize secret data without reading it | `region:SetAlphaFromBoolean(secretBool, 1.0, 0.3)` |
| Duration Objects | Display cooldowns from secret time values | `cooldown:SetCooldownFromDurationObject(durObj)` |
| `CreateUnitHealPredictionCalculator()` | Heal predictions without raw numbers | `local calc = CreateUnitHealPredictionCalculator()` |
| C_Housing / C_HousingCatalog / etc. | **Unrestricted** housing APIs | Full greenfield |
| Addon comms (outside encounters) | `C_ChatInfo.SendAddonMessage` | Normal between pulls |
| `/combatlog` disk file | Warcraft Logs, unaffected | Always works |

### :yellow_circle: BORROWED TIME — Works today, expect restrictions

| Technique | Risk Factor | Why |
|-----------|-------------|-----|
| Spell whitelists (8+8 spells) | Blizzard warned: *"We will likely re-protect these spells once our own filtering solution is in place"* | [Wowhead](https://www.wowhead.com/news/many-healer-spells-no-longer-secret-aura-n-midnight-launch-380525) |
| Healer spell visibility | Same warning — temporary concession for healers | Blizzard blue post |
| `UNIT_SPELLCAST_SUCCEEDED` in instances | Fires with real data today, but on the radar | Used by DBM/BigWigs |
| Frame strata on Blizzard frames (out of combat) | Restricted in-combat since 11.1.7, full lockdown possible | [Wiki](https://warcraft.wiki.gg/wiki/API_Frame_SetFrameStrata) |
| Tooltip scanning (items outside combat) | Works, but `C_TooltipInfo` callbacks receive secrets for unit data in instances | [AllTheThings #2261](https://github.com/ATTWoWAddon/AllTheThings/issues/2261) |
| CLEU in open world | Fires with real values outside instances/combat. Could be restricted further | [Wiki](https://warcraft.wiki.gg/wiki/COMBAT_LOG_EVENT) |

### :red_circle: NEXT ON THE BLOCK — Likely restricted in 12.0.5/12.1

| Technique | Why It's Targeted |
|-----------|-------------------|
| Hard-coded boss timers | Wowhead investigation: *"Addons continue to provide a massive advantage"* — [source](https://www.wowhead.com/news/addons-continue-to-provide-a-massive-advantage-in-midnight-despite-blizzards-379870) |
| Macro-based workarounds | March 11 hotfix already restricted target markers and encounter chat macros |
| Nameplate mob-type coloring in instances | Plater's remaining useful feature in instances |
| Addon communication timing exploits | Queue-and-flush on `ENCOUNTER_END` is a known pattern |

### :skull: ALREADY DEAD — Confirmed blocked, don't bother

| Technique | How It Died |
|-----------|------------|
| CLEU parsing in instances | Combat log data wrapped in KStrings/Secret Values. `C_CombatLog.IsCombatLogRestricted()` returns true |
| `secretunwrap()` | Removed from global table entirely |
| UI reload to clear secrets | Bug fixed |
| Aura instance ID comparison bypass | Patched |
| Secret value arithmetic/comparison | Lua error on any operation: compare, math, length, index, truthiness test |
| `tonumber()` on secrets | Explicitly blocked |
| Addon comms during encounters | `C_ChatInfo.InChatMessagingLockdown()` returns true |
| Macro whisper relay to external players | Whispers restricted to in-instance targets during encounters |
| Reading secret values at all | *"Combat events are in a black box; addons can change the size or shape of the box, and they can paint it a different color, but what they can't do is look inside."* — Blizzard |

---

## :rotating_light: Top 10 Threatened Addons

Ranked by impact and evidence of vulnerability.

### 1. :skull: WeakAuras — CONFIRMED DEAD

> *"We don't currently plan to release a WeakAuras version for Midnight. It's a purely technical issue."* — WeakAuras Team

**Replacements emerging:** Arc UI (cooldowns), Northern Sky (raid packs), OmniCD (group CDs), MPlusTimer (M+ timer), Platynator (nameplate auras), MidnightSimpleAuras (basic alerts), TargetedSpells (spell alerts)

Sources: [Wowhead](https://www.wowhead.com/news/no-weakauras-addon-for-midnight-378740), [Patreon statement](https://www.patreon.com/posts/midnight-144610594)

### 2. :skull: Hekili — CONFIRMED DEAD

Rotation helpers are the canonical example of what Secret Values targeted. *"A computer with access to complete information about the current combat state will be able to make the correct decision far faster than any human."* — Ion

**Native replacement:** Blizzard's One-Button Rotation and Assisted Highlight tools

### 3. :skull: GTFO — CONFIRMED DEAD

Audio warnings for standing in fire require knowing damage source. Secret Values block this entirely.

**Native replacement:** Blizzard's Combat Audio Alerts with avoidable damage detection

### 4. :skull: Shadowed Unit Frames — CONFIRMED DEAD

Developer explicitly refused to update. Unit frame data restrictions made it unviable.

**Replacements:** Unhalted Unit Frames (445K+), MidnightSimpleUnitFrames (171K+), BetterBlizzFrames (3.9M+)

### 5. :red_circle: Details! — GUTTED, Major Features Lost

330M+ downloads can't be wrong, but the addon is a shadow of itself. Server-validated numbers are real but unprocessable.

**What's gone:** Overhealing, CC breaks, buff uptime, pet segmentation, spell merging, advanced death breakdowns, chat reporting

**What remains:** Visual presentation layer, familiar UI, per-ability breakdowns via `C_DamageMeter`

Sources: [Wowhead](https://www.wowhead.com/news/blizzards-damage-meter-shortcomings-in-midnight-pre-patch-and-addon-alternatives-379992)

### 6. :red_circle: Plater Nameplates — GUTTED in Instances

v635 ships but lost the features that made it essential for M+/raids.

**What's gone (instances):** NPC-specific coloring, aura-based scaling, cast bar interrupt tracking, fixate detection, time-to-death, execute range, health thresholds, custom fonts

**Rising alternative:** Platynator — "design within the box" philosophy

Sources: [Xepheris technical deep-dive](https://gerritalex.de/blog/nameplates-in-midnight)

### 7. :red_circle: DBM / BigWigs — GUTTED but Essential

MysticalOS (DBM) met personally with Ion and lead engineer Andy Churchill. 30+ point releases since 12.0. Still the best option for encounters — just fundamentally different under the hood.

**The shift:** Independent combat analysis :arrow_right: visual layer over Blizzard's encounter system

Sources: [Wowhead](https://www.wowhead.com/news/more-boss-mod-functionality-coming-in-midnight-dbm-and-big-wigs-meet-with-ion-379149)

### 8. :yellow_circle: ElvUI — ADAPTED with Scars

Dramatic arc: quit Oct 2025, returned Dec 2025, shipped v15.0.0. Lost combat-dependent features but core UI overhaul intact.

**The QuaziiUI drama:** When ElvUI initially quit, QuaziiUI swooped in with alleged code theft. ElvUI's LuckyOne called it out, QuaziiUI imploded.

Sources: [Icy Veins](https://www.icy-veins.com/wow/news/elvui-is-done-for-midnight-wows-most-popular-addon-just-quit/), [Kaylriene](https://kaylriene.com/2026/01/27/a-mini-summary-of-week-one-of-the-new-wow-ui-era-blizzards-own-lua-errors-the-vibecoded-addon-wars-of-2026/)

### 9. :yellow_circle: VuhDo / Healbot — IN DEVELOPMENT

The healer addon question consumed more forum threads than any other topic. Both are being updated but weren't stable at launch.

**Meanwhile:** Cell shipped a thorough 12.0 rewrite. Grid2 and Clique confirmed working. Danders Frames gaining traction as the new kid.

Sources: [Blizzard Forums](https://us.forums.blizzard.com/en/wow/t/healing-addons-their-status/2235343)

### 10. :yellow_circle: AllTheThings — TOOLTIP ISSUES

Filed issue #2261 documenting tooltip Lua errors from Secret Values. "PlsFixMe Midnight Tooltips" addon exists solely to suppress these errors.

Sources: [GitHub #2261](https://github.com/ATTWoWAddon/AllTheThings/issues/2261), [CurseForge](https://www.curseforge.com/wow/addons/plsfixme-midnight-tooltips)

---

## :shield: Threat Matrix

How different addon categories face different threat types:

| Category | Secret Values | CLEU Loss | Comm Lockdown | API Removal | Taint Expansion | Overall |
|----------|:---:|:---:|:---:|:---:|:---:|:---:|
| **Rotation Helpers** | :skull: | :skull: | — | :skull: | :skull: | :skull: **EXTINCT** |
| **Damage Meters** | :red_circle: | :red_circle: | — | :yellow_circle: | :yellow_circle: | :red_circle: **GUTTED** |
| **Boss Mods** | :red_circle: | :red_circle: | :yellow_circle: | :yellow_circle: | :yellow_circle: | :red_circle: **GUTTED** |
| **Healing Frames** | :yellow_circle: | :yellow_circle: | :yellow_circle: | :green_circle: | :yellow_circle: | :yellow_circle: **STRESSED** |
| **Nameplates** | :red_circle: | :yellow_circle: | — | :yellow_circle: | :yellow_circle: | :red_circle: **GUTTED (instances)** |
| **Unit Frames** | :yellow_circle: | :yellow_circle: | — | :yellow_circle: | :yellow_circle: | :yellow_circle: **STRESSED** |
| **UI Overhauls** | :green_circle: | :green_circle: | — | :yellow_circle: | :yellow_circle: | :green_circle: **ADAPTED** |
| **Tooltip Addons** | :yellow_circle: | — | — | :yellow_circle: | :yellow_circle: | :yellow_circle: **BRUISED** |
| **Raid Tools** | :yellow_circle: | :yellow_circle: | :red_circle: | :green_circle: | :yellow_circle: | :yellow_circle: **FUNCTIONAL** |
| **Housing Addons** | :green_circle: | — | — | :green_circle: | :green_circle: | :green_circle: **THRIVING** |
| **Collection/QoL** | :green_circle: | — | — | :yellow_circle: | :green_circle: | :green_circle: **FINE** |
| **Map/Quest** | :green_circle: | — | — | :green_circle: | :green_circle: | :green_circle: **FINE** |

!!! note "Key"
    :skull: = Existential threat | :red_circle: = Core features lost | :yellow_circle: = Degraded but functional | :green_circle: = Unaffected | `—` = Not applicable

---

## :question: The March 24 Question

!!! warning "Mythic Raids Unlock in 11 Days"
    **March 24, 2026** — For the first time in modern WoW history, the World First race happens without WeakAuras.

### What the Race Guilds Have

| Tool | What It Provides | Limitation |
|------|-----------------|------------|
| **DBM / BigWigs** | Boss timeline reskin, custom audio on 3 alert severity levels, Private Aura integration | Cannot independently identify mechanics or create custom triggers |
| **MRT** | Raid assignments, external buff tracking, planning notes | Communication locked during encounters |
| **Northern Sky** | Timer-based reminders within API constraints | Not as flexible as WeakAura raid packs |
| **Warcraft Logs** | Full post-fight analysis from disk file | Real-time log analysis during pulls is gone |
| **Manual callouts** | Voice comms, assigned callers | Welcome back to 2005 |

### The Three Scenarios

**Scenario A: "It's Fine"**
:green_circle: Boss mods prove sufficient. DBM's 30+ versions of encounter data cover the mechanics. Race completes on expected timeline. Ion declares victory.

**Scenario B: "Controlled Chaos"**
:yellow_circle: First few bosses are clean. Later Mythic bosses expose gaps where custom triggers would have caught edge cases. More wipes, longer race, but guilds adapt through repetition and voice calls.

**Scenario C: "Emergency Hotfix"**
:red_circle: A specific mechanic proves nearly unplayable without addon assistance. Blizzard hotfixes the encounter or emergency-whitelists specific APIs mid-race.

### What Ion Has Said

> *"Our goal has never been to get people who enjoy the customization that addons offer to stop using them entirely."*

> *"It's not that we view a spoken countdown as inherently problematic; rather, we feel that it would be inappropriate to allow only addon users to have that functionality."*
> — [PCGamesN](https://www.pcgamesn.com/world-of-warcraft/ion-hazzikostas-countdown-addons-reply)

> *"We probably should've done something sooner."*
> — [GamesRadar](https://www.gamesradar.com/games/world-of-warcraft/we-probably-shouldve-done-something-sooner-world-of-warcraft-director-says-the-mmos-addon-changes-have-been-a-long-time-coming-but-better-late-than-never/)

### The Community Split

The community is divided on what happens March 24:

- **Optimists**: *"Boss mods are fine. WarcraftLogs isn't affected. Skill gap increases. This is actually better."*
- **Pessimists**: *"Mistakes addons once covered are now fully on the player."* — [dtgre.com](https://www.dtgre.com/2026/03/wow-midnight-2026-combat-changes-addon-restrictions.html)
- **Realists**: DBM met with Ion directly. MysticalOS has encounter timers for Manaforge, Voidspire, Dreamrift, and March on Quel'Danas already shipping. The real question isn't whether guilds can clear — it's how many extra hours it takes.

### The Accessibility Shadow

!!! danger "The unresolved issue"
    Players with disabilities lost critical functionality. A dedicated [Blizzard forum thread](https://us.forums.blizzard.com/en/wow/t/accessibility-concerns-about-midnight/2178274) documented the impact:

    > *"I have disabilities, and literally don't know whether Midnight will be playable beyond LFR for me."* — Hallany

    > *"I am profoundly deaf gamer so text to speech is 100% useless."* — SilentKiller

    Blizzard added Combat Audio Alerts with TTS and improved visual telegraphs, but gaps remain — especially for deaf players and those with motor/anxiety disorders who relied on WeakAuras as assistive technology.

---

## :books: Sources

All claims verified March 13, 2026. Every URL was fetched and confirmed live by automated research agents.

### Official Blizzard

| Source | URL |
|--------|-----|
| Combat Philosophy and Addon Disarmament | [news.blizzard.com](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight) |
| Hotfixes March 11, 2026 | [news.blizzard.com](https://news.blizzard.com/en-us/article/24266320/hotfixes-march-11-2026) |
| 12.0.5 PTR Development Notes | [us.forums.blizzard.com](https://us.forums.blizzard.com/en/wow/t/1205-ptr-development-notes/2270121) |

### API Documentation

| Source | URL |
|--------|-----|
| Patch 12.0.0 API Changes (437 new, 138 removed) | [warcraft.wiki.gg](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) |
| Patch 12.0.0 Planned API Changes | [warcraft.wiki.gg](https://warcraft.wiki.gg/wiki/Patch_12.0.0/Planned_API_changes) |
| Patch 12.0.1 API Changes (59 new, 8 removed) | [warcraft.wiki.gg](https://warcraft.wiki.gg/wiki/Patch_12.0.1/API_changes) |
| Secret Values Documentation | [warcraft.wiki.gg](https://warcraft.wiki.gg/wiki/Secret_Values) |
| COMBAT_LOG_EVENT | [warcraft.wiki.gg](https://warcraft.wiki.gg/wiki/COMBAT_LOG_EVENT) |

### Developer Sources

| Source | URL |
|--------|-----|
| Cell PR #457 (best single technical reference) | [github.com](https://github.com/enderneko/Cell/pull/457) |
| AllTheThings Tooltip Issue #2261 | [github.com](https://github.com/ATTWoWAddon/AllTheThings/issues/2261) |
| WeakAuras Patreon Statement | [patreon.com](https://www.patreon.com/posts/midnight-144610594) |
| Wago.io Security Notice | [accounts.wago.io](https://accounts.wago.io/security-notice) |
| Xepheris Nameplate Deep-Dive | [gerritalex.de](https://gerritalex.de/blog/nameplates-in-midnight) |

### News & Analysis

| Source | URL |
|--------|-----|
| Wowhead: No WeakAuras for Midnight | [wowhead.com](https://www.wowhead.com/news/no-weakauras-addon-for-midnight-378740) |
| Wowhead: Boss Mods in Midnight | [wowhead.com](https://www.wowhead.com/news/boss-mod-addons-in-midnight-teaching-old-mods-new-tricks-380024) |
| Wowhead: DBM/BigWigs Meet with Ion | [wowhead.com](https://www.wowhead.com/news/more-boss-mod-functionality-coming-in-midnight-dbm-and-big-wigs-meet-with-ion-379149) |
| Wowhead: Damage Meter Shortcomings | [wowhead.com](https://www.wowhead.com/news/blizzards-damage-meter-shortcomings-in-midnight-pre-patch-and-addon-alternatives-379992) |
| Wowhead: Addons Still Provide Advantage | [wowhead.com](https://www.wowhead.com/news/addons-continue-to-provide-a-massive-advantage-in-midnight-despite-blizzards-379870) |
| Wowhead: Macro Changes Post-Launch | [wowhead.com](https://www.wowhead.com/news/macro-changes-now-live-to-prevent-workarounds-for-addon-restrictions-380594) |
| Wowhead: Whitelisted Spells Finalized | [wowhead.com](https://www.wowhead.com/news/majority-of-addon-changes-finalized-for-midnight-pre-patch-whitelisted-spells-379738) |
| Wowhead: Nameplates in Midnight | [wowhead.com](https://www.wowhead.com/news/nameplates-in-midnight-whats-changing-and-what-add-ons-can-i-use-379924) |
| Icy Veins: WeakAuras Responds | [icy-veins.com](https://www.icy-veins.com/wow/news/weakauras-responds-to-addon-limitation-loosening-in-midnight/) |
| Icy Veins: Blizzard Relaxing Limitations | [icy-veins.com](https://www.icy-veins.com/wow/news/blizzard-relaxing-more-addon-limitations-in-midnight/) |
| Icy Veins: The Addonpocalypse Working List | [icy-veins.com](https://www.icy-veins.com/wow/news/the-addonpocalypse-is-upon-us-heres-a-list-of-working-addons-for-your-life-raft/) |
| Icy Veins: Cell Stripped-Down for Midnight | [icy-veins.com](https://www.icy-veins.com/wow/news/cell-confirms-a-stripped-down-version-for-midnight/) |
| PC Gamer: Don't Miss Combat Addons | [pcgamer.com](https://www.pcgamer.com/games/world-of-warcraft/after-playing-a-bunch-of-midnight-i-dont-think-i-miss-wows-combat-addons-or-my-old-class-design-at-all/) |
| PC Gamer: Classes Pruned for Midnight | [pcgamer.com](https://www.pcgamer.com/games/world-of-warcraft/wows-classes-were-pruned-for-midnight-because-many-were-built-in-a-world-where-its-devs-assumed-theyd-be-using-addons/) |
| PCGamesN: Ion on Countdown Addons | [pcgamesn.com](https://www.pcgamesn.com/world-of-warcraft/ion-hazzikostas-countdown-addons-reply) |
| GamesRadar: Ion on Addon Changes | [gamesradar.com](https://www.gamesradar.com/games/world-of-warcraft/we-probably-shouldve-done-something-sooner-world-of-warcraft-director-says-the-mmos-addon-changes-have-been-a-long-time-coming-but-better-late-than-never/) |
| Screen Rant: Addon Changes Spark Controversy | [screenrant.com](https://screenrant.com/world-warcraft-midnight-addons-changes-problems/) |
| Kaylriene: Vibecoded Addon Wars | [kaylriene.com](https://kaylriene.com/2026/01/27/a-mini-summary-of-week-one-of-the-new-wow-ui-era-blizzards-own-lua-errors-the-vibecoded-addon-wars-of-2026/) |
| BlizzardWatch: What's Happening With Addons | [blizzardwatch.com](https://blizzardwatch.com/2025/11/03/heck-happening-wow-addons-midnight/) |

### Community

| Source | URL |
|--------|-----|
| Blizzard Forums: Great Addon Purge Updates | [us.forums.blizzard.com](https://us.forums.blizzard.com/en/wow/t/updates-on-what-we-know-so-far-about-the-great-addon-purge-of-2026/2184210) |
| Blizzard Forums: Accessibility Concerns | [us.forums.blizzard.com](https://us.forums.blizzard.com/en/wow/t/accessibility-concerns-about-midnight/2178274) |
| Blizzard Forums: Disabled Players Excluded | [us.forums.blizzard.com](https://us.forums.blizzard.com/en/wow/t/midnight-addon-changes-exclude-disabled-players-like-me/2215814) |
| Blizzard Forums: Shoutout Favorite Addon | [us.forums.blizzard.com](https://us.forums.blizzard.com/en/wow/t/shoutout-your-favorite-midnight-addon/2268368) |
| Blizzard Forums: Healer Addons Status | [us.forums.blizzard.com](https://us.forums.blizzard.com/en/wow/t/healing-addons-their-status/2235343) |
| Blizzard Forums: List of Addons Going Away | [us.forums.blizzard.com](https://us.forums.blizzard.com/en/wow/t/list-of-addons-going-away-in-midnight/2214572) |
| EU Forums: Addon Devs Refused to Continue | [eu.forums.blizzard.com](https://eu.forums.blizzard.com/en/wow/t/add-on-devs-refused-to-continue-in-the-midnight-era-due-to-restrictions-imposed-by-blizz/602690) |
| dtgre.com: Combat Changes & Restrictions | [dtgre.com](https://www.dtgre.com/2026/03/wow-midnight-2026-combat-changes-addon-restrictions.html) |
| WowCoach: Best Raid Tools 2026 | [wowcoach.gg](https://wowcoach.gg/blog/best-raid-tools-wow-midnight-2026) |

---

*This page is part of the [WoW Midnight Addon Development Guide](/). Data sourced from 4 parallel research agents executing 50+ web searches and 60+ page fetches across official Blizzard channels, major gaming outlets, addon platforms, developer blogs, and community forums. March 13, 2026.*
