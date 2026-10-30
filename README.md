-- ═══════════════════════════════════════════════════════════
--  JIRNYAK V6  •  Blade Ball  •  Delta Mobile (Android)
--  CoreGui UI + Auto-Parry + Auto-Radius + Binder
-- ═══════════════════════════════════════════════════════════

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace  = game:GetService("Workspace")
local Stats      = game:GetService("Stats")
local UIS        = game:GetService("UserInputService")
local CoreGui    = game:GetService("CoreGui")
local TweenService = game:GetService("TweenService")
local LP         = Players.LocalPlayer

-- анти-детект: рандомная задержка
task.wait(math.random(15, 90) / 100)

local _G = getgenv()
local storeKey = "bb_" .. tostring(math.random(1e5, 9e5))
_G[storeKey] = _G[storeKey] or {}
local ST = _G[storeKey]

-- конфиг
local C = {
    AutoParry     = true,
    AutoRadius    = true,
    OnlyIncoming  = true,
    AutoSwing     = false,
    AutoSpam      = false,
    ESPBall       = false,
    Cooldown      = 0.12,
    MinRadius     = 7,
    MaxRadius     = 26,
    SafetyFactor  = 1.35,
    SmoothAlpha   = 0.25,
    BallSpeed     = 120,
    SampleHz      = 2,
    MenuKey       = Enum.KeyCode.RightShift,
    ParryKey      = Enum.KeyCode.Q,
    SwingKey      = Enum.KeyCode.E,
}
ST.Config = C

local S = {
    lastParry = 0, radius = C.MinRadius, ping = 80,
    remote = nil, menuOpen = true, dragging = false,
    dragOff = {x=0,y=0}, waitBind = nil,
    touchSlider = nil,
}

local function safe(fn) return function(...) pcall(fn, ...) end end

-- ── утилиты ──
local function getBall()
    for _, o in ipairs(Workspace:GetDescendants()) do
        if o:IsA("BasePart") and (o.Name == "Ball" or o.Name:find("Ball") or o.Name:find("Projectile")) then
            return o
        end
    end
end

local function findRemote()
    local n = {"Parry","Block","Swing","ParryRemote","BlockEvent","SwingEvent"}
    for _, x in ipairs(n) do
        local r = LP:FindFirstChild("Remotes") and LP.Remotes:FindFirstChild(x)
        if r and r:IsA("RemoteEvent") then return r end
    end
    for _, d in ipairs(LP:GetDescendants()) do
        if d:IsA("RemoteEvent") and table.find(n, d.Name) then return d end
    end
end

local function readPing()
    local ok, v = pcall(function()
        return Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
    end)
    return (ok and type(v) == "number" and v > 0) and v or S.ping
end

local function isIncoming(ball, pos)
    local v = ball.AssemblyLinearVelocity
    if v.Magnitude < 1 then return false end
    return v.Unit:Dot((pos - ball.Position).Unit) > 0.5
end

S.remote = findRemote()

local function fireRemote()
    if S.remote then pcall(function() S.remote:FireServer() end) end
end

local function parry()
    local now = tick()
    if now - S.lastParry < C.Cooldown then return end
    S.lastParry = now
    fireRemote()
end

-- ── логика ──
local acc = 0
RunService.Heartbeat:Connect(safe(function(dt)
    if C.AutoRadius then
        acc = acc + dt
        if acc >= 1 / C.SampleHz then
            acc = 0
            S.ping = readPing()
            local b = getBall()
            local spd = b and b.AssemblyLinearVelocity.Magnitude or C.BallSpeed
            if spd < 20 then spd = C.BallSpeed end
            local reach = spd * (S.ping / 1000) * 0.5 * C.SafetyFactor
            local target = math.clamp(reach + 4, C.MinRadius, C.MaxRadius)
            S.radius = S.radius + (target - S.radius) * C.SmoothAlpha
        end
    else
        S.radius = C.MinRadius
    end

    local char = LP.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local myPos = hrp.Position

    if C.AutoParry then
        local b = getBall()
        if b then
            local v = b.AssemblyLinearVelocity
            local pred = b.Position + v * (S.ping / 1000)
            if (pred - myPos).Magnitude <= S.radius then
                if not C.OnlyIncoming or isIncoming(b, myPos) then parry() end
            end
        end
    end

    if C.AutoSwing then
        local b = getBall()
        if b and (b.Position - myPos).Magnitude <= 12 then fireRemote() end
    end
end))

