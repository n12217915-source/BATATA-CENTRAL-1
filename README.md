--// ============================================================
--// MÓDULO AIM LOCK v1 — BLOCK 1
--// Setup, Save/Load e Estado
--// ============================================================

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local Cam = workspace.CurrentCamera

local api = ReplicatedStorage:WaitForChild("BatataHub_RegisterTab")

local CONFIG_FOLDER = "Batata Central"
local CONFIG_FILE = "aimlock.json"

local function getConfigPath()
	if not isfolder(CONFIG_FOLDER) then
		makefolder(CONFIG_FOLDER)
	end
	return CONFIG_FOLDER .. "/" .. CONFIG_FILE
end

local function saveAim(data)
	pcall(function()
		writefile(getConfigPath(), HttpService:JSONEncode(data))
	end)
end

local function loadAim()
	local path = getConfigPath()
	if isfile(path) then
		local ok, decoded = pcall(function()
			return HttpService:JSONDecode(readfile(path))
		end)
		if ok and type(decoded) == "table" then
			return decoded
		end
	end
	local default = {}
	saveAim(default)
	return default
end

local S = loadAim()

local Aim = {
	Enabled = S.Enabled == true or false,
	FOV = S.FOV or 200,
	Part = S.Part or "Head",
	Walls = S.Walls == true or false,
	FOVVisible = S.FOVVisible == true or false,
	FOVColor = S.FOVColor or "Branco",
	FOVThickness = S.FOVThickness or 1,
	FOVFilled = S.FOVFilled == true or false,
	FOVFillTransparency = S.FOVFillTransparency or 95,
	FOVPulse = S.FOVPulse == true or false,
	LockKey = S.LockKey or "E",
	ToggleMode = S.ToggleMode == true or false,
	AutoLock = S.AutoLock == true or false,
	DisguiseMode = S.DisguiseMode == true or false,
	SmoothLocked = S.SmoothLocked or 8,
	SmoothAcquire = S.SmoothAcquire or 4,
	MaxAngularSpeed = S.MaxAngularSpeed or 25,
	JitterAmount = S.JitterAmount or 0.15,
	PredictEnabled = S.PredictEnabled == true or false,
	PredictStrength = S.PredictStrength or 0.08,
	PredictAccel = S.PredictAccel == true or false,
	AutoReleaseOnWall = S.AutoReleaseOnWall == true or false,
	ReacquireDelay = S.ReacquireDelay or 0.35,
	BreakMouse = S.BreakMouse or 20,
	BreakDistance = S.BreakDistance or 120,
	TeamCheck = S.TeamCheck == true or false,
	IgnoreFriends = S.IgnoreFriends == true or false,
	OnlyVisible = S.OnlyVisible == true or false,
}

local Cores = {
	["Branco"]   = Color3.fromRGB(255, 255, 255),
	["Vermelho"] = Color3.fromRGB(255, 60, 60),
	["Verde"]    = Color3.fromRGB(60, 255, 100),
	["Azul"]     = Color3.fromRGB(80, 160, 255),
	["Amarelo"]  = Color3.fromRGB(255, 220, 80),
	["Roxo"]     = Color3.fromRGB(200, 100, 255),
	["Rosa"]     = Color3.fromRGB(255, 120, 200),
	["Ciano"]    = Color3.fromRGB(80, 240, 240),
}

