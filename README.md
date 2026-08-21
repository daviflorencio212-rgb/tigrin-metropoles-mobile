--[[ TIGRINHO FREE v1 | laranja | mobile + PC ]]
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local StarterGui = game:GetService("StarterGui")
local TextChat = game:GetService("TextChatService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LP = Players.LocalPlayer
local Cam = Workspace.CurrentCamera
local mobile = UIS.TouchEnabled

local C = {
	Aimbot = false,
	Silent = false,
	Aura = false,
	AuraDist = 30,
	ShowFOV = false,
	FOV = 180,
	Smooth = 0.12,
	MaxDist = 500,
	ESP = false,
	ESPDist = 350,
	Noclip = false,
	Fly = false,
	FlySpeed = 55
}

local holdAim, menuOn, dead = false, true, false
local nConn, flyBV, auraLock, lootT = nil, nil, nil, 0
local drag, d0, p0 = false, nil, nil
local esp = {}
local FORCE = 10000000000

local Th = {
	Bg = Color3.fromRGB(18, 12, 8),
	Top = Color3.fromRGB(28, 18, 10),
	Card = Color3.fromRGB(36, 24, 14),
	Bar = Color3.fromRGB(55, 35, 18),
	Ac = Color3.fromRGB(255, 140, 40),
	Soft = Color3.fromRGB(255, 190, 120),
	Tx = Color3.fromRGB(255, 245, 230),
	Dim = Color3.fromRGB(160, 130, 100),
	Off = Color3.fromRGB(50, 35, 22)
}

local function nt(t)
	pcall(function()
		StarterGui:SetCore("SendNotification", {Title = "Tigrinho Free", Text = tostring(t), Duration = 2})
	end)
end

local function corner(o, r)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, r or 10)
	c.Parent = o
end

local function stroke(o, col, t)
	local s = Instance.new("UIStroke")
	s.Color = col or Th.Ac
	s.Thickness = t or 1
	s.Transparency = 0.4
	s.Parent = o
	return s
end

local function headOf(ch)
	if not ch then return nil end
	return ch:FindFirstChild("Head") or ch:FindFirstChild("HumanoidRootPart")
end

local function body(ch)
	if not ch then return nil end
	return ch:FindFirstChild("HumanoidRootPart") or ch:FindFirstChild("Head")
end

local function screenDist(pos)
	local sp, on = Cam:WorldToViewportPoint(pos)
	if not on or sp.Z <= 0 then return 999999, false end
	local c = Cam.ViewportSize / 2
	local dx = sp.X - c.X
	local dy = sp.Y - c.Y
	return math.sqrt(dx * dx + dy * dy), true
end

local function lookScore(pos)
	local d = pos - Cam.CFrame.Position
	if d.Magnitude < 1 then return 1 end
	return Cam.CFrame.LookVector:Dot(d.Unit)
end

local function validEnemy(plr)
	if not plr or plr == LP or not plr.Character then return false end
	local h = plr.Character:FindFirstChildOfClass("Humanoid")
	return h and h.Health > 0
end

local function findT(noFov)
	local lim = noFov and 999999 or C.FOV
	local best, bs = nil, -1
	local meu = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
	if not meu then return nil end
	for _, plr in ipairs(Players:GetPlayers()) do
		if validEnemy(plr) then
			local part = headOf(plr.Character)
			if part then
				local wd = (part.Position - meu.Position).Magnitude
				if wd <= C.MaxDist then
					local sd, on = screenDist(part.Position)
					if on and sd <= lim then
						local sc = lookScore(part.Position) * 2 - (sd / math.max(lim, 1))
						if sc > bs then
							bs = sc
							best = part.Position
						end
					end
				end
			end
		end
	end
	return best
end

