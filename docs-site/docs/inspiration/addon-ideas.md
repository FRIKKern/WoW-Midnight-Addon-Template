---
title: "Creative Addon Ideas for Midnight 12.0"
description: "12 creative, buildable WoW addon ideas for Patch 12.0 Midnight — from beginner poison reminders to advanced boss timeline enhancers, each with APIs, difficulty, and build commands."
---

# Creative Addon Ideas for Midnight 12.0

Midnight closed some doors and opened many more. `COMBAT_LOG_EVENT_UNFILTERED` is gone. Raw combat numbers are locked behind Secret Values. Addons that replaced Blizzard's secure frames wholesale no longer work. For a moment, the addon community held its breath.

Then the new APIs arrived — **437 of them**. Entire namespaces that never existed before: `C_Housing` for player housing, `C_EncounterEvents` for boss ability timelines, `C_HousingPhotoSharing` for social screenshot galleries, `C_DamageMeter` for Blizzard's own combat metrics. The `Duration` and `ColorCurve` objects give addons new ways to display time and health without ever touching a raw number. `Frame:RegisterEventCallback()` simplifies event handling. The Cooldown Manager is now a first-class system addons can hook into.

The "BetterX" pattern — **enhance Blizzard's UI rather than replace it** — is the dominant design philosophy for Midnight. Addons that skin, extend, overlay, and augment Blizzard's frames thrive. Addons that rip them out and start over break. This isn't a limitation; it's a design constraint that produces better, more resilient addons.

Every idea below is buildable today. Each includes the WoW 12.0 APIs involved, a difficulty rating, the recommended [development mode](../better-addons.md), and a `/wow-create` command to scaffold it instantly. Whether you're writing your first addon or your fiftieth, there's something here for you.

---

## Quality of Life Addons

The bread and butter of addon development. These solve small, specific annoyances that affect millions of players every day.

### PoisonPal — Rogue Poison Reminder :material-star: Beginner { #poisonpal }

**What it does:** Monitors your active poison weapon enchants, warns you when they expire or are missing entirely, and auto-suggests the right poisons based on your current spec and talent build.

**Why players want it:** Every Rogue has zoned into a dungeon, pulled the first pack, and realized their poisons expired ten minutes ago. PoisonPal eliminates that moment of shame with a subtle reminder on screen and a pre-pull check.

