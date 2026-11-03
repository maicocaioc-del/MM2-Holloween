-- LocalScript (StarterGui ou StarterPlayerScripts)
-- Interface leve do MM2 Halloween. Os callbacks ficam vazios para você ligar à sua lógica.

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- Estado / callbacks ---------------------------------------------------------
local State = {
	AutoFarm = false, AutoWin = false, XPFarm = false,
	AutoRespawn = false, AntiFling = false, FarmSpeed = 25,
}
local Callbacks = {
	AutoFarm = function(on) end,
	AutoWin = function(on) end,
	XPFarm = function(on) end,
	AutoRespawn = function(on) end,
	AntiFling = function(on) end,
	FarmSpeed = function(value) end,
}

-- Chama o callback sem deixar um erro seu quebrar a interface
local function fire(key, value)
	local ok, err = pcall(Callbacks[key], value)
	if not ok then warn("[MM2UI] erro em " .. key .. ": " .. tostring(err)) end
end

-- Saco de moedas: monitora o remote ReplicatedStorage.Remotes.Gameplay.CoinCollected --
local MAX_COINS = 50  -- tamanho do saco, usado se o remote não informar o máximo
local DEBUG = false   -- true = imprime no console tudo que o CoinCollected manda

local Bag = { count = 0, max = MAX_COINS, full = false }
local onBagFull, onBagReset -- definidos no fim do arquivo

local function setBag(count, max)
	Bag.count = count
	if max then Bag.max = max end
	local nowFull = Bag.count >= Bag.max
	if nowFull and not Bag.full then
		Bag.full = true
		task.spawn(onBagFull)
	elseif not nowFull and Bag.full then
		Bag.full = false
		task.spawn(onBagReset)
	end
end

local function resetBag()
	setBag(0)
end

