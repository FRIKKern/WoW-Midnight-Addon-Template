---
title: "Addon & API Threat Bingo"
description: "The ultimate guide to what will break, what will die, and what will thrive in WoW's Midnight addon ecosystem — presented as the most entertaining threat analysis you've ever read."
---

# Addon & API Threat Bingo

> *"We probably should've done something sooner."*
> — **Ion Hazzikostas**, Game Director, January 23, 2026

Welcome to the most dangerous game in World of Warcraft: **predicting what Blizzard will break next.**

Midnight's "Addon Disarmament" was the opening salvo. Secret Values, CLEU removal, 138 deprecated APIs, addon communication restrictions — the landscape has been scorched. But the war isn't over. Blizzard has already announced Patch 12.0.5 will bring the next round of API changes, the whitelist keeps shifting, and addon developers are already finding creative workarounds that paint targets on their own backs.

This page is part bingo card, part threat assessment, part love letter to the most chaotic ecosystem in gaming. Every prediction is grounded in real trends, real API changes, and real developer behavior. Some of these will absolutely happen. Some are long shots. All of them are plausible.

**Grab your dauber. Let's play.**

---

## :dart: The Midnight Addon Bingo Card

!!! abstract "How to play"
    Print this out (or screenshot it). Cross off squares as they happen throughout Midnight's patch cycle. First to get five in a row wins... nothing, except the smug satisfaction of having predicted the addon apocalypse correctly.

| | **B** | **I** | **N** | **G** | **O** |
|---|---|---|---|---|---|
| **1** | WeakAuras loses another feature to whitelist changes | Blizzard adds native cooldown text to nameplates | ElvUI taint breaks mid-mythic raid prog | Someone vibecodes a raid addon that wipes a guild | A housing addon hits 10M downloads |
| **2** | Details! becomes literally just a skin with zero unique data | DBM dev meets Ion again for emergency summit | Hard-coded timer addon gets restricted in instances | Plater user discovers mob classification is now secret | Northern Sky Raid Tools passes 10M downloads |
| **3** | New "BetterBoss" addon enhances native timeline | Blizzard's own UI throws a Secret Values LUA error | **:star: FREE SPACE :star:** (Addon breaks on patch day) | WeakAuras team posts another "still no" update | Wago.io gets a second security incident |
| **4** | Rotation helper resurfaces using audio-only cues | `C_Timer` abuse gets rate-limited in combat | Cell PR becomes required reading in CS courses | Healer main quits over Secret Values accessibility | Blizzard whitelists a spell that re-enables a dead addon |
| **5** | An AI-generated addon hits #1 on CurseForge | Macro restrictions expand to cover weakaura-like behavior | MRT becomes the last addon standing from pre-Midnight | Patch 12.0.5 breaks 50+ addons simultaneously | Housing addon developer gets hired by Blizzard |

---

### Bingo Card Explanations

Every cell on that card has a story behind it. Here's why each one earned its spot.

??? info "B1: WeakAuras loses another feature to whitelist changes"
    WeakAuras is already dead for combat in Midnight — the team explicitly refused to ship a version. But non-combat auras still function outside instances, and some edge-case triggers remain. Every time Blizzard adjusts the Secret Values whitelist, it shifts what's possible. A whitelist *removal* (which has happened — string functions were briefly removed before being restored) could eliminate even more non-combat functionality. The WeakAuras team has stated they evaluate each change, and their Patreon confirms the ~$500/month income isn't the deciding factor — it's purely technical. Each whitelist tweak is another potential nail in the coffin.

??? info "B2: Details! becomes literally just a skin with zero unique data"
    Details! already lost overhealing tracking, CC break detection, buff uptime analysis, resource generation tracking, pet damage segmentation, spell merging, advanced death breakdowns, and chat reporting. It's functionally a visual skin over Blizzard's `C_DamageMeter` namespace. If Blizzard further restricts what the native meter exposes — or if `C_DamageMeter.GetCombatSessionSourceFromID` starts returning secret values — Details! could lose even the limited breakdown data it still displays. With 330M+ downloads on CurseForge, this would be the most-downloaded addon to become a pure cosmetic wrapper.

??? info "B3: New 'BetterBoss' addon enhances native timeline"
    The "Better___" pattern (BetterBlizzFrames at 3.9M downloads, BetterBlizzPlates, BetterBags at 3.6M, BetterCooldownManager) is the dominant Midnight design philosophy. Better Timeline already exists as a standalone addon born from the RaidAbilityTimeline WeakAura. A "BetterBoss" or "BetterTimeline" addon that enhances Blizzard's native Boss Timeline HUD with custom layouts, colors, and filtering — without touching combat data — is practically inevitable. The `C_EncounterTimeline` namespace has 25+ APIs just waiting to be skinned.

??? info "B4: Rotation helper resurfaces using audio-only cues"
    Hekili is dead. Rotation helpers are "exactly what Blizzard targeted." But Blizzard's own Combat Audio Alerts system has three severity pools (minor, medium, critical) with customizable TTS. What if someone builds a "rotation guide" that uses only audio patterns — click patterns, rhythm-based sound cues — that technically doesn't access any combat data but teaches muscle memory through sound? It's the kind of creative boundary-pushing that WoW addon developers have done for twenty years. The Private Aura audio integration that DBM discovered proves custom audio hooks exist.

??? info "B5: An AI-generated addon hits #1 on CurseForge"
    NephUI was openly "vibe coded" (AI-generated Lua) and gained real traction before plagiarism accusations shut it down in January 2026. The code reportedly worked but was "brittle." AI coding tools are improving rapidly. With Midnight's simpler API surface (visual customization only, no complex combat logic), the bar for AI-generated addons is actually *lower* than before. A housing addon, a "BetterSomething" enhancer, or a UI suite generated primarily by AI that hits the top of CurseForge is a matter of when, not if.

??? info "I1: Blizzard adds native cooldown text to nameplates"
    Blizzard has been systematically absorbing addon functionality into the default UI: damage meters, boss timelines, cooldown managers, combat audio alerts, quest tracking. Cooldown text on nameplates is one of the most-requested features and was previously handled by addons like Plater (which lost this ability under Secret Values since `UnitName` returns secrets in instances). The 32 new CVars added in 12.0.1 include nameplate scaling options, suggesting Blizzard is actively iterating on the nameplate system. Native cooldown text is a logical next step.

??? info "I2: DBM dev meets Ion again for emergency summit"
    It already happened once. MysticalOS and BigWigs developer Funkeh met directly with Ion Hazzikostas and lead software engineer Andy Churchill, resulting in spell cast counting, custom audio pack support, and Private Aura integration. If Patch 12.0.5 introduces changes that break the current boss mod workarounds — particularly the hard-coded timer systems or Private Aura audio hooks — another emergency summit is likely. Boss mods are the one addon category Blizzard explicitly negotiated with rather than simply restricting.

??? info "I3: Blizzard's own UI throws a Secret Values LUA error"
    **This already happened.** In the first week of the pre-patch, mousing over vendor prices during combat triggered errors because gold values were incorrectly flagged as secret. The community-made MoneyFrameFix addon was created to patch *Blizzard's own bug*. Nameplate developer Xepheris noted the Cooldown Manager was "barely functional" at launch. The Secret Values system is complex enough that Blizzard's own code will continue to have edge cases where values are incorrectly classified. This square is practically a free space.

??? info "I4: `C_Timer` abuse gets rate-limited in combat"
    `C_Timer.After()` is currently the backbone of every "hard-coded timer" addon. Northern Sky Raid Tools, Viserio Cooldowns, and numerous reminder addons use pre-authored `C_Timer` chains to recreate boss ability timelines without parsing combat events. If these become too accurate — essentially reconstructing boss mod functionality through timing alone — Blizzard could rate-limit `C_Timer` calls during encounters. The March 2026 hotfix that restricted macros from setting more than 3 target markers "within a short time" shows Blizzard is willing to add rate limits to prevent abuse.