**Mode: :material-shield-check: Blizzard Faithful** — Pure data monitoring with non-intrusive alerts. No frame replacement, no combat interaction.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_UnitAuras.GetAuraDataByIndex()` | Check active poison buffs on the player |
    | `GetWeaponEnchantInfo()` | Detect weapon enchant (poison) presence and remaining duration |
    | `C_ClassTalents.GetActiveConfigID()` | Determine current spec for poison suggestions |
    | `Settings.RegisterAddOnSetting()` | User preferences (alert style, sound toggle) |
    | `AddonCompartmentFunc` | Minimap dropdown access |

**Technical approach:**

- Register `UNIT_AURA`, `PLAYER_ENTERING_WORLD`, `PLAYER_EQUIPMENT_CHANGED`, and `ZONE_CHANGED_NEW_AREA` events
- On each trigger, call `GetWeaponEnchantInfo()` to check main-hand and off-hand enchant status
- Compare against a lookup table of poison spell IDs per spec (Assassination vs. Subtlety vs. Outlaw)
- Show a small alert frame anchored to `UIParent` with a fade-in animation when poisons are missing
- Suppress alerts in rest areas and non-combat zones via `IsResting()`

!!! tip "Build This"

    ```
    /wow-create PoisonPal — A rogue poison reminder addon that warns when
    poisons expire or are missing, suggests correct poisons for current spec,
    uses GetWeaponEnchantInfo and UNIT_AURA, Blizzard Faithful mode
    ```

---

### RepairBuddy — Smart Auto-Repair & Vendor :material-star: Beginner { #repairbuddy }

**What it does:** Automatically repairs all gear when you visit a vendor (using guild funds if available and permitted), auto-sells grey/junk items, tracks cumulative repair costs per session and lifetime, and warns when any equipped item drops below 20% durability.

**Why players want it:** Repair and junk-selling is pure tedium. Every player does it, nobody enjoys clicking through the vendor UI. RepairBuddy handles it silently and gives you data about where your gold actually goes.

**Mode: :material-shield-check: Blizzard Faithful** — Automates existing vendor interactions with zero UI replacement.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `MERCHANT_SHOW` event | Fires when vendor window opens |
    | `RepairAllItems(useGuildFunds)` | Repair all equipped gear |
    | `GetRepairAllCost()` | Get total repair cost before repairing |
    | `C_Item.GetItemInfo()` | Check item quality (0 = grey/junk) |
    | `C_Container.GetContainerItemInfo()` | Scan bags for junk items |
    | `C_Container.UseContainerItem()` | Sell an item to the vendor |
    | `GetInventoryItemDurability(slot)` | Check durability of equipped items |

**Technical approach:**

- On `MERCHANT_SHOW`, call `GetRepairAllCost()` to check if repairs are needed
- If cost > 0, attempt `RepairAllItems(true)` for guild repair first, fall back to `RepairAllItems(false)`
- Iterate all bag slots with `C_Container.GetContainerItemInfo()`, sell items with quality == 0
- Print a summary to chat: "Repaired (14g 32s) | Sold 7 junk items (2g 88s)"
- Track totals in `SavedVariables` with per-session and per-character breakdowns
- On `PLAYER_EQUIPMENT_CHANGED`, scan all 19 inventory slots for low durability and show a warning

!!! tip "Build This"

    ```
    /wow-create RepairBuddy — Auto-repair and junk-sell addon with guild
    repair support, durability warnings, cost tracking in SavedVariables,
    Blizzard Faithful mode
    ```

---

### WeeklyWise — Activity Checklist :material-star::material-star: Intermediate { #weeklywise }

**What it does:** A cross-character weekly activity tracker that monitors Great Vault progress, Mythic+ runs, world boss kills, weekly quest completions, and raid lockouts. Displays a clean checklist showing what's done and what's left — across all your alts.

**Why players want it:** Players with multiple characters lose track of which alt has capped their Vault, which one killed the world boss, and which still needs a Mythic+ run. WeeklyWise gives you a single dashboard across all characters.

**Mode: :material-palette: Enhancement Artist** — Custom UI panel that enhances the existing Great Vault and activity tracking systems.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_WeeklyRewards.GetActivities()` | Great Vault progress for current character |
    | `C_WeeklyRewards.CanClaimRewards()` | Check if vault is ready to open |
    | `C_MythicPlus.GetRunHistory()` | Mythic+ runs this week |
    | `GetSavedInstanceInfo(index)` | Raid lockout status |
    | `C_QuestLog.IsQuestFlaggedCompleted()` | Weekly quest tracking |
    | `C_DateAndTime.GetSecondsUntilWeeklyReset()` | Time until Tuesday reset |

**Technical approach:**

- Use `SavedVariables` with a table keyed by `UnitGUID("player")` to store per-character data
- On `PLAYER_LOGIN`, snapshot current character's vault progress, M+ history, and lockouts
- On `WEEKLY_REWARDS_UPDATE` and `CHALLENGE_MODE_COMPLETED`, refresh vault data
- Build a ScrollFrame-based UI panel showing all characters in rows with checkmark columns
- Calculate reset countdown using `C_DateAndTime.GetSecondsUntilWeeklyReset()` and display in header
- Register a slash command (`/weeklywise` or `/ww`) and Addon Compartment entry for access

!!! warning "Cross-Character Data"

    You cannot query another character's data while logged into a different one. WeeklyWise snapshots each character's state at login time and stores it in account-wide `SavedVariables`. Data is only as fresh as the last login on each character.