local function persist()
	saveAim({
		Enabled = Aim.Enabled, FOV = Aim.FOV, Part = Aim.Part, Walls = Aim.Walls,
		FOVVisible = Aim.FOVVisible, FOVColor = Aim.FOVColor, FOVThickness = Aim.FOVThickness,
		FOVFilled = Aim.FOVFilled, FOVFillTransparency = Aim.FOVFillTransparency, FOVPulse = Aim.FOVPulse,
		LockKey = Aim.LockKey, ToggleMode = Aim.ToggleMode, AutoLock = Aim.AutoLock,
		DisguiseMode = Aim.DisguiseMode, SmoothLocked = Aim.SmoothLocked, SmoothAcquire = Aim.SmoothAcquire,
		MaxAngularSpeed = Aim.MaxAngularSpeed, JitterAmount = Aim.JitterAmount,
		PredictEnabled = Aim.PredictEnabled, PredictStrength = Aim.PredictStrength, PredictAccel = Aim.PredictAccel,
		AutoReleaseOnWall = Aim.AutoReleaseOnWall, ReacquireDelay = Aim.ReacquireDelay,
		BreakMouse = Aim.BreakMouse, BreakDistance = Aim.BreakDistance,
		TeamCheck = Aim.TeamCheck, IgnoreFriends = Aim.IgnoreFriends, OnlyVisible = Aim.OnlyVisible,
	})
end
--// ============================================================
--// BLOCK 2 — FOV Circle Visual
--// ============================================================

local visualGui = Instance.new("ScreenGui")
visualGui.Name = "BatataHub_AimVisuals"
visualGui.ResetOnSpawn = false
visualGui.IgnoreGuiInset = true
visualGui.Parent = player:WaitForChild("PlayerGui")

local fovCircle = Instance.new("Frame")
fovCircle.AnchorPoint = Vector2.new(0.5, 0.5)
fovCircle.Position = UDim2.new(0.5, 0, 0.5, 0)
fovCircle.BackgroundTransparency = 1
fovCircle.ZIndex = 5
fovCircle.Visible = false
fovCircle.Parent = visualGui

Instance.new("UICorner", fovCircle).CornerRadius = UDim.new(1, 0)

local fovStroke = Instance.new("UIStroke")
fovStroke.Color = Color3.fromRGB(255, 255, 255)
fovStroke.Thickness = 1
fovStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
fovStroke.Parent = fovCircle

local fovFill = Instance.new("Frame")
fovFill.AnchorPoint = Vector2.new(0.5, 0.5)
fovFill.Position = UDim2.new(0.5, 0, 0.5, 0)
fovFill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
fovFill.BackgroundTransparency = 0.95
fovFill.BorderSizePixel = 0
fovFill.ZIndex = 4
fovFill.Visible = false
fovFill.Parent = visualGui

Instance.new("UICorner", fovFill).CornerRadius = UDim.new(1, 0)

local pulseState = 0
local pulseDir = 1

local function updateFOV()
	local color = Cores[Aim.FOVColor] or Color3.fromRGB(255, 255, 255)
	fovStroke.Color = color
	fovStroke.Thickness = Aim.FOVThickness
	fovCircle.Size = UDim2.new(0, Aim.FOV, 0, Aim.FOV)
	fovCircle.Visible = Aim.FOVVisible
	fovFill.Size = UDim2.new(0, Aim.FOV, 0, Aim.FOV)
	fovFill.BackgroundColor3 = color
	fovFill.BackgroundTransparency = Aim.FOVFillTransparency / 100
	fovFill.Visible = Aim.FOVVisible and Aim.FOVFilled
end

updateFOV()
Cam:GetPropertyChangedSignal("ViewportSize"):Connect(updateFOV)
--// ============================================================
--// BLOCK 3 — Aim Lock Core
--// ============================================================

local Lock = {
	target = nil, locked = false,
	lastMousePos = UserInputService:GetMouseLocation(),
	lastVisibleCheck = 0, lastTargetWasVisible = false,
	lastReleaseTime = 0, acquireProgress = 0,
}

local isLockedGlobal = false

local friendSet = {}
task.spawn(function()
	local ok, list = pcall(function() return Players:GetFriendsAsync(player.UserId) end)
	if ok and list then
		while true do
			for _, info in ipairs(list:GetCurrentPage()) do friendSet[info.Id] = true end
			if list.IsFinished then break end
			pcall(function() list:AdvanceToNextPageAsync() end)
		end
	end
end)

