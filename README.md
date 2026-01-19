--[[
    NOVA CHEST V3 - MEGA ROBUST EDITION (TWEEN + TP FIX)
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
local LocalPlayer = Players.LocalPlayer

-- // CONFIGURAÇÃO INTERNA (FALLBACK)
local Settings = {
    ChestLimit = 100,
    TweenSpeed = 165,
    AutoTP = false,      -- Se true, usa Teleporte instantâneo
    AutoTween = true,   -- Se true, usa o movimento suave (deslizar)
    DistanceToCollect = 20
}

-- Sincronização com config externa (Se existir)
if _G.Settings and _G.Settings.Main then
    local cfg = _G.Settings.Main
    Settings.ChestLimit = cfg["chest limit"] or 100
    Settings.TweenSpeed = cfg["tween speed"] or 165
    -- Detecta o modo vindo da config global
    if cfg["auto chest tp"] ~= nil then Settings.AutoTP = cfg["auto chest tp"] end
    if cfg["auto chest twen"] ~= nil then Settings.AutoTween = cfg["auto chest twen"] end
end

-- // VARIÁVEIS DE CONTROLE
local State = {
    Counter = 0,
    Blacklist = {},
    IsHopping = false,
}

-- // [CONSTRUÇÃO DA INTERFACE V3]
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaChest_V3_Stable"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 250, 0, 165)
MainFrame.Position = UDim2.new(0.5, -125, 0.1, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
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
Title.Size = UDim2.new(1, 0, 0, 35)
Title.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
Title.Text = "NOVA CHEST V3 (MODES)"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 18
Title.Parent = MainFrame
local TitleCorner = Instance.new("UICorner")
TitleCorner.Parent = Title

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, 0, 0, 25)
StatusLabel.Position = UDim2.new(0, 0, 0, 40)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Aguardando..."
StatusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
StatusLabel.Font = Enum.Font.SourceSansItalic
StatusLabel.TextSize = 14
StatusLabel.Parent = MainFrame

local ModeDisplay = Instance.new("TextLabel")
ModeDisplay.Size = UDim2.new(1, 0, 0, 25)
ModeDisplay.Position = UDim2.new(0, 0, 0, 65)
ModeDisplay.BackgroundTransparency = 1
ModeDisplay.TextColor3 = Color3.fromRGB(0, 255, 127)
ModeDisplay.Font = Enum.Font.SourceSansBold
ModeDisplay.TextSize = 14
ModeDisplay.Parent = MainFrame

local StopBtn = Instance.new("TextButton")
StopBtn.Size = UDim2.new(0, 220, 0, 40)
StopBtn.Position = UDim2.new(0, 15, 0, 105)
StopBtn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
StopBtn.Text = "PARAR AUTO CHEST"
StopBtn.TextColor3 = Color3.new(1,1,1)
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 18
StopBtn.Parent = MainFrame
local BtnCorner = Instance.new("UICorner")
BtnCorner.Parent = StopBtn

-- Janela de Infos (FPS/Executor)
local InfoFrame = Instance.new("Frame")
InfoFrame.Size = UDim2.new(0, 200, 0, 100)
InfoFrame.Position = UDim2.new(1, -210, 0.8, 0)
InfoFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
InfoFrame.Parent = ScreenGui
local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -20, 1, -10)
InfoText.Position = UDim2.new(0, 10, 0, 5)
InfoText.BackgroundTransparency = 1
InfoText.TextColor3 = Color3.fromRGB(255, 255, 255)
InfoText.Font = Enum.Font.Code
InfoText.TextSize = 13
InfoText.Parent = InfoFrame

-- // [FUNÇÕES DE MOVIMENTO]
local function GetHRP()
    return LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
end

-- Função de deslizar (Tween)
local function TweenTo(targetCFrame)
    local hrp = GetHRP()
    if not hrp then return end
    
    local distance = (hrp.Position - targetCFrame.Position).Magnitude
    local duration = distance / Settings.TweenSpeed
    
    local info = TweenInfo.new(duration, Enum.EasingStyle.Linear)
    local tween = TS:Create(hrp, info, {CFrame = targetCFrame})
    
    local completed = false
    local connection
    connection = tween.Completed:Connect(function()
        completed = true
        connection:Disconnect()
    end)
    
    tween:Play()
    
    -- Loop enquanto o tween acontece para garantir estabilidade e noclip
    while not completed and _G.Start_Kaitun do
        hrp.Velocity = Vector3.new(0, 0.05, 0) -- Evita queda por gravidade
        if not hrp or not hrp.Parent then break end
        task.wait()
    end
    
    if not _G.Start_Kaitun then tween:Cancel() end
end

-- Server Hop
local function ServerHop()
    if State.IsHopping then return end
    State.IsHopping = true
    StatusLabel.Text = "Status: Trocando Servidor..."
    pcall(function()
        local Servers = HTTP:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"))
        for _, s in pairs(Servers.data) do
            if s.playing < s.maxPlayers and s.id ~= game.JobId then
                Teleport:TeleportToPlaceInstance(game.PlaceId, s.id, LocalPlayer)
            end
        end
    end)
end

-- // [LÓGICA PRINCIPAL]
task.spawn(function()
    while true do
        task.wait(0.1)
        
        -- Atualiza Modo na UI
        if Settings.AutoTP then ModeDisplay.Text = "MODO: TELEPORT (INSTANT)"
        elseif Settings.AutoTween then ModeDisplay.Text = "MODO: TWEEN (SMOOTH)"
        else ModeDisplay.Text = "MODO: DESATIVADO" end

        if _G.Start_Kaitun then
            if State.Counter >= Settings.ChestLimit then
                ServerHop()
                break
            end

            local hrp = GetHRP()
            local targetChest = nil
            local minDistance = math.huge

            -- Busca baús
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
                
                if Settings.AutoTP then
                    -- Lógica de Teleporte
                    hrp.CFrame = targetChest.CFrame
                    task.wait(0.1)
                elseif Settings.AutoTween then
                    -- Lógica de Tween (Deslizar)
                    TweenTo(targetChest.CFrame)
                end

                -- Coleta
                if (hrp.Position - targetChest.Position).Magnitude < Settings.DistanceToCollect then
                    State.Blacklist[targetChest] = true
                    State.Counter = State.Counter + 1
                    StatusLabel.Text = "Status: Coletado ("..State.Counter..")"
                end
            else
                StatusLabel.Text = "Status: Buscando novos baús..."
                ServerHop()
                break
            end
        end
    end
end)

-- Botão Stop
StopBtn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    StopBtn.Text = _G.Start_Kaitun and "PARAR AUTO CHEST" or "INICIAR AUTO CHEST"
    StopBtn.BackgroundColor3 = _G.Start_Kaitun and Color3.fromRGB(220, 60, 60) or Color3.fromRGB(60, 220, 60)
end)

-- Loop de Info
task.spawn(function()
    while task.wait(0.5) do
        local fps = math.floor(1/RunS.RenderStepped:Wait())
        InfoText.Text = string.format("FPS: %d\nWORLD: %s\nBAÚS: %d/%d\nESTADO: OK", 
            fps, game.PlaceId, State.Counter, Settings.ChestLimit)
    end
end)

-- NoClip Permanente enquanto ativo
RunS.Stepped:Connect(function()
    if _G.Start_Kaitun and LocalPlayer.Character then
        for _, part in pairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then part.CanCollide = false end
        end
    end
end)

-- Anti-AFK
LocalPlayer.Idled:Connect(function()
    VU:CaptureController()
    VU:ClickButton2(Vector2.new())
end)

print("NOVA CHEST V3 - TWEEN & TP CARREGADOS")