!!! tip "Build This"

    ```
    /wow-create WeeklyWise — Cross-character weekly activity tracker with
    Great Vault progress, M+ history, world boss kills, raid lockouts,
    ScrollFrame UI, account-wide SavedVariables, Enhancement Artist mode
    ```

---

## Housing Addons — The New Frontier

Player housing is Midnight's headline feature, and it comes with an entirely new API surface. The `C_Housing` and `C_HousingPhotoSharing` namespaces are brand new — meaning there's almost no competition. If you build a housing addon now, you're first to market.

### HomeSnap — Housing Photo Gallery :material-star::material-star: Intermediate { #homesnap }

**What it does:** A social housing screenshot browser. Players can capture screenshots of their housing plots, tag them with categories (cozy, gothic, nature, PvP arena), and share them via the `C_HousingPhotoSharing` API. Other players can browse, rate, and favorite designs — all in-game.

**Why players want it:** Housing is inherently social and creative. Players want to show off their builds, get inspiration from others, and curate a gallery of favorites. HomeSnap turns housing screenshots from ephemeral moments into a browsable, shareable collection.

**Mode: :material-palette: Enhancement Artist** — Extends Blizzard's housing photo system with a richer browsing and social experience.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_HousingPhotoSharing.SharePhoto()` | Share a housing screenshot |
    | `C_HousingPhotoSharing.GetSharedPhotos()` | Retrieve shared photos |
    | `C_HousingPhotoSharing.GetPhotoMetadata()` | Photo tags, owner, timestamp |
    | `C_Housing.GetPlotInfo()` | Current plot details |
    | `Screenshot()` | Capture a screenshot |
    | `C_ChatInfo.SendAddonMessage()` | Share favorites with guild/party |

**Technical approach:**

- Hook into Blizzard's housing UI to add a "Share to HomeSnap" button using `hooksecurefunc`
- Use `C_HousingPhotoSharing` to publish and retrieve photos with metadata tags
- Build a grid-view gallery frame with thumbnail previews using `Texture` widgets
- Implement a tagging system stored in `SavedVariables` for local categorization
- Add guild-level photo sharing via `C_ChatInfo.SendAddonMessage()` on the `GUILD` channel

!!! warning "Communication Lockdown"

    `C_ChatInfo.SendAddonMessage()` is blocked during M+ keystones, rated PvP, and boss encounters. HomeSnap should only send photo-sharing messages outside of restricted contexts. Check `C_ChatInfo.InChatMessagingLockdown()` before sending.

!!! tip "Build This"

    ```
    /wow-create HomeSnap — Housing photo gallery addon using
    C_HousingPhotoSharing API, grid thumbnail browser, tagging system,
    guild sharing via addon messaging, Enhancement Artist mode
    ```

---

### DecorTracker — Housing Decoration Collector :material-star::material-star: Intermediate { #decortracker }

**What it does:** A comprehensive decoration collection tracker. Catalogs all 1,690+ housing decorations in the game, shows which ones you own, which are craftable by your professions, which drop from specific bosses or vendors, and highlights the cheapest path to acquire missing ones.

**Why players want it:** Housing decorations come from dozens of sources — vendors, crafting, drops, achievements, reputation, and the Trading Post. Without a tracker, players have no idea what they're missing or where to get it.

