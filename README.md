---====== Fluster FULL ======---
-- sobe + pedras | sala vermelha | range 15 | 10 dano / 1s
-- morte → jumpscare (rosto + static) | 1/45 ao abrir porta

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local camera = Workspace.CurrentCamera

local MODEL_ID = 106247629671516
local JUMP_FACE = "rbxassetid://132929275840201"
local STATIC1 = "rbxassetid://236542974"
local STATIC2 = "rbxassetid://184251462"

local DAMAGE = 10
local DAMAGE_CD = 1
local TOUCH_RANGE = 15
local HEIGHT = 1.2
local SINK_DEPTH = 45
local RISE_FROM = 18
local CHANCE = 45 -- aparece bem mais (antes 666)
local MIN_ROOM = 5

local entityModel = nil
local active = false
local lastHit = 0
local touchConn = nil
local doorConn = nil
local redCC = nil
local oldFogEnd, oldAmbient, oldBrightness
local killing = false

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHum()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function getRoom(val)
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil end
	val = val or ReplicatedStorage.GameData.LatestRoom.Value
	return rooms:FindFirstChild(tostring(val)) or rooms:FindFirstChild(val)
end

local function getRoomCenter(room)
	if not room then return nil end
	local entrance = room:FindFirstChild("RoomEntrance")
	local exit = room:FindFirstChild("RoomExit")
	if entrance and exit and entrance:IsA("BasePart") and exit:IsA("BasePart") then
		return entrance.Position:Lerp(exit.Position, 0.5) + Vector3.new(0, HEIGHT, 0)
	end
	local sum, n = Vector3.zero, 0
	for _, v in ipairs(room:GetDescendants()) do
		if v:IsA("BasePart") then
			local name = v.Name:lower()
			if not name:find("door") and not name:find("wall") and not name:find("ceil") then
				sum += v.Position
				n += 1
			end
		end
	end
	if n > 0 then return sum / n + Vector3.new(0, HEIGHT, 0) end
	return nil
end

local function loadModel()
	local model
	pcall(function()
		model = game:GetObjects("rbxassetid://" .. tostring(MODEL_ID))[1]
	end)
	if not model then
		pcall(function()
			local asset = game:GetService("InsertService"):LoadAsset(MODEL_ID)
			model = asset:GetChildren()[1]
		end)
	end
	return model
end

local function startRed()
	oldFogEnd = Lighting.FogEnd
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness
	Lighting.FogColor = Color3.fromRGB(80, 0, 0)
	Lighting.FogEnd = 90
	Lighting.Ambient = Color3.fromRGB(90, 15, 15)
	Lighting.Brightness = 0.45
	redCC = Lighting:FindFirstChild("FlusterRed")
	if not redCC then
		redCC = Instance.new("ColorCorrectionEffect")
		redCC.Name = "FlusterRed"
		redCC.Parent = Lighting
	end
	redCC.TintColor = Color3.fromRGB(255, 70, 70)
	redCC.Saturation = 0.15
	redCC.Contrast = 0.12
end

local function stopRed()
	if redCC then
		TweenService:Create(redCC, TweenInfo.new(1.2), {
			TintColor = Color3.fromRGB(255, 255, 255),
			Saturation = 0,
			Contrast = 0
		}):Play()
		task.delay(1.3, function()
			if redCC then redCC:Destroy() redCC = nil end
		end)
	end
	pcall(function()
		if oldFogEnd then Lighting.FogEnd = oldFogEnd end
		if oldAmbient then Lighting.Ambient = oldAmbient end
		if oldBrightness then Lighting.Brightness = oldBrightness end
		Lighting.FogColor = Color3.fromRGB(191, 191, 191)
	end)
end

local function brownRocks(center)
	for i = 1, 14 do
		local rock = Instance.new("Part")
		rock.Name = "FlusterRock"
		rock.Size = Vector3.new(math.random(10, 22) / 10, math.random(8, 16) / 10, math.random(10, 22) / 10)
		rock.Color = Color3.fromRGB(math.random(95, 130), math.random(55, 80), math.random(30, 50))
		rock.Material = Enum.Material.Ground
		rock.Anchored = false
		rock.CanCollide = false
		rock.Position = center + Vector3.new((math.random() - 0.5) * 7, 0.2, (math.random() - 0.5) * 7)
		rock.Orientation = Vector3.new(math.random(0, 360), math.random(0, 360), math.random(0, 360))
		rock.Parent = Workspace
		local bv = Instance.new("BodyVelocity")
		bv.MaxForce = Vector3.new(1e5, 1e5, 1e5)
		bv.Velocity = Vector3.new((math.random() - 0.5) * 22, math.random(14, 32), (math.random() - 0.5) * 22)
		bv.Parent = rock
		Debris:AddItem(bv, 0.3)
		Debris:AddItem(rock, 2.5)
	end
	local s = Instance.new("Sound", Workspace)
	s.SoundId = "rbxassetid://9118614058"
	s.Volume = 3.2
	s:Play()
	Debris:AddItem(s, 5)
