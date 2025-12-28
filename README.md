-- Loader + Aimbot (com persistência local para executores)
-- Suporta queue_on_teleport (syn/fluxus/queue_on_teleport) e salva configurações localmente
-- usando writefile/readfile/isfile (método comum a executores como Synapse/Fluxus/etc).
-- Cole/execute este arquivo no seu executor. Ele registra o script para re-execução
-- após teleport e também o executa imediatamente.

-- ========================
-- START LOADER (NAO EDITAR)
-- ========================
local SCRIPT = [[
-- LocalScript: MouseAimbotWithUI_v6_beautified.lua (com persistência para executor)
-- Versão final com rodapé personalizado: caveira, hover e fallback.
-- Coloque este LocalScript no LocalPlayer (PlayerScripts) ou carregue no CoreGui conforme seu fluxo.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ContentProvider = game:GetService("ContentProvider")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local CoreGui = game:GetService("CoreGui")

-- ===== CONFIG DEFAULTS =====
local CONFIG = {
    FOV_RADIUS = 150,
    LOCK_PART = "Head",
    AIM_SENSITIVITY = 1,
    IGNORE_SNIPER_ZOOM = true,
    ENABLED = true,
    FOV_VISIBLE = true,
    SMOOTH_AIM = false,
    AUTO_SHOOT = false,
    LOCK_KEY = Enum.UserInputType.MouseButton2,
    HIGHLIGHT_COLOR = Color3.fromRGB(0, 255, 0), -- cor padrão do highlight
}

-- Asset id da caveira (fácil de trocar aqui)
local SKULL_ASSET_ID = "11722153703"

-- ===== Executor persistence (writefile/readfile) =====
local SETTINGS_FILE = "AimbotSettings.json"
local hasIsFile = (type(isfile) == "function")
local hasReadFile = (type(readfile) == "function")
local hasWriteFile = (type(writefile) == "function")

local function serializeLockKey(k)
    -- Armazena Enum.UserInputType como "UIT:Name" e Enum.KeyCode como "KeyCode:Name"
    if typeof(k) == "EnumItem" then
        if Enum.UserInputType[k.Name] then
            return "UIT:" .. k.Name
        elseif Enum.KeyCode[k.Name] then
            return "KeyCode:" .. k.Name
        else
            return "Enum:" .. k.Name
        end
    end
    return tostring(k)
end

local function deserializeLockKey(s)
    if type(s) ~= "string" then return Enum.UserInputType.MouseButton2 end
    local prefix, name = s:match("^(%a+):(.+)$")
    if prefix == "UIT" and Enum.UserInputType[name] then return Enum.UserInputType[name] end
    if prefix == "KeyCode" and Enum.KeyCode[name] then return Enum.KeyCode[name] end
    -- fallback: try directly by name
    if Enum.UserInputType[s] then return Enum.UserInputType[s] end
    if Enum.KeyCode[s] then return Enum.KeyCode[s] end
    return Enum.UserInputType.MouseButton2
end

local function loadLocalSettings()
    if not hasIsFile or not hasReadFile then
        return
    end
    if not isfile(SETTINGS_FILE) then
        return
    end
    local ok, content = pcall(function() return readfile(SETTINGS_FILE) end)
    if not ok or not content then return end
    local ok2, tbl = pcall(function() return HttpService:JSONDecode(content) end)
    if not ok2 or type(tbl) ~= "table" then return end

    -- Aplicar valores carregados (com checagens)
    if type(tbl.FOV_RADIUS) == "number" then CONFIG.FOV_RADIUS = tbl.FOV_RADIUS end
    if type(tbl.AIM_SENSITIVITY) == "number" then CONFIG.AIM_SENSITIVITY = tbl.AIM_SENSITIVITY end
    if type(tbl.IGNORE_SNIPER_ZOOM) == "boolean" then CONFIG.IGNORE_SNIPER_ZOOM = tbl.IGNORE_SNIPER_ZOOM end
    if type(tbl.ENABLED) == "boolean" then CONFIG.ENABLED = tbl.ENABLED end
    if type(tbl.FOV_VISIBLE) == "boolean" then CONFIG.FOV_VISIBLE = tbl.FOV_VISIBLE end
    if type(tbl.SMOOTH_AIM) == "boolean" then CONFIG.SMOOTH_AIM = tbl.SMOOTH_AIM end
    if type(tbl.AUTO_SHOOT) == "boolean" then CONFIG.AUTO_SHOOT = tbl.AUTO_SHOOT end
    if type(tbl.LOCK_PART) == "string" then CONFIG.LOCK_PART = tbl.LOCK_PART end
    if type(tbl.LOCK_KEY) == "string" then CONFIG.LOCK_KEY = deserializeLockKey(tbl.LOCK_KEY) end
    if type(tbl.HIGHLIGHT_COLOR) == "table" then
        local c = tbl.HIGHLIGHT_COLOR
        CONFIG.HIGHLIGHT_COLOR = Color3.fromRGB(c.r or 0, c.g or 0, c.b or 0)
    end
end

local function saveLocalSettings()
    if not hasWriteFile then return false end
    local out = {
        FOV_RADIUS = CONFIG.FOV_RADIUS,
        AIM_SENSITIVITY = CONFIG.AIM_SENSITIVITY,
        IGNORE_SNIPER_ZOOM = CONFIG.IGNORE_SNIPER_ZOOM,
        ENABLED = CONFIG.ENABLED,
        FOV_VISIBLE = CONFIG.FOV_VISIBLE,
        SMOOTH_AIM = CONFIG.SMOOTH_AIM,
        AUTO_SHOOT = CONFIG.AUTO_SHOOT,
        LOCK_PART = CONFIG.LOCK_PART,
        LOCK_KEY = serializeLockKey(CONFIG.LOCK_KEY),
        HIGHLIGHT_COLOR = {
            r = math.floor(CONFIG.HIGHLIGHT_COLOR.R * 255 + 0.5),
            g = math.floor(CONFIG.HIGHLIGHT_COLOR.G * 255 + 0.5),
            b = math.floor(CONFIG.HIGHLIGHT_COLOR.B * 255 + 0.5),
        },
    }
    local ok, data = pcall(function() return HttpService:JSONEncode(out) end)
    if not ok or not data then return false end
    pcall(function() writefile(SETTINGS_FILE, data) end)
    return true
end

-- Debounce save (agrupar múltiplas mudanças)
local savePending = false
local function scheduleSave()
    if not hasWriteFile then return end
    if savePending then return end
    savePending = true
    delay(0.9, function()
        pcall(saveLocalSettings)
        savePending = false
    end)
end

-- Expor API simples (opcional)
pcall(function() _G.AimbotSettingsExecutor = { GetConfig = function() return CONFIG end, SaveNow = saveLocalSettings, ScheduleSave = scheduleSave } end)

-- Carrega configurações locais (antes de construir UI para refletir valores)
pcall(function() loadLocalSettings() end)

-- ===== Helpers =====
local function tweenObject(obj, props, time, style, dir)
    local info = TweenInfo.new(time or 0.15, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out)
    local t = TweenService:Create(obj, info, props)
    t:Play()
    return t
end

-- ===== ESP/Config =====
local ESPEnabled = false
local ESPContainers = {}

local function updateAllHighlightsColor(color)
    for plr, highlight in pairs(ESPContainers) do
        if highlight and highlight.Parent then
            pcall(function()
                highlight.FillColor = color
                highlight.OutlineColor = color
            end)
        end
    end
end

local function createHighlight(plr)
    if ESPContainers[plr] then
        ESPContainers[plr]:Destroy()
        ESPContainers[plr] = nil
    end
    local character = plr.Character
    if not character then return end
    local highlight = Instance.new("Highlight")
    highlight.Name = "ESPHighlight"
    highlight.FillColor = CONFIG.HIGHLIGHT_COLOR
    highlight.FillTransparency = 0.6
    highlight.OutlineColor = CONFIG.HIGHLIGHT_COLOR
    highlight.OutlineTransparency = 0
    highlight.Adornee = character
    highlight.Parent = workspace
    ESPContainers[plr] = highlight

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if humanoid then
        humanoid.Died:Connect(function()
            if ESPContainers[plr] then
                ESPContainers[plr]:Destroy()
                ESPContainers[plr] = nil
            end
        end)
    end
end

local function setupESPForPlayer(plr)
    plr.CharacterAdded:Connect(function()
        if ESPEnabled then createHighlight(plr) end
    end)
    if ESPEnabled and plr.Character then createHighlight(plr) end
end

Players.PlayerAdded:Connect(setupESPForPlayer)
for _, plr in pairs(Players:GetPlayers()) do
    setupESPForPlayer(plr)
end

local ESPConnection
ESPConnection = RunService.RenderStepped:Connect(function()
    if ESPEnabled then
        for _, plr in pairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild("Humanoid") then
                if not ESPContainers[plr] then createHighlight(plr) end
            end
        end
        for plr, highlight in pairs(ESPContainers) do
            if not plr.Parent or not plr.Character then
                highlight:Destroy()
                ESPContainers[plr] = nil
            end
        end
    else
        for _, highlight in pairs(ESPContainers) do
            highlight:Destroy()
        end
        ESPContainers = {}
    end
end)

-- ===== FOV Drawing =====
local FOVCircle
local function createFov()
    if FOVCircle then pcall(function() FOVCircle:Remove() end) end
    local ok, d = pcall(function() return Drawing.new("Circle") end)
    if ok and d then
        FOVCircle = d
        FOVCircle.Visible = CONFIG.FOV_VISIBLE
        FOVCircle.Radius = CONFIG.FOV_RADIUS
        FOVCircle.Color = Color3.fromRGB(255,0,0)
        FOVCircle.Thickness = 1
        FOVCircle.Filled = false
    end
end
createFov()

-- ===== Input Handling =====
local holding = false
local activeConnections = {}

local function updateLockKey(newKey) CONFIG.LOCK_KEY = newKey scheduleSave() end

local function inputBegan(input)
    if input.UserInputType == CONFIG.LOCK_KEY or input.KeyCode == CONFIG.LOCK_KEY then holding = true end
end
local function inputEnded(input)
    if input.UserInputType == CONFIG.LOCK_KEY or input.KeyCode == CONFIG.LOCK_KEY then holding = false end
end
table.insert(activeConnections, UserInputService.InputBegan:Connect(inputBegan))
table.insert(activeConnections, UserInputService.InputEnded:Connect(inputEnded))

-- ===== Sniper check =====
local function IsSniperZoomed()
    if not CONFIG.IGNORE_SNIPER_ZOOM then return false end
    local char = LocalPlayer.Character
    if char then
        local weapon = char:FindFirstChildWhichIsA("Tool")
        if weapon and string.lower(weapon.Name or ""):find("awp") then
            if Camera.FieldOfView < 70 then return true end
        end
    end
    return false
end

-- ===== Targeting =====
local function GetClosestPlayer()
    local closest, shortest = nil, math.huge
    local mousePos = UserInputService:GetMouseLocation()
    for _, plr in pairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChild(CONFIG.LOCK_PART) then
            local part = plr.Character[CONFIG.LOCK_PART]
            local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
            if onScreen then
                local dist = (Vector2.new(screenPos.X,screenPos.Y)-Vector2.new(mousePos.X,mousePos.Y)).Magnitude
                if dist < shortest and dist <= CONFIG.FOV_RADIUS then
                    shortest = dist
                    closest = plr
                end
            end
        end
    end
    return closest
end

-- ===== Aimbot Loop =====
local aimbotConnection
local currentTarget = nil
local function startAimbotLoop()
    if aimbotConnection then aimbotConnection:Disconnect() end
    aimbotConnection = RunService.RenderStepped:Connect(function()
        local mousePos = UserInputService:GetMouseLocation()
        if FOVCircle then
            FOVCircle.Position = mousePos
            FOVCircle.Radius = CONFIG.FOV_RADIUS
            FOVCircle.Visible = CONFIG.FOV_VISIBLE
        end
        if CONFIG.ENABLED and holding and not IsSniperZoomed() then
            if currentTarget then
                local part = currentTarget.Character and currentTarget.Character:FindFirstChild(CONFIG.LOCK_PART)
                if not part or not currentTarget.Parent then currentTarget = nil end
            end
            if not currentTarget then currentTarget = GetClosestPlayer() end
            if currentTarget then
                local part = currentTarget.Character and currentTarget.Character:FindFirstChild(CONFIG.LOCK_PART)
                if part then
                    local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
                    if onScreen then
                        local dx, dy = (screenPos.X - mousePos.X) * CONFIG.AIM_SENSITIVITY, (screenPos.Y - mousePos.Y) * CONFIG.AIM_SENSITIVITY
                        if CONFIG.SMOOTH_AIM then dx, dy = dx*0.5, dy*0.5 end
                        pcall(function() mousemoverel(dx, dy) end)
                        if CONFIG.AUTO_SHOOT then pcall(function() mouse1click() end) end
                    end
                end
            end
        else
            currentTarget = nil
        end
    end)
end
startAimbotLoop()

-- ===== GUI =====
pcall(function()
    local old = LocalPlayer:FindFirstChild("PlayerGui") and LocalPlayer.PlayerGui:FindFirstChild("Aimbot_UI")
    if old then old:Destroy() end
    local oldc = CoreGui:FindFirstChild("Aimbot_UI")
    if oldc then oldc:Destroy() end
end)

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "Aimbot_UI"
screenGui.ResetOnSpawn = false
pcall(function() screenGui.Parent = CoreGui end)
if not screenGui.Parent then screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui") end

local THEME = {
    Background = Color3.fromRGB(30,30,30),
    Panel = Color3.fromRGB(20,20,20),
    Accent = Color3.fromRGB(255,80,80),
    Text = Color3.fromRGB(230,230,230),
    SubText = Color3.fromRGB(170,170,170),
}

-- Main frame: use anchor (0,0) and center initially with ViewportSize
local mainFrame = Instance.new("Frame", screenGui)
mainFrame.Size = UDim2.new(0,520,0,300)

-- center initially using viewport (anchor 0,0)
local vp = Camera and Camera.ViewportSize or Vector2.new(1920,1080)
local initX = math.floor((vp.X - 520) / 2)
local initY = math.floor((vp.Y - 300) / 2)
mainFrame.Position = UDim2.new(0, initX, 0, initY)

mainFrame.AnchorPoint = Vector2.new(0,0)
mainFrame.BackgroundColor3 = THEME.Panel
mainFrame.BackgroundTransparency = 0.06
mainFrame.ClipsDescendants = true
local mainCorner = Instance.new("UICorner", mainFrame)
mainCorner.CornerRadius = UDim.new(0,14)

local mainGradient = Instance.new("UIGradient", mainFrame)
mainGradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(30,30,30)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(18,18,18))
}
mainGradient.Rotation = 90