**Mode: :material-palette: Enhancement Artist** — Collection management UI that enhances the housing decoration browser.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_Housing.GetDecorationCategories()` | All decoration category IDs |
    | `C_Housing.GetDecorationsForCategory()` | Decorations in a category |
    | `C_Housing.IsDecorationOwned(decorID)` | Check ownership |
    | `C_Housing.GetDecorationSourceInfo()` | Where a decoration comes from |
    | `C_TradeSkillUI.GetRecipeInfo()` | Crafting recipe data |
    | `C_CurrencyInfo.GetCurrencyInfo()` | Currency costs |

**Technical approach:**

- On `PLAYER_LOGIN`, scan all decoration categories and build a complete catalog in memory
- Cross-reference with `C_Housing.IsDecorationOwned()` to build owned/missing lists
- Group by source type: vendor, crafting, dungeon drop, achievement, reputation, Trading Post
- For craftable items, check if the player's professions can craft them via `C_TradeSkillUI`
- Build a filterable list UI with search, category tabs, and source-type filters
- Show collection progress as "247 / 1,693" with a progress bar per category

!!! tip "Build This"

    ```
    /wow-create DecorTracker — Housing decoration collection tracker,
    catalogs all decorations with source info, ownership status, crafting
    cross-reference, filterable list UI, Enhancement Artist mode
    ```

---

## UI Enhancement Addons

These addons demonstrate the "BetterX" pattern at its best: hooking into Blizzard's existing frames to make them look and work better without replacing the underlying secure infrastructure.

### BetterCooldowns — Cooldown Manager Skin :material-star::material-star: Intermediate { #bettercooldowns }

**What it does:** Enhances Blizzard's built-in Cooldown Manager with cleaner visuals, customizable countdown text, spell-specific color coding, and group-organized cooldown categories. Uses the new `Duration` objects for Secret Values-safe cooldown display.

**Why players want it:** Blizzard's Cooldown Manager (introduced in 11.0) shows your ability cooldowns, but the default presentation is minimal. BetterCooldowns adds polish — colored borders by cooldown length, category grouping (offensive / defensive / utility), and configurable text formatting.

**Mode: :material-palette: Enhancement Artist** — Skins and extends the existing Cooldown Manager frames.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_Spell.GetSpellCooldownDuration(spellID)` | Returns a `Duration` object (Secret Values-safe) |
    | `Cooldown:SetCooldownFromDurationObject(dur)` | Display cooldown from Duration |
    | `StatusBar:SetTimerDuration(duration)` | Self-updating timer bar |
    | `hooksecurefunc()` | Hook Blizzard's cooldown frame updates |
    | `C_Spell.GetSpellTexture(spellID)` | Get spell icon |
    | `C_Spell.GetSpellInfo(spellID)` | Get spell metadata (returns table) |

**Technical approach:**

- Use `hooksecurefunc` to hook the Cooldown Manager's frame update functions
- On each cooldown frame update, apply custom styling: border color based on remaining time, category icon overlays
- Use `C_Spell.GetSpellCooldownDuration()` to get Duration objects — never raw seconds
- Pass Duration objects directly to `Cooldown:SetCooldownFromDurationObject()` for display
- Add a Settings panel for color customization, text format preferences, and category definitions
- Defer all frame modifications behind `InCombatLockdown()` checks for secure frames

```lua
-- Duration-safe cooldown display (Midnight pattern)
local dur = C_Spell.GetSpellCooldownDuration(spellID)
if dur then
    cooldownFrame:SetCooldownFromDurationObject(dur)
end
```

!!! tip "Build This"

    ```
    /wow-create BetterCooldowns — Cooldown Manager enhancement with
    Duration objects, spell color coding, category grouping, hooksecurefunc
    on Blizzard frames, Enhancement Artist mode
    ```

---

### PlateStylist — Nameplate Restyler :material-star::material-star::material-star: Advanced { #platestylist }

**What it does:** Reskins Blizzard's default nameplates with class-colored health bars, enhanced threat indicators, cast bar improvements, target highlighting, and aura filtering — all without replacing the secure nameplate frames.

**Why players want it:** Default nameplates are functional but visually bland. PlateStylist makes them information-dense and attractive while keeping the secure frame infrastructure intact. Unlike full nameplate replacements, it survives Blizzard UI updates gracefully.

