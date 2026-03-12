---
title: "Blizzard UI Systems Deep Dive"
description: "Complete reference for Blizzard's built-in UI systems in WoW Midnight (Patch 12.0+) — Cooldown Manager, Edit Mode, Settings Panel, ScrollBox, Action Bars, Secret Values, and more."
---

# Blizzard UI Systems Deep Dive

Blizzard ships dozens of UI systems that addon developers can hook, extend, or integrate with. Midnight (12.0) didn't replace these systems — it *hardened* them with Secret Values and consolidated APIs into namespaces. Understanding what Blizzard provides is the first step to building addons that cooperate with the game instead of fighting it.

This page covers every major Blizzard UI system relevant to addon development, with real API signatures, working code examples, and the pitfalls that will waste your afternoon if you don't know about them.

!!! tip "Prerequisites"
    This page assumes you've read [Coding for Midnight](midnight-patterns.md) for the philosophy of hooking vs. replacing, and [Security Model](security.md) for taint fundamentals. Here we focus on *what Blizzard provides* and *how to interact with it*.

---

## Cooldown Manager

Blizzard's built-in Cooldown Manager (also called "Essential Cooldowns") was added in Patch 11.1.5 and significantly expanded through 11.2.5. In Midnight, cooldown data flows through Secret Values — making this system critical to understand.

### The Cooldown Widget

The `Cooldown` frame type renders clock-sweep animations. Every action button, buff icon, and item slot uses one.

```lua
-- Basic cooldown display
local cooldown = CreateFrame("Cooldown", nil, parentFrame, "CooldownFrameTemplate")
cooldown:SetAllPoints(parentFrame)

-- Pre-12.0 way (still works outside restricted contexts)
cooldown:SetCooldown(startTime, duration)

-- Read back cooldown times
local start, duration = cooldown:GetCooldownTimes()
```

### Midnight 12.0: Duration Objects

The old `SetCooldown(start, duration)` breaks when those values are Secret Values. Blizzard's solution: **Duration Objects** — opaque containers that carry cooldown timing without exposing the numbers.

```lua
-- Create a reusable Duration Object
local duration = C_DurationUtil.CreateDuration()

-- Three ways to populate it:
duration:SetTimeSpan(startTime, endTime)
duration:SetTimeFromStart(startTime, durationSecs)
duration:SetTimeFromEnd(endTime, durationSecs)

-- Pass it straight to the cooldown widget — no math needed
cooldownFrame:SetCooldownFromDurationObject(duration)
```

New `Cooldown` widget methods in 12.0:

| Method | Purpose |
|--------|---------|
| `SetCooldownFromDurationObject(duration [, clearIfZero])` | Secret Values-compatible cooldown display |
| `SetCooldownFromExpirationTime(expTime, dur [, modRate])` | Expiration-based alternative |
| `SetPaused(paused)` | Pause/resume the sweep animation |
| `GetCountdownFontString()` | Access the countdown text overlay |

### Spell Cooldown APIs

```lua
-- 11.0+ standard API (returns SpellCooldownInfo table)
local info = C_Spell.GetSpellCooldown(spellID)
-- info.startTime, info.duration, info.isEnabled, info.modRate

-- GCD check: use spell ID 61304
local gcdInfo = C_Spell.GetSpellCooldown(61304)
```

New 12.0 Duration-returning APIs (return Secret Values in restricted contexts):

| Function | Purpose |
|----------|---------|
| `C_Spell.GetSpellCooldownDuration()` | Spell cooldown as Duration Object |
| `C_Spell.GetSpellChargeDuration()` | Charge cooldown as Duration Object |
| `C_Spell.GetSpellLossOfControlCooldownDuration()` | LoC cooldown as Duration Object |
| `C_SpellBook.GetSpellBookItemCooldownDuration()` | Spellbook item cooldown |

### The Built-in Cooldown Viewer

Blizzard's own cooldown tracker (the Edit Mode "Cooldown Viewer" frame):

- **Disabled by default** — players enable it in Options > Advanced Options
- Requires level 10+
- Categories: Essential, Utility, Hidden by Default (for cooldowns), Icons/Bars/Hidden (for buffs)
- Positioned via Edit Mode using `EditModeCooldownViewerSystemMixin`
- **11.2.5+:** Advanced drag-and-drop configuration, per-spec display, category reassignment

!!! warning "Don't duplicate what Blizzard already provides"
    Before building a custom cooldown tracker, check whether the built-in Cooldown Viewer already does what you need. Many players now use it, and addons that duplicate its functionality create visual clutter. Consider *supplementing* it instead — add cooldowns Blizzard doesn't track, or provide a different layout.

### Complete Cooldown Pattern (12.0+)