local mainStroke = Instance.new("UIStroke", mainFrame)
mainStroke.Thickness = 1.6
mainStroke.Transparency = 0.25
mainStroke.Color = THEME.Accent

-- Accent bar aligned inside
local accentBar = Instance.new("Frame", mainFrame)
accentBar.Size = UDim2.new(0,6,0,40)
accentBar.Position = UDim2.new(0,10,0,8)
accentBar.BackgroundColor3 = THEME.Accent
accentBar.BorderSizePixel = 0
local accentCorner = Instance.new("UICorner", accentBar)
accentCorner.CornerRadius = UDim.new(0,4)
local accentGrad = Instance.new("UIGradient", accentBar)
accentGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255,120,120)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(120,120,255))
}
accentBar.ZIndex = 5

-- HEADER
local HEADER_HEIGHT = 44
local headerFrame = Instance.new("Frame", mainFrame)
headerFrame.Name = "Header"
headerFrame.Size = UDim2.new(1, -12, 0, HEADER_HEIGHT)
headerFrame.Position = UDim2.new(0,6,0,6)
headerFrame.BackgroundTransparency = 1
headerFrame.ZIndex = 6
local headerCorner = Instance.new("UICorner", headerFrame)
headerCorner.CornerRadius = UDim.new(0,10)

