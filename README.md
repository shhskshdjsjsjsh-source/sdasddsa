--[[
    NOVA CHEST V3 - MEGA ROBUST EDITION
    ESTADO: ESTÁVEL / ANTI-CRASH / UI FIX
]]

-- // SEGURANÇA DE EXECUÇÃO
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then 
    warn("Nova Chest já está em execução!")
    return 
end
_G.Nova_Loaded = true
_G.Start_Kaitun = true

-- // SERVIÇOS DO JOGO
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players = Services.Players
local TS = Services.TweenService
local HTTP = Services.HttpService
local Teleport = Services.TeleportService
local RunS = Services.RunService
local VU = Services.VirtualUser
local SG = Services.StarterGui
local LocalPlayer = Players.LocalPlayer

-- // CONFIGURAÇÃO INTERNA (FALLBACK)
local Settings = {
    ChestLimit = 100,
    TweenSpeed = 165,
    AutoTP = true,
    DistanceToCollect = 20
}

-- Tenta sincronizar com sua config externa se existir
if _G.Settings and _G.Settings.Main then
    Settings.ChestLimit = _G.Settings.Main["chest limit"] or 100
    Settings.TweenSpeed = _G.Settings.Main["tween speed"] or 165
end

-- // VARIÁVEIS DE CONTROLE
local State = {
    Counter = 0,
    Blacklist = {},
    IsHopping = false,
    ValidWorlds = {2753915549, 4442272183, 7449423635}
}

-- // [CONSTRUÇÃO DA INTERFACE V3 - GRANDE]
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaChest_V3_Stable"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

-- Janela Principal (Centro-Topo)
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 250, 0, 160)
MainFrame.Position = UDim2.new(0.5, -125, 0.1, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Thickness = 2
MainStroke.Color = Color3.fromRGB(0, 170, 255)
MainStroke.Parent = MainFrame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 40)
Title.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
Title.Text = "NOVA CHEST V3"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 20
Title.Parent = MainFrame
local TitleCorner = Instance.new("UICorner")
TitleCorner.Parent = Title

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, 0, 0, 30)
StatusLabel.Position = UDim2.new(0, 0, 0, 45)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Aguardando..."
StatusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
StatusLabel.Font = Enum.Font.SourceSansItalic
StatusLabel.TextSize = 16
StatusLabel.Parent = MainFrame

local StopBtn = Instance.new("TextButton")
StopBtn.Size = UDim2.new(0, 220, 0, 45)
StopBtn.Position = UDim2.new(0, 15, 0, 90)
StopBtn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
StopBtn.Text = "PARAR AUTO CHEST"
StopBtn.TextColor3 = Color3.new(1,1,1)
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 18
StopBtn.Parent = MainFrame
local BtnCorner = Instance.new("UICorner")
BtnCorner.Parent = StopBtn

-- Janela de Informações (Canto Inferior)
local InfoFrame = Instance.new("Frame")
InfoFrame.Size = UDim2.new(0, 220, 0, 120)
InfoFrame.Position = UDim2.new(1, -230, 0.75, 0)
InfoFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
InfoFrame.BackgroundTransparency = 0.1
InfoFrame.Active = true
InfoFrame.Draggable = true
InfoFrame.Parent = ScreenGui

local InfoCorner = Instance.new("UICorner")
InfoCorner.Parent = InfoFrame

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -20, 1, -10)
InfoText.Position = UDim2.new(0, 10, 0, 5)
InfoText.BackgroundTransparency = 1
InfoText.TextColor3 = Color3.fromRGB(255, 255, 255)
InfoText.TextXAlignment = Enum.TextXAlignment.Left
InfoText.Font = Enum.Font.Code
InfoText.TextSize = 14
InfoText.Parent = InfoFrame

-- // [FUNÇÕES DE APOIO]
local function GetHRP()
    return LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
end

local function GetExecutor()
    return (identifyexecutor or function() return "Unknown" end)()
end

