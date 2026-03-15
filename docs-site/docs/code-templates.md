# Code Templates for Midnight (Interface 120001)

Complete, copy-paste-ready addon templates for WoW Patch 12.0+. Every template includes the full `.toc` file and all `.lua` files with every line commented.

!!! tip "How to use these templates"
    1. Create a folder in `Interface/AddOns/` matching the addon name exactly
    2. Copy the `.toc` contents into `AddonName.toc`
    3. Copy each `.lua` file into the same folder
    4. Restart the WoW client (or `/reload` if the addon folder already existed)

---

## 1. Hello World

The absolute minimum addon. One `.toc`, one `.lua`, one chat message on login.

### File Structure

```
Interface/AddOns/HelloWorld/
├── HelloWorld.toc
└── HelloWorld.lua
```

=== "HelloWorld.toc"

    ```toc
    ## Interface: 120001
    ## -- Target Midnight (12.0.1) client; flags outdated if wrong
    ## Title: Hello World
    ## -- Name shown in the in-game Addon List
    ## Notes: Prints a message when you log in.
    ## -- Tooltip text in the Addon List
    ## Author: YourName
    ## -- Your name or handle
    ## Version: 1.0.0
    ## -- Version string (cosmetic; no auto-update logic)
    ## Category: Miscellaneous
    ## -- Groups addon under this heading in the Addon List (11.1.0+)
    ## IconTexture: Interface\Icons\INV_Misc_Note_01
    ## -- Icon shown next to the addon name in the list (10.1.0+)

    # Files below are loaded in the order listed.
    # Only one file in this addon — it runs on startup.
    HelloWorld.lua
    ```

=== "HelloWorld.lua"

    ```lua
    -- HelloWorld.lua
    -- The simplest possible WoW addon: print a message when the player logs in.

    -- Every .lua file in an addon receives two values via the `...` varargs:
    --   addonName (string) — matches the folder name, e.g. "HelloWorld"
    --   ns (table)         — a private namespace table shared across all
    --                        files in this addon (empty by default)
    local addonName, ns = ...

    -- CreateFrame("Frame") makes an invisible frame that can listen for events.
    -- We never show this frame on screen — it's purely an event receiver.
    local frame = CreateFrame("Frame")

    -- RegisterEvent tells the client to fire this frame's OnEvent when
    -- PLAYER_LOGIN occurs. PLAYER_LOGIN fires exactly once, after all addons
    -- are loaded but before the loading screen disappears. Character data
    -- (name, level, class, etc.) is available at this point.
    frame:RegisterEvent("PLAYER_LOGIN")

    -- SetScript("OnEvent", handler) attaches our handler function.
    -- The handler receives: self (the frame), event (string), ... (event args).
    frame:SetScript("OnEvent", function(self, event, ...)
        -- Since we only registered one event we could skip this check,
        -- but it's good practice for when you add more events later.
        if event == "PLAYER_LOGIN" then
            -- UnitName("player") returns the logged-in character's name.
            local playerName = UnitName("player")

            -- print() writes to the default chat frame.
            -- |cff00ff00 starts green-colored text (ff = alpha, 00ff00 = RGB).
            -- |r resets the color back to default.
            print("|cff00ff00" .. addonName .. "|r loaded! Welcome, " .. playerName .. "!")

            -- Unregister the event so it won't fire again (good hygiene,
            -- though PLAYER_LOGIN only fires once per session anyway).
            self:UnregisterEvent("PLAYER_LOGIN")
        end
    end)
    ```

---

## 2. SavedVariables Addon

Full namespace pattern with table-dispatch event handling, defaults merging, per-character and account-wide storage, and slash command registration.

### File Structure

```
Interface/AddOns/EventDriven/
├── EventDriven.toc
└── EventDriven.lua
```

=== "EventDriven.toc"

    ```toc
    ## Interface: 120001
    ## Title: Event Driven
    ## Notes: Demonstrates event handling, saved variables, and slash commands.
    ## Author: YourName
    ## Version: 1.0.0

    ## -- SavedVariables declares global Lua tables that persist between sessions.
    ## -- The client writes them to WTF/Account/.../SavedVariables/EventDriven.lua
    ## -- on logout. They become available after ADDON_LOADED fires.
    ## SavedVariables: EventDrivenDB
    ## -- SavedVariablesPerCharacter creates a separate copy per character,
    ## -- stored in WTF/Account/.../RealmName/CharName/SavedVariables/.
    ## SavedVariablesPerCharacter: EventDrivenCharDB

    # Only one Lua file — loaded at startup.
    EventDriven.lua
    ```

=== "EventDriven.lua"

    ```lua
    -- EventDriven.lua
    -- Full event-driven addon with saved variables, defaults merging,
    -- table dispatch, and slash commands.

    -- Capture the addon name and the shared private namespace table.
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- UTILITIES
    -- Convenience functions used throughout the addon.
    -- ---------------------------------------------------------------------------

    -- Print a colored message prefixed with the addon name.
    -- ns:Print("hello") -> "[EventDriven]: hello" in chat (cyan colored).
    function ns:Print(msg)
        -- |cff33ccff = cyan color, |r = reset to default.
        print("|cff33ccff" .. addonName .. ":|r " .. msg)
    end

    -- Conditional debug output — only prints when debug mode is enabled.
    -- Accepts multiple arguments for convenience: ns:Debug("foo", 42, bar).
    function ns:Debug(...)
        -- Early-out if debug is off (or db not yet loaded).
        if not (ns.db and ns.db.debug) then return end
        -- Collect all varargs into a table of strings.
        local parts = {}
        for i = 1, select("#", ...) do
            -- select(i, ...) returns the i-th vararg.
            -- tostring() handles nil, numbers, booleans safely.
            parts[#parts + 1] = tostring(select(i, ...))
        end
        -- table.concat joins the parts with spaces, printed in orange.
        print("|cffff9900[" .. addonName .. " Debug]|r " .. table.concat(parts, " "))
    end

    -- ---------------------------------------------------------------------------
    -- DEFAULTS
    -- Tables of initial values. When a user first installs the addon (or
    -- when a new setting is added in an update), these fill in the gaps.
    -- ---------------------------------------------------------------------------

    -- Account-wide settings (shared by all characters on the account).
    local DEFAULTS = {
        welcomeMessage = true,   -- show greeting on login
        debug          = false,  -- enable debug output
        soundEnabled   = true,   -- play notification sounds
        trackingMode   = "auto", -- "auto", "manual", or "off"
    }

    -- Per-character settings (each character gets their own copy).
    local CHAR_DEFAULTS = {
        enabled   = true, -- per-character on/off toggle
        lastLogin = nil,  -- timestamp of last login (nil until first login)
    }

    -- Merge default values into a saved table without overwriting existing keys.
    -- This handles the case where a new version adds new settings: existing
    -- user choices are preserved, but new keys get their default values.
    local function MergeDefaults(saved, defaults)
        for key, value in pairs(defaults) do
            -- Check for nil specifically (not falsiness) so that intentional
            -- `false` values set by the user are preserved.
            if saved[key] == nil then
                -- For table values, deep-copy so the default table isn't shared
                -- between the defaults and saved data.
                if type(value) == "table" then
                    -- CopyTable is a WoW global that recursively copies tables.
                    saved[key] = CopyTable(value)
                else
                    saved[key] = value
                end
            end
        end
    end

    -- ---------------------------------------------------------------------------
    -- EVENT DISPATCH TABLE
    -- Each key is an event name; each value is a handler function.
    -- The OnEvent script below looks up event names in this table and calls
    -- the matching handler. This avoids long if/elseif chains.
    -- ---------------------------------------------------------------------------

    -- Create an invisible frame to receive events.
    local frame = CreateFrame("Frame")
    -- The dispatch table: event name -> handler function.
    local events = {}

    -- ADDON_LOADED fires once per addon after its files + saved vars are ready.
    -- loadedAddon: the name of the addon that just finished loading.
    function events:ADDON_LOADED(loadedAddon)
        -- Only act when our addon loads (not when other addons load).
        if loadedAddon ~= addonName then return end

        -- Initialize account-wide saved variables.
        -- If the global doesn't exist yet (first install), create an empty table.
        EventDrivenDB = EventDrivenDB or {}
        -- Merge defaults into the saved table.
        MergeDefaults(EventDrivenDB, DEFAULTS)
        -- Store a shorthand on the namespace for convenient access.
        ns.db = EventDrivenDB

        -- Same process for per-character saved variables.
        EventDrivenCharDB = EventDrivenCharDB or {}
        MergeDefaults(EventDrivenCharDB, CHAR_DEFAULTS)
        ns.charDB = EventDrivenCharDB

        -- Debug output to verify initialization.
        ns:Debug("Saved variables initialized.",
            "Account keys:", ns:CountKeys(EventDrivenDB),
            "Char keys:", ns:CountKeys(EventDrivenCharDB))

        -- We only need ADDON_LOADED once — unregister it.
        self:UnregisterEvent("ADDON_LOADED")
    end

    -- PLAYER_LOGIN fires once after ALL addons are loaded.
    -- Character data (name, class, level, talents, inventory) is available.
    function events:PLAYER_LOGIN()
        -- Record the login timestamp for this character.
        ns.charDB.lastLogin = date("%Y-%m-%d %H:%M:%S")

        -- Register slash commands now that saved variables are ready.
        ns:RegisterSlashCommands()

        -- Show a welcome message if the user hasn't turned it off.
        if ns.db.welcomeMessage then
            -- UnitName("player") returns just the character name.
            ns:Print("Welcome, " .. UnitName("player") .. "!")
        end
    end

    -- PLAYER_ENTERING_WORLD fires on login, /reload, and every zone transition.
    -- isInitialLogin: true only on the very first load of the session.
    -- isReloadingUi:  true when the player types /reload.
    function events:PLAYER_ENTERING_WORLD(isInitialLogin, isReloadingUi)
        -- Only log this in debug mode to avoid chat spam.
        ns:Debug("PLAYER_ENTERING_WORLD",
            "initial=" .. tostring(isInitialLogin),
            "reload=" .. tostring(isReloadingUi))
    end

    -- PLAYER_LOGOUT fires right before the client disconnects.
    -- Saved variables are written to disk after this event, so any last-minute
    -- changes to the saved table will be persisted automatically.
    function events:PLAYER_LOGOUT()
        ns:Debug("Logging out. Saved variables will be written to disk.")
    end

    -- The single OnEvent handler: dispatch to the table.
    -- self = the frame, event = the event name, ... = event-specific args.
    frame:SetScript("OnEvent", function(self, event, ...)
        -- Look up the event in our dispatch table.
        if events[event] then
            -- Call it, passing the frame as self and all event args.
            events[event](self, ...)
        end
    end)

    -- Auto-register every event that has a handler in the dispatch table.
    for eventName in pairs(events) do
        frame:RegisterEvent(eventName)
    end

    -- ---------------------------------------------------------------------------
    -- UTILITIES (continued)
    -- ---------------------------------------------------------------------------

    -- Count the number of keys in a table (useful for debug output).
    function ns:CountKeys(t)
        local n = 0
        for _ in pairs(t) do n = n + 1 end
        return n
    end

    -- ---------------------------------------------------------------------------
    -- SLASH COMMANDS
    -- Registered during PLAYER_LOGIN so saved variables are ready.
    -- ---------------------------------------------------------------------------

    function ns:RegisterSlashCommands()
        -- SLASH_EVENTDRIVEN1, SLASH_EVENTDRIVEN2, etc. define the command aliases.
        -- They must be consecutive globals starting at 1. The matching
        -- SlashCmdList["EVENTDRIVEN"] handler processes any of them.
        SLASH_EVENTDRIVEN1 = "/eventdriven"  -- full name
        SLASH_EVENTDRIVEN2 = "/ed"           -- short alias

        -- The handler receives everything typed after the slash command.
        -- "/ed reset all" -> msg = "reset all"
        SlashCmdList["EVENTDRIVEN"] = function(msg)
            -- Split the first word (command) from the rest (arguments).
            -- ^(%S*) captures the first non-space sequence.
            -- %s*(.-) captures everything after the first space, trimmed.
            -- $ anchors to end of string.
            local cmd, rest = msg:match("^(%S*)%s*(.-)$")
            -- Normalize command to lowercase for case-insensitive matching.
            cmd = (cmd or ""):lower()

            if cmd == "reset" then
                -- Wipe all saved data and re-apply defaults.
                wipe(EventDrivenDB)
                wipe(EventDrivenCharDB)
                MergeDefaults(EventDrivenDB, DEFAULTS)
                MergeDefaults(EventDrivenCharDB, CHAR_DEFAULTS)
                -- Update the shortcuts.
                ns.db = EventDrivenDB
                ns.charDB = EventDrivenCharDB
                ns:Print("All settings reset to defaults.")

            elseif cmd == "toggle" then
                -- Toggle the per-character enabled flag.
                ns.charDB.enabled = not ns.charDB.enabled
                -- Format an ON/OFF string with color.
                local state = ns.charDB.enabled
                    and "|cff00ff00ON|r"   -- green
                    or "|cffff0000OFF|r"   -- red
                ns:Print("Addon " .. state .. " for this character.")

            elseif cmd == "debug" then
                -- Toggle debug output.
                ns.db.debug = not ns.db.debug
                local state = ns.db.debug
                    and "|cff00ff00ON|r"
                    or "|cffff0000OFF|r"
                ns:Print("Debug mode " .. state)

            elseif cmd == "status" then
                -- Print current state.
                ns:Print("Status:")
                print("  Enabled: " .. tostring(ns.charDB.enabled))
                print("  Debug:   " .. tostring(ns.db.debug))
                print("  Sound:   " .. tostring(ns.db.soundEnabled))
                print("  Mode:    " .. ns.db.trackingMode)
                print("  Last login: " .. (ns.charDB.lastLogin or "never"))

            else
                -- No recognized command — show help.
                ns:Print("Commands:")
                print("  /ed status  — Show current settings")
                print("  /ed toggle  — Enable/disable for this character")
                print("  /ed debug   — Toggle debug output")
                print("  /ed reset   — Reset all settings to defaults")
            end
        end
    end
    ```