-- O remote costuma mandar (tipo, atual, máximo). Se vier assim, usa os números do jogo;
-- se vier diferente, conta +1 por moeda coletada.
local function onCoinCollected(...)
	if DEBUG then print("[MM2UI] CoinCollected:", ...) end
	local nums = {}
	for _, v in ipairs({ ... }) do
		if type(v) == "number" then nums[#nums + 1] = v end
	end
	local cur, max = nums[1], nums[2]
	if cur and max and max >= 1 and cur >= 0 and cur <= max then
		setBag(cur, max)
	else
		setBag(Bag.count + 1)
	end
end

-- Novo round -> saco zera
local ROUND_START_REMOTES = { "RoundStart", "LoadingMap" }

task.spawn(function()
	local remotes = ReplicatedStorage:WaitForChild("Remotes", 15)
	local gameplay = remotes and remotes:WaitForChild("Gameplay", 15)
	local coinRemote = gameplay and gameplay:WaitForChild("CoinCollected", 15)
	if not coinRemote then
		warn("[MM2UI] Remotes.Gameplay.CoinCollected não encontrado")
		return
	end
	coinRemote.OnClientEvent:Connect(onCoinCollected)

	for _, name in ipairs(ROUND_START_REMOTES) do
		local r = gameplay:FindFirstChild(name)
		if r and r:IsA("RemoteEvent") then
			r.OnClientEvent:Connect(resetBag)
		end
	end
end)

-- O mapa do novo round cria um CoinContainer novo
workspace.DescendantAdded:Connect(function(d)
	if d.Name == "CoinContainer" then resetBag() end
end)

-- Auto Farm: vai por BAIXO do mapa até as moedas (Serve_coin / Coin_Server) ------
local COIN_NAMES = { Coin_Server = true, Serve_coin = true } -- nomes das moedas
local SPEED_STEP = 1       -- cada ponto da barrinha = 1 stud/seg (1..50)
local UNDER_OFFSET = 1.4   -- studs abaixo da moeda

local function farmSpeed()
	return State.FarmSpeed * SPEED_STEP
end

-- Boneco deitado (horizontal) na posição pos
local function lyingCF(pos)
	return CFrame.new(pos) * CFrame.Angles(math.pi / 2, 0, 0)
end

local farmConn, noclipConn
local engaged, underground = false, false -- engaged = boneco deitado por baixo do mapa
local target, deadline, nextScan = nil, 0, 0
local currentChar, savedCF = nil, nil
local coinContainer
local skipped = setmetatable({}, { __mode = "k" })
local noclipped = {}

local function getContainer()
	if coinContainer and coinContainer.Parent then return coinContainer end
	coinContainer = nil
	for _, d in ipairs(workspace:GetDescendants()) do
		if d.Name == "CoinContainer" then coinContainer = d break end
	end
	return coinContainer
end

local function pickCoin(from)
	local best, bestDist = nil, math.huge
	for _, d in ipairs((getContainer() or workspace):GetDescendants()) do
		if COIN_NAMES[d.Name] and d:IsA("BasePart") and not skipped[d] then
			local dist = (d.Position - from).Magnitude
			if dist < bestDist then best, bestDist = d, dist end
		end
	end
	return best
end

local function setAnimations(char, hum, enabled)
	local animate = char:FindFirstChild("Animate")
	if animate then animate.Disabled = not enabled end
	if not enabled then
		for _, t in ipairs(hum:GetPlayingAnimationTracks()) do t:Stop(0) end
	end
end

-- Só entra no modo "por baixo do mapa" quando existe moeda para pegar
local function engage(root)
	engaged, underground = true, false
	savedCF = root.CFrame
	if not noclipConn then
		noclipConn = RunService.Stepped:Connect(function()
			local c = player.Character
			if not (engaged and c) then return end
			for _, part in ipairs(c:GetDescendants()) do
				if part:IsA("BasePart") then
					if part.CanCollide then noclipped[part] = true end
					part.CanCollide = false
				end
			end
		end)
	end
end

-- Sai do modo: boneco em pé, com animação e colisão, na posição de antes
local function release(char, root, hum)
	if not engaged then return end
	engaged, underground = false, false
	if hum then
		hum.PlatformStand = false
		setAnimations(char, hum, true)
	end
	for part in pairs(noclipped) do
		if part.Parent then part.CanCollide = true end
	end
	noclipped = {}
	if root and savedCF then
		root.AssemblyLinearVelocity = Vector3.zero
		root.AssemblyAngularVelocity = Vector3.zero
		root.CFrame = savedCF
	end
	savedCF = nil
end

local function stopFarm()
	if farmConn then farmConn:Disconnect(); farmConn = nil end
	local char = player.Character
	if char and char == currentChar then
		release(char, char:FindFirstChild("HumanoidRootPart"), char:FindFirstChildOfClass("Humanoid"))
	end
	engaged, underground, target = false, false, nil
	if noclipConn then noclipConn:Disconnect(); noclipConn = nil end
	noclipped, savedCF, currentChar = {}, nil, nil
end

local function farmStep(dt)
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if not root or not hum or hum.Health <= 0 then return end

	-- personagem novo (respawn): recomeça do zero
	if char ~= currentChar then
		currentChar = char
		engaged, underground, target, savedCF = false, false, nil, nil
		noclipped = {}
	end

	if target and not target.Parent then target = nil end
	if not target then
		if os.clock() < nextScan then return end
		target = pickCoin(root.Position)
		if not target then
			-- sem moeda: fica normal (não deixa o boneco cair no void)
			nextScan = os.clock() + (coinContainer and 0.5 or 2)
			release(char, root, hum)
			return
		end
		local far = (target.Position - root.Position).Magnitude
		deadline = os.clock() + far / math.max(farmSpeed(), 0.1) + 2
	end

	if not engaged then engage(root) end
	hum.PlatformStand = true
	setAnimations(char, hum, false)

	local pos = root.Position
	local goal = Vector3.new(target.Position.X, target.Position.Y - UNDER_OFFSET, target.Position.Z)

	-- primeira vez: desce reto para baixo do mapa
	if not underground then
		pos = Vector3.new(pos.X, goal.Y, pos.Z)
		underground = true
	end

	local diff = goal - pos
	local dist = diff.Magnitude
	if dist < 0.3 or os.clock() > deadline then
		skipped[target] = true
		target = nil
	else
		pos += diff.Unit * math.min(farmSpeed() * dt, dist)
	end

	-- segura o boneco no lugar (sem gravidade) mesmo entre uma moeda e outra
	root.AssemblyLinearVelocity = Vector3.zero
	root.AssemblyAngularVelocity = Vector3.zero
	root.CFrame = lyingCF(pos)
end

local function startFarm()
	stopFarm()
	nextScan = 0
	farmConn = RunService.Heartbeat:Connect(farmStep)
end

-- Tema -----------------------------------------------------------------------
local C = {
	bg = Color3.fromRGB(26, 14, 48),
	row = Color3.fromRGB(38, 24, 66),
	stroke = Color3.fromRGB(120, 70, 200),
	orange = Color3.fromRGB(255, 140, 20),
	offDot = Color3.fromRGB(255, 90, 160),
	onDot = Color3.fromRGB(90, 255, 140),
	toggleOff = Color3.fromRGB(70, 55, 100),
	text = Color3.fromRGB(255, 255, 255),
	sub = Color3.fromRGB(170, 150, 210),
}

local WIDTH, FULL_HEIGHT, MINI_HEIGHT = 310, 340, 62

-- Helpers --------------------------------------------------------------------
local function new(class, props, parent)
	local o = Instance.new(class)
	for k, v in pairs(props) do o[k] = v end
	o.Parent = parent
	return o
end
local function corner(parent, r) new("UICorner", { CornerRadius = UDim.new(0, r) }, parent) end
local function isPress(i)
	return i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch
end
local function isMove(i)
	return i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch
end

-- Arrasto genérico: escuta movimento só enquanto o dedo/mouse está pressionado.
-- Retorna uma função que cancela o arrasto em andamento.
local activeDrags = {}
local function beginDrag(input, onMove)
	local conn
	local function stop()
		if conn then conn:Disconnect(); conn = nil end
		activeDrags[stop] = nil
	end
	conn = UIS.InputChanged:Connect(function(m)
		if isMove(m) then onMove(m) end
	end)
	activeDrags[stop] = true
	input.Changed:Connect(function()
		if input.UserInputState == Enum.UserInputState.End then stop() end
	end)
end
local function cancelAllDrags()
	for stop in pairs(activeDrags) do stop() end
end

-- Janela ---------------------------------------------------------------------
local old = playerGui:FindFirstChild("MM2HalloweenUI")
if old then old:Destroy() end -- evita janelas duplicadas ao reexecutar

local gui = new("ScreenGui", {
	Name = "MM2HalloweenUI", ResetOnSpawn = false, ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
}, playerGui)

local main = new("Frame", {
	Size = UDim2.fromOffset(WIDTH, FULL_HEIGHT), Position = UDim2.new(0.5, 0, 0.5, -FULL_HEIGHT / 2),
	AnchorPoint = Vector2.new(0.5, 0), BackgroundColor3 = C.bg, BackgroundTransparency = 0.25,
	ClipsDescendants = true, Active = true,
}, gui)
corner(main, 14)
new("UIStroke", { Color = C.stroke, Thickness = 2 }, main)

-- Cabeçalho ------------------------------------------------------------------
local header = new("Frame", { Size = UDim2.new(1, 0, 0, MINI_HEIGHT), BackgroundTransparency = 1, Active = true }, main)

new("TextLabel", {
	Text = "🎃", Size = UDim2.fromOffset(40, 40), Position = UDim2.fromOffset(12, 10),
	BackgroundTransparency = 1, TextScaled = true,
}, header)
new("TextLabel", {
	Text = "MM2", Size = UDim2.fromOffset(60, 30), Position = UDim2.fromOffset(58, 6),
	BackgroundTransparency = 1, Font = Enum.Font.FredokaOne, TextSize = 26,
	TextColor3 = C.text, TextXAlignment = Enum.TextXAlignment.Left,
}, header)
new("TextLabel", {
	Text = "HALLOWEEN", Size = UDim2.fromOffset(150, 30), Position = UDim2.fromOffset(105, 4),
	BackgroundTransparency = 1, Font = Enum.Font.FredokaOne, TextSize = 28,
	TextColor3 = C.orange, TextXAlignment = Enum.TextXAlignment.Left,
}, header)
new("TextLabel", {
	Text = "AUTOFARM / SPOOKY EDITION", Size = UDim2.fromOffset(200, 14),
	Position = UDim2.fromOffset(60, 36), BackgroundTransparency = 1,
	Font = Enum.Font.GothamBold, TextSize = 9, TextColor3 = C.sub,
	TextXAlignment = Enum.TextXAlignment.Left,
}, header)

local function headerButton(txt, x)
	local b = new("TextButton", {
		Text = txt, Size = UDim2.fromOffset(22, 22), Position = UDim2.new(1, x, 0, 10),
		BackgroundColor3 = C.row, TextColor3 = C.text, Font = Enum.Font.GothamBold,
		TextSize = 14, AutoButtonColor = false,
	}, header)
	corner(b, 6)
	return b
end
local minBtn = headerButton("–", -56)
local closeBtn = headerButton("×", -30)

-- Conteúdo -------------------------------------------------------------------
local content = new("ScrollingFrame", {
	Size = UDim2.new(1, -24, 1, -(MINI_HEIGHT + 26)), Position = UDim2.fromOffset(12, MINI_HEIGHT + 2),
	BackgroundTransparency = 1, BorderSizePixel = 0,
	CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y,
	ScrollingDirection = Enum.ScrollingDirection.Y, ScrollBarThickness = 3,
	ScrollBarImageColor3 = C.orange, ElasticBehavior = Enum.ElasticBehavior.Never,
}, main)
new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, content)

