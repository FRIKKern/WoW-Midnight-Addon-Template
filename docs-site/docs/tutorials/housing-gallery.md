---
title: "Build-Along: HomeSnap — Housing Photo Gallery"
description: "Step-by-step tutorial building a housing photo gallery addon using C_HousingPhotoSharing, ScrollBox, and addon messaging for WoW Midnight 12.0.1+."
---

# Build-Along: HomeSnap — Housing Photo Gallery

:material-palette: **Mode:** Enhancement Artist | :material-clock-outline: **Time:** 25 minutes | :material-star-half-full: **Difficulty:** Beginner-Intermediate

---

## Step 1: What We're Building

Player housing arrived in World of Warcraft with Midnight (Patch 12.0) — the first time in the game's twenty-year history that players can build, decorate, and visit persistent homes. With Patch 12.0.1, Blizzard expanded the housing API with `C_HousingPhotoSharing`, a brand-new namespace that lets addons capture, browse, and share screenshots of player houses.

We're going to build **HomeSnap** — a photo gallery addon that:

- **Browses** housing screenshots from houses you've visited
- **Shares** photos with guildmates via addon messaging
- **Organizes** photos by house theme, owner, and date
- **Integrates** into Blizzard's housing UI with an Enhancement Artist-style hook

Along the way, you'll learn:

- **Housing APIs** — `C_Housing`, `C_HousingDecor`, and `C_HousingPhotoSharing`
- **ScrollBox with DataProvider** — the modern scrollable list system (replacement for the ancient `FauxScrollFrame`)
- **Addon messaging** — `C_ChatInfo.SendAddonMessage` for guild-wide photo sharing
- **Frame skinning** — hooking Blizzard's housing UI to add a Gallery button without replacing anything

!!! danger "12.0.1 Required"
    The `C_HousingPhotoSharing` API was added in Patch 12.0.1. If you're on 12.0.0, the photo sharing functions will not exist. Check your game version in the login screen or with `GetBuildInfo()` in-game.

!!! warning "New API"
    Housing APIs are very new. While `C_Housing` and `C_HousingDecor` have been stable since 12.0 launch, `C_HousingPhotoSharing` is a 12.0.1 addition. Function signatures or event names may change in future patches. Use `/wow-research C_HousingPhotoSharing` and `/wow-verify` to confirm current API details before publishing.

---

## Step 2: Prerequisites

Before you start, make sure you have:

- **WoW 12.0.1 or later** — the housing photo sharing API does not exist in 12.0.0
- **A text editor** — VS Code, Notepad++, or any editor that can save UTF-8 files without BOM
- **Claude Code with the WoW addon plugin** — for `/wow-mode`, `/wow-research`, and `/wow-verify`
- **Basic Lua knowledge** — variables, tables, functions, and string formatting

Set your development mode to Enhancement Artist (it's the default, but let's be explicit):

```
/wow-mode artist
```

You should see confirmation that Enhancement Artist mode is active. This ensures all AI-assisted code follows the "enhance don't replace" philosophy.

!!! tip "New to WoW addon development?"
    If this is your first addon, read [Getting Started](../getting-started.md) and [Code Templates](../code-templates.md) first. This tutorial assumes you understand the `.toc` format, the namespace pattern (`local addonName, ns = ...`), and basic event handling.

---

## Step 3: Project Setup

Create the addon folder and files:

```
Interface/AddOns/HomeSnap/
├── HomeSnap.toc
├── Core.lua
├── UI.lua
└── Config.lua
```

### The TOC File

```toc
## Interface: 120001
## -- Requires Midnight 12.0.1 for C_HousingPhotoSharing
## Title: HomeSnap
## Notes: Housing photo gallery — browse, share, and organize screenshots of player houses.
## Author: YourName
## Version: 1.0.0
## Category: Housing
## -- Groups under the new Housing category in Addon List
## IconTexture: Interface\Icons\INV_Misc_Camera_02
## -- Camera icon in the Addon List
## SavedVariables: HomeSnapDB
## -- Account-wide: photo library, settings
## AddonCompartmentFunc: HomeSnap_OnAddonCompartmentClick
## -- Click minimap addon button to open gallery
## AddonCompartmentFuncOnEnter: HomeSnap_OnAddonCompartmentEnter
## AddonCompartmentFuncOnLeave: GameTooltip_Hide

# Load order matters: Core first (data), then UI (frames), then Config (settings)
Core.lua
UI.lua
Config.lua
```

!!! info "Interface: 120001"
    The `120001` interface version targets Patch 12.0.1 specifically. If you set this to `120000`, the addon will load on 12.0.0 — but `C_HousingPhotoSharing` won't exist there. Using `120001` ensures your addon only loads on clients that have the API.

### What Each File Does

| File | Responsibility |
|---|---|
| `Core.lua` | Namespace setup, event dispatch, photo data management, sharing logic |
| `UI.lua` | Gallery frame, ScrollBox, thumbnails, click handlers |
| `Config.lua` | Settings panel via Settings API, Addon Compartment handlers |

---

## Step 4: Understanding the Housing APIs

Housing in Midnight is split across several `C_` namespaces. Here's what matters for our addon:

### C_Housing — The Foundation

The base namespace for housing state. Use it to check whether the player is in a house and whose house it is:

```lua
-- Check if player is currently in a housing instance
local isInHouse = C_Housing.IsInHouse()

-- Get info about the current house
local houseInfo = C_Housing.GetCurrentHouseInfo()
-- houseInfo.ownerGUID, houseInfo.ownerName, houseInfo.houseName, houseInfo.houseTheme
```

### C_HousingDecor — Decoration Data

Covers decoration placement, inventory, and the housing catalog. We won't use this directly, but it's useful context — photos are tied to decorated houses.

```lua
-- Get all placed decorations in current house
local decorations = C_HousingDecor.GetPlacedDecorations()

-- Preview a decoration item
C_HousingDecor.PreviewDecorItem(itemID)
```

### C_HousingPhotoSharing — The Star of This Tutorial

Added in 12.0.1, this namespace handles everything about housing screenshots:

```lua
-- Get photos from a house visit
local photos = C_HousingPhotoSharing.GetVisitPhotos()
-- Returns: { { photoID, ownerName, houseName, timestamp, thumbnailFileID, ... }, ... }

-- Get a specific photo's full data (for sharing)
local photoData = C_HousingPhotoSharing.GetPhotoData(photoID)

-- Share a photo (triggers HOUSING_PHOTO_SHARED event on recipients)
C_HousingPhotoSharing.SharePhoto(photoData)

-- Get shared photos received from others
local sharedPhotos = C_HousingPhotoSharing.GetSharedPhotos()

-- Delete a photo from local storage
C_HousingPhotoSharing.DeletePhoto(photoID)
```

### Housing Events

These events fire during housing interactions:

| Event | When It Fires |
|---|---|
| `HOUSING_ENTERED` | Player enters a housing instance |
| `HOUSING_LEFT` | Player leaves a housing instance |
| `HOUSING_PHOTOS_UPDATED` | Photo library changes (new capture, delete) |
| `HOUSING_PHOTO_SHARED` | A shared photo arrives from another player |
| `HOUSING_PHOTO_SHARE_RESULT` | Result callback after calling `SharePhoto()` |

!!! warning "New API"
    These API signatures are based on 12.0.1. Housing APIs are under active development by Blizzard. Before publishing your addon, run `/wow-verify C_HousingPhotoSharing.GetVisitPhotos returns table of photo objects` to confirm the current signatures match what's documented here.

!!! tip "Research first, code second"
    When working with new APIs, always run `/wow-research C_HousingPhotoSharing` before writing code. The Research agent checks warcraft.wiki.gg, Townlong Yak, and other sources for the latest API documentation. For housing specifically, the APIs may receive additions or changes with each minor patch.

---

## Step 5: Core Logic — Photo Management

Open `Core.lua`. This file sets up the addon namespace, handles events, and manages the photo library.

```lua
-- Core.lua
-- Mode: Enhancement Artist | Enhance don't replace
-- Photo data management, event handling, and sharing logic.

local addonName, ns = ...

-- ---------------------------------------------------------------------------
-- DEFAULTS
-- ---------------------------------------------------------------------------

ns.defaults = {
    photos = {},           -- { [photoID] = { owner, house, theme, timestamp } }
    favorites = {},        -- { [photoID] = true }
    maxPhotos = 200,       -- storage limit
    autoShare = false,     -- auto-share new photos with guild
    thumbnailSize = 64,    -- thumbnail display size
}

-- ---------------------------------------------------------------------------
-- ADDON MESSAGE PREFIX
-- ---------------------------------------------------------------------------

local MSG_PREFIX = "HomeSnap"
local MSG_SHARE = "SHARE"
local MSG_REQUEST = "REQ"

-- ---------------------------------------------------------------------------
-- EVENT DISPATCH
-- ---------------------------------------------------------------------------

local events = {}

local eventFrame = CreateFrame("Frame")
eventFrame:SetScript("OnEvent", function(self, event, ...)
    local handler = events[event]
    if handler then handler(self, ...) end
end)

-- ---------------------------------------------------------------------------
-- INITIALIZATION
-- ---------------------------------------------------------------------------

function events:ADDON_LOADED(loadedAddon)
    if loadedAddon ~= addonName then return end

    -- Initialize saved variables
    HomeSnapDB = HomeSnapDB or CopyTable(ns.defaults)
    ns.db = HomeSnapDB

    -- Ensure new defaults are added on update
    for k, v in pairs(ns.defaults) do
        if ns.db[k] == nil then
            ns.db[k] = v
        end
    end

    -- Register addon message prefix for guild sharing
    C_ChatInfo.RegisterAddonMessagePrefix(MSG_PREFIX)

    self:UnregisterEvent("ADDON_LOADED")
end

-- ---------------------------------------------------------------------------
-- HOUSING EVENTS
-- ---------------------------------------------------------------------------

function events:HOUSING_ENTERED()
    ns.isInHouse = true
    local info = C_Housing.GetCurrentHouseInfo()
    if info then
        ns.currentHouse = {
            ownerName = info.ownerName,
            houseName = info.houseName,
            houseTheme = info.houseTheme,
            ownerGUID = info.ownerGUID,
        }
    end
end

function events:HOUSING_LEFT()
    ns.isInHouse = false
    ns.currentHouse = nil
end

function events:HOUSING_PHOTOS_UPDATED()
    -- Refresh our local photo index
    ns:RefreshPhotoLibrary()

    -- Auto-share if enabled
    if ns.db.autoShare and ns.isInHouse then
        ns:ShareLatestPhoto()
    end
end

function events:HOUSING_PHOTO_SHARED(senderName, photoID)
    -- A photo arrived from another player
    ns:OnPhotoReceived(senderName, photoID)
end

-- ---------------------------------------------------------------------------
-- PHOTO LIBRARY
-- ---------------------------------------------------------------------------

function ns:RefreshPhotoLibrary()
    local visitPhotos = C_HousingPhotoSharing.GetVisitPhotos() or {}
    local sharedPhotos = C_HousingPhotoSharing.GetSharedPhotos() or {}

    -- Index visit photos
    for _, photo in ipairs(visitPhotos) do
        if not self.db.photos[photo.photoID] then
            self.db.photos[photo.photoID] = {
                owner = photo.ownerName,
                house = photo.houseName,
                theme = photo.houseTheme or "Unknown",
                timestamp = photo.timestamp,
                thumbnailFileID = photo.thumbnailFileID,
                source = "visit",
            }
        end
    end

    -- Index shared photos
    for _, photo in ipairs(sharedPhotos) do
        if not self.db.photos[photo.photoID] then
            self.db.photos[photo.photoID] = {
                owner = photo.ownerName,
                house = photo.houseName,
                theme = photo.houseTheme or "Unknown",
                timestamp = photo.timestamp,
                thumbnailFileID = photo.thumbnailFileID,
                source = "shared",
                sharedBy = photo.sharedBy,
            }
        end
    end

    -- Enforce storage limit
    self:EnforceStorageLimit()

    -- Notify UI to update
    if self.OnLibraryUpdated then
        self:OnLibraryUpdated()
    end
end

function ns:EnforceStorageLimit()
    local photos = self.db.photos
    local count = 0
    local oldest = nil
    local oldestTime = math.huge

    for id, data in pairs(photos) do
        count = count + 1
        if not self.db.favorites[id] and data.timestamp < oldestTime then
            oldestTime = data.timestamp
            oldest = id
        end
    end

    -- Remove oldest non-favorite photos until under limit
    while count > self.db.maxPhotos and oldest do
        photos[oldest] = nil
        count = count - 1
        oldest = nil
        oldestTime = math.huge
        for id, data in pairs(photos) do
            if not self.db.favorites[id] and data.timestamp < oldestTime then
                oldestTime = data.timestamp
                oldest = id
            end
        end
    end
end

-- ---------------------------------------------------------------------------
-- SHARING
-- ---------------------------------------------------------------------------

function ns:ShareLatestPhoto()
    local latest = nil
    local latestTime = 0

    for id, data in pairs(self.db.photos) do
        if data.source == "visit" and data.timestamp > latestTime then
            latestTime = data.timestamp
            latest = id
        end
    end

    if latest then
        self:SharePhotoWithGuild(latest)
    end
end

function ns:SharePhotoWithGuild(photoID)
    if not IsInGuild() then return end

    local photoData = C_HousingPhotoSharing.GetPhotoData(photoID)
    if not photoData then return end

    -- Send via guild channel
    local payload = MSG_SHARE .. ":" .. tostring(photoID)
    C_ChatInfo.SendAddonMessage(MSG_PREFIX, payload, "GUILD")

    -- Share the actual photo data through the housing API
    C_HousingPhotoSharing.SharePhoto(photoData)
end

function ns:OnPhotoReceived(senderName, photoID)
    -- Refresh library to pick up the new shared photo
    self:RefreshPhotoLibrary()
    print("|cff00ccffHomeSnap:|r Received a housing photo from " .. senderName)
end

-- ---------------------------------------------------------------------------
-- ADDON MESSAGES
-- ---------------------------------------------------------------------------

function events:CHAT_MSG_ADDON(prefix, message, channel, sender)
    if prefix ~= MSG_PREFIX then return end
    if sender == UnitName("player") then return end  -- ignore self

    local msgType, data = message:match("^(%a+):(.+)$")
    if msgType == MSG_SHARE then
        -- Photo share notification — actual photo arrives via HOUSING_PHOTO_SHARED
        -- No action needed here; the housing API handles delivery
    elseif msgType == MSG_REQUEST then
        -- Someone requested our photo library list (future feature)
    end
end

-- ---------------------------------------------------------------------------
-- UTILITY
-- ---------------------------------------------------------------------------

function ns:GetSortedPhotos(sortField)
    sortField = sortField or "timestamp"
    local sorted = {}

    for id, data in pairs(self.db.photos) do
        data.photoID = id
        sorted[#sorted + 1] = data
    end

    table.sort(sorted, function(a, b)
        if sortField == "timestamp" then
            return (a.timestamp or 0) > (b.timestamp or 0)
        elseif sortField == "owner" then
            return (a.owner or "") < (b.owner or "")
        elseif sortField == "theme" then
            return (a.theme or "") < (b.theme or "")
        end
        return false
    end)

    return sorted
end

function ns:ToggleFavorite(photoID)
    if self.db.favorites[photoID] then
        self.db.favorites[photoID] = nil
    else
        self.db.favorites[photoID] = true
    end
end

-- ---------------------------------------------------------------------------
-- REGISTER ALL EVENTS
-- ---------------------------------------------------------------------------

for event in pairs(events) do
    eventFrame:RegisterEvent(event)
end
```