**Mode: :material-rocket: Boundary Pusher** — Deep hooks into nameplate frames, walking the line between enhancement and replacement.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `NAME_PLATE_UNIT_ADDED` event | Fires when a nameplate appears |
    | `NAME_PLATE_UNIT_REMOVED` event | Fires when a nameplate disappears |
    | `C_NamePlate.GetNamePlateForUnit(unit)` | Get the nameplate frame for a unit |
    | `CompactUnitFrame_UpdateAll` | Blizzard's internal nameplate update function |
    | `UnitHealthPercent(unit)` | Secret Values-safe health percentage |
    | `UnitThreatSituation(unit)` | Threat status (0-3) |
    | `StatusBar:SetValue()` | Accepts secret values directly |

**Technical approach:**

- Hook `NAME_PLATE_UNIT_ADDED` to apply styling when nameplates appear
- Use `hooksecurefunc("CompactUnitFrame_UpdateHealth", ...)` to restyle after Blizzard updates
- Set health bar color based on `UnitThreatSituation()` return value (0=grey, 1=yellow, 2=orange, 3=red)
- Add class color via `RAID_CLASS_COLORS[select(2, UnitClass(unit))]` for player nameplates
- Use `UnitHealthPercent()` with `StatusBar:SetValue()` — never extract raw health numbers
- Check `frame:IsForbidden()` before every modification to avoid taint on protected frames
- Apply cast bar enhancements by hooking `CastingBarFrame_OnEvent` equivalents

```lua
-- Safe nameplate styling (check IsForbidden first)
hooksecurefunc("CompactUnitFrame_UpdateHealth", function(frame)
    if frame:IsForbidden() then return end
    local unit = frame.unit
    if not unit then return end

    local _, class = UnitClass(unit)
    if class and UnitIsPlayer(unit) then
        local color = RAID_CLASS_COLORS[class]
        frame.healthBar:SetStatusBarColor(color.r, color.g, color.b)
    end
end)
```

!!! warning "Nameplate Taint"

    Nameplate frames are partially secure. Always call `frame:IsForbidden()` before modifying any nameplate child frame. Never call `:Hide()` on nameplate elements — use `:SetAlpha(0)` or reparent to a hidden frame instead. Test extensively in M+ and raid encounters where the secure model is most strictly enforced.

!!! tip "Build This"

    ```
    /wow-create PlateStylist — Nameplate restyler with class colors, threat
    coloring, cast bar enhancement, hooksecurefunc on CompactUnitFrame,
    IsForbidden checks, Boundary Pusher mode
    ```

---

### BossTimeline+ — Boss Timeline Enhancer :material-star::material-star::material-star: Advanced { #bosstimeline }

**What it does:** Enhances Blizzard's native boss ability timeline (new in 12.0) with custom annotations, phase markers, personal cooldown overlay, and historical comparison from previous pulls. Uses the `C_EncounterEvents` namespace to read boss ability data that Blizzard now exposes natively.

**Why players want it:** Blizzard added a built-in boss ability timeline in Midnight, but it's minimal. BossTimeline+ layers additional context on top — showing when your defensive cooldowns will be available relative to boss abilities, marking phase transitions, and letting raid leaders annotate the timeline with strategy notes.

**Mode: :material-rocket: Boundary Pusher** — Deeply extends Blizzard's encounter timeline system with overlay frames and custom data layers.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_EncounterEvents.GetEncounterTimeline()` | Boss ability timeline data |
    | `C_EncounterEvents.GetCurrentPhase()` | Current encounter phase |
    | `C_EncounterEvents.GetAbilityInfo(abilityID)` | Details for a specific boss ability |
    | `C_Spell.GetSpellCooldownDuration(spellID)` | Player cooldown as Duration object |
    | `ENCOUNTER_START` / `ENCOUNTER_END` events | Encounter boundaries |
    | `C_ChatInfo.SendAddonMessage()` | Share annotations with raid (outside lockdown) |

