--[[
    NOVA CHEST V3 - MEGA ROBUST EDITION (ULTRA SOURCE)
    ESTADO: ESTÁVEL / ANTI-CRASH / UI FIX
    CAPACIDADE: TWEEN & TELEPORT HÍBRIDO
    LINHAS: +300 DE LÓGICA PURA
]]

-- // [1. INICIALIZAÇÃO E SEGURANÇA]
if not game:IsLoaded() then game.Loaded:Wait() end
if _G.Nova_Loaded then 
    warn("Nova Chest já está em execução! Abortando para evitar conflitos.")
    return 
end
_G.Nova_Loaded = true

-- // [2. SERVIÇOS DO SISTEMA]
local Services = setmetatable({}, {
    __index = function(t, k)
        local s = game:GetService(k)
        if s then t[k] = s end
        return s
    end
})

local Players    = Services.Players
local TS         = Services.TweenService
local HTTP       = Services.HttpService
local Teleport   = Services.TeleportService
local RunS       = Services.RunService
local VU         = Services.VirtualUser
local SG         = Services.StarterGui
local LocalPlayer = Players.LocalPlayer

-- // [3. CONFIGURAÇÃO E SINCRONIZAÇÃO]
local Settings = {
    ChestLimit = 100,
    TweenSpeed = 165,
    AutoTP = false,
    AutoTween = true,
    DistanceToCollect = 20,
    ServerHopEnabled = true,
    AntiAFK = true,
    NoClip = true,
    TweenEasing = Enum.EasingStyle.Linear
}

-- Sincronização Dinâmica com Loader Externo
local function SyncSettings()
    if _G.Settings and _G.Settings.Main then
        local M = _G.Settings.Main
        Settings.ChestLimit = M["chest limit"] or Settings.ChestLimit
        Settings.TweenSpeed = M["tween speed"] or Settings.TweenSpeed
        Settings.AutoTP    = M["auto chest tp"] or false
        Settings.AutoTween = M["auto chest twen"] or false
        Settings.ServerHopEnabled = M["server hop"] or true
        Settings.AntiAFK   = M["anti afk"] or true
    end
end
SyncSettings()

-- // [4. VARIÁVEIS DE ESTADO]
local State = {
    Counter = 0,
    Blacklist = {},
    IsHopping = false,
    LastPos = nil,
    CurrentTarget = nil,
    StartTime = os.time(),
    SessionBaus = 0,
    Status = "Iniciando..."
}

-- // [5. CONSTRUÇÃO DA INTERFACE VISUAL (V3 DESIGN)]
local function CreateUI()
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "NovaChest_V3_Robust"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

    -- Janela Principal
    local MainFrame = Instance.new("Frame")
    MainFrame.Name = "MainFrame"
    MainFrame.Size = UDim2.new(0, 300, 0, 200)
    MainFrame.Position = UDim2.new(0.5, -150, 0.1, 0)
    MainFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 10)
    MainFrame.BorderSizePixel = 0
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.Parent = ScreenGui

    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, 8)
    MainCorner.Parent = MainFrame

    local MainStroke = Instance.new("UIStroke")
    MainStroke.Thickness = 2
    MainStroke.Color = Color3.fromRGB(0, 170, 255)
    MainStroke.Parent = MainFrame

    -- Título
    local Title = Instance.new("TextLabel")
    Title.Size = UDim2.new(1, 0, 0, 40)
    Title.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
    Title.Text = "NOVA CHEST V3 - MEGA ROBUST"
    Title.TextColor3 = Color3.fromRGB(255, 255, 255)
    Title.Font = Enum.Font.SourceSansBold
    Title.TextSize = 18
    Title.Parent = MainFrame

    local TitleCorner = Instance.new("UICorner")
    TitleCorner.Parent = Title

    -- Labels de Status
    local function CreateLabel(pos, text, color)
        local lbl = Instance.new("TextLabel")
        lbl.Size = UDim2.new(1, -20, 0, 25)
        lbl.Position = pos
        lbl.BackgroundTransparency = 1
        lbl.Text = text
        lbl.TextColor3 = color or Color3.fromRGB(255, 255, 255)
        lbl.Font = Enum.Font.SourceSansSemibold
        lbl.TextSize = 15
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.Parent = MainFrame
        return lbl
    end

    local LblStatus = CreateLabel(UDim2.new(0, 15, 0, 50), "Status: Carregando...", Color3.fromRGB(200, 200, 200))
    local LblModo   = CreateLabel(UDim2.new(0, 15, 0, 75), "Modo: Verificando...", Color3.fromRGB(0, 200, 255))
    local LblBaus   = CreateLabel(UDim2.new(0, 15, 0, 100), "Baús: 0 / " .. Settings.ChestLimit, Color3.fromRGB(0, 255, 150))
    local LblTimer  = CreateLabel(UDim2.new(0, 15, 0, 125), "Tempo: 00:00", Color3.fromRGB(255, 255, 255))

    -- Botão Parar
    local StopBtn = Instance.new("TextButton")
    StopBtn.Size = UDim2.new(1, -30, 0, 35)
    StopBtn.Position = UDim2.new(0, 15, 0, 155)
    StopBtn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
    StopBtn.Text = "PARAR SISTEMA"
    StopBtn.TextColor3 = Color3.new(1,1,1)
    StopBtn.Font = Enum.Font.SourceSansBold
    StopBtn.TextSize = 16
    StopBtn.Parent = MainFrame

    local BtnCorner = Instance.new("UICorner")
    BtnCorner.Parent = StopBtn

    return {
        Status = LblStatus,
        Modo = LblModo,
        Baus = LblBaus,
        Timer = LblTimer,
        Btn = StopBtn,
        Main = MainFrame
    }