!!! info "API Note"
    `C_ChatInfo.SendAddonMessage` requires a registered prefix (`C_ChatInfo.RegisterAddonMessagePrefix`). The prefix is limited to 16 characters — `"HomeSnap"` fits comfortably. You must register before sending or receiving.

---

## Step 6: Building the Gallery UI with ScrollBox

Open `UI.lua`. This is where we build the photo gallery using the modern ScrollBox/DataProvider system — the same system Blizzard uses internally for all scrollable lists since Dragonflight.

```lua
-- UI.lua
-- Mode: Enhancement Artist | Enhance don't replace
-- Gallery frame with ScrollBox grid for photo thumbnails.

local addonName, ns = ...

-- ---------------------------------------------------------------------------
-- GALLERY FRAME
-- ---------------------------------------------------------------------------

local GALLERY_WIDTH = 600
local GALLERY_HEIGHT = 450
local THUMB_PADDING = 8

local gallery = CreateFrame("Frame", "HomeSnapGallery", UIParent, "BackdropTemplate")
gallery:SetSize(GALLERY_WIDTH, GALLERY_HEIGHT)
gallery:SetPoint("CENTER")
gallery:SetBackdrop({
    bgFile = "Interface\\Buttons\\WHITE8x8",
    edgeFile = "Interface\\Buttons\\WHITE8x8",
    edgeSize = 1,
})
gallery:SetBackdropColor(0.05, 0.05, 0.08, 0.95)
gallery:SetBackdropBorderColor(0.3, 0.3, 0.35, 1)
gallery:SetMovable(true)
gallery:EnableMouse(true)
gallery:RegisterForDrag("LeftButton")
gallery:SetScript("OnDragStart", gallery.StartMoving)
gallery:SetScript("OnDragStop", gallery.StopMovingOrSizing)
gallery:SetClampedToScreen(true)
gallery:SetFrameStrata("HIGH")
gallery:Hide()

-- Title bar
local titleBar = CreateFrame("Frame", nil, gallery)
titleBar:SetHeight(30)
titleBar:SetPoint("TOPLEFT", gallery, "TOPLEFT", 0, 0)
titleBar:SetPoint("TOPRIGHT", gallery, "TOPRIGHT", 0, 0)

local titleText = titleBar:CreateFontString(nil, "OVERLAY", "GameFontNormalLarge")
titleText:SetPoint("LEFT", titleBar, "LEFT", 12, 0)
titleText:SetText("HomeSnap Gallery")
titleText:SetTextColor(0.9, 0.8, 0.6)

-- Close button
local closeBtn = CreateFrame("Button", nil, gallery, "UIPanelCloseButton")
closeBtn:SetPoint("TOPRIGHT", gallery, "TOPRIGHT", -2, -2)

-- Sort dropdown area
local sortLabel = gallery:CreateFontString(nil, "OVERLAY", "GameFontNormal")
sortLabel:SetPoint("TOPLEFT", titleBar, "BOTTOMLEFT", 12, -8)
sortLabel:SetText("Sort by:")

local sortBtns = {}
local sortOptions = { "timestamp", "owner", "theme" }
local sortLabels = { timestamp = "Date", owner = "Owner", theme = "Theme" }

for i, field in ipairs(sortOptions) do
    local btn = CreateFrame("Button", nil, gallery, "UIPanelButtonTemplate")
    btn:SetSize(60, 22)
    btn:SetText(sortLabels[field])
    btn:SetPoint("LEFT", sortLabel, "RIGHT", 8 + (i - 1) * 68, 0)
    btn:SetScript("OnClick", function()
        ns:UpdateGallery(field)
    end)
    sortBtns[i] = btn
end

-- Photo count
local countText = gallery:CreateFontString(nil, "OVERLAY", "GameFontNormalSmall")
countText:SetPoint("TOPRIGHT", titleBar, "BOTTOMRIGHT", -12, -12)
countText:SetTextColor(0.6, 0.6, 0.6)

-- ---------------------------------------------------------------------------
-- SCROLLBOX SETUP
-- ---------------------------------------------------------------------------

local scrollBox = CreateFrame("Frame", nil, gallery, "WowScrollBoxList")
scrollBox:SetPoint("TOPLEFT", gallery, "TOPLEFT", 10, -70)
scrollBox:SetPoint("BOTTOMRIGHT", gallery, "BOTTOMRIGHT", -30, 10)

local scrollBar = CreateFrame("EventFrame", nil, gallery, "MinimalScrollBar")
scrollBar:SetPoint("TOPLEFT", scrollBox, "TOPRIGHT", 4, 0)
scrollBar:SetPoint("BOTTOMLEFT", scrollBox, "BOTTOMRIGHT", 4, 0)

-- Create a linear view for photo entries
local view = CreateScrollBoxListLinearView()
local ENTRY_HEIGHT = 72

view:SetElementExtent(ENTRY_HEIGHT)

-- Element initializer — called for each visible row
view:SetElementInitializer("Frame", function(frame, data)
    -- Build the row UI on first use
    if not frame.initialized then
        frame:SetSize(scrollBox:GetWidth(), ENTRY_HEIGHT)

        -- Thumbnail
        frame.thumb = frame:CreateTexture(nil, "ARTWORK")
        frame.thumb:SetSize(ns.db.thumbnailSize, ns.db.thumbnailSize)
        frame.thumb:SetPoint("LEFT", frame, "LEFT", 4, 0)
        frame.thumb:SetTexCoord(0.08, 0.92, 0.08, 0.92)

        -- Owner name
        frame.ownerText = frame:CreateFontString(nil, "OVERLAY", "GameFontNormal")
        frame.ownerText:SetPoint("TOPLEFT", frame.thumb, "TOPRIGHT", 10, -4)

        -- House name
        frame.houseText = frame:CreateFontString(nil, "OVERLAY",
            "GameFontHighlightSmall")
        frame.houseText:SetPoint("TOPLEFT", frame.ownerText, "BOTTOMLEFT", 0, -2)

        -- Theme + date
        frame.metaText = frame:CreateFontString(nil, "OVERLAY",
            "GameFontDisableSmall")
        frame.metaText:SetPoint("TOPLEFT", frame.houseText, "BOTTOMLEFT", 0, -2)

        -- Favorite star
        frame.favBtn = CreateFrame("Button", nil, frame)
        frame.favBtn:SetSize(16, 16)
        frame.favBtn:SetPoint("RIGHT", frame, "RIGHT", -40, 0)
        frame.favStar = frame.favBtn:CreateTexture(nil, "ARTWORK")
        frame.favStar:SetAllPoints()

        -- Share button
        frame.shareBtn = CreateFrame("Button", nil, frame, "UIPanelButtonTemplate")
        frame.shareBtn:SetSize(50, 20)
        frame.shareBtn:SetPoint("RIGHT", frame, "RIGHT", -4, 0)
        frame.shareBtn:SetText("Share")

        -- Divider line
        frame.divider = frame:CreateTexture(nil, "BACKGROUND")
        frame.divider:SetHeight(1)
        frame.divider:SetPoint("BOTTOMLEFT", frame, "BOTTOMLEFT", 4, 0)
        frame.divider:SetPoint("BOTTOMRIGHT", frame, "BOTTOMRIGHT", -4, 0)
        frame.divider:SetColorTexture(0.2, 0.2, 0.25, 1)

        frame.initialized = true
    end

    -- Populate with data
    if data.thumbnailFileID then
        frame.thumb:SetTexture(data.thumbnailFileID)
    else
        frame.thumb:SetTexture("Interface\\Icons\\INV_Misc_Camera_02")
    end

    frame.ownerText:SetText(data.owner or "Unknown")
    frame.houseText:SetText(data.house or "Unnamed House")

    local dateStr = data.timestamp and date("%b %d, %Y", data.timestamp) or "Unknown"
    frame.metaText:SetText((data.theme or "Unknown") .. "  |  " .. dateStr)

    -- Source indicator
    if data.source == "shared" then
        frame.ownerText:SetTextColor(0.4, 0.8, 1.0)
    else
        frame.ownerText:SetTextColor(1.0, 0.82, 0.0)
    end

    -- Favorite state
    local isFav = ns.db.favorites[data.photoID]
    frame.favStar:SetTexture(isFav
        and "Interface\\COMMON\\FavoritesIcon"
        or "Interface\\COMMON\\FavoritesIcon-Disabled")

    frame.favBtn:SetScript("OnClick", function()
        ns:ToggleFavorite(data.photoID)
        local nowFav = ns.db.favorites[data.photoID]
        frame.favStar:SetTexture(nowFav
            and "Interface\\COMMON\\FavoritesIcon"
            or "Interface\\COMMON\\FavoritesIcon-Disabled")
    end)

    -- Share button
    frame.shareBtn:SetScript("OnClick", function()
        ns:SharePhotoWithGuild(data.photoID)
        print("|cff00ccffHomeSnap:|r Photo shared with guild!")
    end)
end)

-- Wire up ScrollBox + ScrollBar + View
ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, view)

-- ---------------------------------------------------------------------------
-- DATA PROVIDER
-- ---------------------------------------------------------------------------

local dataProvider = CreateDataProvider()

function ns:UpdateGallery(sortField)
    dataProvider:Flush()

    local photos = self:GetSortedPhotos(sortField)
    for _, photo in ipairs(photos) do
        dataProvider:Insert(photo)
    end

    scrollBox:SetDataProvider(dataProvider)
    countText:SetText(#photos .. " photos")
end

-- ---------------------------------------------------------------------------
-- LIBRARY UPDATE CALLBACK
-- ---------------------------------------------------------------------------

function ns:OnLibraryUpdated()
    if gallery:IsShown() then
        self:UpdateGallery()
    end
end

-- ---------------------------------------------------------------------------
-- TOGGLE
-- ---------------------------------------------------------------------------

function ns:ToggleGallery()
    if gallery:IsShown() then
        gallery:Hide()
    else
        self:UpdateGallery()
        gallery:Show()
    end
end

-- Slash command
SLASH_HOMESNAP1 = "/homesnap"
SLASH_HOMESNAP2 = "/hs"
SlashCmdList["HOMESNAP"] = function(msg)
    if msg == "share" and ns.isInHouse then
        ns:ShareLatestPhoto()
    else
        ns:ToggleGallery()
    end
end
```

