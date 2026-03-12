---
title: "Better Addon Patterns"
description: "10 battle-tested code patterns from BetterBags, ElvUI, Masque, Bartender4, and other top WoW addons — complete working examples for Midnight 12.0+."
---

# Better Addon Patterns

Real techniques extracted from production addons that enhance Blizzard's UI without replacing it. Every pattern includes complete, copy-pasteable code targeting Interface `120001`.

!!! tip "Prerequisites"
    This page assumes familiarity with [Midnight Patterns](midnight-patterns.md) for the security model and API changes. Here we focus on *architecture* — how mature addons structure hooking, skinning, combat safety, and lifecycle management.

---

## Pattern 1: Hooking Secure Functions

Post-hook Blizzard functions without tainting the secure execution chain. The only safe way to react to protected function calls.

```lua
local addonName, ns = ...

-- ─── Hook a global function ───────────────────────────────
-- Fires AFTER Blizzard's function completes. Same arguments, no return.
hooksecurefunc("CompactUnitFrame_UpdateHealth", function(frame)
    if frame:IsForbidden() then return end
    frame.healthBar:SetStatusBarTexture(
        "Interface\\AddOns\\" .. addonName .. "\\Textures\\StatusBar"
    )
end)

-- ─── Hook an object method ────────────────────────────────
hooksecurefunc(GameTooltip, "SetAction", function(self, slot)
    local kind, id = GetActionInfo(slot)
    if issecretvalue(id) then return end -- Midnight: handle secret values
    if kind == "spell" then
        self:AddLine("Spell ID: " .. id, 1, 1, 1)
        self:Show()
    end
end)

-- ─── Hook a metatable (all widgets of a type) ────────────
-- OmniCC-style: intercept ALL cooldown starts globally
local Cooldown_MT = getmetatable(
    CreateFrame("Cooldown", nil, nil, "CooldownFrameTemplate")
).__index

hooksecurefunc(Cooldown_MT, "SetCooldown", function(self, start, duration)
    if duration and duration > 1.5 then
        -- Create or update timer text on this cooldown
        if not self.__myTimer then
            self.__myTimer = self:GetParent():CreateFontString(
                nil, "OVERLAY", "GameFontNormal"
            )
            self.__myTimer:SetPoint("CENTER")
        end
        self.__myTimer:SetText(math.floor(duration) .. "s")
    end
end)
```

!!! warning "Hooks are permanent"
    `hooksecurefunc` cannot be unhooked — it persists until UI reload. Multiple hooks on the same function all execute in order. Design accordingly.

!!! danger "Midnight 12.0: Secret Values"
    Hook arguments may be opaque `SecretValue` tokens in combat contexts. Always guard with `issecretvalue(arg)` before using values in conditionals.

**Inspired by:** idTip (tooltip IDs), OmniCC (cooldown text), ElvUI (frame skinning)

??? failure "Common Mistakes"
    - **Wrapping instead of hooking** — replacing a secure function taints the call chain. Always use `hooksecurefunc`, never `originalFunc = SomeFunc; SomeFunc = myWrapper`.
    - **Hooking mixin tables** — mixin methods are *copied* onto instances. Hook the frame instance, not the mixin table.
    - **Forgetting `IsForbidden()`** — accessing a forbidden frame throws a hard error. Check first.
    - **Creating objects in mouse-event hooks** — BetterBags discovered that creating context objects before protected clicks causes taint. Keep hook callbacks lightweight.

---

## Pattern 2: Mixin Extension

Compose behavior from reusable tables using Blizzard's mixin system.

```lua
local addonName, ns = ...

-- ─── Define mixins ────────────────────────────────────────
ns.EventMixin = {}

function ns.EventMixin:InitEvents()
    self.eventFrame = self.eventFrame or CreateFrame("Frame")
    self.eventFrame:SetScript("OnEvent", function(_, event, ...)
        if self[event] then
            self[event](self, ...)
        end
    end)
end

function ns.EventMixin:RegisterAddonEvent(event)
    self:InitEvents()
    self.eventFrame:RegisterEvent(event)
end

-- ─── Tooltip mixin ────────────────────────────────────────
ns.TooltipMixin = {}

function ns.TooltipMixin:ShowTooltip(text)
    GameTooltip:SetOwner(self.frame or self, "ANCHOR_RIGHT")
    GameTooltip:SetText(text)
    GameTooltip:Show()
end

function ns.TooltipMixin:HideTooltip()
    GameTooltip:Hide()
end

-- ─── Compose multiple mixins ──────────────────────────────
ns.WidgetMixin = CreateFromMixins(ns.EventMixin, ns.TooltipMixin)

function ns.WidgetMixin:Init(parent)
    self.frame = CreateFrame("Frame", nil, parent, "BackdropTemplate")
    self.frame:SetSize(200, 50)
    self:RegisterAddonEvent("PLAYER_ENTERING_WORLD")
end

function ns.WidgetMixin:PLAYER_ENTERING_WORLD()
    self:ShowTooltip("Widget loaded!")
end

-- ─── Usage ────────────────────────────────────────────────
local widget = CreateFromMixins(ns.WidgetMixin)
widget:Init(UIParent)

-- ─── Apply mixin to an existing Blizzard frame ───────────
-- Hook the instance AFTER mixin methods are copied onto it
local frame = CreateFrame("Frame", nil, UIParent)
Mixin(frame, ns.TooltipMixin)
frame:SetScript("OnEnter", function(self)
    self:ShowTooltip("Hover text")
end)
frame:SetScript("OnLeave", function(self)
    self:HideTooltip()
end)
```