local function isAlly(p)
	if Aim.TeamCheck and p.Team and p.Team == player.Team then return true end
	if Aim.IgnoreFriends and friendSet[p.UserId] then return true end
	return false
end

local function getPart(p)
	if not p.Character then return nil end
	return p.Character:FindFirstChild(Aim.Part) or p.Character:FindFirstChild("HumanoidRootPart")
end

local function isVisible(p)
	if not Aim.Walls then return true end
	local char = p.Character
	if not char then return false end
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hum or hum.Health <= 0 then return false end
	local part = getPart(p)
	if not part then return false end
	local origin = Cam.CFrame.Position
	local direction = part.Position - origin
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = {player.Character, Cam}
	params.IgnoreWater = true
	local result = workspace:Raycast(origin, direction, params)
	if not result then return true end
	return result.Instance:IsDescendantOf(char)
end

local function getTarget()
	local best = nil
	local bestScore = math.huge
	local center = Cam.ViewportSize / 2
	for _, p in ipairs(Players:GetPlayers()) do
		if p == player then continue end
		if isAlly(p) then continue end
		if not p.Character then continue end
		local hum = p.Character:FindFirstChildOfClass("Humanoid")
		if not hum or hum.Health <= 0 then continue end
		local part = getPart(p)
		if not part then continue end
		local sp, onScreen = Cam:WorldToViewportPoint(part.Position)
		if not onScreen then continue end
		local dist2D = (Vector2.new(sp.X, sp.Y) - center).Magnitude
		if dist2D > Aim.FOV / 2 then continue end
		if Aim.OnlyVisible and not isVisible(p) then continue end
		local dist3D = (part.Position - Cam.CFrame.Position).Magnitude
		local score = dist2D + dist3D * 0.05
		if score < bestScore then bestScore = score; best = p end
	end
	return best
end

local function getPredictedPosition(target)
	if not target.Character then return nil end
	local part = getPart(target)
	if not part then return nil end
	local pos = part.Position
	if Aim.PredictEnabled then
		local hum = target.Character:FindFirstChildOfClass("Humanoid")
		if hum then
			local vel = hum.MoveDirection * hum.WalkSpeed
			pos = pos + vel * Aim.PredictStrength
			if Aim.PredictAccel then
				local state = hum:GetState()
				if state == Enum.HumanoidStateType.Jumping or state == Enum.HumanoidStateType.Freefall then
					pos = pos + Vector3.new(0, 4, 0) * Aim.PredictStrength
				end
			end
		end
	end
	return pos
end

local function applyJitter(cframe)
	if Aim.JitterAmount <= 0 then return cframe end
	local mag = Aim.JitterAmount * 0.5
	local jx = (math.random() - 0.5) * mag
	local jy = (math.random() - 0.5) * mag
	return cframe * CFrame.Angles(math.rad(jy), math.rad(jx), 0)
end

local function applyAim(target, isAcquiring)
	if not target.Character then return end
	local targetPos = getPredictedPosition(target)
	if not targetPos then return end
	local targetCFrame = CFrame.lookAt(Cam.CFrame.Position, targetPos)
	targetCFrame = applyJitter(targetCFrame)
	if Aim.DisguiseMode then
		local delta = targetCFrame.LookVector - Cam.CFrame.LookVector
		local angle = math.deg(math.acos(math.clamp(delta.Magnitude, -1, 1)))
		local maxStep = Aim.MaxAngularSpeed / 60
		local alpha = isAcquiring and math.clamp(1 / Aim.SmoothAcquire, 0, 1) or math.clamp(1 / Aim.SmoothLocked, 0, 1)
		if angle > maxStep * 3 then alpha = alpha * 0.5 end
		Cam.CFrame = Cam.CFrame:Lerp(targetCFrame, alpha)
	else
		local alpha = isAcquiring and 0.25 or 0.5
		Cam.CFrame = Cam.CFrame:Lerp(targetCFrame, alpha)
	end
end