!!! info "API Note"
    `CreateScrollBoxListLinearView` and `ScrollUtil.InitScrollBoxListWithScrollBar` are the modern scroll system introduced in 10.0 (Dragonflight). They replace the old `FauxScrollFrame` pattern entirely. The `DataProvider` handles data insertion, removal, and sorting — you never manage scroll offsets manually.

!!! tip "ScrollBox best practices"
    - Set `SetElementExtent` to a fixed row height for performance (avoids per-element measurement)
    - The element initializer runs for each *visible* row, not each data entry — rows are recycled automatically
    - Use `dataProvider:Flush()` before `Insert()` loops to rebuild the list cleanly
    - Call `scrollBox:SetDataProvider(dp)` after population to trigger a full layout pass

---

## Step 7: Sharing System

HomeSnap uses two layers for photo sharing:

1. **`C_HousingPhotoSharing.SharePhoto()`** — delivers the actual photo data through Blizzard's housing system
2. **`C_ChatInfo.SendAddonMessage()`** — sends a lightweight notification to guildmates so they know a photo was shared

The Core.lua code above already implements both. Here's how the sharing flow works:

### Sending a Photo

```
Player clicks "Share" → SharePhotoWithGuild(photoID)
  1. C_HousingPhotoSharing.GetPhotoData(photoID)  -- fetch full photo
  2. C_HousingPhotoSharing.SharePhoto(photoData)   -- deliver via housing API
  3. C_ChatInfo.SendAddonMessage("HomeSnap", "SHARE:123", "GUILD")  -- notify
```

### Receiving a Photo

```
Guild member shares a photo →
  1. HOUSING_PHOTO_SHARED event fires with senderName, photoID
  2. events:HOUSING_PHOTO_SHARED() calls ns:OnPhotoReceived()
  3. RefreshPhotoLibrary() picks up the new photo from GetSharedPhotos()
  4. UI updates if gallery is open
```