```lua
local addonName, ns = ...

-- Reusable Duration Object (avoid creating per-frame)
local durationObj = C_DurationUtil.CreateDuration()

local function UpdateCooldownDisplay(cooldownFrame, spellID)
    -- Try the Duration API first (Secret Values-safe)
    local duration = C_Spell.GetSpellCooldownDuration(spellID)
    if duration then
        cooldownFrame:SetCooldownFromDurationObject(duration)
        return
    end

    -- Fallback for non-restricted contexts
    local info = C_Spell.GetSpellCooldown(spellID)
    if info and info.duration > 0 then
        cooldownFrame:SetCooldown(info.startTime, info.duration, info.modRate)
    else
        cooldownFrame:Clear()
    end
end
```

!!! danger "ActionButton cooldown split"
    In 12.0, `ActionButtonTemplate` split its single cooldown frame into **three** children: `cooldown`, `chargeCooldown`, and `lossOfControlCooldown`. If your addon skins action buttons by accessing the cooldown child, update your code to handle all three. The global function `ActionButton_ApplyCooldown()` handles this automatically using secure delegates.

---

## Edit Mode System

Edit Mode (added in 10.0) lets players rearrange HUD elements by dragging them. It's the system behind the grid-snapping, layout-saving UI that appears when you press Escape > Edit Mode.

### Core Architecture

```
EditModeManagerFrame (DIALOG strata, global singleton)
    ├── Registered system frames (action bars, minimap, chat, etc.)
    ├── EditModeSystemSettingsDialog (per-system settings panel)
    └── Layout storage (C_EditMode namespace)
```

### C_EditMode Namespace

| Function | Purpose |
|----------|---------|
| `C_EditMode.GetLayouts()` | Get all saved layouts |
| `C_EditMode.SaveLayouts(layoutInfo)` | Persist layout changes |
| `C_EditMode.SetActiveLayout(index)` | Switch active layout |
| `C_EditMode.GetAccountSettings()` | Account-wide settings |
| `C_EditMode.SetAccountSetting(setting, value)` | Set account-wide setting |
| `C_EditMode.ConvertLayoutInfoToString()` | Serialize layout for sharing |

### Events

| Event | Payload |
|-------|---------|
| `EDIT_MODE_LAYOUTS_UPDATED` | `layoutInfo`, `reconcileLayouts` |
| `EventRegistry "EditMode.Enter"` | — |
| `EventRegistry "EditMode.Exit"` | — |
| `EventRegistry "EditMode.SavedLayouts"` | — |

### Detecting Edit Mode State

```lua
-- React when Edit Mode opens/closes
EventRegistry:RegisterCallback("EditMode.Enter", function()
    -- Player entered Edit Mode — show configuration UI, unlock frames, etc.
    MyAddon:EnterEditMode()
end)

EventRegistry:RegisterCallback("EditMode.Exit", function()
    -- Player left Edit Mode — lock frames, save positions
    MyAddon:ExitEditMode()
end)
```

### System Mixins

Every draggable element in Edit Mode uses a specialized mixin inheriting from `EditModeSystemMixin`:

| Mixin | System |
|-------|--------|
| `EditModeActionBarSystemMixin` | Action Bars 1–8 |
| `EditModeCastBarSystemMixin` | Cast Bar |
| `EditModeMinimapSystemMixin` | Minimap |
| `EditModeChatFrameSystemMixin` | Chat Frame |
| `EditModeUnitFrameSystemMixin` | Unit Frames |
| `EditModeAuraFrameSystemMixin` | Buffs/Debuffs |
| `EditModeBagsSystemMixin` | Bags |
| `EditModeMicroMenuSystemMixin` | Micro Menu |
| `EditModeObjectiveTrackerSystemMixin` | Objective Tracker |
| `EditModeCooldownViewerSystemMixin` | Cooldown Viewer |

Settings are controlled by: `EditModeSettingDropdownMixin`, `EditModeSettingSliderMixin`, `EditModeSettingCheckboxMixin`.

### Registering Custom Frames (The Taint Problem)

Blizzard provides **no official API** for addons to register custom frames with Edit Mode. Calling `EditModeManagerFrame:RegisterSystemFrame()` from addon code introduces taint and can break the entire Edit Mode UI.

!!! danger "Do NOT call `RegisterSystemFrame()` directly"
    ```lua
    -- THIS WILL TAINT EDIT MODE:
    EditModeManagerFrame:RegisterSystemFrame(myFrame)
    ```
    This is an internal Blizzard method. Use a community library instead.

**Community libraries** that solve this problem:

