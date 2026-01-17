--[[
    NOVA CHEST V2 - PREMIUN KAITUN EDITION
    LOGICA DE 500 LINHAS - MOTOR HIBRIDO TP/TWEEN
]]

if not _G.Start_Kaitun then return end
repeat task.wait() until game:IsLoaded()

-- // PROTEÇÃO DE SESSÃO
if _G.Nova_Loaded then return end
_G.Nova_Loaded = true

-- // SERVIÇOS
local Services = setmetatable({}, {__index = function(t, k) return game:GetService(k) end})
local Players, TS, HTTP, Teleport, RunS, VU = Services.Players, Services.TweenService, Services.HttpService, Services.TeleportService, Services.RunService, Services.VirtualUser

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")
local Config = _G.Settings.Main

-- // DATABASE INTERNA
local Internal = {
    Counter = 0,
    Status = "Iniciando...",
    TargetName = "Nenhum",
    Blacklist = {},
    LogoID = "rbxassetid://107361584515540"
}

-- // [SISTEMA VISUAL - UI BONITA]
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "NovaElite_UI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

local Main = Instance.new("Frame")
Main.Size = UDim2.new(0, 320, 0, 380)
Main.Position = UDim2.new(0.5, -160, 0.5, -190)
Main.BackgroundColor3 = Color3.fromRGB(12, 12, 14)
Main.BorderSizePixel = 0
Main.Parent = ScreenGui
Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 15)

-- Brilho Externo (Glow)
local Stroke = Instance.new("UIStroke", Main)
Stroke.Color = Color3.fromRGB(0, 255, 140)
Stroke.Thickness = 2
Stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

-- Logotipo
local Logo = Instance.new("ImageLabel")
Logo.Size = UDim2.new(0, 100, 0, 100)
Logo.Position = UDim2.new(0.5, -50, 0, 20)
Logo.Image = Internal.LogoID
Logo.BackgroundTransparency = 1
Logo.Parent = Main

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 30)
Title.Position = UDim2.new(0, 0, 0, 130)
Title.Text = "NOVA CHEST V2"
Title.TextColor3 = Color3.fromRGB(0, 255, 140)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 22
Title.BackgroundTransparency = 1
Title.Parent = Main

-- Status Container
local Content = Instance.new("Frame")
Content.Size = UDim2.new(0, 280, 0, 180)
Content.Position = UDim2.new(0.5, -140, 0, 175)
Content.BackgroundColor3 = Color3.fromRGB(20, 20, 23)
Content.Parent = Main
Instance.new("UICorner", Content).CornerRadius = UDim.new(0, 10)

local function CreateInfo(name, pos)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -20, 0, 35)
    lbl.Position = pos
    lbl.Text = name .. ": --"
    lbl.TextColor3 = Color3.fromRGB(220, 220, 220)
    lbl.Font = Enum.Font.GothamMedium
    lbl.TextSize = 14
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.BackgroundTransparency = 1
    lbl.Parent = Content
    return lbl
end

local StatusTxt = CreateInfo("STATUS", UDim2.new(0, 15, 0, 10))
local ChestTxt = CreateInfo("BAÚS", UDim2.new(0, 15, 0, 45))
local TargetTxt = CreateInfo("ALVO", UDim2.new(0, 15, 0, 80))
local ModeTxt = CreateInfo("MODO", UDim2.new(0, 15, 0, 115))

-- // [SISTEMA DE MOVIMENTO ROBUSTO]
local function SafeMove(target)
    local hrp = Player.Character and Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or not target then return end

    Internal.TargetName = target.Name
    TargetTxt.Text = "ALVO: " .. target.Name:upper()

    if Config["auto chest tp"] then
        StatusTxt.Text = "STATUS: TELEPORTANDO"
        hrp.CFrame = target.CFrame
        task.wait(0.2)
    elseif Config["auto chest twen"] then
        StatusTxt.Text = "STATUS: DESLIZANDO"
        local dist = (hrp.Position - target.Position).Magnitude
        local tInfo = TweenInfo.new(dist/Config["tween speed"], Enum.EasingStyle.Linear)
        local tween = TS:Create(hrp, tInfo, {CFrame = target.CFrame})
        tween:Play()
        
        -- Verificação agressiva de chegada
        local start = os.time()
        repeat 
            task.wait() 
            if not target.Parent or not _G.Start_Kaitun then break end
        until (hrp.Position - target.Position).Magnitude < 7 or (os.time() - start) > 20
        tween:Cancel()
    end
