--[[ 
    NOVA CHEST V2 - CORE ENGINE + UI EDITION
    Customizada com Interface e Verificação de Mundo
]]

if not _G.Start_Kaitun then return end
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then return end
_G.Nova_Loaded = true

-- // SERVIÇOS
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players, TS, RS, HTTP, Teleport, RunS, VU, SG = Services.Players, Services.TweenService, Services.ReplicatedStorage, Services.HttpService, Services.TeleportService, Services.RunService, Services.VirtualUser, Services.StarterGui
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

-- // [SISTEMA DE UI]
local ScreenGui = Instance.new("ScreenGui", Player:FindFirstChildOfClass("PlayerGui"))
ScreenGui.Name = "NovaChestUI"

local MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 220, 0, 110)
MainFrame.Position = UDim2.new(0.5, -110, 0.1, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
MainFrame.BorderSizePixel = 0

local UICorner = Instance.new("UICorner", MainFrame)
local UIStroke = Instance.new("UIStroke", MainFrame)
UIStroke.Color = Color3.fromRGB(0, 170, 255)
UIStroke.Thickness = 2

local Title = Instance.new("TextLabel", MainFrame)
Title.Text = "NOVA CHEST V2"
Title.Size = UDim2.new(1, 0, 0, 30)
Title.TextColor3 = Color3.fromRGB(0, 170, 255)
Title.BackgroundTransparency = 1
Title.Font = Enum.Font.GothamBold
Title.TextSize = 14

local ModeLabel = Instance.new("TextLabel", MainFrame)
ModeLabel.Text = "Modo: Aguardando..."
ModeLabel.Position = UDim2.new(0, 10, 0, 35)
ModeLabel.Size = UDim2.new(1, -20, 0, 20)
ModeLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
ModeLabel.BackgroundTransparency = 1
ModeLabel.TextXAlignment = Enum.TextXAlignment.Left
ModeLabel.Font = Enum.Font.Gotham
ModeLabel.TextSize = 12

local CountLabel = Instance.new("TextLabel", MainFrame)
CountLabel.Text = "Faltam: --"
CountLabel.Position = UDim2.new(0, 10, 0, 55)
CountLabel.Size = UDim2.new(1, -20, 0, 20)
CountLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
CountLabel.BackgroundTransparency = 1
CountLabel.TextXAlignment = Enum.TextXAlignment.Left
CountLabel.Font = Enum.Font.Gotham
CountLabel.TextSize = 12

local StatusLabel = Instance.new("TextLabel", MainFrame)
StatusLabel.Text = "Status: Ativo"
StatusLabel.Position = UDim2.new(0, 10, 0, 75)
StatusLabel.Size = UDim2.new(1, -20, 0, 20)
StatusLabel.TextColor3 = Color3.fromRGB(150, 150, 150)
StatusLabel.BackgroundTransparency = 1
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextSize = 11

-- // [FUNÇÕES DE ATUALIZAÇÃO]
local function UpdateUI()
    local modo = "Nenhum"
    if Config["auto chest tp"] then modo = "TELEPORT (Instante)"
    elseif Config["auto chest twen"] then modo = "TWEEN (Deslizar)" end
    
    local faltam = math.max(0, Config["chest limit"] - Internal.Counter)
    
    ModeLabel.Text = "Modo: " .. modo
    CountLabel.Text = "Restantes p/ Hop: " .. faltam
    
    if not Internal.WorldValid then
        StatusLabel.Text = "Status: Mundo Inválido p/ Hop"
        StatusLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
    end
end

-- // [SISTEMA DE MOVIMENTO]
local function GetHRP() return Player.Character and Player.Character:FindFirstChild("HumanoidRootPart") end

local function ExecuteTween(target)
    local hrp = GetHRP()
    if not hrp or not target then return end
    local dist = (hrp.Position - target.Position).Magnitude
    local tTime = dist / Config["tween speed"]
    local tween = TS:Create(hrp, TweenInfo.new(tTime, Enum.EasingStyle.Linear), {CFrame = target.CFrame})
    
    local finished = false
    local connection = tween.Completed:Connect(function() finished = true end)
    tween:Play()
    while not finished and _G.Start_Kaitun do
        if not target.Parent or not Config["auto chest twen"] then tween:Cancel() break end
        hrp.Velocity = Vector3.new(0, 0.05, 0)
        task.wait()
    end
    connection:Disconnect()
end

local function ExecuteTP(target)
    local hrp = GetHRP()
    if not hrp or not target then return end
    hrp.CFrame = target.CFrame + Vector3.new(0, 2, 0)
    task.wait(0.1)
    hrp.CFrame = target.CFrame
    task.wait(0.1)
end

-- // [SERVER HOP COM VERIFICAÇÃO DE MUNDO]
local function PerformServerHop()
    if not Internal.WorldValid then 
        print("Nova Chest: Hop cancelado - Mapa não suportado.")
        return 
    end
    if Internal.HopLock then return end
    Internal.HopLock = true
    StatusLabel.Text = "Status: Trocando Servidor..."
    
    pcall(function()
        local url = "https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"
        local body = HTTP:JSONDecode(game:HttpGet(url))
        if body and body.data then
            for _, server in pairs(body.data) do
                if server.playing < server.maxPlayers and server.id ~= game.JobId then
                    Teleport:TeleportToPlaceInstance(game.PlaceId, server.id, Player)
                    return
                end
            end
        end
        Teleport:Teleport(game.PlaceId, Player)
    end)
end

local function FindNearestChest()
    local hrp = GetHRP()
    if not hrp then return nil end
    local nearest, min_dist = nil, math.huge
    for _, v in pairs(workspace:GetDescendants()) do
        if v:IsA("TouchTransmitter") and v.Parent then
            local obj = v.Parent
            if (obj.Name:find("Chest") or obj.Name:find("Baú")) and not Internal.Blacklist[obj] then
                local d = (hrp.Position - obj.Position).Magnitude
                if d < min_dist then min_dist = d; nearest = obj end
            end
        end
    end
    return nearest
end

-- // [CORE LOOP]
task.spawn(function()
    while _G.Start_Kaitun do
        UpdateUI()
        task.wait(0.5)
        
        if Internal.Counter >= Config["chest limit"] then
            PerformServerHop()
            break
        end

        local chest = FindNearestChest()
        if chest then
            if Config["auto chest tp"] then ExecuteTP(chest)
            elseif Config["auto chest twen"] then ExecuteTween(chest) end
            
            local hrp = GetHRP()
            if hrp and (hrp.Position - chest.Position).Magnitude < 25 then
                Internal.Blacklist[chest] = true
                Internal.Counter = Internal.Counter + 1
                task.wait(0.1)
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

if Config["anti afk"] then
    Player.Idled:Connect(function() VU:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame); task.wait(1); VU:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame) end)
end

SG:SetCore("SendNotification", {Title = "Nova Chest V2", Text = "Interface Carregada!", Duration = 5})