local function row(height, order)
	local f = new("Frame", {
		Size = UDim2.new(1, -8, 0, height), BackgroundColor3 = C.row,
		BackgroundTransparency = 0.2, LayoutOrder = order,
	}, content)
	corner(f, 10)
	return f
end

local function label(parent, text, size, pos, font, textSize, color, align)
	return new("TextLabel", {
		Text = text, Size = size, Position = pos, BackgroundTransparency = 1,
		Font = font, TextSize = textSize, TextColor3 = color,
		TextXAlignment = align or Enum.TextXAlignment.Left,
	}, parent)
end

-- Toggle ---------------------------------------------------------------------
local function makeToggle(parent, onChange)
	local track = new("TextButton", {
		Text = "", Size = UDim2.fromOffset(52, 26), Position = UDim2.new(1, -64, 0.5, -13),
		BackgroundColor3 = C.toggleOff, AutoButtonColor = false,
	}, parent)
	corner(track, 13)
	local knob = new("Frame", {
		Size = UDim2.fromOffset(20, 20), Position = UDim2.fromOffset(3, 3), BackgroundColor3 = C.text,
	}, track)
	corner(knob, 10)

	local on = false
	track.Activated:Connect(function()
		on = not on
		knob.Position = on and UDim2.fromOffset(29, 3) or UDim2.fromOffset(3, 3)
		track.BackgroundColor3 = on and C.orange or C.toggleOff
		onChange(on)
	end)
