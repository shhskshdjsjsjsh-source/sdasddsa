--[[
    NOVA CHEST V3 - MEGA ROBUST EDITION (FULL INFO VERSION)
    ESTADO: ESTÁVEL / ANTI-CRASH / UI FIX / INFO DISPLAY
    CAPACIDADE: TWEEN & TELEPORT HÍBRIDO
]]

-- // [1. INICIALIZAÇÃO E SEGURANÇA]
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then 
    warn("Nova Chest já está em execução!")
    return 
end
_G.Nova_Loaded = true

-- // [2. SERVIÇOS DO SISTEMA]
local Services = setmetatable({}, {
    __index = function(t, k)
        return game:GetService(k)
    end
})

local Players    = Services.Players
local TS         = Services.TweenService
local HTTP       = Services.HttpService
local Teleport   = Services.TeleportService
local RunS       = Services.RunService
local VU         = Services.VirtualUser
local Stats      = Services.Stats
local LocalPlayer = Players.LocalPlayer

-- // [3. CONFIGURAÇÃO SINCRONIZADA]
local Settings = {
    ChestLimit = 100,
    TweenSpeed = 165,
    AutoTP = false,
    AutoTween = true,
    DistanceToCollect = 20,
    ServerHopEnabled = true
}

local function SyncSettings()
    if _G.Settings and _G.Settings.Main then
        local M = _G.Settings.Main
        Settings.ChestLimit = M["chest limit"] or Settings.ChestLimit
        Settings.TweenSpeed = M["tween speed"] or Settings.TweenSpeed
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
    StartTime = os.time(),
    Version = "V3.0.2 STABLE"
}

-- // [5. FUNÇÕES DE INFORMAÇÃO]
local function GetExecutor()
    return (identifyexecutor or function() return "Unknown" end)()
end

local function GetFPS()
    return math.floor(Stats.WorkspaceRawStats.TotResTime:GetValue()) -- Método alternativo estável
end

-- // [6. CONSTRUÇÃO DA INTERFACE VISUAL]
local ScreenGui = Instance.new("ScreenGui", LocalPlayer:WaitForChild("PlayerGui"))
ScreenGui.Name = "NovaChest_V3_Mega"
ScreenGui.ResetOnSpawn = false