local function findAura()
	local meu = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
	if not meu then
		auraLock = nil
		return nil
	end
	if auraLock and validEnemy(auraLock) then
		local part = headOf(auraLock.Character)
		if part and (part.Position - meu.Position).Magnitude <= C.AuraDist + 8 then
			return part.Position
		end
	end
	auraLock = nil
	local bp, bs = nil, -999
	for _, plr in ipairs(Players:GetPlayers()) do
		if validEnemy(plr) then
			local part = headOf(plr.Character)
			if part and (part.Position - meu.Position).Magnitude <= C.AuraDist then
				local sd = select(1, screenDist(part.Position))
				local sc = lookScore(part.Position) * 3 - (sd / 400)
				if sc > bs then
					bs = sc
					bp = plr
				end
			end
		end
	end
	if bp then
		auraLock = bp
		local p = headOf(bp.Character)
		if p then return p.Position end
	end
	return nil
end

local function findFOV()
	local best, bd = nil, C.FOV
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= LP and plr.Character then
			local p = body(plr.Character)
			if p then
				local d, on = screenDist(p.Position)
				if on and d < C.FOV and d < bd then
					bd = d
					best = plr
				end
			end
		end
	end
	return best
end

local function aim(pos, sil)
	if not pos then return end
	local s = 1 - C.Smooth
	if sil or C.Silent then s = 0.99 end
	if s < 0.4 then s = 0.4 end
	if s > 0.97 then s = 0.97 end
	Cam.CFrame = Cam.CFrame:Lerp(CFrame.lookAt(Cam.CFrame.Position, pos), s)
end

local function chat(msg)
	pcall(function()
		local ch = TextChat:FindFirstChild("TextChannels")
		local c = ch and (ch:FindFirstChild("RBXGeneral") or ch:FindFirstChildWhichIsA("TextChannel"))
		if c and c.SendAsync then
			c:SendAsync(msg)
		else
			LP:Chat(msg)
		end
	end)
end

local function fireRev(t)
	pcall(function()
		local r = ReplicatedStorage:FindFirstChild("shared")
		r = r and r:FindFirstChild("Eventos")
		r = r and r:FindFirstChild("revisar_function")
		if r then
			if r:IsA("RemoteFunction") then
				r:InvokeServer(t)
			else
				r:FireServer(t)
			end
		end
	end)
end

local function revistar()
	if dead or tick() < lootT then return end
	local t = findFOV()
	if not t then
		nt("Ninguem no FOV")
		return
	end
	lootT = tick() + 1.5
	local n = tostring(t.Name or "")
	chat("/revistar " .. n)
	fireRev(t)
	local h = t.Character and t.Character:FindFirstChildOfClass("Humanoid")
	nt(((not h or h.Health <= 0) and "MORTO: " or "VIVO: ") .. n)
end

local function setNoclip(on)
	C.Noclip = on
	if nConn then nConn:Disconnect() nConn = nil end
	if not on then
		local ch = LP.Character
		if ch then
			for _, p in ipairs(ch:GetDescendants()) do
				if p:IsA("BasePart") then
					p.CanCollide = (p.Name ~= "HumanoidRootPart")
				end
			end
		end
		return
	end
	nConn = RunService.Stepped:Connect(function()
		if dead or not C.Noclip then return end
		local c = LP.Character
		if not c then return end
		for _, p in ipairs(c:GetDescendants()) do
			if p:IsA("BasePart") then p.CanCollide = false end
		end
	end)
end

local function setFly(on)
	C.Fly = on
	if flyBV then pcall(function() flyBV:Destroy() end) flyBV = nil end
	local r = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
	if not on then
		if r then r.AssemblyLinearVelocity = Vector3.zero end
		return
	end
	if not r then return end
	flyBV = Instance.new("BodyVelocity")
	flyBV.MaxForce = Vector3.new(FORCE, FORCE, FORCE)
	flyBV.Parent = r
end

local function clearE(id)
	local c = esp[id]
	if not c then return end
	for _, o in pairs(c) do
		if typeof(o) == "Instance" and o.Parent then o:Destroy() end
	end
	esp[id] = nil