-- Server Hop Otimizado
local function ServerHop()
    if State.IsHopping then return end
    State.IsHopping = true
    StatusLabel.Text = "Status: Trocando Servidor..."
    
    local function Hop()
        pcall(function()
            local Servers = HTTP:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"))
            for _, s in pairs(Servers.data) do
                if s.playing < s.maxPlayers and s.id ~= game.JobId then
                    Teleport:TeleportToPlaceInstance(game.PlaceId, s.id, LocalPlayer)
                end
            end
        end)
    end
    
    while task.wait(2) do Hop() end
end

-- // [BOTÃO TOGGLE]
StopBtn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    if _G.Start_Kaitun then
        StopBtn.Text = "PARAR AUTO CHEST"
        StopBtn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
        StatusLabel.Text = "Status: Ativado"
    else
        StopBtn.Text = "INICIAR AUTO CHEST"
        StopBtn.BackgroundColor3 = Color3.fromRGB(60, 220, 60)
        StatusLabel.Text = "Status: Parado"
    end
end)

-- // [SISTEMA DE INFOS (LOOP)]
task.spawn(function()
    while task.wait(0.2) do
        local fps = math.floor(1/RunS.RenderStepped:Wait())
        InfoText.Text = string.format(
            "EXECUTOR: %s\nFPS: %d\nVERSÃO: V3 STABLE\nROBLOX: %s\nBAÚS: %d",
            GetExecutor(),
            fps,
            version():sub(1,10),
            State.Counter
        )
    end
end)

-- // [LÓGICA PRINCIPAL DE FARM]
task.spawn(function()
    while true do
        task.wait(0.1)
        
        if _G.Start_Kaitun then
            if State.Counter >= Settings.ChestLimit then
                ServerHop()
                break
            end

            local hrp = GetHRP()
            local targetChest = nil
            local minDistance = math.huge

            -- Escaneia baús com pcall para evitar erros de leitura de memória
            pcall(function()
                for _, obj in pairs(workspace:GetDescendants()) do
                    if obj:IsA("TouchTransmitter") and obj.Parent then
                        local chest = obj.Parent
                        if chest.Name:find("Chest") or chest.Name:find("Baú") then
                            if not State.Blacklist[chest] and hrp then
                                local dist = (hrp.Position - chest.Position).Magnitude
                                if dist < minDistance then
                                    minDistance = dist
                                    targetChest = chest
                                end
                            end
                        end
                    end
                end
            end)

            if targetChest then
                StatusLabel.Text = "Status: Indo para Baú"
                
                -- Movimento (TP ou Tween)
                if Settings.AutoTP then
                    hrp.CFrame = targetChest.CFrame
                    task.wait(0.2)
                else
                    local tween = TS:Create(hrp, TweenInfo.new(minDistance/Settings.TweenSpeed, Enum.EasingStyle.Linear), {CFrame = targetChest.CFrame})
                    tween:Play()
                    
                    repeat task.wait() until (hrp.Position - targetChest.Position).Magnitude < 10 or not _G.Start_Kaitun or not targetChest.Parent
                    tween:Cancel()
                end

                -- Coleta e Blacklist
                if (hrp.Position - targetChest.Position).Magnitude < Settings.DistanceToCollect then
                    State.Blacklist[targetChest] = true
                    State.Counter = State.Counter + 1
                    StatusLabel.Text = "Status: Baú Coletado!"
                    task.wait(0.1)
                end
            else
                StatusLabel.Text = "Status: Sem baús. Pulando..."
                ServerHop()
                break
            end
        end
    end
end)

-- // [SISTEMAS DE PROTEÇÃO (GIGANTE)]
-- NoClip
RunS.Stepped:Connect(function()
    if _G.Start_Kaitun and LocalPlayer.Character then
        for _, part in pairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end
end)

-- Anti-AFK
LocalPlayer.Idled:Connect(function()
    VU:CaptureController()
    VU:ClickButton2(Vector2.new())
end)

-- Tratamento de erros de CoreGui (Garante que a UI não suma)
ScreenGui.DescendantRemoving:Connect(function(obj)
    if obj == MainFrame then
        _G.Nova_Loaded = false -- Permite re-executar se a UI for deletada
    end
end)

print([[ 
    ------------------------------
    NOVA CHEST V3 CARREGADO!
    STATUS: OK
    ------------------------------
]])