??? info "I5: Macro restrictions expand to cover WeakAura-like behavior"
    March 2026 hotfixes already restricted macros from sending chat messages during encounters and from setting target markers on more than 3 units within a short time. Complex macros that use conditional sequences to approximate addon behavior — especially those using `/castsequence`, `/stopmacro`, and conditional evaluation — are an obvious next target. If players start sharing "macro WeakAuras" that chain conditionals to approximate combat triggers, Blizzard will almost certainly clamp down.

??? info "N1: ElvUI taint breaks mid-mythic raid prog"
    ElvUI v15.0.0 shipped after a dramatic comeback — initially cancelled in October 2025, returned in December 2025. The oUF framework required "completely disabling core functionality such as nameplates, tags, castbars and auras" just to stop throwing errors. Style Filters and Portraits were removed *across all game versions*. With this much architectural surgery, taint issues during combat are almost inevitable. ElvUI serves millions of players, and a taint-triggered protected action failure during a Mythic raid progression attempt would be felt across the entire competitive raiding scene. The team themselves described it as "not perfectly fine but a very good start."

??? info "N3: FREE SPACE — Addon breaks on patch day"
    Every single WoW patch in history has broken addons. Patch 12.0.0 alone removed 138 APIs and added 437 new ones. The Midnight launch on March 2, 2026 saw "a solid chunk of the addon ecosystem quietly collapse into rubble" within 48 hours. Blizzard has confirmed Patch 12.0.5 will contain the next round of API changes. This isn't a prediction — it's a mathematical certainty. Your addons *will* break on patch day. The only question is how many and how badly.

??? info "N4: WeakAuras team posts another 'still no' update"
    The WeakAuras team has been remarkably consistent. Their Patreon statement (November 2025): "We consider tracking your own combat state the core functionality of WeakAuras." Their response to loosened restrictions (November 29, 2025 via PCGamesN): stance "remains unchanged." They've listed three conditions for returning — secret value computation, personal-only data access, or complete reversal — and none are on Blizzard's roadmap. Every time Blizzard adjusts something, the community asks "does this bring WeakAuras back?" and every time the answer is no. Expect at least one more "still no" post per major patch.

??? info "N5: MRT becomes the last addon standing from pre-Midnight"
    Method Raid Tools v5260 is shipping for 12.0.1 and is described as "the backbone of organized raiding in 2026." Its core features — raid cooldown tracking, external buff assignments, raid planning notes, ready checks — don't rely on combat parsing. While other legacy addons were gutted or killed, MRT's note-sharing and assignment functionality is inherently compatible with Secret Values because it's *prescriptive* (telling players what to do) rather than *reactive* (responding to combat events). If Blizzard continues restricting combat-reactive addons, MRT's planning-focused design makes it the ultimate survivor.

??? info "G1: Someone vibecodes a raid addon that wipes a guild"
    The Vibecoded Addon Wars of January 2026 proved that AI-generated addon code gains real adoption. NephUI's code "worked but was reportedly brittle." Now imagine an AI-generated raid tool — a timer addon, a cooldown tracker, a position indicator — that has a subtle bug. Maybe it shows the wrong timer for a boss ability, causing the raid to stack when they should spread. Maybe it fires a warning sound at the wrong time, causing a tank to use a cooldown prematurely. With over 100,000 players affected by the QuaziiUI/NephUI shutdowns alone, vibecoded addons are reaching real raid groups. A guild wipe caused by brittle AI-generated code is disturbingly plausible.

??? info "G2: Plater user discovers mob classification is now secret"
    Mob type/classification coloring is described as **"the only real OP feature kept"** in Plater after Midnight. It's how players identify priority interrupt targets in M+ dungeons. Blizzard has already restricted `UnitName` to return secrets in instances — what if `UnitClassification()` or creature type data follows? The "Addons Still Win" Wowhead investigation specifically called out "color-coded nameplates use general criteria (mob classification) to identify priority targets" as a remaining competitive advantage. If Blizzard reads that article and agrees, this data could move behind the Secret Values curtain.

??? info "G4: Healer main quits over Secret Values accessibility"
    A Blizzard forum thread highlighted players with impaired vision and limited hand mobility who relied on WeakAuras for custom combat cues. Blizzard responded with Combat Audio Alerts with TTS, but the community consensus is "it doesn't fully replace customizable WeakAuras." Healers were hit particularly hard — they need real-time data about who has what debuff, who needs dispelling, who's about to die. The new aura filter categories (`CROWD_CONTROL`, `BIG_DEFENSIVE`, `RAID_PLAYER_DISPELLABLE`, `RAID_IN_COMBAT`) added in 12.0.1 help, but if a high-profile healer streamer or content creator quits citing Secret Values' impact on accessibility and healing gameplay, it would reignite the entire debate.

??? info "G5: Blizzard whitelists a spell that re-enables a dead addon"
    The whitelist has been growing since Alpha: Skyriding abilities, Maelstrom Weapon (Shaman), Soul Fragments (Demon Hunter), Combat Resurrection spells, and string functions that were "mistakenly removed then restored." Each whitelist addition slightly expands what addons can track. If Blizzard whitelists enough personal combat resources — combo points for Rogues, Holy Power for Paladins, Insanity for Priests — an addon like WeakAuras could theoretically track *some* personal state again. One strategic whitelist addition could bring a dead addon category back from the grave.

??? info "O1: A housing addon hits 10M downloads"
    HomeBound is already at 3M+ downloads and was updated February 27, 2026. HomeDecor offers vendor tracking, Endeavor dashboards, and AH integration. The `C_Housing` and `C_HousingDecor` APIs are extensive and *completely unrestricted* — no Secret Values, no combat restrictions, a total greenfield. Player Housing launched December 2, 2025 (early access) and is the single largest content addition in Midnight. With Housing being the one area where addon developers can go absolutely wild with zero restrictions, a housing addon hitting 10M downloads is not just possible — it's the most likely outcome on this entire card.

??? info "O2: Northern Sky Raid Tools passes 10M downloads"
    Already at 3.2M+ downloads, Northern Sky Raid Tools by developer Reloe has become the de facto WeakAuras replacement for raid teams. It uses timer-based reminders within API constraints — exactly the kind of addon Blizzard seems to tolerate (for now). As more raid tiers release and more teams discover they can't use WeakAura raid packs anymore, NSRT's growth could accelerate dramatically. The addon also spawned MPlusTimer as a standalone conversion, showing Reloe's pattern of capturing former WeakAura users.

??? info "O3: Wago.io gets a second security incident"
    Wago.io suffered a confirmed data breach on February 24, 2026 — unauthorized remote access to a web server, with usernames and email addresses potentially compromised. The platform hosts millions of addon profiles, WeakAura imports, Plater profiles, and UI packs. It's a high-value target. The Blizzard forums PSA warned of increased phishing risk. A second incident — whether a breach, a malicious addon upload, or a supply chain attack through compromised Plater/WeakAura imports — would be devastating to the addon distribution ecosystem.

??? info "O4: Patch 12.0.5 breaks 50+ addons simultaneously"
    Patch 12.0.0 removed 138 APIs and added 437 new ones. Patch 12.0.1 added 59 more. Blizzard has confirmed the API is frozen until 12.0.5, and the next round will contain API changes. If 12.0.5 is even half as aggressive as 12.0.0, dozens of addons that stabilized during the API freeze will break again. The freeze itself is almost cruel — it gives developers a stable target, lets them build confidence, and then pulls the rug. Addon developers who got comfortable during the freeze will be caught off guard.

??? info "O5: Housing addon developer gets hired by Blizzard"
    It wouldn't be the first time. Blizzard has a long history of hiring addon developers — the original Deadly Boss Mods contributors, the dungeon journal developer, and others have all been recruited. The `C_Housing` API space is massive and unrestricted, which means housing addon developers are building deep expertise in a system Blizzard is actively developing. A developer who creates an exceptionally polished housing tool — especially one that solves UX problems Blizzard's native UI doesn't address — would be a natural hire for the housing team.