end

local function clearAll()
	for id in pairs(esp) do clearE(id) end
end

local function updESP(plr)
	local ch = plr.Character
	if not ch then clearE(plr.UserId) return end
	local hum = ch:FindFirstChildOfClass("Humanoid")
	local root = body(ch)
	local meu = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
	if not root or not meu then clearE(plr.UserId) return end
	local dist = (root.Position - meu.Position).Magnitude
	if not C.ESP or dist > C.ESPDist then clearE(plr.UserId) return end
	if not esp[plr.UserId] then esp[plr.UserId] = {} end
	local c = esp[plr.UserId]
	local vivo = hum and hum.Health > 0
	local head = ch:FindFirstChild("Head") or root

	if not c.hl or not c.hl.Parent then
		local hl = Instance.new("Highlight")
		hl.FillColor = Th.Ac
		hl.FillTransparency = 0.85
		hl.OutlineColor = Th.Ac
		hl.OutlineTransparency = 0.25
		hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		hl.Parent = ch
		c.hl = hl
	end

	if head then
		if not c.info or not c.info.Parent then
			local bb = Instance.new("BillboardGui")
			bb.Name = "TigFreeESP"
			bb.Adornee = head
			bb.Size = UDim2.fromOffset(130, 14)
			bb.StudsOffset = Vector3.new(0, 2.4, 0)
			bb.AlwaysOnTop = true
			bb.Parent = ch
			local lb = Instance.new("TextLabel")
			lb.Size = UDim2.fromScale(1, 1)
			lb.BackgroundTransparency = 1
			lb.Font = Enum.Font.GothamBold
			lb.TextSize = 12
			lb.TextStrokeTransparency = 0.4
			lb.TextColor3 = Color3.new(1, 1, 1)
			lb.Parent = bb
			c.info = bb
			c.lb = lb
		end
		local tx = plr.Name
		if not vivo then tx = tx .. " [M]" end
		tx = tx .. " " .. math.floor(dist) .. "m"
		c.lb.Text = tx
		c.lb.TextColor3 = vivo and Th.Soft or Color3.fromRGB(255, 90, 70)
	end
end

-- GUI
local pg = LP:WaitForChild("PlayerGui")
pcall(function()
	for _, v in pairs(pg:GetChildren()) do
		if v.Name == "TigrinhoFree" then v:Destroy() end
	end
end)

local gui = Instance.new("ScreenGui")
gui.Name = "TigrinhoFree"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.DisplayOrder = 40
gui.Parent = pg

-- FOV
local fovF = Instance.new("Frame")
fovF.AnchorPoint = Vector2.new(0.5, 0.5)
fovF.Position = UDim2.fromScale(0.5, 0.5)
fovF.Size = UDim2.fromOffset(C.FOV * 2, C.FOV * 2)
fovF.BackgroundTransparency = 1
fovF.Visible = false
fovF.Parent = gui
corner(fovF, 999)
stroke(fovF, Th.Ac, 1.5)

-- Menu principal (formato diferente: card vertical arredondado)
local mainW = mobile and 300 or 340
local mainH = mobile and 420 or 460

local main = Instance.new("Frame")
main.Size = UDim2.fromOffset(mainW, mainH)
main.Position = UDim2.new(0.5, -mainW / 2, 0.5, -mainH / 2)
main.BackgroundColor3 = Th.Bg
main.BorderSizePixel = 0
main.Active = true
main.ClipsDescendants = true
main.Parent = gui
corner(main, 16)
stroke(main, Th.Ac, 1.2)

-- barra topo com gradiente visual
local top = Instance.new("Frame")
top.Size = UDim2.new(1, 0, 0, 48)
top.BackgroundColor3 = Th.Top
top.BorderSizePixel = 0
top.Parent = main
corner(top, 16)
local topFix = Instance.new("Frame")
topFix.Size = UDim2.new(1, 0, 0, 16)
topFix.Position = UDim2.new(0, 0, 1, -16)
topFix.BackgroundColor3 = Th.Top
topFix.BorderSizePixel = 0
topFix.Parent = top

