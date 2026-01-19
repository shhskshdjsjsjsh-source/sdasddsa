--[[ 
    NOVA CHEST V3 - PREMIUM STABLE VERSION
    FIX: UI REFEITA, INFO SYSTEM & TOGGLE BUTTON
]]

if not _G.Start_Kaitun then _G.Start_Kaitun = true end
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then return end
_G.Nova_Loaded = true

-- // SERVIÇOS
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players, TS, HTTP, Teleport, RunS, VU, SG = Services.Players, Services.TweenService, Services.HttpService, Services.TeleportService, Services.RunService, Services.VirtualUser, Services.StarterGui
local Player = Players.LocalPlayer

-- // CONFIGURAÇÕES (PADRÃO SE NÃO EXISTIR)
local Config = {
    ["auto chest tp"] = true,
    ["auto chest twen"] = false,
    ["chest limit"] = 50,
    ["tween speed"] = 150
}

if _G.Settings and _G.Settings.Main then
    Config = _G.Settings.Main
end

-- // VARIÁVEIS DE ESTADO
local Internal = {
    Counter = 0,
    Blacklist = {},
    CurrentTarget = nil,
    HopLock = false,
    WorldValid = (game.PlaceId == 2753915549 or game.PlaceId == 4442272183 or game.PlaceId == 7449423635)
}

-- // [SISTEMA DE UI - NOVA CHEST V3]
local PlayerGui = Player:WaitForChild("PlayerGui")
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaChestV3_System"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

-- JANELA PRINCIPAL
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 210, 0, 140)
MainFrame.Position = UDim2.new(0.5, -105, 0.2, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 6)
UICorner.Parent = MainFrame

local UIStroke = Instance.new("UIStroke")
UIStroke.Color = Color3.fromRGB(0, 170, 255)
UIStroke.Thickness = 2
UIStroke.Parent = MainFrame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Text = "NOVA CHEST V3"
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 16
Title.Parent = MainFrame

local ModeLabel = Instance.new("TextLabel")
ModeLabel.Size = UDim2.new(1, 0, 0, 25)
ModeLabel.Position = UDim2.new(0, 0, 0, 35)
ModeLabel.BackgroundTransparency = 1
ModeLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
ModeLabel.Text = "Status: Iniciando..."
ModeLabel.Font = Enum.Font.SourceSans
ModeLabel.TextSize = 14
ModeLabel.Parent = MainFrame

local StopBtn = Instance.new("TextButton")
StopBtn.Name = "StopBtn"
StopBtn.Size = UDim2.new(0, 190, 0, 35)
StopBtn.Position = UDim2.new(0, 10, 0, 95)
StopBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
StopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StopBtn.Text = "PARAR AUTO CHEST"
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 14
StopBtn.Parent = MainFrame

local BtnCorner = Instance.new("UICorner")
BtnCorner.CornerRadius = UDim.new(0, 4)
BtnCorner.Parent = StopBtn

-- JANELA DE INFORMAÇÕES (MÓVEL NO CANTO)
local InfoFrame = Instance.new("Frame")
InfoFrame.Size = UDim2.new(0, 190, 0, 110)
InfoFrame.Position = UDim2.new(1, -200, 0.7, 0)
InfoFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
InfoFrame.BackgroundTransparency = 0.1
InfoFrame.Active = true
InfoFrame.Draggable = true
InfoFrame.Parent = ScreenGui

local InfoCorner = Instance.new("UICorner")
InfoCorner.Parent = InfoFrame

local InfoStroke = Instance.new("UIStroke")
InfoStroke.Color = Color3.fromRGB(255, 255, 255)
InfoStroke.Transparency = 0.8
InfoStroke.Parent = InfoFrame

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -15, 1, -15)
InfoText.Position = UDim2.new(0, 10, 0, 5)
InfoText.BackgroundTransparency = 1
InfoText.TextColor3 = Color3.fromRGB(255, 255, 255)
InfoText.TextXAlignment = Enum.TextXAlignment.Left
InfoText.Font = Enum.Font.Code
InfoText.TextSize = 13
InfoText.Parent = InfoFrame

-- // [LÓGICA DE INTERFACE]
StopBtn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    if _G.Start_Kaitun then
        StopBtn.Text = "PARAR AUTO CHEST"
        StopBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
        ModeLabel.Text = "Status: Rodando"
    else
        StopBtn.Text = "INICIAR AUTO CHEST"
        StopBtn.BackgroundColor3 = Color3.fromRGB(50, 200, 50)
        ModeLabel.Text = "Status: Pausado"
    end