local function shouldBreakLock()
	if not Lock.locked or not Lock.target then return false end
	local part = getPart(Lock.target)
	if not part then return true end
	local sp = Cam:WorldToViewportPoint(part.Position)
	local center = Cam.ViewportSize / 2
	local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
	return d > Aim.BreakDistance
end

local function handleWallRelease(now)
	if not Aim.AutoReleaseOnWall then return end
	if not Lock.locked or not Lock.target then return end
	if now - Lock.lastVisibleCheck < 0.05 then return end
	Lock.lastVisibleCheck = now
	local visible = isVisible(Lock.target)
	if not visible and Lock.lastTargetWasVisible then
		Lock.lastTargetWasVisible = false
		Lock.lastReleaseTime = now
	elseif visible and not Lock.lastTargetWasVisible then
		Lock.lastTargetWasVisible = true
		Lock.lastReleaseTime = 0
	end
end

local function shouldSkipAim()
	if not Aim.AutoReleaseOnWall then return false end
	if not Lock.locked or not Lock.target then return false end
	if Lock.lastTargetWasVisible then return false end
	if Lock.lastReleaseTime > 0 and (tick() - Lock.lastReleaseTime) > (Aim.ReacquireDelay * 3) then
		Lock.locked = false
		Lock.target = nil
		Lock.lastTargetWasVisible = false
		Lock.lastReleaseTime = 0
		isLockedGlobal = false
		return true
	end
	return true
end

UserInputService.InputBegan:Connect(function(input, gp)
	if gp then return end
	if input.KeyCode ~= Enum.KeyCode[Aim.LockKey] then return end
	if Aim.AutoLock then return end
	if Aim.ToggleMode then
		Lock.locked = not Lock.locked
		if Lock.locked then
			Lock.target = getTarget()
			Lock.lastMousePos = UserInputService:GetMouseLocation()
			Lock.lastTargetWasVisible = Lock.target and isVisible(Lock.target) or false
			Lock.acquireProgress = 0
		else
			Lock.target = nil
		end
		isLockedGlobal = Lock.locked
	else
		Lock.locked = true
		Lock.target = getTarget()
		Lock.lastMousePos = UserInputService:GetMouseLocation()
		Lock.lastTargetWasVisible = Lock.target and isVisible(Lock.target) or false
		Lock.acquireProgress = 0
		isLockedGlobal = true
	end
end)

UserInputService.InputEnded:Connect(function(input, gp)
	if gp then return end
	if Aim.ToggleMode then return end
	if input.KeyCode ~= Enum.KeyCode[Aim.LockKey] then return end
	Lock.locked = false
	Lock.target = nil
	isLockedGlobal = false
end)

UserInputService.InputChanged:Connect(function(input)
	if input.UserInputType ~= Enum.UserInputType.MouseMovement then return end
	local pos = UserInputService:GetMouseLocation()
	if Lock.locked then
		local delta = (pos - Lock.lastMousePos).Magnitude
		if delta > Aim.BreakMouse then
			Lock.locked = false
			Lock.target = nil
			isLockedGlobal = false
		end
	end
	Lock.lastMousePos = pos
end)