---

## :trophy: Top 10 Most Threatened Addons

The Great Addon Purge of 2026 already killed several addons outright. But the survivors aren't safe. Here's who's most at risk of further damage, obsolescence, or death in the coming patches.

### Threat Tier: :skull: DEAD — No Path to Recovery

---

#### 1. WeakAuras (Combat)

| | |
|---|---|
| **What it does** | Conditional visual/audio triggers based on combat state — the "Swiss Army knife" of WoW |
| **Threat level** | :skull: **DEAD** |
| **Why it's dead** | Secret Values destroy the core architecture. Cannot glow icons on cooldown ready, change colors on low health, trigger audio from buff/debuff state, or combine data sources into compound triggers |
| **What would save it** | Ability to compute new secret values from existing ones (e.g., `secretA + secretB = secretC`), or reverting restrictions for personal combat state only |
| **Current status** | Team explicitly refuses to develop Midnight version. Last statement (Nov 29, 2025): "stance remains unchanged." Non-combat auras *may* still function outside instances |

!!! quote "WeakAuras team ([Patreon](https://www.patreon.com/posts/midnight-144610594))"
    "We consider tracking your own combat state the core functionality of WeakAuras... a WeakAuras version that only consists of e.g. reputation and experience triggers is nearly useless."

The elephant in the room. WeakAuras isn't just an addon — it was a **platform**. Entire raid strategies were distributed as import strings. UI customization guides assumed you had it. Class guides linked WA packs for every spec. Its loss is the single largest hole in the Midnight addon ecosystem, and nothing has fully replaced it.

---

#### 2. Hekili (Rotation Helper)

| | |
|---|---|
| **What it does** | Real-time rotation suggestions — shows which ability to press next based on SimulationCraft priority lists |
| **Threat level** | :skull: **DEAD** |
| **Why it's dead** | Rotation helpers are "exactly what Blizzard targeted." Cannot access cooldown states, resource levels, or buff/debuff conditions under Secret Values |
| **What would save it** | Complete reversal of Secret Values (extremely unlikely) |
| **Current status** | Fundamentally impossible. No workaround exists within the Midnight framework |

Rotation helpers represent the purest form of what Blizzard calls "competitive advantage through addons." They literally tell you what button to press. Blizzard explicitly cited them when explaining Addon Disarmament, and the class pruning (Roll the Bones simplified, tank defensives reduced) was designed to make the game playable *without* them.

---

#### 3. GTFO

| | |
|---|---|
| **What it does** | Plays warning sounds when standing in damaging ground effects |
| **Threat level** | :skull: **DEAD** |
| **Why it's dead** | Cannot detect ground effects under Secret Values. Required CLEU parsing for damage source detection |
| **What would save it** | Blizzard whitelisting environmental damage events, or expanding native Combat Audio Alerts to cover ground effects more comprehensively |
| **Current status** | Non-functional post-Midnight. Blizzard's native Combat Audio Alerts with TTS partially cover this use case |

GTFO was the addon you recommended to new raiders who kept dying to fire. Its loss directly impacts learning players and accessibility. Blizzard's Combat Audio Alerts have three severity pools (minor, medium, critical) but don't provide the same granular "you're standing in the bad" feedback.

---

### Threat Tier: :red_circle: CRITICAL — Surviving but Severely Diminished

---

#### 4. Plater Nameplates

| | |
|---|---|
| **What it does** | Comprehensive nameplate customization with scripting, profiles, and NPC-specific rules |
| **Threat level** | :red_circle: **CRITICAL** |
| **Why it's threatened** | Lost NPC-specific coloring, aura-based scaling, cast bar interrupt tracking, fixate detection, time-to-death, execute range indicators, health threshold markers, quest progress, buff/debuff repositioning, custom fonts in instances |
| **What would save it** | Expanded whitelist for unit data in instances, or `UnitName` being un-secreted |
| **Current status** | Shipping but described by community as "demoted." Mob type/classification coloring is "the only real OP feature kept." Platynator emerging as Midnight-native alternative |

Plater lost more functionality than any other surviving addon. The community quote says it all: *"The addon apocalypse didn't kill Plater; it just demoted it."* Meanwhile, Platynator — built from scratch for Midnight's constraints — is rising as a native alternative that doesn't fight the restrictions. Plater's future depends on whether Blizzard loosens nameplate data or whether Platynator's "design within the box" philosophy wins.

---

#### 5. Details! Damage Meter

| | |
|---|---|
| **What it does** | Damage/healing meter with spell breakdowns, death logs, and fight analysis |
| **Threat level** | :red_circle: **CRITICAL** |
| **Why it's threatened** | Now functionally a skin over Blizzard's `C_DamageMeter` data. Lost overhealing, CC breaks, buff uptime, resource tracking, pet segmentation, spell merging, death breakdowns, chat reporting. Doesn't persist across logouts |
| **What would save it** | Blizzard expanding `C_DamageMeter` API with more data fields, or allowing limited post-combat log access |
| **Current status** | 330M+ downloads. Shipping but community calls it "just a skin." Players increasingly rely on Warcraft Logs for analysis |

Details! went from being the most sophisticated in-game analytical tool to a pretty wrapper around Blizzard's data. The irony: it's still popular because the familiar UI is comfort food, even if the nutritional content has been gutted. If Blizzard improves their native meter's visual presentation, even the "pretty skin" value proposition erodes.

---

#### 6. ElvUI

| | |
|---|---|
| **What it does** | Complete UI replacement — action bars, unit frames, nameplates, chat, bags, minimap, everything |
| **Threat level** | :red_circle: **CRITICAL** |
| **Why it's threatened** | oUF framework required disabling core functionality. Style Filters and Portraits removed across ALL game versions. Supports 5 simultaneous game versions (Classic, TBC, Mists, Retail, Midnight). Taint risk from architectural surgery on secure frames |
| **What would save it** | Blizzard providing a stable, documented skinning API for secure frames. Or Edit Mode becoming flexible enough that ElvUI's value proposition shrinks to just visual skinning |
| **Current status** | v15.0.0 shipping. Dramatic comeback after October 2025 cancellation. Described as "not perfectly fine but a very good start." Developer Luckyone returned after initially quitting |

ElvUI's dramatic arc — halt development, community panic, Vibecoded Wars, dramatic return — is the most theatrical story in Midnight addon history. But the reduced-feature v15.0.0 raises existential questions: if ElvUI can't do Style Filters, Portraits, or Cutaway Bars, is it still ElvUI? Or is it slowly becoming another "Better___" enhancer wearing a trenchcoat?

---

### Threat Tier: :yellow_circle: ENDANGERED — Adapting but Vulnerable

---

#### 7. DBM (Deadly Boss Mods)

| | |
|---|---|
| **What it does** | Boss encounter warnings, timers, and alerts |
| **Threat level** | :yellow_circle: **ENDANGERED** |
| **Why it's threatened** | Cannot independently detect boss mechanics. Functions as enhancement layer over Blizzard's native Boss Timeline. Hard-coded timers could be restricted if they become too accurate |
| **What would save it** | Continued Blizzard cooperation (MysticalOS has a direct line to Ion). Expanded `C_EncounterEvents` and `C_EncounterTimeline` APIs |
| **Current status** | v12.0.12+ shipping. Custom timers, Private Aura audio, timeline reskinning. Developer met with Ion and Andy Churchill. Still the go-to boss mod |

DBM survived because its developer got a seat at the table — literally meeting with the Game Director. But the current implementation is fundamentally different from pre-Midnight DBM. It's less "boss mod" and more "boss timeline enhancer with pre-authored timers." If Blizzard decides the timer system reconstructs too much boss mod functionality, DBM's workarounds are at risk. The March 2026 hotfix restricting macro behavior during encounters shows Blizzard is actively monitoring in-combat addon-like behavior.