**Technical approach:**

- Hook Blizzard's encounter timeline frame to add overlay elements using `hooksecurefunc`
- On `ENCOUNTER_START`, query `C_EncounterEvents.GetEncounterTimeline()` for the full ability schedule
- Create a secondary timeline track showing the player's defensive cooldown availability
- Use `C_Spell.GetSpellCooldownDuration()` to project when cooldowns will be available (Duration objects)
- Store per-boss annotations in `SavedVariables` keyed by encounter ID
- Share annotations with raid via `C_ChatInfo.SendAddonMessage()` — only outside encounter lockdown
- On `ENCOUNTER_END`, snapshot the pull for historical comparison

!!! warning "Encounter Communication Lockdown"

    `SendAddonMessage()` is completely blocked during boss encounters in Midnight. BossTimeline+ must pre-share all annotations before the pull starts and queue any mid-fight updates for transmission after `ENCOUNTER_END`. Use `C_ChatInfo.InChatMessagingLockdown()` to detect the restricted state.

!!! tip "Build This"

    ```
    /wow-create BossTimeline+ — Boss timeline enhancer using
    C_EncounterEvents API, personal cooldown overlay with Duration objects,
    annotation system, raid sharing via addon comms, Boundary Pusher mode
    ```

---

## Social & Economy Addons

WoW is an MMO — the best addons enhance the social and economic experience, not just combat.

### GuildBoard — Guild Event Coordinator :material-star::material-star: Intermediate { #guildboard }

**What it does:** An in-game guild event board where officers can post events (raid nights, M+ groups, housing tours, PvP nights) and members can sign up with role preferences. Think a lightweight in-game calendar enhancement with RSVP tracking.

**Why players want it:** The built-in calendar is clunky and limited. Discord handles scheduling but requires tabbing out. GuildBoard keeps event coordination where it belongs — inside the game, visible at a glance when you log in.

