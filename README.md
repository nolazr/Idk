-- LocalScript en StarterGui - VERSIÓN MOBILE COMPATIBLE
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local Camera = Workspace.CurrentCamera
local player = Players.LocalPlayer

-- ========== DETECTAR MOBILE ==========
local isMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

-- ========== VARIABLES GLOBALES ==========
local currentTab = "AIMLOCK"

-- ========== VARIABLES AIMLOCK ==========
local aimlockActive = false
local aimlockEnabled = false
local aimlockConnection = nil
local selectedTarget = nil
local aimlockKey = Enum.KeyCode.Z
local smoothness = 0.3
local maxDistance = 200
local fovSize = 90
local showFovCircle = true
local aimTarget = "Cabeza"

-- ========== VARIABLES MISC ==========
local targetPlayerMode = "Cualquiera"
local specificTargetPlayer = nil

-- ========== VARIABLES ESP ==========
local espActive = false
local espConnection = nil
local espObjects = {}

-- ========== VARIABLES WHITELIST / BLACKLIST ==========
-- Whitelist: solo estos jugadores serán apuntados/visibles en ESP
-- Blacklist: estos jugadores NUNCA serán apuntados/visibles en ESP
-- Si whitelist tiene nombres, solo esos jugadores son válidos.
-- Si whitelist está vacía, todos son válidos excepto los de blacklist.
local whitelist = {
    -- Agrega nombres aquí, por ejemplo:
    -- "Jugador1",
    -- "Jugador2",
}
local blacklist = {
    -- Agrega nombres aquí, por ejemplo:
    -- "Amigo1",
    -- "Amigo2",
}

local function isPlayerAllowed(p)
    local name = p.Name
    -- Primero revisar blacklist
    for _, bName in ipairs(blacklist) do
        if bName:lower() == name:lower() then
            return false
        end
    end
    -- Si whitelist tiene entradas, solo permitir los que estén en ella
    if #whitelist > 0 then
        for _, wName in ipairs(whitelist) do
            if wName:lower() == name:lower() then
                return true
            end
        end
        return false
    end
    return true
end

-- ========== TAMAÑOS ADAPTATIVOS (viewport real) ==========
-- Diseño base referencia: 360x640 puntos lógicos
-- Una pantalla 2400x1080 física ~= 360x800 lógicos a 3x DPI
local viewport = Camera.ViewportSize
local shortSide = math.min(viewport.X, viewport.Y)
local longSide  = math.max(viewport.X, viewport.Y)

-- Estimación de puntos lógicos: dividimos por densidad promedio
-- 2400x1080 a 440ppi tiene ~3.0x de escala, usamos eso como referencia
local logicalW = shortSide / 3
local logicalH = longSide  / 3

-- Escala relativa al diseño base (360 puntos de ancho)
local SCALE = math.clamp(logicalW / 360, 0.85, 2.6)

-- Tamaño del GUI: 88% del ancho lógico, 74% del alto lógico
local GUI_WIDTH  = math.clamp(math.floor(logicalW * 0.88), 300, 500)
local GUI_HEIGHT = math.clamp(math.floor(logicalH * 0.74), 480, 740)

local BUTTON_HEIGHT   = math.floor(44 * SCALE)
local TEXT_SIZE_SM    = math.floor(12 * SCALE)
local TEXT_SIZE_MD    = math.floor(14 * SCALE)
local TEXT_SIZE_LG    = math.floor(18 * SCALE)
local TEXT_SIZE_TITLE = math.floor(20 * SCALE)
local SLIDER_HEIGHT   = math.floor(18 * SCALE)
local KNOB_SIZE       = math.floor(22 * SCALE)
local TOPBAR_H        = math.floor(54 * SCALE)
local TAB_H           = math.floor(48 * SCALE)
local FLOAT_BTN_SIZE  = math.floor(62 * SCALE)

-- ========== CÍRCULO FOV ==========
local fovCircle = Drawing.new("Circle")
fovCircle.Visible = false
fovCircle.Color = Color3.fromRGB(255, 0, 255)
fovCircle.Thickness = 2
fovCircle.NumSides = 60
fovCircle.Transparency = 1
fovCircle.Radius = 100
fovCircle.Filled = false