local title = Instance.new("TextLabel", headerFrame)
title.Size = UDim2.new(1, -120, 0, 26)
title.Position = UDim2.new(0,26,0,8)
title.BackgroundTransparency = 1
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.Text = "Aimbot v6 — Interface Profissional"
title.TextColor3 = THEME.Text
title.TextXAlignment = Enum.TextXAlignment.Left
title.TextTruncate = Enum.TextTruncate.AtEnd
title.ZIndex = 7

local btnEject = Instance.new("TextButton", headerFrame)
btnEject.Name = "Eject"
btnEject.Size = UDim2.new(0,18,0,18)
btnEject.Position = UDim2.new(1,-44,0,12)
btnEject.Text = "X"
btnEject.TextSize = 12
btnEject.Font = Enum.Font.GothamBold
btnEject.TextColor3 = Color3.fromRGB(240,240,240)
btnEject.BackgroundColor3 = Color3.fromRGB(160,30,30)
btnEject.BorderSizePixel = 0
local cornerEject = Instance.new("UICorner", btnEject)
cornerEject.CornerRadius = UDim.new(0,6)
btnEject.ZIndex = 8

local btnMinimize = Instance.new("TextButton", headerFrame)
btnMinimize.Name = "Minimize"
btnMinimize.Size = UDim2.new(0,18,0,18)
btnMinimize.Position = UDim2.new(1,-22,0,12)
btnMinimize.Text = "—"
btnMinimize.TextSize = 14
btnMinimize.Font = Enum.Font.GothamBold
btnMinimize.TextColor3 = THEME.Text
btnMinimize.BackgroundColor3 = Color3.fromRGB(40,40,40)
btnMinimize.BorderSizePixel = 0
local cornerMin = Instance.new("UICorner", btnMinimize)
cornerMin.CornerRadius = UDim.new(0,6)
btnMinimize.ZIndex = 8