task.spawn(safe(function()
    while task.wait(0.05) do
        if C.AutoSpam then fireRemote() end
    end
end))

-- ═══════════════════════════════════════════════════════════
--  UI (CoreGui ScreenGui, маскировка)
-- ═══════════════════════════════════════════════════════════
local guiName = "SystemOverlay_" .. tostring(math.random(1000, 9999))
local gui = Instance.new("ScreenGui")
gui.Name = guiName
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.DisplayOrder = 999
pcall(function() gui.Parent = CoreGui end)
if not gui.Parent then gui.Parent = LP:WaitForChild("PlayerGui") end

local theme = {
    bg      = Color3.fromRGB(16,16,20),
    bg2     = Color3.fromRGB(24,24,30),
    accent  = Color3.fromRGB(120,80,255),
    accent2 = Color3.fromRGB(80,200,255),
    text    = Color3.fromRGB(230,230,240),
    sub     = Color3.fromRGB(150,150,165),
    off     = Color3.fromRGB(58,58,68),
    good    = Color3.fromRGB(80,220,140),
    bad     = Color3.fromRGB(240,90,90),
}

local W, H = 320, 400
local root = Instance.new("Frame")
root.Size = UDim2.new(0, W, 0, H)
root.Position = UDim2.new(0, 40, 0, 60)
root.BackgroundColor3 = theme.bg
root.BackgroundTransparency = 0.08
root.BorderSizePixel = 0
root.ClipsDescendants = true
root.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = root

local stroke = Instance.new("UIStroke")
stroke.Color = theme.accent
stroke.Transparency = 0.4
stroke.Thickness = 1
stroke.Parent = root

-- шапка
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 40)
header.BackgroundColor3 = theme.bg2
header.BorderSizePixel = 0
header.Parent = root

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -20, 1, 0)
title.Position = UDim2.new(0, 12, 0, 0)
title.BackgroundTransparency = 1
title.Text = "JIRNYAK V6"
title.TextColor3 = theme.text
title.TextSize = 15
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = header

local status = Instance.new("TextLabel")
status.Size = UDim2.new(0, 120, 1, 0)
status.Position = UDim2.new(1, -125, 0, 0)
status.BackgroundTransparency = 1
status.Text = "ready"
status.TextColor3 = theme.good
status.TextSize = 12
status.Font = Enum.Font.Gotham
status.TextXAlignment = Enum.TextXAlignment.Right
status.Parent = header

-- контент
local content = Instance.new("ScrollingFrame")
content.Size = UDim2.new(1, -16, 1, -52)
content.Position = UDim2.new(0, 8, 0, 46)
content.BackgroundTransparency = 1
content.BorderSizePixel = 0
content.ScrollBarThickness = 2
content.ScrollBarImageColor3 = theme.accent
content.CanvasSize = UDim2.new(0, 0, 0, 0)
content.AutomaticCanvasSize = Enum.AutomaticSize.Y
content.Parent = root

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 6)
layout.SortOrder = Enum.SortOrder.LayoutOrder
layout.Parent = content

local function addLabel(txt, color, size)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 0, 22)
    l.BackgroundTransparency = 1
    l.Text = txt
    l.TextColor3 = color or theme.text
    l.TextSize = size or 13
    l.Font = Enum.Font.GothamBold
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = content
    return l
end