local fovText = Drawing.new("Text")
fovText.Visible = false
fovText.Color = Color3.fromRGB(255, 255, 255)
fovText.Size = TEXT_SIZE_SM
fovText.Center = true
fovText.Outline = true
fovText.OutlineColor = Color3.fromRGB(0, 0, 0)
fovText.Font = 2
fovText.Text = "FOV: 90°"

-- ========== CREAR GUI PRINCIPAL (STREAMPROOF) ==========
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "PolloHub"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true

-- Streamproof: oculta el GUI de capturas de pantalla y streams
-- Intenta los distintos métodos según el executor disponible
if syn and syn.protect_gui then
    -- Synapse X
    syn.protect_gui(ScreenGui)
    ScreenGui.Parent = game:GetService("CoreGui")
elseif protect_gui then
    -- KRNL y otros con protect_gui global
    protect_gui(ScreenGui)
    ScreenGui.Parent = game:GetService("CoreGui")
elseif gethui then
    -- Wave, Fluxus, Oxygen U y ejecutores modernos
    ScreenGui.Parent = gethui()
else
    -- Fallback sin streamproof (executor no soportado)
    ScreenGui.Parent = player:WaitForChild("PlayerGui")
end

-- Botón flotante para abrir/cerrar (útil en mobile)
local ToggleButton = Instance.new("TextButton")
ToggleButton.Size = UDim2.new(0, FLOAT_BTN_SIZE, 0, FLOAT_BTN_SIZE)
ToggleButton.Position = UDim2.new(0, 10, 0.5, -FLOAT_BTN_SIZE/2)
ToggleButton.Text = "🐔"
ToggleButton.BackgroundColor3 = Color3.fromRGB(255, 200, 100)
ToggleButton.TextColor3 = Color3.fromRGB(0, 0, 0)
ToggleButton.Font = Enum.Font.GothamBold
ToggleButton.TextSize = math.floor(28 * SCALE)
ToggleButton.ZIndex = 10
ToggleButton.Parent = ScreenGui

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(1, 0)
ToggleCorner.Parent = ToggleButton

-- ========== BOTÓN AIM RÁPIDO (esquina superior derecha) ==========
local AimQuickButton = Instance.new("TextButton")
AimQuickButton.Size = UDim2.new(0, FLOAT_BTN_SIZE, 0, FLOAT_BTN_SIZE)
AimQuickButton.Position = UDim2.new(1, -(FLOAT_BTN_SIZE + 10), 0, 10)
AimQuickButton.Text = "🎯\nOFF"
AimQuickButton.BackgroundColor3 = Color3.fromRGB(80, 80, 100)
AimQuickButton.TextColor3 = Color3.fromRGB(200, 200, 255)
AimQuickButton.Font = Enum.Font.GothamBold
AimQuickButton.TextSize = math.floor(15 * SCALE)
AimQuickButton.ZIndex = 10
AimQuickButton.LineHeight = 1.1
AimQuickButton.Parent = ScreenGui

local AimQuickCorner = Instance.new("UICorner")
AimQuickCorner.CornerRadius = UDim.new(0, 14)
AimQuickCorner.Parent = AimQuickButton

-- Borde de color para destacarlo
local AimQuickStroke = Instance.new("UIStroke")
AimQuickStroke.Color = Color3.fromRGB(255, 100, 100)
AimQuickStroke.Thickness = 2
AimQuickStroke.Parent = AimQuickButton

-- Marco Principal
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, GUI_WIDTH, 0, GUI_HEIGHT)
MainFrame.Position = UDim2.new(0.5, -GUI_WIDTH/2, 0.5, -GUI_HEIGHT/2)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

-- Barra Superior
local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, TOPBAR_H)
TopBar.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
TopBar.BorderSizePixel = 0
TopBar.Active = true
TopBar.Parent = MainFrame

-- Título
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -70, 1, 0)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.Text = "🐔 POLLO HUB  ✥"
Title.BackgroundTransparency = 1
Title.TextColor3 = Color3.fromRGB(255, 200, 100)
Title.Font = Enum.Font.GothamBold
Title.TextSize = TEXT_SIZE_TITLE
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = TopBar