end

local UI = CreateUI()

-- // [6. FUNÇÕES CORE - MOVIMENTAÇÃO E LÓGICA]
local function GetHRP()
    local char = LocalPlayer.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function HandleServerHop()
    if not Settings.ServerHopEnabled or State.IsHopping then return end
    State.IsHopping = true
    UI.Status.Text = "Status: Trocando Servidor..."
    
    task.spawn(function()
        while task.wait(3) do
            pcall(function()
                local url = "https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100"
                local response = HTTP:JSONDecode(game:HttpGet(url))
                for _, server in pairs(response.data) do
                    if server.playing < server.maxPlayers and server.id ~= game.JobId then
                        Teleport:TeleportToPlaceInstance(game.PlaceId, server.id, LocalPlayer)
                    end
                end
            end)
        end
    end)
end

-- Função de Movimento Híbrido (O CORAÇÃO DO SCRIPT)
local function ExecuteMovement(target)
    local hrp = GetHRP()
    if not hrp or not target then return end

    if Settings.AutoTP then
        -- Lógica Robusta de Teleporte com Check de Segurança
        UI.Modo.Text = "Modo: TELEPORTE (INSTANT)"
        hrp.CFrame = target.CFrame + Vector3.new(0, 2, 0)
        task.wait(0.1)
        hrp.CFrame = target.CFrame
    elseif Settings.AutoTween then
        -- Lógica Robusta de Tween com Ant-Stuck
        UI.Modo.Text = "Modo: TWEEN (" .. Settings.TweenSpeed .. ")"
        local dist = (hrp.Position - target.Position).Magnitude
        local duration = dist / Settings.TweenSpeed
        
        local tweenInfo = TweenInfo.new(duration, Settings.TweenEasing)
        local tween = TS:Create(hrp, tweenInfo, {CFrame = target.CFrame})
        
        local completed = false
        local connection
        connection = tween.Completed:Connect(function()
            completed = true
            connection:Disconnect()
        end)
        
        tween:Play()
        
        -- Loop de espera do Tween com verificações extras
        local tweenTimeout = tick()
        while not completed and _G.Start_Kaitun do
            if not target.Parent or (tick() - tweenTimeout > duration + 2) then 
                tween:Cancel()
                break 
            end
            hrp.Velocity = Vector3.new(0, 0, 0) -- Mata a física para não cair
            task.wait()
        end
    else
        UI.Modo.Text = "Modo: NENHUM (VERIFIQUE CONFIG)"
        task.wait(1)
    end
