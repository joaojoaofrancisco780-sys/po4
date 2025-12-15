local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ContextActionService = game:GetService("ContextActionService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- Helper para esperar RemoteEvent
local function waitEvent(name, timeout)
    local ok, ev = pcall(function() return ReplicatedStorage:WaitForChild(name, timeout or 10) end)
    if not ok or not ev then
        warn("[Client] RemoteEvent não encontrado:", name)
        return nil
    end
    return ev
end

local RequestOpenLockpickEvent = waitEvent("RequestOpenLockpick")
local OpenLockpickGuiEvent = waitEvent("OpenLockpickGui")
local AttemptLockpickEvent = waitEvent("AttemptLockpick")
local LockpickResultEvent = waitEvent("LockpickResult")
local ClientNotificationEvent = waitEvent("ClientNotification")

-- UI mínima com botão "Abrir Lockpick"
if playerGui:FindFirstChild("AutoOpenLockpickUI") then playerGui.AutoOpenLockpickUI:Destroy() end
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AutoOpenLockpickUI"
screenGui.Parent = playerGui
screenGui.ResetOnSpawn = false

local btn = Instance.new("TextButton")
btn.Size = UDim2.new(0,220,0,40)
btn.Position = UDim2.new(0, 10, 0, 80)
btn.Text = "Abrir Lockpick (botão)"
btn.Parent = screenGui

local info = Instance.new("TextLabel")
info.Size = UDim2.new(0,300,0,24)
info.Position = UDim2.new(0, 10, 0, 125)
info.BackgroundTransparency = 1
info.Text = ""
info.Parent = screenGui

-- Minigame simples (igual visual do exemplo anterior, mas reduzido)
local mg = Instance.new("Frame", screenGui)
mg.Size = UDim2.new(0, 420, 0, 140)
mg.Position = UDim2.new(0.5, -210, 0.66, -70)
mg.AnchorPoint = Vector2.new(0.5,0)
mg.BackgroundColor3 = Color3.fromRGB(20,20,20)
mg.Visible = false

local bar = Instance.new("Frame", mg)
bar.Size = UDim2.new(0.92,0,0,36)
bar.Position = UDim2.new(0.04,0,0,46)
bar.BackgroundColor3 = Color3.fromRGB(200,200,200)

local mover = Instance.new("Frame", bar)
mover.Size = UDim2.new(0.06,0,1,0)
mover.Position = UDim2.new(0,0,0,0)
mover.BackgroundColor3 = Color3.fromRGB(255,80,80)

local target = Instance.new("Frame", bar)
target.Size = UDim2.new(0.14,0,1,0)
target.Position = UDim2.new(0.43,0,0,0)
target.BackgroundColor3 = Color3.fromRGB(100,220,120)
target.Transparency = 0.15

local mgInfo = Instance.new("TextLabel", mg)
mgInfo.Size = UDim2.new(1, -24, 0, 28)
mgInfo.Position = UDim2.new(0,12,0,88)
mgInfo.BackgroundTransparency = 1
mgInfo.Text = "Clique para tentar"
mgInfo.TextColor3 = Color3.fromRGB(230,230,230)

local running = false
local dir = 1
local speed = 0.9
local currentCar = nil

local function openMinigame(carName)
    currentCar = carName
    mg.Visible = true
    running = true
    mover.Position = UDim2.new(0,0,0,0)
    dir = 1
    mgInfo.Text = "Clique para tentar"
    info.Text = ""
end
local function closeMinigame()
    running = false
    currentCar = nil
    mg.Visible = false
end

-- Ao iniciar, aguarda um pouco e pede ao servidor para abrir o lockpick automaticamente
delay(0.5, function()
    if RequestOpenLockpickEvent then
        RequestOpenLockpickEvent:FireServer()
        info.Text = "Solicitando lockpick automaticamente..."
    else
        info.Text = "Evento RequestOpenLockpick indisponível."
    end
end)

-- Botão manual também disponível
btn.MouseButton1Click:Connect(function()
    if RequestOpenLockpickEvent then
        RequestOpenLockpickEvent:FireServer()
        info.Text = "Pedido enviado..."
    end
end)

bar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 and running and currentCar then
        running = false
        local moverCenter = mover.AbsolutePosition.X + mover.AbsoluteSize.X * 0.5
        local targetStart = target.AbsolutePosition.X
        local targetEnd = target.AbsolutePosition.X + target.AbsoluteSize.X
        local inside = moverCenter >= targetStart and moverCenter <= targetEnd
        if AttemptLockpickEvent then
            AttemptLockpickEvent:FireServer(currentCar, inside)
        end
        closeMinigame()
        info.Text = "Tentando lockpick..."
    end
end)

RunService.Heartbeat:Connect(function(dt)
    if running then
        local pos = mover.Position.X.Scale
        pos = pos + dir * speed * dt
        if pos <= 0 then pos = 0; dir = 1
        elseif pos + mover.Size.X.Scale >= 1 then pos = 1 - mover.Size.X.Scale; dir = -1 end
        mover.Position = UDim2.new(pos,0,0,0)
    end
end)

OpenLockpickGuiEvent.OnClientEvent:Connect(function(carNameOrFalse, maybeMsg)
    if carNameOrFalse == false then
        info.Text = maybeMsg or "Nenhum carro por perto."
        delay(2, function() if info then info.Text = "" end end)
        return
    end
    openMinigame(carNameOrFalse)
end)

LockpickResultEvent.OnClientEvent:Connect(function(success, message)
    info.Text = message or (success and "Desbloqueado!" or "Falhou.")
    delay(2, function() if info then info.Text = "" end end)
end)

print("[Client] Local_AutoOpenLockpick iniciado. Solicitando lockpick automaticamente ao servidor.")
