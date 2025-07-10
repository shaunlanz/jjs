local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer

local flying = false
local bodyVelocity
local bodyGyro
local flightConnection
local humanoidRootPart
local animateConnection

local idleAnimId = "rbxassetid://91428863336534"
local moveAnimId = "rbxassetid://107554693613496"

local idleTrack
local moveTrack

local FLIGHT_SPEED = 90

local function getCharacterRoot()
	local character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
	return character:FindFirstChild("HumanoidRootPart"), character
end

local function getTargetVelocity()
	local root, character = getCharacterRoot()
	if not character then return Vector3.zero end

	local humanoid = character:FindFirstChildOfClass("Humanoid")
	if not humanoid then return Vector3.zero end

	local moveDir = humanoid.MoveDirection
	if moveDir.Magnitude == 0 then
		return Vector3.zero
	end

	local camera = workspace.CurrentCamera
	if not camera then return Vector3.zero end

	local lookVector = camera.CFrame.LookVector.Unit
	local horizontalDir = Vector3.new(moveDir.X, 0, moveDir.Z).Unit

	local forwardAmount = moveDir:Dot(Vector3.new(camera.CFrame.LookVector.X, 0, camera.CFrame.LookVector.Z).Unit)
	local verticalVelocity = lookVector.Y * forwardAmount * FLIGHT_SPEED

	local horizontalVelocity = horizontalDir * FLIGHT_SPEED
	return Vector3.new(horizontalVelocity.X, verticalVelocity, horizontalVelocity.Z)
end

local function playIdleAnimation(humanoid)
	if moveTrack then
		moveTrack:Stop()
		moveTrack:Destroy()
		moveTrack = nil
	end

	if not idleTrack then
		local anim = Instance.new("Animation")
		anim.AnimationId = idleAnimId
		idleTrack = humanoid:LoadAnimation(anim)
		idleTrack:Play()
		idleTrack.Looped = true
	end
end

local function playMovingAnimation(humanoid)
	if idleTrack then
		idleTrack:Stop()
		idleTrack:Destroy()
		idleTrack = nil
	end

	if not moveTrack then
		local anim = Instance.new("Animation")
		anim.AnimationId = moveAnimId
		moveTrack = humanoid:LoadAnimation(anim)
		moveTrack:Play()
	end

	if moveTrack then
		moveTrack:AdjustSpeed(1)
		moveTrack.TimePosition = 0.57
		task.delay(0.23, function() -- 0.57 + 0.23 = 0.8
			if moveTrack then
				moveTrack.TimePosition = 0.8
				moveTrack:AdjustSpeed(0)
			end
		end)
	end
end

local function stopAnimations()
	if idleTrack then
		idleTrack:Stop()
		idleTrack:Destroy()
		idleTrack = nil
	end
	if moveTrack then
		moveTrack:Stop()
		moveTrack:Destroy()
		moveTrack = nil
	end
end

local function startFlight()
	if flying then return end
	local root, character = getCharacterRoot()
	humanoidRootPart = root
	if not root or not character then return end

	if bodyVelocity then bodyVelocity:Destroy() end
	if bodyGyro then bodyGyro:Destroy() end

	bodyVelocity = Instance.new("BodyVelocity")
	bodyVelocity.MaxForce = Vector3.new(1e5, 1e5, 1e5)
	bodyVelocity.P = 1e4
	bodyVelocity.Velocity = Vector3.zero
	bodyVelocity.Parent = humanoidRootPart

	bodyGyro = Instance.new("BodyGyro")
	bodyGyro.MaxTorque = Vector3.new(1e5, 1e5, 1e5)
	bodyGyro.P = 2e4
	bodyGyro.CFrame = workspace.CurrentCamera.CFrame
	bodyGyro.Parent = humanoidRootPart

	flying = true

	local humanoid = character:FindFirstChildOfClass("Humanoid")
	if animateConnection then animateConnection:Disconnect() end
	animateConnection = nil
	if humanoid then
		animateConnection = RunService.RenderStepped:Connect(function()
			if flying and humanoid then
				local moveDirMag = humanoid.MoveDirection.Magnitude
				if moveDirMag <= 0.05 then
					playIdleAnimation(humanoid)
				else
					playMovingAnimation(humanoid)
				end
			end
		end)
	end

	flightConnection = RunService.RenderStepped:Connect(function()
		local targetVelocity = getTargetVelocity()
		if bodyVelocity then
			bodyVelocity.Velocity = targetVelocity
		end
		if bodyGyro and humanoidRootPart then
			bodyGyro.CFrame = CFrame.new(humanoidRootPart.Position, humanoidRootPart.Position + workspace.CurrentCamera.CFrame.LookVector)
		end
	end)
end

local function stopFlight()
	if not flying then return end
	flying = false
	if flightConnection then flightConnection:Disconnect() flightConnection = nil end
	if animateConnection then animateConnection:Disconnect() animateConnection = nil end
	if bodyVelocity then bodyVelocity:Destroy() bodyVelocity = nil end
	if bodyGyro then bodyGyro:Destroy() bodyGyro = nil end
	stopAnimations()
end

UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.UserInputType == Enum.UserInputType.Keyboard then
		if input.KeyCode == Enum.KeyCode.K then
			if not flying then
				startFlight()
			else
				stopFlight()
			end
		end
	end
end)

-- GUI creation
local playerGui = LocalPlayer:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FlightControlGui"
screenGui.Parent = playerGui

-- ==== Flying Toggle Button ====
local toggleButton = Instance.new("TextButton")
toggleButton.Size = UDim2.new(0, 50, 0, 50) -- small square
toggleButton.Position = UDim2.new(0, 20, 0, 20)
toggleButton.BackgroundColor3 = Color3.fromRGB(25, 25, 112)
toggleButton.TextColor3 = Color3.new(1,1,1)
toggleButton.Font = Enum.Font.SourceSansBold
toggleButton.TextSize = 22
toggleButton.Text = "Fly"
toggleButton.AutoButtonColor = false
toggleButton.Parent = screenGui

-- Drag support for toggle button
local draggingToggle = false
local dragInputToggle, dragStartToggle, startPosToggle

local function updateToggle(input)
	local delta = input.Position - dragStartToggle
	toggleButton.Position = UDim2.new(startPosToggle.X.Scale, startPosToggle.X.Offset + delta.X,
									startPosToggle.Y.Scale, startPosToggle.Y.Offset + delta.Y)
end

toggleButton.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		draggingToggle = true
		dragStartToggle = input.Position
		startPosToggle = toggleButton.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				draggingToggle = false
			end
		end)
	end
end)

toggleButton.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		dragInputToggle = input
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if input == dragInputToggle and draggingToggle then
		updateToggle(input)
	end
end)

toggleButton.MouseButton1Click:Connect(function()
	if not flying then
		startFlight()
		toggleButton.Text = "Stop"
		toggleButton.BackgroundColor3 = Color3.fromRGB(178, 34, 34)
	else
		stopFlight()
		toggleButton.Text = "Fly"
		toggleButton.BackgroundColor3 = Color3.fromRGB(25, 25, 112)
	end
end)

-- ==== Speed Control Frame ====
local speedFrame = Instance.new("Frame")
speedFrame.Size = UDim2.new(0, 220, 0, 70)
speedFrame.Position = UDim2.new(0, 20, 0, 90)
speedFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 85)
speedFrame.BorderSizePixel = 0
speedFrame.Parent = screenGui

-- Draggable header for speed frame
local header = Instance.new("TextLabel")
header.Size = UDim2.new(1, 0, 0, 25)
header.BackgroundColor3 = Color3.fromRGB(50, 50, 120)
header.TextColor3 = Color3.new(1,1,1)
header.Font = Enum.Font.SourceSansBold
header.TextSize = 18
header.Text = "Flight Speed"
header.Parent = speedFrame

-- Speed value display
local speedValueLabel = Instance.new("TextLabel")
speedValueLabel.Size = UDim2.new(0, 50, 0, 25)
speedValueLabel.Position = UDim2.new(1, -55, 0, 5)
speedValueLabel.BackgroundTransparency = 1
speedValueLabel.TextColor3 = Color3.new(1,1,1)
speedValueLabel.Font = Enum.Font.SourceSansBold
speedValueLabel.TextSize = 18
speedValueLabel.Text = tostring(FLIGHT_SPEED)
speedValueLabel.Parent = speedFrame

-- Slider background bar
local sliderBar = Instance.new("Frame")
sliderBar.Size = UDim2.new(1, -60, 0, 20)
sliderBar.Position = UDim2.new(0, 5, 0, 40)
sliderBar.BackgroundColor3 = Color3.fromRGB(70, 70, 140)
sliderBar.BorderSizePixel = 0
sliderBar.Parent = speedFrame

-- Slider draggable knob
local knob = Instance.new("Frame")
knob.Size = UDim2.new(0, 20, 1, 0)
knob.Position = UDim2.new((FLIGHT_SPEED - 50) / 150, 0, 0, 0) -- normalized position from speed 50-200
knob.BackgroundColor3 = Color3.fromRGB(178, 34, 34)
knob.BorderSizePixel = 0
knob.Parent = sliderBar
knob.Active = true
knob.Draggable = false

-- Drag logic for knob
local draggingKnob = false
local dragStartPos
local knobStartPos

local function updateKnob(input)
	local sliderAbsoluteSize = sliderBar.AbsoluteSize.X
	local delta = input.Position.X - dragStartPos.X
	local newPos = knobStartPos + delta
	newPos = math.clamp(newPos, 0, sliderAbsoluteSize - knob.AbsoluteSize.X)
	
	local norm = newPos / (sliderAbsoluteSize - knob.AbsoluteSize.X)
	knob.Position = UDim2.new(0, newPos, 0, 0)
	
	FLIGHT_SPEED = math.floor(50 + 150 * norm)
	speedValueLabel.Text = tostring(FLIGHT_SPEED)
end

knob.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		draggingKnob = true
		dragStartPos = input.Position
		knobStartPos = knob.AbsolutePosition.X - sliderBar.AbsolutePosition.X
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				draggingKnob = false
			end
		end)
	end
end)

knob.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		if draggingKnob then
			updateKnob(input)
		end
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if draggingKnob then
		updateKnob(input)
	end
end)