---

## 3. Movable Settings Frame

A draggable options panel with `BackdropTemplate`, checkboxes, a slider, close button, and ESC-to-close. This is the UI pattern you'll reuse for any custom settings window.

### File Structure

```
Interface/AddOns/SettingsFrame/
├── SettingsFrame.toc
└── SettingsFrame.lua
```

=== "SettingsFrame.toc"

    ```toc
    ## Interface: 120001
    ## Title: Settings Frame Demo
    ## Notes: Draggable settings panel with checkboxes, slider, and ESC-to-close.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Miscellaneous
    ## IconTexture: Interface\Icons\Trade_Engineering

    ## -- Account-wide saved variables for user settings.
    ## SavedVariables: SettingsFrameDB

    # Single file — creates the panel and handles events.
    SettingsFrame.lua
    ```

=== "SettingsFrame.lua"

    ```lua
    -- SettingsFrame.lua
    -- A movable settings panel with BackdropTemplate, checkboxes, a slider,
    -- close button, and ESC-to-close integration.

    -- Capture addon name and private namespace.
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- UTILITIES & DEFAULTS
    -- ---------------------------------------------------------------------------

    -- Colored print helper.
    function ns:Print(msg)
        print("|cff33ccff" .. addonName .. ":|r " .. msg)
    end

    -- Default settings that new users start with.
    local DEFAULTS = {
        featureEnabled = true,   -- master feature toggle
        notifications  = true,   -- show notification popups
        volume         = 50,     -- notification volume (0-100)
    }

    -- Merge defaults into saved table without overwriting user choices.
    local function MergeDefaults(saved, defaults)
        for k, v in pairs(defaults) do
            if saved[k] == nil then
                -- Deep-copy table values; simple-assign everything else.
                saved[k] = type(v) == "table" and CopyTable(v) or v
            end
        end
    end

    -- ---------------------------------------------------------------------------
    -- SETTINGS PANEL CREATION
    -- Called once during ADDON_LOADED. Builds the entire panel and all controls.
    -- ---------------------------------------------------------------------------

    local function CreateSettingsPanel()
        -- Create the main frame with BackdropTemplate.
        -- BackdropTemplate is a mixin that gives :SetBackdrop(), :SetBackdropColor(), etc.
        -- The second arg is a global name — required for UISpecialFrames (ESC-to-close).
        local panel = CreateFrame("Frame", "SettingsFramePanel", UIParent,
            "BackdropTemplate")
        -- Set the panel size in pixels (width x height).
        panel:SetSize(320, 300)
        -- Center it on screen.
        panel:SetPoint("CENTER")
        -- Apply a backdrop: background fill + border + insets.
        panel:SetBackdrop({
            -- bgFile: texture used for the solid background fill.
            bgFile   = "Interface\\Tooltips\\UI-Tooltip-Background",
            -- edgeFile: texture used for the border.
            edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
            -- tile: whether to tile (repeat) the background texture.
            tile     = true,
            -- tileSize: how many pixels per tile repetition.
            tileSize = 16,
            -- edgeSize: thickness of the border in pixels.
            edgeSize = 16,
            -- insets: padding between the border and the content area.
            insets   = { left = 4, right = 4, top = 4, bottom = 4 },
        })
        -- Background color: dark gray, 90% opaque. Args: R, G, B, A (0-1 each).
        panel:SetBackdropColor(0.1, 0.1, 0.1, 0.9)
        -- Border color: medium gray, fully opaque.
        panel:SetBackdropBorderColor(0.4, 0.4, 0.4, 1)
        -- DIALOG strata renders above most game UI (bags, map, etc.).
        panel:SetFrameStrata("DIALOG")

        -- MAKE IT DRAGGABLE
        -- SetMovable allows the frame to be repositioned by dragging.
        panel:SetMovable(true)
        -- EnableMouse lets the frame receive mouse events (clicks, drags).
        panel:EnableMouse(true)
        -- RegisterForDrag specifies which mouse button starts a drag.
        panel:RegisterForDrag("LeftButton")
        -- OnDragStart: begin moving when the user clicks and holds.
        -- panel.StartMoving is a shorthand for function(self) self:StartMoving() end.
        panel:SetScript("OnDragStart", panel.StartMoving)
        -- OnDragStop: stop moving when the user releases the mouse button.
        panel:SetScript("OnDragStop", panel.StopMovingOrSizing)
        -- SetClampedToScreen prevents dragging the panel offscreen.
        panel:SetClampedToScreen(true)

        -- ESC-TO-CLOSE (method 1): handle the Escape key directly via OnKeyDown.
        -- When Escape is pressed, hide the panel and eat the keypress.
        -- For other keys, let the event propagate normally.
        panel:SetScript("OnKeyDown", function(self, key)
            if key == "ESCAPE" then
                -- Don't let this Escape propagate (would open game menu).
                self:SetPropagateKeyboardInput(false)
                self:Hide()
            else
                -- Let all other keys pass through to the game normally.
                self:SetPropagateKeyboardInput(true)
            end
        end)
        -- EnableKeyboard lets the frame receive keyboard events.
        panel:EnableKeyboard(true)

        -- ESC-TO-CLOSE (method 2): also register with UISpecialFrames.
        -- This is the traditional approach — WoW auto-hides frames in this list
        -- when Escape is pressed. Both methods work; belt-and-suspenders.
        tinsert(UISpecialFrames, "SettingsFramePanel")

        -- ----- TITLE BAR -----
        -- CreateFontString makes a text label attached to the frame.
        -- "OVERLAY" layer ensures it renders above the backdrop.
        -- "GameFontNormalLarge" is a built-in font template (large, white).
        local title = panel:CreateFontString(nil, "OVERLAY", "GameFontNormalLarge")
        -- Anchor 12px below the top edge of the panel.
        title:SetPoint("TOP", panel, "TOP", 0, -12)
        title:SetText(addonName .. " Settings")

        -- ----- CLOSE BUTTON -----
        -- UIPanelCloseButton is Blizzard's standard X button template.
        local closeBtn = CreateFrame("Button", nil, panel, "UIPanelCloseButton")
        -- Position at the top-right corner of the panel.
        closeBtn:SetPoint("TOPRIGHT", panel, "TOPRIGHT", -2, -2)
        -- Clicking hides the parent panel.
        closeBtn:SetScript("OnClick", function() panel:Hide() end)

        -- ----- CHECKBOX: Feature Enabled -----
        -- InterfaceOptionsCheckButtonTemplate provides a pre-styled checkbox
        -- with a .Text child FontString for the label.
        local cb1 = CreateFrame("CheckButton", "SettingsFrameCB1", panel,
            "InterfaceOptionsCheckButtonTemplate")
        -- Position inside the panel, 20px from left edge, 50px below top.
        cb1:SetPoint("TOPLEFT", panel, "TOPLEFT", 20, -50)
        -- Set the label text.
        cb1.Text:SetText("Feature Enabled")
        -- Initialize the checked state from saved data.
        cb1:SetChecked(ns.db.featureEnabled)
        -- OnClick fires when the user toggles the checkbox.
        cb1:SetScript("OnClick", function(self)
            -- GetChecked() returns the NEW state after the click.
            ns.db.featureEnabled = self:GetChecked()
            ns:Print("Feature " .. (ns.db.featureEnabled and "enabled" or "disabled"))
        end)

        -- ----- CHECKBOX: Notifications -----
        local cb2 = CreateFrame("CheckButton", "SettingsFrameCB2", panel,
            "InterfaceOptionsCheckButtonTemplate")
        -- Anchor below the first checkbox with 10px vertical spacing.
        cb2:SetPoint("TOPLEFT", cb1, "BOTTOMLEFT", 0, -10)
        cb2.Text:SetText("Show Notifications")
        cb2:SetChecked(ns.db.notifications)
        cb2:SetScript("OnClick", function(self)
            ns.db.notifications = self:GetChecked()
        end)

        -- ----- SLIDER: Volume -----
        -- OptionsSliderTemplate provides a pre-styled horizontal slider
        -- with .Low, .High, and .Text child FontStrings.
        local slider = CreateFrame("Slider", "SettingsFrameSlider", panel,
            "OptionsSliderTemplate")
        -- Position below the checkboxes with extra padding for the label.
        slider:SetPoint("TOPLEFT", cb2, "BOTTOMLEFT", 0, -30)
        -- Set the slider width (height is fixed by the template).
        slider:SetWidth(200)
        slider:SetHeight(17)
        -- Value range: 0 to 100 (volume percentage).
        slider:SetMinMaxValues(0, 100)
        -- Step size: the slider snaps to whole numbers.
        slider:SetValueStep(1)
        -- Obey the step when dragging (prevents fractional values).
        slider:SetObeyStepOnDrag(true)
        -- Initialize from saved data.
        slider:SetValue(ns.db.volume)
        -- Set the endpoint labels.
        slider.Low:SetText("0")
        slider.High:SetText("100")
        -- Set the current-value label above the slider.
        slider.Text:SetText("Volume: " .. ns.db.volume)
        -- OnValueChanged fires every time the slider moves.
        slider:SetScript("OnValueChanged", function(self, value)
            -- Round to nearest integer (safety against floating-point drift).
            value = math.floor(value + 0.5)
            -- Save to the database.
            ns.db.volume = value
            -- Update the label.
            self.Text:SetText("Volume: " .. value)
        end)

        -- ----- RESET BUTTON -----
        -- UIPanelButtonTemplate is Blizzard's standard push button.
        local resetBtn = CreateFrame("Button", nil, panel, "UIPanelButtonTemplate")
        resetBtn:SetSize(100, 24)
        -- Anchor at the bottom-left of the panel.
        resetBtn:SetPoint("BOTTOMLEFT", panel, "BOTTOMLEFT", 20, 16)
        resetBtn:SetText("Reset")
        resetBtn:SetScript("OnClick", function()
            -- Wipe all saved data.
            wipe(SettingsFrameDB)
            -- Re-apply defaults.
            MergeDefaults(SettingsFrameDB, DEFAULTS)
            ns.db = SettingsFrameDB
            -- Refresh all controls to reflect the new defaults.
            cb1:SetChecked(ns.db.featureEnabled)
            cb2:SetChecked(ns.db.notifications)
            slider:SetValue(ns.db.volume)
            ns:Print("Settings reset to defaults.")
        end)

        -- ----- DONE BUTTON -----
        local doneBtn = CreateFrame("Button", nil, panel, "UIPanelButtonTemplate")
        doneBtn:SetSize(80, 24)
        -- Anchor at the bottom-right of the panel.
        doneBtn:SetPoint("BOTTOMRIGHT", panel, "BOTTOMRIGHT", -20, 16)
        doneBtn:SetText("Done")
        -- Clicking Done simply hides the panel (settings auto-save).
        doneBtn:SetScript("OnClick", function() panel:Hide() end)

        -- Start hidden — opened via slash command.
        panel:Hide()
        -- Store on the namespace so other code can show/toggle it.
        ns.panel = panel
    end

    -- ---------------------------------------------------------------------------
    -- EVENT DISPATCH
    -- ---------------------------------------------------------------------------

    local events = {}

    -- ADDON_LOADED: initialize saved variables and build the UI.
    function events:ADDON_LOADED(loadedAddon)
        if loadedAddon ~= addonName then return end
        -- Initialize or create saved variables.
        SettingsFrameDB = SettingsFrameDB or {}
        MergeDefaults(SettingsFrameDB, DEFAULTS)
        ns.db = SettingsFrameDB
        -- Build the settings panel.
        CreateSettingsPanel()
        -- Done with this event.
        self:UnregisterEvent("ADDON_LOADED")
    end

    -- PLAYER_LOGIN: register slash commands.
    function events:PLAYER_LOGIN()
        -- Two aliases for the slash command.
        SLASH_SETTINGSFRAME1 = "/settingsframe"
        SLASH_SETTINGSFRAME2 = "/sf"
        SlashCmdList["SETTINGSFRAME"] = function(msg)
            -- Toggle panel visibility.
            if ns.panel:IsShown() then
                ns.panel:Hide()
            else
                ns.panel:Show()
            end
        end
        -- Let the user know how to open settings.
        ns:Print("Type /sf to open settings.")
    end

    -- Wire up the event frame.
    local eventFrame = CreateFrame("Frame")
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then events[event](self, ...) end
    end)
    for ev in pairs(events) do
        eventFrame:RegisterEvent(ev)
    end
    ```