!!! warning "Hooking mixin-based Blizzard frames"
    Mixin methods are **copied** onto each frame instance. Hooking the mixin table does nothing for already-created frames:
    ```lua
    -- WRONG: this mixin is already copied, hook won't fire on existing frames
    hooksecurefunc(WorldMapBountyBoardMixin, "ShowBountyTooltip", myFunc)

    -- CORRECT: hook the actual frame instance
    hooksecurefunc(WorldMapFrame.BountyBoard, "ShowBountyTooltip", myFunc)
    ```

**Inspired by:** Blizzard FrameXML (`CreateFromMixins` throughout), ElvUI (module system), BetterBags (strategy pattern via mixins)

??? failure "Common Mistakes"
    - **Hooking the mixin table instead of the instance** — the table is just a blueprint; methods are copied at creation time.
    - **Forgetting `CreateFromMixins` returns a new table** — it's `Mixin({}, ...)`, not a reference to the original.
    - **Later mixins silently overwrite earlier ones** — if two mixins define `OnLoad`, the last one wins with no warning.

---

## Pattern 3: Frame Skinning Without Taint

Strip Blizzard textures and apply custom visuals without breaking secure behavior.

```lua
local addonName, ns = ...

-- ─── Core texture stripping ──────────────────────────────
function ns.StripTextures(frame, keepSpecial)
    for i = 1, frame:GetNumRegions() do
        local region = select(i, frame:GetRegions())
        if region and region:IsObjectType("Texture") then
            if keepSpecial and region:GetDrawLayer() == "HIGHLIGHT" then
                -- Preserve highlight textures for interactivity
            else
                region:SetTexture(nil)
                region:SetAtlas("")
            end
        end
    end
end

-- ─── Apply flat backdrop ─────────────────────────────────
function ns.ApplyBackdrop(frame, r, g, b, a)
    if not frame.SetBackdrop then
        Mixin(frame, BackdropTemplateMixin)
    end
    frame:SetBackdrop({
        bgFile   = "Interface\\Buttons\\WHITE8x8",
        edgeFile = "Interface\\Buttons\\WHITE8x8",
        edgeSize = 1,
        insets   = { left = 1, right = 1, top = 1, bottom = 1 },
    })
    frame:SetBackdropColor(r or 0.1, g or 0.1, b or 0.1, a or 0.9)
    frame:SetBackdropBorderColor(0.3, 0.3, 0.3, 1)
end

-- ─── Skin a Blizzard panel (deferred until it loads) ─────
local eventFrame = CreateFrame("Frame")
eventFrame:RegisterEvent("ADDON_LOADED")
eventFrame:SetScript("OnEvent", function(self, event, addon)
    if addon == "Blizzard_AuctionHouseUI" then
        local ahf = AuctionHouseFrame
        if not ahf then return end

        ns.StripTextures(ahf)
        ns.ApplyBackdrop(ahf)

        -- Skin the close button
        if ahf.CloseButton then
            ns.StripTextures(ahf.CloseButton)
        end

        -- Hide portrait without destroying it
        if ahf.Portrait then
            ahf.Portrait:SetAlpha(0)
        end
        if ahf.PortraitOverlay then
            ahf.PortraitOverlay:SetAlpha(0)
        end

        -- Re-skin on tab change (dynamic content)
        hooksecurefunc(ahf, "SetDisplayMode", function(frame)
            ns.StripTextures(frame, true)
        end)
    end
end)

-- ─── Skin a button (ElvUI-style) ─────────────────────────
function ns.SkinButton(button)
    ns.StripTextures(button)
    button:SetNormalTexture("")
    button:SetHighlightTexture("")
    button:SetPushedTexture("")

    ns.ApplyBackdrop(button)

    -- Custom highlight
    local highlight = button:CreateTexture(nil, "HIGHLIGHT")
    highlight:SetAllPoints()
    highlight:SetColorTexture(1, 1, 1, 0.15)
    highlight:SetBlendMode("ADD")
end
```

!!! tip "SetBackdrop requires a new table each time"
    Passing the same table reference to `SetBackdrop` silently fails. Always pass a new table literal.

**Inspired by:** ElvUI (`Toolkit.lua` StripTextures, `Skins.lua` HandleFrame), Aurora (flat reskin)