---

#### 8. BigWigs / LittleWigs

| | |
|---|---|
| **What it does** | Boss encounter warnings (BigWigs for raids, LittleWigs for dungeons) |
| **Threat level** | :yellow_circle: **ENDANGERED** |
| **Why it's threatened** | Same limitations as DBM — no independent mechanic detection, limited to enhancement layer |
| **What would save it** | Same as DBM — Blizzard cooperation and API expansion |
| **Current status** | Shipping with similar adaptations to DBM. "Much more limited in scope" than pre-Midnight |

BigWigs faces the same existential questions as DBM but without the documented direct relationship with Blizzard's leadership. Its dungeon counterpart LittleWigs is particularly vulnerable because M+ dungeons change seasonally, requiring constant timer updates that could become a maintenance burden without combat log access.

---

#### 9. Cell Raid Frames

| | |
|---|---|
| **What it does** | Raid frame replacement for healers with debuff tracking, click-casting, and layout management |
| **Threat level** | :yellow_circle: **ENDANGERED** |
| **Why it's threatened** | Required the most technically thorough adaptation of any addon (PR #457 is a masterclass). AoE healing module completely disabled. CLEU replacement strategies vary by module. Communication restrictions during encounters. Heal prediction bugs from shared calculators |
| **What would save it** | Expanded aura filter categories, whitelist expansion for healing spells, improved native healer raid frames |
| **Current status** | r274/r275 shipping. Initial debuffs for all 12 Midnight instances (6 raids, 6 dungeons, 41 bosses). Enhanced Raid Frames (separate addon) ceased functioning |

Cell's adaptation is the most technically impressive in the entire Midnight transition. [PR #457](https://github.com/enderneko/Cell/pull/457) documents granular per-aura secret detection, CLEU replacement strategies, communication queuing, and a subtle heal prediction fix involving shared calculators. But "most impressive" also means "most complex" — and complexity breeds fragility. Each new raid tier requires manual debuff cataloguing that used to be automatic.

---

### Threat Tier: :green_circle: ADAPTING — Positioned to Survive

---

#### 10. OPie (Radial Action Menu)

| | |
|---|---|
| **What it does** | Radial menu for spells, items, and macros — hold a key, point at the action, release |
| **Threat level** | :green_circle: **ADAPTING** |
| **Why it's threatened** | Uses secure action buttons which are subject to increasing taint restrictions. If Blizzard locks down `SecureActionButtonTemplate` interactions, radial menus could be affected |
| **What would save it** | OPie's design is inherently input-driven (player physically aims at an action), which aligns with Blizzard's philosophy that addons should present information but players make decisions |
| **Current status** | Updated for Midnight Pre-Patch (February 5, 2026). Functional and stable |

OPie is the sleeper on this list. It doesn't process combat data — it just presents actions in a radial layout. But its reliance on `SecureActionButtonTemplate` means it interacts with the protected frame system. Any expansion of taint restrictions on secure frames could create issues. For now, though, OPie's design philosophy — "present options, player decides" — is exactly what Blizzard says addons should do.

---

### Honorable Mentions

| Addon | Threat Level | Note |
|---|---|---|
| **Z-Perl Unit Frames** | :skull: DEAD | Legacy frames broken, no active development |
| **Shadowed Unit Frames** | :skull: DEAD | Legacy frames broken, no active development |
| **Enhanced Raid Frames** | :skull: DEAD | Ceased functioning due to Midnight changes |
| **Bartender4** | :yellow_circle: ENDANGERED | Action bar addon relying on `SecureActionButtonTemplate` — functional but any secure frame lockdown is a risk |
| **VuhDo** | :yellow_circle: ENDANGERED | Healer click-cast frames — similar challenges to Cell |
| **Skada** | :red_circle: CRITICAL | Same data limitations as Details! but with smaller development team |
| **TellMeWhen** | :skull: DEAD | Conditional display addon — same architectural problem as WeakAuras |

---

## :zap: Top 10 API "Exploits" That Power Cutting-Edge Addons

These aren't exploits in the security sense — they're **creative uses of legitimate APIs** that push beyond what Blizzard probably intended. They're the techniques that make Midnight's best addons possible. They're also the techniques most likely to be restricted next.

!!! warning "Why this list matters"
    Every technique Blizzard has ever restricted followed the same pattern: addon developers found a creative use, it became popular, Blizzard decided it gave too much advantage, and the API was locked down. These ten techniques are the ones currently powering the most impressive addons — and the ones most likely to be on Blizzard's radar for the next round of changes in Patch 12.0.5.

---

### 1. :one: Hard-Coded Timer Chains via `C_Timer.After()`

| | |
|---|---|
| **Risk Rating** | :red_circle: **ENDANGERED** |
| **API/Function** | `C_Timer.After(seconds, callback)`, `C_Timer.NewTimer()`, `C_Timer.NewTicker()` |
| **Who uses it** | Northern Sky Raid Tools (3.2M+ DL), Viserio Cooldowns, custom reminder addons |
| **What it achieves** | Pre-authored boss ability timelines without parsing combat events. Timers fire at known intervals matching boss ability cadences, effectively reconstructing boss mod functionality |
| **Why Blizzard might close it** | Wowhead's investigation confirmed these addons "alert on defensives/offensives unavailable in default UI" and "create independent timer systems with dynamic phase-tracking." This is exactly the competitive advantage Blizzard said they wanted to eliminate |
| **Devastation if closed** | :boom: **MASSIVE** — Would kill the entire "reminder addon" category and leave boss mods with only Blizzard's native timeline |

The most powerful workaround in the Midnight era. Addon developers realized that if you can't *detect* boss abilities, you can *predict* them. Boss encounters follow scripted timelines — ability X fires at 0:15, 0:45, 1:15, etc. Pre-authoring these timelines as `C_Timer` chains recreates boss mod functionality without touching any restricted API.

```lua
-- Simplified example of how timer-based boss tracking works
local function StartBossTimers()
    C_Timer.After(15, function()
        PlaySound(SOUNDKIT.RAID_WARNING)
        -- "Boss is about to cast Shockwave"
    end)
    C_Timer.After(45, function()
        PlaySound(SOUNDKIT.RAID_WARNING)
        -- "Shockwave incoming again"
    end)
end
```

The problem? These addons are getting *really good*. Northern Sky Raid Tools doesn't just fire static timers — it detects phase transitions (using Blizzard's own timeline data) and adjusts timer chains dynamically. That's starting to look less like "pre-authored reminders" and more like "automated boss mod."

---

### 2. :two: Private Aura Audio Injection

| | |
|---|---|
| **Risk Rating** | :yellow_circle: **WATCH** |
| **API/Function** | Private Aura system + `C_CombatAudioAlert` severity pools |
| **Who uses it** | DBM (v12.0.12+), BigWigs, custom boss audio packs |
| **What it achieves** | Custom audio alerts attached to encounter debuffs without the addon "knowing" what debuff is active. Boss mods can play sounds when you get targeted without accessing the targeting data |
| **Why Blizzard might close it** | It's a creative workaround that restores part of what Blizzard removed. If custom audio becomes sophisticated enough (different sounds for different mechanics), it reconstructs boss mod awareness through the audio channel |
| **Devastation if closed** | :boom: **HIGH** — Would gut DBM and BigWigs' remaining useful functionality. Boss mod developers explicitly negotiated this capability with Ion |

This is the technique born from the Ion meeting. Private Auras — Blizzard's system for encounter debuffs — accept custom audio alerts. The addon doesn't know *which* debuff you have, but it can attach a sound to "any private aura in the critical severity pool." It's brilliant because it works within the spirit of Secret Values: the addon doesn't *know* what's happening, it just makes a sound when *something* happens.