-- LEFT BAR and CONTENT below header
local leftBar = Instance.new("Frame", mainFrame)
leftBar.Size = UDim2.new(0,120,1, -HEADER_HEIGHT - 12)
leftBar.Position = UDim2.new(0,0,0, HEADER_HEIGHT + 6)
leftBar.BackgroundTransparency = 1
leftBar.BorderSizePixel = 0
local leftCorner = Instance.new("UICorner", leftBar)
leftCorner.CornerRadius = UDim.new(0,14)
leftBar.ZIndex = 5

-- category buttons list for selection handling
local categoryButtons = {}

local function makeCategoryButton(parent, text, y, tabName)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1, -24, 0, 36)
    btn.Position = UDim2.new(0, 12, 0, y)
    btn.BackgroundColor3 = Color3.fromRGB(34,34,34)
    btn.BorderSizePixel = 0
    btn.Text = text
    btn.Font = Enum.Font.GothamSemibold
    btn.TextSize = 15
    btn.TextColor3 = THEME.Text
    btn.Name = tabName
    local c = Instance.new("UICorner", btn)
    c.CornerRadius = UDim.new(0,10)
    local stroke = Instance.new("UIStroke", btn)
    stroke.Color = Color3.fromRGB(60,60,60)
    stroke.Transparency = 0.6
    stroke.Thickness = 1

    btn.MouseEnter:Connect(function()
        if not btn:GetAttribute("Selected") then
            tweenObject(btn, {BackgroundColor3 = Color3.fromRGB(40,40,40)}, 0.12)
            tweenObject(stroke, {Transparency = 0.25}, 0.12)
        end
    end)
    btn.MouseLeave:Connect(function()
        if not btn:GetAttribute("Selected") then
            tweenObject(btn, {BackgroundColor3 = Color3.fromRGB(34,34,34)}, 0.12)
            tweenObject(stroke, {Transparency = 0.6}, 0.12)
        end
    end)

    table.insert(categoryButtons, btn)
    return btn
end

local btnAimbot = makeCategoryButton(leftBar, "AIMBOT", 12, "Aimbot")
local btnAdvanced = makeCategoryButton(leftBar, "AVANÇADO", 62, "Advanced")
local btnVisual = makeCategoryButton(leftBar, "VISUAL", 112, "Visual")

local content = Instance.new("Frame", mainFrame)
content.Size = UDim2.new(1, -120, 1, -HEADER_HEIGHT - 12)
content.Position = UDim2.new(0,120,0, HEADER_HEIGHT + 6)
content.BackgroundTransparency = 1
content.ZIndex = 5

-- UI Builders
local function makeLabel(parent, text, y)
    local lbl = Instance.new("TextLabel", parent)
    lbl.Position = UDim2.new(0,12,0,y)
    lbl.Size = UDim2.new(1,-24,0,22)
    lbl.Text = text
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 14
    lbl.TextColor3 = THEME.Text
    lbl.BackgroundTransparency = 1
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.ZIndex = 5
    return lbl
end

local function makeToggle(parent, text, y, default)
    local lbl = makeLabel(parent, text, y)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(0,66,0,26)
    btn.Position = UDim2.new(1,-78,0,y)
    btn.Text = default and "ON" or "OFF"
    btn.Font = Enum.Font.GothamSemibold
    btn.TextSize = 13
    btn.TextColor3 = THEME.Text
    btn.BackgroundColor3 = default and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
    btn.BorderSizePixel = 0
    local c = Instance.new("UICorner", btn)
    c.CornerRadius = UDim.new(0,8)
    local stroke = Instance.new("UIStroke", btn)
    stroke.Color = Color3.fromRGB(18,18,18)
    stroke.Transparency = 0.6
    stroke.Thickness = 1
    return btn
end

local function makeSlider(parent, text, y, minVal, maxVal, initial)
    makeLabel(parent, text, y)
    local sliderBg = Instance.new("Frame", parent)
    sliderBg.Size = UDim2.new(1,-28,0,18)
    sliderBg.Position = UDim2.new(0,12,0,y+24)
    sliderBg.BackgroundColor3 = Color3.fromRGB(36,36,36)
    sliderBg.BorderSizePixel = 0
    local bgCorner = Instance.new("UICorner", sliderBg)
    bgCorner.CornerRadius = UDim.new(0,8)
    local sliderBar = Instance.new("Frame", sliderBg)
    sliderBar.Size = UDim2.new(1, -8, 0, 6)
    sliderBar.Position = UDim2.new(0,4,0,6)
    sliderBar.BackgroundColor3 = Color3.fromRGB(60,60,60)
    sliderBar.BorderSizePixel = 0
    local barCorner = Instance.new("UICorner", sliderBar)
    barCorner.CornerRadius = UDim.new(0,6)
    local knob = Instance.new("ImageButton", sliderBg)
    knob.Size = UDim2.new(0,14,0,14)
    knob.Position = UDim2.new((initial-minVal)/(maxVal-minVal), -7, 0, 2)
    knob.AutoButtonColor = false
    knob.BackgroundColor3 = THEME.Accent
    knob.BorderSizePixel = 0
    local knobCorner = Instance.new("UICorner", knob)
    knobCorner.CornerRadius = UDim.new(0,8)
    local draggingKnob = false
    knob.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then draggingKnob = true end
    end)
    knob.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then draggingKnob = false end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if draggingKnob and input.UserInputType == Enum.UserInputType.MouseMovement then
            local rel = math.clamp((input.Position.X - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X, 0, 1)
            knob.Position = UDim2.new(rel, -7, 0, 2)
        end
    end)
    local function getValue()
        return minVal + knob.Position.X.Scale * (maxVal - minVal)
    end
    return { frame = sliderBg, knob = knob, getValue = getValue, setValue = function(v)
        local rel = math.clamp((v - minVal) / (maxVal - minVal), 0, 1)
        knob.Position = UDim2.new(rel, -7, 0, 2)
    end}
end

-- CONTENT FRAMES
local aimbotContent = Instance.new("Frame", content)
aimbotContent.Size = UDim2.new(1,0,1,0)
aimbotContent.BackgroundTransparency = 1
aimbotContent.Visible = true

