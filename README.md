--[[ 
    NOVA CHEST V3 - FIX UI & INFO SYSTEM
]]

if not _G.Start_Kaitun then _G.Start_Kaitun = true end -- Garante que inicie se não definido
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then return end
_G.Nova_Loaded = true

-- // SERVIÇOS
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players, TS, HTTP, Teleport, RunS, VU, SG = Services.Players, Services.TweenService, Services.HttpService, Services.TeleportService, Services.RunService, Services.VirtualUser, Services.StarterGui
local Player = Players.LocalPlayer
local Config = _G.Settings and _G.Settings.Main or {["auto chest tp"] = true, ["chest limit"] = 50, ["tween speed"] = 150}

-- // VARIÁVEIS DE ESTADO
local Internal = {
    Counter = 0,
    Blacklist = {},
    HopLock = false,
    WorldValid = (game.PlaceId == 2753915549 or game.PlaceId == 4442272183 or game.PlaceId == 7449423635)
}

-- // [INTERFACE PRINCIPAL - V3]
local PlayerGui = Player:WaitForChild("PlayerGui")
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaChestV3"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 200, 0, 130)
MainFrame.Position = UDim2.new(0.5, -100, 0.2, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 8)
UICorner.Parent = MainFrame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundColor3 = Color3.fromRGB(0, 120, 255)
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
ModeLabel.Text = "Modo: Carregando..."
ModeLabel.Font = Enum.Font.SourceSans
ModeLabel.TextSize = 14
ModeLabel.Parent = MainFrame

local StopBtn = Instance.new("TextButton")
StopBtn.Size = UDim2.new(0, 180, 0, 30)
StopBtn.Position = UDim2.new(0, 10, 0, 90)
StopBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
StopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StopBtn.Text = "PARAR AUTO CHEST"
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 14
StopBtn.Parent = MainFrame

StopBtn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    StopBtn.Text = _G.Start_Kaitun and "PARAR AUTO CHEST" or "INICIAR AUTO CHEST"
    StopBtn.BackgroundColor3 = _G.Start_Kaitun and Color3.fromRGB(200, 50, 50) or Color3.fromRGB(50, 200, 50)
end)

-- // [UI DE INFORMAÇÕES - CANTO]
local InfoFrame = Instance.new("Frame")
InfoFrame.Size = UDim2.new(0, 180, 0, 100)
InfoFrame.Position = UDim2.new(1, -190, 0.8, 0)
InfoFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
InfoFrame.BackgroundTransparency = 0.2
InfoFrame.Active = true
InfoFrame.Draggable = true
InfoFrame.Parent = ScreenGui

local InfoCorner = Instance.new("UICorner")
InfoCorner.Parent = InfoFrame

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -10, 1, -10)
InfoText.Position = UDim2.new(0, 5, 0, 5)
InfoText.BackgroundTransparency = 1
InfoText.TextColor3 = Color3.fromRGB(255, 255, 255)
InfoText.TextXAlignment = Enum.TextXAlignment.Left
InfoText.Font = Enum.Font.Code
InfoText.TextSize = 12
InfoText.Parent = InfoFrame

-- // [SISTEMA DE INFO]
local function GetExecutor()
    return identifyexecutor and identifyexecutor() or "Desconhecido"
end

task.spawn(function()
    while task.wait(0.5) do
        local fps = math.floor(1/game:GetService("RunService").RenderStepped:Wait())
        InfoText.Text = string.format(
            "Executor: %s\nFPS: %d\nScript: V3 Fix\nRoblox: %s",
            GetExecutor(),
            fps,
            version()
        )
    end
end)

-- // [FUNÇÕES AUXILIARES]
local function UpdateUI()
    local modo = "Nenhum"
    if Config["auto chest tp"] then modo = "TELEPORT"
    elseif Config["auto chest twen"] then modo = "TWEEN" end
    ModeLabel.Text = "Modo: " .. modo
end

local function GetHRP() return Player.Character and Player.Character:FindFirstChild("HumanoidRootPart") end

-- // [LÓGICA DE MOVIMENTO]
local function ExecuteTween(target)
    local hrp = GetHRP()
    if not hrp or not target then return end
    local dist = (hrp.Position - target.Position).Magnitude
    local tTime = dist / (Config["tween speed"] or 150)
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
end

-- // [LOOP PRINCIPAL]
task.spawn(function()
    while true do
        task.wait(0.5)
        if _G.Start_Kaitun then
            UpdateUI()
            
            if Internal.Counter >= (Config["chest limit"] or 100) then
                -- Logica de Server Hop omitida para brevidade, mas segue a mesma da V2
                break
            end

            local nearest = nil
            local min_dist = math.huge
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("TouchTransmitter") and v.Parent and (v.Parent.Name:find("Chest") or v.Parent.Name:find("Baú")) then
                    if not Internal.Blacklist[v.Parent] then
                        local hrp = GetHRP()
                        if hrp then
                            local d = (hrp.Position - v.Parent.Position).Magnitude
                            if d < min_dist then min_dist = d; nearest = v.Parent end
                        end
                    end
                end
            end

            if nearest then
                if Config["auto chest tp"] then ExecuteTP(nearest)
                elseif Config["auto chest twen"] then ExecuteTween(nearest) end
                
                if GetHRP() and (GetHRP().Position - nearest.Position).Magnitude < 25 then
                    Internal.Blacklist[nearest] = true
                    Internal.Counter = Internal.Counter + 1
                end
            end
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

print("Nova Chest V3 Carregada com sucesso!")