!!! warning "Legacy Template"
    `InterfaceOptionsCheckButtonTemplate` is a legacy template from the deprecated InterfaceOptions system. It still works for custom settings frames (as shown above), but for integrating with the Blizzard settings panel use the modern Settings API (`Settings.RegisterAddOnSetting` + `Settings.CreateCheckbox`). See the [Blizzard Systems guide](../blizzard-systems.md) for the modern approach.

---

## 4. Minimap Button & Addon Compartment

Registers your addon in the minimap's **Addon Compartment** dropdown (added in 10.1.0) — no LibDataBroker or custom minimap button code needed.

### File Structure

```
Interface/AddOns/CompartmentDemo/
├── CompartmentDemo.toc
└── CompartmentDemo.lua
```

=== "CompartmentDemo.toc"

    ```toc
    ## Interface: 120001
    ## Title: Compartment Demo
    ## Notes: Shows how to use the Addon Compartment (minimap dropdown).
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Miscellaneous

    ## -- IconTexture sets the addon's icon in the Addon List.
    ## IconTexture: Interface\Icons\Ability_Spy
    ## -- IconAtlas uses a texture atlas for the icon (alternative to IconTexture).
    ## IconAtlas: Warlock-ReadyShard

    ## -- SavedVariables for click tracking demo.
    ## SavedVariables: CompartmentDemoDB

    ## -- ADDON COMPARTMENT registration via TOC fields.
    ## -- These three fields point to global Lua function names.
    ## -- The game calls them when the user interacts with the
    ## -- addon's entry in the minimap Addon Compartment dropdown.

    ## -- Called when the user clicks the compartment entry.
    ## AddonCompartmentFunc: CompartmentDemo_OnClick
    ## -- Called when the mouse enters the entry (show tooltip).
    ## AddonCompartmentFuncOnEnter: CompartmentDemo_OnEnter
    ## -- Called when the mouse leaves the entry (hide tooltip).
    ## AddonCompartmentFuncOnLeave: CompartmentDemo_OnLeave

    # Single file addon.
    CompartmentDemo.lua
    ```

=== "CompartmentDemo.lua"

    ```lua
    -- CompartmentDemo.lua
    -- Demonstrates the Addon Compartment system: a minimap dropdown button
    -- without needing LibDataBroker or custom minimap handling.

    -- Capture addon name and namespace.
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- SAVED VARIABLES
    -- ---------------------------------------------------------------------------

    -- Default values for the saved table.
    ns.defaults = {
        clickCount = 0,  -- tracks total compartment button clicks
    }

    -- Initialize saved variables (called from ADDON_LOADED).
    function ns:InitDB()
        -- Create the table if it doesn't exist (first install).
        if not CompartmentDemoDB then CompartmentDemoDB = {} end
        -- Fill in any missing keys from defaults.
        for k, v in pairs(ns.defaults) do
            if CompartmentDemoDB[k] == nil then
                CompartmentDemoDB[k] = v
            end
        end
        -- Store a shortcut on the namespace.
        ns.db = CompartmentDemoDB
    end

    -- ---------------------------------------------------------------------------
    -- ADDON COMPARTMENT HANDLERS
    -- These MUST be global functions (not local, not on a table).
    -- The function names MUST exactly match the TOC field values.
    -- ---------------------------------------------------------------------------

    -- Called when the user LEFT-clicks or RIGHT-clicks the compartment entry.
    -- name: the addon name string (e.g. "CompartmentDemo")
    -- mouseButton: "LeftButton" or "RightButton"
    function CompartmentDemo_OnClick(name, mouseButton)
        if mouseButton == "LeftButton" then
            -- Left-click: increment the counter and print it.
            ns.db.clickCount = ns.db.clickCount + 1
            print("|cff00ff00" .. addonName .. "|r: Clicked "
                .. ns.db.clickCount .. " time(s)!")
        elseif mouseButton == "RightButton" then
            -- Right-click: could open settings, toggle a feature, etc.
            -- Here we just print a message as a placeholder.
            print("|cff00ff00" .. addonName .. "|r: Right-click! "
                .. "(Bind your settings toggle here)")
        end
    end

    -- Called when the mouse cursor enters the compartment entry.
    -- Used to show a tooltip. menuButtonFrame is the anchor for the tooltip.
    function CompartmentDemo_OnEnter(name, menuButtonFrame)
        -- GameTooltip is the global shared tooltip frame used by most UI elements.
        -- SetOwner tells it where to appear and how to anchor.
        -- "ANCHOR_LEFT" positions the tooltip to the left of the anchor frame.
        GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")

        -- SetText sets the title line (bold, larger font).
        GameTooltip:SetText("Compartment Demo")

        -- AddLine adds a body line. Args: text, r, g, b (0-1 color values).
        -- White text for primary instructions.
        GameTooltip:AddLine("Left-click to increment counter", 1, 1, 1)
        -- Gray text for secondary instructions.
        GameTooltip:AddLine("Right-click for settings", 0.7, 0.7, 0.7)

        -- AddDoubleLine creates a two-column row.
        -- Args: leftText, rightText, leftR, leftG, leftB, rightR, rightG, rightB.
        GameTooltip:AddDoubleLine(
            "Total clicks:",                              -- left column
            tostring(ns.db and ns.db.clickCount or 0),    -- right column
            1, 0.82, 0,  -- left color: gold
            1, 1, 1      -- right color: white
        )

        -- Show() makes the tooltip visible with all the lines we've added.
        GameTooltip:Show()
    end

    -- Called when the mouse cursor leaves the compartment entry.
    -- Always hide the tooltip here to prevent it from lingering on screen.
    function CompartmentDemo_OnLeave(name, menuButtonFrame)
        GameTooltip:Hide()
    end

    -- ---------------------------------------------------------------------------
    -- INITIALIZATION
    -- ---------------------------------------------------------------------------

    local events = {}

    -- Initialize saved variables when our addon loads.
    function events:ADDON_LOADED(loadedName)
        if loadedName ~= addonName then return end
        ns:InitDB()
        self:UnregisterEvent("ADDON_LOADED")
    end

    -- Register slash commands after all addons are loaded.
    function events:PLAYER_LOGIN()
        -- Provide a slash command as a fallback for players who prefer typing.
        SLASH_COMPARTMENTDEMO1 = "/compartmentdemo"
        SLASH_COMPARTMENTDEMO2 = "/cdemo"
        SlashCmdList["COMPARTMENTDEMO"] = function(msg)
            print("|cff00ff00" .. addonName .. "|r: Total clicks: "
                .. ns.db.clickCount)
        end
    end

    -- Wire up the event frame.
    local eventFrame = CreateFrame("Frame")
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then events[event](self, ...) end
    end)
    for ev in pairs(events) do
        eventFrame:RegisterEvent(ev)
    end
    ```