The risk? If addon developers start mapping specific sounds to specific debuffs by trial and error (play a unique sound for each encounter, identify which sound maps to which mechanic through testing), they're effectively reverse-engineering the Private Aura system. Blizzard might respond by randomizing audio pool assignments.

---

### 3. :three: `hooksecurefunc()` on Secure Frame Render Methods

| | |
|---|---|
| **Risk Rating** | :yellow_circle: **WATCH** |
| **API/Function** | `hooksecurefunc(frame, "method", callback)` on Blizzard UI frames |
| **Who uses it** | BetterBlizzFrames (3.9M+ DL), ElvUI, every "Better___" addon, Masque, AddOnSkins |
| **What it achieves** | Run custom code after Blizzard's UI renders — change colors, add overlays, modify textures, inject custom elements. The backbone of the "enhance, don't replace" philosophy |
| **Why Blizzard might close it** | Patch 11.0.0 already restricted hooks on "functions with certain protected names." If Blizzard expands this restriction to cover more render methods, the entire "Better___" ecosystem breaks |
| **Devastation if closed** | :boom: :boom: **CATASTROPHIC** — Would destroy the entire "enhance, don't replace" philosophy that Blizzard themselves encouraged |

`hooksecurefunc` is the single most important function in Midnight addon development. It lets addons run code *after* Blizzard's own code executes, without modifying the original behavior or causing taint. BetterBlizzFrames hooks into `CompactUnitFrame_UpdateRoleIcon`, `CompactUnitFrame_UpdateHealPrediction`, `UnitFrameHealPredictionBars_Update`, and dozens more to apply visual modifications after Blizzard's code runs.

The crucial property: hooks cannot prevent execution, cannot modify return values, and cannot be unhooked. This makes them safe — but also makes them sticky. Every hook persists until `/reload`, and they stack if multiple addons hook the same function. Heavy hook stacking on frequently-called functions (like nameplate update methods) can cause performance issues.

```lua
-- The pattern that powers the "Better___" ecosystem
hooksecurefunc("CompactUnitFrame_UpdateRoleIcon", function(frame)
    -- Blizzard's code already ran. Now we add our modifications.
    if frame.roleIcon then
        frame.roleIcon:SetAlpha(0.8)  -- Subtle but visible
    end
end)
```

If Blizzard expands the list of protected function names that can't be hooked, it would undermine the very design philosophy they encouraged addon developers to adopt.

---

### 4. :four: Mob Classification / Creature Type Branching

| | |
|---|---|
| **Risk Rating** | :red_circle: **ENDANGERED** |
| **API/Function** | `UnitClassification("unit")`, `UnitCreatureType("unit")`, `UnitCreatureFamily("unit")` |
| **Who uses it** | Plater, Platynator, BetterBlizzPlates, nameplate addons generally |
| **What it achieves** | Color-coding nameplates by mob type (elite, rare, boss) to identify priority interrupt targets and dangerous enemies in M+ |
| **Why Blizzard might close it** | Wowhead's "Addons Still Win" investigation explicitly called out nameplate coloring by mob classification as a remaining competitive advantage. `UnitName` already returns secrets in instances — classification data is a logical next restriction |
| **Devastation if closed** | :boom: **HIGH** — Would remove the last meaningful data-driven feature from nameplate addons in instances |

This is the last bastion. After losing NPC-specific coloring, aura-based scaling, cast bar tracking, fixate detection, and everything else, mob classification coloring is what makes Plater still worth running. In a Mythic+ dungeon, seeing a yellow nameplate (elite) vs. a standard nameplate (normal) instantly tells you what to prioritize.

Blizzard's philosophy says addons shouldn't provide competitive advantage. Color-coding enemies by type is definitionally a competitive advantage — it helps you make faster decisions about what to interrupt, CC, or focus. The question isn't whether Blizzard wants to restrict it. The question is whether they'll do it in 12.0.5 or 12.1.

---

### 5. :five: Tooltip Scanning for Information Extraction

| | |
|---|---|
| **Risk Rating** | :yellow_circle: **WATCH** |
| **API/Function** | `GameTooltip:SetAction()`, `GameTooltip:SetUnitBuff()`, `GameTooltip:SetSpellByID()`, various tooltip setter methods |
| **Who uses it** | idTip, Details!, auction addons, TSM (TradeSkillMaster), item comparison addons |
| **What it achieves** | Extract information from the tooltip text that isn't available through direct API calls. Tooltip text often contains data that the API doesn't expose directly |
| **Why Blizzard might close it** | Tooltip text in combat context could expose Secret Values information through the text rendering pipeline. If addons parse tooltip strings to determine combat state, it's a Secret Values bypass |
| **Devastation if closed** | :boom: **MODERATE** — Would break information addons but most combat-critical tooltip data is already secreted |

Tooltip scanning has been a core addon technique since WoW's launch. The trick: call a tooltip setter method, read the resulting text, parse it for data. This bypasses the need for dedicated APIs — if the information appears in a tooltip anywhere, you can extract it programmatically.

```lua
-- Classic tooltip scanning pattern
hooksecurefunc(GameTooltip, "SetUnitBuff", function(self, ...)
    local id = select(10, UnitBuff(...))
    if id then
        self:AddLine("Spell ID: " .. id, 1, 1, 1)
        self:Show()
    end
end)
```

In Midnight, much of the combat-relevant tooltip data is already protected. But non-combat tooltips — items, achievements, profession recipes — still expose rich data. If Blizzard decides to extend Secret Values to cover more tooltip contexts (e.g., item stats during combat), it could break a wide range of information addons.

---

### 6. :six: Frame Strata and Level Manipulation

| | |
|---|---|
| **Risk Rating** | :green_circle: **SAFE** (for now) |
| **API/Function** | `frame:SetFrameStrata()`, `frame:SetFrameLevel()`, `frame:SetParent()` |
| **Who uses it** | ElvUI, every UI overhaul, custom frame addons |
| **What it achieves** | Control rendering order to ensure addon frames appear above or below Blizzard's UI. Reparenting frames to hide them without calling `:Hide()` (which would taint secure frames) |
| **Why Blizzard might close it** | The hidden-frame reparenting trick (`frame:SetParent(sneakyFrame)`) is how addons like BetterBags hide Blizzard's native bags without causing taint. It's clever but it's also a taint avoidance technique |
| **Devastation if closed** | :boom: **HIGH** — Would break every addon that hides native Blizzard frames via reparenting |

The sneaky frame reparenting pattern is everywhere:

```lua
-- BetterBags' approach to hiding Blizzard's bags
local sneakyFrame = CreateFrame("Frame", "BetterBagsSneakyFrame")
sneakyFrame:Hide()
ContainerFrameCombinedBags:SetParent(sneakyFrame)  -- Gone, not tainted
```

This works because `SetParent` to a hidden frame effectively hides the child without calling `:Hide()` directly. It's the foundation of every "full replacement" addon (bags, chat, unit frames). Blizzard has tolerated it for years, but if they want to prevent addons from hiding native UI elements entirely, restricting `SetParent` on protected frames would do it.

---

### 7. :seven: Metatable and Raw Function Replacement on Non-Secure Objects

| | |
|---|---|
| **Risk Rating** | :yellow_circle: **WATCH** |
| **API/Function** | `setmetatable()`, `rawset()`, `rawget()`, direct function replacement on addon-created objects |
| **Who uses it** | Advanced addons, library frameworks, mixin-heavy architectures |
| **What it achieves** | Intercept method calls on non-secure objects, implement lazy loading, create proxy objects that transform data before display |
| **Why Blizzard might close it** | `setfenv()` calling on hooked functions already throws errors. If Blizzard extends restrictions to metatable manipulation on any frame that touches combat data, complex addon architectures break |
| **Devastation if closed** | :boom: **MODERATE** — Would force simpler addon architectures but wouldn't kill the ecosystem |

