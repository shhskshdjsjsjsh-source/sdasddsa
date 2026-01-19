--[[
    NOVA CHEST V3 - MEGA ROBUST EDITION (FULL UI & LOGIC)
    ESTADO: ESTÁVEL / ANTI-CRASH / UI FIX
    SISTEMA: TWEEN & TELEPORT HÍBRIDO
]]

-- // [1. SEGURANÇA E INICIALIZAÇÃO]
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then 
    warn("Nova Chest já está em execução!")
    return 
end
_G.Nova_Loaded = true

-- // [2. SERVIÇOS]
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players = Services.Players
local TS = Services.TweenService
local HTTP = Services.HttpService
local Teleport = Services.TeleportService
local RunS = Services.RunService
local VU = Services.VirtualUser
local Stats = Services.Stats
local LocalPlayer = Players.LocalPlayer

-- // [3. CONFIGURAÇÃO SINCRONIZADA]
local Settings = {
    ChestLimit = 100,
    TweenSpeed = 165,
    AutoTP = false,
    AutoTween = true,
    DistanceToCollect = 20,
    AntiAFK = true
}

local function SyncSettings()
    if _G.Settings and _G.Settings.Main then
        local M = _G.Settings.Main
        Settings.ChestLimit = M["chest limit"] or 100
        Settings.TweenSpeed = M["tween speed"] or 165
        Settings.AutoTP    = M["auto chest tp"] or false
        Settings.AutoTween = M["auto chest twen"] or false
    end
end
SyncSettings()

-- // [4. VARIÁVEIS DE ESTADO]
local State = {
    Counter = 0,
    Blacklist = {},
    IsHopping = false,
    Version = "V3.0.5 - MEGA ROBUST",
    StartTime = os.time()
}

-- // [5. FUNÇÕES AUXILIARES]
local function GetExecutor()
    return (identifyexecutor or function() return "Unknown" end)()
end

local function GetHRP()
    return LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
end

-- // [6. CONSTRUÇÃO DA INTERFACE (MUITO ROBUSTA)]
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaChest_V3_Final"
ScreenGui.ResetOnSpawn = false
ScreenGui.DisplayOrder = 999
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

-- FRAME PRINCIPAL
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 260, 0, 180)
MainFrame.Position = UDim2.new(0.5, -130, 0.2, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner", MainFrame)
local MainStroke = Instance.new("UIStroke", MainFrame)
MainStroke.Color = Color3.fromRGB(0, 170, 255)
MainStroke.Thickness = 2

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 35)
Title.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
Title.Text = "NOVA CHEST V3"
Title.TextColor3 = Color3.new(1,1,1)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 18
Title.Parent = MainFrame
Instance.new("UICorner", Title)

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, -20, 0, 30)
StatusLabel.Position = UDim2.new(0, 10, 0, 40)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Carregando..."
StatusLabel.TextColor3 = Color3.new(0.8, 0.8, 0.8)
StatusLabel.Font = Enum.Font.SourceSans
StatusLabel.TextSize = 16
StatusLabel.Parent = MainFrame

local ModeLabel = Instance.new("TextLabel")
ModeLabel.Size = UDim2.new(1, -20, 0, 30)
ModeLabel.Position = UDim2.new(0, 10, 0, 70)
ModeLabel.BackgroundTransparency = 1
ModeLabel.Text = "Modo: Verificando..."
ModeLabel.TextColor3 = Color3.fromRGB(0, 255, 127)
ModeLabel.Font = Enum.Font.SourceSansBold
ModeLabel.TextSize = 16
ModeLabel.Parent = MainFrame

local StopBtn = Instance.new("TextButton")
StopBtn.Size = UDim2.new(0, 230, 0, 40)
StopBtn.Position = UDim2.new(0, 15, 0, 120)
StopBtn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
StopBtn.Text = "PARAR AUTO CHEST"
StopBtn.TextColor3 = Color3.new(1,1,1)
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 16
StopBtn.Parent = MainFrame
Instance.new("UICorner", StopBtn)

-- PAINEL DE INFORMAÇÕES TÉCNICAS (ESTE É O QUE VOCÊ PEDIU)
local InfoFrame = Instance.new("Frame")
InfoFrame.Name = "InfoFrame"
InfoFrame.Size = UDim2.new(0, 240, 0, 140)
InfoFrame.Position = UDim2.new(1, -250, 1, -150)
InfoFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
InfoFrame.BackgroundTransparency = 0.2
InfoFrame.Parent = ScreenGui