-- Overlay transparente para capturar el drag (cubre TopBar excepto el botón ✕)
local DragHandle = Instance.new("TextButton")
DragHandle.Size = UDim2.new(1, -math.floor(TOPBAR_H * 1.1), 1, 0)
DragHandle.Position = UDim2.new(0, 0, 0, 0)
DragHandle.Text = ""
DragHandle.BackgroundTransparency = 1
DragHandle.ZIndex = 2
DragHandle.Parent = TopBar

-- Botón Cerrar (más grande en mobile)
local CloseButton = Instance.new("TextButton")
CloseButton.Size = UDim2.new(0, math.floor(TOPBAR_H * 0.8), 0, math.floor(TOPBAR_H * 0.8))
CloseButton.Position = UDim2.new(1, -math.floor(TOPBAR_H * 0.9), 0.5, -math.floor(TOPBAR_H * 0.4))
CloseButton.Text = "✕"
CloseButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.Font = Enum.Font.GothamBold
CloseButton.TextSize = TEXT_SIZE_LG
CloseButton.ZIndex = 5
CloseButton.Parent = TopBar

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 6)
CloseCorner.Parent = CloseButton

-- ========== PESTAÑAS ==========
local tabBarY = TOPBAR_H + 5
local TabContainer = Instance.new("Frame")
TabContainer.Size = UDim2.new(1, -20, 0, TAB_H)
TabContainer.Position = UDim2.new(0, 10, 0, tabBarY)
TabContainer.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
TabContainer.BorderSizePixel = 0
TabContainer.Parent = MainFrame

local AimlockTab = Instance.new("TextButton")
AimlockTab.Size = UDim2.new(0.25, -4, 1, -6)
AimlockTab.Position = UDim2.new(0, 3, 0, 3)
AimlockTab.Text = "🎯 AIM"
AimlockTab.BackgroundColor3 = Color3.fromRGB(255, 100, 100)
AimlockTab.TextColor3 = Color3.fromRGB(255, 255, 255)
AimlockTab.Font = Enum.Font.GothamBold
AimlockTab.TextSize = TEXT_SIZE_MD
AimlockTab.Parent = TabContainer

local EspTab = Instance.new("TextButton")
EspTab.Size = UDim2.new(0.25, -4, 1, -6)
EspTab.Position = UDim2.new(0.25, 2, 0, 3)
EspTab.Text = "👁️ ESP"
EspTab.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
EspTab.TextColor3 = Color3.fromRGB(200, 200, 255)
EspTab.Font = Enum.Font.GothamBold
EspTab.TextSize = TEXT_SIZE_MD
EspTab.Parent = TabContainer

local MiscTab = Instance.new("TextButton")
MiscTab.Size = UDim2.new(0.25, -4, 1, -6)
MiscTab.Position = UDim2.new(0.50, 2, 0, 3)
MiscTab.Text = "⚙️ MISC"
MiscTab.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
MiscTab.TextColor3 = Color3.fromRGB(200, 200, 255)
MiscTab.Font = Enum.Font.GothamBold
MiscTab.TextSize = TEXT_SIZE_MD
MiscTab.Parent = TabContainer

local ListsTab = Instance.new("TextButton")
ListsTab.Size = UDim2.new(0.25, -4, 1, -6)
ListsTab.Position = UDim2.new(0.75, 2, 0, 3)
ListsTab.Text = "📋 LISTAS"
ListsTab.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
ListsTab.TextColor3 = Color3.fromRGB(200, 200, 255)
ListsTab.Font = Enum.Font.GothamBold
ListsTab.TextSize = TEXT_SIZE_MD
ListsTab.Parent = TabContainer

-- ========== CONTENEDOR DE CONTENIDO ==========
local contentY = tabBarY + TAB_H + 5
local ContentContainer = Instance.new("Frame")
ContentContainer.Size = UDim2.new(1, -20, 1, -(contentY + 10))
ContentContainer.Position = UDim2.new(0, 10, 0, contentY)
ContentContainer.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
ContentContainer.BorderSizePixel = 0
ContentContainer.ClipsDescendants = true
ContentContainer.Parent = MainFrame