---

## 5. Addon Communication

Register a message prefix, send addon messages to party/raid/guild, and receive them. Includes throttle-aware sending and encounter-safe queueing for Midnight.

### File Structure

```
Interface/AddOns/CommDemo/
├── CommDemo.toc
└── CommDemo.lua
```

=== "CommDemo.toc"

    ```toc
    ## Interface: 120001
    ## Title: Comm Demo
    ## Notes: Addon-to-addon communication via C_ChatInfo.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Miscellaneous
    ## IconTexture: Interface\Icons\INV_Letter_15

    CommDemo.lua
    ```

=== "CommDemo.lua"

    ```lua
    -- CommDemo.lua
    -- Demonstrates addon-to-addon communication using C_ChatInfo.
    -- Includes encounter-safe message queueing for Midnight (12.0+).

    -- Capture addon name and namespace.
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- CONSTANTS
    -- ---------------------------------------------------------------------------

    -- The addon message prefix. Max 16 characters. This string identifies
    -- our messages so we don't accidentally process messages from other addons.
    local PREFIX = "CommDemo"

    -- Maximum message body length per the WoW API.
    -- Messages longer than this will be silently truncated by the server.
    local MAX_MSG_LEN = 255

    -- ---------------------------------------------------------------------------
    -- PRINT HELPER
    -- ---------------------------------------------------------------------------

    -- Colored print prefixed with addon name.
    function ns:Print(msg)
        print("|cff33ccff" .. addonName .. ":|r " .. msg)
    end

    -- ---------------------------------------------------------------------------
    -- MESSAGE QUEUE (Midnight encounter-safe)
    -- In Midnight (12.0+), SendAddonMessage is throttled or blocked during
    -- active raid encounters and M+ timed runs. We queue messages and flush
    -- them when the encounter ends.
    -- ---------------------------------------------------------------------------

    -- Table holding queued messages. Each entry: {message, channel, target}.
    ns.messageQueue = {}

    -- Send a message safely. If an encounter is in progress, queue it.
    -- message: the string to send (max 255 bytes)
    -- channel: "PARTY", "RAID", "GUILD", "WHISPER", or "CHANNEL"
    -- target:  player name (required only for "WHISPER" channel)
    -- Returns: true if sent immediately, false if queued or failed.
    function ns:SendMessage(message, channel, target)
        -- Validate message length before sending.
        if #message > MAX_MSG_LEN then
            ns:Print("|cffff0000Message too long (" .. #message
                .. "/" .. MAX_MSG_LEN .. " bytes).|r")
            return false
        end

        -- IsEncounterInProgress() returns true during boss fights.
        -- This API exists on retail; guard with a nil check for safety.
        if IsEncounterInProgress and IsEncounterInProgress() then
            -- Queue the message — it will be sent after the encounter.
            tinsert(ns.messageQueue, { message, channel, target })
            return false
        end

        -- Not in an encounter — send immediately.
        -- C_ChatInfo.SendAddonMessage(prefix, message, channel [, target])
        C_ChatInfo.SendAddonMessage(PREFIX, message, channel, target)
        return true
    end

    -- Flush all queued messages. Called when an encounter ends.
    function ns:FlushQueue()
        for _, entry in ipairs(ns.messageQueue) do
            -- unpack extracts the {message, channel, target} fields.
            C_ChatInfo.SendAddonMessage(PREFIX, unpack(entry))
        end
        -- wipe() empties the table in-place without creating garbage.
        wipe(ns.messageQueue)
    end

    -- ---------------------------------------------------------------------------
    -- INCOMING MESSAGE HANDLER
    -- Processes messages received from other players running this addon.
    -- In a real addon you'd deserialize structured data here.
    -- ---------------------------------------------------------------------------

    function ns:HandleMessage(message, sender, channel)
        -- Ignore messages from ourselves to prevent echo loops.
        -- UnitName("player") returns just the name without the realm.
        local myName = UnitName("player")
        -- The sender string may include "-RealmName"; strip it for comparison.
        local senderName = sender:match("^([^%-]+)")
        if senderName == myName then return end

        -- Simple demo protocol: PING / PONG.
        if message == "PING" then
            ns:Print("PING from " .. sender .. " (" .. channel .. ")")
            -- Auto-reply with PONG on the same channel.
            ns:SendMessage("PONG", channel)

        elseif message == "PONG" then
            ns:Print("PONG from " .. sender .. " (" .. channel .. ")")

        else
            -- Unknown message — log it for debugging.
            ns:Print("[" .. channel .. "] " .. sender .. ": " .. message)
        end
    end

    -- ---------------------------------------------------------------------------
    -- EVENTS
    -- ---------------------------------------------------------------------------

    local events = {}

    -- ADDON_LOADED: register our message prefix.
    function events:ADDON_LOADED(name)
        if name ~= addonName then return end

        -- Register the prefix. Must happen before we can send or receive.
        -- Returns true on success, false if already registered or at the limit.
        local ok = C_ChatInfo.RegisterAddonMessagePrefix(PREFIX)
        if not ok then
            ns:Print("|cffff0000Failed to register prefix '" .. PREFIX .. "'|r")
        end

        -- Done with this event.
        self:UnregisterEvent("ADDON_LOADED")
    end

    -- CHAT_MSG_ADDON fires when any addon message with a registered prefix
    -- is received from any player (including yourself).
    -- prefix:  the message prefix string (matches what was registered)
    -- message: the body text (up to 255 bytes)
    -- channel: "PARTY", "RAID", "GUILD", "WHISPER", or "CHANNEL"
    -- sender:  "PlayerName" or "PlayerName-RealmName"
    function events:CHAT_MSG_ADDON(prefix, message, channel, sender)
        -- Only handle messages with OUR prefix.
        if prefix ~= PREFIX then return end
        -- Dispatch to our handler.
        ns:HandleMessage(message, sender, channel)
    end

    -- ENCOUNTER_END fires when a raid boss encounter finishes.
    -- encounterID:   numeric ID of the encounter
    -- encounterName: localized name of the boss
    -- difficultyID:  difficulty setting
    -- groupSize:     number of players in the group
    -- success:       1 if killed, 0 if wiped
    function events:ENCOUNTER_END(encounterID, encounterName, difficultyID,
                                   groupSize, success)
        -- Flush any messages that were queued during the fight.
        ns:FlushQueue()
    end

    -- PLAYER_LOGIN: register slash commands.
    function events:PLAYER_LOGIN()
        SLASH_COMMDEMO1 = "/commdemo"
        SLASH_COMMDEMO2 = "/cd"
        SlashCmdList["COMMDEMO"] = function(msg)
            -- Split command from arguments.
            local cmd, rest = msg:match("^(%S*)%s*(.-)$")
            cmd = (cmd or ""):lower()

            if cmd == "ping" then
                -- Send PING to the group (raid > party fallback).
                local channel = IsInRaid() and "RAID" or "PARTY"
                if ns:SendMessage("PING", channel) then
                    ns:Print("Sent PING to " .. channel)
                else
                    ns:Print("PING queued (encounter in progress)")
                end

            elseif cmd == "send" and rest ~= "" then
                -- Send a custom message: /cd send Hello world
                local channel = IsInRaid() and "RAID" or "PARTY"
                ns:SendMessage(rest, channel)

            elseif cmd == "whisper" then
                -- /cd whisper PlayerName Hello
                local target, text = rest:match("^(%S+)%s+(.+)$")
                if target and text then
                    ns:SendMessage(text, "WHISPER", target)
                else
                    ns:Print("Usage: /cd whisper <name> <message>")
                end

            else
                -- Show help.
                ns:Print("Commands:")
                print("  /cd ping              — Ping your group")
                print("  /cd send <text>       — Send message to group")
                print("  /cd whisper <name> <text> — Whisper a player")
            end
        end
    end

    -- Wire up the event frame.
    local eventFrame = CreateFrame("Frame")
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then events[event](self, ...) end
    end)
    for ev in pairs(events) do
        eventFrame:RegisterEvent(ev)
    end
    ```

---

## 6. Combat Lockdown Safe Pattern

Demonstrates how to safely manipulate frames around combat lockdown. Actions that would fail during combat are queued and replayed when combat ends.

### File Structure

```
Interface/AddOns/CombatSafe/
├── CombatSafe.toc
└── CombatSafe.lua
```

=== "CombatSafe.toc"

    ```toc
    ## Interface: 120001
    ## Title: Combat Safe Demo
    ## Notes: Queues UI changes during combat; applies them when combat ends.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Miscellaneous
    ## IconTexture: Interface\Icons\Ability_Warrior_DefensiveStance

    ## SavedVariables: CombatSafeDB

    CombatSafe.lua
    ```