-- тумблер (мобильный)
local function addToggle(name, key)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 34)
    row.BackgroundTransparency = 1
    row.Parent = content

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -60, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = name
    lbl.TextColor3 = theme.text
    lbl.TextSize = 13
    lbl.Font = Enum.Font.Gotham
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 50, 0, 24)
    btn.Position = UDim2.new(1, -50, 0.5, -12)
    btn.BackgroundColor3 = C[key] and theme.accent or theme.off
    btn.BorderSizePixel = 0
    btn.Text = C[key] and "ON" or "OFF"
    btn.TextColor3 = theme.text
    btn.TextSize = 12
    btn.Font = Enum.Font.GothamBold
    btn.Parent = row

    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, 6); c.Parent = btn

    btn.MouseButton1Click:Connect(safe(function()
        C[key] = not C[key]
        btn.BackgroundColor3 = C[key] and theme.accent or theme.off
        btn.Text = C[key] and "ON" or "OFF"
    end))
end

-- слайдер (мобильный, тач)
local function addSlider(name, key, mn, mx, st)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 44)
    row.BackgroundTransparency = 1
    row.Parent = content

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -60, 0, 18)
    lbl.BackgroundTransparency = 1
    lbl.Text = name
    lbl.TextColor3 = theme.text
    lbl.TextSize = 13
    lbl.Font = Enum.Font.Gotham
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local val = Instance.new("TextLabel")
    val.Size = UDim2.new(0, 60, 0, 18)
    val.Position = UDim2.new(1, -60, 0, 0)
    val.BackgroundTransparency = 1
    val.Text = tostring(C[key])
    val.TextColor3 = theme.accent2
    val.TextSize = 13
    val.Font = Enum.Font.GothamBold
    val.TextXAlignment = Enum.TextXAlignment.Right
    val.Parent = row

    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, 0, 0, 6)
    track.Position = UDim2.new(0, 0, 0, 26)
    track.BackgroundColor3 = theme.off
    track.BorderSizePixel = 0
    track.Parent = row
    local tc = Instance.new("UICorner"); tc.CornerRadius = UDim.new(0, 3); tc.Parent = track

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((C[key]-mn)/(mx-mn), 0, 1, 0)
    fill.BackgroundColor3 = theme.accent
    fill.BorderSizePixel = 0
    fill.Parent = track
    local fc = Instance.new("UICorner"); fc.CornerRadius = UDim.new(0, 3); fc.Parent = fill

    local knob = Instance.new("Frame")
    knob.Size = UDim2.new(0, 14, 0, 14)
    knob.Position = UDim2.new((C[key]-mn)/(mx-mn), -7, 0.5, -7)
    knob.BackgroundColor3 = theme.text
    knob.BorderSizePixel = 0
    knob.Parent = track
    local kc = Instance.new("UICorner"); kc.CornerRadius = UDim.new(1, 0); kc.Parent = knob

    local dragging = false
    local function setVal(xPos)
        local rel = math.clamp((xPos - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
        local vv = mn + (mx - mn) * rel
        vv = math.floor(vv / st + 0.5) * st
        C[key] = vv
        fill.Size = UDim2.new((vv-mn)/(mx-mn), 0, 1, 0)
        knob.Position = UDim2.new((vv-mn)/(mx-mn), -7, 0.5, -7)
        val.Text = tostring(vv)
    end

    track.InputBegan:Connect(safe(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            setVal(input.Position.X)
        end
    end))
    UIS.InputChanged:Connect(safe(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
            setVal(input.Position.X)
        end
    end))
    UIS.InputEnded:Connect(safe(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end))
end

-- биндер
local function addBinder(name, key)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 34)
    row.BackgroundTransparency = 1
    row.Parent = content

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -80, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = name
    lbl.TextColor3 = theme.text
    lbl.TextSize = 13
    lbl.Font = Enum.Font.Gotham
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 70, 0, 24)
    btn.Position = UDim2.new(1, -70, 0.5, -12)
    btn.BackgroundColor3 = theme.bg2
    btn.BorderSizePixel = 0
    btn.Text = C[key].Name
    btn.TextColor3 = theme.accent2
    btn.TextSize = 12
    btn.Font = Enum.Font.GothamBold
    btn.Parent = row
    local c = Instance.new("UICorner"); c.CornerRadius = UDim.new(0, 6); c.Parent = btn

    btn.MouseButton1Click:Connect(safe(function()
        S.waitBind = {key = key, btn = btn}
        btn.Text = "..."
        btn.TextColor3 = theme.good
    end))
