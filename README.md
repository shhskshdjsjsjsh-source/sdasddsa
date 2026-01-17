--[[ 
    NOVA CHEST V2 - FIX UI VERSION
]]

if not _G.Start_Kaitun then return end
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then return end
_G.Nova_Loaded = true

-- // SERVIÇOS
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players, TS, HTTP, Teleport, RunS, VU, SG = Services.Players, Services.TweenService, Services.HttpService, Services.TeleportService, Services.RunService, Services.VirtualUser, Services.StarterGui
local Player = Players.LocalPlayer
local Config = _G.Settings.Main

-- // VARIÁVEIS DE ESTADO
local Internal = {
    Counter = 0,
    Blacklist = {},
    CurrentTarget = nil,
    HopLock = false,
    WorldValid = (game.PlaceId == 2753915549 or game.PlaceId == 4442272183 or game.PlaceId == 7449423635)
}

-- // [SISTEMA DE UI - REFEITO PARA COMPATIBILIDADE]
local PlayerGui = Player:WaitForChild("PlayerGui")
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaChestFix"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 200, 0, 100)
MainFrame.Position = UDim2.new(0.5, -100, 0.2, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
MainFrame.BorderSizePixel = 2
MainFrame.BorderColor3 = Color3.fromRGB(0, 170, 255)
MainFrame.Active = true
MainFrame.Draggable = true -- Você pode arrastar a UI agora
MainFrame.Parent = ScreenGui

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 25)
Title.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Text = "NOVA CHEST V2"
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 16
Title.Parent = MainFrame

local ModeLabel = Instance.new("TextLabel")
ModeLabel.Size = UDim2.new(1, 0, 0, 25)
ModeLabel.Position = UDim2.new(0, 0, 0, 30)
ModeLabel.BackgroundTransparency = 1
ModeLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
ModeLabel.Text = "Modo: Verificando..."
ModeLabel.Font = Enum.Font.SourceSans
ModeLabel.TextSize = 14
ModeLabel.Parent = MainFrame

local CountLabel = Instance.new("TextLabel")
CountLabel.Size = UDim2.new(1, 0, 0, 25)
CountLabel.Position = UDim2.new(0, 0, 0, 55)
CountLabel.BackgroundTransparency = 1
CountLabel.TextColor3 = Color3.fromRGB(0, 255, 127)
CountLabel.Text = "Faltam: --"
CountLabel.Font = Enum.Font.SourceSans
CountLabel.TextSize = 14
CountLabel.Parent = MainFrame

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, 0, 0, 20)
StatusLabel.Position = UDim2.new(0, 0, 0, 80)
StatusLabel.BackgroundTransparency = 1
StatusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
StatusLabel.TextSize = 12
StatusLabel.Text = "Status: OK"
StatusLabel.Parent = MainFrame

-- // [FUNÇÕES AUXILIARES]
local function UpdateUI()
    local modo = "Nenhum"
    if Config["auto chest tp"] then modo = "TELEPORT (TP)"
    elseif Config["auto chest twen"] then modo = "TWEEN (Deslizar)" end
    
    local faltam = math.max(0, Config["chest limit"] - Internal.Counter)
    
    ModeLabel.Text = "Modo: " .. modo
    CountLabel.Text = "Faltam: " .. faltam .. " baús para Hop"
    
    if not Internal.WorldValid then
        StatusLabel.Text = "Mundo Inválido para Server Hop"
        StatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
    end
end

local function GetHRP() return Player.Character and Player.Character:FindFirstChild("HumanoidRootPart") end

-- // [LOGICA DE MOVIMENTO]
local function ExecuteTween(target)
    local hrp = GetHRP()
    if not hrp or not target then return end
    local dist = (hrp.Position - target.Position).Magnitude
    local tTime = dist / Config["tween speed"]
    local tween = TS:Create(hrp, TweenInfo.new(tTime, Enum.EasingStyle.Linear), {CFrame = target.CFrame})
    local finished = false
    tween.Completed:Connect(function() finished = true end)
    tween:Play()
    while not finished and _G.Start_Kaitun do
        if not target.Parent or not Config["auto chest twen"] then tween:Cancel() break end
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
    if not Internal.WorldValid then return end
    if Internal.HopLock then return end
    Internal.HopLock = true
    StatusLabel.Text = "Trocando de Servidor..."
    
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
    while _G.Start_Kaitun do
        UpdateUI()
        task.wait(0.5)
        
        if Internal.Counter >= Config["chest limit"] then
            PerformServerHop()
            break
        end

        local nearest = nil
        local min_dist = math.huge
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("TouchTransmitter") and v.Parent and (v.Parent.Name:find("Chest") or v.Parent.Name:find("Baú")) then
                if not Internal.Blacklist[v.Parent] then
                    local d = (GetHRP().Position - v.Parent.Position).Magnitude
                    if d < min_dist then min_dist = d; nearest = v.Parent end
                end
            end
        end

        if nearest then
            if Config["auto chest tp"] then ExecuteTP(nearest)
            elseif Config["auto chest twen"] then ExecuteTween(nearest) end
            
            if (GetHRP().Position - nearest.Position).Magnitude < 25 then
                Internal.Blacklist[nearest] = true
                Internal.Counter = Internal.Counter + 1
            end
        else
            PerformServerHop()
            break
        end
    end
end)

-- Anti AFK e NoClip
RunS.Stepped:Connect(function()
    if _G.Start_Kaitun and Player.Character then
        for _, v in pairs(Player.Character:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
    end
end)

print("Nova Chest Engine v2 com UI Estável Carregada.")