end

-- Linha padrão: ícone + nome + toggle (+ status opcional)
local function toggleRow(order, height, icon, name, key, withStatus)
	local r = row(height, order)
	label(r, icon, UDim2.fromOffset(30, 30), UDim2.new(0, 10, 0.5, -15),
		Enum.Font.GothamBold, 20, C.orange, Enum.TextXAlignment.Center)
	label(r, name, UDim2.fromOffset(150, 20),
		withStatus and UDim2.fromOffset(48, 8) or UDim2.new(0, 48, 0.5, -10),
		Enum.Font.GothamBold, 15, C.text)

	local dot, status
	if withStatus then
		dot = new("Frame", {
			Size = UDim2.fromOffset(7, 7), Position = UDim2.fromOffset(48, 33), BackgroundColor3 = C.offDot,
		}, r)
		corner(dot, 4)
		status = label(r, "OFF", UDim2.fromOffset(60, 12), UDim2.fromOffset(60, 30),
			Enum.Font.GothamBold, 10, C.offDot)
	end

	makeToggle(r, function(on)
		State[key] = on
		if withStatus then
			local col = on and C.onDot or C.offDot
			status.Text = on and "ON" or "OFF"
			status.TextColor3 = col
			dot.BackgroundColor3 = col
		end
		fire(key, on)
	end)
	return r
end

toggleRow(1, 54, "🌱", "Auto Farm", "AutoFarm", true)
toggleRow(2, 44, "🏆", "Auto Win", "AutoWin")
toggleRow(3, 44, "⭐", "XP Farm", "XPFarm")
toggleRow(4, 44, "🔄", "Auto Respawn", "AutoRespawn")

-- Farm Speed (slider 1 a 50) -------------------------------------------------
local MIN, MAX = 1, 50
local r5 = row(64, 5)
label(r5, "⏱", UDim2.fromOffset(30, 30), UDim2.fromOffset(10, 4),
	Enum.Font.GothamBold, 20, C.orange, Enum.TextXAlignment.Center)
label(r5, "Farm Speed", UDim2.fromOffset(120, 20), UDim2.fromOffset(48, 8),
	Enum.Font.GothamBold, 15, C.text)
local valueBox = new("TextLabel", {
	Text = tostring(State.FarmSpeed), Size = UDim2.fromOffset(64, 22),
	Position = UDim2.new(1, -76, 0, 8), BackgroundColor3 = C.bg,
	Font = Enum.Font.GothamBold, TextSize = 13, TextColor3 = C.text,
}, r5)
corner(valueBox, 6)

local bar = new("Frame", {
	Size = UDim2.new(1, -32, 0, 6), Position = UDim2.new(0, 16, 0, 44), BackgroundColor3 = C.toggleOff, Active = true,
}, r5)
corner(bar, 3)
local fill = new("Frame", { Size = UDim2.fromScale(0, 1), BackgroundColor3 = C.orange }, bar)
corner(fill, 3)
local sKnob = new("Frame", {
	Size = UDim2.fromOffset(16, 16), AnchorPoint = Vector2.new(0.5, 0.5),
	Position = UDim2.fromScale(0, 0.5), BackgroundColor3 = C.orange,
}, bar)
corner(sKnob, 8)

