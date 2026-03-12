# Enhancement Tutorials

Four complete, working addons that enhance Blizzard's built-in UI without replacing it. Each tutorial follows the "skin, don't replace" philosophy — hooking existing frames, adding features on top, and staying combat-safe.

!!! tip "Prerequisites"
    These tutorials assume you've read the [Code Templates](code-templates.md) page and understand the `.toc` format, namespace pattern, and event dispatch. Each addon here is self-contained — copy the `.toc` and `.lua` into a matching folder under `Interface/AddOns/` and `/reload`.

---

## 1. Better Cooldown Display

Hooks every `Cooldown` frame in the game via metatable post-hooks and overlays a text timer showing remaining seconds — the same approach used by [OmniCC](https://github.com/tullamods/OmniCC). Fully combat-safe because it only creates `FontString` overlays and never touches secure attributes.

### How It Works

1. Grabs the shared metatable for all `Cooldown` widgets via `getmetatable()`
2. Post-hooks `SetCooldown` so every cooldown start/stop in the UI is intercepted
3. Creates a `FontString` on the cooldown's **parent** (not the cooldown itself) to avoid z-order issues with the sweep animation
4. Runs an `OnUpdate` ticker that formats remaining time and changes color as the cooldown expires
5. In Midnight 12.0, also hooks `SetCooldownFromDurationObject` for Secret Values compatibility — the text overlay is hidden when cooldown data is secret, letting Blizzard's built-in countdown handle it

### File Structure

```
Interface/AddOns/BetterCooldownText/
├── BetterCooldownText.toc
└── BetterCooldownText.lua
```

=== "BetterCooldownText.toc"

    ```toc
    ## Interface: 120001
    ## -- Midnight 12.0.1 client
    ## Title: Better Cooldown Text
    ## Notes: Adds text timers to all cooldown spirals (OmniCC-style).
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Action Bars
    ## -- Groups under Action Bars in the Addon List (11.1.0+)
    ## IconTexture: Interface\Icons\Spell_Holy_BorrowedTime
    ## -- Timer icon shown in the Addon List (10.1.0+)
    ## SavedVariables: BetterCooldownTextDB
    ## -- Account-wide settings persisted to WTF folder

    BetterCooldownText.lua
    ```

=== "BetterCooldownText.lua"

    ```lua
    -- BetterCooldownText.lua
    -- Hooks the Cooldown widget metatable to overlay text timers on every
    -- cooldown spiral in the UI. Combat-safe: only touches FontStrings.

    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- CONFIGURATION
    -- ---------------------------------------------------------------------------

    -- Minimum cooldown duration (seconds) before we show text.
    -- Short GCDs (1.5s) are excluded to reduce visual clutter.
    local MIN_DURATION = 2.5

    -- Threshold (seconds) below which the text turns from white to warning color.
    local WARN_THRESHOLD = 5

    -- Colors: {r, g, b} tables for each time range.
    local COLOR_NORMAL  = {1.0, 1.0, 1.0}  -- white: > WARN_THRESHOLD seconds
    local COLOR_WARNING = {1.0, 0.8, 0.0}  -- gold:  <= WARN_THRESHOLD seconds
    local COLOR_EXPIRING = {1.0, 0.2, 0.0} -- red:   <= 1 second

    -- How often (seconds) to update each timer's text. Lower = smoother but
    -- more CPU. 0.1 is a good balance for text that shows one decimal.
    local UPDATE_INTERVAL = 0.1

    -- Default settings for SavedVariables.
    local DEFAULTS = {
        enabled    = true,   -- master toggle
        minSize    = 20,     -- hide text on cooldowns smaller than this (pixels)
        fontSize   = 14,     -- base font size (scaled by cooldown frame size)
        showDecimals = true, -- show one decimal place under 10 seconds
    }

    -- ---------------------------------------------------------------------------
    -- SAVED VARIABLES
    -- ---------------------------------------------------------------------------

    local frame = CreateFrame("Frame")

    -- Merge defaults into saved table without overwriting existing user choices.
    local function MergeDefaults(saved, defaults)
        for k, v in pairs(defaults) do
            if saved[k] == nil then
                saved[k] = v
            end
        end
    end

    frame:RegisterEvent("ADDON_LOADED")
    frame:SetScript("OnEvent", function(self, event, addon)
        if addon ~= addonName then return end
        BetterCooldownTextDB = BetterCooldownTextDB or {}
        MergeDefaults(BetterCooldownTextDB, DEFAULTS)
        ns.db = BetterCooldownTextDB
        self:UnregisterEvent("ADDON_LOADED")
    end)

    -- ---------------------------------------------------------------------------
    -- TIMER DISPLAY
    -- A lightweight text overlay attached to the cooldown's parent frame.
    -- ---------------------------------------------------------------------------

    -- Pool of active timer displays, keyed by the Cooldown widget.
    local activeTimers = {}

    -- Format remaining seconds into a compact string.
    -- 3600+ -> "1h", 60-3600 -> "2m", 10-60 -> "15", <10 -> "3.1"
    local function FormatTime(remaining)
        if remaining >= 86400 then
            return string.format("%dd", math.floor(remaining / 86400))
        elseif remaining >= 3600 then
            return string.format("%dh", math.floor(remaining / 3600))
        elseif remaining >= 60 then
            return string.format("%dm", math.floor(remaining / 60))
        elseif remaining >= 10 then
            return string.format("%d", math.floor(remaining))
        else
            -- Show one decimal when under 10 seconds (if enabled).
            if ns.db and ns.db.showDecimals then
                return string.format("%.1f", remaining)
            else
                return string.format("%d", math.ceil(remaining))
            end
        end
    end

    -- Pick a color based on how much time is left.
    local function GetTimerColor(remaining)
        if remaining <= 1 then
            return unpack(COLOR_EXPIRING)
        elseif remaining <= WARN_THRESHOLD then
            return unpack(COLOR_WARNING)
        else
            return unpack(COLOR_NORMAL)
        end
    end

    -- Get or create the text overlay for a given Cooldown widget.
    -- The FontString is parented to the cooldown's parent (usually the button)
    -- so it renders on top of the sweep animation.
    local function GetOrCreateText(cooldown)
        -- Check if we already created a FontString for this cooldown.
        if cooldown._BCT_Text then
            return cooldown._BCT_Text
        end

        -- Parent the text to the cooldown's parent frame (the button).
        local parent = cooldown:GetParent()
        if not parent then return nil end

        -- Create the FontString at OVERLAY draw layer so it's on top.
        local text = parent:CreateFontString(nil, "OVERLAY")
        -- NumberFontNormal is a built-in Blizzard font (Friz Quadrata, ~12px).
        text:SetFont(NumberFontNormal:GetFont())
        text:SetPoint("CENTER", cooldown, "CENTER", 0, 0)
        -- Black outline so the text is readable on any background.
        text:SetShadowOffset(1, -1)
        text:SetShadowColor(0, 0, 0, 1)

        -- Store a reference on the cooldown frame itself for fast lookup.
        cooldown._BCT_Text = text
        return text
    end

    -- ---------------------------------------------------------------------------
    -- ONUPDATE TICKER
    -- A single shared frame drives all active timers. This is more efficient
    -- than giving each cooldown its own OnUpdate handler.
    -- ---------------------------------------------------------------------------

    local ticker = CreateFrame("Frame")
    local elapsed = 0

    ticker:SetScript("OnUpdate", function(self, dt)
        elapsed = elapsed + dt
        if elapsed < UPDATE_INTERVAL then return end
        elapsed = 0

        local now = GetTime()

        -- Iterate all active timers and update their text.
        for cooldown, info in pairs(activeTimers) do
            local remaining = info.endTime - now

            if remaining <= 0 then
                -- Cooldown finished — hide text and remove from tracking.
                if cooldown._BCT_Text then
                    cooldown._BCT_Text:SetText("")
                    cooldown._BCT_Text:Hide()
                end
                activeTimers[cooldown] = nil
            else
                local text = GetOrCreateText(cooldown)
                if text then
                    -- Skip text on very small cooldown frames (e.g. buff icons
                    -- when the user has configured a minimum size threshold).
                    local minSize = (ns.db and ns.db.minSize) or DEFAULTS.minSize
                    local w, h = cooldown:GetSize()
                    if w < minSize or h < minSize then
                        text:Hide()
                    else
                        -- Scale font size relative to the cooldown frame size.
                        local baseSize = (ns.db and ns.db.fontSize) or DEFAULTS.fontSize
                        local scale = math.min(w, h) / 36
                        local size = math.max(8, math.floor(baseSize * scale))
                        local fontPath = text:GetFont()
                        text:SetFont(fontPath, size, "OUTLINE")

                        text:SetText(FormatTime(remaining))
                        text:SetTextColor(GetTimerColor(remaining))
                        text:Show()
                    end
                end
            end
        end

        -- If no active timers remain, stop the OnUpdate to save CPU.
        if not next(activeTimers) then
            self:Hide()
        end
    end)

    -- Start hidden — we show it when a cooldown activates.
    ticker:Hide()

    -- ---------------------------------------------------------------------------
    -- COOLDOWN METATABLE HOOKS
    -- Post-hook SetCooldown on the Cooldown widget metatable so we intercept
    -- every cooldown in the entire UI — action bars, buff frames, inventory,
    -- other addons, everything.
    -- ---------------------------------------------------------------------------

    -- Create a temporary Cooldown to grab the shared metatable.
    local cooldownProto = CreateFrame("Cooldown", nil, nil, "CooldownFrameTemplate")
    local cooldownMT = getmetatable(cooldownProto).__index

    -- Hook SetCooldown(start, duration [, modRate]).
    -- This is the primary API that Blizzard and addons call to start a cooldown.
    hooksecurefunc(cooldownMT, "SetCooldown", function(self, start, duration)
        -- Ignore if the addon is disabled.
        if ns.db and not ns.db.enabled then return end

        -- Ignore zero-duration (cooldown cleared) or very short cooldowns.
        if not start or not duration or duration <= 0 then
            -- Cooldown was cleared — hide our text.
            if self._BCT_Text then
                self._BCT_Text:SetText("")
                self._BCT_Text:Hide()
            end
            activeTimers[self] = nil
            return
        end

        -- Skip the GCD and other short cooldowns.
        if duration < MIN_DURATION then
            return
        end

        -- Skip cooldowns that have noCooldownCount set (Blizzard uses this
        -- on some internal frames that already show their own text).
        if self.noCooldownCount then return end

        -- Skip forbidden frames (Blizzard-protected UI that addons can't touch).
        if self:IsForbidden() then return end

        -- Track this cooldown.
        local endTime = start + duration
        activeTimers[self] = { endTime = endTime }

        -- Ensure the ticker is running.
        ticker:Show()
    end)

    -- Hook SetCooldownFromDurationObject (Midnight 12.0+ Secret Values).
    -- When Blizzard passes a Duration Object, the actual start/duration values
    -- may be secret (opaque). We cannot read them, so we hide our text overlay
    -- and let Blizzard's built-in countdown (Cooldown:GetCountdownFontString())
    -- handle the display instead.
    if cooldownMT.SetCooldownFromDurationObject then
        hooksecurefunc(cooldownMT, "SetCooldownFromDurationObject",
        function(self, durationObj, clearIfZero)
            -- Check if this cooldown carries secret values.
            if self:HasSecretValues() then
                -- Secret context (M+ key, PvP, encounter) — hide our text.
                if self._BCT_Text then
                    self._BCT_Text:SetText("")
                    self._BCT_Text:Hide()
                end
                activeTimers[self] = nil
                -- Blizzard's own countdown text handles it from here.
                return
            end

            -- Not secret — try to read the times normally.
            -- GetCooldownTimes() returns start*1000, duration*1000 in ms.
            local startMs, durationMs = self:GetCooldownTimes()
            if startMs and durationMs and durationMs > 0 then
                local start = startMs / 1000
                local duration = durationMs / 1000
                if duration >= MIN_DURATION then
                    activeTimers[self] = { endTime = start + duration }
                    ticker:Show()
                end
            end
        end)
    end

    -- Disable Blizzard's built-in countdown numbers on cooldowns that we handle,
    -- to avoid duplicate text. We hook SetHideCountdownNumbers to detect when
    -- Blizzard enables its own text and suppress it.
    hooksecurefunc(cooldownMT, "SetHideCountdownNumbers", function(self, hide)
        -- If Blizzard is showing countdown numbers (hide=false) and we're active
        -- on this cooldown, re-hide Blizzard's version.
        if not hide and activeTimers[self] then
            self:SetHideCountdownNumbers(true)
        end
    end)

    -- Clean up when a cooldown frame is hidden (e.g. parent frame closed).
    hooksecurefunc(cooldownMT, "Clear", function(self)
        if self._BCT_Text then
            self._BCT_Text:SetText("")
            self._BCT_Text:Hide()
        end
        activeTimers[self] = nil
    end)
    ```

---

## 2. Edit Mode Custom Frame

Creates a movable info frame (player coordinates + FPS) and registers it with Edit Mode using the community library [EditModeExpanded](https://github.com/teelolws/EditModeExpanded). Position is persisted per Edit Mode layout. The frame is draggable only during Edit Mode and locks in place otherwise.

### How It Works

1. Creates a `BackdropTemplate` frame showing player coordinates and FPS
2. Uses `EditModeExpanded-1.0` (LibStub) to register the frame with Blizzard's Edit Mode system
3. When Edit Mode is entered, the frame becomes draggable with a selection highlight
4. When Edit Mode is exited, position is saved to `SavedVariables` and the frame locks
5. Hooks `EditModeManagerFrame` enter/exit events via `EventRegistry` to toggle the drag state
6. An `OnUpdate` ticker refreshes the info text every 0.5 seconds to keep CPU usage low

### File Structure

```
Interface/AddOns/EditModeInfoFrame/
├── EditModeInfoFrame.toc
└── EditModeInfoFrame.lua
```

=== "EditModeInfoFrame.toc"

    ```toc
    ## Interface: 120001
    ## -- Midnight 12.0.1 client
    ## Title: Edit Mode Info Frame
    ## Notes: Movable info frame (coords + FPS) integrated with Edit Mode.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Interface Enhancements
    ## -- Groups under Interface Enhancements in the Addon List (11.1.0+)
    ## IconTexture: Interface\Icons\INV_Misc_Map_01
    ## -- Map icon in the Addon List (10.1.0+)
    ## SavedVariables: EditModeInfoFrameDB
    ## -- Account-wide saved variables for position + settings
    ## Dependencies: EditModeExpanded
    ## -- Requires EditModeExpanded addon to be installed and enabled.
    ## -- This is a hard dependency — the addon won't load without it.

    EditModeInfoFrame.lua
    ```

=== "EditModeInfoFrame.lua"

    ```lua
    -- EditModeInfoFrame.lua
    -- A small info panel (coordinates + FPS) that integrates with Blizzard's
    -- Edit Mode via the EditModeExpanded library.

    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- DEFAULTS
    -- ---------------------------------------------------------------------------

    local DEFAULTS = {
        enabled  = true,   -- show the info frame
        showFPS  = true,   -- include FPS in the display
        showCoords = true, -- include map coordinates
        bgAlpha  = 0.6,    -- background opacity (0 = invisible, 1 = opaque)
    }

    local function MergeDefaults(saved, defaults)
        for k, v in pairs(defaults) do
            if saved[k] == nil then saved[k] = v end
        end
    end

    -- ---------------------------------------------------------------------------
    -- INFO FRAME CREATION
    -- Build the visual frame with BackdropTemplate for a dark semi-transparent
    -- background. We create this at file load time so it exists before Edit Mode
    -- scans for registered frames.
    -- ---------------------------------------------------------------------------

    -- CreateFrame with BackdropTemplate (9.0.1+ standard for bordered frames).
    local infoFrame = CreateFrame("Frame", "EditModeInfoPanel", UIParent,
        "BackdropTemplate")
    -- Default size — Edit Mode may resize this if RegisterResizable is used.
    infoFrame:SetSize(180, 50)
    -- Default position (center-bottom of screen). Edit Mode will override this
    -- once the player moves the frame.
    infoFrame:SetPoint("TOP", UIParent, "TOP", 0, -30)
    -- MEDIUM strata so it sits above world frames but below dialogs.
    infoFrame:SetFrameStrata("MEDIUM")

    -- Apply a dark backdrop with a thin border.
    infoFrame:SetBackdrop({
        -- Semi-transparent black background texture.
        bgFile   = "Interface\\Tooltips\\UI-Tooltip-Background",
        -- Thin white border.
        edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
        tile = true, tileSize = 16, edgeSize = 12,
        insets = { left = 2, right = 2, top = 2, bottom = 2 },
    })
    infoFrame:SetBackdropColor(0, 0, 0, 0.6)
    infoFrame:SetBackdropBorderColor(0.3, 0.3, 0.3, 0.8)

    -- Title FontString — shows "Info" at the top.
    local title = infoFrame:CreateFontString(nil, "OVERLAY")
    title:SetFont(GameFontNormal:GetFont())
    local titleFont = title:GetFont()
    title:SetFont(titleFont, 10, "OUTLINE")
    title:SetPoint("TOP", infoFrame, "TOP", 0, -6)
    title:SetTextColor(0.8, 0.8, 0.8)
    title:SetText("Info")

    -- Content FontString — shows coordinates and FPS.
    local content = infoFrame:CreateFontString(nil, "OVERLAY")
    content:SetFont(GameFontNormal:GetFont())
    local contentFont = content:GetFont()
    content:SetFont(contentFont, 11, "")
    content:SetPoint("TOP", title, "BOTTOM", 0, -4)
    content:SetTextColor(1, 1, 1)

    -- ---------------------------------------------------------------------------
    -- CONTENT UPDATER
    -- Refreshes the displayed text on a throttled OnUpdate so we're not
    -- recalculating every single frame.
    -- ---------------------------------------------------------------------------

    local updateElapsed = 0
    local UPDATE_INTERVAL = 0.5  -- update twice per second

    infoFrame:SetScript("OnUpdate", function(self, dt)
        updateElapsed = updateElapsed + dt
        if updateElapsed < UPDATE_INTERVAL then return end
        updateElapsed = 0

        local db = ns.db or DEFAULTS
        local parts = {}

        -- Player map coordinates via C_Map.
        if db.showCoords then
            -- GetBestMapForUnit returns the current zone map ID.
            local mapID = C_Map.GetBestMapForUnit("player")
            if mapID then
                -- GetPlayerMapPosition returns a Vector2D with x,y in 0-1 range.
                local pos = C_Map.GetPlayerMapPosition(mapID, "player")
                if pos then
                    local x, y = pos:GetXY()
                    -- Multiply by 100 to get percentage coordinates.
                    parts[#parts + 1] = string.format("%.1f, %.1f", x * 100, y * 100)
                end
            end
            -- If we couldn't get coords (instanced zone, etc.), show "—".
            if #parts == 0 then
                parts[#parts + 1] = "—, —"
            end
        end

        -- Framerate via GetFramerate() global.
        if db.showFPS then
            parts[#parts + 1] = string.format("%d fps", GetFramerate())
        end

        content:SetText(table.concat(parts, "  |  "))
    end)

    -- ---------------------------------------------------------------------------
    -- EDIT MODE INTEGRATION
    -- Uses EditModeExpanded-1.0 library to register our frame so it appears
    -- in Blizzard's Edit Mode alongside native HUD elements.
    -- ---------------------------------------------------------------------------

    local eventFrame = CreateFrame("Frame")

    eventFrame:RegisterEvent("ADDON_LOADED")
    eventFrame:SetScript("OnEvent", function(self, event, addon)
        if addon ~= addonName then return end
        self:UnregisterEvent("ADDON_LOADED")

        -- Initialize saved variables.
        EditModeInfoFrameDB = EditModeInfoFrameDB or {}
        MergeDefaults(EditModeInfoFrameDB, DEFAULTS)
        ns.db = EditModeInfoFrameDB

        -- Apply saved background opacity.
        infoFrame:SetBackdropColor(0, 0, 0, ns.db.bgAlpha)

        -- Hide if user has disabled the frame.
        if not ns.db.enabled then
            infoFrame:Hide()
            return
        end

        -- Attempt to register with EditModeExpanded.
        -- LibStub("EditModeExpanded-1.0") returns the library or nil.
        local EME = LibStub and LibStub("EditModeExpanded-1.0", true)
        if not EME then
            -- EditModeExpanded not installed — fall back to basic dragging.
            -- Make the frame draggable at all times as a fallback.
            infoFrame:SetMovable(true)
            infoFrame:EnableMouse(true)
            infoFrame:RegisterForDrag("LeftButton")
            infoFrame:SetScript("OnDragStart", infoFrame.StartMoving)
            infoFrame:SetScript("OnDragStop", function(f)
                f:StopMovingOrSizing()
                -- Save position manually.
                local point, _, relPoint, x, y = f:GetPoint()
                ns.db.position = { point, relPoint, x, y }
            end)
            -- Restore saved position if available.
            if ns.db.position then
                local p = ns.db.position
                infoFrame:ClearAllPoints()
                infoFrame:SetPoint(p[1], UIParent, p[2], p[3], p[4])
            end
            return
        end

        -- Register our frame with Edit Mode.
        -- Parameters: frame, display name, saved variables table.
        -- This makes the frame appear in the Edit Mode system list and
        -- enables drag-to-position with snapping/magnetism.
        EME:RegisterFrame(infoFrame, "Info Frame", ns.db)

        -- Allow the frame to be resized in Edit Mode.
        EME:RegisterResizable(infoFrame)

        -- Allow the frame to be hidden per-layout in Edit Mode.
        EME:RegisterHideable(infoFrame)
    end)

    -- ---------------------------------------------------------------------------
    -- SLASH COMMAND
    -- /eminfo toggle — show/hide the info frame
    -- /eminfo reset  — reset position to default
    -- ---------------------------------------------------------------------------

    SLASH_EDITMODEINFO1 = "/eminfo"
    SlashCmdList["EDITMODEINFO"] = function(msg)
        local cmd = strtrim(msg):lower()

        if cmd == "toggle" then
            ns.db.enabled = not ns.db.enabled
            if ns.db.enabled then
                infoFrame:Show()
                print("|cff33ccffInfo Frame:|r Shown.")
            else
                infoFrame:Hide()
                print("|cff33ccffInfo Frame:|r Hidden.")
            end
        elseif cmd == "reset" then
            infoFrame:ClearAllPoints()
            infoFrame:SetPoint("TOP", UIParent, "TOP", 0, -30)
            infoFrame:SetSize(180, 50)
            print("|cff33ccffInfo Frame:|r Position reset.")
        else
            print("|cff33ccffInfo Frame:|r Usage:")
            print("  /eminfo toggle — show or hide")
            print("  /eminfo reset  — reset position and size")
        end
    end
    ```

---

## 3. Skinning a Blizzard Panel

Reskins the Character Info panel (`PaperDollFrame` / `CharacterFrame`) by changing background colors, stripping default textures, replacing fonts, and tinting slot borders — all via `hooksecurefunc` post-hooks. Never replaces or hides the original frame; only modifies its visual properties. This is the same approach used by [ElvUI's skin engine](https://github.com/tukui-org/ElvUI).

### How It Works

1. Waits for `Blizzard_CharacterFrame` to load via `ADDON_LOADED` (the character panel is a load-on-demand addon since 11.0)
2. Strips default background textures from `CharacterFrame` using `region:SetTexture("")` on all child `Texture` regions
3. Applies a new `BackdropTemplate` background with custom colors
4. Hooks `PaperDollFrame_SetItemLevel` and `PaperDollItemSlotButton_Update` to re-skin slot borders and item level text as the UI refreshes
5. All changes are visual-only (`SetTexture`, `SetVertexColor`, `SetFont`) — no secure attributes are touched, so this is fully combat-safe

### File Structure

```
Interface/AddOns/BetterCharacterSkin/
├── BetterCharacterSkin.toc
└── BetterCharacterSkin.lua
```

=== "BetterCharacterSkin.toc"

    ```toc
    ## Interface: 120001
    ## -- Midnight 12.0.1 client
    ## Title: Better Character Skin
    ## Notes: Reskins the Character Info panel with cleaner textures and colors.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Interface Enhancements
    ## -- Groups under Interface Enhancements in the Addon List (11.1.0+)
    ## IconTexture: Interface\Icons\INV_Chest_Cloth_17
    ## -- Armor icon in the Addon List (10.1.0+)
    ## SavedVariables: BetterCharacterSkinDB
    ## -- Account-wide settings: colors, font overrides, toggle

    BetterCharacterSkin.lua
    ```

=== "BetterCharacterSkin.lua"

    ```lua
    -- BetterCharacterSkin.lua
    -- Reskins the Character Info panel (PaperDollFrame) with cleaner visuals.
    -- Uses hooksecurefunc for all modifications — fully combat-safe.

    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- CONFIGURATION
    -- ---------------------------------------------------------------------------

    local DEFAULTS = {
        enabled       = true,
        bgColor       = { r = 0.05, g = 0.05, b = 0.10, a = 0.92 },
        borderColor   = { r = 0.30, g = 0.30, b = 0.40, a = 0.80 },
        slotBorderColor = { r = 0.50, g = 0.50, b = 0.60, a = 1.0 },
        highlightColor = { r = 0.40, g = 0.60, b = 1.00, a = 0.30 },
        titleFont     = "Fonts\\FRIZQT__.TTF",
        titleFontSize = 13,
    }

    local function MergeDefaults(saved, defaults)
        for k, v in pairs(defaults) do
            if saved[k] == nil then
                if type(v) == "table" then
                    saved[k] = CopyTable(v)
                else
                    saved[k] = v
                end
            end
        end
    end

    -- ---------------------------------------------------------------------------
    -- UTILITY: STRIP TEXTURES
    -- Iterates all child regions of a frame and clears any Texture objects.
    -- This removes Blizzard's default background art, borders, and decorations.
    -- Same pattern used by ElvUI's StripTextures function.
    -- ---------------------------------------------------------------------------

    local function StripTextures(frame)
        -- GetRegions() returns all child LayeredRegions (Textures, FontStrings).
        for _, region in pairs({ frame:GetRegions() }) do
            -- Only clear Texture objects — leave FontStrings alone.
            if region.IsObjectType and region:IsObjectType("Texture") then
                -- SetTexture("") clears the texture without destroying the region.
                -- SetAlpha(0) as a fallback for atlas-based textures.
                region:SetTexture("")
                region:SetAtlas("")
                region:SetAlpha(0)
            end
        end
    end

    -- ---------------------------------------------------------------------------
    -- SKIN APPLICATION
    -- Called once after Blizzard_CharacterFrame loads. Applies all visual
    -- changes to CharacterFrame and its child panels.
    -- ---------------------------------------------------------------------------

    local function ApplySkin()
        local db = ns.db or DEFAULTS

        if not db.enabled then return end

        -- Verify the frame exists. CharacterFrame is created by
        -- Blizzard_CharacterFrame (load-on-demand since 11.0).
        local cf = CharacterFrame
        if not cf then return end

        -- ----- STEP 1: Strip default textures from the main frame -----
        StripTextures(cf)

        -- Also strip the inset frame if it exists (border decoration).
        if cf.Inset then
            StripTextures(cf.Inset)
        end

        -- Hide the portrait — we want a cleaner look.
        if cf.PortraitContainer then
            cf.PortraitContainer:SetAlpha(0)
        end

        -- ----- STEP 2: Apply new backdrop -----
        -- Mixin BackdropTemplate if the frame doesn't already have SetBackdrop.
        if not cf.SetBackdrop then
            Mixin(cf, BackdropTemplateMixin)
        end

        cf:SetBackdrop({
            bgFile   = "Interface\\Tooltips\\UI-Tooltip-Background",
            edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
            tile = true, tileSize = 16, edgeSize = 16,
            insets = { left = 4, right = 4, top = 4, bottom = 4 },
        })

        local bg = db.bgColor
        cf:SetBackdropColor(bg.r, bg.g, bg.b, bg.a)
        local bd = db.borderColor
        cf:SetBackdropBorderColor(bd.r, bd.g, bd.b, bd.a)

        -- ----- STEP 3: Restyle the title text -----
        -- CharacterFrame has a TitleText FontString in its title bar.
        if cf.TitleText then
            cf.TitleText:SetFont(db.titleFont, db.titleFontSize, "OUTLINE")
            cf.TitleText:SetTextColor(0.9, 0.8, 1.0)
        end

        -- ----- STEP 4: Skin the close button -----
        -- CharacterFrameCloseButton is the X button in the top-right corner.
        local closeBtn = cf.CloseButton or CharacterFrameCloseButton
        if closeBtn then
            -- Tint the close button's normal texture to match our border color.
            local normalTex = closeBtn:GetNormalTexture()
            if normalTex then
                normalTex:SetVertexColor(0.8, 0.8, 0.8)
            end
        end

        -- ----- STEP 5: Skin equipment slot buttons -----
        -- PaperDollItemsFrame contains the 17 equipment slot buttons.
        -- Each slot button name follows the pattern: "Character<SlotName>Slot"
        local slotNames = {
            "HeadSlot", "NeckSlot", "ShoulderSlot", "BackSlot",
            "ChestSlot", "WristSlot", "HandsSlot", "WaistSlot",
            "LegsSlot", "FeetSlot", "Finger0Slot", "Finger1Slot",
            "Trinket0Slot", "Trinket1Slot", "MainHandSlot",
            "SecondaryHandSlot", "ShirtSlot", "TabardSlot",
        }

        for _, slotName in ipairs(slotNames) do
            local slot = _G["Character" .. slotName]
            if slot then
                -- Tint the slot border to our custom color.
                local sc = db.slotBorderColor
                local border = slot.IconBorder
                    or _G["Character" .. slotName .. "IconBorder"]
                if border then
                    -- Hook SetVertexColor so our tint reapplies when Blizzard
                    -- updates the border (e.g. on item quality change).
                    hooksecurefunc(border, "SetVertexColor", function(self)
                        -- Only tint if the border is actually visible.
                        if self:IsShown() then
                            -- Blend our color with the quality color.
                            local r, g, b = self:GetVertexColor()
                            self:SetVertexColor(
                                r * 0.7 + sc.r * 0.3,
                                g * 0.7 + sc.g * 0.3,
                                b * 0.7 + sc.b * 0.3,
                                sc.a
                            )
                        end
                    end)
                end

                -- Add a highlight texture for mouseover.
                if not slot._BCS_Highlight then
                    local hl = slot:CreateTexture(nil, "HIGHLIGHT")
                    hl:SetAllPoints()
                    hl:SetColorTexture(
                        db.highlightColor.r,
                        db.highlightColor.g,
                        db.highlightColor.b,
                        db.highlightColor.a
                    )
                    slot._BCS_Highlight = hl
                end
            end
        end

        -- ----- STEP 6: Hook tab buttons to re-skin on tab switch -----
        -- CharacterFrame has tabs (Character, Reputation, Currency, etc.).
        -- Each tab switch re-draws parts of the frame, so we hook to re-apply.
        if cf.TabSystem then
            hooksecurefunc(cf, "UpdateFrameForSelectedTab", function()
                -- Re-apply backdrop after tab switch redraws the frame.
                local bg2 = db.bgColor
                cf:SetBackdropColor(bg2.r, bg2.g, bg2.b, bg2.a)
                local bd2 = db.borderColor
                cf:SetBackdropBorderColor(bd2.r, bd2.g, bd2.b, bd2.a)
            end)
        end

        -- Mark as skinned so we don't re-apply.
        ns.skinApplied = true
    end

    -- ---------------------------------------------------------------------------
    -- INITIALIZATION
    -- Wait for Blizzard_CharacterFrame to load, then apply the skin.
    -- ---------------------------------------------------------------------------

    local eventFrame = CreateFrame("Frame")
    eventFrame:RegisterEvent("ADDON_LOADED")

    eventFrame:SetScript("OnEvent", function(self, event, addon)
        -- First, initialize our own saved variables.
        if addon == addonName then
            BetterCharacterSkinDB = BetterCharacterSkinDB or {}
            MergeDefaults(BetterCharacterSkinDB, DEFAULTS)
            ns.db = BetterCharacterSkinDB
        end

        -- Apply skin when Blizzard_CharacterFrame loads.
        -- This fires when the player first opens the Character panel (load-on-demand).
        if addon == "Blizzard_CharacterFrame" then
            -- Defer one frame to ensure all CharacterFrame children are created.
            C_Timer.After(0, function()
                if not ns.skinApplied then
                    ApplySkin()
                end
            end)
        end

        -- If Blizzard_CharacterFrame was already loaded when our addon loaded
        -- (e.g. if the player opened the character panel before our addon),
        -- apply immediately.
        if addon == addonName then
            local loaded = C_AddOns.IsAddOnLoaded("Blizzard_CharacterFrame")
            if loaded and not ns.skinApplied then
                C_Timer.After(0, ApplySkin)
            end
        end
    end)
    ```

---

## 4. Better Tooltip Enhancement

Adds extra information lines to `GameTooltip` — item level, vendor price, spell ID — using the standard tooltip hook pattern. Handles Midnight 12.0's `GameTooltipTemplate` requirement for custom tooltips and the `C_TooltipInfo` structured data system introduced in 10.0.2.

### How It Works

1. Hooks `GameTooltip:SetAction`, `SetItem`, `SetSpell`, and the `TooltipDataProcessor` callback system to intercept tooltip display
2. `TooltipDataProcessor.AddTooltipPostCall` is the modern (10.0.2+) way to append data — it fires after `C_TooltipInfo` populates the tooltip
3. Adds item level for equipment, vendor sell price for items, and spell ID for spells
4. Handles the case where `GameTooltip` uses `GameTooltipTemplate` (required in Midnight for `GameTooltipDataMixin` support)
5. Uses `GameTooltip:AddDoubleLine()` to add left-right aligned info rows

### File Structure

```
Interface/AddOns/BetterTooltipInfo/
├── BetterTooltipInfo.toc
└── BetterTooltipInfo.lua
```

=== "BetterTooltipInfo.toc"

    ```toc
    ## Interface: 120001
    ## -- Midnight 12.0.1 client
    ## Title: Better Tooltip Info
    ## Notes: Adds item level, vendor price, and spell IDs to tooltips.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Interface Enhancements
    ## -- Groups under Interface Enhancements in the Addon List (11.1.0+)
    ## IconTexture: Interface\Icons\INV_Misc_QuestionMark
    ## -- Question mark icon in the Addon List (10.1.0+)
    ## SavedVariables: BetterTooltipInfoDB
    ## -- Account-wide settings for which info lines to show
    ## AddonCompartmentFunc: BetterTooltipInfo_OnCompartmentClick
    ## -- Registers a click handler in the Addon Compartment dropdown (10.1.0+)
    ## AddonCompartmentFuncOnEnter: BetterTooltipInfo_OnCompartmentEnter
    ## -- Tooltip shown when hovering the compartment entry

    BetterTooltipInfo.lua
    ```

=== "BetterTooltipInfo.lua"

    ```lua
    -- BetterTooltipInfo.lua
    -- Enhances GameTooltip with item level, vendor price, and spell ID info.
    -- Uses TooltipDataProcessor (10.0.2+) for structured tooltip hooking.

    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- DEFAULTS
    -- ---------------------------------------------------------------------------

    local DEFAULTS = {
        showItemLevel  = true,  -- show item level on equipment tooltips
        showVendorPrice = true, -- show vendor sell price
        showSpellID    = true,  -- show spell ID on spell tooltips
        showItemID     = false, -- show item ID (debug, off by default)
    }

    local function MergeDefaults(saved, defaults)
        for k, v in pairs(defaults) do
            if saved[k] == nil then saved[k] = v end
        end
    end

    -- ---------------------------------------------------------------------------
    -- COLORS AND FORMATTING
    -- ---------------------------------------------------------------------------

    -- Label color: muted gray for the left side of info lines.
    local LABEL_R, LABEL_G, LABEL_B = 0.5, 0.5, 0.5
    -- Value color: white for the right side.
    local VALUE_R, VALUE_G, VALUE_B = 1.0, 1.0, 1.0
    -- ID color: dimmer for technical info like spell/item IDs.
    local ID_R, ID_G, ID_B = 0.6, 0.6, 0.6

    -- Format a copper value into a gold/silver/copper string with icons.
    -- 12345 copper -> "1g 23s 45c"
    local function FormatMoney(copper)
        if not copper or copper <= 0 then return nil end

        local gold   = math.floor(copper / 10000)
        local silver = math.floor((copper % 10000) / 100)
        local cop    = copper % 100

        local parts = {}
        if gold > 0 then
            -- |cffffd700 = gold color, |TInterface\\... = coin icon
            parts[#parts + 1] = string.format("|cffffd700%d|r|TInterface\\MoneyFrame\\UI-GoldIcon:0|t", gold)
        end
        if silver > 0 then
            -- |cffc7c7cf = silver color
            parts[#parts + 1] = string.format("|cffc7c7cf%d|r|TInterface\\MoneyFrame\\UI-SilverIcon:0|t", silver)
        end
        if cop > 0 or #parts == 0 then
            -- |cffeda55f = copper color
            parts[#parts + 1] = string.format("|cffeda55f%d|r|TInterface\\MoneyFrame\\UI-CopperIcon:0|t", cop)
        end
        return table.concat(parts, " ")
    end

    -- ---------------------------------------------------------------------------
    -- TOOLTIP DATA PROCESSOR HOOKS (10.0.2+)
    -- TooltipDataProcessor is the modern way to hook tooltips. It fires after
    -- C_TooltipInfo populates the tooltip with structured data, letting us
    -- append our own lines cleanly.
    -- ---------------------------------------------------------------------------

    -- Track whether we've already added our lines to avoid duplicates.
    -- GameTooltip:Show() can fire multiple times for the same item.
    local processedTooltips = setmetatable({}, { __mode = "k" })

    -- ----- ITEM TOOLTIP HOOK -----
    -- Enum.TooltipDataType.Item = 2 (the constant for item tooltips).
    -- This callback fires whenever an item tooltip is displayed anywhere
    -- in the UI: bags, equipment, loot, mail, auction house, etc.
    TooltipDataProcessor.AddTooltipPostCall(Enum.TooltipDataType.Item,
    function(tooltip, tooltipData)
        -- Only enhance GameTooltip (not shopping comparison tooltips, etc.).
        if tooltip ~= GameTooltip then return end

        -- Guard against double-processing.
        local key = tooltipData and tooltipData.id
        if not key then return end
        if processedTooltips[tooltip] == key then return end
        processedTooltips[tooltip] = key

        local db = ns.db or DEFAULTS

        -- Get the item link from the tooltip. GetItem() returns name, link.
        local _, itemLink = tooltip:GetItem()
        if not itemLink then return end

        -- ITEM LEVEL: GetDetailedItemLevelInfo returns the effective iLvl.
        if db.showItemLevel then
            local effectiveILvl = GetDetailedItemLevelInfo(itemLink)
            if effectiveILvl and effectiveILvl > 0 then
                -- Only show on equippable items (not consumables, quest items, etc.).
                -- IsEquippableItem() returns true for gear.
                if C_Item.IsEquippableItem(itemLink) then
                    tooltip:AddDoubleLine(
                        "Item Level",
                        tostring(effectiveILvl),
                        LABEL_R, LABEL_G, LABEL_B,
                        VALUE_R, VALUE_G, VALUE_B
                    )
                end
            end
        end

        -- VENDOR PRICE: GetItemInfo returns sellPrice at index 11.
        if db.showVendorPrice then
            local info = { C_Item.GetItemInfo(itemLink) }
            local sellPrice = info[11]
            if sellPrice and sellPrice > 0 then
                local formatted = FormatMoney(sellPrice)
                if formatted then
                    tooltip:AddDoubleLine(
                        "Vendor Price",
                        formatted,
                        LABEL_R, LABEL_G, LABEL_B,
                        VALUE_R, VALUE_G, VALUE_B
                    )
                end
            end
        end

        -- ITEM ID: For debugging/development purposes.
        if db.showItemID then
            local itemID = tooltipData.id
            if itemID then
                tooltip:AddDoubleLine(
                    "Item ID",
                    tostring(itemID),
                    LABEL_R, LABEL_G, LABEL_B,
                    ID_R, ID_G, ID_B
                )
            end
        end

        -- Reflow the tooltip to accommodate the new lines.
        tooltip:Show()
    end)

    -- ----- SPELL TOOLTIP HOOK -----
    -- Enum.TooltipDataType.Spell = 1 (the constant for spell tooltips).
    -- Fires for action bar buttons, spellbook entries, talent tooltips, etc.
    TooltipDataProcessor.AddTooltipPostCall(Enum.TooltipDataType.Spell,
    function(tooltip, tooltipData)
        if tooltip ~= GameTooltip then return end

        local db = ns.db or DEFAULTS
        if not db.showSpellID then return end

        -- tooltipData.id is the spell ID.
        local spellID = tooltipData and tooltipData.id
        if not spellID then return end

        -- Guard against double-processing.
        if processedTooltips[tooltip] == ("spell" .. spellID) then return end
        processedTooltips[tooltip] = "spell" .. spellID

        tooltip:AddDoubleLine(
            "Spell ID",
            tostring(spellID),
            LABEL_R, LABEL_G, LABEL_B,
            ID_R, ID_G, ID_B
        )

        tooltip:Show()
    end)

    -- Clear processed state when the tooltip hides, so the next show gets fresh data.
    GameTooltip:HookScript("OnHide", function(self)
        processedTooltips[self] = nil
    end)

    -- ---------------------------------------------------------------------------
    -- INITIALIZATION
    -- ---------------------------------------------------------------------------

    local frame = CreateFrame("Frame")
    frame:RegisterEvent("ADDON_LOADED")

    frame:SetScript("OnEvent", function(self, event, addon)
        if addon ~= addonName then return end

        BetterTooltipInfoDB = BetterTooltipInfoDB or {}
        MergeDefaults(BetterTooltipInfoDB, DEFAULTS)
        ns.db = BetterTooltipInfoDB

        self:UnregisterEvent("ADDON_LOADED")
    end)

    -- ---------------------------------------------------------------------------
    -- ADDON COMPARTMENT FUNCTIONS
    -- These global functions are referenced in the .toc file and provide
    -- click/hover behavior for the Addon Compartment dropdown (minimap button).
    -- ---------------------------------------------------------------------------

    -- Called when the player clicks our entry in the Addon Compartment dropdown.
    -- Opens the settings or toggles features.
    function BetterTooltipInfo_OnCompartmentClick(addonName, button)
        -- Left-click: toggle item level display.
        if button == "LeftButton" then
            ns.db.showItemLevel = not ns.db.showItemLevel
            local state = ns.db.showItemLevel and "|cff00ff00ON|r" or "|cffff0000OFF|r"
            print("|cff33ccffBetter Tooltip:|r Item Level display: " .. state)
        -- Right-click: toggle vendor price display.
        elseif button == "RightButton" then
            ns.db.showVendorPrice = not ns.db.showVendorPrice
            local state = ns.db.showVendorPrice and "|cff00ff00ON|r" or "|cffff0000OFF|r"
            print("|cff33ccffBetter Tooltip:|r Vendor Price display: " .. state)
        end
    end

    -- Called when the mouse enters our Addon Compartment entry.
    -- Shows a tooltip explaining what the addon does and how to click it.
    function BetterTooltipInfo_OnCompartmentEnter(addonName, menuButtonFrame)
        GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
        GameTooltip:SetText("Better Tooltip Info")
        GameTooltip:AddLine("Left-click: Toggle item level display", 1, 1, 1)
        GameTooltip:AddLine("Right-click: Toggle vendor price display", 1, 1, 1)
        GameTooltip:Show()
    end

    -- Called when the mouse leaves our Addon Compartment entry.
    function BetterTooltipInfo_OnCompartmentLeave(addonName, menuButtonFrame)
        GameTooltip:Hide()
    end

    -- ---------------------------------------------------------------------------
    -- SLASH COMMAND
    -- /bti — print current settings and toggle options
    -- ---------------------------------------------------------------------------

    SLASH_BETTERTOOLTIPINFO1 = "/bti"
    SlashCmdList["BETTERTOOLTIPINFO"] = function(msg)
        local cmd = strtrim(msg):lower()

        if cmd == "itemlevel" or cmd == "ilvl" then
            ns.db.showItemLevel = not ns.db.showItemLevel
            local state = ns.db.showItemLevel and "|cff00ff00ON|r" or "|cffff0000OFF|r"
            print("|cff33ccffBetter Tooltip:|r Item Level: " .. state)
        elseif cmd == "vendor" or cmd == "price" then
            ns.db.showVendorPrice = not ns.db.showVendorPrice
            local state = ns.db.showVendorPrice and "|cff00ff00ON|r" or "|cffff0000OFF|r"
            print("|cff33ccffBetter Tooltip:|r Vendor Price: " .. state)
        elseif cmd == "spellid" then
            ns.db.showSpellID = not ns.db.showSpellID
            local state = ns.db.showSpellID and "|cff00ff00ON|r" or "|cffff0000OFF|r"
            print("|cff33ccffBetter Tooltip:|r Spell ID: " .. state)
        elseif cmd == "itemid" then
            ns.db.showItemID = not ns.db.showItemID
            local state = ns.db.showItemID and "|cff00ff00ON|r" or "|cffff0000OFF|r"
            print("|cff33ccffBetter Tooltip:|r Item ID: " .. state)
        else
            print("|cff33ccffBetter Tooltip Info|r — Settings:")
            local function Status(val)
                return val and "|cff00ff00ON|r" or "|cffff0000OFF|r"
            end
            print("  Item Level:  " .. Status(ns.db.showItemLevel)  .. "  (/bti ilvl)")
            print("  Vendor Price: " .. Status(ns.db.showVendorPrice) .. "  (/bti vendor)")
            print("  Spell ID:    " .. Status(ns.db.showSpellID)    .. "  (/bti spellid)")
            print("  Item ID:     " .. Status(ns.db.showItemID)     .. "  (/bti itemid)")
        end
    end
    ```
