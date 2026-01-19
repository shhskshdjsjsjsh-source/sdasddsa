--[[ 
    NOVA CHEST V3 - FULL FIX
]]

-- Garante que o script rode mesmo se as variáveis externas não existirem
_G.Start_Kaitun = true
if _G.Nova_Loaded then return end
_G.Nova_Loaded = true

if not game:IsLoaded() then game.Loaded:Wait() end

-- // SERVIÇOS
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players, TS, HTTP, Teleport, RunS = Services.Players, Services.TweenService, Services.HttpService, Services.TeleportService, Services.RunService
local Player = Players.LocalPlayer

-- // CONFIGURAÇÕES PADRÃO (Caso a sua config principal falhe)
local Config = {
    ["auto chest tp"] = true,
    ["auto chest twen"] = false,
    ["chest limit"] = 100,
    ["tween speed"] = 150
}

-- Tenta puxar sua config original se ela existir
if _G.Settings and _G.Settings.Main then
    Config = _G.Settings.Main
end

local Internal = {
    Counter = 0,
    Blacklist = {},
    HopLock = false,
    WorldValid = (game.PlaceId == 2753915549 or game.PlaceId == 4442272183 or game.PlaceId == 7449423635)
}

-- // [INTERFACE UI]
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaV3Fix"
ScreenGui.Parent = Player:WaitForChild("PlayerGui")
ScreenGui.ResetOnSpawn = false

-- Janela Principal
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 200, 0, 130)
MainFrame.Position = UDim2.new(0.5, -100, 0.4, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
Title.Text = "NOVA CHEST V3"
Title.TextColor3 = Color3.new(1,1,1)
Title.Font = Enum.Font.SourceSansBold
Title.Parent = MainFrame

local ModeLabel = Instance.new("TextLabel")
ModeLabel.Size = UDim2.new(1, 0, 0, 30)
ModeLabel.Position = UDim2.new(0,0,0,35)
ModeLabel.BackgroundTransparency = 1
ModeLabel.TextColor3 = Color3.new(1,1,1)
ModeLabel.Text = "Status: Ativo"
ModeLabel.Parent = MainFrame

local StopBtn = Instance.new("TextButton")
StopBtn.Size = UDim2.new(0, 180, 0, 35)
StopBtn.Position = UDim2.new(0, 10, 0, 80)
StopBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
StopBtn.Text = "PARAR AUTO CHEST"
StopBtn.TextColor3 = Color3.new(1,1,1)
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.Parent = MainFrame

-- Janela de Informações (Canto)
local InfoFrame = Instance.new("Frame")
InfoFrame.Size = UDim2.new(0, 180, 0, 100)
InfoFrame.Position = UDim2.new(1, -190, 0.8, 0)
InfoFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
InfoFrame.Active = true
InfoFrame.Draggable = true
InfoFrame.Parent = ScreenGui

local InfoText = Instance.new("TextLabel")
InfoText.Size = UDim2.new(1, -10, 1, -10)
InfoText.Position = UDim2.new(0, 5, 0, 5)
InfoText.BackgroundTransparency = 1
InfoText.TextColor3 = Color3.new(1,1,1)
InfoText.TextXAlignment = Enum.TextXAlignment.Left
InfoText.Font = Enum.Font.Code
InfoText.TextSize = 12
InfoText.Parent = InfoFrame

-- Botão Lógica
StopBtn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    StopBtn.Text = _G.Start_Kaitun and "PARAR AUTO CHEST" or "INICIAR AUTO CHEST"
    StopBtn.BackgroundColor3 = _G.Start_Kaitun and Color3.fromRGB(200, 50, 50) or Color3.fromRGB(50, 200, 50)
    ModeLabel.Text = _G.Start_Kaitun and "Status: Ativo" or "Status: Parado"
end)

-- Info Update
task.spawn(function()
    while task.wait(0.5) do
        local fps = math.floor(1/RunS.RenderStepped:Wait())
        local exec = (identifyexecutor or function() return "Desconhecido" end)()
        InfoText.Text = string.format("Executor: %s\nFPS: %d\nVersão: V3\nRoblox: %s", exec, fps, version())
    end
end)

-- // FUNÇÕES DE MOVIMENTO
local function GetHRP() return Player.Character and Player.Character:FindFirstChild("HumanoidRootPart") end

local function Collect(target)
    local hrp = GetHRP()
    if not hrp or not target then return end
    
    if Config["auto chest tp"] then
        hrp.CFrame = target.CFrame
        task.wait(0.1)
    else
        local dist = (hrp.Position - target.Position).Magnitude
        local tween = TS:Create(hrp, TweenInfo.new(dist/Config["tween speed"], Enum.EasingStyle.Linear), {CFrame = target.CFrame})
        tween:Play()
        tween.Completed:Wait()
    end
end

-- // LOOP PRINCIPAL
task.spawn(function()
    while true do
        task.wait(0.1)
        if _G.Start_Kaitun then
            local target = nil
            local dist = math.huge
            
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("TouchTransmitter") and v.Parent and v.Parent.Name:find("Chest") then
                    if not Internal.Blacklist[v.Parent] then
                        local hrp = GetHRP()
                        if hrp then
                            local d = (hrp.Position - v.Parent.Position).Magnitude
                            if d < dist then dist = d; target = v.Parent end
                        end
                    end
                end
            end
            
            if target then
                Collect(target)
                Internal.Blacklist[target] = true
                Internal.Counter = Internal.Counter + 1
            end
        end
    end
end)

-- NoClip
RunS.Stepped:Connect(function()
    if _G.Start_Kaitun and Player.Character then
        for _, v in pairs(Player.Character:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
    end
end)

print("Nova Chest V3 executada!")
