--// SERVICES
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera

--// CONFIGS
local LP = Players.LocalPlayer
local MAX_SPEED = 200
local RunSpeed, FlySpeed, LockRange = 50, 100, 100
local Running, Flying = false, false
local CurrentTarget = nil

--// UI PRINCIPAL
local ScreenGui = Instance.new("ScreenGui", LP:WaitForChild("PlayerGui"))
ScreenGui.Name = "RestoredLegacy"
ScreenGui.ResetOnSpawn = false

local Frame = Instance.new("Frame", ScreenGui)
Frame.Size = UDim2.new(0, 250, 0, 280) 
Frame.Position = UDim2.new(0.5, -125, 0.5, -140)
Frame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
Frame.BorderSizePixel = 0
Instance.new("UICorner", Frame).CornerRadius = UDim.new(0, 10)

-- FOTO DA PRAIA (CARREGADA VIA ASSET ID)
local Photo = Instance.new("ImageLabel", Frame)
Photo.Size = UDim2.new(0, 60, 0, 60)
Photo.Position = UDim2.new(0.5, -30, 0, 12)
Photo.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
-- ID convertido para Asset do Roblox (Foto da moça na praia)
Photo.Image = "rbxassetid://12373722008" 
Photo.BorderSizePixel = 0
local PhotoCorner = Instance.new("UICorner", Photo)
PhotoCorner.CornerRadius = UDim.new(1, 0)

local PhotoStroke = Instance.new("UIStroke", Photo)
PhotoStroke.Thickness = 2
PhotoStroke.Color = Color3.fromRGB(255, 255, 255)

-- BOTÃO FECHAR
local Close = Instance.new("TextButton", Frame)
Close.Size = UDim2.new(0, 25, 0, 25)
Close.Position = UDim2.new(1, -30, 0, 5)
Close.BackgroundTransparency = 1; Close.Text = "×"; Close.TextColor3 = Color3.fromRGB(255, 80, 80); Close.TextSize = 25
Close.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)

-- SLIDERS ORIGINAIS
local function CreateSlider(name, yPos, defaultVal, maxVal, color, varType)
    local label = Instance.new("TextLabel", Frame)
    label.Size = UDim2.new(1, -20, 0, 18); label.Position = UDim2.new(0, 10, 0, yPos)
    label.Text = name..": "..defaultVal; label.TextColor3 = Color3.fromRGB(220, 220, 220); label.BackgroundTransparency = 1; label.Font = Enum.Font.Gotham; label.TextSize = 13; label.TextXAlignment = Enum.TextXAlignment.Left
    
    local bg = Instance.new("Frame", Frame)
    bg.Size = UDim2.new(1, -20, 0, 10); bg.Position = UDim2.new(0, 10, 0, yPos + 20); bg.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)
    
    local bar = Instance.new("Frame", bg)
    bar.Size = UDim2.new(defaultVal/maxVal, 0, 1, 0); bar.BackgroundColor3 = color; Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)

    bg.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            local connection
            connection = RunService.RenderStepped:Connect(function()
                if not UIS:IsMouseButtonPressed(Enum.UserInputType.MouseButton1) then connection:Disconnect() return end
                local r = math.clamp((UIS:GetMouseLocation().X - bg.AbsolutePosition.X)/bg.AbsoluteSize.X, 0, 1)
                local val = math.floor(r * maxVal)
                bar.Size = UDim2.new(r, 0, 1, 0); label.Text = name..": "..val
                if varType == 1 then LockRange = val elseif varType == 2 then RunSpeed = val elseif varType == 3 then FlySpeed = val end
            end)
        end
    end)
end

CreateSlider("Range", 85, LockRange, 250, Color3.fromRGB(80, 255, 80), 1)
CreateSlider("Corrida", 145, RunSpeed, MAX_SPEED, Color3.fromRGB(80, 150, 255), 2)
CreateSlider("Voo (Fly)", 205, FlySpeed, MAX_SPEED, Color3.fromRGB(255, 150, 80), 3)

--// ESP DE CAIXAS (BOXES) ANTIGO
local function AddESP(p)
    if p == LP then return end
    local Box = Instance.new("BoxHandleAdornment")
    Box.AlwaysOnTop = true; Box.ZIndex = 10; Box.Transparency = 0.6; Box.Color3 = Color3.new(1, 0, 0); Box.Size = Vector3.new(4, 6, 0.5)
    
    local function Update()
        if p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            Box.Adornee = p.Character.HumanoidRootPart; Box.Parent = game.CoreGui
        else
            Box.Adornee = nil
        end
    end
    RunService.RenderStepped:Connect(Update)
end
for _, p in pairs(Players:GetPlayers()) do AddESP(p) end
Players.PlayerAdded:Connect(AddESP)

--// LOCK-ON ANTIGO (FOCO AUTOMÁTICO)
local function GetClosest()
    local target, dist = nil, LockRange
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LP and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local pos, onScreen = Camera:WorldToViewportPoint(p.Character.HumanoidRootPart.Position)
            if onScreen then
                local mag = (Vector2.new(pos.X, pos.Y) - Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)).Magnitude
                if mag < dist then dist = mag; target = p.Character.HumanoidRootPart end
            end
        end
    end
    return target
end

--// LOOP DE MOVIMENTO
RunService.RenderStepped:Connect(function()
    if Running and LP.Character and LP.Character:FindFirstChild("Humanoid") then LP.Character.Humanoid.WalkSpeed = RunSpeed end
    if Flying and LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") and LP.Character.HumanoidRootPart:FindFirstChild("FlyBV") then
        local dir = (Camera.CFrame.LeftVector * (UIS:IsKeyDown(Enum.KeyCode.A) and 1 or UIS:IsKeyDown(Enum.KeyCode.D) and -1 or 0) + Camera.CFrame.LookVector * (UIS:IsKeyDown(Enum.KeyCode.W) and 1 or UIS:IsKeyDown(Enum.KeyCode.S) and -1 or 0))
        LP.Character.HumanoidRootPart.FlyBV.Velocity = dir * FlySpeed
        LP.Character.HumanoidRootPart.FlyBG.CFrame = Camera.CFrame
    end
    if CurrentTarget then Camera.CFrame = CFrame.new(Camera.CFrame.Position, CurrentTarget.Position) end
end)

--// INPUTS (TECLAS)
UIS.InputBegan:Connect(function(i, gp)
    if gp then return end
    if i.KeyCode == Enum.KeyCode.V then Running = true
    elseif i.KeyCode == Enum.KeyCode.T then CurrentTarget = (CurrentTarget and nil or GetClosest())
    elseif i.KeyCode == Enum.KeyCode.Semicolon then Frame.Visible = not Frame.Visible
    elseif i.KeyCode == Enum.KeyCode.N then
        Flying = not Flying
        local hrp = LP.Character:FindFirstChild("HumanoidRootPart")
        if Flying then
            local bv = Instance.new("BodyVelocity", hrp); bv.Name = "FlyBV"; bv.MaxForce = Vector3.new(1e9,1e9,1e9)
            local bg = Instance.new("BodyGyro", hrp); bg.Name = "FlyBG"; bg.MaxTorque = Vector3.new(1e9,1e9,1e9)
        else
            if hrp:FindFirstChild("FlyBV") then hrp.FlyBV:Destroy() end
            if hrp:FindFirstChild("FlyBG") then hrp.FlyBG:Destroy() end
        end
    end
end)
UIS.InputEnded:Connect(function(i) if i.KeyCode == Enum.KeyCode.V then Running = false end end)