### Channel Restrictions

```lua
-- Available channels for SendAddonMessage:
-- "GUILD"     — guild members (most common for HomeSnap)
-- "PARTY"     — party members
-- "RAID"      — raid members
-- "WHISPER"   — specific player (requires target name)
```

!!! danger "Encounter Restrictions"
    `C_ChatInfo.SendAddonMessage()` is **blocked during active encounters** — Mythic+, PvP, and boss fights. This is a Midnight 12.0 restriction. Since housing and encounters are mutually exclusive (you can't be in a house during a boss fight), this doesn't affect HomeSnap in practice. But if you extend the addon to share photos from non-housing contexts, be aware of this limitation.

!!! warning "Message Size"
    Addon messages are limited to 255 bytes per message. Our notification payload (`"SHARE:123"`) is well under this limit. If you need to send larger payloads (like photo metadata), split them across multiple messages using a chunking protocol. Libraries like LibSerialize and LibDeflate can help compress data.

---

## Step 8: Enhancement Artist Touches

This is where the Enhancement Artist philosophy shines. Instead of creating a standalone button to open our gallery, we'll hook into Blizzard's housing UI and add a "Gallery" tab directly.

Add this block to the bottom of `UI.lua`:

```lua
-- ---------------------------------------------------------------------------
-- ENHANCEMENT: Hook into Blizzard's Housing UI
-- ---------------------------------------------------------------------------

-- Wait for the housing UI to load, then hook it
local isProcessing = false

local function HookHousingUI()
    -- The housing frame may not exist yet — Blizzard loads it on demand
    local housingFrame = HousingFrame
    if not housingFrame then return end

    -- Add a Gallery button to the housing frame's tab bar
    local galleryTab = CreateFrame("Button", nil, housingFrame,
        "UIPanelButtonTemplate")
    galleryTab:SetSize(80, 24)
    galleryTab:SetText("Gallery")
    galleryTab:SetPoint("TOPRIGHT", housingFrame, "TOPRIGHT", -40, -4)
    galleryTab:SetScript("OnClick", function()
        ns:ToggleGallery()
    end)

    -- Hook the housing frame's Show to refresh our photo count
    hooksecurefunc(housingFrame, "Show", function(self)
        if isProcessing then return end
        if self.IsForbidden and self:IsForbidden() then return end
        isProcessing = true

        -- Refresh library when housing UI opens
        ns:RefreshPhotoLibrary()

        isProcessing = false
    end)

    -- Hook the housing frame's Hide to close our gallery too
    housingFrame:HookScript("OnHide", function()
        if gallery:IsShown() then
            gallery:Hide()
        end
    end)
end

-- The housing UI loads on demand — watch for it
EventUtil.ContinueOnAddOnLoaded("Blizzard_HousingUI", function()
    HookHousingUI()
end)
```

### Why These Patterns Matter

Every line follows Enhancement Artist rules:

| Pattern | Why |
|---|---|
| `hooksecurefunc()` instead of `SetScript()` | Post-hook only — never override Blizzard's own handler |
| `IsForbidden()` check | Blizzard frames can become forbidden in certain contexts |
| Recursion guard (`isProcessing`) | Our hook calls `RefreshPhotoLibrary` which could trigger UI updates |
| `HookScript("OnHide")` | Adds our handler without removing Blizzard's existing OnHide |
| `EventUtil.ContinueOnAddOnLoaded` | Waits for Blizzard's housing addon to load before hooking |
| `CreateFrame` with `UIPanelButtonTemplate` | Uses Blizzard's own button template for visual consistency |

!!! tip "The Enhancement Artist test"
    Ask yourself: *"If I disable my addon, does Blizzard's housing UI still work perfectly?"* If yes, you're enhancing. If no, you're replacing. HomeSnap passes this test — removing it leaves the housing UI completely untouched.

---

## Step 9: Settings and Addon Compartment

Open `Config.lua`. This file creates the Settings panel and handles the Addon Compartment (minimap button).

```lua
-- Config.lua
-- Mode: Enhancement Artist | Enhance don't replace
-- Settings panel and Addon Compartment integration.

local addonName, ns = ...

-- ---------------------------------------------------------------------------
-- SETTINGS PANEL
-- ---------------------------------------------------------------------------

EventUtil.ContinueOnAddOnLoaded(addonName, function()
    local category, layout = Settings.RegisterVerticalLayoutCategory("HomeSnap")
    ns.settingsCategoryID = category:GetID()

    -- Storage limit slider
    local maxPhotosSetting = Settings.RegisterAddOnSetting(
        category, "maxPhotos", "maxPhotos",
        ns.db, type(1), "Maximum Photos", ns.defaults.maxPhotos
    )
    local maxPhotosOptions = Settings.CreateSliderOptions(50, 500, 10)
    Settings.CreateSlider(category, maxPhotosSetting, maxPhotosOptions,
        "Maximum number of photos to keep in your library. "
        .. "Oldest non-favorite photos are removed first.")

    -- Thumbnail size slider
    local thumbSetting = Settings.RegisterAddOnSetting(
        category, "thumbnailSize", "thumbnailSize",
        ns.db, type(1), "Thumbnail Size", ns.defaults.thumbnailSize
    )
    local thumbOptions = Settings.CreateSliderOptions(32, 128, 16)
    Settings.CreateSlider(category, thumbSetting, thumbOptions,
        "Size of photo thumbnails in the gallery view.")

    -- Auto-share toggle
    local autoShareSetting = Settings.RegisterAddOnSetting(
        category, "autoShare", "autoShare",
        ns.db, type(true), "Auto-Share with Guild", ns.defaults.autoShare
    )
    Settings.CreateCheckbox(category, autoShareSetting,
        "Automatically share new housing photos with your guild when you visit a house.")

    Settings.RegisterAddOnCategory(category)
end)

-- ---------------------------------------------------------------------------
-- ADDON COMPARTMENT (Minimap Button)
-- ---------------------------------------------------------------------------

-- These functions are referenced by name in the TOC file's
-- AddonCompartmentFunc / AddonCompartmentFuncOnEnter fields.

function HomeSnap_OnAddonCompartmentClick(_, button)
    if button == "LeftButton" then
        ns:ToggleGallery()
    elseif button == "RightButton" then
        Settings.OpenToCategory(ns.settingsCategoryID)
    end
end

function HomeSnap_OnAddonCompartmentEnter(_, menuButtonFrame)
    GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
    GameTooltip:SetText("HomeSnap", 1, 0.82, 0)
    GameTooltip:AddLine("Housing Photo Gallery", 1, 1, 1)
    GameTooltip:AddLine(" ")
    GameTooltip:AddLine("|cff00ff00Left-click:|r Open gallery", 0.8, 0.8, 0.8)
    GameTooltip:AddLine("|cff00ff00Right-click:|r Settings", 0.8, 0.8, 0.8)

    local photoCount = 0
    for _ in pairs(ns.db.photos) do
        photoCount = photoCount + 1
    end
    GameTooltip:AddLine(" ")
    GameTooltip:AddLine(photoCount .. " photos in library", 0.6, 0.6, 0.6)
    GameTooltip:Show()
end
```

!!! info "API Note"
    The Addon Compartment is Blizzard's official replacement for LibDBIcon minimap buttons. Declare the handler function names in your TOC file, and Blizzard adds your addon to the compartment dropdown automatically. No frame creation, no position saving, no minimap dragging code.

!!! tip "Settings API signatures"
    The Settings API changed in 11.0.2. The correct signature is:
    `Settings.RegisterAddOnSetting(category, variable, variableKey, variableTbl, variableType, name, defaultValue)`.
    If you find outdated examples online, run `/wow-verify Settings.RegisterAddOnSetting signature has 7 parameters` to confirm.

---

## Step 10: Complete Code

Here is the full, copy-pasteable code for all three files. Create a folder called `HomeSnap` in your `Interface/AddOns/` directory and save each file.

### File Structure

```
Interface/AddOns/HomeSnap/
├── HomeSnap.toc
├── Core.lua
├── UI.lua
└── Config.lua
```

=== "HomeSnap.toc"

    ```toc
    ## Interface: 120001
    ## Title: HomeSnap
    ## Notes: Housing photo gallery — browse, share, and organize screenshots of player houses.
    ## Author: YourName
    ## Version: 1.0.0
    ## Category: Housing
    ## IconTexture: Interface\Icons\INV_Misc_Camera_02
    ## SavedVariables: HomeSnapDB
    ## AddonCompartmentFunc: HomeSnap_OnAddonCompartmentClick
    ## AddonCompartmentFuncOnEnter: HomeSnap_OnAddonCompartmentEnter
    ## AddonCompartmentFuncOnLeave: GameTooltip_Hide

    Core.lua
    UI.lua
    Config.lua
    ```

=== "Core.lua"

    ```lua
    -- Core.lua
    -- Mode: Enhancement Artist | Enhance don't replace
    local addonName, ns = ...

    ns.defaults = {
        photos = {},
        favorites = {},
        maxPhotos = 200,
        autoShare = false,
        thumbnailSize = 64,
    }

    local MSG_PREFIX = "HomeSnap"

    local events = {}
    local eventFrame = CreateFrame("Frame")
    eventFrame:SetScript("OnEvent", function(self, event, ...)
        local handler = events[event]
        if handler then handler(self, ...) end
    end)

    function events:ADDON_LOADED(loadedAddon)
        if loadedAddon ~= addonName then return end
        HomeSnapDB = HomeSnapDB or CopyTable(ns.defaults)
        ns.db = HomeSnapDB
        for k, v in pairs(ns.defaults) do
            if ns.db[k] == nil then ns.db[k] = v end
        end
        C_ChatInfo.RegisterAddonMessagePrefix(MSG_PREFIX)
        self:UnregisterEvent("ADDON_LOADED")
    end

    function events:HOUSING_ENTERED()
        ns.isInHouse = true
        local info = C_Housing.GetCurrentHouseInfo()
        if info then
            ns.currentHouse = {
                ownerName = info.ownerName,
                houseName = info.houseName,
                houseTheme = info.houseTheme,
                ownerGUID = info.ownerGUID,
            }
        end
    end

    function events:HOUSING_LEFT()
        ns.isInHouse = false
        ns.currentHouse = nil
    end

    function events:HOUSING_PHOTOS_UPDATED()
        ns:RefreshPhotoLibrary()
        if ns.db.autoShare and ns.isInHouse then
            ns:ShareLatestPhoto()
        end
    end

    function events:HOUSING_PHOTO_SHARED(senderName, photoID)
        ns:OnPhotoReceived(senderName, photoID)
    end

    function events:CHAT_MSG_ADDON(prefix, message, channel, sender)
        if prefix ~= MSG_PREFIX then return end
        if sender == UnitName("player") then return end
    end

    function ns:RefreshPhotoLibrary()
        local visitPhotos = C_HousingPhotoSharing.GetVisitPhotos() or {}
        local sharedPhotos = C_HousingPhotoSharing.GetSharedPhotos() or {}

        for _, photo in ipairs(visitPhotos) do
            if not self.db.photos[photo.photoID] then
                self.db.photos[photo.photoID] = {
                    owner = photo.ownerName,
                    house = photo.houseName,
                    theme = photo.houseTheme or "Unknown",
                    timestamp = photo.timestamp,
                    thumbnailFileID = photo.thumbnailFileID,
                    source = "visit",
                }
            end
        end

        for _, photo in ipairs(sharedPhotos) do
            if not self.db.photos[photo.photoID] then
                self.db.photos[photo.photoID] = {
                    owner = photo.ownerName,
                    house = photo.houseName,
                    theme = photo.houseTheme or "Unknown",
                    timestamp = photo.timestamp,
                    thumbnailFileID = photo.thumbnailFileID,
                    source = "shared",
                    sharedBy = photo.sharedBy,
                }
            end
        end

        self:EnforceStorageLimit()
        if self.OnLibraryUpdated then self:OnLibraryUpdated() end
    end

    function ns:EnforceStorageLimit()
        local photos = self.db.photos
        local count = 0
        local oldest, oldestTime = nil, math.huge

        for id, data in pairs(photos) do
            count = count + 1
            if not self.db.favorites[id] and data.timestamp < oldestTime then
                oldestTime = data.timestamp
                oldest = id
            end
        end

        while count > self.db.maxPhotos and oldest do
            photos[oldest] = nil
            count = count - 1
            oldest, oldestTime = nil, math.huge
            for id, data in pairs(photos) do
                if not self.db.favorites[id] and data.timestamp < oldestTime then
                    oldestTime = data.timestamp
                    oldest = id
                end
            end
        end
    end

    function ns:ShareLatestPhoto()
        local latest, latestTime = nil, 0
        for id, data in pairs(self.db.photos) do
            if data.source == "visit" and data.timestamp > latestTime then
                latestTime = data.timestamp
                latest = id
            end
        end
        if latest then self:SharePhotoWithGuild(latest) end
    end

    function ns:SharePhotoWithGuild(photoID)
        if not IsInGuild() then return end
        local photoData = C_HousingPhotoSharing.GetPhotoData(photoID)
        if not photoData then return end
        C_ChatInfo.SendAddonMessage(MSG_PREFIX, "SHARE:" .. tostring(photoID), "GUILD")
        C_HousingPhotoSharing.SharePhoto(photoData)
    end

    function ns:OnPhotoReceived(senderName, photoID)
        self:RefreshPhotoLibrary()
        print("|cff00ccffHomeSnap:|r Received a housing photo from " .. senderName)
    end

    function ns:GetSortedPhotos(sortField)
        sortField = sortField or "timestamp"
        local sorted = {}
        for id, data in pairs(self.db.photos) do
            data.photoID = id
            sorted[#sorted + 1] = data
        end
        table.sort(sorted, function(a, b)
            if sortField == "timestamp" then
                return (a.timestamp or 0) > (b.timestamp or 0)
            elseif sortField == "owner" then
                return (a.owner or "") < (b.owner or "")
            elseif sortField == "theme" then
                return (a.theme or "") < (b.theme or "")
            end
            return false
        end)
        return sorted
    end

    function ns:ToggleFavorite(photoID)
        if self.db.favorites[photoID] then
            self.db.favorites[photoID] = nil
        else
            self.db.favorites[photoID] = true
        end
    end

    for event in pairs(events) do
        eventFrame:RegisterEvent(event)
    end
    ```

=== "UI.lua"

    ```lua
    -- UI.lua
    -- Mode: Enhancement Artist | Enhance don't replace
    local addonName, ns = ...

    local GALLERY_WIDTH = 600
    local GALLERY_HEIGHT = 450

    -- Gallery frame
    local gallery = CreateFrame("Frame", "HomeSnapGallery", UIParent,
        "BackdropTemplate")
    gallery:SetSize(GALLERY_WIDTH, GALLERY_HEIGHT)
    gallery:SetPoint("CENTER")
    gallery:SetBackdrop({
        bgFile = "Interface\\Buttons\\WHITE8x8",
        edgeFile = "Interface\\Buttons\\WHITE8x8",
        edgeSize = 1,
    })
    gallery:SetBackdropColor(0.05, 0.05, 0.08, 0.95)
    gallery:SetBackdropBorderColor(0.3, 0.3, 0.35, 1)
    gallery:SetMovable(true)
    gallery:EnableMouse(true)
    gallery:RegisterForDrag("LeftButton")
    gallery:SetScript("OnDragStart", gallery.StartMoving)
    gallery:SetScript("OnDragStop", gallery.StopMovingOrSizing)
    gallery:SetClampedToScreen(true)
    gallery:SetFrameStrata("HIGH")
    gallery:Hide()

    -- Title
    local titleBar = CreateFrame("Frame", nil, gallery)
    titleBar:SetHeight(30)
    titleBar:SetPoint("TOPLEFT")
    titleBar:SetPoint("TOPRIGHT")

    local titleText = titleBar:CreateFontString(nil, "OVERLAY",
        "GameFontNormalLarge")
    titleText:SetPoint("LEFT", titleBar, "LEFT", 12, 0)
    titleText:SetText("HomeSnap Gallery")
    titleText:SetTextColor(0.9, 0.8, 0.6)

    -- Close button
    local closeBtn = CreateFrame("Button", nil, gallery, "UIPanelCloseButton")
    closeBtn:SetPoint("TOPRIGHT", -2, -2)

    -- Sort buttons
    local sortLabel = gallery:CreateFontString(nil, "OVERLAY", "GameFontNormal")
    sortLabel:SetPoint("TOPLEFT", titleBar, "BOTTOMLEFT", 12, -8)
    sortLabel:SetText("Sort by:")

    local sortOptions = { "timestamp", "owner", "theme" }
    local sortLabels = { timestamp = "Date", owner = "Owner", theme = "Theme" }

    for i, field in ipairs(sortOptions) do
        local btn = CreateFrame("Button", nil, gallery, "UIPanelButtonTemplate")
        btn:SetSize(60, 22)
        btn:SetText(sortLabels[field])
        btn:SetPoint("LEFT", sortLabel, "RIGHT", 8 + (i - 1) * 68, 0)
        btn:SetScript("OnClick", function()
            ns:UpdateGallery(field)
        end)
    end

    -- Photo count
    local countText = gallery:CreateFontString(nil, "OVERLAY",
        "GameFontNormalSmall")
    countText:SetPoint("TOPRIGHT", titleBar, "BOTTOMRIGHT", -12, -12)
    countText:SetTextColor(0.6, 0.6, 0.6)

    -- ScrollBox
    local scrollBox = CreateFrame("Frame", nil, gallery, "WowScrollBoxList")
    scrollBox:SetPoint("TOPLEFT", gallery, "TOPLEFT", 10, -70)
    scrollBox:SetPoint("BOTTOMRIGHT", gallery, "BOTTOMRIGHT", -30, 10)

    local scrollBar = CreateFrame("EventFrame", nil, gallery, "MinimalScrollBar")
    scrollBar:SetPoint("TOPLEFT", scrollBox, "TOPRIGHT", 4, 0)
    scrollBar:SetPoint("BOTTOMLEFT", scrollBox, "BOTTOMRIGHT", 4, 0)

    local view = CreateScrollBoxListLinearView()
    local ENTRY_HEIGHT = 72
    view:SetElementExtent(ENTRY_HEIGHT)

    view:SetElementInitializer("Frame", function(frame, data)
        if not frame.initialized then
            frame:SetSize(scrollBox:GetWidth(), ENTRY_HEIGHT)

            frame.thumb = frame:CreateTexture(nil, "ARTWORK")
            frame.thumb:SetSize(ns.db.thumbnailSize, ns.db.thumbnailSize)
            frame.thumb:SetPoint("LEFT", 4, 0)
            frame.thumb:SetTexCoord(0.08, 0.92, 0.08, 0.92)

            frame.ownerText = frame:CreateFontString(nil, "OVERLAY",
                "GameFontNormal")
            frame.ownerText:SetPoint("TOPLEFT", frame.thumb, "TOPRIGHT", 10, -4)

            frame.houseText = frame:CreateFontString(nil, "OVERLAY",
                "GameFontHighlightSmall")
            frame.houseText:SetPoint("TOPLEFT", frame.ownerText,
                "BOTTOMLEFT", 0, -2)

            frame.metaText = frame:CreateFontString(nil, "OVERLAY",
                "GameFontDisableSmall")
            frame.metaText:SetPoint("TOPLEFT", frame.houseText,
                "BOTTOMLEFT", 0, -2)

            frame.favBtn = CreateFrame("Button", nil, frame)
            frame.favBtn:SetSize(16, 16)
            frame.favBtn:SetPoint("RIGHT", frame, "RIGHT", -40, 0)
            frame.favStar = frame.favBtn:CreateTexture(nil, "ARTWORK")
            frame.favStar:SetAllPoints()

            frame.shareBtn = CreateFrame("Button", nil, frame,
                "UIPanelButtonTemplate")
            frame.shareBtn:SetSize(50, 20)
            frame.shareBtn:SetPoint("RIGHT", -4, 0)
            frame.shareBtn:SetText("Share")

            frame.divider = frame:CreateTexture(nil, "BACKGROUND")
            frame.divider:SetHeight(1)
            frame.divider:SetPoint("BOTTOMLEFT", 4, 0)
            frame.divider:SetPoint("BOTTOMRIGHT", -4, 0)
            frame.divider:SetColorTexture(0.2, 0.2, 0.25, 1)

            frame.initialized = true
        end

        frame.thumb:SetTexture(data.thumbnailFileID
            or "Interface\\Icons\\INV_Misc_Camera_02")
        frame.ownerText:SetText(data.owner or "Unknown")
        frame.houseText:SetText(data.house or "Unnamed House")

        local dateStr = data.timestamp
            and date("%b %d, %Y", data.timestamp) or "Unknown"
        frame.metaText:SetText(
            (data.theme or "Unknown") .. "  |  " .. dateStr)

        if data.source == "shared" then
            frame.ownerText:SetTextColor(0.4, 0.8, 1.0)
        else
            frame.ownerText:SetTextColor(1.0, 0.82, 0.0)
        end

        local isFav = ns.db.favorites[data.photoID]
        frame.favStar:SetTexture(isFav
            and "Interface\\COMMON\\FavoritesIcon"
            or "Interface\\COMMON\\FavoritesIcon-Disabled")

        frame.favBtn:SetScript("OnClick", function()
            ns:ToggleFavorite(data.photoID)
            local nowFav = ns.db.favorites[data.photoID]
            frame.favStar:SetTexture(nowFav
                and "Interface\\COMMON\\FavoritesIcon"
                or "Interface\\COMMON\\FavoritesIcon-Disabled")
        end)

        frame.shareBtn:SetScript("OnClick", function()
            ns:SharePhotoWithGuild(data.photoID)
            print("|cff00ccffHomeSnap:|r Photo shared with guild!")
        end)
    end)

    ScrollUtil.InitScrollBoxListWithScrollBar(scrollBox, scrollBar, view)

    local dataProvider = CreateDataProvider()

    function ns:UpdateGallery(sortField)
        dataProvider:Flush()
        local photos = self:GetSortedPhotos(sortField)
        for _, photo in ipairs(photos) do
            dataProvider:Insert(photo)
        end
        scrollBox:SetDataProvider(dataProvider)
        countText:SetText(#photos .. " photos")
    end

    function ns:OnLibraryUpdated()
        if gallery:IsShown() then
            self:UpdateGallery()
        end
    end

    function ns:ToggleGallery()
        if gallery:IsShown() then
            gallery:Hide()
        else
            self:UpdateGallery()
            gallery:Show()
        end
    end

    -- Slash commands
    SLASH_HOMESNAP1 = "/homesnap"
    SLASH_HOMESNAP2 = "/hs"
    SlashCmdList["HOMESNAP"] = function(msg)
        if msg == "share" and ns.isInHouse then
            ns:ShareLatestPhoto()
        else
            ns:ToggleGallery()
        end
    end

    -- Enhancement: hook into Blizzard's Housing UI
    local isProcessing = false

    EventUtil.ContinueOnAddOnLoaded("Blizzard_HousingUI", function()
        local housingFrame = HousingFrame
        if not housingFrame then return end

        local galleryTab = CreateFrame("Button", nil, housingFrame,
            "UIPanelButtonTemplate")
        galleryTab:SetSize(80, 24)
        galleryTab:SetText("Gallery")
        galleryTab:SetPoint("TOPRIGHT", housingFrame, "TOPRIGHT", -40, -4)
        galleryTab:SetScript("OnClick", function()
            ns:ToggleGallery()
        end)

        hooksecurefunc(housingFrame, "Show", function(self)
            if isProcessing then return end
            if self.IsForbidden and self:IsForbidden() then return end
            isProcessing = true
            ns:RefreshPhotoLibrary()
            isProcessing = false
        end)

        housingFrame:HookScript("OnHide", function()
            if gallery:IsShown() then gallery:Hide() end
        end)
    end)
    ```

=== "Config.lua"

    ```lua
    -- Config.lua
    -- Mode: Enhancement Artist | Enhance don't replace
    local addonName, ns = ...

    EventUtil.ContinueOnAddOnLoaded(addonName, function()
        local category, layout =
            Settings.RegisterVerticalLayoutCategory("HomeSnap")
        ns.settingsCategoryID = category:GetID()

        local maxPhotosSetting = Settings.RegisterAddOnSetting(
            category, "maxPhotos", "maxPhotos",
            ns.db, type(1), "Maximum Photos", ns.defaults.maxPhotos
        )
        local maxPhotosOptions = Settings.CreateSliderOptions(50, 500, 10)
        Settings.CreateSlider(category, maxPhotosSetting, maxPhotosOptions,
            "Maximum number of photos to keep. "
            .. "Oldest non-favorites removed first.")

        local thumbSetting = Settings.RegisterAddOnSetting(
            category, "thumbnailSize", "thumbnailSize",
            ns.db, type(1), "Thumbnail Size", ns.defaults.thumbnailSize
        )
        local thumbOptions = Settings.CreateSliderOptions(32, 128, 16)
        Settings.CreateSlider(category, thumbSetting, thumbOptions,
            "Size of photo thumbnails in the gallery.")

        local autoShareSetting = Settings.RegisterAddOnSetting(
            category, "autoShare", "autoShare",
            ns.db, type(true), "Auto-Share with Guild",
            ns.defaults.autoShare
        )
        Settings.CreateCheckbox(category, autoShareSetting,
            "Automatically share new photos with guild "
            .. "when visiting a house.")

        Settings.RegisterAddOnCategory(category)
    end)

    function HomeSnap_OnAddonCompartmentClick(_, button)
        if button == "LeftButton" then
            ns:ToggleGallery()
        elseif button == "RightButton" then
            Settings.OpenToCategory(ns.settingsCategoryID)
        end
    end

    function HomeSnap_OnAddonCompartmentEnter(_, menuButtonFrame)
        GameTooltip:SetOwner(menuButtonFrame, "ANCHOR_LEFT")
        GameTooltip:SetText("HomeSnap", 1, 0.82, 0)
        GameTooltip:AddLine("Housing Photo Gallery", 1, 1, 1)
        GameTooltip:AddLine(" ")
        GameTooltip:AddLine(
            "|cff00ff00Left-click:|r Open gallery", 0.8, 0.8, 0.8)
        GameTooltip:AddLine(
            "|cff00ff00Right-click:|r Settings", 0.8, 0.8, 0.8)
        local count = 0
        for _ in pairs(ns.db.photos) do count = count + 1 end
        GameTooltip:AddLine(" ")
        GameTooltip:AddLine(count .. " photos in library", 0.6, 0.6, 0.6)
        GameTooltip:Show()
    end
    ```

!!! success "You built it!"
    Copy all four files into `Interface/AddOns/HomeSnap/`, restart WoW (or `/reload` if the folder already existed), and type `/homesnap` to open the gallery. Visit a player house and your photo library will start populating automatically.

### What to Try Next

- **Add a photo preview** — click a thumbnail to show a larger view in a separate frame
- **Add filter buttons** — filter by "My Visits" vs "Shared" photos
- **Add theme icons** — show a theme-specific icon next to each photo entry
- **Try Boundary Pusher mode** — hook the housing photo capture button itself to auto-add photos to your gallery the instant they're taken

!!! warning "New API"
    Before publishing to CurseForge or WoWInterface, run `/wow-verify` on every `C_HousingPhotoSharing` call in your addon. The housing photo API is the newest namespace in WoW — function signatures may evolve in 12.0.2 or later patches. Pin your TOC to `120001` and test on PTR before each patch.