local sensSlider = makeSlider(aimbotContent,"Sensibilidade",60,0.05,2,CONFIG.AIM_SENSITIVITY)
local fovSlider = makeSlider(aimbotContent,"Tamanho do FOV",120,20,450,CONFIG.FOV_RADIUS)
local toggleEnable = makeToggle(aimbotContent,"Ativar AIMBOT",12,CONFIG.ENABLED)
local toggleFOV = makeToggle(aimbotContent,"FOV visível",180,CONFIG.FOV_VISIBLE)
local toggleIgnoreSniper = makeToggle(aimbotContent,"Ignorar Sniper Zoom",220,CONFIG.IGNORE_SNIPER_ZOOM)

local advancedContent = Instance.new("Frame", content)
advancedContent.Size = UDim2.new(1,0,1,0)
advancedContent.BackgroundTransparency = 1
advancedContent.Visible = false
local toggleSmooth = makeToggle(advancedContent,"Smooth Aim",12,CONFIG.SMOOTH_AIM)
local toggleAutoShoot = makeToggle(advancedContent,"Auto Shoot",52,CONFIG.AUTO_SHOOT)

local lockPartOptions = {"Head","Torso","HumanoidRootPart"}
local lockPartIndex = 1
for i, v in ipairs(lockPartOptions) do
    if v == CONFIG.LOCK_PART then lockPartIndex = i break end
end
local lockPartBtn = Instance.new("TextButton", advancedContent)
lockPartBtn.Size = UDim2.new(0,150,0,28)
lockPartBtn.Position = UDim2.new(0,12,0,100)
lockPartBtn.Text = "LOCK_PART: "..CONFIG.LOCK_PART
lockPartBtn.Font = Enum.Font.GothamSemibold
lockPartBtn.TextSize = 14
lockPartBtn.TextColor3 = THEME.Text
lockPartBtn.BackgroundColor3 = Color3.fromRGB(36,36,36)
lockPartBtn.BorderSizePixel = 0
local cornerLP = Instance.new("UICorner", lockPartBtn)
cornerLP.CornerRadius = UDim.new(0,8)
lockPartBtn.MouseButton1Click:Connect(function()
    lockPartIndex = lockPartIndex + 1
    if lockPartIndex > #lockPartOptions then lockPartIndex = 1 end
    CONFIG.LOCK_PART = lockPartOptions[lockPartIndex]
    lockPartBtn.Text = "LOCK_PART: "..CONFIG.LOCK_PART
    scheduleSave()
end)

local lockKeyBtn = Instance.new("TextButton", advancedContent)
lockKeyBtn.Size = UDim2.new(0,180,0,28)
lockKeyBtn.Position = UDim2.new(0,12,0,140)
lockKeyBtn.Text = "BOTÃO AIMBOT: "..( (typeof(CONFIG.LOCK_KEY) == "EnumItem" and CONFIG.LOCK_KEY.Name) or tostring(CONFIG.LOCK_KEY) )
lockKeyBtn.Font = Enum.Font.GothamSemibold
lockKeyBtn.TextSize = 14
lockKeyBtn.TextColor3 = THEME.Text
lockKeyBtn.BackgroundColor3 = Color3.fromRGB(36,36,36)
lockKeyBtn.BorderSizePixel = 0
local cornerLK = Instance.new("UICorner", lockKeyBtn)
cornerLK.CornerRadius = UDim.new(0,8)
lockKeyBtn.MouseButton1Click:Connect(function()
    lockKeyBtn.Text = "Pressione uma tecla..."
    local connection
    connection = UserInputService.InputBegan:Connect(function(input)
        if input.UserInputType ~= Enum.UserInputType.Keyboard then
            CONFIG.LOCK_KEY = input.UserInputType
            if input.UserInputType and input.UserInputType.Name then
                lockKeyBtn.Text = "BOTÃO AIMBOT: "..input.UserInputType.Name
            else
                lockKeyBtn.Text = "BOTÃO AIMBOT: "..tostring(input.UserInputType)
            end
        else
            CONFIG.LOCK_KEY = input.KeyCode
            if input.KeyCode and input.KeyCode.Name then
                lockKeyBtn.Text = "BOTÃO AIMBOT: "..input.KeyCode.Name
            else
                lockKeyBtn.Text = "BOTÃO AIMBOT: "..tostring(input.KeyCode)
            end
        end
        scheduleSave()
        connection:Disconnect()
    end)
end)

local visualContent = Instance.new("Frame", content)
visualContent.Size = UDim2.new(1,0,1,0)
visualContent.BackgroundTransparency = 1
visualContent.Visible = false
local toggleESP = makeToggle(visualContent,"Ativar ESP",12,ESPEnabled)

-- ===== Visual: Escolha de cor do Highlight =====
-- Label
local lblHighlight = makeLabel(visualContent, "Cor do Highlight", 52)

-- Container for swatches
local swatchContainer = Instance.new("Frame", visualContent)
swatchContainer.Position = UDim2.new(0,12,0,80)
swatchContainer.Size = UDim2.new(1,-28,0,36)
swatchContainer.BackgroundTransparency = 1

-- Predefined swatches
local swatches = {
    { name = "Verde", color = Color3.fromRGB(0,255,0) },
    { name = "Vermelho", color = Color3.fromRGB(255,80,80) },
    { name = "Azul", color = Color3.fromRGB(80,160,255) },
    { name = "Amarelo", color = Color3.fromRGB(255,220,80) },
    { name = "Roxo", color = Color3.fromRGB(180,80,255) },
    { name = "Branco", color = Color3.fromRGB(230,230,230) },
    { name = "Preto", color = Color3.fromRGB(20,20,20) },
}