??? failure "Common Mistakes"
    - **Calling `SetScript("OnShow", ...)` on Blizzard frames** — use `HookScript` instead. `SetScript` destroys existing handlers and all hooks, and taints secure frames.
    - **Replacing texture setter functions with `noop`** — this prevents Blizzard from updating textures. Use `hooksecurefunc` to re-apply your skin after Blizzard sets textures instead.
    - **Skinning before the target addon loads** — many Blizzard panels are demand-loaded addons. Wait for `ADDON_LOADED`.

---

## Pattern 4: Combat-Safe Enhancement

Defer frame modifications until combat ends. Every addon that touches secure frames needs this.

```lua
local addonName, ns = ...

-- ─── Simple combat deferral ──────────────────────────────
ns.afterCombatQueue = {}

local combatFrame = CreateFrame("Frame")
combatFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
combatFrame:SetScript("OnEvent", function()
    for i, cb in ipairs(ns.afterCombatQueue) do
        cb()
    end
    wipe(ns.afterCombatQueue)
end)

function ns.AfterCombat(fn)
    if InCombatLockdown() then
        table.insert(ns.afterCombatQueue, fn)
    else
        fn()
    end
end

-- ─── Usage: safe frame modification ──────────────────────
function ns.UpdateBarVisibility(bar, shouldShow)
    ns.AfterCombat(function()
        if shouldShow then
            bar:Show()
        else
            bar:Hide()
        end
    end)
end

-- ─── Debounced refresh with combat awareness ─────────────
-- BetterBags-style: batch rapid events, defer during combat
ns.Refresh = {
    pendingBackpack = false,
    pendingBank = false,
    debounceTimer = nil,
}

function ns.Refresh:Request(backpack, bank)
    self.pendingBackpack = self.pendingBackpack or backpack
    self.pendingBank = self.pendingBank or bank

    if self.debounceTimer then
        self.debounceTimer:Cancel()
    end

    self.debounceTimer = C_Timer.NewTimer(0.05, function()
        self:Execute()
    end)
end

function ns.Refresh:Execute()
    if InCombatLockdown() then
        -- Don't wipe flags — they'll be flushed after combat
        return
    end

    if self.pendingBackpack then
        -- Refresh backpack display
        ns.RefreshBackpack()
    end

    if self.pendingBank then
        -- Refresh bank display
        ns.RefreshBank()
    end

    self.pendingBackpack = false
    self.pendingBank = false
end

-- ─── Event registration ──────────────────────────────────
local refreshFrame = CreateFrame("Frame")
refreshFrame:RegisterEvent("BAG_UPDATE_DELAYED")
refreshFrame:RegisterEvent("PLAYER_REGEN_ENABLED")
refreshFrame:SetScript("OnEvent", function(self, event)
    if event == "BAG_UPDATE_DELAYED" then
        ns.Refresh:Request(true, true)
    elseif event == "PLAYER_REGEN_ENABLED" then
        -- Flush anything deferred during combat
        if ns.Refresh.pendingBackpack or ns.Refresh.pendingBank then
            ns.Refresh:Execute()
        end
    end
end)
```

!!! warning "Use `BAG_UPDATE_DELAYED`, not `BAG_UPDATE`"
    `BAG_UPDATE` fires per-slot during batch operations (vendor buyback, mail). `BAG_UPDATE_DELAYED` fires once after all slots are updated. BetterBags, ElvUI, and AdiBags all use the delayed variant.

**Inspired by:** BetterBags (`data/refresh.lua`), ElvUI (`ActionBars.lua` defer pattern), Bartender4 (`StateBar.lua`)

??? failure "Common Mistakes"
    - **Checking `InCombatLockdown()` only at event time** — combat can start between your event and your timer callback. Check again in the callback.
    - **Forgetting `PLAYER_REGEN_ENABLED` flush** — without it, changes requested during combat are silently lost.
    - **Debouncing with `C_Timer.After` instead of `C_Timer.NewTimer`** — `After` returns nothing; `NewTimer` returns a handle you can `:Cancel()`.

---

## Pattern 5: Callback-Based Skinning

Let other addons skin your buttons. The Masque pattern: register buttons with a group, let external skin addons restyle them.