| Library | Approach | Usage |
|---------|----------|-------|
| **LibEditMode** (p3lim) | Parallel selection/dialog system alongside Blizzard's | `lib:AddFrame(frame, callback, default, name)` |
| **EditModeExpanded** (teelolws) | Extensive feature registration with full settings | `lib:RegisterFrame(frame, name, db)` |
| **LibEditModeOverride** (plusmouse) | Programmatic layout manipulation | Uses `C_EditMode.GetLayouts()` / `SaveLayouts()` |

### Source Files

All in `Interface/AddOns/Blizzard_EditMode/Shared/`:

| File | Purpose |
|------|---------|
| `EditModeManager.lua` | Frame registration, layout management |
| `EditModeSystemTemplates.lua` | All system mixins |
| `EditModeDialogs.lua` | Settings dialog |
| `EditModeSettingDisplayInfo.lua` | System-to-setting mapping |
| `EditModeUtil.lua` | Magnetism/snapping logic |

---

## Settings Panel API

The modern Settings Panel (added in 10.0, breaking changes in 11.0.2) replaces the old `InterfaceOptions_AddCategory` system. Your addon's settings appear in the game's Settings > AddOns tab.

### Category Registration

```lua
-- Vertical layout: auto-stacking controls (recommended for most addons)
local category = Settings.RegisterVerticalLayoutCategory("My AddOn")

-- Subcategory under a parent
local sub = Settings.RegisterVerticalLayoutSubcategory(category, "Advanced")

-- Canvas layout: fully custom frame
local frame = CreateFrame("Frame")
local category = Settings.RegisterCanvasLayoutCategory(frame, "My AddOn")

-- Register in the AddOns tab
Settings.RegisterAddOnCategory(category)
```

### Setting Registration (11.0.2+ Signatures)

!!! warning "11.0.2 broke the old signatures"
    Patch 11.0.2 added `variableKey` and `variableTbl` parameters to `RegisterAddOnSetting`. Code written for 10.0–11.0.1 will error. The signatures below are current through 12.0.1.

| Function | Purpose |
|----------|---------|
| `Settings.RegisterAddOnSetting(category, variable, variableKey, variableTbl, variableType, name, defaultValue)` | Direct table read/write |
| `Settings.RegisterProxySetting(category, variable, variableType, name, defaultValue, getValue, setValue)` | Custom getter/setter |
| `Settings.RegisterCVarSetting(category, variable, variableType, name)` | Bind to a CVar |

### Control Creation

| Function | Purpose |
|----------|---------|
| `Settings.CreateCheckbox(category, setting, tooltip)` | Boolean toggle |
| `Settings.CreateSlider(category, setting, options, tooltip)` | Numeric slider |
| `Settings.CreateSliderOptions(min, max, step)` | Slider range config |
| `Settings.CreateDropdown(category, setting, getOptions, tooltip)` | Dropdown selector |
| `Settings.CreateControlTextContainer()` | Build dropdown options |

### Complete Working Example

```lua
-- ## SavedVariables: MyAddOn_DB
MyAddOn_DB = {}  -- populated by WoW from SavedVariables

local category = Settings.RegisterVerticalLayoutCategory("My AddOn")

-- Checkbox
do
    local setting = Settings.RegisterAddOnSetting(category,
        "MyAddOn_Toggle",  -- unique variable name
        "toggle",          -- key in SavedVariables table
        MyAddOn_DB,        -- the SavedVariables table
        type(false),       -- variableType ("boolean")
        "Enable Feature",  -- display name
        false              -- default value
    )
    Settings.CreateCheckbox(category, setting, "Enables the main feature.")
end

-- Slider
do
    local setting = Settings.RegisterAddOnSetting(category,
        "MyAddOn_Scale", "scale", MyAddOn_DB,
        type(1), "UI Scale", 100
    )
    local options = Settings.CreateSliderOptions(50, 200, 5)
    options:SetLabelFormatter(MinimalSliderWithSteppersMixin.Label.Right)
    Settings.CreateSlider(category, setting, options, "Adjust scale percentage.")
end

-- Dropdown
do
    local function GetOptions()
        local container = Settings.CreateControlTextContainer()
        container:Add(1, "Small")
        container:Add(2, "Medium")
        container:Add(3, "Large")
        return container:GetData()
    end
    local setting = Settings.RegisterAddOnSetting(category,
        "MyAddOn_Size", "size", MyAddOn_DB,
        type(1), "Frame Size", 2
    )
    Settings.CreateDropdown(category, setting, GetOptions, "Select frame size.")
end

Settings.RegisterAddOnCategory(category)
```

### Reading and Writing Settings

```lua
-- Read
local value = Settings.GetValue("MyAddOn_Toggle")

-- Write
Settings.SetValue("MyAddOn_Toggle", true)

-- Get the setting object for callbacks
local setting = Settings.GetSetting("MyAddOn_Toggle")
setting:SetValueChangedCallback(function(setting, value)
    print("Toggle changed to:", value)
end)

-- Open the settings panel to your category
Settings.OpenToCategory(category:GetID())
```

