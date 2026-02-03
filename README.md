--// SERVICES
local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera

--// PLAYER
local LP = Players.LocalPlayer
local PlayerGui = LP:WaitForChild("PlayerGui")local RunKey = Enum.KeyCode.V
local FlyKey = Enum.KeyCode.M
local LockOnKey = Enum.KeyCode.Y
local TogglePanelKey = Enum.KeyCode.Semicolon -- Ç

--// SPEED
local MIN_SPEED = 16
local MAX_SPEED = 100
local RunSpeed = 50
local Running = false
local NormalSpeed = 16

--// STATES
local Flying = false
local PanelVisible = true

--// FLY OBJECTS
local BV, BG

--// LOCK-ON
local LockRange = 100
local CurrentTarget = nil
local ESPBoxes = {}

--// FUNCTIONS
local function getChar() return LP.Character end
local function getHum()
	local c = getChar()
	return c and c:FindFirstChildOfClass("Humanoid")
end
local function getHRP()
	local c = getChar()
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function IsValidTarget(player)
	if player == LP then return false end
	if not player.Character then return false end
	local hum = player.Character:FindFirstChild("Humanoid")
	local hrp = player.Character:FindFirstChild("HumanoidRootPart")
	if not hum or hum.Health <= 0 or not hrp then return false end
	return true
end

local function GetClosestPlayerOnScreen()
	local closest, shortest = nil, LockRange
	for _, p in pairs(Players:GetPlayers()) do
		if IsValidTarget(p) then
			local pos, onScreen = Camera:WorldToViewportPoint(p.Character.HumanoidRootPart.Position)
			if onScreen then
				local dist = (Vector2.new(pos.X,pos.Y) - Vector2.new(Camera.ViewportSize.X/2,Camera.ViewportSize.Y/2)).Magnitude
				if dist <= shortest then
					shortest = dist
					closest = p
				end
			end
		end
	end
	return closest
end

--// UI
local ScreenGui = Instance.new("ScreenGui", PlayerGui)
ScreenGui.Name = "LockOnUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true

-- Frame
local Frame = Instance.new("Frame", ScreenGui)
Frame.Size = UDim2.new(0,250,0,210)
Frame.Position = UDim2.new(0.5,-125,0.8,-105)
Frame.BackgroundColor3 = Color3.fromRGB(30,30,30)
Frame.BorderSizePixel = 0
Instance.new("UICorner", Frame).CornerRadius = UDim.new(0,8)

local StatusLabel = Instance.new("TextLabel", Frame)
StatusLabel.Size = UDim2.new(1,0,0,30)
StatusLabel.Position = UDim2.new(0,0,0,5)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "LOCK-OFF"
StatusLabel.TextColor3 = Color3.fromRGB(255,80,80)
StatusLabel.Font = Enum.Font.GothamBold
StatusLabel.TextSize = 20

-- Range
local RangeLabel = Instance.new("TextLabel", Frame)
RangeLabel.Size = UDim2.new(1,-20,0,20)
RangeLabel.Position = UDim2.new(0,10,0,40)
RangeLabel.BackgroundTransparency = 1
RangeLabel.Text = "Range: "..LockRange
RangeLabel.TextColor3 = Color3.fromRGB(200,200,200)
RangeLabel.Font = Enum.Font.Gotham
RangeLabel.TextSize = 16
RangeLabel.TextXAlignment = Enum.TextXAlignment.Left

local RangeSlider = Instance.new("Frame", Frame)
RangeSlider.Size = UDim2.new(1,-20,0,20)
RangeSlider.Position = UDim2.new(0,10,0,60)
RangeSlider.BackgroundColor3 = Color3.fromRGB(60,60,60)
local SliderBar = Instance.new("Frame", RangeSlider)
SliderBar.Size = UDim2.new((LockRange-50)/200,0,1,0)
SliderBar.BackgroundColor3 = Color3.fromRGB(80,255,80)

