-- Coord Tracker (меню для телефона)
-- Показывает твою координату, копирует её, ведёт историю,
-- ставит точки в мире и соединяет их линиями по порядку копирования.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local player = Players.LocalPlayer

-- ===== Настройки =====
local DECIMALS = 1           -- знаков после запятой
local POINT_SIZE = 1.2
local LINE_THICKNESS = 0.25

-- ===== Подготовка =====
local parent = (gethui and gethui()) or player:WaitForChild("PlayerGui")
local old = parent:FindFirstChild("CoordTrackerGui")
if old then old:Destroy() end
local oldFolder = workspace.CurrentCamera and workspace.CurrentCamera:FindFirstChild("CoordTrackerPoints")
if oldFolder then oldFolder:Destroy() end

local connections = {}
local alive = true

local pointsFolder = Instance.new("Folder")
pointsFolder.Name = "CoordTrackerPoints"
pointsFolder.Parent = workspace.CurrentCamera -- локально, другие игроки не видят

local gui = Instance.new("ScreenGui")
gui.Name = "CoordTrackerGui"
gui.ResetOnSpawn = false
gui.DisplayOrder = 999
gui.Parent = parent

local function mk(class, props, par)
	local o = Instance.new(class)
	for k, v in pairs(props) do o[k] = v end
	o.Parent = par
	return o
end
local function round(o, r) Instance.new("UICorner", o).CornerRadius = UDim.new(0, r or 6) end

-- ===== Меню =====
local main = mk("Frame", {
	Size = UDim2.new(0, 230, 0, 262), Position = UDim2.new(0, 10, 0.3, 0),
	BackgroundColor3 = Color3.fromRGB(28, 28, 36), BorderSizePixel = 0, ClipsDescendants = true,
}, gui)
round(main, 8)

local bar = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 24), BackgroundColor3 = Color3.fromRGB(40, 40, 52), BorderSizePixel = 0,
}, main)
local title = mk("TextLabel", {
	Size = UDim2.new(1, -34, 1, 0), Position = UDim2.new(0, 8, 0, 0), BackgroundTransparency = 1,
	Text = "Coord Tracker", TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold,
	TextSize = 12, TextXAlignment = Enum.TextXAlignment.Left,
}, bar)
local minBtn = mk("TextButton", {
	Size = UDim2.new(0, 28, 0, 24), Position = UDim2.new(1, -28, 0, 0), BackgroundTransparency = 1,
	Text = "–", TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = 16,
}, bar)

local body = mk("Frame", {
	Size = UDim2.new(1, 0, 1, -24), Position = UDim2.new(0, 0, 0, 24), BackgroundTransparency = 1,
}, main)

local coordLabel = mk("TextLabel", {
	Size = UDim2.new(1, -12, 0, 44), Position = UDim2.new(0, 6, 0, 6),
	BackgroundColor3 = Color3.fromRGB(18, 18, 24), TextColor3 = Color3.fromRGB(140, 255, 160),
	Font = Enum.Font.Code, TextSize = 13, Text = "X: --\nY: --\nZ: --",
	TextXAlignment = Enum.TextXAlignment.Left,
}, body)
round(coordLabel, 5)
Instance.new("UIPadding", coordLabel).PaddingLeft = UDim.new(0, 8)

local copyBtn = mk("TextButton", {
	Size = UDim2.new(1, -12, 0, 32), Position = UDim2.new(0, 6, 0, 56),
	BackgroundColor3 = Color3.fromRGB(50, 150, 90), Text = "Копировать координату",
	TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = 12,
}, body)
round(copyBtn, 6)

mk("TextLabel", {
	Size = UDim2.new(1, -12, 0, 14), Position = UDim2.new(0, 8, 0, 92), BackgroundTransparency = 1,
	Text = "История (тап = копировать ещё раз)", TextColor3 = Color3.fromRGB(160, 160, 175),
	Font = Enum.Font.Gotham, TextSize = 10, TextXAlignment = Enum.TextXAlignment.Left,
}, body)

local list = mk("ScrollingFrame", {
	Size = UDim2.new(1, -12, 0, 90), Position = UDim2.new(0, 6, 0, 108),
	BackgroundColor3 = Color3.fromRGB(36, 36, 46), BorderSizePixel = 0, ScrollBarThickness = 3,
	CanvasSize = UDim2.new(0, 0, 0, 0), AutomaticCanvasSize = Enum.AutomaticSize.Y,
}, body)
round(list, 5)
local layout = Instance.new("UIListLayout", list)
layout.Padding = UDim.new(0, 2)
layout.SortOrder = Enum.SortOrder.LayoutOrder

local clearBtn = mk("TextButton", {
	Size = UDim2.new(0.5, -9, 0, 30), Position = UDim2.new(0, 6, 0, 204),
	BackgroundColor3 = Color3.fromRGB(200, 130, 40), Text = "Очистить точки",
	TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = 11,
}, body)
round(clearBtn, 6)

local offBtn = mk("TextButton", {
	Size = UDim2.new(0.5, -9, 0, 30), Position = UDim2.new(0.5, 3, 0, 204),
	BackgroundColor3 = Color3.fromRGB(180, 60, 60), Text = "Выключить",
	TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.GothamBold, TextSize = 11,
}, body)
round(offBtn, 6)