**Mode: :material-shield-check: Blizzard Faithful** — Uses standard addon messaging for data sync. Clean, non-intrusive UI.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_ChatInfo.SendAddonMessage()` | Sync event data with guild members |
    | `C_ChatInfo.RegisterAddonMessagePrefix()` | Register addon comm channel |
    | `C_GuildInfo.GetGuildRosterInfo()` | Guild member data |
    | `C_Calendar.GetDayEvent()` | Read existing calendar events |
    | `C_DateAndTime.GetCurrentCalendarTime()` | Current date/time |
    | `MenuUtil.CreateContextMenu()` | Right-click menus for event actions |

**Technical approach:**

- Officers create events via a simple form frame (title, date, time, description, role slots)
- Serialize event data and broadcast to guild via `C_ChatInfo.SendAddonMessage("GuildBoard", data, "GUILD")`
- Members receive events and store them in local `SavedVariables`
- Sign-up responses sent back via addon message with role preference (tank/healer/DPS)
- Display a bulletin board frame with upcoming events sorted by date
- Use `MenuUtil.CreateContextMenu()` for right-click actions (edit, delete, sign up, withdraw)
- Show a login notification if new events were posted since last session

!!! warning "Message Size Limits"

    `SendAddonMessage()` has a 255-character limit per message. Serialize event data into multiple chunks with sequence numbers. Use a simple prefix protocol: `"EVT:1/3:data..."` for chunked transmission.

!!! tip "Build This"

    ```
    /wow-create GuildBoard — Guild event board with RSVP system, addon
    messaging sync, officer event creation, role sign-ups, bulletin board
    UI, Blizzard Faithful mode
    ```

---

### CraftCalc — Crafting Profit Calculator :material-star::material-star: Intermediate { #craftcalc }

**What it does:** Scans your known crafting recipes, cross-references reagent costs from the Auction House, and calculates profit margins for every craftable item. Highlights the most profitable crafts, tracks material price trends, and integrates with the crafting order system.

**Why players want it:** The crafting economy in WoW is deep but opaque. Players craft items without knowing if they're profitable. CraftCalc turns the crafting UI into a profit dashboard, showing exactly which recipes make gold and which burn it.

**Mode: :material-speedometer: Performance Zealot** — Heavy data processing with thousands of recipe and price calculations. Must be fast and memory-efficient.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `C_TradeSkillUI.GetRecipeInfo(recipeID)` | Recipe metadata |
    | `C_TradeSkillUI.GetRecipeSchematic(recipeID)` | Reagent requirements |
    | `C_AuctionHouse.GetItemSearchResultInfo()` | AH price data |
    | `C_AuctionHouse.SendSearchQuery()` | Query AH for prices |
    | `C_CraftingOrders.GetOrdersForRecipe()` | Active crafting orders |
    | `C_TradeSkillUI.GetAllRecipeIDs()` | All known recipes |

**Technical approach:**

- On `TRADE_SKILL_SHOW`, scan all recipes via `C_TradeSkillUI.GetAllRecipeIDs()`
- For each recipe, get reagent list via `C_TradeSkillUI.GetRecipeSchematic()`
- Query AH prices for both reagents and finished products (batch queries to avoid throttling)
- Calculate: `profit = sellPrice - sum(reagentCosts) - AH cut (5%)`
- Cache price data in `SavedVariables` with timestamps to avoid excessive AH queries
- Display a sortable table: recipe name, reagent cost, sell price, profit, margin %
- Use `C_Timer.NewTicker()` to stagger AH queries (max 1 per second to avoid server throttling)
- Highlight recipes with profit > 0 in green, losses in red

!!! tip "Build This"

    ```
    /wow-create CraftCalc — Crafting profit calculator with AH price
    lookups, reagent cost summation, profit margin display, price caching,
    sortable recipe table, Performance Zealot mode
    ```

---

## Fun & Novelty Addons

Not everything has to optimize your gameplay. These addons add joy, personality, and memorable moments to the game.

### AchieveMore — Achievement Progress Dashboard :material-star: Beginner { #achievemore }

**What it does:** A visual achievement progress dashboard that shows your completion percentage across all achievement categories, highlights nearly-complete achievements (90%+ progress), and suggests the easiest ones to finish next based on your current progress.

**Why players want it:** The default achievement UI is a sprawling tree that's hard to navigate. AchieveMore gives you a bird's-eye view of where you stand and what's low-hanging fruit. Achievement hunters live for that "97% complete" nudge.

**Mode: :material-palette: Enhancement Artist** — Custom dashboard UI that reads from Blizzard's achievement data.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `GetAchievementInfo(achievementID)` | Achievement name, description, completion status |
    | `GetAchievementCriteriaInfo(achieveID, index)` | Individual criteria progress |
    | `GetCategoryList()` | All achievement category IDs |
    | `GetCategoryInfo(categoryID)` | Category name and parent |
    | `GetCategoryNumAchievements(catID)` | Count of achievements in category |
    | `GetStatistic(achieveID)` | Statistic value for tracking achievements |

**Technical approach:**

- On slash command or Compartment click, scan all achievement categories via `GetCategoryList()`
- For each category, iterate achievements with `GetCategoryNumAchievements()` and `GetAchievementInfo()`
- Calculate completion percentage per category and overall
- For incomplete achievements, check criteria progress to find "almost done" candidates
- Sort near-complete achievements by remaining criteria count (ascending) for the "Finish These" list
- Display as a grid of category cards with progress bars and a "Suggested Next" sidebar
- Cache the scan results for the session — achievements don't change frequently

!!! tip "Build This"

    ```
    /wow-create AchieveMore — Achievement progress dashboard with category
    completion percentages, near-complete suggestions, progress bar grid,
    Enhancement Artist mode
    ```

---

### ScreenShotter — Smart Screenshot Automator :material-star::material-star: Intermediate { #screenshotter }

**What it does:** Automatically captures screenshots at memorable moments — achievement unlocks, level-ups, boss kills, rare mount drops, first-time dungeon completions, and housing milestones. Optionally hides the UI before capturing for clean shots, and logs every screenshot with metadata (date, zone, event type) in SavedVariables.

**Why players want it:** Years later, players wish they had screenshots of their first Mythic kill or that rare mount drop. ScreenShotter ensures you never miss a moment, building an automatic photo journal of your WoW journey.

**Mode: :material-shield-check: Blizzard Faithful** — Pure event monitoring with the built-in `Screenshot()` function. Zero UI modification.

!!! info "Key APIs"

    | API | Purpose |
    |-----|---------|
    | `Screenshot()` | Capture a screenshot |
    | `ACHIEVEMENT_EARNED` event | Achievement unlock |
    | `PLAYER_LEVEL_UP` event | Level gained |
    | `ENCOUNTER_END` event | Boss kill (check success param) |
    | `CHAT_MSG_LOOT` event | Loot received (filter for mounts/legendaries) |
    | `NEW_MOUNT_ADDED` event | New mount learned |
    | `C_Map.GetBestMapForUnit("player")` | Current zone for metadata |
    | `C_DateAndTime.GetCurrentCalendarTime()` | Timestamp for metadata |

**Technical approach:**

- Register all trigger events: `ACHIEVEMENT_EARNED`, `PLAYER_LEVEL_UP`, `ENCOUNTER_END`, `NEW_MOUNT_ADDED`, `CHAT_MSG_LOOT`, `ZONE_CHANGED_NEW_AREA`
- On each trigger, optionally hide UI with `UIParent:SetAlpha(0)`, take screenshot, restore with `C_Timer.After(0.1, ...)`
- Add a brief delay (`C_Timer.After(0.5, Screenshot)`) to let the game render the moment properly
- Log each screenshot in `SavedVariables`: `{event, zone, timestamp, mapID}`
- Provide a `/screenshots` command showing the log with filters by event type
- Settings panel for toggling which events trigger screenshots and whether to hide UI
- Throttle screenshots to max 1 per 5 seconds to prevent spam during rapid events

```lua
-- Smart screenshot with UI hide
local function SmartScreenshot(eventType)
    if ns.db.hideUI then
        UIParent:SetAlpha(0)
        C_Timer.After(0.1, function()
            Screenshot()
            C_Timer.After(0.1, function()
                UIParent:SetAlpha(1)
            end)
        end)
    else
        Screenshot()
    end
    -- Log metadata
    local map = C_Map.GetBestMapForUnit("player")
    table.insert(ns.db.log, {
        event = eventType,
        mapID = map,
        time = time(),
    })
end
```

!!! tip "Build This"

    ```
    /wow-create ScreenShotter — Smart screenshot automator that captures
    on achievements, level-ups, boss kills, mount drops, with UI-hide
    option, metadata logging, throttling, Blizzard Faithful mode
    ```

---

## What's Next?

These twelve ideas barely scratch the surface. Midnight's 437 new APIs open up entire categories of addons that were impossible before — housing automation, encounter timeline tools, built-in damage meter integrations, and photo-sharing social features.

Ready to start building?

- **[Getting Started](../getting-started.md)** — Set up your development environment and build your first addon in 60 seconds
- **[Starter Template](../starter-template.md)** — The production-ready template with all the patterns you need
- **[Midnight Coding Patterns](../midnight-patterns.md)** — Secret Values, Duration objects, and the APIs that matter
- **[Building Better Addons](../better-addons.md)** — The "enhance, don't replace" philosophy in depth
- **[Code Templates](../code-templates.md)** — Copy-paste patterns for common addon features

Every idea on this page can be scaffolded with a single `/wow-create` command. Pick one, build it, and ship it. The Midnight addon ecosystem is wide open.