-- ========== PESTAÑA AIMLOCK ==========
local AimlockContent = Instance.new("ScrollingFrame")
AimlockContent.Size = UDim2.new(1, 0, 1, 0)
AimlockContent.BackgroundTransparency = 1
AimlockContent.ScrollBarThickness = isMobile and 6 or 4
AimlockContent.CanvasSize = UDim2.new(0, 0, 0, math.floor(420 * SCALE))
AimlockContent.Visible = true
AimlockContent.Parent = ContentContainer

local innerPad = 10

-- Info
local AimlockInfo = Instance.new("TextLabel")
AimlockInfo.Size = UDim2.new(1, -innerPad*2, 0, math.floor(35 * SCALE))
AimlockInfo.Position = UDim2.new(0, innerPad, 0, innerPad)
AimlockInfo.Text = "🎯 AIMLOCK POLLO"
AimlockInfo.BackgroundTransparency = 1
AimlockInfo.TextColor3 = Color3.fromRGB(255, 200, 100)
AimlockInfo.Font = Enum.Font.GothamBold
AimlockInfo.TextSize = TEXT_SIZE_MD
AimlockInfo.TextXAlignment = Enum.TextXAlignment.Left
AimlockInfo.Parent = AimlockContent

-- BOTÓN TOGGLE AIMLOCK (grande, táctil)
local AimlockToggle = Instance.new("TextButton")
AimlockToggle.Size = UDim2.new(1, -innerPad*2, 0, BUTTON_HEIGHT)
AimlockToggle.Position = UDim2.new(0, innerPad, 0, math.floor(50 * SCALE))
AimlockToggle.Text = "🔴 AIMLOCK: APAGADO"
AimlockToggle.BackgroundColor3 = Color3.fromRGB(80, 80, 100)
AimlockToggle.TextColor3 = Color3.fromRGB(200, 200, 255)
AimlockToggle.Font = Enum.Font.GothamBold
AimlockToggle.TextSize = TEXT_SIZE_MD
AimlockToggle.Parent = AimlockContent

local AimlockCorner = Instance.new("UICorner")
AimlockCorner.CornerRadius = UDim.new(0, 8)
AimlockCorner.Parent = AimlockToggle

-- En mobile mostramos "Activar con botón" en lugar de keybind
local KeybindButton = Instance.new("TextButton")
KeybindButton.Size = UDim2.new(1, -innerPad*2, 0, math.floor(32 * SCALE))
KeybindButton.Position = UDim2.new(0, innerPad, 0, math.floor(50 * SCALE) + BUTTON_HEIGHT + 8)
if isMobile then
    KeybindButton.Text = "📱 Modo: BOTÓN TÁCTIL"
    KeybindButton.Active = false
else
    KeybindButton.Text = "🔑 Tecla: Q (clic para cambiar)"
end
KeybindButton.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
KeybindButton.TextColor3 = Color3.fromRGB(180, 180, 220)
KeybindButton.Font = Enum.Font.Gotham
KeybindButton.TextSize = TEXT_SIZE_SM
KeybindButton.Parent = AimlockContent

local KBCorner = Instance.new("UICorner")
KBCorner.CornerRadius = UDim.new(0, 6)
KBCorner.Parent = KeybindButton

-- Zona de apuntado
local zoneY = math.floor(50 * SCALE) + BUTTON_HEIGHT + math.floor(32 * SCALE) + 24

local ZoneLabel = Instance.new("TextLabel")
ZoneLabel.Size = UDim2.new(1, -innerPad*2, 0, 20)
ZoneLabel.Position = UDim2.new(0, innerPad, 0, zoneY)
ZoneLabel.Text = "🎯 ZONA DE APUNTADO"
ZoneLabel.BackgroundTransparency = 1
ZoneLabel.TextColor3 = Color3.fromRGB(180, 180, 220)
ZoneLabel.Font = Enum.Font.Gotham
ZoneLabel.TextSize = TEXT_SIZE_SM
ZoneLabel.TextXAlignment = Enum.TextXAlignment.Left
ZoneLabel.Parent = AimlockContent

local ZoneSelector = Instance.new("Frame")
ZoneSelector.Size = UDim2.new(1, -innerPad*2, 0, BUTTON_HEIGHT)
ZoneSelector.Position = UDim2.new(0, innerPad, 0, zoneY + 24)
ZoneSelector.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
ZoneSelector.BorderSizePixel = 0
ZoneSelector.Parent = AimlockContent