RunService:BindToRenderStep("BatataHub_AimLock", 201, function(dt)
	local now = tick()

	if Aim.FOVVisible and Aim.FOVPulse then
		if not isLockedGlobal then
			pulseState = pulseState * 0.9
			fovStroke.Transparency = math.clamp(pulseState, 0, 0.5)
		else
			pulseState = pulseState + pulseDir * dt * 2
			if pulseState >= 1 then pulseState = 1; pulseDir = -1 end
			if pulseState <= 0 then pulseState = 0; pulseDir = 1 end
			fovStroke.Transparency = math.clamp(pulseState * 0.4, 0, 0.5)
		end
	else
		fovStroke.Transparency = 0
	end

	if not Aim.Enabled then isLockedGlobal = false; return end

	handleWallRelease(now)

	if Aim.AutoLock and not Lock.locked then
		local t = getTarget()
		if t then
			Lock.target = t; Lock.locked = true
			Lock.lastTargetWasVisible = isVisible(t)
			Lock.acquireProgress = 0
			isLockedGlobal = true
		end
	end

	if Lock.locked and shouldBreakLock() then
		Lock.locked = false; Lock.target = nil; isLockedGlobal = false
	end

	if Lock.locked and Lock.target then
		local part = getPart(Lock.target)
		if not part then Lock.locked = false; Lock.target = nil; isLockedGlobal = false end
	end

	if shouldSkipAim() then isLockedGlobal = false; return end

	if Lock.locked and Lock.target then
		Lock.acquireProgress = math.min(1, Lock.acquireProgress + dt * 3)
		applyAim(Lock.target, Lock.acquireProgress < 1)
		isLockedGlobal = true
	end
end)
--// ============================================================
--// BLOCK 4 — Helpers de UI
--// ============================================================

local function makeToggle(container, ctx, labelText, getter, setter)
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, -8, 0, 26)
	row.BackgroundColor3 = ctx.colors.CARD
	row.BorderSizePixel = 0
	row.Parent = container
	Instance.new("UICorner", row).CornerRadius = UDim.new(0, 5)

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Position = UDim2.fromOffset(8, 0)
	label.Size = UDim2.new(1, -50, 1, 0)
	label.Font = Enum.Font.Gotham
	label.Text = labelText
	label.TextSize = 10
	label.TextColor3 = ctx.colors.TEXT
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = row

	local btn = Instance.new("TextButton")
	btn.AnchorPoint = Vector2.new(1, 0.5)
	btn.Position = UDim2.new(1, -6, 0.5, 0)
	btn.Size = UDim2.fromOffset(34, 18)
	btn.BackgroundColor3 = getter() and ctx.colors.ACCENT or ctx.colors.PANEL
	btn.Text = ""
	btn.AutoButtonColor = false
	btn.Parent = row
	Instance.new("UICorner", btn).CornerRadius = UDim.new(1, 0)

	local knob = Instance.new("Frame")
	knob.Size = UDim2.fromOffset(14, 14)
	knob.Position = getter() and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
	knob.BackgroundColor3 = getter() and ctx.colors.BLACK or ctx.colors.SUBTEXT
	knob.BorderSizePixel = 0
	knob.Parent = btn
	Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

	btn.MouseButton1Click:Connect(function()
		local state = not getter()
		setter(state)
		btn.BackgroundColor3 = state and ctx.colors.ACCENT or ctx.colors.PANEL
		knob.BackgroundColor3 = state and ctx.colors.BLACK or ctx.colors.SUBTEXT
		knob.Position = state and UDim2.new(1, -16, 0.5, -7) or UDim2.new(0, 2, 0.5, -7)
	end)
end