local accent = Instance.new("Frame")
accent.Size = UDim2.new(1, 0, 0, 3)
accent.Position = UDim2.new(0, 0, 1, -3)
accent.BackgroundColor3 = Th.Ac
accent.BorderSizePixel = 0
accent.Parent = top

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -50, 1, 0)
title.Position = UDim2.fromOffset(14, 0)
title.BackgroundTransparency = 1
title.Text = "TIGRINHO  FREE"
title.Font = Enum.Font.GothamBold
title.TextSize = mobile and 15 or 16
title.TextColor3 = Th.Ac
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = top

local xBtn = Instance.new("TextButton")
xBtn.Size = UDim2.fromOffset(28, 28)
xBtn.Position = UDim2.new(1, -38, 0.5, -14)
xBtn.BackgroundColor3 = Th.Card
xBtn.Text = "X"
xBtn.Font = Enum.Font.GothamBold
xBtn.TextSize = 13
xBtn.TextColor3 = Th.Dim
xBtn.Parent = top
corner(xBtn, 8)

-- scroll conteudo
local content = Instance.new("ScrollingFrame")
content.Size = UDim2.new(1, -16, 1, -64)
content.Position = UDim2.fromOffset(8, 56)
content.BackgroundTransparency = 1
content.BorderSizePixel = 0
content.ScrollBarThickness = 3
content.ScrollBarImageColor3 = Th.Ac
content.AutomaticCanvasSize = Enum.AutomaticSize.Y
content.CanvasSize = UDim2.new(0, 0, 0, 0)
content.Parent = main

local list = Instance.new("UIListLayout")
list.Padding = UDim.new(0, 6)
list.SortOrder = Enum.SortOrder.LayoutOrder
list.Parent = content

-- botao abrir (mobile/pc)
local openBtn = Instance.new("TextButton")
openBtn.Size = UDim2.fromOffset(46, 46)
openBtn.Position = UDim2.new(0, 12, 1, -64)
openBtn.BackgroundColor3 = Th.Ac
openBtn.Text = "T"
openBtn.Font = Enum.Font.GothamBold
openBtn.TextSize = 18
openBtn.TextColor3 = Color3.new(1, 1, 1)
openBtn.Visible = false
openBtn.ZIndex = 30
openBtn.Parent = gui
corner(openBtn, 12)
stroke(openBtn, Color3.new(1, 1, 1), 1)

-- botao revistar flutuante
local revBtn = Instance.new("TextButton")
revBtn.Size = UDim2.fromOffset(110, 36)
revBtn.Position = UDim2.fromOffset(12, mobile and 100 or 130)
revBtn.BackgroundColor3 = Th.Ac
revBtn.Text = "REVISTAR"
revBtn.Font = Enum.Font.GothamBold
revBtn.TextSize = 13
revBtn.TextColor3 = Color3.new(1, 1, 1)
revBtn.ZIndex = 50
revBtn.Active = true
revBtn.Parent = gui
corner(revBtn, 10)
stroke(revBtn, Color3.new(1, 1, 1), 0.8)

-- helpers UI
local order = 0
local function nextOrder()
	order = order + 1
	return order
end

local function section(txt)
	local l = Instance.new("TextLabel")
	l.Size = UDim2.new(1, 0, 0, 18)
	l.BackgroundTransparency = 1
	l.Text = txt
	l.Font = Enum.Font.GothamBold
	l.TextSize = 11
	l.TextColor3 = Th.Dim
	l.TextXAlignment = Enum.TextXAlignment.Left
	l.LayoutOrder = nextOrder()
	l.Parent = content
end