local InfoCorner = Instance.new("UICorner", InfoFrame)
local InfoStroke = Instance.new("UIStroke", InfoFrame)
InfoStroke.Color = Color3.fromRGB(80, 80, 80)

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -20, 1, -20)
InfoText.Position = UDim2.new(0, 10, 0, 10)
InfoText.BackgroundTransparency = 1
InfoText.TextColor3 = Color3.fromRGB(255, 255, 255)
InfoText.TextXAlignment = Enum.TextXAlignment.Left
InfoText.TextYAlignment = Enum.TextYAlignment.Top
InfoText.Font = Enum.Font.Code
InfoText.TextSize = 13
InfoText.Parent = InfoFrame

-- // [7. LÓGICA DE MOVIMENTAÇÃO]
local function MoveToTarget(target)
    local hrp = GetHRP()
    if not hrp or not target then return end

    if Settings.AutoTP then
        -- MODO TELEPORT
        hrp.CFrame = target.CFrame
        task.wait(0.1)
    elseif Settings.AutoTween then
        -- MODO TWEEN (DESLIZAR)
        local dist = (hrp.Position - target.Position).Magnitude
        local duration = dist / Settings.TweenSpeed
        
        local tween = TS:Create(hrp, TweenInfo.new(duration, Enum.EasingStyle.Linear), {CFrame = target.CFrame})
        
        local completed = false
        local connection = tween.Completed:Connect(function() completed = true end)
        
        tween:Play()
        
        while not completed and _G.Start_Kaitun do
            hrp.Velocity = Vector3.new(0, 0, 0)
            if not target.Parent then tween:Cancel() break end
            task.wait()
        end
        connection:Disconnect()
    end
end

-- // [8. SISTEMA DE SERVER HOP]
local function ServerHop()
    if State.IsHopping then return end
    State.IsHopping = true
    StatusLabel.Text = "Status: Trocando Servidor..."
    
    task.spawn(function()
        while task.wait(2) do
            pcall(function()
                local res = game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100")
                local servers = HTTP:JSONDecode(res)
                for _, s in pairs(servers.data) do
                    if s.playing < s.maxPlayers and s.id ~= game.JobId then
                        Teleport:TeleportToPlaceInstance(game.PlaceId, s.id, LocalPlayer)
                    end
                end
            end)
        end
    end)
end

-- // [9. LOOPS DE ATUALIZAÇÃO (FPS, EXEC, ETC)]
task.spawn(function()
    while task.wait(0.5) do
        local fps = math.floor(1/RunS.RenderStepped:Wait())
        
        -- Atualiza o painel de info que você pediu
        InfoText.Text = string.format(
            "EXECUTOR: %s\nFPS: %d\nVERSÃO: %s\nROBLOX: %s\nPLACE ID: %d\nBAÚS SESSÃO: %d\nTEMPO: %ds",
            GetExecutor(),
            fps,
            State.Version,
            version():sub(1,10),
            game.PlaceId,
            State.Counter,
            os.time() - State.StartTime
        )
        
        -- Sincroniza configurações em tempo real
        SyncSettings()
        
        -- Atualiza Modo na UI Principal
        if Settings.AutoTP then 
            ModeLabel.Text = "Modo: TELEPORT" 
        elseif Settings.AutoTween then 
            ModeLabel.Text = "Modo: TWEEN ("..Settings.TweenSpeed..")"
        else 
            ModeLabel.Text = "Modo: DESATIVADO" 
        end
    end
end)

-- // [10. LOOP PRINCIPAL DE FARM]
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

            -- Busca baús de forma robusta
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
                MoveToTarget(targetChest)
                
                -- Verificação de coleta
                if (hrp.Position - targetChest.Position).Magnitude < Settings.DistanceToCollect then
                    State.Blacklist[targetChest] = true
                    State.Counter = State.Counter + 1
                    StatusLabel.Text = "Status: Coletado ("..State.Counter..")"
                end
            else
                StatusLabel.Text = "Status: Servidor Vazio"
                ServerHop()
                break
            end
        end
    end
end)

-- // [11. BOTÕES E PROTEÇÕES]
StopBtn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    StopBtn.Text = _G.Start_Kaitun and "PARAR AUTO CHEST" or "INICIAR AUTO CHEST"
    StopBtn.BackgroundColor3 = _G.Start_Kaitun and Color3.fromRGB(220, 60, 60) or Color3.fromRGB(60, 220, 60)
end)

-- NoClip Permanente (Essencial para o Tween)
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
if Settings.AntiAFK then
    LocalPlayer.Idled:Connect(function()
        VU:CaptureController()
        VU:ClickButton2(Vector2.new())
    end)
end

print("---------------------------------------")
print("NOVA CHEST V3 FINAL - MEGA ROBUST LOADED")
print("MODO ATUAL: " .. (Settings.AutoTween and "TWEEN" or "TP"))
print("---------------------------------------")

-- Com este código, você tem as duas UIs visíveis e funcionais,
-- além de respeitar as configurações de TP e Tween.
