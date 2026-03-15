# API Quick-Reference Cheat Sheet

This page exists because **AI coding assistants hallucinate WoW API function signatures constantly.** Every signature here is verified against the 12.0.1 (Midnight) API. When in doubt, trust this page over AI-generated code.

!!! danger "Midnight Changed Everything"
    Patch 12.0 deprecated dozens of global functions and moved them to C_ namespaces. AI models trained on pre-12.0 code **will** generate deprecated calls. This cheat sheet marks every change.

**Canonical reference:** [Widget API — Warcraft Wiki](https://warcraft.wiki.gg/wiki/Widget_API) | [World of Warcraft API](https://warcraft.wiki.gg/wiki/World_of_Warcraft_API)

---

## Frame Creation & Management

```lua
CreateFrame(frameType [, name, parent, template, id]) -> frame
-- frameType: "Frame", "Button", "CheckButton", "EditBox", "ScrollFrame",
--            "Slider", "StatusBar", "Cooldown", "GameTooltip", "ModelScene", etc.
-- name:     Global name (creates _G["name"]). Use nil for anonymous frames.
-- parent:   Parent frame. Usually UIParent.
-- template: Comma-separated string, e.g. "BackdropTemplate,SecureActionButtonTemplate"
-- id:       Numeric ID, accessible via self:GetID()
```

### Sizing & Positioning

```lua
frame:SetSize(width, height)
frame:SetWidth(width)
frame:SetHeight(height)
frame:GetSize() -> width, height
frame:GetWidth() -> width
frame:GetHeight() -> height

frame:SetPoint(point [, relativeTo, relativePoint, offsetX, offsetY])
-- point: "TOPLEFT", "TOP", "TOPRIGHT", "LEFT", "CENTER", "RIGHT",
--        "BOTTOMLEFT", "BOTTOM", "BOTTOMRIGHT"
-- relativeTo: reference frame (nil = parent)
-- relativePoint: defaults to same as point if omitted
-- offsetX: pixels, positive = right
-- offsetY: pixels, positive = up

frame:SetAllPoints([relativeTo])      -- Fill entire reference frame
frame:ClearAllPoints()                -- MUST call before re-anchoring
frame:GetPoint(index) -> point, relativeTo, relativePoint, offsetX, offsetY
frame:GetNumPoints() -> count
```

!!! tip
    Always call `ClearAllPoints()` before setting new anchors on a previously-positioned frame. Stale anchors cause layout bugs that are hard to debug.

### Visibility

```lua
frame:Show()
frame:Hide()
frame:SetShown(bool)
frame:IsShown() -> bool    -- Is THIS frame shown? (ignores parent)
frame:IsVisible() -> bool  -- Is this frame AND all parents shown?
frame:SetAlpha(alpha)      -- 0.0 to 1.0
frame:GetAlpha() -> alpha
frame:GetEffectiveAlpha() -> alpha  -- Includes parent alpha multiplication
```

!!! warning "IsShown vs IsVisible"
    `IsShown()` returns `true` even if a parent is hidden (frame is "shown" but not visible). `IsVisible()` returns `true` only if the frame AND all ancestors are shown. AI frequently uses the wrong one.

### Strata & Level

```lua
frame:SetFrameStrata(strata)
-- Strata (low to high): "WORLD", "BACKGROUND", "LOW", "MEDIUM",
--   "HIGH", "DIALOG", "FULLSCREEN", "FULLSCREEN_DIALOG", "TOOLTIP"
frame:GetFrameStrata() -> strata

frame:SetFrameLevel(level)   -- Within same strata, higher = on top
frame:GetFrameLevel() -> level
```

### Mouse & Movement

```lua
frame:EnableMouse(bool)
frame:SetMovable(bool)
frame:SetResizable(bool)
frame:SetClampedToScreen(bool)
frame:RegisterForDrag("LeftButton" [, "RightButton"])
frame:StartMoving()
frame:StopMovingOrSizing()
frame:StartSizing(point)     -- "BOTTOMRIGHT", "RIGHT", etc.
frame:SetResizeBounds(minW, minH [, maxW, maxH])
```

---

## Textures & FontStrings

### Texture

```lua
frame:CreateTexture([name, layer, inherits, subLevel]) -> texture
-- layer: "BACKGROUND", "BORDER", "ARTWORK", "OVERLAY", "HIGHLIGHT"
-- subLevel: -8 to +7 (default 0)

texture:SetTexture(pathOrFileID)              -- "Interface\\Icons\\..." or numeric fileID
texture:SetAtlas(atlasName [, useAtlasSize])  -- Named atlas texture
texture:SetColorTexture(r, g, b [, a])        -- Solid color fill
texture:SetTexCoord(left, right, top, bottom) -- Crop/flip texture
texture:SetVertexColor(r, g, b [, a])         -- Tint
texture:SetBlendMode(mode)                    -- "BLEND", "ADD", "ALPHAKEY", "DISABLE"
texture:SetAllPoints([relativeTo])
texture:SetSize(width, height)
texture:SetDrawLayer(layer [, subLevel])
```

!!! danger "SetTexCoord Parameter Order"
    The correct order is `(left, right, top, bottom)` — NOT `(left, top, right, bottom)`. AI frequently gets this wrong because other graphics APIs use LTRB order.

    ```lua
    -- ✅ CORRECT: left, right, top, bottom
    texture:SetTexCoord(0, 0.5, 0, 0.5)  -- Top-left quarter

    -- ❌ WRONG (AI writes this): left, top, right, bottom
    -- texture:SetTexCoord(0, 0, 0.5, 0.5)  -- WRONG ORDER
    ```

### FontString

```lua
frame:CreateFontString([name, layer, inherits]) -> fontstring
-- Common inherits: "GameFontNormal", "GameFontNormalLarge",
--   "GameFontHighlight", "GameFontHighlightSmall", "NumberFontNormal"

fontstring:SetText(text)
fontstring:GetText() -> text
fontstring:SetFont(fontPath, size [, flags]) -> didSet
-- flags: "OUTLINE", "THICKOUTLINE", "MONOCHROME", or combinations
-- e.g. "OUTLINE, MONOCHROME"

fontstring:SetTextColor(r, g, b [, a])
fontstring:SetJustifyH(justify)   -- "LEFT", "CENTER", "RIGHT"
fontstring:SetJustifyV(justify)   -- "TOP", "MIDDLE", "BOTTOM"
fontstring:SetWordWrap(bool)
fontstring:SetMaxLines(count)
fontstring:GetStringWidth() -> width   -- Actual rendered text width
fontstring:GetStringHeight() -> height
```

### Line & MaskTexture

```lua
frame:CreateLine([name, layer, inherits, subLevel]) -> line
line:SetThickness(thickness)
line:SetStartPoint(point, relativeTo, offsetX, offsetY)
line:SetEndPoint(point, relativeTo, offsetX, offsetY)
line:SetColorTexture(r, g, b [, a])

frame:CreateMaskTexture([name, layer]) -> mask
mask:SetTexture(path)
texture:AddMaskTexture(mask)
texture:RemoveMaskTexture(mask)
```

---

## Events & Scripts

### Event Registration

```lua
frame:RegisterEvent(event) -> bool
frame:UnregisterEvent(event)
frame:UnregisterAllEvents()
frame:IsEventRegistered(event) -> bool

-- Unit-filtered events (more efficient than RegisterEvent + manual filtering):
frame:RegisterUnitEvent(event, unit1 [, unit2])
-- Only fires for specified units. Works with:
-- UNIT_HEALTH, UNIT_POWER_UPDATE, UNIT_AURA, UNIT_SPELLCAST_*, etc.
```

!!! tip "RegisterUnitEvent"
    AI almost never uses `RegisterUnitEvent`. It's significantly more efficient than registering the general event and filtering in the handler. Always prefer it for unit-specific events.

### Script Handlers

```lua
frame:SetScript(scriptType, handler)
-- handler receives (self, ...) where ... varies by script type

frame:GetScript(scriptType) -> handler
frame:HasScript(scriptType) -> bool

frame:HookScript(scriptType, handler)
-- Runs AFTER the existing handler. Cannot prevent or modify the original.
-- Does NOT replace the original like SetScript does.
```

**Common script types and their arguments:**

| Script | Handler Arguments |
|--------|-------------------|
| `OnEvent` | `(self, event, ...)` |
| `OnUpdate` | `(self, elapsed)` — elapsed is seconds since last frame |
| `OnShow` | `(self)` |
| `OnHide` | `(self)` |
| `OnClick` | `(self, button, down)` |
| `OnEnter` | `(self, motion)` |
| `OnLeave` | `(self, motion)` |
| `OnMouseDown` | `(self, button)` |
| `OnMouseUp` | `(self, button)` |
| `OnDragStart` | `(self, button)` |
| `OnDragStop` | `(self)` |
| `OnSizeChanged` | `(self, width, height)` |
| `OnValueChanged` | `(self, value)` — for Slider/StatusBar |
| `OnTextChanged` | `(self, isUserInput)` — for EditBox |
| `OnEnterPressed` | `(self)` — for EditBox |
| `OnEscapePressed` | `(self)` — for EditBox |

### Global Hook

```lua
hooksecurefunc([table,] "funcName", hookFunc)
-- Runs AFTER the original function. Cannot change return values.
-- Does NOT taint the original. Safe to use.
-- hookFunc receives the SAME arguments as the original.

-- Example: Hook a Blizzard function
hooksecurefunc("TargetFrame_Update", function(self)
    -- Runs after every TargetFrame_Update call
end)

-- Example: Hook a method on a table
hooksecurefunc(GameTooltip, "SetUnit", function(self)
    -- Runs after GameTooltip:SetUnit()
end)
```

---

## Timers

```lua
C_Timer.After(seconds, callback)
-- One-shot timer. Fires once after delay.
-- ⚠️ DOES NOT RETURN A HANDLE. Cannot be cancelled!

local ticker = C_Timer.NewTicker(seconds, callback [, iterations])
-- Repeating timer. Optional iteration limit.
ticker:Cancel()
ticker:IsCancelled() -> bool

local timer = C_Timer.NewTimer(seconds, callback)
-- One-shot timer WITH a cancel handle.
timer:Cancel()
timer:IsCancelled() -> bool
```

!!! danger "C_Timer.After Cannot Be Cancelled"
    This is the #1 timer mistake AI makes:
    ```lua
    -- ❌ WRONG: AI writes this constantly
    local timer = C_Timer.After(5, myFunc)
    timer:Cancel()  -- ERROR: C_Timer.After returns nil!

    -- ✅ CORRECT: Use NewTimer if you need cancellation
    local timer = C_Timer.NewTimer(5, myFunc)
    timer:Cancel()  -- Works
    ```

---

## Unit Functions

```lua
-- Identity
UnitName(unit) -> name, realm
UnitFullName(unit) -> name, realm
UnitGUID(unit) -> guid                    -- "Player-1234-0ABCDEF0"
UnitClass(unit) -> className, classFile, classID
-- className: localized ("Warrior"), classFile: token ("WARRIOR"), classID: number

-- Status
UnitExists(unit) -> bool
UnitIsPlayer(unit) -> bool
UnitIsDead(unit) -> bool
UnitIsDeadOrGhost(unit) -> bool
UnitIsAFK(unit) -> bool
UnitAffectingCombat(unit) -> bool
UnitIsConnected(unit) -> bool

-- Health & Power
UnitHealth(unit) -> health
UnitHealthMax(unit) -> maxHealth
UnitPower(unit [, powerType]) -> power
UnitPowerMax(unit [, powerType]) -> maxPower
-- powerType: Enum.PowerType.Mana (0), .Rage (1), .Focus (2), .Energy (3),
--   .ComboPoints (4), .RunicPower (6), etc.

-- Level & Faction
UnitLevel(unit) -> level  -- Returns -1 for ?? mobs
UnitFactionGroup(unit) -> faction, localizedFaction
UnitRace(unit) -> raceName, raceFile, raceID

-- Targeting
UnitIsUnit(unit1, unit2) -> bool  -- Are they the same unit?
UnitIsFriend(unit1, unit2) -> bool
UnitIsEnemy(unit1, unit2) -> bool
UnitCanAttack(unit1, unit2) -> bool
```

!!! warning "Secret Values in Midnight"
    In combat/instanced content, many unit functions return **Secret Values** instead of raw numbers. These can be displayed but not compared or used in arithmetic. Functions affected include `UnitHealth`, `UnitPower`, and others for enemy units.

**Common unit tokens:**

| Token | Description |
|-------|-------------|
| `"player"` | The player |
| `"target"` | Player's target |
| `"focus"` | Focus target |
| `"pet"` | Player's pet |
| `"mouseover"` | Unit under cursor |
| `"party1"`–`"party4"` | Party members |
| `"raid1"`–`"raid40"` | Raid members |
| `"boss1"`–`"boss8"` | Boss frames |
| `"arena1"`–`"arena5"` | Arena opponents |
| `"nameplate1"`–`"nameplateN"` | Visible nameplates |

---

## Auras (Buffs & Debuffs)

```lua
-- 12.0+ Aura API (replaces old UnitBuff/UnitDebuff):
C_UnitAuras.GetAuraDataByIndex(unit, index [, filter]) -> AuraData or nil
C_UnitAuras.GetAuraDataByAuraInstanceID(unit, auraInstanceID) -> AuraData or nil
C_UnitAuras.GetAuraDataBySpellName(unit, spellName [, filter]) -> AuraData or nil
C_UnitAuras.GetPlayerAuraBySpellID(spellID) -> AuraData or nil

-- AuraData table fields:
-- .name, .icon, .count (stacks), .duration, .expirationTime,
-- .sourceUnit, .isStealable, .spellId, .auraInstanceID,
-- .isHarmful, .isHelpful, .points (array of effect values)

-- Filter strings: "HELPFUL", "HARMFUL", "PLAYER", "CANCELABLE", "NOT_CANCELABLE"
-- Combine with |: "HELPFUL|PLAYER"

-- Iteration helper:
AuraUtil.ForEachAura(unit, filter, maxCount, func)
-- func receives (auraData) for each aura, return true to stop
```

!!! danger "Deprecated Aura Functions"
    ```lua
    -- ❌ REMOVED in 12.0:
    -- UnitBuff(unit, index) -- GONE
    -- UnitDebuff(unit, index) -- GONE
    -- UnitAura(unit, index, filter) -- GONE

    -- ✅ Use C_UnitAuras instead (see above)
    ```

---

## Spells (CHANGED IN MIDNIGHT)

```lua
-- ✅ CORRECT 12.0+ API:
C_Spell.GetSpellInfo(spellID) -> SpellInfo or nil
-- SpellInfo: { name, iconID, originalIconID, castTime, minRange, maxRange, spellID }

C_Spell.GetSpellName(spellID) -> name
C_Spell.GetSpellTexture(spellID) -> iconID
C_Spell.GetSpellDescription(spellID) -> description

C_Spell.GetSpellCooldown(spellID) -> CooldownInfo
-- CooldownInfo: { startTime, duration, isEnabled, modRate }

C_Spell.GetSpellCharges(spellID) -> SpellChargeInfo or nil
-- SpellChargeInfo: { currentCharges, maxCharges, cooldownStartTime, cooldownDuration, chargeModRate }

C_Spell.IsSpellUsable(spellID) -> isUsable, notEnoughPower
C_Spell.IsSpellInRange(spellID, unit) -> bool or nil
```

!!! danger "GetSpellInfo Is Deprecated"
    ```lua
    -- ❌ OLD (pre-12.0) — AI WILL generate this:
    local name, rank, icon, castTime, minRange, maxRange, spellID = GetSpellInfo(spellID)

    -- ✅ NEW (12.0+) — Returns a TABLE, not multiple values:
    local info = C_Spell.GetSpellInfo(spellID)
    local name = info.name
    local icon = info.iconID
    local castTime = info.castTime
    ```

---

## Items (CHANGED IN MIDNIGHT)

```lua
-- ✅ CORRECT 12.0+ API:
C_Item.GetItemInfo(itemID) -> ItemInfo or nil  -- May return nil until cached!
-- ItemInfo: { itemName, itemLink, itemQuality, itemLevel, itemMinLevel,
--   itemType, itemSubType, itemStackCount, itemEquipLoc, itemTexture, ... }

C_Item.GetItemName(itemID) -> name or nil
C_Item.GetItemIconByID(itemID) -> iconID or nil
C_Item.GetItemQualityByID(itemID) -> quality or nil

-- Force item data to load (async):
C_Item.RequestLoadItemDataByID(itemID)
-- Listen for ITEM_DATA_LOAD_RESULT event after calling this

C_Item.IsItemDataCachedByID(itemID) -> bool
C_Item.DoesItemExistByID(itemID) -> bool

-- Item links:
C_Item.GetItemLink(itemID) -> link or nil
```

!!! danger "GetItemInfo Is Deprecated"
    ```lua
    -- ❌ OLD — AI generates this constantly:
    local name, link, quality, iLevel, reqLevel, class, subclass,
          maxStack, equipSlot, texture, vendorPrice = GetItemInfo(itemID)

    -- ✅ NEW (12.0+) — Returns a TABLE:
    local info = C_Item.GetItemInfo(itemID)
    if info then
        local name = info.itemName
        local icon = info.itemTexture
    end
    -- IMPORTANT: May return nil! Item data loads asynchronously.
    ```

---

## Addon Communication

```lua
C_ChatInfo.RegisterAddonMessagePrefix(prefix)
-- prefix: max 16 characters. Must register before sending/receiving.

C_ChatInfo.SendAddonMessage(prefix, text, chatType [, target])
-- text: max 255 bytes
-- chatType: "PARTY", "RAID", "GUILD", "WHISPER", "CHANNEL"
-- target: required for WHISPER (player name) and CHANNEL (channel index)

-- Receive via event:
-- CHAT_MSG_ADDON: prefix, text, channel, sender
```

!!! warning "Midnight Instance Restriction"
    In Midnight (12.0+), addon-to-addon chat communication is **blocked inside instances** (dungeons, raids, arenas, battlegrounds). Addons cannot send or receive `CHAT_MSG_ADDON` messages while in instanced content. This breaks addons that relied on real-time data sharing between group members.

---

## SavedVariables

```lua
-- Declared in .toc file:
-- ## SavedVariables: MyAddonDB
-- ## SavedVariablesPerCharacter: MyAddonCharDB

-- Access pattern (the ONLY correct way):
local addonName, ns = ...  -- In your main .lua file

local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, loadedAddon)
    if loadedAddon == addonName then
        -- NOW MyAddonDB is available as a global
        MyAddonDB = MyAddonDB or {}  -- Initialize with defaults
        self:UnregisterEvent("ADDON_LOADED")
    end
end)
```

!!! tip "LoadSavedVariablesFirst (11.1.5+)"
    With `## LoadSavedVariablesFirst: 1` in your TOC, saved variables are available immediately in top-level code — no need to wait for `ADDON_LOADED`. This is a significant simplification for new addons targeting 12.0+.

---

## Slash Commands

```lua
SLASH_MYADDON1 = "/myaddon"
SLASH_MYADDON2 = "/ma"
SlashCmdList["MYADDON"] = function(msg, editBox)
    -- msg: everything after the slash command (trimmed)
    local cmd, rest = msg:match("^(%S*)%s*(.-)$")
    if cmd == "config" then
        Settings.OpenToCategory("MyAddon")
    elseif cmd == "reset" then
        -- reset something
    else
        print("Usage: /myaddon config | reset")
    end
end
```

!!! danger "Slash Command Naming"
    The `SLASH_` global has a numeric suffix (`SLASH_MYADDON1`), but the `SlashCmdList` key does **NOT** (`SlashCmdList["MYADDON"]`). AI gets this wrong constantly:
    ```lua
    -- ❌ WRONG:
    SlashCmdList["MYADDON1"] = function(msg) end  -- Key should NOT have "1"

    -- ❌ WRONG:
    SLASH_MYADDON = "/myaddon"  -- MUST have numeric suffix

    -- ✅ CORRECT:
    SLASH_MYADDON1 = "/myaddon"         -- Suffix "1" on the global
    SlashCmdList["MYADDON"] = function(msg) end  -- No suffix on the key
    ```

---

## Combat & Security

```lua
InCombatLockdown() -> bool
-- true = in combat. Cannot create/modify secure frames, change bindings,
-- call protected functions, or load addons.

issecurevariable([table,] "variable") -> isSecure, taintedBy
-- Check if a variable is untainted (safe to use in secure context)

securecall(func, ...) -> ...
-- Call a function without spreading taint

hooksecurefunc([table,] "funcName", hookFunc)
-- Post-hook that runs AFTER the original. Does NOT taint.
-- hookFunc receives the SAME arguments as the original.
-- Cannot modify return values or prevent execution.
```

---

## Tooltip

```lua
-- Use the global GameTooltip (do NOT create your own for simple tooltips):
GameTooltip:SetOwner(ownerFrame, anchor [, offsetX, offsetY])
-- anchor: "ANCHOR_TOP", "ANCHOR_BOTTOM", "ANCHOR_LEFT", "ANCHOR_RIGHT",
--   "ANCHOR_TOPLEFT", "ANCHOR_TOPRIGHT", "ANCHOR_BOTTOMLEFT",
--   "ANCHOR_BOTTOMRIGHT", "ANCHOR_CURSOR", "ANCHOR_NONE"

GameTooltip:AddLine(text [, r, g, b, wrapText])
GameTooltip:AddDoubleLine(leftText, rightText [, lR, lG, lB, rR, rG, rB])
GameTooltip:SetText(text [, r, g, b, wrapText])  -- Replaces ALL content
GameTooltip:Show()
GameTooltip:Hide()

-- Setting tooltip to show item/spell info:
GameTooltip:SetItemByID(itemID)
GameTooltip:SetSpellByID(spellID)
GameTooltip:SetUnitAura(unit, index [, filter])
```

---

## Addon Compartment (Minimap, 10.1.0+)

```lua
-- Declared in .toc file:
-- ## AddonCompartmentFunc: MyAddon_OnClick
-- ## AddonCompartmentFuncOnEnter: MyAddon_OnEnter
-- ## AddonCompartmentFuncOnLeave: MyAddon_OnLeave

-- Must be GLOBAL functions:
function MyAddon_OnClick(addonName, mouseButton)
    Settings.OpenToCategory("MyAddon")
end

function MyAddon_OnEnter(addonName, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("My Addon")
    GameTooltip:AddLine("Click to open settings", 1, 1, 1)
    GameTooltip:Show()
end

function MyAddon_OnLeave(addonName, menuButtonFrame)
    GameTooltip:Hide()
end
```

---

## Settings (Modern API, 10.0+)

```lua
-- Create a settings category
local category = Settings.RegisterVerticalLayoutCategory("My Addon")

-- Add a checkbox setting
local variable = "MyAddonDB.enableFeature"
local name = "Enable Feature"
local tooltip = "Toggles the main feature on or off."
local defaultValue = true

local setting = Settings.RegisterAddOnSetting(category, variable, name, MyAddonDB, type(defaultValue), defaultValue)
Settings.CreateCheckbox(category, setting, tooltip)

-- Add a slider setting
local sliderSetting = Settings.RegisterAddOnSetting(category, "MyAddonDB.opacity", "Opacity", MyAddonDB, type(1.0), 1.0)
local options = Settings.CreateSliderOptions(0, 1, 0.05)
options:SetLabelFormatter(MinimalSliderWithSteppersMixin.Label.Right, function(value)
    return string.format("%.0f%%", value * 100)
end)
Settings.CreateSlider(category, sliderSetting, options, "Set the frame opacity.")

-- Register the category
Settings.RegisterAddOnCategory(category)

-- Open settings to your addon
Settings.OpenToCategory(category:GetID())
```

---

## Mixins & Templates

```lua
-- Apply mixin to an existing object:
Mixin(target, mixin1 [, mixin2, ...])
-- Shallow-copies all keys from mixin tables onto target.
-- Later mixins overwrite earlier ones on key collision.

-- Create a new table from mixins:
local obj = CreateFromMixins(mixin1 [, mixin2, ...])

-- Common usage pattern:
MyFrameMixin = {}
function MyFrameMixin:OnLoad()
    self:RegisterEvent("PLAYER_LOGIN")
end
function MyFrameMixin:OnEvent(event, ...)
    -- handle events
end

-- In XML:
-- <Frame mixin="MyFrameMixin">
--   <Scripts>
--     <OnLoad method="OnLoad"/>
--     <OnEvent method="OnEvent"/>
--   </Scripts>
-- </Frame>

-- In Lua:
local frame = CreateFrame("Frame", nil, UIParent)
Mixin(frame, MyFrameMixin)
frame:OnLoad()  -- Must call manually when applying via Lua
```

---

## Addon Metadata

```lua
-- Query metadata from .toc:
C_AddOns.GetAddOnMetadata(addonName, field) -> value
-- field: "Title", "Notes", "Author", "Version", "X-Website", etc.

C_AddOns.GetNumAddOns() -> count
C_AddOns.GetAddOnInfo(index) -> name, title, notes, loadable, reason, security
C_AddOns.IsAddOnLoaded(addonNameOrIndex) -> isLoaded, isLoadFinished
C_AddOns.LoadAddOn(addonName) -- Load a LoadOnDemand addon
C_AddOns.EnableAddOn(addonName [, character])
C_AddOns.DisableAddOn(addonName [, character])
```

!!! warning "Deprecated: GetAddOnMetadata without C_AddOns"
    ```lua
    -- ❌ OLD:
    GetAddOnMetadata("MyAddon", "Version")

    -- ✅ NEW:
    C_AddOns.GetAddOnMetadata("MyAddon", "Version")
    ```

---

## Common AI Mistakes

This section catalogs the most frequent errors AI coding assistants make when generating WoW addon code. Each entry shows the wrong code, the correct code, and why the mistake happens.

---

### Mistake 1: Using deprecated GetSpellInfo

AI was trained on millions of lines of pre-12.0 code where `GetSpellInfo` was the standard.

```lua
-- ❌ WRONG (AI writes this):
local name, rank, icon, castTime = GetSpellInfo(spellID)

-- ✅ CORRECT (12.0+):
local info = C_Spell.GetSpellInfo(spellID)
local name = info and info.name
local icon = info and info.iconID
```

---

### Mistake 2: Using deprecated GetItemInfo

Same issue — old global function replaced by C_ namespace.

```lua
-- ❌ WRONG:
local name, link, quality = GetItemInfo(itemID)

-- ✅ CORRECT (12.0+):
local info = C_Item.GetItemInfo(itemID)
-- May return nil! Check before accessing fields.
if info then
    local name = info.itemName
end
```

---

### Mistake 3: Trying to cancel C_Timer.After

`C_Timer.After` returns **nil**. AI assumes all timer functions return handles.

```lua
-- ❌ WRONG:
local timer = C_Timer.After(5, myCallback)
timer:Cancel()  -- ERROR: timer is nil

-- ✅ CORRECT:
local timer = C_Timer.NewTimer(5, myCallback)
timer:Cancel()  -- Works
```

---

### Mistake 4: SetTexCoord parameter order

WoW uses `(left, right, top, bottom)`. Most other graphics APIs use `(left, top, right, bottom)`.

```lua
-- ❌ WRONG (AI uses LTRB from other APIs):
texture:SetTexCoord(0, 0, 1, 1)

-- ✅ CORRECT (WoW uses LRTB):
texture:SetTexCoord(0, 1, 0, 1)  -- Full texture, no crop
texture:SetTexCoord(0, 0.5, 0, 0.5)  -- Top-left quarter
```

---

### Mistake 5: Slash command key mismatch

AI confuses the naming convention between `SLASH_` globals and `SlashCmdList` keys.

```lua
-- ❌ WRONG:
SLASH_MYADDON = "/myaddon"              -- Missing numeric suffix
SlashCmdList["MYADDON1"] = function() end  -- Key should NOT have suffix

-- ✅ CORRECT:
SLASH_MYADDON1 = "/myaddon"             -- "1" suffix on SLASH_ global
SlashCmdList["MYADDON"] = function() end -- No suffix on SlashCmdList key
```

---

### Mistake 6: Using UnitBuff / UnitDebuff (removed in 12.0)

These functions were removed entirely. AI doesn't know.

```lua
-- ❌ WRONG (removed in 12.0):
local name, icon, count, debuffType, duration, expirationTime = UnitBuff("player", i)

-- ✅ CORRECT:
local auraData = C_UnitAuras.GetAuraDataByIndex("player", i, "HELPFUL")
if auraData then
    local name = auraData.name
    local icon = auraData.icon
    local stacks = auraData.count
end
```

---

### Mistake 7: SetBackdrop without BackdropTemplate

Since 9.0, `SetBackdrop` requires inheriting `BackdropTemplate`. AI trained on older code omits this.

```lua
-- ❌ WRONG (errors since 9.0):
local frame = CreateFrame("Frame", nil, UIParent)
frame:SetBackdrop({...})  -- ERROR: SetBackdrop is nil

-- ✅ CORRECT:
local frame = CreateFrame("Frame", nil, UIParent, "BackdropTemplate")
frame:SetBackdrop({
    bgFile = "Interface\\Tooltips\\UI-Tooltip-Background",
    edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
    tile = true, tileSize = 16, edgeSize = 16,
    insets = { left = 4, right = 4, top = 4, bottom = 4 },
})
```

---

### Mistake 8: Confusing HookScript and hooksecurefunc

These are completely different mechanisms. AI mixes them up.

```lua
-- ❌ WRONG (mixing up the two):
frame:hooksecurefunc("OnClick", myHandler)  -- Not a frame method
hooksecurefunc(frame, "OnClick", myHandler)  -- OnClick isn't a named method

-- ✅ CORRECT — For script hooks on frames:
frame:HookScript("OnClick", myHandler)

-- ✅ CORRECT — For global/table function hooks:
hooksecurefunc("ChatFrame_OpenChat", myHandler)
hooksecurefunc(GameTooltip, "SetUnit", myHandler)
```

---

### Mistake 9: Forgetting ClearAllPoints before re-anchoring

AI sets new points without clearing old ones, causing layout bugs.

```lua
-- ❌ WRONG (stale anchors remain):
frame:SetPoint("CENTER", UIParent, "CENTER")
-- ... later ...
frame:SetPoint("TOPLEFT", UIParent, "TOPLEFT", 10, -10)
-- Frame now has TWO anchor sets = unpredictable layout

-- ✅ CORRECT:
frame:ClearAllPoints()
frame:SetPoint("TOPLEFT", UIParent, "TOPLEFT", 10, -10)
```

---

### Mistake 10: Accessing SavedVariables before ADDON_LOADED

AI accesses the global before it exists.

```lua
-- ❌ WRONG (MyAddonDB is nil at load time):
MyAddonDB = MyAddonDB or {}
MyAddonDB.setting = true

-- ✅ CORRECT:
local frame = CreateFrame("Frame")
frame:RegisterEvent("ADDON_LOADED")
frame:SetScript("OnEvent", function(self, event, addon)
    if addon == "MyAddon" then
        MyAddonDB = MyAddonDB or {}
        -- NOW it's safe to use
    end
end)
```

---

### Mistake 11: Using COMBAT_LOG_EVENT_UNFILTERED (removed in 12.0)

This event was entirely removed in Midnight. AI generates handlers for it constantly.

```lua
-- ❌ WRONG (event removed in 12.0):
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")
frame:SetScript("OnEvent", function(self, event)
    local _, subEvent, _, sourceGUID = CombatLogGetCurrentEventInfo()
end)

-- ✅ CORRECT: There is no direct replacement. Use:
-- - C_UnitAuras for buff/debuff tracking
-- - Blizzard's native damage meter API for combat stats
-- - Boss mod frameworks (DBM/BigWigs) for encounter mechanics
```

---

### Mistake 12: Wrong ChatFrame print method

AI invents a `ChatFrame:Print()` or `print()` with color arguments.

```lua
-- ❌ WRONG:
DEFAULT_CHAT_FRAME:Print("Hello")  -- Print doesn't exist on chat frames
print("Hello", 1, 0, 0)           -- print doesn't accept color args

-- ✅ CORRECT:
print("Hello")                     -- Simple output to chat
DEFAULT_CHAT_FRAME:AddMessage("Hello", 1, 0.5, 0)  -- With color (r,g,b)
```

---

### Mistake 13: Using GetAddOnMetadata without C_AddOns prefix

Old global is deprecated.

```lua
-- ❌ WRONG:
local version = GetAddOnMetadata("MyAddon", "Version")

-- ✅ CORRECT:
local version = C_AddOns.GetAddOnMetadata("MyAddon", "Version")
```

---

### Mistake 14: Creating EditBox without clearing focus

AI forgets that EditBoxes capture keyboard input until focus is explicitly cleared.

```lua
-- ❌ WRONG (steals keyboard focus forever):
local editBox = CreateFrame("EditBox", nil, UIParent, "InputBoxTemplate")
editBox:SetSize(200, 30)
editBox:SetPoint("CENTER")
-- No OnEscapePressed handler = player can't type in chat!

-- ✅ CORRECT:
local editBox = CreateFrame("EditBox", nil, UIParent, "InputBoxTemplate")
editBox:SetSize(200, 30)
editBox:SetPoint("CENTER")
editBox:SetAutoFocus(false)  -- Don't steal focus on show
editBox:SetScript("OnEscapePressed", function(self)
    self:ClearFocus()
end)
editBox:SetScript("OnEnterPressed", function(self)
    local text = self:GetText()
    -- process text
    self:ClearFocus()
end)
```

---

### Mistake 15: IsShown vs IsVisible confusion

AI uses `IsShown()` when it means "is this frame actually visible on screen."

```lua
-- ❌ WRONG (returns true even if parent is hidden):
if frame:IsShown() then
    -- Frame might NOT be visible if parent is hidden!
end

-- ✅ CORRECT (checks full visibility chain):
if frame:IsVisible() then
    -- Frame is truly visible on screen
end
```

---

### Mistake 16: Using SendChatMessage for addon communication

AI confuses player chat with addon messaging.

```lua
-- ❌ WRONG:
SendChatMessage("addon:data:here", "PARTY")  -- Sends visible chat!

-- ✅ CORRECT:
C_ChatInfo.RegisterAddonMessagePrefix("MyAddon")
C_ChatInfo.SendAddonMessage("MyAddon", "data:here", "PARTY")
-- Invisible to players, received via CHAT_MSG_ADDON event
```

---

### Mistake 17: Registering events on regions instead of frames

AI sometimes tries to register events on textures or fontstrings.

```lua
-- ❌ WRONG:
local tex = frame:CreateTexture(nil, "ARTWORK")
tex:SetScript("OnEvent", handler)  -- ERROR: textures can't have scripts

-- ✅ CORRECT:
-- Only Frame objects (and descendants) can register events and scripts.
-- Use the parent frame for event handling.
frame:RegisterEvent("PLAYER_LOGIN")
frame:SetScript("OnEvent", handler)
```

---

### Mistake 18: OnUpdate without throttling

AI generates `OnUpdate` handlers that do expensive work every frame.

```lua
-- ❌ WRONG (runs 60-240+ times per second):
frame:SetScript("OnUpdate", function(self, elapsed)
    self.text:SetText(UnitHealth("player"))  -- Runs EVERY FRAME
end)

-- ✅ CORRECT (throttled):
local elapsed_total = 0
frame:SetScript("OnUpdate", function(self, elapsed)
    elapsed_total = elapsed_total + elapsed
    if elapsed_total >= 0.1 then  -- 10 times per second max
        elapsed_total = 0
        self.text:SetText(UnitHealth("player"))
    end
end)
-- Even better: use UNIT_HEALTH event instead of OnUpdate polling
```

---

### Mistake 19: Using GetWeaponEnchantInfo() to detect Rogue poisons

Since Dragonflight (10.0), Rogue poisons are **player buffs**, not temporary weapon enchants. AI trained on older code generates `GetWeaponEnchantInfo()` for poison detection — this silently returns no data for poisons and the addon appears broken.

```lua
-- ❌ WRONG (poisons are not weapon enchants since Dragonflight):
local hasMainHand, mainExpiration = GetWeaponEnchantInfo()
if not hasMainHand then
    print("No poison on main hand!")  -- Always fires, even with poisons applied!
end

-- ✅ CORRECT (poisons are player buffs, check by spell ID):
local LETHAL_POISONS = {
    [2823]   = "Deadly Poison",
    [315584] = "Instant Poison",
    [8679]   = "Wound Poison",
    [381637] = "Amplifying Poison",
}

for spellID, name in pairs(LETHAL_POISONS) do
    local aura = C_UnitAuras.GetPlayerAuraBySpellID(spellID)
    if aura then
        print("Active lethal poison: " .. name)
        break
    end
end

-- Also wrong: listening to UNIT_INVENTORY_CHANGED for poison changes.
-- ✅ Use UNIT_AURA instead — poisons are buffs, not equipment.
```