local function makeToggle(txt, def, cb)
	local f = Instance.new("Frame")
	f.Size = UDim2.new(1, 0, 0, 34)
	f.BackgroundColor3 = Th.Card
	f.BorderSizePixel = 0
	f.LayoutOrder = nextOrder()
	f.Parent = content
	corner(f, 9)

	local l = Instance.new("TextLabel")
	l.Size = UDim2.new(1, -50, 1, 0)
	l.Position = UDim2.fromOffset(10, 0)
	l.BackgroundTransparency = 1
	l.Text = txt
	l.Font = Enum.Font.Gotham
	l.TextSize = 13
	l.TextColor3 = Th.Tx
	l.TextXAlignment = Enum.TextXAlignment.Left
	l.Parent = f

	local b = Instance.new("TextButton")
	b.Size = UDim2.fromOffset(40, 20)
	b.Position = UDim2.new(1, -48, 0.5, -10)
	b.BackgroundColor3 = def and Th.Ac or Th.Off
	b.Text = ""
	b.Parent = f
	corner(b, 10)

	local d = Instance.new("Frame")
	d.Size = UDim2.fromOffset(16, 16)
	d.Position = def and UDim2.new(1, -18, 0.5, -8) or UDim2.fromOffset(2, 2)
	d.BackgroundColor3 = Color3.new(1, 1, 1)
	d.BorderSizePixel = 0
	d.Parent = b
	corner(d, 8)

	local st = def and true or false
	b.MouseButton1Click:Connect(function()
		if dead then return end
		st = not st
		b.BackgroundColor3 = st and Th.Ac or Th.Off
		d.Position = st and UDim2.new(1, -18, 0.5, -8) or UDim2.fromOffset(2, 2)
		cb(st)
	end)
end

local function makeSlider(txt, def, minV, maxV, cb)
	local f = Instance.new("Frame")
	f.Size = UDim2.new(1, 0, 0, 48)
	f.BackgroundColor3 = Th.Card
	f.BorderSizePixel = 0
	f.LayoutOrder = nextOrder()
	f.Parent = content
	corner(f, 9)

	local l = Instance.new("TextLabel")
	l.Size = UDim2.new(0.65, 0, 0, 16)
	l.Position = UDim2.fromOffset(10, 6)
	l.BackgroundTransparency = 1
	l.Text = txt
	l.Font = Enum.Font.Gotham
	l.TextSize = 12
	l.TextColor3 = Th.Tx
	l.TextXAlignment = Enum.TextXAlignment.Left
	l.Parent = f

	local vl = Instance.new("TextLabel")
	vl.Size = UDim2.new(0.3, -8, 0, 16)
	vl.Position = UDim2.new(0.68, 0, 0, 6)
	vl.BackgroundTransparency = 1
	vl.Text = tostring(def)
	vl.Font = Enum.Font.GothamBold
	vl.TextSize = 12
	vl.TextColor3 = Th.Soft
	vl.TextXAlignment = Enum.TextXAlignment.Right
	vl.Parent = f

	local bg = Instance.new("Frame")
	bg.Size = UDim2.new(1, -20, 0, 6)
	bg.Position = UDim2.fromOffset(10, 32)
	bg.BackgroundColor3 = Th.Bar
	bg.BorderSizePixel = 0
	bg.Parent = f
	corner(bg, 3)

	local range = maxV - minV
	if range < 1 then range = 1 end
	local bar = Instance.new("Frame")
	bar.Size = UDim2.new(math.clamp((def - minV) / range, 0, 1), 0, 1, 0)
	bar.BackgroundColor3 = Th.Ac
	bar.BorderSizePixel = 0
	bar.Parent = bg
	corner(bar, 3)

	local sliding = false
	local function upd(inp)
		local w = bg.AbsoluteSize.X
		if w < 1 then w = 1 end
		local rel = math.clamp((inp.Position.X - bg.AbsolutePosition.X) / w, 0, 1)
		local v = math.floor(minV + range * rel)
		bar.Size = UDim2.new(rel, 0, 1, 0)
		vl.Text = tostring(v)
		cb(v)
	end

	bg.InputBegan:Connect(function(i)
		if dead then return end
		if i.UserInputType == Enum.UserInputType.MouseButton1
			or i.UserInputType == Enum.UserInputType.Touch then
			sliding = true
			upd(i)
		end
	end)
	UIS.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1
			or i.UserInputType == Enum.UserInputType.Touch then
			sliding = false
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if dead or not sliding then return end
		if i.UserInputType == Enum.UserInputType.MouseMovement
			or i.UserInputType == Enum.UserInputType.Touch then
			upd(i)
		end
	end)