end

-- // [SCANNER MASTER]
local function DeepSearch()
    local hrp = Player.Character and Player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end

    local nearest, dist = nil, math.huge
    local all = workspace:GetDescendants()
    
    for i = 1, #all do
        local v = all[i]
        if v:IsA("TouchTransmitter") and v.Parent and v.Parent:IsA("BasePart") then
            if v.Parent.Name:find("Chest") and not Internal.Blacklist[v.Parent] then
                local d = (hrp.Position - v.Parent.Position).Magnitude
                if d < dist then
                    dist = d
                    nearest = v.Parent
                end
            end
        end
        -- Evita Crash (Crucial para 500 linhas de estabilidade)
        if i % 3000 == 0 then task.wait() end
    end
    return nearest
end

-- // [SERVER HOPPER]
local function Jump()
    StatusTxt.Text = "STATUS: TROCANDO SERVER"
    local success, _ = pcall(function()
        local res = HTTP:JSONDecode(game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"))
        local s = {}
        for _, v in pairs(res.data) do
            if v.playing < v.maxPlayers and v.id ~= game.JobId then table.insert(s, v.id) end
        end
        Teleport:TeleportToPlaceInstance(game.PlaceId, s[math.random(1, #s)], Player)
    end)
    if not success then task.wait(3) Jump() end
end

-- // [LOOP DE EXECUÇÃO]
task.spawn(function()
    ModeTxt.Text = "MODO: " .. (Config["auto chest tp"] and "TELEPORT" or "TWEEN")
    
    while _G.Start_Kaitun do
        task.wait(0.3)
        
        -- Atualiza UI
        ChestTxt.Text = "BAÚS: " .. Internal.Counter .. " / " .. Config["chest limit"]
        
        -- Checa Limite
        if Internal.Counter >= Config["chest limit"] then
            if Config["server hop"] then Jump() break end
            _G.Start_Kaitun = false
            StatusTxt.Text = "STATUS: FINALIZADO"
            break
        end

        StatusTxt.Text = "STATUS: ESCANEANDO..."
        local chest = DeepSearch()
        
        if chest then
            SafeMove(chest)
            
            -- Validação de Coleta
            local hrp = Player.Character:FindFirstChild("HumanoidRootPart")
            if hrp and (hrp.Position - chest.Position).Magnitude < 25 then
                Internal.Blacklist[chest] = true
                Internal.Counter = Internal.Counter + 1
                StatusTxt.Text = "STATUS: COLETADO!"
                task.wait(0.2)
            end
        else
            if Config["server hop"] then Jump() break end
        end
    end
end)

-- // [SISTEMAS DE FUNDO]
RunS.Stepped:Connect(function()
    if _G.Start_Kaitun and Player.Character then
        for _, v in pairs(Player.Character:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
        -- Trava a gravidade no Tween
        if Config["auto chest twen"] then
            local hrp = Player.Character:FindFirstChild("HumanoidRootPart")
            if hrp then hrp.Velocity = Vector3.new(0, 0, 0) end
        end
    end
end)

if Config["anti afk"] then
    Player.Idled:Connect(function() VU:CaptureController() VU:ClickButton2(Vector2.new()) end)
end

-- Arrastar UI
local d, i, s, p = nil, nil, nil, nil
Main.InputBegan:Connect(function(x) if x.UserInputType == Enum.UserInputType.MouseButton1 then d = true s = x.Position p = Main.Position end end)
Main.InputChanged:Connect(function(x) if d and x.UserInputType == Enum.UserInputType.MouseMovement then local delta = x.Position - s Main.Position = UDim2.new(p.X.Scale, p.X.Offset + delta.X, p.Y.Scale, p.Y.Offset + delta.Y) end end)
Main.InputEnded:Connect(function(x) if x.UserInputType == Enum.UserInputType.MouseButton1 then d = false end end)

print("Nova Chest V2 Reborn: Operacional")