local HeadButton = Instance.new("TextButton")
HeadButton.Size = UDim2.new(0.5, -2, 1, -6)
HeadButton.Position = UDim2.new(0, 3, 0, 3)
HeadButton.Text = "👤 CABEZA"
HeadButton.BackgroundColor3 = Color3.fromRGB(255, 100, 100)
HeadButton.TextColor3 = Color3.fromRGB(255, 255, 255)
HeadButton.Font = Enum.Font.GothamBold
HeadButton.TextSize = TEXT_SIZE_MD
HeadButton.Parent = ZoneSelector

local TorsoButton = Instance.new("TextButton")
TorsoButton.Size = UDim2.new(0.5, -2, 1, -6)
TorsoButton.Position = UDim2.new(0.5, -1, 0, 3)
TorsoButton.Text = "👕 CUERPO"
TorsoButton.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
TorsoButton.TextColor3 = Color3.fromRGB(200, 200, 255)
TorsoButton.Font = Enum.Font.GothamBold
TorsoButton.TextSize = TEXT_SIZE_MD
TorsoButton.Parent = ZoneSelector

-- Sliders section
local sliderStartY = zoneY + BUTTON_HEIGHT + 36

local function createSlider(parent, labelText, labelColor, fillColor, defaultRatio, yOffset)
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.35, 0, 0, math.floor(28 * SCALE))
    label.Position = UDim2.new(0, innerPad, 0, sliderStartY + yOffset)
    label.Text = labelText
    label.BackgroundTransparency = 1
    label.TextColor3 = labelColor
    label.Font = Enum.Font.GothamBold
    label.TextSize = TEXT_SIZE_SM
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = parent

    local track = Instance.new("Frame")
    track.Size = UDim2.new(0.6, -innerPad, 0, SLIDER_HEIGHT)
    track.Position = UDim2.new(0.38, 0, 0, sliderStartY + yOffset + math.floor(6 * SCALE))
    track.BackgroundColor3 = Color3.fromRGB(50, 50, 70)
    track.BorderSizePixel = 0
    track.Parent = parent

    local trackCorner = Instance.new("UICorner")
    trackCorner.CornerRadius = UDim.new(1, 0)
    trackCorner.Parent = track

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(defaultRatio, 0, 1, 0)
    fill.BackgroundColor3 = fillColor
    fill.BorderSizePixel = 0
    fill.Parent = track

    local fillCorner = Instance.new("UICorner")
    fillCorner.CornerRadius = UDim.new(1, 0)
    fillCorner.Parent = fill

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, KNOB_SIZE, 0, KNOB_SIZE)
    knob.Position = UDim2.new(defaultRatio, -KNOB_SIZE/2, 0.5, -KNOB_SIZE/2)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.BorderSizePixel = 0
    knob.Parent = track

    local knobCorner = Instance.new("UICorner")
    knobCorner.CornerRadius = UDim.new(1, 0)
    knobCorner.Parent = knob

    return label, track, fill, knob
end

local fovSpacing = math.floor(38 * SCALE)
local FOVLabel, FOVSlider, FOVFill, FOVKnob = createSlider(
    AimlockContent,
    "FOV: 90°",
    Color3.fromRGB(255, 180, 100),
    Color3.fromRGB(255, 180, 100),
    fovSize / 180,
    0
)

local SmoothLabel, SmoothSlider, SmoothFill, SmoothKnob = createSlider(
    AimlockContent,
    "Suave: 0.3",
    Color3.fromRGB(255, 100, 100),
    Color3.fromRGB(255, 100, 100),
    smoothness,
    fovSpacing
)

local DistanceLabel, DistanceSlider, DistanceFill, DistanceKnob = createSlider(
    AimlockContent,
    "Dist: 200",
    Color3.fromRGB(100, 200, 255),
    Color3.fromRGB(100, 200, 255),
    maxDistance / 500,
    fovSpacing * 2
)