local swatchSize = 28
local spacing = 8
local startX = 0
local preview = Instance.new("Frame", visualContent)
preview.Size = UDim2.new(0,28,0,28)
preview.Position = UDim2.new(1,-46,0,76)
preview.BackgroundColor3 = CONFIG.HIGHLIGHT_COLOR
preview.BorderSizePixel = 0
local previewCorner = Instance.new("UICorner", preview)
previewCorner.CornerRadius = UDim.new(0,6)
local previewLabel = Instance.new("TextLabel", visualContent)
previewLabel.Size = UDim2.new(0,100,0,22)
previewLabel.Position = UDim2.new(1,-150,0,84)
previewLabel.BackgroundTransparency = 1
previewLabel.Font = Enum.Font.Gotham
previewLabel.TextSize = 12
previewLabel.TextColor3 = THEME.SubText
previewLabel.TextXAlignment = Enum.TextXAlignment.Right
previewLabel.Text = "Preview"

for i, s in ipairs(swatches) do
    local btn = Instance.new("TextButton", swatchContainer)
    btn.Size = UDim2.new(0, swatchSize, 0, swatchSize)
    btn.Position = UDim2.new(0, startX + (i-1) * (swatchSize + spacing), 0, 0)
    btn.BackgroundColor3 = s.color
    btn.Text = ""
    btn.BorderSizePixel = 0
    local c = Instance.new("UICorner", btn)
    c.CornerRadius = UDim.new(0,6)
    btn.MouseButton1Click:Connect(function()
        CONFIG.HIGHLIGHT_COLOR = s.color
        preview.BackgroundColor3 = s.color
        updateAllHighlightsColor(s.color)
        scheduleSave()
    end)
    -- hover effect
    btn.MouseEnter:Connect(function() tweenObject(btn, {Size = UDim2.new(0, swatchSize+4, 0, swatchSize+4)}, 0.12) end)
    btn.MouseLeave:Connect(function() tweenObject(btn, {Size = UDim2.new(0, swatchSize, 0, swatchSize)}, 0.12) end)
end

-- Small button to reset to default
local resetBtn = Instance.new("TextButton", visualContent)
resetBtn.Size = UDim2.new(0,112,0,26)
resetBtn.Position = UDim2.new(0,12,0,124)
resetBtn.Text = "Resetar para padrão"
resetBtn.Font = Enum.Font.GothamSemibold
resetBtn.TextSize = 13
resetBtn.TextColor3 = THEME.Text
resetBtn.BackgroundColor3 = Color3.fromRGB(36,36,36)
resetBtn.BorderSizePixel = 0
local resetCorner = Instance.new("UICorner", resetBtn)
resetCorner.CornerRadius = UDim.new(0,8)
resetBtn.MouseButton1Click:Connect(function()
    CONFIG.HIGHLIGHT_COLOR = Color3.fromRGB(0,255,0)
    preview.BackgroundColor3 = CONFIG.HIGHLIGHT_COLOR
    updateAllHighlightsColor(CONFIG.HIGHLIGHT_COLOR)
    scheduleSave()
end)

-- Make toggle buttons functional (adicionando scheduleSave() nos handlers)
toggleEnable.MouseButton1Click:Connect(function()
    CONFIG.ENABLED = not CONFIG.ENABLED
    toggleEnable.Text = CONFIG.ENABLED and "ON" or "OFF"
    toggleEnable.BackgroundColor3 = CONFIG.ENABLED and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
    scheduleSave()
end)

toggleFOV.MouseButton1Click:Connect(function()
    CONFIG.FOV_VISIBLE = not CONFIG.FOV_VISIBLE
    toggleFOV.Text = CONFIG.FOV_VISIBLE and "ON" or "OFF"
    toggleFOV.BackgroundColor3 = CONFIG.FOV_VISIBLE and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
    if FOVCircle then pcall(function() FOVCircle.Visible = CONFIG.FOV_VISIBLE end) end
    scheduleSave()
end)

toggleIgnoreSniper.MouseButton1Click:Connect(function()
    CONFIG.IGNORE_SNIPER_ZOOM = not CONFIG.IGNORE_SNIPER_ZOOM
    toggleIgnoreSniper.Text = CONFIG.IGNORE_SNIPER_ZOOM and "ON" or "OFF"
    toggleIgnoreSniper.BackgroundColor3 = CONFIG.IGNORE_SNIPER_ZOOM and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
    scheduleSave()
end)

toggleSmooth.MouseButton1Click:Connect(function()
    CONFIG.SMOOTH_AIM = not CONFIG.SMOOTH_AIM
    toggleSmooth.Text = CONFIG.SMOOTH_AIM and "ON" or "OFF"
    toggleSmooth.BackgroundColor3 = CONFIG.SMOOTH_AIM and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
    scheduleSave()
end)

toggleAutoShoot.MouseButton1Click:Connect(function()
    CONFIG.AUTO_SHOOT = not CONFIG.AUTO_SHOOT
    toggleAutoShoot.Text = CONFIG.AUTO_SHOOT and "ON" or "OFF"
    toggleAutoShoot.BackgroundColor3 = CONFIG.AUTO_SHOOT and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
    scheduleSave()
end)

-- Substitui o handler direto por uma função reutilizável e adiciona atalho (seta para baixo)
-- Função utilitária para ativar/desativar ESP (usar no botão e na tecla)
local function setESPEnabled(value)
    ESPEnabled = value
    -- Atualiza botão visual (se existir)
    if toggleESP then
        toggleESP.Text = ESPEnabled and "ON" or "OFF"
        toggleESP.BackgroundColor3 = ESPEnabled and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
    end

    -- Cria ou remove highlights
    if ESPEnabled then
        for _, plr in pairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character and plr.Character:FindFirstChildOfClass("Humanoid") then
                -- createHighlight já destrói o highlight anterior se existir
                createHighlight(plr)
            end
        end
        updateAllHighlightsColor(CONFIG.HIGHLIGHT_COLOR)
    else
        for _, highlight in pairs(ESPContainers) do
            pcall(function() highlight:Destroy() end)
        end
        ESPContainers = {}
    end

    scheduleSave()
end

-- Conecta o botão do UI para usar a função centralizada
if toggleESP then
    toggleESP.MouseButton1Click:Connect(function()
        setESPEnabled(not ESPEnabled)
    end)
end

-- Atalho: Seta para baixo para alternar ESP (Down Arrow)
local downToggleConn = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.Down then
        setESPEnabled(not ESPEnabled)
    end
