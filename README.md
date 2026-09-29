--=============================================================
--  AIM LOCK — solta do alvo ao arrastar a mira pra fora
--=============================================================
local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")
local Workspace         = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Camera      = Workspace.CurrentCamera

local Config = {
	Enabled      = false,
	AutoLock     = true,
	FOV          = 120,
	Smooth       = 0.25,
	AimPart      = "Head",
	Prediction   = 0.15,
	ShowFOV      = true,
	Highlight    = true,
	TeamCheck    = true,
	DragOutDist  = 40,     -- px que o mouse pode sair do alvo pra soltar
	BindPC       = Enum.KeyCode.E,
	ToggleBindPC = Enum.KeyCode.RightAlt,
}

local CurrentTarget = nil
local StickyTarget  = nil   -- alvo "grudado" enquanto não arrastar pra fora
local HighlightObj  = nil

--=============================================================
--  Helpers
--=============================================================
local function isAlive(plr)
	local char = plr.Character
	if not char then return false end
	local hum = char:FindFirstChildOfClass("Humanoid")
	return hum and hum.Health > 0
end

local function isEnemy(plr)
	if plr == LocalPlayer then return false end
	if not Config.TeamCheck then return true end
	return plr.Team ~= LocalPlayer.Team
end

local function getAimPart(plr)
	local char = plr.Character
	if not char then return nil end
	if Config.AimPart == "Head" then
		return char:FindFirstChild("Head")
	elseif Config.AimPart == "Torso" then
		return char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
	else
		return char:FindFirstChild("HumanoidRootPart")
	end
end

local function worldToScreen(pos)
	local sp, onScreen = Camera:WorldToViewportPoint(pos)
	return Vector2.new(sp.X, sp.Y), onScreen
end

local function getScreenCenter()
	local vp = Camera.ViewportSize
	return Vector2.new(vp.X / 2, vp.Y / 2)
end

local function getMousePos()
	return UserInputService:GetMouseLocation()
end

--=============================================================
--  VISÃO: SEMPRE raycast (não mira em quem tá atrás da parede)
--=============================================================
local function isVisible(part)
	local origin = Camera.CFrame.Position
	local dir    = part.Position - origin
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { LocalPlayer.Character, part.Parent, Camera }
	params.IgnoreWater = true
	local result = Workspace:Raycast(origin, dir, params)
	return result == nil or result.Instance:IsDescendantOf(part.Parent)
end

--=============================================================
--  Seleção de alvo
--=============================================================
local function getClosestTarget()
	local center = getScreenCenter()
	local best, bestDist = nil, Config.FOV

	for _, plr in ipairs(Players:GetPlayers()) do
		if not isAlive(plr) or not isEnemy(plr) then continue end
		local part = getAimPart(plr)
		if not part then continue end

		local screenPos, onScreen = worldToScreen(part.Position)
		if not onScreen then continue end

		local dist = (screenPos - center).Magnitude
		if dist < bestDist and isVisible(part) then
			bestDist = dist
			best = plr
		end
	end
	return best
end

--=============================================================
--  Prediction
--=============================================================
local function predictPosition(part)
	if Config.Prediction <= 0 then return part.Position end
	return part.Position + part.AssemblyLinearVelocity * Config.Prediction
end

--=============================================================
--  Highlight
--=============================================================
local function clearHighlight()
	if HighlightObj then HighlightObj:Destroy() HighlightObj = nil end
end

local function applyHighlight(plr)
	clearHighlight()
	if not Config.Highlight or not plr or not plr.Character then return end
	local h = Instance.new("Highlight")
	h.FillColor        = Color3.fromRGB(255, 60, 60)
	h.OutlineColor     = Color3.fromRGB(255, 255, 255)
	h.FillTransparency = 0.6
	h.DepthMode        = Enum.HighlightDepthMode.AlwaysOnTop
	h.Adornee          = plr.Character
	h.Parent           = plr.Character
	HighlightObj = h
end

--=============================================================
--  Lógica de soltar por arrastar pra fora
--=============================================================
local function shouldReleaseFromTarget(plr)
	if not plr or not plr.Character then return true end
	local part = getAimPart(plr)
	if not part then return true end
	local screenPos, onScreen = worldToScreen(part.Position)
	if not onScreen then return true end
	local mouse = getMousePos()
	return (mouse - screenPos).Magnitude > Config.DragOutDist
end

--=============================================================
--  Loop principal
--=============================================================
RunService.RenderStepped:Connect(function()
	if not Config.Enabled then
		StickyTarget = nil
		clearHighlight()
		return
	end

	-- decide se pega ou mantém alvo
	local wantsLock = Config.AutoLock or UserInputService:IsKeyDown(Config.BindPC)

	if not wantsLock then
		StickyTarget = nil
	else
		-- Se tem sticky, mantém até arrastar pra fora
		if StickyTarget and (not isAlive(StickyTarget) or shouldReleaseFromTarget(StickyTarget)) then
			StickyTarget = nil
		end

		if not StickyTarget then
			StickyTarget = getClosestTarget()
		end
	end

	CurrentTarget = StickyTarget

	if not CurrentTarget or not isAlive(CurrentTarget) then
		clearHighlight()
		return
	end

	local part = getAimPart(CurrentTarget)
	if not part or not isVisible(part) then
		StickyTarget = nil
		clearHighlight()
		return
	end

	applyHighlight(CurrentTarget)

	local targetPos = predictPosition(part)
	local desired   = CFrame.new(Camera.CFrame.Position, targetPos)
	Camera.CFrame  = Camera.CFrame:Lerp(desired, math.clamp(1 - Config.Smooth, 0, 1))
end)