```lua
local addonName, ns = ...

-- ─── As the addon WITH buttons (registering with Masque) ─
local MSQ = LibStub and LibStub("Masque", true) -- soft dependency
local msqGroup

if MSQ then
    msqGroup = MSQ:Group(addonName, "Action Buttons")
end

function ns.CreateActionButton(parent, index)
    local name = addonName .. "Button" .. index
    local button = CreateFrame("Button", name, parent,
        "SecureActionButtonTemplate, ActionButtonTemplate")
    button:SetSize(36, 36)

    -- Set up button visuals
    button.icon = button:CreateTexture(nil, "ARTWORK")
    button.icon:SetAllPoints()

    button.cooldown = CreateFrame("Cooldown", name .. "Cooldown",
        button, "CooldownFrameTemplate")
    button.cooldown:SetAllPoints()

    -- Register with Masque for external skinning
    if msqGroup then
        msqGroup:AddButton(button, {
            Icon     = button.icon,
            Cooldown = button.cooldown,
            Normal   = button:GetNormalTexture(),
            Pushed   = button:GetPushedTexture(),
            Highlight = button:GetHighlightTexture(),
        })
    end

    return button
end

-- ─── As a SKIN PROVIDER (creating a Masque skin) ─────────
-- Other addons can provide skins that apply to all Masque-aware buttons
local MSQ_Skin = LibStub and LibStub("Masque", true)
if MSQ_Skin then
    MSQ_Skin:AddSkin("My Clean Skin", {
        Backdrop = {
            Texture  = "Interface\\AddOns\\" .. addonName .. "\\Textures\\Backdrop",
            Width    = 40,
            Height   = 40,
            OffsetX  = 0,
            OffsetY  = 0,
        },
        Icon = {
            TexCoords = { 0.07, 0.93, 0.07, 0.93 }, -- Crop icon edges
        },
        Normal = {
            Texture  = "Interface\\AddOns\\" .. addonName .. "\\Textures\\Normal",
            Width    = 40,
            Height   = 40,
            Color    = { 0.3, 0.3, 0.3, 1 },
        },
        Gloss = {
            Texture  = "Interface\\AddOns\\" .. addonName .. "\\Textures\\Gloss",
            Width    = 40,
            Height   = 40,
            Color    = { 1, 1, 1, 0.4 },
        },
    })
end
```

!!! tip "Masque hook exit flags"
    Masque installs `hooksecurefunc` hooks on buttons that can't be removed. When a skin is disabled, it sets `_MSQ_Exit_*` flags so the hooks return immediately. Design your own callback systems with similar disable flags.

**Inspired by:** Masque (`Core/Button.lua`, `Core/Group.lua`), BetterBags (`integrations/masque.lua`)

??? failure "Common Mistakes"
    - **Hard-depending on Masque** — always use `LibStub("Masque", true)` with the `true` (silent) flag. Masque is optional.
    - **Not passing region references** — Masque needs explicit references to Icon, Cooldown, Normal textures. Without them, skinning fails silently.
    - **Forgetting to call `Group:ReSkin()`** — after adding buttons dynamically, call `ReSkin()` to apply the current skin.

---

## Pattern 6: Taint Isolation

Prevent your addon's frame operations from tainting Blizzard's secure execution paths.

```lua
local addonName, ns = ...

-- ─── Hidden parent isolation (BetterBags approach) ───────
-- Create item buttons parented to a hidden intermediate frame.
-- This isolates taint from the actual container frame.
function ns.CreateIsolatedButton(name, parent)
    -- Hidden parent absorbs taint from the button's creation
    local isolator = CreateFrame("Button", name .. "Parent")
    local button = CreateFrame("ItemButton", name, isolator,
        "ContainerFrameItemButtonTemplate")

    -- Clear default textures that carry Blizzard state
    button.PushedTexture:SetTexture("")
    button.NormalTexture:SetTexture("")
    if button.BattlepayItemTexture then
        button.BattlepayItemTexture:Hide()
    end
    if button.NewItemTexture then
        button.NewItemTexture:Hide()
    end

    -- Use plain HookScript, NOT library wrappers.
    -- Creating complex objects (contexts, state) during mouse events
    -- before a protected click causes taint.
    button:HookScript("OnMouseDown", function()
        -- Keep it lightweight — no object creation
        button:GetParent():SetAlpha(0.8)
    end)
    button:HookScript("OnMouseUp", function()
        button:GetParent():SetAlpha(1.0)
    end)

    return button
end

-- ─── Frame "kill" without taint (ElvUI approach) ─────────
-- Never call Hide() on Blizzard frames you want to suppress.
-- Instead, reparent to a hidden frame.
ns.UIHider = CreateFrame("Frame", addonName .. "UIHider")
ns.UIHider:Hide()

function ns.KillFrame(frame)
    if not frame then return end

    -- Remove from managed positions so Blizzard layout code
    -- doesn't try to reposition it
    if UIPARENT_MANAGED_FRAME_POSITIONS then
        local name = frame:GetName()
        if name then
            UIPARENT_MANAGED_FRAME_POSITIONS[name] = nil
        end
    end

    -- Unregister all events so it stops reacting
    if frame.UnregisterAllEvents then
        frame:UnregisterAllEvents()
    end

    -- Reparent to hidden frame (avoids calling Hide() directly)
    frame:SetParent(ns.UIHider)
end

-- ─── Deferred state machine (BetterBags approach) ────────
-- Never show/hide bags directly in event handlers.
-- Set flags, process them in OnUpdate or next frame.
ns.bagState = {
    shouldOpen  = false,
    shouldClose = false,
}

function ns.RequestBagOpen()
    ns.bagState.shouldOpen = true
    ns.bagState.shouldClose = false
    -- Defer to next frame via C_Timer
    C_Timer.After(0, ns.ProcessBagState)
end

function ns.RequestBagClose()
    ns.bagState.shouldClose = true
    ns.bagState.shouldOpen = false
    C_Timer.After(0, ns.ProcessBagState)
end

function ns.ProcessBagState()
    if ns.bagState.shouldOpen then
        ns.bagState.shouldOpen = false
        -- Safe to show here — we're outside any protected call stack
        ns.BagFrame:Show()
    elseif ns.bagState.shouldClose then
        ns.bagState.shouldClose = false
        ns.BagFrame:Hide()
    end
end
```