local function makeSlider(container, ctx, labelText, min, max, getter, setter, onUpdate)
	local wrap = Instance.new("Frame")
	wrap.Size = UDim2.new(1, -8, 0, 36)
	wrap.BackgroundColor3 = ctx.colors.CARD
	wrap.BorderSizePixel = 0
	wrap.Parent = container
	Instance.new("UICorner", wrap).CornerRadius = UDim.new(0, 5)

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Position = UDim2.fromOffset(8, 3)
	label.Size = UDim2.new(0.7, -10, 0, 12)
	label.Font = Enum.Font.Gotham
	label.Text = labelText
	label.TextSize = 10
	label.TextColor3 = ctx.colors.TEXT
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = wrap

	local valueLabel = Instance.new("TextLabel")
	valueLabel.BackgroundTransparency = 1
	valueLabel.AnchorPoint = Vector2.new(1, 0)
	valueLabel.Position = UDim2.new(1, -8, 0, 3)
	valueLabel.Size = UDim2.new(0.3, 0, 0, 12)
	valueLabel.Font = Enum.Font.GothamBold
	valueLabel.Text = tostring(getter())
	valueLabel.TextSize = 10
	valueLabel.TextColor3 = ctx.colors.ACCENT
	valueLabel.TextXAlignment = Enum.TextXAlignment.Right
	valueLabel.Parent = wrap

	local track = Instance.new("Frame")
	track.Position = UDim2.fromOffset(8, 22)
	track.Size = UDim2.new(1, -16, 0, 5)
	track.BackgroundColor3 = ctx.colors.PANEL
	track.BorderSizePixel = 0
	track.Parent = wrap
	Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)

	local fraction = (getter() - min) / (max - min)
	local fill = Instance.new("Frame")
	fill.Size = UDim2.new(fraction, 0, 1, 0)
	fill.BackgroundColor3 = ctx.colors.ACCENT
	fill.BorderSizePixel = 0
	fill.Parent = track
	Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

	local knob = Instance.new("Frame")
	knob.AnchorPoint = Vector2.new(0.5, 0.5)
	knob.Position = UDim2.new(fraction, 0, 0.5, 0)
	knob.Size = UDim2.fromOffset(11, 11)
	knob.BackgroundColor3 = ctx.colors.TEXT
	knob.BorderSizePixel = 0
	knob.Parent = track
	Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

	local dragging = false
	local function setFromX(x)
		local rel = math.clamp((x - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
		fill.Size = UDim2.new(rel, 0, 1, 0)
		knob.Position = UDim2.new(rel, 0, 0.5, 0)
		local v = math.floor(min + rel * (max - min) + 0.5)
		valueLabel.Text = tostring(v)
		setter(v)
		if onUpdate then onUpdate() end
	end

	track.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			setFromX(input.Position.X)
		end
	end)
	knob.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
		end
	end)
	UserInputService.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
			setFromX(input.Position.X)
		end
	end)
	UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end)
end