--=============================================================
--  Toggle bind (PC)
--=============================================================
UserInputService.InputBegan:Connect(function(input, gpe)
	if gpe then return end
	if input.KeyCode == Config.ToggleBindPC then
		Config.Enabled = not Config.Enabled
	end
end)

--=============================================================
--  FOV visual
--=============================================================
local fovGui = Instance.new("ScreenGui")
fovGui.Name = "AimLockFOV"
fovGui.ResetOnSpawn = false
fovGui.IgnoreGuiInset = true
fovGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local fovFrame = Instance.new("Frame")
fovFrame.AnchorPoint = Vector2.new(0.5,0.5)
fovFrame.Position    = UDim2.fromScale(0.5,0.5)
fovFrame.BackgroundTransparency = 1
fovFrame.Size        = UDim2.fromOffset(Config.FOV*2, Config.FOV*2)
fovFrame.Parent      = fovGui

local fovStroke = Instance.new("UIStroke")
fovStroke.Thickness = 1
fovStroke.Color     = Color3.fromRGB(255,255,255)
fovStroke.Transparency = 0.5
fovStroke.Parent = fovFrame

local fovCorner = Instance.new("UICorner")
fovCorner.CornerRadius = UDim.new(1,0)
fovCorner.Parent = fovFrame

-- botão mobile
if UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled then
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.fromOffset(60,60)
	btn.Position = UDim2.new(1,-80,0.5,-30)
	btn.BackgroundColor3 = Color3.fromRGB(30,30,30)
	btn.BackgroundTransparency = 0.25
	btn.TextColor3 = Color3.new(1,1,1)
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 11
	btn.Text = "AIM\nOFF"
	btn.BorderSizePixel = 0
	btn.Parent = fovGui
	local c = Instance.new("UICorner") c.CornerRadius = UDim.new(1,0) c.Parent = btn
	btn.MouseButton1Click:Connect(function()
		Config.Enabled = not Config.Enabled
		btn.Text = Config.Enabled and "AIM\nON" or "AIM\nOFF"
		btn.BackgroundColor3 = Config.Enabled and Color3.fromRGB(40,160,70) or Color3.fromRGB(30,30,30)
	end)
end

--=============================================================
--  Registra aba (com delay 0s — primeira aba)
--=============================================================
task.wait(0)

local api = ReplicatedStorage:WaitForChild("BatataHub_RegisterTab")