!!! danger "Never touch BankPanel in OnHide"
    BetterBags documents this critical lesson: `OnHide` for frames in `UISpecialFrames` (ESC-to-close) runs in a protected context. Any BankPanel manipulation there causes **persistent taint** that breaks `UseContainerItem()` for ALL containers.

**Inspired by:** BetterBags (`frames/item.lua` hidden parent, `core/hooks.lua` deferred state), ElvUI (`Toolkit.lua` Kill function, UIPARENT clearing)

??? failure "Common Mistakes"
    - **Calling `Hide()` on Blizzard bars** — use `HideBase()` when available (Bartender4 pattern). EditMode overrides `Hide()` and introduces taint.
    - **Creating complex Lua objects in hook callbacks** — even creating tables or calling library functions in `OnMouseDown` can taint the subsequent protected click.
    - **Directly toggling bags in event handlers** — interaction events (banker, merchant) fire in protected context. Defer visibility changes to the next frame.

---

## Pattern 7: ScrollBox Enhancement

Work with Blizzard's modern ScrollBox/DataProvider system (replaced FauxScrollFrame in 10.0).

```lua
local addonName, ns = ...

-- ─── Create a complete ScrollBox list ────────────────────
function ns.CreateScrollList(parent)
    -- 1. Create components
    local scrollBox = CreateFrame("Frame", nil, parent, "WowScrollBoxList")
    scrollBox:SetPoint("TOPLEFT", 4, -4)
    scrollBox:SetPoint("BOTTOMRIGHT", -24, 4)

    local scrollBar = CreateFrame("EventFrame", nil, parent,
        "MinimalScrollBar")
    scrollBar:SetPoint("TOPLEFT", scrollBox, "TOPRIGHT", 4, 0)
    scrollBar:SetPoint("BOTTOMLEFT", scrollBox, "BOTTOMRIGHT", 4, 0)

    -- 2. Configure view with fixed row height
    local view = CreateScrollBoxListLinearView()
    view:SetElementExtent(24)

    -- 3. Element initializer (called when a row becomes visible)
    view:SetElementInitializer("BackdropTemplate", function(frame, data)
        if not frame.text then
            frame.text = frame:CreateFontString(nil, "OVERLAY",
                "GameFontNormal")
            frame.text:SetPoint("LEFT", 8, 0)
        end
        frame.text:SetText(data.name)

        frame:SetScript("OnEnter", function(self)
            GameTooltip:SetOwner(self, "ANCHOR_RIGHT")
            GameTooltip:SetText(data.tooltip or data.name)
            GameTooltip:Show()
        end)
        frame:SetScript("OnLeave", GameTooltip_Hide)
    end)

    -- 4. Element resetter (cleanup when row scrolls out of view)
    view:SetElementResetter(function(frame)
        frame:SetScript("OnEnter", nil)
        frame:SetScript("OnLeave", nil)
    end)

    -- 5. Connect everything
    ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, view)

    -- 6. Auto-hide scrollbar when not needed
    ScrollUtil.AddManagedScrollBarVisibilityBehavior(scrollBox, scrollBar)

    return scrollBox
end

-- ─── Populate with data ──────────────────────────────────
local scrollBox = ns.CreateScrollList(myParentFrame)
local dataProvider = CreateDataProvider()
for i = 1, 100 do
    dataProvider:Insert({ name = "Item " .. i, tooltip = "Details for item " .. i })
end
scrollBox:SetDataProvider(dataProvider)

-- ─── Filter / search ─────────────────────────────────────
function ns.FilterScrollList(scrollBox, allData, query)
    local filtered = {}
    local lowerQuery = query:lower()
    for _, entry in ipairs(allData) do
        if entry.name:lower():find(lowerQuery, 1, true) then
            table.insert(filtered, entry)
        end
    end
    scrollBox:SetDataProvider(CreateDataProvider(filtered))
end

-- ─── Hook an existing Blizzard ScrollBox ─────────────────
-- ElvUI-style: skin rows as they're recycled by the pool
hooksecurefunc(SomeBlizzardScrollBox, "Update", function(self)
    for _, frame in self:EnumerateFrames() do
        if not frame.__skinned then
            ns.StripTextures(frame)
            ns.ApplyBackdrop(frame, 0.05, 0.05, 0.05, 0.9)
            frame.__skinned = true
        end
    end
end)

-- ─── Variable-height elements ────────────────────────────
local variableView = CreateScrollBoxListLinearView()
variableView:SetElementExtentCalculator(function(dataIndex, data)
    if data.isHeader then return 32 end
    return 20
end)
```

**Inspired by:** ElvUI (ScrollBox hooking for dynamic reskinning), BetterBags (virtualized item grid), Blizzard FrameXML (`ScrollUtil`)