end

-- ========== JUMPSCARE (igual vídeo) ==========
local function playJumpscare()
	if killing then return end
	killing = true

	pcall(function()
		local old = playerGui:FindFirstChild("FlusterJumpscare")
		if old then old:Destroy() end
	end)

	local gui = Instance.new("ScreenGui")
	gui.Name = "FlusterJumpscare"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 999
	gui.ResetOnSpawn = false
	gui.Parent = playerGui

	-- fundo estático 1
	local staticBg = Instance.new("ImageLabel")
	staticBg.Name = "Static"
	staticBg.Size = UDim2.new(1, 0, 1, 0)
	staticBg.BackgroundColor3 = Color3.new(0, 0, 0)
	staticBg.BorderSizePixel = 0
	staticBg.Image = STATIC1
	staticBg.ScaleType = Enum.ScaleType.Tile
	staticBg.TileSize = UDim2.new(0, 280, 0, 280)
	staticBg.ImageTransparency = 0.15
	staticBg.ZIndex = 1
	staticBg.Parent = gui

	-- overlay estático 2 (mais glitch)
	local static2 = Instance.new("ImageLabel")
	static2.Name = "Static2"
	static2.Size = UDim2.new(1.2, 0, 1.2, 0)
	static2.Position = UDim2.new(-0.1, 0, -0.1, 0)
	static2.BackgroundTransparency = 1
	static2.Image = STATIC2
	static2.ScaleType = Enum.ScaleType.Tile
	static2.TileSize = UDim2.new(0, 200, 0, 200)
	static2.ImageTransparency = 0.35
	static2.ZIndex = 2
	static2.Parent = gui

	-- rosto grande no centro
	local face = Instance.new("ImageLabel")
	face.Name = "Face"
	face.AnchorPoint = Vector2.new(0.5, 0.5)
	face.Position = UDim2.new(0.5, 0, 0.48, 0)
	face.Size = UDim2.new(0.95, 0, 0.95, 0)
	face.BackgroundTransparency = 1
	face.Image = JUMP_FACE
	face.ScaleType = Enum.ScaleType.Fit
	face.ImageTransparency = 0
	face.ZIndex = 5
	face.Parent = gui

	-- som estourado
	local sfx = Instance.new("Sound", Workspace)
	sfx.SoundId = "rbxassetid://9125472062"
	sfx.Volume = 5
	sfx.PlaybackSpeed = 0.85
	sfx:Play()
	Debris:AddItem(sfx, 8)

	-- anima: treme + troca static + zoom no rosto
	local t0 = tick()
	local duration = 2.8
	while tick() - t0 < duration do
		local a = (tick() - t0) / duration
		-- troca static
		if math.floor(tick() * 12) % 2 == 0 then
			staticBg.Image = STATIC1
		else
			staticBg.Image = STATIC2
		end
		static2.Position = UDim2.new(
			-0.1 + (math.random() - 0.5) * 0.08,
			0,
			-0.1 + (math.random() - 0.5) * 0.08,
			0
		)
		-- zoom / shake no rosto
		local shake = (math.random() - 0.5) * 0.04
		local size = 0.85 + a * 0.35 + (math.random() - 0.5) * 0.06
		face.Size = UDim2.new(size, 0, size, 0)
		face.Position = UDim2.new(0.5 + shake, 0, 0.48 + shake * 0.5, 0)
		face.ImageTransparency = math.clamp(a * 0.15, 0, 0.3)

		-- treme câmera
		camera.CFrame = camera.CFrame * CFrame.new(
			(math.random() - 0.5) * 0.8,
			(math.random() - 0.5) * 0.8,
			0
		)
		RunService.RenderStepped:Wait()
	end

	-- dissolve pixelado no final
	local t1 = tick()
	while tick() - t1 < 0.6 do
		local b = (tick() - t1) / 0.6
		face.ImageTransparency = b
		staticBg.ImageTransparency = 0.15 + b * 0.85
		static2.ImageTransparency = 0.35 + b * 0.65
		RunService.RenderStepped:Wait()
	end

	gui:Destroy()
end