### Canvas Layout (Custom Frame)

For complex settings UIs that don't fit the vertical layout:

```lua
local frame = CreateFrame("Frame")
frame:SetSize(600, 400)

-- These three functions are REQUIRED for Canvas layouts
function frame.OnCommit()
    -- Called when the player clicks "Okay" or "Apply"
    -- Save your settings here
end

function frame.OnDefault()
    -- Called when the player clicks "Defaults"
    -- Reset your settings here
end

function frame.OnRefresh()
    -- Called when the panel is shown
    -- Update your controls to match current settings
end

local category = Settings.RegisterCanvasLayoutCategory(frame, "My AddOn")
Settings.RegisterAddOnCategory(category)
```

### Migration from Pre-10.0

| Old System | New System |
|------------|------------|
| `InterfaceOptions_AddCategory(panel)` | `Settings.RegisterAddOnCategory(category)` |
| `InterfaceOptionsFrame_OpenToCategory(name)` | `Settings.OpenToCategory(category:GetID())` |
| `InterfaceOptionsCheckButtonTemplate` | `Settings.CreateCheckbox()` |
| `OptionsSliderTemplate` | `Settings.CreateSlider()` |
| `UIDropDownMenu_Initialize` | `Settings.CreateDropdown()` |

---

## Addon Compartment

The Addon Compartment (added in 10.0, TOC registration in 10.1) is a unified dropdown on the minimap that replaces the old pattern of each addon creating its own minimap button. It's the small addon icon button next to the minimap.

### TOC Registration (Recommended)

The simplest approach — no Lua code required for basic presence:

```toc
## AddonCompartmentFunc: MyAddon_OnCompartmentClick
## AddonCompartmentFuncOnEnter: MyAddon_OnCompartmentEnter
## AddonCompartmentFuncOnLeave: MyAddon_OnCompartmentLeave
## IconTexture: Interface\Icons\INV_Misc_QuestionMark
```

Only `AddonCompartmentFunc` is required. The callback functions **must be global**.

### Callback Signatures

```lua
-- Click handler (TOC-registered)
function MyAddon_OnCompartmentClick(addonName, buttonName)
    if buttonName == "LeftButton" then
        Settings.OpenToCategory(MyAddon.categoryID)
    elseif buttonName == "RightButton" then
        MyAddon:ToggleMainWindow()
    end
end

-- Tooltip on hover
function MyAddon_OnCompartmentEnter(addonName, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("My Addon")
    GameTooltip:AddLine("Left-click: Settings", 1, 1, 1)
    GameTooltip:AddLine("Right-click: Toggle window", 1, 1, 1)
    GameTooltip:Show()
end

function MyAddon_OnCompartmentLeave(addonName, menuButtonFrame)
    GameTooltip:Hide()
end
```

### Programmatic Registration

For addons that need to register at runtime:

```lua
AddonCompartmentFrame:RegisterAddon({
    text = "My Addon",
    icon = "Interface\\Icons\\INV_Misc_QuestionMark",
    func = function(button, menuInputData, menu)
        local mouseButton = menuInputData.buttonName
        -- Handle click
    end,
    funcOnEnter = function(button, menuInputData, menu)
        GameTooltip:SetOwner(button, "ANCHOR_LEFT")
        GameTooltip:SetText("My Addon")
        GameTooltip:Show()
    end,
    funcOnLeave = function(button, menuInputData, menu)
        GameTooltip:Hide()
    end,
    notCheckable = true,
})
```

!!! warning "Compartment pitfalls"
    - `IconTexture` takes priority over `IconAtlas` if both are set
    - There is **no** `UnregisterAddon` API — once registered, it's permanent until reload
    - The dropdown does not live-update while open
    - In 11.0.0, TOC-based registrations lost the `menuButtonFrame` parameter in click handlers; programmatic registrations receive a table as the second argument instead
    - No compartment-specific changes in 12.0

---

## ScrollBox & DataProvider

The ScrollBox/DataProvider system (added in 10.0) replaced `FauxScrollFrame` and `HybridScrollFrame`. It's data-driven and pool-based — you provide data, it handles virtualization and recycling.

### Architecture

```
DataProvider (data store)
    │
    ▼
ScrollView (layout + element factory)
    │
    ▼
ScrollBox (viewport frame)  ◄──►  ScrollBar (scroll control)
```

| Component | How to Create |
|-----------|---------------|
| ScrollBox | `CreateFrame("Frame", nil, parent, "WowScrollBoxList")` |
| ScrollBar | `CreateFrame("EventFrame", nil, parent, "MinimalScrollBar")` |
| ScrollView | `CreateScrollBoxListLinearView()` |
| DataProvider | `CreateDataProvider()` |