local function makeDropdown(container, ctx, labelText, options, getter, setter, onUpdate)
	local wrap = Instance.new("Frame")
	wrap.Size = UDim2.new(1, -8, 0, 40)
	wrap.BackgroundColor3 = ctx.colors.CARD
	wrap.BorderSizePixel = 0
	wrap.Parent = container
	Instance.new("UICorner", wrap).CornerRadius = UDim.new(0, 5)

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Position = UDim2.fromOffset(8, 3)
	label.Size = UDim2.new(1, -16, 0, 12)
	label.Font = Enum.Font.Gotham
	label.Text = labelText
	label.TextSize = 10
	label.TextColor3 = ctx.colors.TEXT
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = wrap

	local btn = Instance.new("TextButton")
	btn.Position = UDim2.fromOffset(8, 18)
	btn.Size = UDim2.new(1, -16, 0, 16)
	btn.BackgroundColor3 = ctx.colors.PANEL
	btn.Text = "  " .. getter() .. "  ▼"
	btn.Font = Enum.Font.Gotham
	btn.TextSize = 10
	btn.TextColor3 = ctx.colors.TEXT
	btn.TextXAlignment = Enum.TextXAlignment.Left
	btn.AutoButtonColor = false
	btn.Parent = wrap
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)

	local expanded = false
	local optionFrame

	btn.MouseButton1Click:Connect(function()
		expanded = not expanded
		if expanded then
			optionFrame = Instance.new("Frame")
			optionFrame.Size = UDim2.new(1, 0, 0, #options * 18 + 6)
			optionFrame.Position = UDim2.new(0, 0, 1, 2)
			optionFrame.BackgroundColor3 = ctx.colors.PANEL
			optionFrame.BorderSizePixel = 0
			optionFrame.ZIndex = 20
			optionFrame.Parent = wrap
			Instance.new("UICorner", optionFrame).CornerRadius = UDim.new(0, 4)

			local optLayout = Instance.new("UIListLayout")
			optLayout.Padding = UDim.new(0, 1)
			optLayout.Parent = optionFrame

			local pad = Instance.new("UIPadding")
			pad.PaddingLeft = UDim.new(0, 3)
			pad.PaddingTop = UDim.new(0, 3)
			pad.PaddingRight = UDim.new(0, 3)
			pad.PaddingBottom = UDim.new(0, 3)
			pad.Parent = optionFrame

			for _, opt in ipairs(options) do
				local ob = Instance.new("TextButton")
				ob.Size = UDim2.new(1, 0, 0, 16)
				ob.BackgroundTransparency = 1
				ob.Text = "  " .. opt
				ob.Font = Enum.Font.Gotham
				ob.TextSize = 9
				ob.TextColor3 = ctx.colors.TEXT
				ob.TextXAlignment = Enum.TextXAlignment.Left
				ob.AutoButtonColor = false
				ob.ZIndex = 21
				ob.Parent = optionFrame

				ob.MouseEnter:Connect(function()
					ob.BackgroundTransparency = 0
					ob.BackgroundColor3 = ctx.colors.CARD
				end)
				ob.MouseLeave:Connect(function()
					ob.BackgroundTransparency = 1
				end)
				ob.MouseButton1Click:Connect(function()
					setter(opt)
					btn.Text = "  " .. opt .. "  ▼"
					expanded = false
					if optionFrame then optionFrame:Destroy() end
					if onUpdate then onUpdate() end
				end)
			end
		else
			if optionFrame then optionFrame:Destroy() end
		end
	end)
end
--// ============================================================
--// BLOCK 5 — Registro da Aba AIM + Load Automático
--// ============================================================

api:Invoke("Batata001", {
	Name = "AIM",
	BuildContent = function(container, ctx)
		local title = Instance.new("TextLabel")
		title.BackgroundTransparency = 1
		title.Position = UDim2.fromOffset(9, 6)
		title.Size = UDim2.new(1, -18, 0, 16)
		title.Font = Enum.Font.GothamBlack
		title.Text = "AIM LOCK"
		title.TextSize = 13
		title.TextColor3 = ctx.colors.TEXT
		title.TextXAlignment = Enum.TextXAlignment.Left
		title.Parent = container

		local scroll = Instance.new("ScrollingFrame")
		scroll.Size = UDim2.new(1, -14, 1, -28)
		scroll.Position = UDim2.fromOffset(7, 24)
		scroll.BackgroundTransparency = 1
		scroll.BorderSizePixel = 0
		scroll.ScrollBarThickness = 3
		scroll.ScrollBarImageColor3 = ctx.colors.ACCENT
		scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
		scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
		scroll.Parent = container

		local layout = Instance.new("UIListLayout")
		layout.Padding = UDim.new(0, 4)
		layout.SortOrder = Enum.SortOrder.LayoutOrder
		layout.Parent = scroll

		makeToggle(scroll, ctx, "Ativar Aim Lock", function() return Aim.Enabled end, function(v) Aim.Enabled = v; persist() end)
		makeToggle(scroll, ctx, "Modo Hold (segurar)", function() return Aim.ToggleMode end, function(v) Aim.ToggleMode = v; persist() end)
		makeToggle(scroll, ctx, "Auto Lock", function() return Aim.AutoLock end, function(v) Aim.AutoLock = v; persist() end)
		makeToggle(scroll, ctx, "Ignorar Paredes", function() return Aim.Walls end, function(v) Aim.Walls = v; persist() end)
		makeToggle(scroll, ctx, "Soltar atrás de parede", function() return Aim.AutoReleaseOnWall end, function(v) Aim.AutoReleaseOnWall = v; persist() end)
		makeToggle(scroll, ctx, "Team Check", function() return Aim.TeamCheck end, function(v) Aim.TeamCheck = v; persist() end)
		makeToggle(scroll, ctx, "Ignorar Amigos", function() return Aim.IgnoreFriends end, function(v) Aim.IgnoreFriends = v; persist() end)
		makeToggle(scroll, ctx, "Modo Disfarçado", function() return Aim.DisguiseMode end, function(v) Aim.DisguiseMode = v; persist() end)
		makeToggle(scroll, ctx, "Ativar Predição", function() return Aim.PredictEnabled end, function(v) Aim.PredictEnabled = v; persist() end)
		makeToggle(scroll, ctx, "Mostrar Círculo FOV", function() return Aim.FOVVisible end, function(v) Aim.FOVVisible = v; persist(); updateFOV() end)
		makeToggle(scroll, ctx, "Preencher FOV", function() return Aim.FOVFilled end, function(v) Aim.FOVFilled = v; persist(); updateFOV() end)
		makeToggle(scroll, ctx, "Pulsar quando Lockado", function() return Aim.FOVPulse end, function(v) Aim.FOVPulse = v; persist() end)

		makeSlider(scroll, ctx, "FOV", 50, 800, function() return Aim.FOV end, function(v) Aim.FOV = v; persist() end, updateFOV)
		makeSlider(scroll, ctx, "Suavidade no Lock", 1, 20, function() return Aim.SmoothLocked end, function(v) Aim.SmoothLocked = v; persist() end)
		makeSlider(scroll, ctx, "Velocidade Angular", 5, 90, function() return Aim.MaxAngularSpeed end, function(v) Aim.MaxAngularSpeed = v; persist() end)
		makeSlider(scroll, ctx, "Ruído da Mira", 0, 100, function() return math.floor(Aim.JitterAmount * 100) end, function(v) Aim.JitterAmount = v / 100; persist() end)
		makeSlider(scroll, ctx, "Força da Predição", 1, 30, function() return math.floor(Aim.PredictStrength * 100) end, function(v) Aim.PredictStrength = v / 100; persist() end)

		makeDropdown(scroll, ctx, "Parte do Corpo", {"Head", "HumanoidRootPart", "UpperTorso", "LowerTorso"}, function() return Aim.Part end, function(v) Aim.Part = v; persist() end)
		makeDropdown(scroll, ctx, "Tecla de Lock", {"E", "Q", "F", "G", "R", "T", "V", "C", "X", "Z"}, function() return Aim.LockKey end, function(v) Aim.LockKey = v; persist() end)
		makeDropdown(scroll, ctx, "Cor do FOV", {"Branco", "Vermelho", "Verde", "Azul", "Amarelo", "Roxo", "Rosa", "Ciano"}, function() return Aim.FOVColor end, function(v) Aim.FOVColor = v; persist(); updateFOV() end)
	end,
})

--// Load automático com 1s entre cada ativação
task.spawn(function()
	task.wait(1.5)

	local hasData = next(S) ~= nil
	if not hasData then return end

	local applyOrder = {
		{key = "Enabled", apply = function(v) Aim.Enabled = v end},
		{key = "Walls", apply = function(v) Aim.Walls = v end},
		{key = "AutoLock", apply = function(v) Aim.AutoLock = v end},
		{key = "AutoReleaseOnWall", apply = function(v) Aim.AutoReleaseOnWall = v end},
		{key = "DisguiseMode", apply = function(v) Aim.DisguiseMode = v end},
		{key = "PredictEnabled", apply = function(v) Aim.PredictEnabled = v end},
		{key = "TeamCheck", apply = function(v) Aim.TeamCheck = v end},
		{key = "IgnoreFriends", apply = function(v) Aim.IgnoreFriends = v end},
		{key = "FOVVisible", apply = function(v) Aim.FOVVisible = v; updateFOV() end},
		{key = "FOVFilled", apply = function(v) Aim.FOVFilled = v; updateFOV() end},
		{key = "FOVPulse", apply = function(v) Aim.FOVPulse = v end},
	}

	for _, entry in ipairs(applyOrder) do
		if S[entry.key] == true then
			entry.apply(true)
		end
		task.wait(1)
	end

	print("[Batata Hub - AIM] Configurações carregadas automaticamente.")
end)

print("[Módulo AIM LOCK v2] Registrado na Batata Hub.")