-- Speed
local SpeedLabel = Instance.new("TextLabel", Frame)
SpeedLabel.Size = UDim2.new(1,-20,0,20)
SpeedLabel.Position = UDim2.new(0,10,0,90)
SpeedLabel.BackgroundTransparency = 1
SpeedLabel.Text = "Speed: "..RunSpeed
SpeedLabel.TextColor3 = Color3.fromRGB(200,200,200)
SpeedLabel.Font = Enum.Font.Gotham
SpeedLabel.TextSize = 16
SpeedLabel.TextXAlignment = Enum.TextXAlignment.Left

local SpeedSlider = Instance.new("Frame", Frame)
SpeedSlider.Size = UDim2.new(1,-20,0,20)
SpeedSlider.Position = UDim2.new(0,10,0,110)
SpeedSlider.BackgroundColor3 = Color3.fromRGB(60,60,60)
local SpeedBar = Instance.new("Frame", SpeedSlider)
SpeedBar.Size = UDim2.new((RunSpeed-MIN_SPEED)/(MAX_SPEED-MIN_SPEED),0,1,0)
SpeedBar.BackgroundColor3 = Color3.fromRGB(80,150,255)

-- Lock Circle
local LockCircle = Instance.new("Frame", ScreenGui)
LockCircle.Size = UDim2.new(0,LockRange*2,0,LockRange*2)
LockCircle.AnchorPoint = Vector2.new(0.5,0.5)
LockCircle.Position = UDim2.new(0.5,0,0.5,0)
LockCircle.BackgroundTransparency = 1
Instance.new("UICorner", LockCircle).CornerRadius = UDim.new(1,0)
Instance.new("UIStroke", LockCircle).Thickness = 2

-- ESP Folder
local ESPFolder = Instance.new("Folder", ScreenGui)
ESPFolder.Name = "ESPFolder"

-- Input
local draggingRange, draggingSpeed = false,false
RangeSlider.InputBegan:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 then draggingRange=true end end)
RangeSlider.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 then draggingRange=false end end)
SpeedSlider.InputBegan:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 then draggingSpeed=true end end)
SpeedSlider.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 then draggingSpeed=false end end)

UIS.InputChanged:Connect(function(i)
	if i.UserInputType~=Enum.UserInputType.MouseMovement then return end
	if draggingRange then
		local r = math.clamp((i.Position.X-RangeSlider.AbsolutePosition.X)/RangeSlider.AbsoluteSize.X,0,1)
		LockRange = math.floor(50 + r*200)
		RangeLabel.Text = "Range: "..LockRange
		LockCircle.Size = UDim2.new(0,LockRange*2,0,LockRange*2)
		SliderBar.Size = UDim2.new(r,0,1,0)
	end
	if draggingSpeed then
		local r = math.clamp((i.Position.X-SpeedSlider.AbsolutePosition.X)/SpeedSlider.AbsoluteSize.X,0,1)
		RunSpeed = math.floor(MIN_SPEED + r*(MAX_SPEED-MIN_SPEED))
		SpeedLabel.Text = "Speed: "..RunSpeed
		SpeedBar.Size = UDim2.new(r,0,1,0)
	end
end)

UIS.InputBegan:Connect(function(i,gp)
	if gp then return end
	if i.KeyCode==TogglePanelKey then
		PanelVisible = not PanelVisible
		Frame.Visible = PanelVisible
		LockCircle.Visible = PanelVisible
	end
	if i.KeyCode==RunKey then
		local hum=getHum()
		if hum then Running=true;NormalSpeed=hum.WalkSpeed;hum.WalkSpeed=RunSpeed end
	end
	if i.KeyCode==FlyKey then
		Flying=not Flying
		local hrp,h=getHRP(),getHum()
		if not hrp or not h then return end
		if Flying then
			h:ChangeState(Enum.HumanoidStateType.Physics)
			BV=Instance.new("BodyVelocity",hrp)
			BV.MaxForce=Vector3.new(1e9,1e9,1e9)
			BG=Instance.new("BodyGyro",hrp)
			BG.MaxTorque=Vector3.new(1e9,1e9,1e9)
		else
			if BV then BV:Destroy() end
			if BG then BG:Destroy() end
			h:ChangeState(Enum.HumanoidStateType.GettingUp)
		end
	end
	if i.KeyCode==LockOnKey then
		if CurrentTarget then
			CurrentTarget=nil
			StatusLabel.Text="LOCK-OFF"
			StatusLabel.TextColor3=Color3.fromRGB(255,80,80)
		else
			local c=GetClosestPlayerOnScreen()
			if c then
				CurrentTarget=c
				StatusLabel.Text="LOCK-ON"
				StatusLabel.TextColor3=Color3.fromRGB(80,255,80)
			end
		end
	end
end)