Metatable manipulation is the advanced technique that separates sophisticated addons from simple ones. By setting custom metatables on objects, addons can intercept property access, implement inheritance, and create proxy layers:

```lua
-- Mixin override: modify before application
local originalOnLoad = SomeBlizzardMixin.OnLoad
SomeBlizzardMixin.OnLoad = function(self, ...)
    originalOnLoad(self, ...)
    self.myCustomData = {}  -- Inject custom data
end
```

The key vulnerability: mixin methods are *copied* onto frame instances, not referenced. Hooking `SomeBlizzardMixin.SomeMethod` does nothing — you must hook the concrete frame instance. This subtlety catches many addon developers and is one reason mixin-based addons sometimes fail silently.

---

### 8. :eight: Addon Communication Channel Workarounds

| | |
|---|---|
| **Risk Rating** | :red_circle: **ENDANGERED** |
| **API/Function** | `C_ChatInfo.SendAddonMessage()`, `C_ChatInfo.RegisterAddonMessagePrefix()` |
| **Who uses it** | MRT, Cell, raid coordination addons, weakaura-sharing systems |
| **What it achieves** | Cross-player data sharing — assignments, cooldown notes, strategy information. Cell queues messages during encounters and flushes on `ENCOUNTER_END` |
| **Why Blizzard might close it** | Already restricted during encounters in Midnight. If addons start encoding combat-state information in post-encounter message flushes (recording who had what debuff and when), Blizzard could restrict post-encounter communication too |
| **Devastation if closed** | :boom: **HIGH** — Would break MRT's note-sharing, Cell's raid frame synchronization, and all cross-player coordination tools |

Cell's communication handling in PR #457 shows the pattern:

```lua
if not IsCommRestricted() then
    C_ChatInfo.SendAddonMessage(prefix, msg, channel)
else
    table.insert(messageQueue, {prefix, msg, channel})
    -- Flush on ENCOUNTER_END
end
```

The message queuing pattern is clever — don't communicate during encounters, but dump everything the moment the encounter ends. If Blizzard decides this post-encounter data dump gives too much analytical advantage (recording damage taken patterns, healing assignments that worked, etc.), they could restrict addon messages for a cooldown period after encounters too.

---

### 9. :nine: `UNIT_AURA` Event Mining with New Filter Categories

| | |
|---|---|
| **Risk Rating** | :green_circle: **SAFE** |
| **API/Function** | `UNIT_AURA` event, new 12.0.1 filter categories: `CROWD_CONTROL`, `BIG_DEFENSIVE`, `RAID_PLAYER_DISPELLABLE`, `RAID_IN_COMBAT` |
| **Who uses it** | Cell, Harreks Raid Frames, healer-focused addons |
| **What it achieves** | Track specific buff/debuff categories on raid frames without knowing the exact spell. "Show me all crowd control effects on this player" without knowing which CC it is |
| **Why Blizzard might close it** | These filters were *added* in 12.0.1 specifically to address healer pain points. Blizzard wants healers to have this data. Low risk of restriction |
| **Devastation if closed** | :boom: **MODERATE** — Would hurt healer addons specifically, but Blizzard added these intentionally |

This is the rare technique that Blizzard *wants* addons to use. The four new aura filter categories were added in 12.0.1 directly in response to healer community feedback. They represent Blizzard's approach to the "information vs. automation" line: addons can *display* that a player has a crowd control effect, but they can't *know* which specific CC it is or make automated decisions about it.

---

### 10. :keycap_ten: Edit Mode API Extension

| | |
|---|---|
| **Risk Rating** | :green_circle: **SAFE** |
| **API/Function** | Edit Mode Lua and XML APIs, `EditModeManagerFrame`, layout system APIs |
| **Who uses it** | Edit Mode Expanded, Edit Mode Tweaks, Edit Mode Features, NephUI (before shutdown) |
| **What it achieves** | Extends Blizzard's Edit Mode with additional movable frames, custom snap points, and layout options that Blizzard didn't include |
| **Why Blizzard might close it** | Very unlikely — Edit Mode is Blizzard's own UI customization framework. Extending it aligns perfectly with the "enhance, don't replace" philosophy |
| **Devastation if closed** | :boom: **LOW** — Would be hypocritical for Blizzard to restrict their own UI customization framework |

Edit Mode is Blizzard's answer to "let players customize without addons." Edit Mode extension addons add more frames to the system, more snap points, more layout options. This is possibly the safest technique on this entire list because restricting it would contradict Blizzard's own stated design goals.

---

### Risk Summary

| Risk Level | Count | Techniques |
|---|---|---|
| :skull: **NEXT PATCH** | 0 | — |
| :red_circle: **ENDANGERED** | 3 | Hard-coded timers, mob classification branching, addon communication channels |
| :yellow_circle: **WATCH** | 4 | Private Aura audio, `hooksecurefunc`, tooltip scanning, metatable manipulation |
| :green_circle: **SAFE** | 3 | Frame strata manipulation, `UNIT_AURA` filter mining, Edit Mode extension |

---

## :bar_chart: The Threat Matrix

How do different addon categories face different types of threats? This matrix maps every major addon category against every threat vector active in Midnight.

!!! abstract "How to read this matrix"
    Each cell rates the threat level for that category-threat combination:

    - :skull: = **Already dead** — This threat already killed or crippled this category
    - :red_circle: = **Critical** — Active and severe threat with no good workaround
    - :yellow_circle: = **Moderate** — Threat exists but workarounds are available
    - :green_circle: = **Low** — Category is relatively safe from this threat
    - :white_circle: = **Not applicable** — This threat doesn't affect this category

### The Full Matrix

| Category | API Removal | Native Replacement | Taint Lockdown | Secret Values | Comm Restrictions | Whitelist Changes | Overall Threat |
|---|---|---|---|---|---|---|---|
| **Rotation Helpers** | :skull: | :skull: | :red_circle: | :skull: | :white_circle: | :skull: | :skull: **EXTINCT** |
| **Combat WeakAuras** | :skull: | :yellow_circle: | :red_circle: | :skull: | :white_circle: | :red_circle: | :skull: **EXTINCT** |
| **Damage Meters** | :skull: | :red_circle: | :yellow_circle: | :skull: | :white_circle: | :yellow_circle: | :red_circle: **CRITICAL** |
| **Nameplate Addons** | :red_circle: | :yellow_circle: | :yellow_circle: | :skull: | :white_circle: | :red_circle: | :red_circle: **CRITICAL** |
| **Boss Mods** | :red_circle: | :red_circle: | :yellow_circle: | :red_circle: | :red_circle: | :yellow_circle: | :red_circle: **CRITICAL** |
| **Raid Frames** | :red_circle: | :yellow_circle: | :red_circle: | :red_circle: | :red_circle: | :yellow_circle: | :red_circle: **CRITICAL** |
| **UI Overhauls** | :yellow_circle: | :yellow_circle: | :red_circle: | :yellow_circle: | :white_circle: | :green_circle: | :yellow_circle: **ENDANGERED** |
| **Action Bars** | :yellow_circle: | :green_circle: | :red_circle: | :yellow_circle: | :white_circle: | :green_circle: | :yellow_circle: **ENDANGERED** |
| **Cooldown Trackers** | :red_circle: | :red_circle: | :yellow_circle: | :red_circle: | :white_circle: | :yellow_circle: | :yellow_circle: **ENDANGERED** |
| **Bag Addons** | :yellow_circle: | :green_circle: | :yellow_circle: | :green_circle: | :white_circle: | :green_circle: | :green_circle: **ADAPTING** |
| **Map / Quest Addons** | :green_circle: | :yellow_circle: | :green_circle: | :green_circle: | :white_circle: | :green_circle: | :green_circle: **ADAPTING** |
| **Chat Addons** | :green_circle: | :green_circle: | :yellow_circle: | :white_circle: | :yellow_circle: | :white_circle: | :green_circle: **ADAPTING** |
| **Auction / Economy** | :green_circle: | :green_circle: | :green_circle: | :green_circle: | :white_circle: | :green_circle: | :green_circle: **THRIVING** |
| **Housing Addons** | :white_circle: | :white_circle: | :green_circle: | :white_circle: | :white_circle: | :white_circle: | :green_circle: **THRIVING** |
| **Transmog / Collection** | :green_circle: | :green_circle: | :green_circle: | :white_circle: | :white_circle: | :white_circle: | :green_circle: **THRIVING** |

