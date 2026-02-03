-- LocalScript / StarterPlayerScripts
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LP = Players.LocalPlayer
local ValidateKey = ReplicatedStorage:WaitForChild("ValidateKey")

-- KEYS
local remainingTime = 0
local isPermanent = false

-- PAINEL
local ScreenGui = Instance.new("ScreenGui", LP.PlayerGui)
ScreenGui.ResetOnSpawn = false

local ToggleCircle = Instance.new("ImageButton", ScreenGui)
ToggleCircle.Size = UDim2.new(0,60,0,60)
ToggleCircle.Position = UDim2.new(1,-80,1,-80)
ToggleCircle.BackgroundTransparency = 1
ToggleCircle.Image = "rbxassetid://SEU_ASSET_ID_AQUI"
Instance.new("UICorner", ToggleCircle).CornerRadius = UDim.new(1,0)

local Panel = Instance.new("Frame", ScreenGui)
Panel.Size = UDim2.new(0,300,0,200)
Panel.Position = UDim2.new(0.5,-150,0.5,-100)
Panel.BackgroundColor3 = Color3.fromRGB(25,25,25)
Panel.Visible = false
Instance.new("UICorner", Panel).CornerRadius = UDim.new(0,10)

local KeyBox = Instance.new("TextBox", Panel)
KeyBox.Size = UDim2.new(1,-40,0,40)
KeyBox.Position = UDim2.new(0,20,0,40)
KeyBox.PlaceholderText = "Digite a key..."

local Submit = Instance.new("TextButton", Panel)
Submit.Size = UDim2.new(1,-40,0,40)
Submit.Position = UDim2.new(0,20,0,90)
Submit.Text = "ATIVAR"

local TimeLabel = Instance.new("TextLabel", Panel)
TimeLabel.Size = UDim2.new(1,0,0,20)
TimeLabel.Position = UDim2.new(0,0,1,-30)
TimeLabel.BackgroundTransparency = 1
TimeLabel.TextColor3 = Color3.fromRGB(80,255,80)
TimeLabel.Font = Enum.Font.GothamBold

local function formatTime(sec)
	local h = math.floor(sec/3600)
	local m = math.floor((sec%3600)/60)
	local s = sec%60
	return string.format("%02d:%02d:%02d",h,m,s)
end

-- Toggle painel
ToggleCircle.MouseButton1Click:Connect(function()
	Panel.Visible = not Panel.Visible
end)

-- Ativar key
Submit.MouseButton1Click:Connect(function()
	local ok, permanent, timeOrErr = ValidateKey:InvokeServer(KeyBox.Text)
	if ok then
		Panel.Visible = true
		isPermanent = permanent
		remainingTime = timeOrErr or 0
	else
		KeyBox.Text = timeOrErr
	end
end)

-- CONTADOR
RunService.RenderStepped:Connect(function()
	if remainingTime > 0 and not isPermanent then
		remainingTime -= 1/60
		TimeLabel.Text = "Tempo restante: "..formatTime(math.max(0,remainingTime))
	end
end)

-- MOVIMENTO & FLY
local MIN_SPEED = 16
local MAX_SPEED = 300
local RunSpeed = 300
local FlySpeed = 300
local Running, Flying = false,false
local BV,BG
local NormalSpeed = 16

local function getHum() return LP.Character and LP.Character:FindFirstChildOfClass("Humanoid") end
local function getHRP() return LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") end

UIS.InputBegan:Connect(function(i,gp)
	if gp then return end
	if i.KeyCode == Enum.KeyCode.V then
		local hum = getHum()
		if hum then Running = true; NormalSpeed = hum.WalkSpeed; hum.WalkSpeed = RunSpeed end
	end
	if i.KeyCode == Enum.KeyCode.N then
		local hrp,h = getHRP(),getHum()
		if hrp and h then
			Flying = not Flying
			if Flying then
				h:ChangeState(Enum.HumanoidStateType.Physics)
				BV = Instance.new("BodyVelocity", hrp)
				BV.MaxForce = Vector3.new(1e9,1e9,1e9)
				BG = Instance.new("BodyGyro", hrp)
				BG.MaxTorque = Vector3.new(1e9,1e9,1e9)
			else
				if BV then BV:Destroy() end
				if BG then BG:Destroy() end
				h:ChangeState(Enum.HumanoidStateType.GettingUp)
			end
		end
	end
end)

UIS.InputEnded:Connect(function(i,gp)
	if gp then return end
	if i.KeyCode == Enum.KeyCode.V then
		local hum = getHum()
		if hum then Running = false; hum.WalkSpeed = NormalSpeed end
	end
end)

-- RENDERSTEPPED LOOP
RunService.RenderStepped:Connect(function()
	local hum,hrp = getHum(),getHRP()
	if Running and hum then hum.WalkSpeed = RunSpeed end
	if Flying and BV and BG then
		local dir = Vector3.zero
		local cam = Camera.CFrame
		if UIS:IsKeyDown(Enum.KeyCode.W) then dir += cam.LookVector end
		if UIS:IsKeyDown(Enum.KeyCode.S) then dir -= cam.LookVector end
		if UIS:IsKeyDown(Enum.KeyCode.A) then dir -= cam.RightVector end
		if UIS:IsKeyDown(Enum.KeyCode.D) then dir += cam.RightVector end
		if UIS:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.new(0,1,0) end
		if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then dir -= Vector3.new(0,1,0) end
		BV.Velocity = dir.Magnitude>0 and dir.Unit*FlySpeed or Vector3.zero
		BG.CFrame = cam
	end
end)
