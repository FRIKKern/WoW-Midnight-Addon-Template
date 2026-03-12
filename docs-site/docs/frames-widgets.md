# Frames & Widgets

The WoW UI is built from a hierarchy of **frames** and **regions**. Every visible element — buttons, health bars, chat windows, tooltips — is a frame or a region attached to a frame. Understanding this system is essential for addon development.

!!! info "Midnight 12.0+ Compatibility"
    This reference covers the widget API as of Patch 12.0 (Midnight). The new "black box" secure UI model means addons can modify visual presentation but cannot replace secure frames. Some legacy methods have been removed or renamed. Always test against the current client.

For the complete upstream reference, see the [Widget API on Warcraft Wiki](https://warcraft.wiki.gg/wiki/Widget_API).

---

## Frame Type Hierarchy

All UI objects inherit from a chain of abstract base types:

```
ScriptObject
  └── ScriptRegion (LayoutFrame)
        ├── LayeredRegion          ← Texture, FontString, Line, MaskTexture
        └── Frame                  ← All interactive frame types
              ├── Button
              │     └── CheckButton
              ├── EditBox
              ├── ScrollFrame
              ├── Slider
              ├── StatusBar
              ├── Cooldown
              ├── GameTooltip
              ├── ModelScene
              ├── ColorSelect
              ├── MessageFrame
              ├── ScrollingMessageFrame
              ├── SimpleHTML
              ├── MovieFrame
              ├── Minimap
              └── Model
                    └── PlayerModel
                          └── DressUpModel
```

### Abstract Base Types

| Abstract Base | Role | Key Methods |
|---|---|---|
| **ScriptObject** | Event handler system | `SetScript()`, `GetScript()`, `HookScript()` |
| **ScriptRegion** (LayoutFrame) | Sizing, anchoring, visibility | `SetSize()`, `SetPoint()`, `Show()`, `Hide()`, `SetAlpha()`, `GetWidth()`, `GetHeight()` |
| **LayeredRegion** | Draw layer support for non-frame regions | `SetDrawLayer()`, `SetVertexColor()`, `SetAlpha()` |
| **Frame** | Full frame: strata, events, children, mouse, backdrop | `RegisterEvent()`, `EnableMouse()`, `SetFrameStrata()`, `CreateTexture()`, `CreateFontString()` |

!!! note
    You never create `ScriptObject`, `ScriptRegion`, or `LayeredRegion` directly. They are abstract bases that provide inherited methods to all descendant types.

---

## Regions

Regions are **visual elements** that live inside a frame's draw layers. They are **not** frames — they cannot receive events, have children, or register for game events.

| Region Type | Purpose | Created With |
|---|---|---|
| **Texture** | Displays an image, atlas, gradient, or solid color | `frame:CreateTexture()` |
| **FontString** | Displays text with font styling | `frame:CreateFontString()` |
| **Line** | Draws a line between two points | `frame:CreateLine()` |
| **MaskTexture** | Alpha mask applied to other textures | `frame:CreateMaskTexture()` |

```lua
-- Creating regions
local tex = frame:CreateTexture(nil, "BACKGROUND")
tex:SetTexture("Interface\\Icons\\INV_Misc_QuestionMark")
tex:SetAllPoints(frame)

local text = frame:CreateFontString(nil, "OVERLAY", "GameFontNormal")
text:SetPoint("CENTER")
text:SetText("Hello World")

local line = frame:CreateLine(nil, "ARTWORK")
line:SetThickness(2)
line:SetStartPoint("TOPLEFT", frame, 0, 0)
line:SetEndPoint("BOTTOMRIGHT", frame, 0, 0)
line:SetColorTexture(1, 0, 0, 1)

local mask = frame:CreateMaskTexture()
mask:SetTexture("Interface\\CHARACTERFRAME\\TempPortalMask", "CLAMPTOBLACKADDITIVE", "CLAMPTOBLACKADDITIVE")
mask:SetAllPoints(tex)
tex:AddMaskTexture(mask)
```

!!! warning
    Regions cannot call `RegisterEvent()`, `SetScript("OnEvent", ...)`, or any frame-specific methods. If you need event handling, use a Frame.

---

## Frame Types

All frame types are created with `CreateFrame("Type", ...)`. Here is the complete list of types available to addons:

| Frame Type | Description | Key Methods |
|---|---|---|
| **Frame** | Base frame. Container, event handler, mouse receiver. | `RegisterEvent()`, `SetScript()`, `EnableMouse()` |
| **Button** | Clickable button with normal/pushed/highlight/disabled textures. | `SetText()`, `SetNormalTexture()`, `Click()`, `Enable()`, `Disable()` |
| **CheckButton** | Toggle button (inherits Button). | `SetChecked()`, `GetChecked()` |
| **EditBox** | Single or multi-line text input field. | `SetText()`, `GetText()`, `SetMaxLetters()`, `SetFocus()`, `ClearFocus()` |
| **ScrollFrame** | Scrollable container with a scroll child. | `SetScrollChild()`, `GetVerticalScroll()`, `SetVerticalScroll()` |
| **ScrollingMessageFrame** | Chat-style scrolling text log. | `AddMessage()`, `SetMaxLines()`, `SetFading()` |
| **SimpleHTML** | Renders a subset of HTML for rich text. | `SetText()` (accepts HTML string) |
| **Slider** | Draggable slider control. | `SetMinMaxValues()`, `SetValue()`, `GetValue()`, `SetValueStep()` |
| **StatusBar** | Progress/health bar. | `SetMinMaxValues()`, `SetValue()`, `SetStatusBarTexture()`, `SetStatusBarColor()` |
| **Cooldown** | Radial cooldown sweep animation. | `SetCooldown(start, duration)`, `SetReverse()` |
| **GameTooltip** | Tooltip frame (usually use the global `GameTooltip`). | `SetOwner()`, `AddLine()`, `AddDoubleLine()`, `Show()` |
| **ColorSelect** | Color picker with hue wheel and value slider. | `SetColorRGB()`, `GetColorRGB()`, `SetColorHSV()` |
| **Model** | Displays a 3D model. | `SetModel()`, `SetPosition()`, `SetFacing()` |
| **PlayerModel** | Model specialized for player/unit display (inherits Model). | `SetUnit()`, `SetAnimation()`, `SetCreature()` |
| **DressUpModel** | Model with equipment preview (inherits PlayerModel). | `TryOn()`, `Undress()` |
| **ModelScene** | Modern 3D model scene (preferred over Model in 12.0+). | `SetFromModelSceneID()`, `GetActorByTag()` |
| **MessageFrame** | Displays scrolling combat/notification text. | `AddMessage()`, `SetInsertMode()` |
| **Minimap** | The minimap (singleton, not typically created by addons). | `SetZoom()`, `GetZoom()` |
| **MovieFrame** | Plays in-game cinematics. | `SetMovie()`, `StartMovie()`, `StopMovie()` |

### CreateFrame Signature

```lua
CreateFrame(frameType, name, parent, template, id)
```

| Parameter | Description |
|---|---|
| `frameType` | String — the widget type (`"Frame"`, `"Button"`, etc.) |
| `name` | Global name (creates `_G["name"]`). Use `nil` for anonymous frames. |
| `parent` | Parent frame. Typically `UIParent`. |
| `template` | Comma-separated template names (e.g., `"BackdropTemplate,SecureActionButtonTemplate"`). |
| `id` | Numeric ID, accessible via `self:GetID()`. |

!!! warning "Avoid Global Names"
    Only use a global name if other addons or Blizzard code needs to find your frame by name. Excess globals pollute the namespace and cause conflicts. Prefer `nil` for internal frames.

### Creation Examples

```lua
-- Simple frame
local frame = CreateFrame("Frame", nil, UIParent)
frame:SetSize(200, 100)
frame:SetPoint("CENTER")

-- Button with template
local btn = CreateFrame("Button", nil, UIParent, "UIPanelButtonTemplate")
btn:SetSize(120, 30)
btn:SetPoint("CENTER")
btn:SetText("Click Me")
btn:SetScript("OnClick", function(self)
    print("Button clicked!")
end)

-- Slider with template
local slider = CreateFrame("Slider", nil, UIParent, "MinimalSliderTemplate")
slider:SetMinMaxValues(0, 100)
slider:SetValue(50)

-- StatusBar from scratch
local bar = CreateFrame("StatusBar", nil, UIParent)
bar:SetSize(200, 20)
bar:SetStatusBarTexture("Interface\\TargetingFrame\\UI-StatusBar")
bar:SetMinMaxValues(0, 100)
bar:SetValue(75)
bar:SetStatusBarColor(0, 1, 0)

-- EditBox for text input
local editBox = CreateFrame("EditBox", nil, UIParent, "InputBoxTemplate")
editBox:SetSize(200, 30)
editBox:SetPoint("CENTER", 0, -50)
editBox:SetAutoFocus(false)
editBox:SetScript("OnEnterPressed", function(self)
    print("Input: " .. self:GetText())
    self:ClearFocus()
end)

-- Cooldown on an icon frame
local icon = CreateFrame("Frame", nil, UIParent)
icon:SetSize(40, 40)
icon:SetPoint("CENTER", 0, 50)
icon.tex = icon:CreateTexture(nil, "ARTWORK")
icon.tex:SetAllPoints()
icon.tex:SetTexture("Interface\\Icons\\Spell_Nature_Heal")
icon.cd = CreateFrame("Cooldown", nil, icon, "CooldownFrameTemplate")
icon.cd:SetAllPoints()
icon.cd:SetCooldown(GetTime(), 10) -- 10-second cooldown starting now
```

---

## Anchor System

Every frame and region is positioned using **anchor points**. An anchor connects one point on your frame to a point on a reference frame.

### The 9 Anchor Points

```
TOPLEFT -------- TOP -------- TOPRIGHT
   |                              |
  LEFT -------- CENTER ------- RIGHT
   |                              |
BOTTOMLEFT --- BOTTOM --- BOTTOMRIGHT
```

### SetPoint

```lua
frame:SetPoint(point, relativeTo, relativePoint, offsetX, offsetY)
```

| Parameter | Description |
|---|---|
| `point` | The point on **this** frame to anchor |
| `relativeTo` | The reference frame (defaults to parent if `nil`) |
| `relativePoint` | The point on the reference frame to attach to (defaults to same as `point`) |
| `offsetX` | Horizontal offset in pixels (positive = right) |
| `offsetY` | Vertical offset in pixels (positive = up) |

```lua
-- Position below parent's bottom-left with 5px gap
frame:SetPoint("TOPLEFT", parent, "BOTTOMLEFT", 0, -5)

-- Center on screen
frame:SetPoint("CENTER", UIParent, "CENTER", 0, 0)
-- Shorthand (defaults fill in):
frame:SetPoint("CENTER")

-- Anchor two points to stretch the frame (creates 10px inset from parent)
frame:SetPoint("TOPLEFT", parent, "TOPLEFT", 10, -10)
frame:SetPoint("BOTTOMRIGHT", parent, "BOTTOMRIGHT", -10, 10)
```

### Other Anchor Methods

```lua
frame:SetAllPoints(relativeTo)   -- Fill the entire reference frame
frame:ClearAllPoints()           -- Remove all anchors (required before re-anchoring)
frame:GetPoint(index)            -- Returns point, relativeTo, relativePoint, x, y
frame:GetNumPoints()             -- Number of active anchors
```

!!! tip
    Always call `ClearAllPoints()` before setting new anchors if the frame was previously positioned. Stale anchors cause layout bugs.

### Anchor Chaining

Frames can be anchored relative to siblings to create dynamic layouts:

```lua
-- Create a row of buttons that automatically flow
local prev = nil
for i = 1, 5 do
    local btn = CreateFrame("Button", nil, parent, "UIPanelButtonTemplate")
    btn:SetSize(80, 25)
    btn:SetText("Tab " .. i)
    if prev then
        btn:SetPoint("LEFT", prev, "RIGHT", 5, 0)
    else
        btn:SetPoint("TOPLEFT", parent, "TOPLEFT", 10, -10)
    end
    prev = btn
end
```

---

## Frame Strata

Frame strata control the **global drawing order** between frames. A frame in a higher stratum always draws on top of frames in lower strata, regardless of frame level.

| Strata (lowest → highest) | Typical Use |
|---|---|
| `WORLD` | World-space UI elements |
| `BACKGROUND` | Behind most UI (world map background) |
| `LOW` | Below standard UI |
| `MEDIUM` | **Default.** Most addon frames. |
| `HIGH` | Above standard UI (unit frames, action bars) |
| `DIALOG` | Dialog boxes, popups |
| `FULLSCREEN` | Full-screen overlays (world map) |
| `FULLSCREEN_DIALOG` | Dialogs on top of fullscreen |
| `TOOLTIP` | Tooltips (always on top) |

```lua
frame:SetFrameStrata("HIGH")
frame:SetFrameLevel(10)  -- Within a strata, higher level = drawn on top

-- Query
local strata = frame:GetFrameStrata()  -- Returns "HIGH"
local level = frame:GetFrameLevel()    -- Returns 10
```

!!! note
    Within the same strata, **frame level** determines draw order. Child frames default to parent's level + 1. Avoid manually setting frame levels unless necessary — let the parent-child relationship handle it.

---

## Draw Layers

Draw layers control the order that **regions** (textures, font strings) are drawn **within a single frame**.

| Layer (lowest → highest) | Typical Use |
|---|---|
| `BACKGROUND` | Background fill, solid colors |
| `BORDER` | Border art, decorative frames around content |
| `ARTWORK` | **Default.** Icons, primary visuals |
| `OVERLAY` | Text labels, badges on top of artwork |
| `HIGHLIGHT` | Mouseover highlight effects (auto-shown on hover) |

Each layer also supports a **sublevel** (-8 to +7) for finer ordering within the same layer:

```lua
local bg = frame:CreateTexture(nil, "BACKGROUND", nil, -1) -- sub-layer -1
local icon = frame:CreateTexture(nil, "ARTWORK")            -- default sub-layer 0
local glow = frame:CreateTexture(nil, "OVERLAY", nil, 3)    -- sub-layer 3
```

!!! tip
    The `HIGHLIGHT` layer is special — textures in this layer automatically show/hide when the frame is moused over (if the frame has mouse enabled). You don't need to write OnEnter/OnLeave scripts for simple highlight effects.

---

## Creating Frames in Lua

### Basic Frame Setup

```lua
local frame = CreateFrame("Frame", "MyAddonMainFrame", UIParent)
frame:SetSize(200, 100)
frame:SetPoint("CENTER")

-- Add a background texture
local bg = frame:CreateTexture(nil, "BACKGROUND")
bg:SetAllPoints(frame)
bg:SetColorTexture(0, 0, 0, 0.8) -- Black, 80% opaque

-- Add a title
local title = frame:CreateFontString(nil, "ARTWORK", "GameFontNormalLarge")
title:SetPoint("TOP", 0, -10)
title:SetText("My Addon")
```

### Event-Driven Frame

```lua
local frame = CreateFrame("Frame")
frame:RegisterEvent("PLAYER_LOGIN")
frame:RegisterEvent("PLAYER_LOGOUT")
frame:RegisterEvent("COMBAT_LOG_EVENT_UNFILTERED")

frame:SetScript("OnEvent", function(self, event, ...)
    if event == "PLAYER_LOGIN" then
        print("Addon loaded!")
    elseif event == "PLAYER_LOGOUT" then
        -- Save data
    elseif event == "COMBAT_LOG_EVENT_UNFILTERED" then
        local _, subEvent = CombatLogGetCurrentEventInfo()
        -- Process combat events
    end
end)
```

### OnUpdate for Per-Frame Logic

```lua
local elapsed_total = 0
frame:SetScript("OnUpdate", function(self, elapsed)
    elapsed_total = elapsed_total + elapsed
    if elapsed_total >= 0.5 then -- Throttle to every 0.5 seconds
        elapsed_total = 0
        -- Do periodic work here
    end
end)
```

!!! warning "OnUpdate Performance"
    `OnUpdate` runs every frame render (potentially 60-240+ times per second). Always throttle expensive operations. Prefer event-driven design over polling.

---

## XML Structure

XML is an alternative (or complement) to Lua for defining UI layouts. Many Blizzard templates are defined in XML.

### Complete XML Example

```xml
<Ui xmlns="http://www.blizzard.com/wow/ui/"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.blizzard.com/wow/ui/ ..\FrameXML\UI.xsd">

  <Frame name="MyAddonFrame" parent="UIParent" frameStrata="MEDIUM"
         enableMouse="true" movable="true" hidden="false">
    <Size x="300" y="200"/>

    <Anchors>
      <Anchor point="CENTER"/>
    </Anchors>

    <Layers>
      <Layer level="BACKGROUND">
        <Texture parentKey="bg" setAllPoints="true">
          <Color r="0" g="0" b="0" a="0.8"/>
        </Texture>
      </Layer>
      <Layer level="BORDER">
        <Texture parentKey="border">
          <Anchors>
            <Anchor point="TOPLEFT" x="-4" y="4"/>
            <Anchor point="BOTTOMRIGHT" x="4" y="-4"/>
          </Anchors>
          <Color r="0.3" g="0.3" b="0.3" a="1"/>
        </Texture>
      </Layer>
      <Layer level="ARTWORK">
        <FontString parentKey="title" inherits="GameFontNormalLarge"
                    text="My Addon">
          <Anchors>
            <Anchor point="TOP" x="0" y="-10"/>
          </Anchors>
        </FontString>
      </Layer>
      <Layer level="HIGHLIGHT">
        <Texture parentKey="highlight" setAllPoints="true" alphaMode="ADD">
          <Color r="1" g="1" b="1" a="0.1"/>
        </Texture>
      </Layer>
    </Layers>

    <Frames>
      <Button parentKey="closeBtn" inherits="UIPanelCloseButton">
        <Anchors>
          <Anchor point="TOPRIGHT" x="-2" y="-2"/>
        </Anchors>
      </Button>
    </Frames>

    <Scripts>
      <OnLoad>
        self:RegisterEvent("PLAYER_LOGIN")
      </OnLoad>
      <OnEvent>
        if event == "PLAYER_LOGIN" then
            print("MyAddon loaded!")
        end
      </OnEvent>
      <OnMouseDown>
        if button == "LeftButton" then
            self:StartMoving()
        end
      </OnMouseDown>
      <OnMouseUp>
        self:StopMovingOrSizing()
      </OnMouseUp>
    </Scripts>
  </Frame>

</Ui>
```

### XML Element Reference

| Element | Description |
|---|---|
| `<Ui>` | Root element. All XML UI files must be wrapped in this. |
| `<Frame>` | Defines a frame (or any frame type via `inherits`). |
| `<Size>` | Sets width (`x`) and height (`y`). |
| `<Anchors>` / `<Anchor>` | Positioning. Attributes: `point`, `relativeTo`, `relativePoint`, `x`, `y`. |
| `<Layers>` / `<Layer>` | Groups regions by draw layer (`level` attribute). |
| `<Texture>` | A texture region. |
| `<FontString>` | A text region. |
| `<Frames>` | Contains child frames. |
| `<Scripts>` | Contains script handlers (`<OnLoad>`, `<OnEvent>`, `<OnClick>`, etc.). |
| `<Attributes>` | Sets frame attributes for secure templates. |

!!! info "XML vs Lua"
    XML is great for declaring static layouts and templates. Lua is better for dynamic UI creation. Most modern addons use Lua exclusively, but understanding XML is important for working with Blizzard's templates and for defining virtual templates.

---

## Virtual Templates

Virtual templates define reusable frame blueprints that are never instantiated directly. Other frames inherit from them via the `template` parameter in `CreateFrame()` or the `inherits` attribute in XML.

### Defining a Template (XML)

```xml
<Frame name="MyAddonRowTemplate" virtual="true" enableMouse="true">
  <Size x="300" y="24"/>

  <Layers>
    <Layer level="BACKGROUND">
      <Texture parentKey="bg" setAllPoints="true">
        <Color r="0.1" g="0.1" b="0.1" a="0.9"/>
      </Texture>
    </Layer>
    <Layer level="ARTWORK">
      <Texture parentKey="icon">
        <Size x="20" y="20"/>
        <Anchors>
          <Anchor point="LEFT" x="4" y="0"/>
        </Anchors>
      </Texture>
      <FontString parentKey="label" inherits="GameFontNormal">
        <Anchors>
          <Anchor point="LEFT" relativeKey="$parent.icon" relativePoint="RIGHT" x="8" y="0"/>
        </Anchors>
      </FontString>
      <FontString parentKey="value" inherits="GameFontHighlight" justifyH="RIGHT">
        <Anchors>
          <Anchor point="RIGHT" x="-8" y="0"/>
        </Anchors>
      </FontString>
    </Layer>
    <Layer level="HIGHLIGHT">
      <Texture parentKey="highlight" setAllPoints="true" alphaMode="ADD">
        <Color r="1" g="1" b="1" a="0.15"/>
      </Texture>
    </Layer>
  </Layers>
</Frame>
```

### Using a Template (Lua)

```lua
-- Create instances from the template
for i = 1, 10 do
    local row = CreateFrame("Frame", nil, scrollContent, "MyAddonRowTemplate")
    row:SetPoint("TOPLEFT", 0, -(i - 1) * 24)
    row.icon:SetTexture("Interface\\Icons\\INV_Misc_Gear_01")
    row.label:SetText("Item " .. i)     -- parentKey gives direct access
    row.value:SetText(math.random(100))
end
```

### Using a Template (XML)

```xml
<Frame name="MySpecificRow" inherits="MyAddonRowTemplate" parent="UIParent">
  <Anchors>
    <Anchor point="CENTER"/>
  </Anchors>
</Frame>
```

### Multiple Template Inheritance

You can inherit from multiple templates by separating with commas:

```lua
local frame = CreateFrame("Frame", nil, UIParent, "BackdropTemplate,SecureHandlerBaseTemplate")
```

Templates are applied left to right. If templates define overlapping keys, the last one wins.

---

## Mixin Pattern

Mixins add methods and behavior to frames by copying functions from a table onto the frame object. This is WoW's primary pattern for object-oriented frame code.

### Defining a Mixin

```lua
MyAddonFrameMixin = {}

function MyAddonFrameMixin:OnLoad()
    self:RegisterEvent("PLAYER_LOGIN")
    self:RegisterEvent("PLAYER_ENTERING_WORLD")
    self.initialized = false
end

function MyAddonFrameMixin:OnEvent(event, ...)
    if event == "PLAYER_LOGIN" then
        self:Setup()
    elseif event == "PLAYER_ENTERING_WORLD" then
        self:Refresh()
    end
end

function MyAddonFrameMixin:Setup()
    self.initialized = true
    self.title:SetText("Ready!")
end

function MyAddonFrameMixin:Refresh()
    if not self.initialized then return end
    -- Update display state
end

function MyAddonFrameMixin:UpdateDisplay(text)
    self.title:SetText(text)
end
```

### Applying a Mixin via XML

```xml
<Frame name="MyAddonFrame" parent="UIParent" mixin="MyAddonFrameMixin">
  <Size x="300" y="200"/>
  <Anchors>
    <Anchor point="CENTER"/>
  </Anchors>
  <Layers>
    <Layer level="ARTWORK">
      <FontString parentKey="title" inherits="GameFontNormalLarge">
        <Anchors>
          <Anchor point="TOP" y="-10"/>
        </Anchors>
      </FontString>
    </Layer>
  </Layers>
  <Scripts>
    <OnLoad method="OnLoad"/>
    <OnEvent method="OnEvent"/>
  </Scripts>
</Frame>
```

### Applying a Mixin via Lua

```lua
local frame = CreateFrame("Frame", nil, UIParent)
Mixin(frame, MyAddonFrameMixin)
frame:OnLoad()  -- Must call manually when applying via Lua
```

### Combining Multiple Mixins

```lua
Mixin(frame, MyBaseMixin, MyEventMixin, MyDisplayMixin)
```

!!! note
    `Mixin(target, ...)` copies all keys from the mixin table(s) onto the target. It does **not** set up metatables — it's a shallow copy. Later mixins overwrite earlier ones if keys collide. Call `Mixin()` before using any mixin methods.

### CreateFromMixins

For creating plain tables (not frames) from mixins:

```lua
-- Create a table pre-mixed with the mixin
local obj = CreateFromMixins(MyAddonFrameMixin)
-- obj now has all methods from MyAddonFrameMixin
```

---

## parentKey Attribute

The `parentKey` attribute in XML automatically assigns a region or child frame as a named field on the parent frame. This replaces the old pattern of using global names to access sub-elements.

### XML Definition

```xml
<Frame name="MyFrame" parent="UIParent">
  <Layers>
    <Layer level="ARTWORK">
      <FontString parentKey="title" inherits="GameFontNormalLarge" text="Hello"/>
      <Texture parentKey="icon"/>
    </Layer>
  </Layers>
  <Frames>
    <Button parentKey="closeBtn" inherits="UIPanelCloseButton">
      <Anchors>
        <Anchor point="TOPRIGHT"/>
      </Anchors>
    </Button>
    <Frame parentKey="content">
      <Anchors>
        <Anchor point="TOPLEFT" x="10" y="-30"/>
        <Anchor point="BOTTOMRIGHT" x="-10" y="10"/>
      </Anchors>
    </Frame>
  </Frames>
</Frame>
```

### Accessing in Lua

```lua
-- parentKey creates direct field references on the parent frame:
MyFrame.title:SetText("Updated Title")
MyFrame.icon:SetTexture("Interface\\Icons\\Spell_Nature_Heal")
MyFrame.closeBtn:SetScript("OnClick", function() MyFrame:Hide() end)
MyFrame.content:SetAlpha(0.5)
```

### Lua Equivalent

```lua
-- parentKey in Lua is simply table field assignment:
local frame = CreateFrame("Frame", "MyFrame", UIParent)
frame.title = frame:CreateFontString(nil, "ARTWORK", "GameFontNormalLarge")
frame.icon = frame:CreateTexture(nil, "ARTWORK")
frame.closeBtn = CreateFrame("Button", nil, frame, "UIPanelCloseButton")
frame.content = CreateFrame("Frame", nil, frame)
```

### Nested parentKey with relativeKey

In XML you can reference sibling parentKeys using `$parent`:

```xml
<FontString parentKey="label">
  <Anchors>
    <Anchor point="LEFT" relativeKey="$parent.icon" relativePoint="RIGHT" x="8"/>
  </Anchors>
</FontString>
```

---

## Common Patterns

### Movable Frame

```lua
local frame = CreateFrame("Frame", "MyMovableFrame", UIParent, "BackdropTemplate")
frame:SetSize(300, 200)
frame:SetPoint("CENTER")
frame:SetMovable(true)
frame:EnableMouse(true)
frame:SetClampedToScreen(true)  -- Prevent dragging off-screen

frame:SetScript("OnMouseDown", function(self, button)
    if button == "LeftButton" then
        self:StartMoving()
    end
end)

frame:SetScript("OnMouseUp", function(self, button)
    self:StopMovingOrSizing()
end)
```

### Movable + Position Saving

```lua
-- Save position to saved variables on drag end
frame:SetScript("OnMouseUp", function(self)
    self:StopMovingOrSizing()
    -- Store position in saved variables
    local point, _, relPoint, x, y = self:GetPoint(1)
    MyAddonDB.position = { point = point, relPoint = relPoint, x = x, y = y }
end)

-- Restore on load
local pos = MyAddonDB and MyAddonDB.position
if pos then
    frame:ClearAllPoints()
    frame:SetPoint(pos.point, UIParent, pos.relPoint, pos.x, pos.y)
end
```

### Frame with Backdrop

```lua
local frame = CreateFrame("Frame", nil, UIParent, "BackdropTemplate")
frame:SetSize(250, 150)
frame:SetPoint("CENTER")
frame:SetBackdrop({
    bgFile = "Interface\\Tooltips\\UI-Tooltip-Background",
    edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
    tile = true,
    tileSize = 16,
    edgeSize = 16,
    insets = { left = 4, right = 4, top = 4, bottom = 4 },
})
frame:SetBackdropColor(0, 0, 0, 0.8)
frame:SetBackdropBorderColor(0.6, 0.6, 0.6, 1)
```

!!! warning "BackdropTemplate Required"
    Since Patch 9.0, you **must** inherit `BackdropTemplate` to use `SetBackdrop()`. Calling it on a plain Frame will error.

### Button with Tooltip

```lua
local btn = CreateFrame("Button", nil, UIParent, "UIPanelButtonTemplate")
btn:SetSize(140, 30)
btn:SetPoint("CENTER", 0, -50)
btn:SetText("Open Settings")

btn:SetScript("OnClick", function(self, button)
    if button == "LeftButton" then
        Settings.OpenToCategory("MyAddon")
    end
end)

btn:SetScript("OnEnter", function(self)
    GameTooltip:SetOwner(self, "ANCHOR_TOP")
    GameTooltip:AddLine("Click to open settings")
    GameTooltip:Show()
end)

btn:SetScript("OnLeave", function(self)
    GameTooltip:Hide()
end)
```

### Scroll Frame with Content

```lua
local scrollFrame = CreateFrame("ScrollFrame", nil, UIParent, "ScrollFrameTemplate")
scrollFrame:SetSize(300, 400)
scrollFrame:SetPoint("CENTER")

local content = CreateFrame("Frame", nil, scrollFrame)
content:SetSize(300, 1000) -- Taller than scroll frame = scrollable
scrollFrame:SetScrollChild(content)

-- Add content to the child frame
for i = 1, 30 do
    local label = content:CreateFontString(nil, "ARTWORK", "GameFontNormal")
    label:SetPoint("TOPLEFT", 10, -20 * i)
    label:SetText("Item " .. i)
end
```

### Closeable, Movable Panel

```lua
local panel = CreateFrame("Frame", "MyAddonPanel", UIParent, "BackdropTemplate")
panel:SetSize(400, 300)
panel:SetPoint("CENTER")
panel:SetMovable(true)
panel:EnableMouse(true)
panel:SetClampedToScreen(true)
panel:SetFrameStrata("DIALOG")

-- Backdrop
panel:SetBackdrop({
    bgFile = "Interface\\Tooltips\\UI-Tooltip-Background",
    edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
    tile = true, tileSize = 16, edgeSize = 16,
    insets = { left = 4, right = 4, top = 4, bottom = 4 },
})
panel:SetBackdropColor(0.05, 0.05, 0.05, 0.95)

-- Title
panel.title = panel:CreateFontString(nil, "ARTWORK", "GameFontNormalLarge")
panel.title:SetPoint("TOPLEFT", 12, -12)
panel.title:SetText("My Addon")

-- Close button
panel.closeBtn = CreateFrame("Button", nil, panel, "UIPanelCloseButton")
panel.closeBtn:SetPoint("TOPRIGHT", -2, -2)

-- Drag to move
panel:SetScript("OnMouseDown", function(self, button)
    if button == "LeftButton" then self:StartMoving() end
end)
panel:SetScript("OnMouseUp", function(self)
    self:StopMovingOrSizing()
end)

-- ESC to close
tinsert(UISpecialFrames, "MyAddonPanel")
```

!!! tip "UISpecialFrames"
    Adding your frame's **global name** to the `UISpecialFrames` table lets players close it by pressing Escape. The frame must have a global name for this to work.

### StatusBar (Health/Progress Bar)

```lua
local bar = CreateFrame("StatusBar", nil, UIParent, "BackdropTemplate")
bar:SetSize(200, 20)
bar:SetPoint("CENTER", 0, 50)
bar:SetMinMaxValues(0, 100)
bar:SetValue(75)
bar:SetStatusBarTexture("Interface\\TargetingFrame\\UI-StatusBar")
bar:SetStatusBarColor(0.2, 0.8, 0.2)

-- Background
bar:SetBackdrop({
    bgFile = "Interface\\Tooltips\\UI-Tooltip-Background",
    insets = { left = 0, right = 0, top = 0, bottom = 0 },
})
bar:SetBackdropColor(0, 0, 0, 0.5)

-- Label
bar.text = bar:CreateFontString(nil, "OVERLAY", "GameFontHighlightSmall")
bar.text:SetPoint("CENTER")
bar.text:SetText("75%")
```

### Resizable Frame

```lua
local frame = CreateFrame("Frame", nil, UIParent, "BackdropTemplate")
frame:SetSize(300, 200)
frame:SetPoint("CENTER")
frame:SetResizable(true)
frame:SetResizeBounds(150, 100, 600, 400) -- min/max width and height

-- Resize grip in bottom-right corner
local grip = CreateFrame("Button", nil, frame)
grip:SetSize(16, 16)
grip:SetPoint("BOTTOMRIGHT")
grip:SetNormalTexture("Interface\\ChatFrame\\UI-ChatIM-SizeGrabber-Up")
grip:SetHighlightTexture("Interface\\ChatFrame\\UI-ChatIM-SizeGrabber-Highlight")
grip:SetPushedTexture("Interface\\ChatFrame\\UI-ChatIM-SizeGrabber-Down")

grip:SetScript("OnMouseDown", function(self)
    self:GetParent():StartSizing("BOTTOMRIGHT")
end)
grip:SetScript("OnMouseUp", function(self)
    self:GetParent():StopMovingOrSizing()
end)
```

---

## Further Reading

- [Widget API — Warcraft Wiki](https://warcraft.wiki.gg/wiki/Widget_API) — Complete API reference for all widget types and methods
- [XML User Interface — Warcraft Wiki](https://warcraft.wiki.gg/wiki/XML_user_interface) — XML schema and element reference
- [Creating a WoW Addon — Warcraft Wiki](https://warcraft.wiki.gg/wiki/Creating_a_WoW_addon) — Getting started guide
- [Handling Events — Warcraft Wiki](https://warcraft.wiki.gg/wiki/Handling_events) — Event system reference