??? failure "Common Mistakes"
    - **Using `FauxScrollFrame` or `HybridScrollFrame`** — these are deprecated since 10.0. Use `ScrollBox` + `DataProvider`.
    - **Not implementing a resetter** — without cleanup, recycled frames retain state from previous data, causing visual glitches.
    - **Calling `SetDataProvider` without `RetainScrollPosition`** — the list jumps to the top. Pass `ScrollBoxConstants.RetainScrollPosition` as the second arg when updating data in-place.

---

## Pattern 8: Settings Integration

Persist addon configuration using `SavedVariables` with defaults and migration support.

```lua
local addonName, ns = ...

-- ─── Define defaults ─────────────────────────────────────
ns.DEFAULTS = {
    profile = {
        enabled = true,
        scale = 1.0,
        position = { point = "CENTER", x = 0, y = 0 },
        theme = "dark",
        categories = {},
    },
}

-- ─── SavedVariables initialization (no library) ─────────
-- In your .toc file:
-- ## SavedVariables: MyAddonDB

local db

local initFrame = CreateFrame("Frame")
initFrame:RegisterEvent("ADDON_LOADED")
initFrame:SetScript("OnEvent", function(self, event, addon)
    if addon ~= addonName then return end
    self:UnregisterEvent("ADDON_LOADED")

    -- Merge saved data with defaults (shallow)
    MyAddonDB = MyAddonDB or {}
    for key, default in pairs(ns.DEFAULTS.profile) do
        if MyAddonDB[key] == nil then
            if type(default) == "table" then
                MyAddonDB[key] = CopyTable(default)
            else
                MyAddonDB[key] = default
            end
        end
    end

    db = MyAddonDB
    ns.db = db

    -- Run migrations
    ns.MigrateDB(db)

    -- Initialize the addon
    ns.Initialize()
end)

-- ─── Database migration ──────────────────────────────────
function ns.MigrateDB(db)
    db._version = db._version or 1

    if db._version < 2 then
        -- v2: renamed "color" to "theme"
        if db.color then
            db.theme = db.color
            db.color = nil
        end
        db._version = 2
    end

    if db._version < 3 then
        -- v3: position changed from {x, y} to {point, x, y}
        if db.position and not db.position.point then
            db.position = {
                point = "CENTER",
                x = db.position[1] or 0,
                y = db.position[2] or 0,
            }
        end
        db._version = 3
    end
end

-- ─── AceDB-3.0 approach (BetterBags/ElvUI style) ────────
-- For profile support, character/global split, and callbacks:

--[[
local AceDB = LibStub("AceDB-3.0")

function ns.InitAceDB()
    ns.db = AceDB:New(addonName .. "DB", ns.DEFAULTS, true)

    -- React to profile changes
    ns.db.RegisterCallback(ns, "OnProfileChanged", "RefreshConfig")
    ns.db.RegisterCallback(ns, "OnProfileCopied", "RefreshConfig")
    ns.db.RegisterCallback(ns, "OnProfileReset", "RefreshConfig")
end

function ns.RefreshConfig()
    -- Re-apply all settings from ns.db.profile
    ns.ApplySettings()
end
--]]

-- ─── Settings panel (Blizzard Settings API) ──────────────
function ns.RegisterSettings()
    local category = Settings.RegisterCanvasLayoutCategory(
        ns.CreateSettingsPanel(), addonName
    )
    Settings.RegisterAddOnCategory(category)
end

function ns.CreateSettingsPanel()
    local panel = CreateFrame("Frame")
    panel.name = addonName

    local title = panel:CreateFontString(nil, "OVERLAY", "GameFontNormalLarge")
    title:SetPoint("TOPLEFT", 16, -16)
    title:SetText(addonName .. " Settings")

    local scaleSlider = CreateFrame("Slider", addonName .. "ScaleSlider",
        panel, "OptionsSliderTemplate")
    scaleSlider:SetPoint("TOPLEFT", title, "BOTTOMLEFT", 0, -32)
    scaleSlider:SetMinMaxValues(0.5, 2.0)
    scaleSlider:SetValueStep(0.1)
    scaleSlider:SetValue(ns.db and ns.db.scale or 1.0)
    scaleSlider.Text:SetText("UI Scale")
    scaleSlider:SetScript("OnValueChanged", function(self, value)
        if ns.db then
            ns.db.scale = value
        end
    end)

    return panel
end
```

!!! tip "ElvUI dual-database pattern"
    ElvUI uses two AceDB databases: `ElvDB` for global profiles (shareable between characters) and `ElvPrivateDB` for per-character settings that require `/reload` to change. This prevents expensive full-UI redraws on profile switch.

**Inspired by:** BetterBags (`core/database.lua` with migration), ElvUI (dual AceDB, profile callbacks), Bartender4 (AceDB profiles)

??? failure "Common Mistakes"
    - **Deep-copying defaults on every load** — only fill in missing keys, don't overwrite existing user settings.
    - **No migration system** — changing your settings structure without migration corrupts existing users' saved data.
    - **Using `SavedVariablesPerCharacter` for everything** — users expect settings to follow them across characters. Use per-character only for position data or character-specific toggles.