---

### Threat Vector Breakdown

??? danger ":skull: API Removal — 138 APIs removed in 12.0.0"
    The most brutal and immediate threat. Patch 12.0.0 removed 138 deprecated APIs and replaced `COMBAT_LOG_EVENT_UNFILTERED` (CLEU) with `COMBAT_LOG_EVENT_INTERNAL_UNFILTERED` and `COMBAT_LOG_MESSAGE`. This single change — removing CLEU — is what killed WeakAuras combat triggers, GTFO's ground effect detection, Details!' independent parsing, and Plater's NPC-specific logic. Addon developers had zero alternatives for some removed APIs. The replacement APIs (437 new ones added) require fundamentally different architectural approaches.

    **Most affected:** Rotation helpers (:skull:), combat WeakAuras (:skull:), damage meters (:skull:), nameplate addons (:red_circle:), boss mods (:red_circle:), raid frames (:red_circle:)

    **Next risk:** Patch 12.0.5 is confirmed to bring another round of API changes. Given the pattern, expect more namespaced replacements of remaining globals and possible restrictions on data-access APIs that are currently unrestricted.

??? danger ":red_circle: Native Replacement — Blizzard building it in"
    Blizzard shipped native replacements for damage meters (`C_DamageMeter`), boss timelines (`C_EncounterTimeline`), cooldown managers, combat audio alerts (`C_CombatAudioAlert`), and enhanced healer raid frames. The pattern is clear: if an addon category is popular enough, Blizzard will eventually build a native version and restrict the APIs the addon relied on.

    **The "Build It In" strategy has precedent:** Quest tracking (killed QuestHelper), item level display (absorbed GearScore), group finder (killed oQueue), threat meters (built into default UI), M+ scoring (absorbed Raider.io's concept). Each time, the native version was functional but less capable — and the addon category was simultaneously restricted.

    **Most affected:** Damage meters (:red_circle: — native meter exists but is inferior), boss mods (:red_circle: — native timeline exists), cooldown trackers (:red_circle: — native cooldown manager exists)

    **Next risk:** Native nameplate improvements (cooldown text, priority coloring), native WeakAura-like personal triggers, improved native raid frames for healers

??? danger ":red_circle: Taint Lockdown — The invisible killer"
    Taint — the system that prevents addon code from executing protected actions — is the most insidious threat because it's invisible until it strikes. An addon that works perfectly in testing can cause "Action blocked by an addon" errors mid-raid when a complex chain of function calls crosses a taint boundary. ElvUI's entire Midnight architecture was rebuilt to avoid taint, but the sheer complexity of a full UI overhaul means taint issues are almost inevitable.

    **Patch 11.0.0 restriction:** Hooks can no longer be installed on functions with "certain protected names." This restriction may expand in future patches.

    **Most affected:** UI overhauls (:red_circle:), action bars (:red_circle:), raid frames (:red_circle:) — anything that touches secure frames

    **Next risk:** Expanded list of protected function names, restrictions on `SetParent` for secure frames, or taint propagation changes that make the hidden-frame-reparenting trick unreliable

??? danger ":skull: Secret Values — The nuclear option"
    The foundational technology behind Midnight's restrictions. Combat data enters a "black box" — addons can customize visual presentation but cannot computationally access the data. `issecretvalue()`, `scrubsecretvalues()`, `secretwrap()`, and frame methods like `HasSecretValues()` define the boundary.

    **What's secret:** Health values in instances, cooldown states, buff/debuff data, combat log events, unit names in instances, targeting information during encounters

    **What's not secret:** Visual properties (size, color, position), mob classification (for now), housing data, non-combat UI state, whitelisted spells/resources

    **Most affected:** Everything combat-related. Rotation helpers (:skull:), combat WeakAuras (:skull:), damage meters (:skull:), nameplate addons (:skull:), boss mods (:red_circle:), raid frames (:red_circle:)

    **Next risk:** Expansion of Secret Values to cover mob classification, tooltip text during combat, or post-combat data access

??? warning ":yellow_circle: Communication Restrictions — The silent nerf"
    Addon-to-addon communication via `C_ChatInfo.SendAddonMessage()` is restricted during encounters. Cell queues messages and flushes on `ENCOUNTER_END`. MRT's note-sharing still works pre-pull. But if Blizzard decides post-encounter data sharing gives too much analytical advantage, they could extend the restriction window.

    **Most affected:** Raid coordination tools (:red_circle:), raid frames (:red_circle:), boss mods (:red_circle:)

    **Next risk:** Post-encounter communication cooldown, restrictions on message payload size, or limitations on registered addon prefixes

??? info ":green_circle: Whitelist Changes — The double-edged sword"
    The Secret Values whitelist determines which spells and resources are *not* secret. It's been growing: Skyriding abilities, Maelstrom Weapon, Soul Fragments, Combat Resurrection spells, string functions. Each addition slightly expands what addons can do. But the whitelist can also *shrink* — string functions were briefly removed before being restored.

    **Most affected (positively):** Cooldown trackers, raid frames, healer addons — each whitelist addition gives them more data to work with

    **Most affected (negatively):** Any addon relying on a whitelisted value that gets removed

    **Next risk:** Whitelist expansion for healing spells (good for healer addons) or whitelist contraction for classification/creature data (bad for nameplate addons)

---

### Category Deep Dives

??? abstract "Rotation Helpers — :skull: EXTINCT"
    **Threat Summary:** Every threat vector is fatal. API removal killed CLEU parsing. Secret Values made cooldown/buff state unreadable. Blizzard's class pruning was *explicitly* designed to reduce the need for rotation helpers. The native Cooldown Manager (however "barely functional") signals Blizzard's intent to handle this in-house.

    **Survivors:** None. Hekili is dead. MaxDPS is dead. All rotation suggestion addons are fundamentally impossible under Secret Values.

    **Could they return?** Only if Blizzard allows personal-only combat state access (WeakAuras' condition #2). Current trajectory suggests this won't happen.

??? abstract "Housing Addons — :green_circle: THRIVING"
    **Threat Summary:** Housing addons face essentially zero threats. The `C_Housing` and `C_HousingDecor` APIs are extensive, well-documented, and completely unrestricted. No Secret Values apply. No taint concerns (housing isn't combat). No communication restrictions (sharing decoration ideas isn't a competitive advantage).

    **Winners:** HomeBound (3M+ downloads), HomeDecor (vendor tracking + AH integration), Plumber (decor duplication), Housing Reps (reputation tracking)

    **Growth potential:** :rocket: Housing is Midnight's biggest new feature. The addon ecosystem here is *nascent* — we're in the "everyone builds a quest helper" phase of housing addons. Expect consolidation, specialization, and eventual Blizzard absorption of the most popular features.

??? abstract "Boss Mods — :red_circle: CRITICAL"
    **Threat Summary:** Boss mods survived by negotiating directly with Blizzard and finding creative workarounds (Private Aura audio, hard-coded timers, timeline reskinning). But every workaround is a potential future restriction. Hard-coded timers are the biggest target — they effectively reconstruct boss mod functionality through pre-authored predictions.

    **The Negotiation Advantage:** DBM's MysticalOS has a demonstrated direct line to Ion Hazzikostas. This political capital is as important as any technical workaround. If Blizzard plans restrictions that would break boss mods, MysticalOS will likely get advance warning.

    **Survival strategy:** Continue the "enhancement layer" approach. Skin the native timeline. Use Private Aura audio for awareness. Accept that independent mechanic detection is gone forever.

??? abstract "The 'Better___' Ecosystem — :green_circle: ADAPTING"
    **Threat Summary:** The "enhance, don't replace" addons are in the best position of any combat-adjacent category. They don't replace Blizzard's UI — they modify it visually after render. BetterBlizzFrames (3.9M DL), BetterBlizzPlates, BetterBags (3.6M DL), BetterCooldownManager all follow this pattern.

    **Key vulnerability:** `hooksecurefunc` restrictions. If Blizzard expands the list of protected functions that can't be hooked, the entire "Better___" philosophy collapses. But restricting hooks would contradict Blizzard's own encouragement of the "enhance" approach.

    **The irony:** These addons exist *because* Blizzard restricted combat addons. The "Better___" wave is a direct response to Addon Disarmament. Restricting them would be Blizzard undermining their own strategy.

---

### The Survivability Formula

Based on the patterns above, here's what determines whether an addon category survives Midnight and future patches:

!!! success "Highest survivability"
    1. **Doesn't touch combat data** (housing, transmog, auction, collections)
    2. **Enhances native UI rather than replacing it** (BetterBlizzFrames, Arc UI)
    3. **Uses only visual customization** (colors, textures, positions, sizes)
    4. **Aligns with Blizzard's stated philosophy** ("present information, player decides")

!!! failure "Lowest survivability"
    1. **Requires computational access to combat state** (WeakAuras, Hekili)
    2. **Replaces Blizzard's native features entirely** (old-style damage meters, unit frame replacements)
    3. **Makes automated decisions based on game state** (rotation helpers, one-button automation)
    4. **Provides competitive advantage through information asymmetry** (spy addons, detailed nameplate data)

---

## The Road Ahead

!!! quote "Ion Hazzikostas ([PC Gamer, January 2026](https://www.pcgamer.com/games/world-of-warcraft/as-world-of-warcraft-winds-down-its-combat-addon-support-director-ion-hazzikostas-is-all-composure-about-rule-breakers-because-frankly-this-is-far-from-the-first-time/))"
    "Going back 20 years, people used to be able to use AddOns like Decursive... we broke that functionality."

The addon ecosystem has survived every restriction Blizzard has ever imposed. Protected functions in 2.0. The addon monetization ban in 2009. AVR's 3D projection lockdown in 2010. The oQueue communication kill in 2014. Spy's combat log range reduction in 2019. Each time, developers adapted, innovated, and found new boundaries to push.

Midnight's Addon Disarmament is the most aggressive restriction yet — but the pattern holds. WeakAuras is dead, but Northern Sky Raid Tools has 3.2M downloads. Plater is gutted, but Platynator was born. ElvUI nearly died, but BetterBlizzFrames hit 3.9M downloads. The ecosystem doesn't die — it evolves.

**Keep your bingo card handy.** Patch 12.0.5 is coming.

---

!!! tip "Building for the future?"
    Start with the [Getting Started](getting-started.md) guide, understand the [Midnight restrictions](midnight.md), and study the [Coding for Midnight](midnight-patterns.md) patterns. The safest addons are the ones that enhance Blizzard's native UI — and the most exciting ones are the housing addons that have no restrictions at all.

---

## Sources

This page synthesizes data from verified reporting across the Midnight addon ecosystem:

- [Blizzard: Combat Philosophy and Addon Disarmament](https://news.blizzard.com/en-us/article/24246290/combat-philosophy-and-addon-disarmament-in-midnight) — Official policy statement
- [Blizzard: Hotfixes March 11, 2026](https://news.blizzard.com/en-us/article/24266320/hotfixes-march-11-2026) — Macro restrictions
- [Wowhead: Addons Continue to Provide Advantage](https://www.wowhead.com/news/addons-continue-to-provide-a-massive-advantage-in-midnight-despite-blizzards-379870) — "Addons Still Win" investigation
- [Wowhead: Boss Mods Teaching Old Tricks](https://www.wowhead.com/news/boss-mod-addons-in-midnight-teaching-old-mods-new-tricks-380024) — Boss mod adaptations
- [Wowhead: DBM & BigWigs Meet Ion](https://www.wowhead.com/news/more-boss-mod-functionality-coming-in-midnight-dbm-and-big-wigs-meet-with-ion-379149) — Developer summit
- [Wowhead: Nameplates in Midnight](https://www.wowhead.com/news/nameplates-in-midnight-whats-changing-and-what-add-ons-can-i-use-379924) — Nameplate analysis
- [Wowhead: Blizzard Damage Meter Shortcomings](https://www.wowhead.com/news/blizzards-damage-meter-shortcomings-in-midnight-pre-patch-and-addon-alternatives-379992) — Native meter assessment
- [Wowhead: Lua API Changes for Launch](https://www.wowhead.com/news/addon-changes-for-midnight-launch-ending-soon-with-release-candidate-coming-380133) — API freeze announcement
- [WeakAuras Patreon Statement](https://www.patreon.com/posts/midnight-144610594) — Official "no Midnight version" position
- [PCGamesN: WeakAuras Update](https://www.pcgamesn.com/world-of-warcraft/midnight-weakauras-update) — "Stance remains unchanged"
- [Icy Veins: WeakAuras Responds](https://www.icy-veins.com/wow/news/weakauras-responds-to-addon-limitation-loosening-in-midnight/) — Loosening evaluation
- [Icy Veins: Blizzard Relaxing Limitations](https://www.icy-veins.com/wow/news/blizzard-relaxing-more-addon-limitations-in-midnight/) — Whitelist expansion
- [PC Gamer: Ion on Combat Addons](https://www.pcgamer.com/games/world-of-warcraft/as-world-of-warcraft-winds-down-its-combat-addon-support-director-ion-hazzikostas-is-all-composure-about-rule-breakers-because-frankly-this-is-far-from-the-first-time/) — "Going back 20 years"
- [PC Gamer: Classes Pruned](https://www.pcgamer.com/games/world-of-warcraft/wows-classes-were-pruned-for-midnight-because-many-were-built-in-a-world-where-its-devs-assumed-theyd-be-using-addons/) — Class design tied to addon dependency
- [GamesRadar: Ion on Addon Changes](https://www.gamesradar.com/games/world-of-warcraft/we-probably-shouldve-done-something-sooner-world-of-warcraft-director-says-the-mmos-addon-changes-have-been-a-long-time-coming-but-better-late-than-never/) — "Should've done something sooner"
- [GitHub: Cell PR #457](https://github.com/enderneko/Cell/pull/457) — Migration masterclass
- [Xepheris: Nameplates Technical Deep-Dive](https://gerritalex.de/blog/nameplates-in-midnight) — Technical analysis
- [Warcraft Tavern: Secret Values Explained](https://www.warcrafttavern.com/wow/news/wow-midnight-developer-talk-new-secret-values-combat-info-cooldown-manager-combat-addons-nerfed/) — Secret Values system
- [Warcraft Wiki: Patch 12.0.0 API Changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) — 138 removed, 437 added
- [Warcraft Wiki: Patch 12.0.1 API Changes](https://warcraft.wiki.gg/wiki/Patch_12.0.1/API_changes) — 59 new APIs
- [The Escapist: Quazii Quits](https://www.escapistmagazine.com/news-popular-wow-content-creator-quazzi-quits/) — Vibecoded Wars
- [Wago.io Security Notice](https://accounts.wago.io/security-notice) — Data breach
- [Blizzard Forums: Accessibility](https://us.forums.blizzard.com/en/wow/t/midnight-addon-changes-exclude-disabled-players-like-me/2215814) — Accessibility impact
- [Blizzard Forums: Great Addon Purge](https://us.forums.blizzard.com/en/wow/t/updates-on-what-we-know-so-far-about-the-great-addon-purge-of-2026/2184210) — Community impact tracking