-- Botón círculo FOV
local fovBtnY = sliderStartY + fovSpacing * 3 + 4
local FovCircleToggle = Instance.new("TextButton")
FovCircleToggle.Size = UDim2.new(1, -innerPad*2, 0, BUTTON_HEIGHT)
FovCircleToggle.Position = UDim2.new(0, innerPad, 0, fovBtnY)
FovCircleToggle.Text = "⭕ CÍRCULO FOV: ON"
FovCircleToggle.BackgroundColor3 = Color3.fromRGB(80, 180, 120)
FovCircleToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
FovCircleToggle.Font = Enum.Font.GothamBold
FovCircleToggle.TextSize = TEXT_SIZE_MD
FovCircleToggle.Parent = AimlockContent

local FovCorner = Instance.new("UICorner")
FovCorner.CornerRadius = UDim.new(0, 8)
FovCorner.Parent = FovCircleToggle

-- Status label
local AimlockStatus = Instance.new("TextLabel")
AimlockStatus.Size = UDim2.new(1, -innerPad*2, 0, 28)
AimlockStatus.Position = UDim2.new(0, innerPad, 0, fovBtnY + BUTTON_HEIGHT + 8)
AimlockStatus.Text = "● AIMLOCK: APAGADO"
AimlockStatus.BackgroundTransparency = 1
AimlockStatus.TextColor3 = Color3.fromRGB(255, 100, 100)
AimlockStatus.Font = Enum.Font.GothamBold
AimlockStatus.TextSize = TEXT_SIZE_MD
AimlockStatus.TextXAlignment = Enum.TextXAlignment.Left
AimlockStatus.Parent = AimlockContent

-- Actualizar canvas size
AimlockContent.CanvasSize = UDim2.new(0, 0, 0, fovBtnY + BUTTON_HEIGHT + 44)

-- ========== PESTAÑA ESP ==========
local EspContent = Instance.new("ScrollingFrame")
EspContent.Size = UDim2.new(1, 0, 1, 0)
EspContent.BackgroundTransparency = 1
EspContent.ScrollBarThickness = isMobile and 6 or 4
EspContent.CanvasSize = UDim2.new(0, 0, 0, math.floor(300 * SCALE))
EspContent.Visible = false
EspContent.Parent = ContentContainer

local EspInfo = Instance.new("TextLabel")
EspInfo.Size = UDim2.new(1, -innerPad*2, 0, 35)
EspInfo.Position = UDim2.new(0, innerPad, 0, innerPad)
EspInfo.Text = "👁️ ESP - Cajas alrededor de jugadores"
EspInfo.BackgroundTransparency = 1
EspInfo.TextColor3 = Color3.fromRGB(255, 200, 100)
EspInfo.Font = Enum.Font.GothamBold
EspInfo.TextSize = TEXT_SIZE_MD
EspInfo.TextXAlignment = Enum.TextXAlignment.Left
EspInfo.Parent = EspContent

local EspSeparator = Instance.new("Frame")
EspSeparator.Size = UDim2.new(1, -innerPad*2, 0, 1)
EspSeparator.Position = UDim2.new(0, innerPad, 0, 48)
EspSeparator.BackgroundColor3 = Color3.fromRGB(60, 60, 80)
EspSeparator.BorderSizePixel = 0
EspSeparator.Parent = EspContent

local EspToggle = Instance.new("TextButton")
EspToggle.Size = UDim2.new(1, -innerPad*2, 0, BUTTON_HEIGHT)
EspToggle.Position = UDim2.new(0, innerPad, 0, 58)
EspToggle.Text = "🔴 ESP: APAGADO"
EspToggle.BackgroundColor3 = Color3.fromRGB(80, 80, 100)
EspToggle.TextColor3 = Color3.fromRGB(200, 200, 255)
EspToggle.Font = Enum.Font.GothamBold
EspToggle.TextSize = TEXT_SIZE_MD
EspToggle.Parent = EspContent

local EspToggleCorner = Instance.new("UICorner")
EspToggleCorner.CornerRadius = UDim.new(0, 8)
EspToggleCorner.Parent = EspToggle

EspContent.CanvasSize = UDim2.new(0, 0, 0, 58 + BUTTON_HEIGHT + 20)

-- ========== PESTAÑA MISC ==========
local MiscContent = Instance.new("ScrollingFrame")
MiscContent.Size = UDim2.new(1, 0, 1, 0)
MiscContent.BackgroundTransparency = 1
MiscContent