---

## Pattern 9: Addon Lifecycle

Structured initialization flow used by production addons. Modules start disabled, enable selectively, handle combat transitions.

```lua
local addonName, ns = ...

-- ─── Phase 1: File load (TOC ordering) ───────────────────
-- This runs immediately when the file is loaded.
-- Only define tables, functions, constants. NO frame creation.
ns.modules = {}
ns.moduleOrder = {}

function ns.RegisterModule(name, module)
    ns.modules[name] = module
    table.insert(ns.moduleOrder, name)
    module.enabled = false
end

-- ─── Phase 2: ADDON_LOADED ───────────────────────────────
-- SavedVariables are available. Initialize database.
local mainFrame = CreateFrame("Frame", addonName .. "Core")
mainFrame:RegisterEvent("ADDON_LOADED")
mainFrame:RegisterEvent("PLAYER_LOGIN")
mainFrame:RegisterEvent("PLAYER_REGEN_ENABLED")

mainFrame:SetScript("OnEvent", function(self, event, ...)
    if event == "ADDON_LOADED" then
        local addon = ...
        if addon == addonName then
            -- Initialize saved variables
            ns.InitDB()

            -- Initialize all modules (but don't enable yet)
            for _, name in ipairs(ns.moduleOrder) do
                local module = ns.modules[name]
                if module.OnInitialize then
                    module:OnInitialize()
                end
            end
        end

    elseif event == "PLAYER_LOGIN" then
        -- ─── Phase 3: PLAYER_LOGIN ──────────────────────
        -- World is ready. Safe to create frames and hook Blizzard UI.
        for _, name in ipairs(ns.moduleOrder) do
            local module = ns.modules[name]
            if module.OnEnable and ns.IsModuleEnabled(name) then
                module:OnEnable()
                module.enabled = true
            end
        end

        -- Check addon compatibility
        ns.CheckCompatibility()

    elseif event == "PLAYER_REGEN_ENABLED" then
        -- ─── Combat end: flush deferred work ────────────
        for _, name in ipairs(ns.moduleOrder) do
            local module = ns.modules[name]
            if module.enabled and module.OnCombatEnd then
                module:OnCombatEnd()
            end
        end
    end
end)

-- ─── Example module ──────────────────────────────────────
local BagModule = {}

function BagModule:OnInitialize()
    -- Called during ADDON_LOADED. Set up data structures.
    self.pendingRefresh = false
end

function BagModule:OnEnable()
    -- Called during PLAYER_LOGIN. Safe to hook and create frames.
    hooksecurefunc("ToggleAllBags", function()
        self:OnToggleBags()
    end)
end

function BagModule:OnToggleBags()
    if InCombatLockdown() then
        self.pendingRefresh = true
        return
    end
    -- Do the actual work
    self:RefreshBags()
end

function BagModule:OnCombatEnd()
    if self.pendingRefresh then
        self.pendingRefresh = false
        self:RefreshBags()
    end
end

function BagModule:RefreshBags()
    -- Actual bag refresh logic
end

ns.RegisterModule("Bags", BagModule)

-- ─── Utility: check if module should be enabled ─────────
function ns.IsModuleEnabled(name)
    if ns.db and ns.db.modules then
        return ns.db.modules[name] ~= false
    end
    return true -- enabled by default
end

-- ─── Compatibility checks (BetterBags style) ────────────
function ns.CheckCompatibility()
    if C_AddOns.IsAddOnLoaded("ElvUI") then
        -- ElvUI's bag module conflicts — warn user
        local elvBags = ElvUI and ElvUI[1] and ElvUI[1]:GetModule("Bags")
        if elvBags and elvBags.isEnabled then
            print("|cffff6600" .. addonName .. ":|r ElvUI bags detected. "
                .. "Disable ElvUI's bag module to avoid conflicts.")
        end
    end
end
```

!!! warning "Don't touch BankPanel during initialization"
    BetterBags documents this lesson: accessing `BankPanel` at init time causes **permanent taint**. Defer bank-related setup until the player actually opens the bank (`BANKFRAME_OPENED`).

**Inspired by:** BetterBags (`core/boot.lua` disabled-by-default modules, `core/init.lua` phased enable), ElvUI (`Core.lua` RegisterModule/InitializeModules)

??? failure "Common Mistakes"
    - **Creating frames during file load** — before `ADDON_LOADED`, the game state is incomplete. Frames created too early may reference nil globals.
    - **Not checking `InCombatLockdown()` in `OnEnable`** — if your addon loads mid-combat (e.g., `/reload` during a fight), keybinding and secure frame operations will fail.
    - **Enabling all modules unconditionally** — BetterBags uses `SetDefaultModuleState(false)` and enables selectively. This prevents unused modules from consuming resources.

---

## Pattern 10: Edit Mode Registration

Register custom frames with Blizzard's Edit Mode so users can reposition them alongside standard UI elements.