end

-- monta menu
section("COMBATE")
makeToggle("Aimbot (segurar mira)", C.Aimbot, function(v) C.Aimbot = v end)
makeToggle("Silent Aim", C.Silent, function(v) C.Silent = v end)
makeToggle("Silent Aura", C.Aura, function(v)
	C.Aura = v
	if not v then auraLock = nil end
end)
makeSlider("Aura distancia", C.AuraDist, 10, 80, function(v) C.AuraDist = v end)
makeToggle("Mostrar FOV", C.ShowFOV, function(v)
	C.ShowFOV = v
	fovF.Visible = v
end)
makeSlider("FOV", C.FOV, 50, 400, function(v)
	C.FOV = v
	fovF.Size = UDim2.fromOffset(v * 2, v * 2)
end)
makeSlider("Suavidade", math.floor(C.Smooth * 100), 5, 40, function(v)
	C.Smooth = v / 100
end)

section("ESP")
makeToggle("ESP (nome + distancia)", C.ESP, function(v)
	C.ESP = v
	if not v then clearAll() end
end)
makeSlider("Distancia ESP", C.ESPDist, 50, 800, function(v) C.ESPDist = v end)

section("PLAYER")
makeToggle("Noclip", C.Noclip, function(v) setNoclip(v) end)
makeToggle("Fly", C.Fly, function(v) setFly(v) end)
makeSlider("Velocidade Fly", C.FlySpeed, 20, 150, function(v) C.FlySpeed = v end)

section("INFO")
local info = Instance.new("TextLabel")
info.Size = UDim2.new(1, 0, 0, 36)
info.BackgroundTransparency = 1
info.Text = mobile and "Mobile: toque na tela = mira\nBotao T reabre o menu"
	or "PC: botao direito = mira\nRightShift = menu"
info.Font = Enum.Font.Gotham
info.TextSize = 11
info.TextColor3 = Th.Dim
info.TextXAlignment = Enum.TextXAlignment.Left
info.TextYAlignment = Enum.TextYAlignment.Top
info.LayoutOrder = nextOrder()
info.Parent = content

-- drag menu
top.InputBegan:Connect(function(i)
	if dead then return end
	if i.UserInputType == Enum.UserInputType.MouseButton1
		or i.UserInputType == Enum.UserInputType.Touch then
		drag = true
		d0 = i.Position
		p0 = main.Position
	end
end)
UIS.InputChanged:Connect(function(i)
	if dead or not drag then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement
		or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - d0
		main.Position = UDim2.new(p0.X.Scale, p0.X.Offset + d.X, p0.Y.Scale, p0.Y.Offset + d.Y)
	end
end)
UIS.InputEnded:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1
		or i.UserInputType == Enum.UserInputType.Touch then
		drag = false
	end
end)

-- drag revistar
do
	local rd, rs, rp = false, nil, nil
	revBtn.InputBegan:Connect(function(i)
		if dead then return end
		if i.UserInputType == Enum.UserInputType.MouseButton1
			or i.UserInputType == Enum.UserInputType.Touch then
			rd = true
			rs = i.Position
			rp = revBtn.Position
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if dead or not rd then return end
		if i.UserInputType == Enum.UserInputType.MouseMovement
			or i.UserInputType == Enum.UserInputType.Touch then
			local d = i.Position - rs
			revBtn.Position = UDim2.new(rp.X.Scale, rp.X.Offset + d.X, rp.Y.Scale, rp.Y.Offset + d.Y)
		end
	end)
	UIS.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1
			or i.UserInputType == Enum.UserInputType.Touch then
			rd = false
		end
	end)