local function sinkAndDestroy()
	if not entityModel or not entityModel.Parent then
		active = false
		stopRed()
		return
	end
	if touchConn then touchConn:Disconnect() touchConn = nil end
	if doorConn then doorConn:Disconnect() doorConn = nil end

	local primary = entityModel.PrimaryPart or entityModel:FindFirstChildWhichIsA("BasePart", true)
	local center = primary and primary.Position or Vector3.zero
	brownRocks(center)
	stopRed()

	if primary then
		local start = entityModel:GetPivot()
		local goalPos = start.Position + Vector3.new(0, -SINK_DEPTH, 0)
		task.spawn(function()
			local t0 = tick()
			local dur = 1.8
			while tick() - t0 < dur and entityModel and entityModel.Parent do
				local a = (tick() - t0) / dur
				local pos = start.Position:Lerp(goalPos, a * a)
				pcall(function()
					entityModel:PivotTo(CFrame.new(pos) * (start - start.Position))
				end)
				RunService.Heartbeat:Wait()
			end
			if entityModel then entityModel:Destroy() entityModel = nil end
			active = false
		end)
	else
		if entityModel then entityModel:Destroy() end
		entityModel = nil
		active = false
	end
end

local function startTouchLoop()
	if touchConn then touchConn:Disconnect() end
	lastHit = 0
	touchConn = RunService.Heartbeat:Connect(function()
		if not entityModel or not entityModel.Parent or killing then return end
		local root = getHRP()
		local hum = getHum()
		local primary = entityModel.PrimaryPart or entityModel:FindFirstChildWhichIsA("BasePart", true)
		if not root or not hum or not primary or hum.Health <= 0 then return end

		if (primary.Position - root.Position).Magnitude <= TOUCH_RANGE then
			local now = tick()
			if now - lastHit >= DAMAGE_CD then
				lastHit = now
				local before = hum.Health
				hum:TakeDamage(DAMAGE)
				print("Fluster → -" .. DAMAGE)
				-- se matou → jumpscare
				if before > 0 and (hum.Health <= 0 or hum.Health < before and hum.Health <= DAMAGE) then
					task.spawn(playJumpscare)
					task.delay(0.1, function()
						if hum and hum.Health > 0 then
							hum.Health = 0
						end
					end)
				end
			end
		end
	end)
end

local function riseFromGround(model, finalPos)
	local startPos = finalPos + Vector3.new(0, -RISE_FROM, 0)
	model:PivotTo(CFrame.new(startPos))
	brownRocks(finalPos)
	local t0 = tick()
	local dur = 1.35
	while tick() - t0 < dur and model and model.Parent do
		local a = (tick() - t0) / dur
		local ease = 1 - (1 - a) * (1 - a)
		pcall(function()
			model:PivotTo(CFrame.new(startPos:Lerp(finalPos, ease)))
		end)
		RunService.Heartbeat:Wait()
	end
	pcall(function() model:PivotTo(CFrame.new(finalPos)) end)
end

local function spawnFluster()
	if active then return end
	killing = false

	local room = getRoom()
	local center = getRoomCenter(room)
	if not center then
		local hrp = getHRP()
		if not hrp then return end
		center = hrp.Position + hrp.CFrame.LookVector * 12 + Vector3.new(0, HEIGHT, 0)
	end

	local model = loadModel()
	if not model then
		warn("Fluster: falha modelo")
		return
	end

	for _, o in ipairs(model:GetDescendants()) do
		if o:IsA("Script") or o:IsA("LocalScript") then
			o:Destroy()
		elseif o:IsA("BasePart") then
			o.Anchored = true
			o.CanCollide = false
			o.Massless = true
		end
	end

	if not model.PrimaryPart then
		model.PrimaryPart = model:FindFirstChildWhichIsA("BasePart", true)
	end
	if not model.PrimaryPart then
		model:Destroy()
		return
	end

	model.Name = "Fluster"
	model.Parent = Workspace
	entityModel = model
	active = true

	startRed()
	riseFromGround(model, center)
	startTouchLoop()

	if doorConn then doorConn:Disconnect() end
	doorConn = ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function()
		if active and not killing then
			sinkAndDestroy()
		end
	end)

	print("Fluster subiu!")
end

task.spawn(function()
	local gd = ReplicatedStorage:WaitForChild("GameData", 40)
	if not gd then return end
	local latest = gd:WaitForChild("LatestRoom", 20)
	if not latest then return end

	latest.Changed:Connect(function(val)
		if active then return end
		local num = tonumber(val) or 0
		if num < MIN_ROOM then return end
		if math.random(1, CHANCE) == 1 then
			print("Fluster 1/" .. CHANCE .. " porta " .. tostring(val))
			task.wait(0.8)
			if not active then spawnFluster() end
		end
	end)
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/fluster" then
		if active and entityModel then
			sinkAndDestroy()
			task.wait(0.5)
		end
		spawnFluster()
	end
end)

print("✅ Fluster FULL | jumpscare static | 1/" .. CHANCE .. " | /fluster")