-- ===== Перетаскивание =====
do
	local dragging, startPos, startInput
	bar.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseButton1 then
			dragging, startPos, startInput = true, main.Position, i.Position
			i.Changed:Connect(function()
				if i.UserInputState == Enum.UserInputState.End then dragging = false end
			end)
		end
	end)
	table.insert(connections, UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.Touch or i.UserInputType == Enum.UserInputType.MouseMovement) then
			local d = i.Position - startInput
			main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
		end
	end))
end

-- ===== Логика =====
local function fmt(n)
	return string.format("%." .. DECIMALS .. "f", n)
end

local function getPos()
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	return root and root.Position
end

local function vecCode(p)
	return string.format("Vector3.new(%s, %s, %s)", fmt(p.X), fmt(p.Y), fmt(p.Z))
end

local function setStatus(t, revert)
	title.Text = t
	if revert then
		task.delay(1.2, function()
			if alive then title.Text = "Coord Tracker" end
		end)
	end
end

local function copyText(text)
	local c = setclipboard or toclipboard or (Clipboard and Clipboard.set)
	if c then
		pcall(c, text)
		setStatus("Скопировано", true)
	else
		setStatus("Нет буфера обмена", true)
	end
end

-- живое обновление координат
local acc = 0
table.insert(connections, RunService.Heartbeat:Connect(function(dt)
	acc += dt
	if acc < 0.1 then return end
	acc = 0
	local p = getPos()
	if p then
		coordLabel.Text = "X: " .. fmt(p.X) .. "\nY: " .. fmt(p.Y) .. "\nZ: " .. fmt(p.Z)
	else
		coordLabel.Text = "X: --\nY: --\nZ: --"
	end
end))

-- точки и линии
local points = {}   -- позиции Vector3
local count = 0

local function addPoint(pos)
	count += 1
	local n = count
	table.insert(points, pos)

	local ball = Instance.new("Part")
	ball.Name = "Point" .. n
	ball.Shape = Enum.PartType.Ball
	ball.Size = Vector3.new(POINT_SIZE, POINT_SIZE, POINT_SIZE)
	ball.Position = pos
	ball.Anchored = true
	ball.CanCollide = false
	ball.CanQuery = false
	ball.CanTouch = false
	ball.CastShadow = false
	ball.Material = Enum.Material.Neon
	ball.Color = Color3.fromHSV(((n - 1) * 0.13) % 1, 0.8, 1)
	ball.Parent = pointsFolder

	local bb = Instance.new("BillboardGui")
	bb.Size = UDim2.new(0, 30, 0, 20)
	bb.StudsOffset = Vector3.new(0, 1.8, 0)
	bb.AlwaysOnTop = true
	bb.Parent = ball
	local tl = Instance.new("TextLabel")
	tl.Size = UDim2.new(1, 0, 1, 0)
	tl.BackgroundTransparency = 0.4
	tl.BackgroundColor3 = Color3.new(0, 0, 0)
	tl.Text = tostring(n)
	tl.TextColor3 = Color3.new(1, 1, 1)
	tl.Font = Enum.Font.GothamBold
	tl.TextSize = 12
	tl.Parent = bb
	Instance.new("UICorner", tl).CornerRadius = UDim.new(0, 4)

	-- линия к предыдущей точке
	if n > 1 then
		local prev = points[n - 1]
		local dist = (pos - prev).Magnitude
		if dist > 0.01 then
			local line = Instance.new("Part")
			line.Name = "Line" .. n
			line.Anchored = true
			line.CanCollide = false
			line.CanQuery = false
			line.CanTouch = false
			line.CastShadow = false
			line.Material = Enum.Material.Neon
			line.Color = Color3.fromRGB(255, 255, 255)
			line.Size = Vector3.new(LINE_THICKNESS, LINE_THICKNESS, dist)
			line.CFrame = CFrame.lookAt(prev, pos) * CFrame.new(0, 0, -dist / 2)
			line.Parent = pointsFolder
		end
	end
	return n
end

local function addHistory(n, pos)
	local code = vecCode(pos)
	local item = mk("TextButton", {
		Size = UDim2.new(1, -6, 0, 22), BackgroundColor3 = Color3.fromRGB(52, 52, 66),
		Text = n .. ". " .. fmt(pos.X) .. ", " .. fmt(pos.Y) .. ", " .. fmt(pos.Z),
		TextColor3 = Color3.new(1, 1, 1), Font = Enum.Font.Code, TextSize = 11,
		TextTruncate = Enum.TextTruncate.AtEnd, LayoutOrder = -n,
	}, list)
	round(item, 4)
	item.Activated:Connect(function() copyText(code) end)
end

copyBtn.Activated:Connect(function()
	local p = getPos()
	if not p then setStatus("Нет персонажа", true) return end
	copyText(vecCode(p))
	local n = addPoint(p)
	addHistory(n, p)
end)

local function clearAll()
	pointsFolder:ClearAllChildren()
	for _, c in ipairs(list:GetChildren()) do
		if c:IsA("TextButton") then c:Destroy() end
	end
	points, count = {}, 0
end

clearBtn.Activated:Connect(function()
	clearAll()
	setStatus("Точки очищены", true)
end)

local minimized = false
minBtn.Activated:Connect(function()
	minimized = not minimized
	body.Visible = not minimized
	main.Size = minimized and UDim2.new(0, 230, 0, 24) or UDim2.new(0, 230, 0, 262)
	minBtn.Text = minimized and "+" or "–"
end)

-- полное выключение
offBtn.Activated:Connect(function()
	alive = false
	for _, c in ipairs(connections) do pcall(function() c:Disconnect() end) end
	pointsFolder:Destroy()
	gui:Destroy()
end)