end)
table.insert(activeConnections, downToggleConn)

-- Tab switching & selection handling
local function setCategorySelected(selectedBtn)
    for _, btn in ipairs(categoryButtons) do
        if btn == selectedBtn then
            btn:SetAttribute("Selected", true)
            btn.BackgroundColor3 = Color3.fromRGB(24,24,24)
            btn.TextColor3 = Color3.fromRGB(245,245,245)
        else
            btn:SetAttribute("Selected", false)
            btn.BackgroundColor3 = Color3.fromRGB(34,34,34)
            btn.TextColor3 = THEME.Text
        end
    end
end

local function switchTab(tabName)
    aimbotContent.Visible = false
    advancedContent.Visible = false
    visualContent.Visible = false
    if tabName == "Aimbot" then
        aimbotContent.Visible = true
        setCategorySelected(btnAimbot)
    elseif tabName == "Advanced" then
        advancedContent.Visible = true
        setCategorySelected(btnAdvanced)
    elseif tabName == "Visual" then
        visualContent.Visible = true
        setCategorySelected(btnVisual)
    end
end

btnAimbot.MouseButton1Click:Connect(function() switchTab("Aimbot") end)
btnAdvanced.MouseButton1Click:Connect(function() switchTab("Advanced") end)
btnVisual.MouseButton1Click:Connect(function() switchTab("Visual") end)

-- ensure initial selection is AIMBOT and sync toggle visuals with config
switchTab("Aimbot")
toggleEnable.Text = CONFIG.ENABLED and "ON" or "OFF"
toggleEnable.BackgroundColor3 = CONFIG.ENABLED and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
toggleFOV.Text = CONFIG.FOV_VISIBLE and "ON" or "OFF"
toggleFOV.BackgroundColor3 = CONFIG.FOV_VISIBLE and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
toggleIgnoreSniper.Text = CONFIG.IGNORE_SNIPER_ZOOM and "ON" or "OFF"
toggleIgnoreSniper.BackgroundColor3 = CONFIG.IGNORE_SNIPER_ZOOM and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
toggleSmooth.Text = CONFIG.SMOOTH_AIM and "ON" or "OFF"
toggleSmooth.BackgroundColor3 = CONFIG.SMOOTH_AIM and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
toggleAutoShoot.Text = CONFIG.AUTO_SHOOT and "ON" or "OFF"
toggleAutoShoot.BackgroundColor3 = CONFIG.AUTO_SHOOT and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
toggleESP.Text = ESPEnabled and "ON" or "OFF"
toggleESP.BackgroundColor3 = ESPEnabled and Color3.fromRGB(30,135,84) or Color3.fromRGB(60,60,60)
preview.BackgroundColor3 = CONFIG.HIGHLIGHT_COLOR

-- FOOTER: copyright and skull image (bottom-left) using provided asset id with preload & fallback
local footer = Instance.new("Frame", mainFrame)
footer.Name = "Footer"
footer.Size = UDim2.new(0,200,0,24)
footer.Position = UDim2.new(0,10,1,-34)
footer.BackgroundTransparency = 1
footer.ZIndex = 9

local skullImg = Instance.new("ImageButton", footer)
skullImg.Name = "SkullImg"
skullImg.Size = UDim2.new(0,18,0,18)
skullImg.Position = UDim2.new(0,0,0,3)
skullImg.BackgroundTransparency = 1
skullImg.ScaleType = Enum.ScaleType.Fit
skullImg.ZIndex = 10
skullImg.AutoButtonColor = false

local skullCorner = Instance.new("UICorner", skullImg)
skullCorner.CornerRadius = UDim.new(0,4)

local copyrightLabel = Instance.new("TextLabel", footer)
copyrightLabel.Name = "Copyright"
copyrightLabel.Size = UDim2.new(1,-26,1,0)
copyrightLabel.Position = UDim2.new(0,26,0,0)
copyrightLabel.BackgroundTransparency = 1
copyrightLabel.Font = Enum.Font.Gotham
copyrightLabel.TextSize = 14
copyrightLabel.Text = "by JackViram"
copyrightLabel.TextColor3 = THEME.SubText
copyrightLabel.TextXAlignment = Enum.TextXAlignment.Left
copyrightLabel.TextYAlignment = Enum.TextYAlignment.Center
copyrightLabel.ZIndex = 10

-- preload and fallback
local preloadOk, preloadErr = pcall(function()
    ContentProvider:PreloadAsync({ "rbxassetid://" .. SKULL_ASSET_ID })
end)

if preloadOk then
    skullImg.Image = "rbxassetid://" .. SKULL_ASSET_ID
else
    skullImg:Destroy()
    local skullLabel = Instance.new("TextLabel", footer)
    skullLabel.Name = "SkullEmoji"
    skullLabel.Size = UDim2.new(0,20,1,0)
    skullLabel.Position = UDim2.new(0,0,0,0)
    skullLabel.BackgroundTransparency = 1
    skullLabel.Font = Enum.Font.GothamBold
    skullLabel.TextSize = 16
    skullLabel.Text = "💀"
    skullLabel.TextColor3 = THEME.Text
    skullLabel.TextXAlignment = Enum.TextXAlignment.Center
    skullLabel.TextYAlignment = Enum.TextYAlignment.Center
    skullLabel.ZIndex = 10
end

if skullImg and skullImg.Parent then
    skullImg.MouseEnter:Connect(function()
        pcall(function() tweenObject(skullImg, {Size = UDim2.new(0,22,0,22)}, 0.12) end)
    end)
    skullImg.MouseLeave:Connect(function()
        pcall(function() tweenObject(skullImg, {Size = UDim2.new(0,18,0,18)}, 0.12) end)
    end)
    skullImg.MouseButton1Click:Connect(function()
        pcall(function()
            tweenObject(skullImg, {Size = UDim2.new(0,20,0,20)}, 0.06)
            wait(0.06)
            tweenObject(skullImg, {Size = UDim2.new(0,18,0,18)}, 0.08)
        end)
    end)
end