=== "CombatSafe.lua"

    ```lua
    -- CombatSafe.lua
    -- Demonstrates combat-safe frame management. Many frame operations
    -- (Show, Hide, SetPoint, SetAttribute on secure frames) are forbidden
    -- during combat lockdown. This pattern queues those actions and replays
    -- them when combat ends via PLAYER_REGEN_ENABLED.

    -- Capture addon name and namespace.
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- DEFAULTS
    -- ---------------------------------------------------------------------------

    -- Settings with their initial values.
    ns.defaults = {
        showFrame  = true,  -- whether our demo frame should be visible
        frameScale = 1.0,   -- scale multiplier for the demo frame
        frameAlpha = 0.9,   -- background opacity of the demo frame
    }

    -- Initialize saved variables.
    function ns:InitDB()
        -- Create table if it doesn't exist (first install).
        if not CombatSafeDB then CombatSafeDB = {} end
        -- Merge defaults into saved data.
        for k, v in pairs(ns.defaults) do
            if CombatSafeDB[k] == nil then CombatSafeDB[k] = v end
        end
        -- Shortcut reference.
        ns.db = CombatSafeDB
    end

    -- Colored print helper.
    function ns:Print(msg)
        print("|cff33ccff" .. addonName .. ":|r " .. msg)
    end

    -- ---------------------------------------------------------------------------
    -- COMBAT-SAFE ACTION QUEUE
    -- The core pattern: if we're in combat, store the action as a closure
    -- and run it later. If we're not in combat, run it immediately.
    -- ---------------------------------------------------------------------------

    -- Pending actions: a list of zero-argument functions to call.
    local pendingActions = {}

    -- Execute all pending actions. Called when combat ends.
    -- Double-checks InCombatLockdown() as a safety guard.
    local function ProcessPendingActions()
        -- Safety: don't run if we're still in combat somehow.
        if InCombatLockdown() then return end
        -- Execute each queued action in the order it was added.
        for i, action in ipairs(pendingActions) do
            action()
        end
        -- Clear the queue. wipe() empties the table without creating garbage.
        wipe(pendingActions)
    end

    -- Public API: queue a function to run outside of combat.
    -- If we're already out of combat, run it immediately.
    -- fn: a zero-argument function (closure) containing the deferred work.
    function ns:QueueAction(fn)
        -- InCombatLockdown() returns true when combat restrictions are active.
        -- Combat lockdown starts at PLAYER_REGEN_DISABLED and ends at
        -- PLAYER_REGEN_ENABLED.
        if InCombatLockdown() then
            -- In combat — add to the queue.
            tinsert(pendingActions, fn)
        else
            -- Not in combat — safe to run now.
            fn()
        end
    end

    -- ---------------------------------------------------------------------------
    -- DEMO UI: A status frame that we show/hide/resize safely.
    -- ---------------------------------------------------------------------------

    -- Create the demo frame.
    function ns:CreateDemoFrame()
        -- Create with BackdropTemplate for the background/border.
        local f = CreateFrame("Frame", "CombatSafeDemoFrame", UIParent,
            "BackdropTemplate")
        -- Size: 200px wide, 60px tall.
        f:SetSize(200, 60)
        -- Position: centered near the top of the screen.
        f:SetPoint("TOP", UIParent, "TOP", 0, -100)
        -- Apply a backdrop (dark blue background, blue-ish border).
        f:SetBackdrop({
            bgFile   = "Interface\\Tooltips\\UI-Tooltip-Background",
            edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
            tile = true, tileSize = 16, edgeSize = 16,
            insets = { left = 4, right = 4, top = 4, bottom = 4 },
        })
        -- Dark blue-gray background with configurable alpha.
        f:SetBackdropColor(0.05, 0.05, 0.15, ns.db.frameAlpha)
        -- Blue-ish border.
        f:SetBackdropBorderColor(0.4, 0.6, 1.0, 1.0)

        -- Status text displayed inside the frame.
        local text = f:CreateFontString(nil, "OVERLAY", "GameFontNormal")
        -- Center the text in the frame.
        text:SetPoint("CENTER")
        text:SetText("|cff00ff00Safe|r")

        -- Store references on the namespace so event handlers can update them.
        ns.demoFrame = f
        ns.statusText = text

        -- Apply initial visibility from saved settings.
        if ns.db.showFrame then
            f:Show()
        else
            f:Hide()
        end
        -- Apply initial scale.
        f:SetScale(ns.db.frameScale)
    end

    -- ---------------------------------------------------------------------------
    -- COMBAT-SAFE FRAME OPERATIONS
    -- These wrap frame operations in QueueAction so they work correctly
    -- whether called in combat or out of combat.
    -- ---------------------------------------------------------------------------

    -- Safely show the demo frame.
    function ns:ShowFrame()
        ns:QueueAction(function()
            ns.demoFrame:Show()
            ns.db.showFrame = true
        end)
    end

    -- Safely hide the demo frame.
    function ns:HideFrame()
        ns:QueueAction(function()
            ns.demoFrame:Hide()
            ns.db.showFrame = false
        end)
    end

    -- Safely change the frame's scale.
    function ns:SetScale(newScale)
        ns:QueueAction(function()
            ns.demoFrame:SetScale(newScale)
            ns.db.frameScale = newScale
        end)
    end

    -- ---------------------------------------------------------------------------
    -- EVENTS
    -- ---------------------------------------------------------------------------

    local events = {}

    -- ADDON_LOADED: initialize DB and create the frame.
    function events:ADDON_LOADED(name)
        if name ~= addonName then return end
        ns:InitDB()
        ns:CreateDemoFrame()
        self:UnregisterEvent("ADDON_LOADED")
    end

    -- PLAYER_REGEN_DISABLED fires when combat lockdown BEGINS.
    -- After this event, InCombatLockdown() returns true.
    -- Protected operations (Show/Hide/SetPoint on secure frames) will be blocked.
    function events:PLAYER_REGEN_DISABLED()
        -- Update the status text to show we're locked.
        if ns.statusText then
            ns.statusText:SetText("|cffff4444In Combat (Locked)|r")
        end
        ns:Print("Combat started — frame operations will be queued.")
    end

    -- PLAYER_REGEN_ENABLED fires when combat lockdown ENDS.
    -- After this event, InCombatLockdown() returns false.
    -- This is where we replay all queued actions.
    function events:PLAYER_REGEN_ENABLED()
        -- Count how many actions are queued (for the status message).
        local queuedCount = #pendingActions

        -- Process all queued actions now that we're out of combat.
        ProcessPendingActions()

        -- Update the status text.
        if ns.statusText then
            ns.statusText:SetText("|cff00ff00Safe|r")
        end

        -- Log how many actions were applied (useful for debugging).
        if queuedCount > 0 then
            ns:Print("Combat ended — applied " .. queuedCount
                .. " queued action(s).")
        end
    end

    -- PLAYER_LOGIN: register slash commands.
    function events:PLAYER_LOGIN()
        SLASH_COMBATSAFE1 = "/combatsafe"
        SLASH_COMBATSAFE2 = "/cs"
        SlashCmdList["COMBATSAFE"] = function(msg)
            -- Parse the first word as the command.
            local cmd = (msg:match("^(%S*)") or ""):lower()

            if cmd == "show" then
                -- Show the frame (queued if in combat).
                ns:ShowFrame()
                -- Tell the user what happened.
                ns:Print("Frame " .. (InCombatLockdown()
                    and "queued to show" or "shown") .. ".")

            elseif cmd == "hide" then
                -- Hide the frame (queued if in combat).
                ns:HideFrame()
                ns:Print("Frame " .. (InCombatLockdown()
                    and "queued to hide" or "hidden") .. ".")

            elseif cmd == "scale" then
                -- Parse a numeric argument: /cs scale 1.5
                local value = tonumber(msg:match("%S+%s+(%S+)"))
                if value and value >= 0.5 and value <= 2.0 then
                    -- Set scale (queued if in combat).
                    ns:SetScale(value)
                    ns:Print("Scale " .. (InCombatLockdown()
                        and "queued" or "set") .. " to " .. value)
                else
                    ns:Print("Usage: /cs scale <0.5-2.0>")
                end

            else
                -- Show help and current combat state.
                ns:Print("Commands:")
                print("  /cs show         — Show the demo frame")
                print("  /cs hide         — Hide the demo frame")
                print("  /cs scale <num>  — Set scale (0.5-2.0)")
                -- Display current combat state.
                print("  State: " .. (InCombatLockdown()
                    and "|cffff4444COMBAT|r" or "|cff00ff00SAFE|r"))
                -- Display number of pending actions.
                print("  Pending: " .. #pendingActions .. " action(s)")
            end
        end
    end

    -- Wire up the event frame.
    local eventFrame = CreateFrame("Frame")
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then events[event](self, ...) end
    end)
    for ev in pairs(events) do
        eventFrame:RegisterEvent(ev)
    end
    ```

---

## 7. Complete Production Addon (Multi-File)

A full production-ready addon combining **all patterns**: multi-file namespace, SavedVariables with defaults merge, table-dispatch events, settings frame with BackdropTemplate, addon compartment, addon communication, combat-safe frame management, and slash commands.

### File Structure

```
Interface/AddOns/ProductionAddon/
├── ProductionAddon.toc
├── Init.lua          ← Namespace, defaults, utilities, combat queue
├── Core.lua          ← Events, slash commands, addon communication
└── Config.lua        ← Settings UI, compartment handlers
```

=== "ProductionAddon.toc"

    ```toc
    ## Interface: 120001
    ## -- Target Midnight 12.0.1 client.
    ## Title: Production Addon
    ## -- Display name in the addon list.
    ## Notes: A complete multi-file addon demonstrating all core patterns.
    ## -- Tooltip in the addon list.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Miscellaneous
    ## -- Groups addon in the Addon List (11.1.0+).

    ## -- Account-wide saved variables (persisted across sessions).
    ## SavedVariables: ProductionAddonDB
    ## -- Per-character saved variables (separate copy for each character).
    ## SavedVariablesPerCharacter: ProductionAddonCharDB

    ## -- Icon shown in the addon list (10.1.0+).
    ## IconTexture: Interface\Icons\Achievement_General

    ## -- ADDON COMPARTMENT registration.
    ## -- These point to global Lua function names defined in Config.lua.
    ## -- The game calls them when the user interacts with the minimap dropdown entry.
    ## AddonCompartmentFunc: ProductionAddon_OnCompartmentClick
    ## AddonCompartmentFuncOnEnter: ProductionAddon_OnCompartmentEnter
    ## AddonCompartmentFuncOnLeave: ProductionAddon_OnCompartmentLeave

    # -----------------------------------------------------------------------
    # FILE LOAD ORDER
    # Files execute top-to-bottom, synchronously, in a single Lua state.
    # Init.lua sets up the namespace and utilities that Core.lua and
    # Config.lua depend on.
    # -----------------------------------------------------------------------

    # First: namespace, defaults, utilities, combat queue
    Init.lua
    # Second: events, slash commands, addon communication
    Core.lua
    # Third: settings UI, compartment handlers
    Config.lua
    ```