### Complete Setup

```lua
-- 1. Create the visual components
local scrollBox = CreateFrame("Frame", nil, parentFrame, "WowScrollBoxList")
scrollBox:SetPoint("TOPLEFT", 4, -4)
scrollBox:SetPoint("BOTTOMRIGHT", -24, 4)

local scrollBar = CreateFrame("EventFrame", nil, parentFrame, "MinimalScrollBar")
scrollBar:SetPoint("TOPLEFT", scrollBox, "TOPRIGHT", 4, 0)
scrollBar:SetPoint("BOTTOMLEFT", scrollBox, "BOTTOMRIGHT", 4, 0)

-- 2. Configure the ScrollView with fixed row height
local scrollView = CreateScrollBoxListLinearView()
scrollView:SetElementExtent(24)

-- 3. Define how each row renders
scrollView:SetElementInitializer("UIPanelButtonTemplate", function(button, data)
    button:SetText(data.name)
    button:SetScript("OnClick", function()
        print("Clicked:", data.name)
    end)
end)

-- 4. Wire everything together
ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, scrollView)

-- 5. Populate with data
local dataProvider = CreateDataProvider()
for i = 1, 100 do
    dataProvider:Insert({ name = "Item " .. i })
end
scrollBox:SetDataProvider(dataProvider)
```

### Element Initializer and Resetter

The initializer runs when a row becomes visible. The resetter runs when it scrolls out of view — use it to clean up scripts and references.

```lua
scrollView:SetElementInitializer("MyItemTemplate", function(frame, data)
    frame.Name:SetText(data.name)
    frame.Icon:SetTexture(data.icon)
    frame:SetScript("OnEnter", function(self)
        GameTooltip:SetOwner(self, "ANCHOR_RIGHT")
        GameTooltip:SetText(data.tooltip)
        GameTooltip:Show()
    end)
    frame:SetScript("OnLeave", GameTooltip_Hide)
end)

scrollView:SetElementResetter(function(frame, data)
    frame:SetScript("OnEnter", nil)
    frame:SetScript("OnLeave", nil)
end)
```

### Variable-Height Rows

```lua
scrollView:SetElementExtentCalculator(function(dataIndex, data)
    if data.isHeader then return 32 end
    return 20
end)
```

### Multiple Templates (Element Factory)

```lua
scrollView:SetElementFactory(function(factory, node)
    local data = node:GetData()
    factory(data.Template, function(frame, node)
        frame:Init(node)
    end)
end)
```

### DataProvider Operations

```lua
dataProvider:Insert({ name = "New Item" })
dataProvider:InsertTable({ {name = "A"}, {name = "B"} })
dataProvider:RemoveIndex(3)
dataProvider:RemoveByPredicate(function(d) return d.name == "B" end)
dataProvider:Flush()  -- clear all data
dataProvider:SetSortComparator(function(a, b) return a.name < b.name end)
dataProvider:Sort()
dataProvider:GetSize()
dataProvider:ForEach(function(data) print(data.name) end)
dataProvider:FindByPredicate(function(d) return d.name == "A" end)
```

### Retaining Scroll Position

```lua
-- Keeps the scroll position when replacing data
scrollBox:SetDataProvider(dataProvider, ScrollBoxConstants.RetainScrollPosition)
```

### Selection Behavior

```lua
local selectionBehavior = ScrollUtil.AddSelectionBehavior(scrollBox)

selectionBehavior:RegisterCallback(
    SelectionBehaviorMixin.Event.OnSelectionChanged,
    function(self, elementData, selected)
        local button = scrollBox:FindFrame(elementData)
        if button then
            if selected then
                button:SetText("[" .. elementData.name .. "]")
            else
                button:SetText(elementData.name)
            end
        end
    end
)
```

### Search / Filter Pattern

```lua
local AllData = {}
for i = 1, 100 do
    tinsert(AllData, { name = "Item " .. i })
end

local dataProvider = CreateDataProvider(AllData)
scrollBox:SetDataProvider(dataProvider)

searchBox:HookScript("OnTextChanged", function(editBox)
    local query = editBox:GetText():lower()
    local filtered = {}
    for _, entry in ipairs(AllData) do
        if entry.name:lower():find(query, 1, true) then
            tinsert(filtered, entry)
        end
    end
    scrollBox:SetDataProvider(CreateDataProvider(filtered))
end)
```

### Tree DataProvider (Hierarchical Lists)