-- Update config from sliders (com detecção de mudança para salvar)
spawn(function()
    local prevSens = CONFIG.AIM_SENSITIVITY
    local prevFov = CONFIG.FOV_RADIUS
    while true do
        wait(0.05)
        local newSens = sensSlider.getValue()
        local newFov = fovSlider.getValue()
        if newSens ~= CONFIG.AIM_SENSITIVITY then
            CONFIG.AIM_SENSITIVITY = newSens
        end
        if newFov ~= CONFIG.FOV_RADIUS then
            CONFIG.FOV_RADIUS = newFov
        end
        -- detecta mudanças e agenda salvamento
        if math.abs(prevSens - CONFIG.AIM_SENSITIVITY) > 1e-6 or math.abs(prevFov - CONFIG.FOV_RADIUS) > 1e-6 then
            prevSens = CONFIG.AIM_SENSITIVITY
            prevFov = CONFIG.FOV_RADIUS
            scheduleSave()
        end
    end
end)

-- Minimize / Eject handlers
local minimized = false
local function ToggleUI()
    minimized = not minimized
    mainFrame.Visible = not minimized
end
btnMinimize.MouseButton1Click:Connect(function() ToggleUI() end)
btnEject.MouseButton1Click:Connect(function()
    -- salvar antes de ejetar/remover
    pcall(saveLocalSettings)
    for _, conn in pairs(activeConnections) do pcall(function() conn:Disconnect() end) end
    if aimbotConnection then pcall(function() aimbotConnection:Disconnect() end) end
    if FOVCircle then pcall(function() FOVCircle:Remove() end) end
    for _, highlight in pairs(ESPContainers) do pcall(function() highlight:Destroy() end) end
    ESPContainers = {}
    pcall(function() screenGui:Destroy() end)
end)

-- Keyboard shortcuts
local deleteConn = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.KeyCode == Enum.KeyCode.Delete and not gameProcessed then
        pcall(saveLocalSettings)
        for _, conn in pairs(activeConnections) do pcall(function() conn:Disconnect() end) end
        if aimbotConnection then pcall(function() aimbotConnection:Disconnect() end) end
        if FOVCircle then pcall(function() FOVCircle:Remove() end) end
        for _, highlight in pairs(ESPContainers) do pcall(function() highlight:Destroy() end) end
        ESPContainers = {}
        pcall(function() screenGui:Destroy() end)
    end
end)
table.insert(activeConnections, deleteConn)

local rcConn = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.KeyCode == Enum.KeyCode.RightControl and not gameProcessed then ToggleUI() end
end)
table.insert(activeConnections, rcConn)
table.insert(activeConnections, ESPConnection)

-- Dragging: header is drag handle (offset-within-frame approach)
local dragging = false
local dragStart = Vector2.new(0,0)
local offsetWithin = Vector2.new(0,0)

headerFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = UserInputService:GetMouseLocation()
        offsetWithin = dragStart - mainFrame.AbsolutePosition
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

headerFrame.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local mousePos = UserInputService:GetMouseLocation()
        local desired = mousePos - offsetWithin
        local vp = Camera and Camera.ViewportSize or Vector2.new(1920,1080)
        local frameSize = mainFrame.AbsoluteSize
        local maxX = math.max(0, vp.X - frameSize.X)
        local maxY = math.max(0, vp.Y - frameSize.Y)
        local clampedX = math.clamp(desired.X, 0, maxX)
        local clampedY = math.clamp(desired.Y, 0, maxY)
        mainFrame.Position = UDim2.new(0, clampedX, 0, clampedY)
    end
end)

-- Animated RGB accent
local startTick = tick()
local function updateAccentColor()
    local hue = (tick() - startTick) % 6 / 6
    local rgb = Color3.fromHSV(hue, 0.85, 0.95)
    mainStroke.Color = rgb
    accentBar.BackgroundColor3 = rgb
    accentGrad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, rgb),
        ColorSequenceKeypoint.new(1, Color3.fromHSV((hue+0.25)%1, 0.8, 0.95))
    }
    for _, frame in pairs(aimbotContent:GetDescendants()) do
        if frame:IsA("ImageButton") and frame.Name ~= "Close" then frame.BackgroundColor3 = rgb end
    end
    for _, frame in pairs(advancedContent:GetDescendants()) do
        if frame:IsA("ImageButton") then frame.BackgroundColor3 = rgb end
    end
end
local accentConn = RunService.RenderStepped:Connect(updateAccentColor)
table.insert(activeConnections, accentConn)

-- Garante que os sliders e previews comecem com os valores carregados
pcall(function()
    if sensSlider and sensSlider.setValue then sensSlider.setValue(CONFIG.AIM_SENSITIVITY) end
    if fovSlider and fovSlider.setValue then fovSlider.setValue(CONFIG.FOV_RADIUS) end
    preview.BackgroundColor3 = CONFIG.HIGHLIGHT_COLOR
end)

-- Salva inicial (garante arquivo criado)
pcall(function() scheduleSave() end)

-- End of GUI
-- Script funcional (ESP, aimbot, etc.) mantido.
]]

-- Tenta registrar para queue_on_teleport (vários executores)
local function try_queue_on_teleport(s)
    local ok, res = pcall(function()
        if syn and syn.queue_on_teleport then
            syn.queue_on_teleport(s)
            return true
        elseif queue_on_teleport then
            queue_on_teleport(s)
            return true
        elseif fluxus and fluxus.queue_on_teleport then
            fluxus.queue_on_teleport(s)
            return true
        end
        return false
    end)
    return ok and res
end

pcall(function() try_queue_on_teleport(SCRIPT) end)

-- Tenta executar o script agora (em muitos executores loadstring ou load existe)
local loader = loadstring or load
if loader then
    local ok, fn = pcall(loader, SCRIPT)
    if ok and fn then
        pcall(function() fn() end)
    end
else
    -- Se não houver loadstring/load (ambiente Roblox normal), o script deve estar
    -- colocado em StarterPlayerScripts para rodar automaticamente em cada servidor.
    -- Se você tem acesso ao jogo: mover o LocalScript para StarterPlayer > StarterPlayerScripts resolve o problema.
end

-- Fim do loader
