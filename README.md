-- JOSU HUB - LocalScript (pegar en StarterPlayer > StarterPlayerScripts)
local Players = game:GetService("Players")
local StarterGui = game:GetService("StarterGui")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer

-- Variables
local savedCFrame = nil
local noclipActive = false
local noclipConnection = nil
local noclipRestoreTimer = nil

-- Util: safe notification
local function notify(title, text, duration)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = title;
            Text = text;
            Duration = duration or 2;
        })
    end)
end

-- Esperar a que el jugador y su GUI estén listos
repeat task.wait() until player and player:FindFirstChild("PlayerGui")

-- Obtiene Character/Root de forma segura y vuelve a suscribirse al respawn
local character, root
local function refreshCharacter()
    character = player.Character or player.CharacterAdded:Wait()
    root = character:WaitForChild("HumanoidRootPart")
end
refreshCharacter()
player.CharacterAdded:Connect(function()
    refreshCharacter()
end)

-- Noclip: mantiene CanCollide = false en partes del personaje mientras noclipActive
local function startNoclip()
    if noclipActive then return end
    noclipActive = true

    -- cancelar conexión previa si existe
    if noclipConnection then
        noclipConnection:Disconnect()
        noclipConnection = nil
    end

    noclipConnection = RunService.Stepped:Connect(function()
        if not character or not character.Parent then return end
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
                -- opcional: evitar que se anule la transparencia de forma extraña
            end
        end
    end)
    notify("JOSU HUB", "🚪 Traspasar activado", 2)
end

local function stopNoclip()
    if not noclipActive then return end
    noclipActive = false
    if noclipConnection then
        noclipConnection:Disconnect()
        noclipConnection = nil
    end
    -- restaurar colisiones (intentar restaurar todas las partes actuales)
    if character and character.Parent then
        for _, part in ipairs(character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = true
            end
        end
    end
    notify("JOSU HUB", "🔒 Colisión restaurada", 2)
end

-- Guardar posición actual (CFrame)
local function guardarPos()
    if root then
        savedCFrame = root.CFrame
        notify("JOSU HUB", "📍 Posición guardada!", 2)
    else
        notify("JOSU HUB", "❗ No hay HumanoidRootPart", 2)
    end
end

-- Teletransportar al savedCFrame (si existe)
local function teleportarAGuardada()
    if savedCFrame and root then
        -- añadir un pequeño offset en Y para evitar quedar dentro del suelo
        root.CFrame = savedCFrame + Vector3.new(0, 3, 0)
        notify("JOSU HUB", "✅ Teletransportado a la posición guardada", 2)
    else
        notify("JOSU HUB", "❗ No hay posición guardada", 2)
    end
end

-- Toggle noclip (si se activa con duración, lo paramos luego)
local function activarNoclipTemporario(duracion)
    startNoclip()
    -- cancelar timer anterior si existía
    if noclipRestoreTimer then
        noclipRestoreTimer:Cancel()
        noclipRestoreTimer = nil
    end
    noclipRestoreTimer = task.delay(duracion or 5, function()
        stopNoclip()
        noclipRestoreTimer = nil
    end)
end

local function toggleNoclip()
    if noclipActive then
        stopNoclip()
    else
        startNoclip()
    end
end

-- UI: crear ScreenGui y botones (para pegar dinámicamente; si preferís, podés crear en Studio y enlazar)
local playerGui = player:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "JOSU_HUB_GUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Name = "Main"
frame.Size = UDim2.new(0, 220, 0, 120)
frame.Position = UDim2.new(0, 12, 0, 80)
frame.AnchorPoint = Vector2.new(0,0)
frame.BackgroundTransparency = 0.15
frame.BackgroundColor3 = Color3.fromRGB(25,25,25)
frame.BorderSizePixel = 0
frame.Parent = screenGui
frame.ClipsDescendants = true
frame.Padding = nil

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -12, 0, 28)
title.Position = UDim2.new(0, 6, 0, 6)
title.BackgroundTransparency = 1
title.Text = "JOSU HUB"
title.Font = Enum.Font.SourceSansBold
title.TextSize = 18
title.TextColor3 = Color3.fromRGB(255,255,255)
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = frame

local function makeButton(name, text, y)
    local btn = Instance.new("TextButton")
    btn.Name = name
    btn.Size = UDim2.new(1, -12, 0, 30)
    btn.Position = UDim2.new(0, 6, 0, y)
    btn.BackgroundColor3 = Color3.fromRGB(35,35,35)
    btn.BorderSizePixel = 0
    btn.AutoButtonColor = true
    btn.Text = text
    btn.Font = Enum.Font.SourceSans
    btn.TextSize = 16
    btn.TextColor3 = Color3.fromRGB(255,255,255)
    btn.Parent = frame
    return btn
end

local btnTP = makeButton("TP", "TP (Guardar)", 36)
local btnTP2 = makeButton("TP2", "TP2 (Ir a guardada)", 72)
local btnTP3 = makeButton("TP3", "TP3 (Noclip ON/OFF)", 108)

-- Ajustar tamaño del frame a los botones
frame.Size = UDim2.new(0, 220, 0, 150)
btnTP.Position = UDim2.new(0, 6, 0, 36)
btnTP2.Position = UDim2.new(0, 6, 0, 72)
btnTP3.Position = UDim2.new(0, 6, 0, 108)

-- Conexiones de botones
btnTP.MouseButton1Click:Connect(function()
    guardarPos()
end)

btnTP2.MouseButton1Click:Connect(function()
    teleportarAGuardada()
end)

btnTP3.MouseButton1Click:Connect(function()
    toggleNoclip()
    if noclipActive then
        notify("JOSU HUB", "🚪 Traspasar activado (hasta desactivar)", 2)
    else
        notify("JOSU HUB", "🔒 Colisión restaurada", 2)
    end
end)

-- Comportamiento automático al iniciar (igual que tu versión original, pero controlado)
-- 1) Guardar posición al entrar
guardarPos()

-- 2) Esperar 2s y teletransportar
task.delay(2, function()
    teleportarAGuardada()
end)

-- 3) Esperar 4s (2s + 2s) y activar noclip temporal por 5s
task.delay(4, function()
    activarNoclipTemporario(5)
end)

-- Manejo adicional: si respawnea mientras noclip activo, asegurarse de aplicar
player.CharacterAdded:Connect(function(char)
    character = char
    root = character:WaitForChild("HumanoidRootPart")
    if noclipActive then
        -- breve espera hasta que partes existan
        task.wait(0.1)
        startNoclip()
    end
end)# JOSU
Josuemessias