```lua
local treeView = CreateScrollBoxListTreeListView()
local treeDataProvider = CreateTreeDataProvider()

treeView:SetElementInitializer("UIPanelButtonTemplate", function(button, node)
    local data = node:GetData()
    local depth = node:GetDepth()
    button:SetText(string.rep("  ", depth) .. data.name)
    button:SetScript("OnClick", function() node:ToggleCollapsed() end)
end)

ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, treeView)
scrollBox:SetDataProvider(treeDataProvider)

local parent = treeDataProvider:Insert({ name = "Category A" })
parent:Insert({ name = "Sub-item 1" })
parent:Insert({ name = "Sub-item 2" })
```

### Hooking Existing Blizzard ScrollBoxes

```lua
-- React when Blizzard populates a ScrollBox
hooksecurefunc(someBlizzardScrollBox, "SetDataProvider", function(self, dp)
    -- Hook the DataProvider to track insertions
    hooksecurefunc(dp, "Insert", function(dataProvider, data)
        -- React to new data entries
    end)
end)
```

!!! tip "Managed ScrollBar visibility"
    Hide the scrollbar when content fits without scrolling:
    ```lua
    ScrollUtil.AddManagedScrollBarVisibilityBehavior(
        scrollBox, scrollBar,
        -- Anchors when scrollbar is visible:
        { CreateAnchor("TOPLEFT", 4, -4), CreateAnchor("BOTTOMRIGHT", scrollBar, -13, 4) },
        -- Anchors when scrollbar is hidden:
        { CreateAnchor("TOPLEFT", 4, -4), CreateAnchor("BOTTOMRIGHT", -4, 4) }
    )
    ```

---

## Action Bar System

Midnight's biggest namespace consolidation. All action bar functions moved into `C_ActionBar` — over 45 functions replacing scattered globals.

### Frame Names and Slot Numbers

| Bar | Frame Name | Action Slots |
|-----|-----------|-------------|
| Action Bar 1 (page 1) | `ActionButton[1-12]` | 1–12 |
| Action Bar 1 (page 2) | `ActionButton[1-12]` | 13–24 |
| Action Bar 4 | `MultiBarRightButton[1-12]` | 25–36 |
| Action Bar 5 | `MultiBarLeftButton[1-12]` | 37–48 |
| Action Bar 3 | `MultiBarBottomRightButton[1-12]` | 49–60 |
| Action Bar 2 | `MultiBarBottomLeftButton[1-12]` | 61–72 |
| Bars 6–8 (added 10.0) | — | 145–180 |

Slots 73–120 are class stance bars. Slots 121–132 are the possession bar.

### C_ActionBar Namespace (12.0)

Legacy globals like `ActionHasRange()`, `GetActionAutocast()`, and `IsItemAction()` were **removed**. Use the namespace equivalents:

| Function | Purpose |
|----------|---------|
| `C_ActionBar.GetActionCooldown()` | Cooldown info (Secret Values in combat) |
| `C_ActionBar.GetActionCharges()` | Charge mechanics |
| `C_ActionBar.GetActionDisplayCount()` | Visual counts |
| `C_ActionBar.IsActionInRange()` | Range validation |
| `C_ActionBar.GetActionBarPage()` | Current bar page |
| `C_ActionBar.SetActionBarPage()` | Switch bar page |
| `C_ActionBar.GetActionTexture()` | Action icon |
| `C_ActionBar.GetActionText()` | Action text |
| `C_ActionBar.IsUsableAction()` | Usability check |
| `C_ActionBar.IsAttackAction()` | Auto-attack check |
| `C_ActionBar.IsAutoRepeatAction()` | Auto-repeat check |
| `C_ActionBar.RegisterActionUIButton()` | Register button (replaces `SetActionUIButton()`) |
| `C_ActionBar.HasBonusActionBar()` | Bonus bar check |
| `C_ActionBar.HasVehicleActionBar()` | Vehicle bar check |

### ActionButton Cooldown Split (12.0)

The single cooldown frame on `ActionButtonTemplate` was split into three:

| Child Frame | Purpose |
|-------------|---------|
| `cooldown` | Standard ability cooldown |
| `chargeCooldown` | Charge-based cooldown |
| `lossOfControlCooldown` | Loss of control overlay |

```lua
-- Old (pre-12.0): one cooldown frame
local cd = actionButton.cooldown

-- New (12.0): three cooldown frames
local cd     = actionButton.cooldown
local charge = actionButton.chargeCooldown
local loc    = actionButton.lossOfControlCooldown
```

The global function `ActionButton_ApplyCooldown()` handles all three using secure delegates for Secret Values.

### SecureActionButtonTemplate

Still functional in 12.0 with additions. Supports 20+ action types: `spell`, `item`, `macro`, `action`, `pet`, `target`, `focus`, `assist`, `cancelaura`, and more.

**12.0 additions:**