label(r5, tostring(MIN), UDim2.fromOffset(20, 10), UDim2.new(0, 14, 0, 52),
	Enum.Font.Gotham, 8, C.sub)
label(r5, tostring(MAX), UDim2.fromOffset(20, 10), UDim2.new(1, -34, 0, 52),
	Enum.Font.Gotham, 8, C.sub, Enum.TextXAlignment.Right)

-- Atualiza só o visual do slider
local function renderSlider(v)
	local a = (v - MIN) / (MAX - MIN)
	fill.Size = UDim2.fromScale(a, 1)
	sKnob.Position = UDim2.fromScale(a, 0.5)
	valueBox.Text = tostring(v)
end

local function setValue(v)
	v = math.clamp(v, MIN, MAX)
	renderSlider(v)
	if v ~= State.FarmSpeed then
		State.FarmSpeed = v
		fire("FarmSpeed", v)
	end
end
renderSlider(State.FarmSpeed)

local function sliderFromInput(i)
	local a = (i.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X
	setValue(math.floor(MIN + (MAX - MIN) * math.clamp(a, 0, 1) + 0.5))
end
bar.InputBegan:Connect(function(i)
	if not isPress(i) then return end
	sliderFromInput(i)
	beginDrag(i, sliderFromInput)
end)

toggleRow(6, 44, "🛡", "Anti Fling / Void", "AntiFling")

-- Rodapé (some quando minimizado) ---------------------------------------------
local footer = label(main, "made by Rocket", UDim2.fromOffset(120, 12),
	UDim2.new(0, 12, 1, -16), Enum.Font.Gotham, 8, C.sub)

-- Minimizar / fechar ---------------------------------------------------------
local minimized = false
minBtn.Activated:Connect(function()
	minimized = not minimized
	content.Visible = not minimized
	footer.Visible = not minimized
	main.Size = UDim2.fromOffset(WIDTH, minimized and MINI_HEIGHT or FULL_HEIGHT)
end)

-- Fechar apenas esconde o painel; digitar "Farm" no chat abre de novo
closeBtn.Activated:Connect(function()
	cancelAllDrags()
	gui.Enabled = false
end)

player.Chatted:Connect(function(msg)
	local m = msg:lower():match("^%s*/?%s*(.-)%s*$")
	if m == "farm" then
		gui.Enabled = true
	end
end)

-- Arrastar janela --------------------------------------------------------------
header.InputBegan:Connect(function(i)
	if not isPress(i) then return end
	local dragStart, startPos = i.Position, main.Position
	beginDrag(i, function(m)
		local d = m.Position - dragStart
		main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X,
			startPos.Y.Scale, startPos.Y.Offset + d.Y)
	end)
end)

-- Ações do saco cheio ------------------------------------------------------------
--   Auto Farm   : pausa (boneco volta ao normal) e retoma quando o saco zera (novo round)
--   XP Farm     : teleporta pro lobby até o saco zerar (novo round)
--   Auto Respawn: reseta o personagem
local LOBBY_CFRAME = nil -- se souber a posição do lobby: CFrame.new(x, y, z)

local function lobbyCFrame()
	if LOBBY_CFRAME then return LOBBY_CFRAME end
	local lobby = workspace:FindFirstChild("Lobby", true)
	if not lobby then return nil end
	for _, d in ipairs(lobby:GetDescendants()) do
		if d:IsA("SpawnLocation") then return d.CFrame + Vector3.new(0, 4, 0) end
	end
	if lobby:IsA("Model") then return lobby:GetPivot() + Vector3.new(0, 6, 0) end
	if lobby:IsA("BasePart") then return lobby.CFrame + Vector3.new(0, 6, 0) end
	return nil
end

-- Auto Win: com o saco cheio, pega a Gun (se não tiver) e atira em quem tem a Knife --
local SHOT_TRIES = 6     -- tentativas de tiro
local SHOT_DELAY = 1     -- segundos entre tentativas (recarga da arma)
local winToken = 0       -- muda a cada novo round / nova execução (cancela a anterior)

local function getTool(plr, name)
	local ch, bp = plr.Character, plr:FindFirstChildOfClass("Backpack")
	local t = (ch and ch:FindFirstChild(name)) or (bp and bp:FindFirstChild(name))
	return (t and t:IsA("Tool")) and t or nil
end

local function myParts()
	local char = player.Character
	local root = char and char:FindFirstChild("HumanoidRootPart")
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if root and hum and hum.Health > 0 then return char, root, hum end
end

local function findGunDrop(from)
	local best, bestDist = nil, math.huge
	for _, d in ipairs(workspace:GetDescendants())