=== "Init.lua"

    ```lua
    -- Init.lua
    -- FIRST file loaded. Sets up the shared namespace, default values,
    -- utility functions, and the combat-safe action queue. All other files
    -- depend on this. No events or UI are created here.

    -- Every .lua file receives two values via `...`:
    --   addonName: string matching the folder name ("ProductionAddon")
    --   ns: a private table shared across all files in this addon
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- ADDON IDENTITY
    -- Store the addon name on the namespace so other files can reference it
    -- without re-capturing the `...` varargs.
    -- ---------------------------------------------------------------------------

    -- The addon's folder name (used in prints, event checks, etc.).
    ns.addonName = addonName
    -- Read version from the TOC at runtime — single source of truth.
    -- C_AddOns.GetAddOnMetadata returns the value of any ## field.
    ns.version = C_AddOns.GetAddOnMetadata(addonName, "Version") or "unknown"

    -- ---------------------------------------------------------------------------
    -- ADDON COMMUNICATION PREFIX
    -- Used by C_ChatInfo.SendAddonMessage to tag our messages.
    -- Must be <= 16 characters.
    -- ---------------------------------------------------------------------------

    ns.PREFIX = "ProdAddon"

    -- ---------------------------------------------------------------------------
    -- PRINT UTILITIES
    -- ---------------------------------------------------------------------------

    -- Print a colored message prefixed with the addon name.
    function ns:Print(msg)
        -- |cff33ccff = cyan, |r = reset.
        print("|cff33ccff" .. ns.addonName .. ":|r " .. msg)
    end

    -- Conditional debug output — only prints when ns.db.debug is true.
    function ns:Debug(...)
        -- Early-out if debug is off or DB not yet loaded.
        if not (ns.db and ns.db.debug) then return end
        local parts = {}
        for i = 1, select("#", ...) do
            parts[#parts + 1] = tostring(select(i, ...))
        end
        print("|cffff9900[" .. ns.addonName .. " Debug]|r "
            .. table.concat(parts, " "))
    end

    -- ---------------------------------------------------------------------------
    -- DEFAULT SETTINGS
    -- These tables define the initial state for new users. When a new setting
    -- is added in an update, the merge logic fills it in automatically.
    -- ---------------------------------------------------------------------------

    -- Account-wide defaults (shared by all characters).
    ns.DEFAULTS = {
        debug           = false,  -- show debug output in chat
        welcomeMessage  = true,   -- greet the player on login
        featureEnabled  = true,   -- master on/off toggle
        notifications   = true,   -- show notification popups
        volume          = 50,     -- notification volume (0-100)
        showInfoFrame   = true,   -- display the status indicator
        infoFrameScale  = 1.0,    -- scale of the status frame
        infoFrameAlpha  = 0.85,   -- opacity of the status frame background
        locked          = false,  -- prevent the status frame from being dragged
    }

    -- Per-character defaults (each character gets their own copy).
    ns.CHAR_DEFAULTS = {
        enabled   = true,  -- per-character on/off toggle
        lastLogin = nil,   -- timestamp of last login (nil until first)
    }

    -- ---------------------------------------------------------------------------
    -- SAVED VARIABLES INITIALIZATION
    -- Merges defaults into existing saved data without overwriting user choices.
    -- Safe to call multiple times (e.g. after a /reset command).
    -- ---------------------------------------------------------------------------

    -- Merge helper: fills missing keys from defaults into saved.
    function ns:MergeDefaults(saved, defaults)
        for key, value in pairs(defaults) do
            -- Check for nil specifically so intentional false is preserved.
            if saved[key] == nil then
                if type(value) == "table" then
                    -- Deep-copy tables to avoid sharing references.
                    saved[key] = CopyTable(value)
                else
                    saved[key] = value
                end
            end
        end
    end

    -- Initialize (or re-initialize) saved variables.
    function ns:InitSavedVars()
        -- Account-wide: create if missing, then merge defaults.
        ProductionAddonDB = ProductionAddonDB or {}
        ns:MergeDefaults(ProductionAddonDB, ns.DEFAULTS)
        -- Shortcut for convenient access: ns.db.debug, ns.db.volume, etc.
        ns.db = ProductionAddonDB

        -- Per-character: same process.
        ProductionAddonCharDB = ProductionAddonCharDB or {}
        ns:MergeDefaults(ProductionAddonCharDB, ns.CHAR_DEFAULTS)
        -- Shortcut: ns.charDB.enabled, ns.charDB.lastLogin, etc.
        ns.charDB = ProductionAddonCharDB
    end

    -- ---------------------------------------------------------------------------
    -- COMBAT-SAFE ACTION QUEUE
    -- Many frame operations are forbidden during combat lockdown.
    -- This utility queues functions and replays them when combat ends.
    -- ---------------------------------------------------------------------------

    -- List of zero-argument functions waiting to run.
    ns.pendingActions = {}

    -- Queue a function to run outside combat. Runs immediately if safe.
    function ns:QueueAction(fn)
        if InCombatLockdown() then
            -- In combat: store for later.
            tinsert(ns.pendingActions, fn)
        else
            -- Out of combat: run now.
            fn()
        end
    end

    -- Execute all queued actions. Called on PLAYER_REGEN_ENABLED.
    function ns:ProcessPendingActions()
        -- Safety guard: don't run if somehow still in combat.
        if InCombatLockdown() then return end
        -- Execute each action in order.
        for _, action in ipairs(ns.pendingActions) do
            action()
        end
        -- Clear the queue.
        wipe(ns.pendingActions)
    end

    -- ---------------------------------------------------------------------------
    -- ADDON MESSAGE QUEUE (encounter-safe communication)
    -- SendAddonMessage is restricted during active encounters in Midnight.
    -- ---------------------------------------------------------------------------

    -- Queued messages: each entry is {message, channel, target}.
    ns.messageQueue = {}

    -- Send an addon message; queues if an encounter is active.
    -- Returns true if sent, false if queued.
    function ns:SendAddonMsg(message, channel, target)
        -- Check if we're in a boss encounter.
        if IsEncounterInProgress and IsEncounterInProgress() then
            tinsert(ns.messageQueue, { message, channel, target })
            return false
        end
        -- Send immediately.
        C_ChatInfo.SendAddonMessage(ns.PREFIX, message, channel, target)
        return true
    end

    -- Flush queued messages (called when an encounter ends).
    function ns:FlushMessageQueue()
        for _, entry in ipairs(ns.messageQueue) do
            C_ChatInfo.SendAddonMessage(ns.PREFIX, unpack(entry))
        end
        wipe(ns.messageQueue)
    end
    ```