```lua
local addonName, ns = ...

-- ─── Method 1: EditModeExpanded library ──────────────────
-- The recommended approach for most addons.
-- Requires: EditModeExpanded-1.0 (embedded or external)

local function RegisterWithEditMode()
    local EME = LibStub and LibStub("EditModeExpanded-1.0", true)
    if not EME then return end

    -- Register your main frame
    EME:RegisterFrame(ns.MainFrame, addonName, ns.db)

    -- Optional capabilities
    EME:RegisterResizable(ns.MainFrame)   -- allow resize in Edit Mode
    EME:RegisterHideable(ns.MainFrame)    -- allow hide toggle
    EME:RegisterToggleable(ns.MainFrame)  -- show checkbox
end

-- ─── Method 2: Manual Edit Mode hooks ───────────────────
-- For addons that want full control without the library.

function ns.SetupEditModeIntegration(frame)
    -- Create drag handle (only visible in Edit Mode)
    local handle = CreateFrame("Frame", nil, frame)
    handle:SetAllPoints()
    handle:EnableMouse(true)
    handle:SetMovable(true)
    handle:RegisterForDrag("LeftButton")
    handle:Hide()

    -- Visual indicator during Edit Mode
    local overlay = handle:CreateTexture(nil, "OVERLAY")
    overlay:SetAllPoints()
    overlay:SetColorTexture(0.2, 0.5, 1.0, 0.3)

    local label = handle:CreateFontString(nil, "OVERLAY", "GameFontNormal")
    label:SetPoint("CENTER")
    label:SetText(addonName)

    handle:SetScript("OnDragStart", function(self)
        frame:StartMoving()
    end)
    handle:SetScript("OnDragStop", function(self)
        frame:StopMovingOrSizing()
        -- Save position
        local point, _, relPoint, x, y = frame:GetPoint(1)
        if ns.db then
            ns.db.position = { point = point, x = x, y = y }
        end
    end)

    -- Hook Edit Mode enter/exit
    if EditModeManagerFrame then
        hooksecurefunc(EditModeManagerFrame, "EnterEditMode", function()
            handle:Show()
            frame:SetMovable(true)
        end)
        hooksecurefunc(EditModeManagerFrame, "ExitEditMode", function()
            handle:Hide()
            frame:SetMovable(false)
        end)
    end

    -- Also respond to Bartender4-style bar unlock
    frame.editModeHandle = handle
end

-- ─── Restore saved position ─────────────────────────────
function ns.RestorePosition(frame)
    if not ns.db or not ns.db.position then return end
    local pos = ns.db.position
    frame:ClearAllPoints()
    frame:SetPoint(pos.point or "CENTER", UIParent, pos.point or "CENTER",
        pos.x or 0, pos.y or 0)
end

-- ─── Integration during addon init ──────────────────────
function ns.Initialize()
    ns.MainFrame = CreateFrame("Frame", addonName .. "Main", UIParent,
        "BackdropTemplate")
    ns.MainFrame:SetSize(300, 200)
    ns.MainFrame:SetClampedToScreen(true)

    ns.ApplyBackdrop(ns.MainFrame)
    ns.RestorePosition(ns.MainFrame)
    ns.SetupEditModeIntegration(ns.MainFrame)

    -- Also try the library approach
    RegisterWithEditMode()
end
```

!!! tip "Per-layout positions"
    EditModeExpanded saves positions per Edit Mode layout, matching Blizzard's native behavior. If you implement manual saving, consider keying positions by layout ID via `C_EditMode.GetActiveLayout()`.

**Inspired by:** EditModeExpanded (`EditModeExpanded-1.0` library API), Bartender4 (`Bartender4.lua` Edit Mode hooks, bar unlock/lock)

??? failure "Common Mistakes"
    - **Not setting `ClampedToScreen`** — without it, users can drag frames off-screen with no way to recover.
    - **Saving position on every `OnDragStart`** — save on `OnDragStop` only, to avoid writing incomplete positions.
    - **Hooking `EnterEditMode` before EditModeManagerFrame exists** — it's a Blizzard addon that may not be loaded yet. Check for its existence or wait for `ADDON_LOADED`.

---

## Quick Reference

| Pattern | Key API | When to Use |
|---------|---------|-------------|
| Secure Hooking | `hooksecurefunc` | React to Blizzard function calls |
| Mixin Extension | `CreateFromMixins`, `Mixin` | Compose reusable behavior |
| Frame Skinning | `StripTextures`, `SetBackdrop` | Restyle Blizzard panels |
| Combat Safety | `InCombatLockdown`, `PLAYER_REGEN_ENABLED` | Defer secure frame changes |
| Callback Skinning | `MSQ:Group`, `AddButton` | Let external addons skin your buttons |
| Taint Isolation | Hidden parents, `C_Timer.After(0)` | Prevent taint propagation |
| ScrollBox | `ScrollUtil`, `CreateDataProvider` | Modern scrollable lists |
| Settings | `SavedVariables`, `AceDB-3.0` | Persistent configuration |
| Addon Lifecycle | `ADDON_LOADED` → `PLAYER_LOGIN` | Phased initialization |
| Edit Mode | `EditModeExpanded-1.0` | User-repositionable frames |