- `SetRaidTarget()` now usable via the template
- New secure action delegates for housing APIs (`VisitHouse`, `TeleportHome`, `ReturnAfterVisitingHouse`)
- New `raidTarget` delegate for setting/clearing/toggling raid targets

!!! tip "Migrating action bar addons"
    The migration is mostly mechanical: replace each removed global with its `C_ActionBar` equivalent. The [Patch 12.0.0 API changes](https://warcraft.wiki.gg/wiki/Patch_12.0.0/API_changes) page on Warcraft Wiki has a complete mapping. Most action bar addons (Bartender4, Dominos, etc.) needed minimal changes.

---

## Secret Values System

The defining change of Midnight. Secret Values are opaque containers that addon code cannot inspect, compare, or do arithmetic on. They exist to prevent automation addons from reacting to combat data faster than humans can.

### When Secrets Apply

Secret Values are enforced during:

- **Mythic keystone runs**
- **PvP matches**
- **Instance encounters** (boss fights)
- **Player combat** (general)

Outside these contexts, values are accessible normally.

### What Returns Secrets

| Data | Status |
|------|--------|
| Spell cooldown info (`C_ActionBar.GetActionCooldown()`, `C_Spell.GetSpellCooldown()`) | Secret in restricted contexts |
| Enemy unit names, GUIDs, and IDs | Secret in instances (regardless of combat) |
| Aura data contents | Secret (but AuraInstanceIDs themselves are NOT secret) |
| Combat log events | **Removed entirely** from addon access |

### What Does NOT Return Secrets

| Data | Reason |
|------|--------|
| Player secondary resources (Combo Points, Runes, Soul Shards, Holy Power, Chi, Arcane Charges, Essence) | Explicitly exempted |
| Profession and Skyriding spells | Flagged "never secret" |
| `UnitIsUnit()` comparisons with target/focus/mouseover/softenemy/softinteract/softfriend | Allowed |
| Specific auras: Maelstrom Weapon, Ebon Might | Explicitly exempted |

### Detection APIs

```lua
-- Check if a specific value is secret
if issecretvalue(someValue) then
    -- Can't read this right now — pass it to a widget or bail
    return
end

-- Check if restrictions are currently active
if C_Secrets.HasSecretRestrictions() then
    -- We're in restricted content
end

-- Specific restriction checks
C_Secrets.ShouldSpellCooldownBeSecret()    -- cooldown restriction
C_Secrets.ShouldAurasBeSecret()            -- aura restriction
C_Secrets.ShouldUnitAuraInstanceBeSecret() -- unit aura restriction
```

### Testing Secret Values

Use these CVars to force Secret Values outside restricted content (useful for development):

```
/console secretAurasForced 1
/console secretCooldownsForced 1
/console secretUnitIdentityForced 1
/console secretSpellcastsForced 1
```

Set to `0` to disable.

### Widget Support

Widgets can detect and handle Secret Values:

```lua
-- Check if a widget contains secret values
frame:HasSecretValues()
frame:HasSecretAspect()

-- Prevent a widget from accepting secret values (will error instead)
frame:SetPreventSecretValues()
```

String operations (`string.format()`, `..`, `string.join()`) all work with Secrets — they produce secret output. You can concatenate secret values into strings for display, but you can't read the result back in Lua.

### The Pattern: Pass Through, Don't Inspect

```lua
-- WRONG: trying to read and branch on secret values
local info = C_Spell.GetSpellCooldown(spellID)
if info.duration > 0 then  -- ERROR: can't compare secret value
    cooldown:SetCooldown(info.startTime, info.duration)
end

-- RIGHT: use Duration Objects that pass through to widgets
local duration = C_Spell.GetSpellCooldownDuration(spellID)
if duration then
    cooldown:SetCooldownFromDurationObject(duration)
end
```

### Boolean-Driven Visuals (12.0)

New widget methods for showing/hiding based on Secret Values without branching:

```lua
-- Instead of: if someSecretBool then region:SetAlpha(1) else region:SetAlpha(0) end
region:SetAlphaFromBoolean(someSecretBool)

-- Instead of: if someSecretBool then region:SetVertexColor(1,0,0) else ... end
region:SetVertexColorFromBoolean(someSecretColor)
```

!!! danger "Combat Log is gone"
    `COMBAT_LOG_EVENT_UNFILTERED` is no longer available to addons in 12.0. The new `C_CombatLog` namespace provides `IsCombatLogRestricted()` and filter management, but raw combat log parsing is over. Addons like Details! and Recount must use `C_DamageMeter` for session data or the new `C_EncounterTimeline` for mechanic tracking.

---

## Finding Blizzard Source Code

You can't effectively hook Blizzard systems without reading their source code. Here's how to find it.

### Exporting Interface Files

At the login/character select screen, open the console and run:

```
/console ExportInterfaceFiles code
/console ExportInterfaceFiles art
```

This creates `BlizzardInterfaceCode/` and `BlizzardInterfaceArt/` in your WoW install directory.

```
BlizzardInterfaceCode/
  Interface/
    SharedXML/           -- Shared utilities (math, tables, colors, events)
    FrameXML/            -- Core UI framework (UIParent, templates)
    AddOns/
      Blizzard_AchievementUI/
      Blizzard_ActionBar/
      Blizzard_AuctionHouseUI/
      Blizzard_APIDocumentationGenerated/
      ... (dozens of Blizzard_ addons)
```

### GitHub Mirrors

| Repository | Description |
|-----------|-------------|
| [Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source) | Canonical community mirror (`live`, `ptr`, `beta` branches) |
| [Gethe/wow-ui-textures](https://github.com/Gethe/wow-ui-textures) | UI texture assets |
| [Ketho/wow-ui-source-midnight](https://github.com/Ketho/wow-ui-source-midnight) | Midnight Beta FrameXML |
| [Ketho/BlizzardInterfaceResources](https://github.com/Ketho/BlizzardInterfaceResources) | API dumps, enums, templates, mixins, atlas info |

### Online Browsers

| Tool | URL | Description |
|------|-----|-------------|
| Townlong Yak | [townlong-yak.com/framexml/](https://www.townlong-yak.com/framexml/) | Browse and diff FrameXML across builds |
| Wago.tools | [wago.tools](https://wago.tools/) | Data mining, DB2 tables, file browser |

### Documentation

| Resource | URL |
|----------|-----|
| World of Warcraft API | [warcraft.wiki.gg/wiki/World_of_Warcraft_API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API) |
| Widget API | [warcraft.wiki.gg/wiki/Widget_API](https://warcraft.wiki.gg/wiki/Widget_API) |
| Events | [warcraft.wiki.gg/wiki/Events](https://warcraft.wiki.gg/wiki/Events) |
| FrameXML Functions | [warcraft.wiki.gg/wiki/FrameXML_functions](https://warcraft.wiki.gg/wiki/FrameXML_functions) |
| ScrollBox Guide | [warcraft.wiki.gg/wiki/Making_scrollable_frames](https://warcraft.wiki.gg/wiki/Making_scrollable_frames) |

### In-Game Debugging

| Command | Purpose |
|---------|---------|
| `/dump <expr>` | Print any Lua expression (tables, frames, API return values) |
| `/eventtrace` | Real-time event viewer with filtering |
| `/framestack` | Show all frames under mouse cursor with hierarchy |

### Key Files to Know

| File | Purpose |
|------|---------|
| `FrameXML/FrameXML.toc` | Master load order |
| `FrameXML/UIParent.lua` | Root UIParent frame, core events |
| `SharedXML/Mixin.lua` | `Mixin()`, `CreateFromMixins()` |
| `SharedXML/TableUtil.lua` | Table utilities |
| `SharedXML/EventRegistry.lua` | Callback-based event system |
| `SharedXML/PixelUtil.lua` | Pixel-perfect positioning |
| `AddOns/Blizzard_APIDocumentationGenerated/` | Generated API docs with signatures |

### How to Trace a Function

1. Use `/framestack` to identify the frame under your cursor
2. Search the [wow-ui-source](https://github.com/Gethe/wow-ui-source) repo for that frame name — find its XML definition and Lua scripts
3. Follow the `inherits` chain in XML for template hierarchy
4. Follow the `Mixin` chain in Lua — `CreateFromMixins(SomeMixin)`
5. Check `RegisterEvent` / `EventRegistry:RegisterCallback` for events the frame listens to
6. Use `/dump SomeFrame:GetPoint(1)` to inspect live positioning

### IDE Support

[Ketho's WoW API VS Code extension](https://marketplace.visualstudio.com/items?itemName=ketho.wow-api) provides IntelliSense via LuaLS. Auto-activates when it detects a `.toc` file in your project.

### Essential Development Addons

| Addon | Purpose |
|-------|---------|
| BugSack + BugGrabber | Captures Lua errors with full stack traces |
| TextureAtlasViewer | Browse atlas names and coordinates |
| TextureViewer | Display any texture by file path |

---

## Interface Version Numbers

Your TOC file's `## Interface:` line determines compatibility. Addons without `120000` or higher **will not load** without a player override.

| Expansion | Patch | Interface |
|-----------|-------|-----------|
| Dragonflight | 10.0.0 | `100000` |
| The War Within | 11.0.0 | `110000` |
| Midnight | 12.0.0 | `120000` |
| Midnight | 12.0.1 | `120001` |

```toc
## Interface: 120001
## Title: My Addon
## Notes: Does cool things
## Author: Me
```