UIS.InputEnded:Connect(function(i,gp)
	if gp then return end
	if i.KeyCode==RunKey then
		local h=getHum()
		if h then Running=false;h.WalkSpeed=NormalSpeed end
	end
end)

-- MAIN LOOP
RunService.RenderStepped:Connect(function()
	local hum,hrp=getHum(),getHRP()
	if not hum or not hrp then return end
	if Running and hum.WalkSpeed~=RunSpeed then hum.WalkSpeed=RunSpeed end
	if Flying and BV and BG then
		local dir=Vector3.zero
		local cam=Camera.CFrame
		if UIS:IsKeyDown(Enum.KeyCode.W) then dir+=cam.LookVector end
		if UIS:IsKeyDown(Enum.KeyCode.S) then dir-=cam.LookVector end
		if UIS:IsKeyDown(Enum.KeyCode.A) then dir-=cam.RightVector end
		if UIS:IsKeyDown(Enum.KeyCode.D) then dir+=cam.RightVector end
		if UIS:IsKeyDown(Enum.KeyCode.Space) then dir+=Vector3.new(0,1,0) end
		if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then dir-=Vector3.new(0,1,0) end
		BV.Velocity=dir.Magnitude>0 and dir.Unit*150 or Vector3.zero
		BG.CFrame=cam
	end

	-- LOCK-ON
	if CurrentTarget and CurrentTarget.Character then
		local tHRP=CurrentTarget.Character:FindFirstChild("HumanoidRootPart")
		if tHRP then Camera.CFrame=CFrame.new(Camera.CFrame.Position,tHRP.Position) end
	end

	-- ESP
	for p,d in pairs(ESPBoxes) do
		if IsValidTarget(p) then
			local pos,onScreen=Camera:WorldToViewportPoint(p.Character.HumanoidRootPart.Position)
			if onScreen then
				d.Box.Position=UDim2.new(0,pos.X-16,0,pos.Y-26)
				d.HP.Text=math.floor(p.Character.Humanoid.Health).."/"..math.floor(p.Character.Humanoid.MaxHealth)
				d.Box.Visible=PanelVisible
			else d.Box.Visible=false end
		else d.Box.Visible=false end
	end
end)

-- ESP LOOP
while true do
	for _,p in pairs(Players:GetPlayers()) do
		if IsValidTarget(p) and not ESPBoxes[p] then
			local box=Instance.new("Frame",ESPFolder)
			box.Size=UDim2.new(0,32,0,52)
			box.BackgroundTransparency=1
			box.BorderSizePixel=2
			box.BorderColor3=Color3.fromRGB(255,255,255)

			local name=Instance.new("TextLabel",box)
			name.Size=UDim2.new(3,0,0,16)
			name.Position=UDim2.new(-1,0,-1.8,0)
			name.BackgroundTransparency=1
			name.Text=p.Name
			name.Font=Enum.Font.GothamBold
			name.TextSize=14
			name.TextColor3=Color3.fromRGB(255,255,255)
			name.TextStrokeTransparency=0.3
			name.TextXAlignment=Enum.TextXAlignment.Center

			local hp=Instance.new("TextLabel",box)
			hp.Size=UDim2.new(1,0,0,15)
			hp.Position=UDim2.new(0,0,-0.9,0)
			hp.BackgroundTransparency=1
			hp.Font=Enum.Font.GothamBold
			hp.TextSize=13
			hp.TextColor3=Color3.fromRGB(80,255,80)
			hp.TextStrokeTransparency=0.4

			ESPBoxes[p]={Box=box,Name=name,HP=hp}
		end
	end
	task.wait(1)
end