end
revBtn.MouseButton1Click:Connect(function()
	if not dead then revistar() end
end)

local function vis(v)
	menuOn = v
	main.Visible = v
	openBtn.Visible = not v
end
xBtn.MouseButton1Click:Connect(function() if not dead then vis(false) end end)
openBtn.MouseButton1Click:Connect(function() if not dead then vis(true) end end)

-- input
UIS.InputBegan:Connect(function(i, gp)
	if dead or gp then return end
	if i.KeyCode == Enum.KeyCode.RightShift then
		vis(not menuOn)
	end
	if i.UserInputType == Enum.UserInputType.MouseButton2 then
		holdAim = true
	end
end)

UIS.InputEnded:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton2 then
		holdAim = false
	end
end)

-- mobile: toque = mira
if mobile then
	UIS.TouchStarted:Connect(function()
		if not dead and (C.Aimbot or C.Silent or C.Aura) then
			holdAim = true
		end
	end)
	UIS.TouchEnded:Connect(function()
		holdAim = false
	end)
end

LP.CharacterAdded:Connect(function()
	if dead then return end
	auraLock = nil
	if flyBV then flyBV:Destroy() flyBV = nil end
	if nConn then nConn:Disconnect() nConn = nil end
	task.wait(0.4)
	if C.Noclip then setNoclip(true) end
	if C.Fly then setFly(true) end
end)

RunService.RenderStepped:Connect(function()
	if dead then return end

	if C.Fly then
		local r = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
		if r then
			if not flyBV or not flyBV.Parent then
				flyBV = Instance.new("BodyVelocity")
				flyBV.MaxForce = Vector3.new(FORCE, FORCE, FORCE)
				flyBV.Parent = r
			end
			local dir = Vector3.zero
			local cf = Cam.CFrame
			if UIS:IsKeyDown(Enum.KeyCode.W) then dir = dir + cf.LookVector end
			if UIS:IsKeyDown(Enum.KeyCode.S) then dir = dir - cf.LookVector end
			if UIS:IsKeyDown(Enum.KeyCode.A) then dir = dir - cf.RightVector end
			if UIS:IsKeyDown(Enum.KeyCode.D) then dir = dir + cf.RightVector end
			if UIS:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.new(0, 1, 0) end
			if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then dir = dir - Vector3.new(0, 1, 0) end
			if dir.Magnitude > 0 then
				flyBV.Velocity = dir.Unit * C.FlySpeed
			else
				flyBV.Velocity = Vector3.zero
			end
		end
	elseif flyBV then
		flyBV:Destroy()
		flyBV = nil
	end

	if C.Aura then
		local pos = findAura()
		if pos then aim(pos, true) end
	elseif (C.Aimbot or C.Silent) and holdAim then
		local pos = findT(C.Silent)
		if pos then aim(pos) end
	end
end)

local lastEsp = 0
RunService.Heartbeat:Connect(function()
	if dead then return end
	if tick() - lastEsp < 0.4 then return end
	lastEsp = tick()
	fovF.Visible = C.ShowFOV
	if C.ESP then
		local seen = {}
		for _, plr in ipairs(Players:GetPlayers()) do
			if plr ~= LP then
				seen[plr.UserId] = true
				updESP(plr)
			end
		end
		for id in pairs(esp) do
			if not seen[id] then clearE(id) end
		end
	else
		clearAll()
	end
end)

print("[Tigrinho Free] v1 | laranja | mobile+pc")
nt("Tigrinho Free carregado")
