-- Services
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Settings
local InfiniteJumpEnabled = true
local ESPEnabled = true
local ShowTracer = true
local TeamCheck = true
local ShowHealthBar = true          -- เปิด/ปิด Health Bar

-- สี
local EnemyColor = Color3.fromRGB(255, 50, 50)
local AllyColor = Color3.fromRGB(50, 255, 100)
local NeutralColor = Color3.fromRGB(255, 255, 50)

-- ============ Infinite Jump ============
UserInputService.JumpRequest:Connect(function()
	if InfiniteJumpEnabled then
		local character = LocalPlayer.Character
		if character then
			local humanoid = character:FindFirstChildOfClass("Humanoid")
			if humanoid then
				humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
			end
		end
	end
end)

-- ============ ESP + Tracer + Health Bar ============
local function GetTeamColor(player)
	if not TeamCheck then
		return EnemyColor
	end

	if not LocalPlayer.Team then
		return NeutralColor
	end

	if player.Team == LocalPlayer.Team then
		return AllyColor
	else
		return EnemyColor
	end
end

local function CreateESP(player)
	if player == LocalPlayer then return end

	-- Box
	local box = Drawing.new("Square")
	box.Thickness = 1.5
	box.Filled = false
	box.Transparency = 1
	box.Visible = false

	-- Name
	local nameTag = Drawing.new("Text")
	nameTag.Size = 15
	nameTag.Center = true
	nameTag.Outline = true
	nameTag.Color = Color3.fromRGB(255, 255, 255)
	nameTag.Visible = false

	-- Distance
	local distanceTag = Drawing.new("Text")
	distanceTag.Size = 13
	distanceTag.Center = true
	distanceTag.Outline = true
	distanceTag.Color = Color3.fromRGB(220, 220, 220)
	distanceTag.Visible = false

	-- Tracer
	local tracer = Drawing.new("Line")
	tracer.Thickness = 1.5
	tracer.Transparency = 1
	tracer.Visible = false

	-- Health Bar (พื้นหลัง)
	local healthBackground = Drawing.new("Square")
	healthBackground.Filled = true
	healthBackground.Thickness = 0
	healthBackground.Color = Color3.fromRGB(30, 30, 30)
	healthBackground.Transparency = 0.3
	healthBackground.Visible = false

	-- Health Bar (เลือด)
	local healthBar = Drawing.new("Square")
	healthBar.Filled = true
	healthBar.Thickness = 0
	healthBar.Transparency = 1
	healthBar.Visible = false

	local connection
	connection = RunService.RenderStepped:Connect(function()
		if not ESPEnabled or not player.Character then
			box.Visible = false
			nameTag.Visible = false
			distanceTag.Visible = false
			tracer.Visible = false
			healthBackground.Visible = false
			healthBar.Visible = false
			return
		end

		local character = player.Character
		local humanoid = character:FindFirstChildOfClass("Humanoid")
		local rootPart = character:FindFirstChild("HumanoidRootPart")

		if not humanoid or not rootPart or humanoid.Health <= 0 then
			box.Visible = false
			nameTag.Visible = false
			distanceTag.Visible = false
			tracer.Visible = false
			healthBackground.Visible = false
			healthBar.Visible = false
			return
		end

		local color = GetTeamColor(player)
		box.Color = color
		tracer.Color = color

		local rootPos, onScreen = Camera:WorldToViewportPoint(rootPart.Position)
		local head = character:FindFirstChild("Head")
		local headPos = head and Camera:WorldToViewportPoint(head.Position) or rootPos

		-- ระยะทาง
		local distance = 0
		if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
			distance = math.floor((rootPart.Position - LocalPlayer.Character.HumanoidRootPart.Position).Magnitude)
		end

		if onScreen then
			-- Box
			local size = Vector2.new(1600 / rootPos.Z, 2700 / rootPos.Z)
			box.Size = size
			box.Position = Vector2.new(rootPos.X - size.X / 2, rootPos.Y - size.Y / 2)
			box.Visible = true

			-- Name
			nameTag.Text = player.Name
			nameTag.Position = Vector2.new(rootPos.X, headPos.Y - 18)
			nameTag.Visible = true

			-- Distance
			distanceTag.Text = distance .. "m"
			distanceTag.Position = Vector2.new(rootPos.X, rootPos.Y + size.Y / 2 + 4)
			distanceTag.Visible = true

			-- Health Bar
			if ShowHealthBar then
				local healthPercent = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
				local barWidth = size.X
				local barHeight = 4
				local barX = rootPos.X - barWidth / 2
				local barY = rootPos.Y + size.Y / 2 + 18   -- อยู่ใต้ระยะทาง

				-- พื้นหลัง
				healthBackground.Size = Vector2.new(barWidth, barHeight)
				healthBackground.Position = Vector2.new(barX, barY)
				healthBackground.Visible = true

				-- เลือด (เปลี่ยนสีตาม % เลือด)
				local healthColor
				if healthPercent > 0.6 then
					healthColor = Color3.fromRGB(50, 255, 80)      -- เขียว
				elseif healthPercent > 0.3 then
					healthColor = Color3.fromRGB(255, 200, 50)     -- เหลือง
				else
					healthColor = Color3.fromRGB(255, 50, 50)      -- แดง
				end

				healthBar.Color = healthColor
				healthBar.Size = Vector2.new(barWidth * healthPercent, barHeight)
				healthBar.Position = Vector2.new(barX, barY)
				healthBar.Visible = true
			else
				healthBackground.Visible = false
				healthBar.Visible = false
			end
		else
			box.Visible = false
			nameTag.Visible = false
			distanceTag.Visible = false
			healthBackground.Visible = false
			healthBar.Visible = false
		end

		-- Tracer
		if ShowTracer then
			local fromPos = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
			local toPos = Vector2.new(rootPos.X, rootPos.Y)

			tracer.From = fromPos
			tracer.To = toPos
			tracer.Visible = true
		else
			tracer.Visible = false
		end
	end)

	-- ลบเมื่อผู้เล่นออก
	player.AncestryChanged:Connect(function()
		if not player:IsDescendantOf(Players) then
			connection:Disconnect()
			box:Remove()
			nameTag:Remove()
			distanceTag:Remove()
			tracer:Remove()
			healthBackground:Remove()
			healthBar:Remove()
		end
	end)
end

-- สร้าง ESP ให้ผู้เล่นที่มีอยู่ + ผู้เล่นใหม่
for _, player in ipairs(Players:GetPlayers()) do
	CreateESP(player)
end

Players.PlayerAdded:Connect(CreateESP)