local ok, err = api:Invoke("Batata001", {
	Name = "AIM LOCK",
	BuildContent = function(page, ctx)
		local function makeTitle(text, y)
			local t = Instance.new("TextLabel")
			t.BackgroundTransparency = 1
			t.Position = UDim2.fromOffset(9, y)
			t.Size = UDim2.new(1,-18,0,16)
			t.Font = Enum.Font.GothamBlack
			t.Text = text
			t.TextSize = 13
			t.TextColor3 = ctx.colors.TEXT
			t.TextXAlignment = Enum.TextXAlignment.Left
			t.Parent = page
		end

		local function makeToggle(text, y, initial, onChange)
			local btn = Instance.new("TextButton")
			btn.Position = UDim2.fromOffset(9, y)
			btn.Size = UDim2.new(1,-18,0,22)
			btn.BackgroundColor3 = ctx.colors.BUTTON or Color3.fromRGB(35,35,35)
			btn.Text = ""
			btn.BorderSizePixel = 0
			btn.Parent = page
			local c = Instance.new("UICorner") c.CornerRadius = UDim.new(0,5) c.Parent = btn
			local lbl = Instance.new("TextLabel")
			lbl.BackgroundTransparency = 1
			lbl.Position = UDim2.fromOffset(8,0)
			lbl.Size = UDim2.new(1,-50,1,0)
			lbl.Font = Enum.Font.Gotham
			lbl.TextSize = 11
			lbl.TextColor3 = ctx.colors.TEXT
			lbl.TextXAlignment = Enum.TextXAlignment.Left
			lbl.Text = text
			lbl.Parent = btn
			local st = Instance.new("TextLabel")
			st.BackgroundTransparency = 1
			st.Position = UDim2.new(1,-45,0,0)
			st.Size = UDim2.fromOffset(40,22)
			st.Font = Enum.Font.GothamBold
			st.TextSize = 11
			st.TextColor3 = initial and Color3.fromRGB(90,220,120) or Color3.fromRGB(220,90,90)
			st.Text = initial and "ON" or "OFF"
			st.Parent = btn
			btn.MouseButton1Click:Connect(function()
				initial = not initial
				st.Text = initial and "ON" or "OFF"
				st.TextColor3 = initial and Color3.fromRGB(90,220,120) or Color3.fromRGB(220,90,90)
				onChange(initial)
			end)
		end

		local function makeSlider(text, y, min, max, initial, onChange)
			local frame = Instance.new("Frame")
			frame.Position = UDim2.fromOffset(9,y)
			frame.Size = UDim2.new(1,-18,0,30)
			frame.BackgroundTransparency = 1
			frame.Parent = page
			local lbl = Instance.new("TextLabel")
			lbl.BackgroundTransparency = 1
			lbl.Size = UDim2.new(1,0,0,14)
			lbl.Font = Enum.Font.Gotham
			lbl.TextSize = 11
			lbl.TextColor3 = ctx.colors.SUBTEXT
			lbl.TextXAlignment = Enum.TextXAlignment.Left
			lbl.Text = text..": "..tostring(initial)
			lbl.Parent = frame
			local bar = Instance.new("Frame")
			bar.Position = UDim2.fromOffset(0,18)
			bar.Size = UDim2.new(1,0,0,6)
			bar.BackgroundColor3 = Color3.fromRGB(50,50,50)
			bar.BorderSizePixel = 0
			bar.Parent = frame
			local bc = Instance.new("UICorner") bc.CornerRadius = UDim.new(1,0) bc.Parent = bar
			local fill = Instance.new("Frame")
			fill.Size = UDim2.new((initial-min)/(max-min),0,1,0)
			fill.BackgroundColor3 = ctx.colors.ACCENT or Color3.fromRGB(120,180,255)
			fill.BorderSizePixel = 0
			fill.Parent = bar
			local fc = Instance.new("UICorner") fc.CornerRadius = UDim.new(1,0) fc.Parent = fill
			local dragging = false
			local function update(input)
				local pos = math.clamp((input.Position.X - bar.AbsolutePosition.X)/bar.AbsoluteSize.X,0,1)
				fill.Size = UDim2.new(pos,0,1,0)
				local val = math.floor(min+(max-min)*pos+0.5)
				lbl.Text = text..": "..tostring(val)
				onChange(val)
			end
			bar.InputBegan:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1
				or input.UserInputType == Enum.UserInputType.Touch then
					dragging = true update(input)
				end
			end)
			UserInputService.InputChanged:Connect(function(input)
				if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
				or input.UserInputType == Enum.UserInputType.Touch) then update(input) end
			end)
			UserInputService.InputEnded:Connect(function(input)
				if input.UserInputType == Enum.UserInputType.MouseButton1
				or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
			end)
		end

		makeTitle("AIM LOCK", 8)

		local scroll = Instance.new("ScrollingFrame")
		scroll.Position = UDim2.fromOffset(0,32)
		scroll.Size = UDim2.new(1,0,1,-32)
		scroll.BackgroundTransparency = 1
		scroll.BorderSizePixel = 0
		scroll.ScrollBarThickness = 4
		scroll.CanvasSize = UDim2.new(0,0,0,480)
		scroll.Parent = page

		makeToggle("Ativado",          8,  Config.Enabled,    function(v) Config.Enabled=v end)
		makeToggle("Auto Lock",        36, Config.AutoLock,   function(v) Config.AutoLock=v end)
		makeToggle("Prediction",       64, Config.Prediction>0, function(v) Config.Prediction = v and 0.15 or 0 end)
		makeToggle("Mostrar FOV",      92, Config.ShowFOV,    function(v) Config.ShowFOV=v fovFrame.Visible=v end)
		makeToggle("Highlight",        120,Config.Highlight,  function(v) Config.Highlight=v end)
		makeToggle("Checar Time",      148,Config.TeamCheck,  function(v) Config.TeamCheck=v end)

		makeSlider("FOV",            184,20,500,Config.FOV, function(v)
			Config.FOV=v fovFrame.Size=UDim2.fromOffset(v*2,v*2)
		end)
		makeSlider("Smooth x100",    224,0,95,math.floor(Config.Smooth*100), function(v) Config.Smooth=v/100 end)
		makeSlider("Prediction x100",264,0,80,math.floor(Config.Prediction*100), function(v) Config.Prediction=v/100 end)
		makeSlider("DragOut px",     304,5,150,Config.DragOutDist, function(v) Config.DragOutDist=v end)

		local info = Instance.new("TextLabel")
		info.BackgroundTransparency = 1
		info.Position = UDim2.fromOffset(9,340)
		info.Size = UDim2.new(1,-18,0,80)
		info.Font = Enum.Font.Gotham
		info.TextSize = 10
		info.TextColor3 = ctx.colors.SUBTEXT
		info.TextXAlignment = Enum.TextXAlignment.Left
		info.TextYAlignment = Enum.TextYAlignment.Top
		info.TextWrapped = true
		info.Text = "PC: ["..Config.ToggleBindPC.Name.."] toggle · ["..Config.BindPC.Name.."] travar (auto off)\n"
			.. "Arraste o mouse pra fora do alvo pra soltar.\n"
			.. "Nunca mira em quem tá atrás de parede."
		info.Parent = scroll
	end
})

if not ok then warn("[AimLock] registro falhou:", err) end