-- Janela Principal (Centro)
local MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 260, 0, 180)
MainFrame.Position = UDim2.new(0.5, -130, 0.15, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(12, 12, 12)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true

local MainCorner = Instance.new("UICorner", MainFrame)
local MainStroke = Instance.new("UIStroke", MainFrame)
MainStroke.Color = Color3.fromRGB(0, 170, 255)
MainStroke.Thickness = 2

local Title = Instance.new("TextLabel", MainFrame)
Title.Size = UDim2.new(1, 0, 0, 35)
Title.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
Title.Text = "NOVA CHEST V3 - MEGA"
Title.TextColor3 = Color3.new(1,1,1)
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 18
Instance.new("UICorner", Title)

-- Labels de Status Central
local StatusLabel = Instance.new("TextLabel", MainFrame)
StatusLabel.Size = UDim2.new(1, 0, 0, 25)
StatusLabel.Position = UDim2.new(0, 0, 0, 40)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Iniciando..."
StatusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
StatusLabel.Font = Enum.Font.SourceSans

local ModeLabel = Instance.new("TextLabel", MainFrame)
ModeLabel.Size = UDim2.new(1, 0, 0, 25)
ModeLabel.Position = UDim2.new(0, 0, 0, 65)
ModeLabel.BackgroundTransparency = 1
ModeLabel.Text = "Modo: Verificando..."
ModeLabel.TextColor3 = Color3.fromRGB(0, 255, 127)
ModeLabel.Font = Enum.Font.SourceSansBold

-- Janela de Informações Técnicas (Canto Inferior Direito)
local InfoFrame = Instance.new("Frame", ScreenGui)
InfoFrame.Size = UDim2.new(0, 220, 0, 130)
InfoFrame.Position = UDim2.new(1, -230, 0.75, 0)
InfoFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
InfoFrame.BackgroundTransparency = 0.2
InfoFrame.Draggable = true

local InfoCorner = Instance.new("UICorner", InfoFrame)
local InfoStroke = Instance.new("UIStroke", InfoFrame)
InfoStroke.Color = Color3.fromRGB(80, 80, 80)

local InfoText = Instance.new("TextLabel", InfoFrame)
InfoText.Size = UDim2.new(1, -20, 1, -10)
InfoText.Position = UDim2.new(0, 10, 0, 5)
InfoText.BackgroundTransparency = 1
InfoText.TextColor3 = Color3.fromRGB(255, 255, 255)
InfoText.TextXAlignment = Enum.TextXAlignment.Left
InfoText.TextYAlignment = Enum.TextYAlignment.Top
InfoText.Font = Enum.Font.Code
InfoText.TextSize = 13

-- Botão Stop
local StopBtn = Instance.new("TextButton", MainFrame)
StopBtn.Size = UDim2.new(0, 230, 0, 40)
StopBtn.Position = UDim2.new(0, 15, 0, 125)
StopBtn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
StopBtn.Text = "PARAR AUTO CHEST"
StopBtn.TextColor3 = Color3.new(1,1,1)
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 16
Instance.new("UICorner", StopBtn)

-- // [7. FUNÇÕES CORE]
local function GetHRP()
    return LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
end

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

-- Movimentação Híbrida Robusta
local function MoveToTarget(target)
    local hrp = GetHRP()
    if not hrp or not target then return end

    if Settings.AutoTP then
        hrp.CFrame = target.CFrame
        task.wait(0.1)
    elseif Settings.AutoTween then
        local dist = (hrp.Position - target.Position).Magnitude
        local duration = dist / Settings.TweenSpeed
        local tween = TS:Create(hrp, TweenInfo.new(duration, Enum.EasingStyle.Linear), {CFrame = target.CFrame})
        
        local completed = false
        local conn = tween.Completed:Connect(function() completed = true end)
        tween:Play()
        
        while not completed and _G.Start_Kaitun do
            hrp.Velocity = Vector3.new(0,0,0) -- Anti-queda
            if not target.Parent then tween:Cancel() break end
            task.wait()
        end
        if conn then conn:Disconnect() end
    end
end

-- // [8. LOOPS DE SINCRONIZAÇÃO E INFO]
task.spawn(function()
    while task.wait(0.2) do
        local fps = math.floor(1/RunS.RenderStepped:Wait())
        InfoText.Text = string.format(
            "EXECUTOR: %s\nFPS: %d\nVERSÃO: %s\nROBLOX: %s\nBAÚS COLETADOS: %d\nTIME: %ds",
            GetExecutor(),
            fps,
            State.Version,
            version():sub(1,10),
            State.Counter,
            os.time() - State.StartTime
        )
        
        SyncSettings()
        
        if Settings.AutoTP then ModeLabel.Text = "MODO: TELEPORT"
        elseif Settings.AutoTween then ModeLabel.Text = "MODO: TWEEN ("..Settings.TweenSpeed..")"
        else ModeLabel.Text = "MODO: AGUARDANDO CONFIG" end
    end
end)

-- // [9. LOOP DE FARM PRINCIPAL]
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

            -- Escaneamento Robusto
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
                StatusLabel.Text = "Status: Indo para Baú..."
                MoveToTarget(targetChest)
                
                if (hrp.Position - targetChest.Position).Magnitude < Settings.DistanceToCollect then
                    State.Blacklist[targetChest] = true
                    State.Counter = State.Counter + 1
                    StatusLabel.Text = "Status: Coletado!"
                end
            else
                StatusLabel.Text = "Status: Servidor Limpo"
                ServerHop()
                break
            end
        end
    end
end)

-- // [10. SISTEMAS DE PROTEÇÃO FINAL]

StopBtn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    StopBtn.Text = _G.Start_Kaitun and "PARAR AUTO CHEST" or "INICIAR AUTO CHEST"
    StopBtn.BackgroundColor3 = _G.Start_Kaitun and Color3.fromRGB(220, 60, 60) or Color3.fromRGB(60, 220, 60)
end)

-- NoClip & Anti-Fall
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

print("---------------------------------------")
print("NOVA CHEST V3 CARREGADO COM SUCESSO")
print("EXECUTOR DETECTADO: " .. GetExecutor())
print("---------------------------------------")

-- Este script contém toda a lógica redundante para evitar crash
-- e o sistema de exibição de informações (InfoFrame) que você solicitou.