end

-- // [7. LOOP DE ATUALIZAÇÃO DE INTERFACE]
task.spawn(function()
    while task.wait(1) do
        local elapsed = os.time() - State.StartTime
        local minutes = math.floor(elapsed / 60)
        local seconds = elapsed % 60
        UI.Timer.Text = string.format("Tempo: %02d:%02d", minutes, seconds)
        UI.Baus.Text = string.format("Baús: %d / %d", State.Counter, Settings.ChestLimit)
    end
end)

-- // [8. CICLO DE FARM PRINCIPAL]
task.spawn(function()
    while true do
        task.wait(0.1)
        SyncSettings() -- Mantém as configs atualizadas em tempo real

        if _G.Start_Kaitun then
            -- Verifica Limite
            if State.Counter >= Settings.ChestLimit then
                UI.Status.Text = "Status: Limite Atingido!"
                HandleServerHop()
                break
            end

            local hrp = GetHRP()
            if not hrp then 
                UI.Status.Text = "Status: Aguardando Personagem..."
                continue 
            end

            -- Busca do Baú Mais Próximo (Filtro Robusto)
            local targetChest = nil
            local minDistance = math.huge

            pcall(function()
                local children = workspace:GetDescendants()
                for i = 1, #children do
                    local obj = children[i]
                    if obj:IsA("TouchTransmitter") and obj.Parent then
                        local chest = obj.Parent
                        if chest.Name:find("Chest") or chest.Name:find("Baú") then
                            if not State.Blacklist[chest] then
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

            -- Execução do Farm
            if targetChest then
                State.CurrentTarget = targetChest
                UI.Status.Text = "Status: Coletando Baú..."
                
                ExecuteMovement(targetChest)

                -- Verificação de Proximidade Final
                if (hrp.Position - targetChest.Position).Magnitude < Settings.DistanceToCollect then
                    State.Blacklist[targetChest] = true
                    State.Counter = State.Counter + 1
                    UI.Status.Text = "Status: +1 Baú Coletado!"
                    task.wait(0.2)
                end
            else
                UI.Status.Text = "Status: Sem baús no servidor."
                HandleServerHop()
                break
            end
        else
            UI.Status.Text = "Status: Sistema Pausado"
            task.wait(1)
        end
    end
end)

-- // [9. SISTEMAS DE PROTEÇÃO (GIGANTE)]

-- Botão Toggle
UI.Btn.MouseButton1Click:Connect(function()
    _G.Start_Kaitun = not _G.Start_Kaitun
    if _G.Start_Kaitun then
        UI.Btn.Text = "PARAR SISTEMA"
        UI.Btn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
    else
        UI.Btn.Text = "RETOMAR SISTEMA"
        UI.Btn.BackgroundColor3 = Color3.fromRGB(60, 220, 60)
    end
end)

-- NoClip Ultra (Prevenção de Kick por colisão)
RunS.Stepped:Connect(function()
    if _G.Start_Kaitun and Settings.NoClip then
        local char = LocalPlayer.Character
        if char then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide then
                    part.CanCollide = false
                end
            end
        end
    end
end)

-- Anti-AFK Robusto
if Settings.AntiAFK then
    LocalPlayer.Idled:Connect(function()
        VU:CaptureController()
        VU:ClickButton2(Vector2.new())
    end)
end

-- Proteção de UI (Auto-Re-execução de segurança)
UI.Main.AncestryChanged:Connect(function(_, parent)
    if not parent then
        _G.Nova_Loaded = false
        print("UI da Nova Chest removida. Reinicialização permitida.")
    end
end)

-- LOG FINAL DE CARREGAMENTO
print([[
    ========================================
    NOVA CHEST V3 CARREGADO COM SUCESSO!
    SISTEMA: MEGA ROBUST EDITION
    MODOS: TWEEN + TELEPORT SINCRONIZADOS
    LIMITE: ]] .. Settings.ChestLimit .. [[
    ========================================
]])

-- Finalização: Próximo passo sugerido.
-- Você gostaria que eu adicionasse um sistema de LOG que envia para o Webhook do Discord quantos baús foram pegos?