end

-- раскладка
addLabel("COMBAT", theme.accent, 14)
addToggle("Auto Parry",    "AutoParry")
addToggle("Auto Radius",   "AutoRadius")
addToggle("Only Incoming", "OnlyIncoming")
addToggle("Auto Swing",    "AutoSwing")
addToggle("Auto Spam",     "AutoSpam")

addLabel("SETTINGS", theme.accent, 14)
addSlider("Cooldown",    "Cooldown",     0.05, 0.5, 0.01)
addSlider("Min Radius",  "MinRadius",    5, 20, 1)
addSlider("Max Radius",  "MaxRadius",    12, 40, 1)
addSlider("Safety",      "SafetyFactor", 1, 2, 0.05)
addSlider("Smoothing",   "SmoothAlpha",  0.05, 0.9, 0.05)

addLabel("KEYBINDS", theme.accent, 14)
addBinder("Menu",  "MenuKey")
addBinder("Parry", "ParryKey")

-- кнопка-переключатель меню (плавающая, для мобилы)
local fab = Instance.new("TextButton")
fab.Size = UDim2.new(0, 44, 0, 44)
fab.Position = UDim2.new(0, 12, 0.5, -22)
fab.BackgroundColor3 = theme.accent
fab.Text = "≡"
fab.TextColor3 = theme.text
fab.TextSize = 22
fab.Font = Enum.Font.GothamBold
fab.BorderSizePixel = 0
fab.Parent = gui
local fabc = Instance.new("UICorner"); fabc.CornerRadius = UDim.new(1, 0); fabc.Parent = fab

fab.MouseButton1Click:Connect(safe(function()
    S.menuOpen = not S.menuOpen
    root.Visible = S.menuOpen
end))

-- перетаскивание за шапку (тач)
local dragging, dragStart, startPos
header.InputBegan:Connect(safe(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = root.Position
    end
end))
UIS.InputChanged:Connect(safe(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseMovement) then
        local delta = input.Position - dragStart
        root.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end))
UIS.InputEnded:Connect(safe(function(input)
    if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end))

-- захват клавиши для биндера (на мобиле — только внешняя клавиатура)
UIS.InputBegan:Connect(safe(function(input, gp)
    if S.waitBind and not gp and input.KeyCode ~= Enum.KeyCode.Unknown then
        C[S.waitBind.key] = input.KeyCode
        S.waitBind.btn.Text = input.KeyCode.Name
        S.waitBind.btn.TextColor3 = theme.accent2
        S.waitBind = nil
        return
    end
    if input.KeyCode == C.MenuKey then
        S.menuOpen = not S.menuOpen
        root.Visible = S.menuOpen
        return
    end
    if not S.menuOpen then return end
    if input.KeyCode == C.ParryKey then parry() end
    if input.KeyCode == C.SwingKey then fireRemote() end
end))

-- статус
RunService.RenderStepped:Connect(safe(function()
    if not S.menuOpen then return end
    status.Text = string.format("ping %dms r%.0f", math.floor(S.ping), S.radius)
    status.TextColor3 = S.ping < 100 and theme.good or (S.ping < 180 and theme.accent2 or theme.bad)
end))

-- чистка следов
task.spawn(safe(function()
    while task.wait(3) do
        pcall(function()
            for _,g in ipairs(LP.PlayerGui:GetChildren()) do
                if g.Name:find("Drawing") or g.Name:find("Delta") or g.Name:find("Jirnyak") then
                    g:Destroy()
                end
            end
        end)
    end
end))

pcall(function()
    game:GetService("StarterGui"):SetCore("SendNotification", {
        Title = "JIRNYAK V6", Text = "Delta build loaded", Duration = 3
    })
end)

print("[JIRNYAK V6 / Delta] ready. Remote:", S.remote and S.remote:GetFullName() or "NOT FOUND")
print("Menu:", C.MenuKey.Name, "| FAB: слева по центру")