=== "Core.lua"

    ```lua
    -- Core.lua
    -- SECOND file loaded. Handles all events, slash commands, and
    -- addon-to-addon communication. Depends on Init.lua for the namespace,
    -- defaults, and utility functions.

    -- Capture the same addon name and namespace from Init.lua.
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- EVENT DISPATCH TABLE
    -- Each key is a WoW event name. The OnEvent handler below dispatches
    -- to the matching function automatically.
    -- ---------------------------------------------------------------------------

    local events = {}

    -- ---------------------------------------------------------------------------
    -- INITIALIZATION EVENTS
    -- ---------------------------------------------------------------------------

    -- ADDON_LOADED: fires once per addon when its files + saved vars are ready.
    function events:ADDON_LOADED(loadedName)
        -- Only respond to our own addon loading.
        if loadedName ~= addonName then return end

        -- Initialize saved variables (defined in Init.lua).
        ns:InitSavedVars()

        -- Register our addon message prefix for communication.
        -- Must happen before we can send or receive with this prefix.
        C_ChatInfo.RegisterAddonMessagePrefix(ns.PREFIX)

        -- Debug: confirm initialization.
        ns:Debug("Saved variables initialized. Prefix registered.")

        -- Only need this once.
        self:UnregisterEvent("ADDON_LOADED")
    end

    -- PLAYER_LOGIN: fires once after ALL addons are loaded.
    -- Character data is fully available at this point.
    function events:PLAYER_LOGIN()
        -- Record per-character login data.
        -- date() formats the current date/time as a string.
        ns.charDB.lastLogin = date("%Y-%m-%d %H:%M:%S")

        -- Build the config UI (defined in Config.lua).
        -- Config.lua is loaded after Core.lua, but since we're calling this
        -- from an event handler (not top-level code), the function exists
        -- by the time this runs.
        if ns.CreateConfigPanel then
            ns:CreateConfigPanel()
        end

        -- Register slash commands.
        ns:RegisterSlashCommands()

        -- Show a welcome message if the user hasn't turned it off.
        if ns.db.welcomeMessage then
            ns:Print("v" .. ns.version .. " loaded. Welcome, "
                .. UnitName("player") .. "!")
        end
    end

    -- PLAYER_ENTERING_WORLD: fires on login, /reload, and zone transitions.
    function events:PLAYER_ENTERING_WORLD(isInitialLogin, isReloadingUi)
        ns:Debug("PLAYER_ENTERING_WORLD",
            "initial=" .. tostring(isInitialLogin),
            "reload=" .. tostring(isReloadingUi))
    end

    -- ---------------------------------------------------------------------------
    -- COMBAT STATE EVENTS
    -- ---------------------------------------------------------------------------

    -- PLAYER_REGEN_DISABLED: combat lockdown begins.
    -- InCombatLockdown() returns true after this fires.
    function events:PLAYER_REGEN_DISABLED()
        -- Update the info frame if it exists (created in Config.lua).
        if ns.infoText then
            ns.infoText:SetText("|cffff4444In Combat|r")
        end
    end

    -- PLAYER_REGEN_ENABLED: combat lockdown ends.
    -- InCombatLockdown() returns false after this fires.
    function events:PLAYER_REGEN_ENABLED()
        -- Replay all queued frame operations (defined in Init.lua).
        ns:ProcessPendingActions()
        -- Update the info frame.
        if ns.infoText then
            ns.infoText:SetText("|cff00ff00Out of Combat|r")
        end
    end

    -- ---------------------------------------------------------------------------
    -- ADDON COMMUNICATION EVENTS
    -- ---------------------------------------------------------------------------

    -- CHAT_MSG_ADDON: received a message with a registered prefix.
    function events:CHAT_MSG_ADDON(prefix, message, channel, sender)
        -- Only process our own prefix.
        if prefix ~= ns.PREFIX then return end
        -- Ignore messages from ourselves.
        local myName = UnitName("player")
        local senderName = sender:match("^([^%-]+)")
        if senderName == myName then return end

        -- Handle known message types.
        if message == "PING" then
            ns:Print("PING from " .. sender .. " (" .. channel .. ")")
            -- Auto-reply with PONG.
            ns:SendAddonMsg("PONG", channel)
        elseif message == "PONG" then
            ns:Print("PONG from " .. sender)
        else
            -- Unknown — log if in debug mode.
            ns:Debug("[" .. channel .. "] " .. sender .. ": " .. message)
        end
    end

    -- ENCOUNTER_END: a raid boss encounter finished.
    function events:ENCOUNTER_END(encounterID, encounterName, difficultyID,
                                   groupSize, success)
        -- Flush any addon messages that were queued during the fight.
        ns:FlushMessageQueue()
    end

    -- ---------------------------------------------------------------------------
    -- EVENT FRAME WIRING
    -- ---------------------------------------------------------------------------

    -- Create a single invisible frame to handle all events.
    local eventFrame = CreateFrame("Frame")

    -- The OnEvent handler: dispatch to the table.
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        if events[event] then
            events[event](self, ...)
        end
    end)

    -- Auto-register every event that has a handler in the dispatch table.
    for eventName in pairs(events) do
        eventFrame:RegisterEvent(eventName)
    end

    -- ---------------------------------------------------------------------------
    -- SLASH COMMANDS
    -- Called from PLAYER_LOGIN to ensure saved variables are ready.
    -- ---------------------------------------------------------------------------

    function ns:RegisterSlashCommands()
        -- Two aliases: full name and abbreviation.
        SLASH_PRODUCTIONADDON1 = "/productionaddon"
        SLASH_PRODUCTIONADDON2 = "/pa"

        -- The handler receives everything typed after the slash command.
        SlashCmdList["PRODUCTIONADDON"] = function(msg)
            -- Split command from arguments.
            local cmd, rest = msg:match("^(%S*)%s*(.-)$")
            cmd = (cmd or ""):lower()

            if cmd == "config" or cmd == "options" then
                -- Toggle the settings panel.
                ns:ToggleConfig()

            elseif cmd == "reset" then
                -- Wipe all saved data and re-initialize.
                wipe(ProductionAddonDB)
                wipe(ProductionAddonCharDB)
                ns:InitSavedVars()
                -- Refresh the config panel if it's open.
                if ns.configPanel and ns.configPanel:IsShown() then
                    ns:RefreshControls()
                end
                ns:Print("All settings reset to defaults.")

            elseif cmd == "toggle" then
                -- Toggle the master enable flag.
                ns.db.featureEnabled = not ns.db.featureEnabled
                local state = ns.db.featureEnabled
                    and "|cff00ff00ENABLED|r" or "|cffff0000DISABLED|r"
                ns:Print("Addon " .. state)

            elseif cmd == "ping" then
                -- Send a PING to the group.
                local channel = IsInRaid() and "RAID" or "PARTY"
                if ns:SendAddonMsg("PING", channel) then
                    ns:Print("Sent PING to " .. channel)
                else
                    ns:Print("PING queued (encounter in progress)")
                end

            elseif cmd == "debug" then
                -- Toggle debug output.
                ns.db.debug = not ns.db.debug
                local state = ns.db.debug
                    and "|cff00ff00ON|r" or "|cffff0000OFF|r"
                ns:Print("Debug mode " .. state)

            elseif cmd == "status" then
                -- Display current status.
                ns:Print("v" .. ns.version .. " Status:")
                print("  Feature: " .. (ns.db.featureEnabled
                    and "|cff00ff00ON|r" or "|cffff0000OFF|r"))
                print("  Debug:   " .. (ns.db.debug
                    and "|cff00ff00ON|r" or "|cffff0000OFF|r"))
                print("  Combat:  " .. (InCombatLockdown()
                    and "|cffff4444LOCKED|r" or "|cff00ff00SAFE|r"))
                print("  Queued:  " .. #ns.pendingActions .. " action(s)")
                print("  Last login: " .. (ns.charDB.lastLogin or "never"))

            else
                -- Show help.
                ns:Print("v" .. ns.version .. " — Commands:")
                print("  /pa config  — Open settings")
                print("  /pa toggle  — Enable/disable addon")
                print("  /pa ping    — Ping group members")
                print("  /pa debug   — Toggle debug output")
                print("  /pa status  — Show current status")
                print("  /pa reset   — Reset all settings")
            end
        end
    end
    ```