end)

local function GetExecutor()
    return (identifyexecutor or function() return "Desconhecido" end)()
end

task.spawn(function()
    while task.wait(0.3) do
        local fps = math.floor(1/RunS.RenderStepped:Wait())
        InfoText.Text = string.format(
            "EXECUTOR: %s\nFPS: %d\nVERSÃO: V3 Fix\nROBLOX: %s\nBAÚS: %d",
            GetExecutor(),
            fps,
            version(),
            Internal.Counter
        )
    end
end)

-- // [FUNÇÕES DE MOVIMENTO]
local function GetHRP() return Player.Character and Player.Character:FindFirstChild("HumanoidRootPart") end

local function ExecuteTween(target)
    local hrp = GetHRP()
    if not hrp or not target then return end
    local dist = (hrp.Position - target.Position).Magnitude
    local speed = Config["tween speed"] or 150
    local tTime = dist / speed
    local tween = TS:Create(hrp, TweenInfo.new(tTime, Enum.EasingStyle.Linear), {CFrame = target.CFrame})
    
    local finished = false
    local connection
    connection = tween.Completed:Connect(function() 
        finished = true 
        if connection then connection:Disconnect() end
    end)
    
    tween:Play()
    while not finished and _G.Start_Kaitun do
        if not target.Parent then tween:Cancel() break end
        hrp.Velocity = Vector3.new(0, 0.05, 0)
        task.wait()
    end
end

local function ExecuteTP(target)
    local hrp = GetHRP()
    if not hrp or not target then return end
    hrp.CFrame = target.CFrame + Vector3.new(0, 2, 0)
    task.wait(0.1)
    hrp.CFrame = target.CFrame
    task.wait(0.1)
end

-- // [SERVER HOP]
local function PerformServerHop()
    if not Internal.WorldValid or Internal.HopLock then return end
    Internal.HopLock = true
    ModeLabel.Text = "Status: Mudando de Servidor"
    
    pcall(function()
        local url = "https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"
        local res = game:HttpGet(url)
        local body = HTTP:JSONDecode(res)
        if body and body.data then
            for _, s in pairs(body.data) do
                if s.playing < s.maxPlayers and s.id ~= game.JobId then
                    Teleport:TeleportToPlaceInstance(game.PlaceId, s.id, Player)
                    return
                end
            end
        end
        Teleport:Teleport(game.PlaceId, Player)
    end)
end

-- // [LOOP PRINCIPAL]
task.spawn(function()
    while true do
        task.wait(0.5)
        if _G.Start_Kaitun then
            if Internal.Counter >= (Config["chest limit"] or 100) then
                PerformServerHop()
                break
            end

            local nearest = nil
            local min_dist = math.huge
            
            -- Busca otimizada por baús
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("TouchTransmitter") and v.Parent and (v.Parent.Name:find("Chest") or v.Parent.Name:find("Baú")) then
                    if not Internal.Blacklist[v.Parent] then
                        local hrp = GetHRP()
                        if hrp then
                            local d = (hrp.Position - v.Parent.Position).Magnitude
                            if d < min_dist then 
                                min_dist = d
                                nearest = v.Parent 
                            end
                        end
                    end
                end
            end

            if nearest then
                ModeLabel.Text = "Status: Coletando Baú"
                if Config["auto chest tp"] then 
                    ExecuteTP(nearest)
                else 
                    ExecuteTween(nearest) 
                end
                
                -- Verifica se chegou perto o suficiente
                local hrp = GetHRP()
                if hrp and (hrp.Position - nearest.Position).Magnitude < 25 then
                    Internal.Blacklist[nearest] = true
                    Internal.Counter = Internal.Counter + 1
                end
            else
                PerformServerHop()
                break
            end
        else
            ModeLabel.Text = "Status: Pausado"
        end
    end
end)

-- Anti AFK & NoClip (Loop de Renderização)
RunS.Stepped:Connect(function()
    if _G.Start_Kaitun and Player.Character then
        for _, v in pairs(Player.Character:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
    end
end)

-- Previne Anti-AFK do Roblox
Player.Idled:Connect(function()
    VU:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    task.wait(1)
    VU:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
end)

print("Nova Chest Engine V3 Carregada com Sucesso.")