=== "Config.lua"

    ```lua
    -- Config.lua
    -- THIRD and final file loaded. Creates the settings UI panel and
    -- the addon compartment handlers. Depends on ns.db being initialized
    -- in Init.lua (but called from PLAYER_LOGIN in Core.lua, so timing is safe).

    -- Same namespace — all three files share this table.
    local addonName, ns = ...

    -- ---------------------------------------------------------------------------
    -- SETTINGS PANEL CREATION
    -- Called from Core.lua's PLAYER_LOGIN handler.
    -- ---------------------------------------------------------------------------

    function ns:CreateConfigPanel()
        -- Guard: only create once.
        if ns.configPanel then return end

        -- ----- INFO FRAME (small always-visible status indicator) -----

        -- Create the info frame with a backdrop.
        local infoFrame = CreateFrame("Frame", "ProductionAddonInfoFrame",
            UIParent, "BackdropTemplate")
        -- Small frame near the top of the screen.
        infoFrame:SetSize(180, 40)
        infoFrame:SetPoint("TOP", UIParent, "TOP", 0, -80)
        infoFrame:SetBackdrop({
            bgFile   = "Interface\\Tooltips\\UI-Tooltip-Background",
            edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
            tile = true, tileSize = 16, edgeSize = 16,
            insets = { left = 4, right = 4, top = 4, bottom = 4 },
        })
        -- Dark background with saved alpha.
        infoFrame:SetBackdropColor(0.05, 0.05, 0.15, ns.db.infoFrameAlpha)
        -- Themed border color.
        infoFrame:SetBackdropBorderColor(0.53, 0.33, 1.0, 1.0)
        -- Apply saved scale.
        infoFrame:SetScale(ns.db.infoFrameScale)

        -- Make the info frame draggable.
        infoFrame:EnableMouse(true)
        infoFrame:SetMovable(true)
        infoFrame:SetClampedToScreen(true)
        infoFrame:RegisterForDrag("LeftButton")
        -- Only allow dragging when not locked.
        infoFrame:SetScript("OnDragStart", function(self)
            if not ns.db.locked then self:StartMoving() end
        end)
        infoFrame:SetScript("OnDragStop", function(self)
            self:StopMovingOrSizing()
        end)

        -- Status text inside the info frame.
        local infoText = infoFrame:CreateFontString(nil, "OVERLAY",
            "GameFontNormal")
        infoText:SetPoint("CENTER")
        -- Initial text — will be updated by combat events in Core.lua.
        infoText:SetText("|cff00ff00Out of Combat|r")

        -- Store references on the namespace for cross-file access.
        ns.infoFrame = infoFrame
        ns.infoText  = infoText

        -- Apply initial visibility from saved settings.
        if ns.db.showInfoFrame then
            infoFrame:Show()
        else
            infoFrame:Hide()
        end

        -- ----- SETTINGS PANEL (draggable, ESC-to-close) -----

        -- Main settings frame with backdrop.
        local panel = CreateFrame("Frame", "ProductionAddonConfigPanel",
            UIParent, "BackdropTemplate")
        panel:SetSize(360, 380)
        panel:SetPoint("CENTER")
        panel:SetBackdrop({
            bgFile   = "Interface\\Tooltips\\UI-Tooltip-Background",
            edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
            tile = true, tileSize = 16, edgeSize = 16,
            insets = { left = 4, right = 4, top = 4, bottom = 4 },
        })
        -- Dark background, nearly opaque.
        panel:SetBackdropColor(0.08, 0.08, 0.12, 0.95)
        -- Themed border.
        panel:SetBackdropBorderColor(0.53, 0.33, 1.0, 1.0)
        -- DIALOG strata: above most game UI.
        panel:SetFrameStrata("DIALOG")

        -- Make it draggable.
        panel:SetMovable(true)
        panel:EnableMouse(true)
        panel:SetClampedToScreen(true)
        panel:RegisterForDrag("LeftButton")
        panel:SetScript("OnDragStart", panel.StartMoving)
        panel:SetScript("OnDragStop", panel.StopMovingOrSizing)

        -- ESC-to-close: register with UISpecialFrames.
        tinsert(UISpecialFrames, "ProductionAddonConfigPanel")

        -- Start hidden.
        panel:Hide()

        -- ----- TITLE -----
        local title = panel:CreateFontString(nil, "OVERLAY",
            "GameFontNormalLarge")
        title:SetPoint("TOP", panel, "TOP", 0, -14)
        title:SetText(ns.addonName .. " Settings")

        -- Version subtitle.
        local version = panel:CreateFontString(nil, "OVERLAY",
            "GameFontNormalSmall")
        version:SetPoint("TOP", title, "BOTTOM", 0, -2)
        version:SetText("v" .. ns.version)
        version:SetTextColor(0.6, 0.6, 0.6)

        -- ----- CLOSE BUTTON -----
        local closeBtn = CreateFrame("Button", nil, panel,
            "UIPanelCloseButton")
        closeBtn:SetPoint("TOPRIGHT", panel, "TOPRIGHT", -2, -2)
        closeBtn:SetScript("OnClick", function() panel:Hide() end)

        -- ----- CHECKBOX HELPER -----
        -- Creates a checkbox anchored relative to the panel.
        local function MakeCheck(globalName, label, yOffset, dbKey)
            local cb = CreateFrame("CheckButton", globalName, panel,
                "InterfaceOptionsCheckButtonTemplate")
            cb:SetPoint("TOPLEFT", panel, "TOPLEFT", 20, yOffset)
            cb.Text:SetText(label)
            cb:SetChecked(ns.db[dbKey])
            cb:SetScript("OnClick", function(self)
                ns.db[dbKey] = self:GetChecked()
            end)
            return cb
        end

        -- ----- CHECKBOXES -----
        -- Each checkbox is anchored at a fixed Y offset from the panel top.
        local cbFeature = MakeCheck("ProdCBFeature",
            "Feature Enabled", -55, "featureEnabled")

        local cbWelcome = MakeCheck("ProdCBWelcome",
            "Show welcome message", -85, "welcomeMessage")

        local cbNotify = MakeCheck("ProdCBNotify",
            "Show notifications", -115, "notifications")

        -- "Show Info Frame" uses combat-safe toggle.
        local cbInfo = CreateFrame("CheckButton", "ProdCBInfo", panel,
            "InterfaceOptionsCheckButtonTemplate")
        cbInfo:SetPoint("TOPLEFT", panel, "TOPLEFT", 20, -145)
        cbInfo.Text:SetText("Show status frame")
        cbInfo:SetChecked(ns.db.showInfoFrame)
        cbInfo:SetScript("OnClick", function(self)
            ns.db.showInfoFrame = self:GetChecked()
            -- Use combat-safe queue to show/hide the info frame.
            ns:QueueAction(function()
                if ns.db.showInfoFrame then
                    ns.infoFrame:Show()
                else
                    ns.infoFrame:Hide()
                end
            end)
        end)

        local cbLocked = MakeCheck("ProdCBLocked",
            "Lock frame position", -175, "locked")

        local cbDebug = MakeCheck("ProdCBDebug",
            "Debug mode", -205, "debug")

        -- ----- SLIDER: Volume -----
        local volSlider = CreateFrame("Slider", "ProdVolSlider", panel,
            "OptionsSliderTemplate")
        volSlider:SetPoint("TOPLEFT", panel, "TOPLEFT", 25, -255)
        volSlider:SetWidth(200)
        volSlider:SetHeight(17)
        volSlider:SetMinMaxValues(0, 100)
        volSlider:SetValueStep(1)
        volSlider:SetObeyStepOnDrag(true)
        volSlider:SetValue(ns.db.volume)
        volSlider.Low:SetText("0")
        volSlider.High:SetText("100")
        volSlider.Text:SetText("Volume: " .. ns.db.volume)
        volSlider:SetScript("OnValueChanged", function(self, value)
            value = math.floor(value + 0.5)
            ns.db.volume = value
            self.Text:SetText("Volume: " .. value)
        end)

        -- ----- SLIDER: Info Frame Scale -----
        local scaleSlider = CreateFrame("Slider", "ProdScaleSlider", panel,
            "OptionsSliderTemplate")
        scaleSlider:SetPoint("TOPLEFT", volSlider, "BOTTOMLEFT", 0, -30)
        scaleSlider:SetWidth(200)
        scaleSlider:SetHeight(17)
        scaleSlider:SetMinMaxValues(0.5, 2.0)
        scaleSlider:SetValueStep(0.05)
        scaleSlider:SetObeyStepOnDrag(true)
        scaleSlider:SetValue(ns.db.infoFrameScale)
        scaleSlider.Low:SetText("50%")
        scaleSlider.High:SetText("200%")
        scaleSlider.Text:SetText("Frame Scale: "
            .. math.floor(ns.db.infoFrameScale * 100) .. "%")
        scaleSlider:SetScript("OnValueChanged", function(self, value)
            -- Round to avoid floating-point drift.
            value = math.floor(value * 100 + 0.5) / 100
            ns.db.infoFrameScale = value
            self.Text:SetText("Frame Scale: " .. math.floor(value * 100) .. "%")
            -- Apply scale via the combat-safe queue.
            ns:QueueAction(function()
                ns.infoFrame:SetScale(value)
            end)
        end)

        -- ----- RESET BUTTON -----
        local resetBtn = CreateFrame("Button", nil, panel,
            "UIPanelButtonTemplate")
        resetBtn:SetSize(100, 24)
        resetBtn:SetPoint("BOTTOMLEFT", panel, "BOTTOMLEFT", 20, 16)
        resetBtn:SetText("Reset")
        resetBtn:SetScript("OnClick", function()
            -- Wipe and re-initialize everything.
            wipe(ProductionAddonDB)
            wipe(ProductionAddonCharDB)
            ns:InitSavedVars()
            -- Refresh all controls.
            ns:RefreshControls()
            ns:Print("Settings reset to defaults.")
        end)

        -- ----- DONE BUTTON -----
        local doneBtn = CreateFrame("Button", nil, panel,
            "UIPanelButtonTemplate")
        doneBtn:SetSize(80, 24)
        doneBtn:SetPoint("BOTTOMRIGHT", panel, "BOTTOMRIGHT", -20, 16)
        doneBtn:SetText("Done")
        doneBtn:SetScript("OnClick", function() panel:Hide() end)

        -- Store references.
        panel:Hide()
        ns.configPanel = panel
        -- Store control references for RefreshControls.
        ns.controls = {
            cbFeature    = cbFeature,
            cbWelcome    = cbWelcome,
            cbNotify     = cbNotify,
            cbInfo       = cbInfo,
            cbLocked     = cbLocked,
            cbDebug      = cbDebug,
            volSlider    = volSlider,
            scaleSlider  = scaleSlider,
        }
    end

    -- ---------------------------------------------------------------------------
    -- REFRESH CONTROLS (after reset or external change)
    -- Updates all UI controls to match current ns.db values.
    -- ---------------------------------------------------------------------------

    function ns:RefreshControls()
        if not ns.controls then return end
        local c = ns.controls
        -- Update each checkbox to match the current saved value.
        c.cbFeature:SetChecked(ns.db.featureEnabled)
        c.cbWelcome:SetChecked(ns.db.welcomeMessage)
        c.cbNotify:SetChecked(ns.db.notifications)
        c.cbInfo:SetChecked(ns.db.showInfoFrame)
        c.cbLocked:SetChecked(ns.db.locked)
        c.cbDebug:SetChecked(ns.db.debug)
        -- Update sliders.
        c.volSlider:SetValue(ns.db.volume)
        c.scaleSlider:SetValue(ns.db.infoFrameScale)
        -- Apply visual changes to the info frame (combat-safe).
        ns:QueueAction(function()
            ns.infoFrame:SetScale(ns.db.infoFrameScale)
            ns.infoFrame:SetBackdropColor(0.05, 0.05, 0.15,
                ns.db.infoFrameAlpha)
            if ns.db.showInfoFrame then
                ns.infoFrame:Show()
            else
                ns.infoFrame:Hide()
            end
        end)
    end

    -- ---------------------------------------------------------------------------
    -- TOGGLE CONFIG (called from slash commands and compartment click)
    -- ---------------------------------------------------------------------------

    function ns:ToggleConfig()
        if not ns.configPanel then return end
        if ns.configPanel:IsShown() then
            ns.configPanel:Hide()
        else
            -- Refresh controls before showing (in case settings changed via CLI).
            ns:RefreshControls()
            ns.configPanel:Show()
        end
    end

    -- ---------------------------------------------------------------------------
    -- ADDON COMPARTMENT HANDLERS
    -- These are GLOBAL functions. The names MUST exactly match the TOC fields.
    -- ---------------------------------------------------------------------------

    -- Called on left-click or right-click of the compartment entry.
    function ProductionAddon_OnCompartmentClick(name, mouseButton)
        if mouseButton == "LeftButton" then
            -- Left-click: toggle the settings panel.
            ns:ToggleConfig()
        elseif mouseButton == "RightButton" then
            -- Right-click: send a ping to the group.
            local channel = IsInRaid() and "RAID" or "PARTY"
            if ns:SendAddonMsg("PING", channel) then
                ns:Print("Sent PING to " .. channel)
            end
        end
    end

    -- Called when the mouse enters the compartment entry (tooltip).
    function ProductionAddon_OnCompartmentEnter(name, menuButtonFrame)
        -- Show a rich tooltip with addon info.
        GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
        GameTooltip:SetText(ns.addonName)
        GameTooltip:AddLine("v" .. ns.version, 0.6, 0.6, 0.6)
        GameTooltip:AddLine(" ")
        GameTooltip:AddLine("Left-click to open settings", 1, 1, 1)
        GameTooltip:AddLine("Right-click to ping group", 1, 1, 1)
        GameTooltip:AddLine(" ")
        -- Show status.
        local status = ns.db.featureEnabled
            and "|cff00ff00Enabled|r" or "|cffff0000Disabled|r"
        GameTooltip:AddDoubleLine("Status:", status, 1, 0.82, 0)
        -- Show combat state.
        local combat = InCombatLockdown()
            and "|cffff4444Combat|r" or "|cff00ff00Safe|r"
        GameTooltip:AddDoubleLine("Combat:", combat, 1, 0.82, 0)
        GameTooltip:Show()
    end

    -- Called when the mouse leaves the compartment entry.
    function ProductionAddon_OnCompartmentLeave(name, menuButtonFrame)
        GameTooltip:Hide()
    end
    ```

!!! warning "Legacy Template"
    `InterfaceOptionsCheckButtonTemplate` is a legacy template from the deprecated InterfaceOptions system. It still works for custom settings frames (as shown above), but for integrating with the Blizzard settings panel use the modern Settings API (`Settings.RegisterAddOnSetting` + `Settings.CreateCheckbox`). See the [Blizzard Systems guide](../blizzard-systems.md) for the modern approach.

!!! tip "File load order matters"
    The TOC lists files in load order. `Init.lua` defines shared utilities and the namespace. `Core.lua` handles events and slash commands, calling functions defined in both `Init.lua` and `Config.lua`. `Config.lua` creates the UI panel. Because `Config.lua` defines `ns:CreateConfigPanel()` as a function (not immediate execution), `Core.lua` can safely call it during `PLAYER_LOGIN` even though `Config.lua` is listed last — the function exists by the time the event fires.

---

## Quick Reference: Common Patterns

| Pattern | Key Concept | Template |
|---------|------------|----------|
| Namespace | `local addonName, ns = ...` | All templates |
| Table dispatch | `events[event](self, ...)` | [#2](#2-savedvariables-addon), [#7](#7-complete-production-addon-multi-file) |
| Defaults merge | Loop `pairs(defaults)`, check `nil`, `CopyTable` for tables | [#2](#2-savedvariables-addon), [#7](#7-complete-production-addon-multi-file) |
| ESC-to-close | `tinsert(UISpecialFrames, name)` + `OnKeyDown` | [#3](#3-movable-settings-frame), [#7](#7-complete-production-addon-multi-file) |
| Combat queue | `InCombatLockdown()` check, execute on `PLAYER_REGEN_ENABLED` | [#6](#6-combat-lockdown-safe-pattern), [#7](#7-complete-production-addon-multi-file) |
| Addon Compartment | TOC directives + global callback functions | [#4](#4-minimap-button--addon-compartment), [#7](#7-complete-production-addon-multi-file) |
| Addon messaging | `C_ChatInfo.RegisterAddonMessagePrefix` + `CHAT_MSG_ADDON` | [#5](#5-addon-communication), [#7](#7-complete-production-addon-multi-file) |
