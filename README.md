--[[
    HunterBlock v8 (Crimson UI)
    - Scoped sound hook (killers only, no global)
    - Auto Block (95 M1 sounds)
    - SSHT (velocity shake)
    - Box visuals
    - Auto Parry
    - Killer dropdown + refresh
    - Crimson tabbed UI (Block / SSHT / Info)
]]

-- ============================================================
-- WAIT FOR GAME AND MATCH
-- ============================================================
if not game:IsLoaded() then
    repeat task.wait(0.5) until game:IsLoaded()
end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

if _G._HunterBlockLoaded then return end
_G._HunterBlockLoaded = true

-- ============================================================
-- STATE
-- ============================================================
local S = {
    AutoBlock = false,
    BlockRange = 18,
    SSHTEnabled = false,
    SSHTMode = "Legit",
    SSHTSpeed = 45,
    ShowRangeBox = false,
    ShowFacingBox = false,
    ParryAim = true,
    ParryPunch = true,
    ManualKiller = "Auto",
}

-- ============================================================
-- NETWORK
-- ============================================================
local remoteEvent, networkModule

task.spawn(function()
    pcall(function()
        local modules = ReplicatedStorage:WaitForChild("Modules", 15)
        local net = modules:WaitForChild("Network", 8)
        local inner = net:WaitForChild("Network", 8)
        remoteEvent = inner:FindFirstChild("RemoteEvent") or inner:WaitForChild("RemoteEvent", 5)
        networkModule = require(inner)
    end)
end)

local function getRemote()
    if remoteEvent and remoteEvent.Parent then return remoteEvent end
    pcall(function() remoteEvent = ReplicatedStorage.Modules.Network.Network.RemoteEvent end)
    return remoteEvent
end

local function fireBlock()
    local re = getRemote()
    if not re then return end
    local ok = pcall(function() re:FireServer("UseActorAbility", { "Block" }) end)
    if not ok then
        pcall(function()
            re:FireServer("UseActorAbility", {
                (function(bytes)
                    local b = buffer.create(#bytes)
                    for i = 1, #bytes do buffer.writeu8(b, i - 1, bytes[i]) end
                    return b
                end)({3, 5, 0, 0, 0, 66, 108, 111, 99, 107})
            })
        end)
    end
end

local function firePunch()
    local re = getRemote()
    if not re then return end
    pcall(function()
        re:FireServer("UseActorAbility", {
            (function(bytes)
                local b = buffer.create(#bytes)
                for i = 1, #bytes do buffer.writeu8(b, i - 1, bytes[i]) end
                return b
            end)({3, 5, 0, 0, 0, 80, 117, 110, 99, 104})
        })
    end)
end

-- ============================================================
-- SOUND DB
-- ============================================================
local SOUND_DB = {
    ["102228729296384"]=true,["140242176732868"]=true,["112809109188560"]=true,
    ["136323728355613"]=true,["115026634746636"]=true,["84116622032112"]=true,
    ["108907358619313"]=true,["127793641088496"]=true,["86174610237192"]=true,
    ["95079963655241"]=true,["101199185291628"]=true,["119942598489800"]=true,
    ["84307400688050"]=true,["113037804008732"]=true,["105200830849301"]=true,
    ["75330693422988"]=true,["82221759983649"]=true,["81702359653578"]=true,
    ["108610718831698"]=true,["112395455254818"]=true,["109431876587852"]=true,
    ["109348678063422"]=true,["85853080745515"]=true,["98733709078792"]=true,
    ["105840448036441"]=true,["114742322778642"]=true,["105415540898010"]=true,
    ["106300477136129"]=true,["80516583309685"]=true,["116581754553533"]=true,
    ["71834552297085"]=true,["119583605486352"]=true,["117173212095661"]=true,
    ["104910828105172"]=true,["79980897195554"]=true,["116527305931161"]=true,
    ["131406927389838"]=true,["94317217837143"]=true,["121954639447247"]=true,
    ["128856426573270"]=true,["131123355704017"]=true,["107444859834748"]=true,
    ["133709029886490"]=true,
    ["12222216"]=true,["71805956520207"]=true,["79391273191671"]=true,
    ["89004992452376"]=true,["101553872555606"]=true,["101698569375359"]=true,
    ["117231507259853"]=true,["119089145505438"]=true,["125213046326879"]=true,
    ["86833981571073"]=true,["110372418055226"]=true,["86494585504534"]=true,
    ["140412278320643"]=true,["140194172008986"]=true,["85544168523099"]=true,
    ["114506382930939"]=true,["99829427721752"]=true,["120059928759346"]=true,
    ["104625283622511"]=true,["105316545074913"]=true,["126131675979001"]=true,
    ["82336352305186"]=true,["93366464803829"]=true,["84069821282466"]=true,
    ["128195973631079"]=true,["124903763333174"]=true,["98111231282218"]=true,
    ["136728245733659"]=true,["76959687420003"]=true,["72425554233832"]=true,
    ["96594507550917"]=true,["139996647355899"]=true,["107345261604889"]=true,
    ["127557531826290"]=true,["108651070773439"]=true,["74842815979546"]=true,
    ["124397369810639"]=true,["76467993976301"]=true,["118493324723683"]=true,
    ["78298577002481"]=true,["5148302439"]=true,["98675142200448"]=true,
    ["128367348686124"]=true,["103684883268194"]=true,["109246041199659"]=true,
    ["80540530406270"]=true,["139523195429581"]=true,["105204810054381"]=true,
    ["116468089135195"]=true,["124234993291213"]=true,["99856718263455"]=true,
    ["74809026448465"]=true,
}

local BAD_SOUNDS = { ["112809109188560"] = true }

-- ============================================================
-- HELPERS
-- ============================================================
local function getRoot(m)
    return m and (m:FindFirstChild("HumanoidRootPart") or m.PrimaryPart)
end

local function getKillersFolder()
    local p = Workspace:FindFirstChild("Players")
    if not p then return nil end
    return p:FindFirstChild("Killers")
end

local function getKillers()
    local f = getKillersFolder()
    if not f then return {} end
    local t = {}
    for _, c in ipairs(f:GetChildren()) do
        if c:IsA("Model") then t[#t+1] = c end
    end
    return t
end

local function getKillerHRP(model)
    if not model then return nil end
    return model:FindFirstChild("HumanoidRootPart")
        or model.PrimaryPart
        or model:FindFirstChildWhichIsA("BasePart", true)
end

local function getNearestKiller(maxDist)
    local our = getRoot(LocalPlayer.Character)
    if not our then return nil end
    local best, bestD = nil, maxDist or 999
    for _, k in ipairs(getKillers()) do
        local hrp = getKillerHRP(k)
        local hum = k:FindFirstChildOfClass("Humanoid")
        if hrp and hum and hum.Health > 0 then
            local d = (hrp.Position - our.Position).Magnitude
            if d < bestD then bestD, best = d, k end
        end
    end
    return best
end

local function faceTarget(hrp)
    if not hrp then return end
    local our = getRoot(LocalPlayer.Character)
    if not our then return end
    local flat = Vector3.new(hrp.Position.X, our.Position.Y, hrp.Position.Z)
    if (flat - our.Position).Magnitude > 0.1 then
        our.CFrame = CFrame.lookAt(our.Position, flat)
    end
end

-- ============================================================
-- BLOCK COOLDOWN
-- ============================================================
local cachedCooldown = nil
local lastCooldownRefresh = 0

local function refreshCooldownRef()
    pcall(function()
        local main = PlayerGui:FindFirstChild("MainUI")
        if not main then cachedCooldown = nil return end
        local ability = main:FindFirstChild("AbilityContainer")
        if not ability then cachedCooldown = nil return end
        local blockBtn = ability:FindFirstChild("Block")
        if not blockBtn then cachedCooldown = nil return end
        cachedCooldown = blockBtn:FindFirstChild("CooldownTime")
    end)
end

local function blockReady()
    local now = tick()
    if not cachedCooldown or not cachedCooldown.Parent or now - lastCooldownRefresh > 2 then
        refreshCooldownRef()
        lastCooldownRefresh = now
    end
    if not cachedCooldown then return true end
    return cachedCooldown.Text == "" or cachedCooldown.Text == " "
end

-- ============================================================
-- SSHT
-- ============================================================
local sshtActive = false
local sshtTargetHRP = nil
local sshtShakeSign = 1

local function startSSHT(hrp)
    if not S.SSHTEnabled or sshtActive then return end
    if not hrp or not hrp.Parent then return end
    sshtActive = true
    sshtTargetHRP = hrp
    sshtShakeSign = 1
end

local function stopSSHT()
    if not sshtActive then return end
    sshtActive = false
    sshtTargetHRP = nil
end

RunService.Heartbeat:Connect(function()
    if not sshtActive then return end
    if not sshtTargetHRP or not sshtTargetHRP.Parent then stopSSHT() return end
    local char = LocalPlayer.Character
    if not char then stopSSHT() return end
    local our = char:FindFirstChild("HumanoidRootPart")
    if not our then stopSSHT() return end

    local diff = sshtTargetHRP.Position - our.Position
    local dist = diff.Magnitude
    if dist < 1 then
        sshtShakeSign = -sshtShakeSign
        our.Velocity = our.Velocity + Vector3.new(0, 0.1 * sshtShakeSign, 0)
        return
    end

    local dir = diff.Unit
    if S.SSHTMode == "Legit" then
        our.Velocity = dir * S.SSHTSpeed
    else
        local shake = Vector3.new(
            math.random(-50, 50) / 50,
            math.random(-50, 50) / 50,
            math.random(-50, 50) / 50
        )
        our.Velocity = dir * (S.SSHTSpeed + dist * 10) + shake
    end
end)

-- ============================================================
-- BOX VISUALS
-- ============================================================
local function spawnBox(cf, size, color)
    if not size or size.X <= 0 or size.Y <= 0 or size.Z <= 0 then return end
    local part = Instance.new("Part")
    part.Name = "HB_Box"
    part.Size = size
    part.Material = Enum.Material.SmoothPlastic
    part.Transparency = 0.5
    part.Color = color
    part.Anchored = true
    part.CanCollide = false
    part.CanQuery = false
    part.CanTouch = false
    part.CastShadow = false
    part.Massless = true
    part.CFrame = cf
    part.Parent = Workspace

    local adorn = Instance.new("BoxHandleAdornment")
    adorn.Adornee = part
    adorn.AlwaysOnTop = true
    adorn.ZIndex = 5
    adorn.Size = size
    adorn.Transparency = 0.5
    adorn.Color3 = color
    adorn.Parent = part

    task.delay(0.25, function()
        pcall(function() part:Destroy() end)
    end)
end

local function showBoxes(killerModel)
    local hrp = getKillerHRP(killerModel)
    if not hrp then return end
    if S.ShowRangeBox then
        spawnBox(
            CFrame.new(hrp.Position + Vector3.new(0, -hrp.Size.Y/2 - 0.3, 0)),
            Vector3.new(S.BlockRange * 2, 0.5, S.BlockRange * 2),
            Color3.fromRGB(255, 220, 40)
        )
    end
    if S.ShowFacingBox then
        spawnBox(
            hrp.CFrame * CFrame.new(0, -hrp.Size.Y/2 - 0.3, -6),
            Vector3.new(6, 0.5, 12),
            Color3.fromRGB(255, 60, 60)
        )
    end
end

-- ============================================================
-- AUTO BLOCK
-- ============================================================
local lastBlock = 0

local function tryBlock(killer, hrp)
    if not S.AutoBlock then return end
    if not hrp or not hrp.Parent then return end
    local our = getRoot(LocalPlayer.Character)
    if not our then return end
    if (our.Position - hrp.Position).Magnitude > S.BlockRange then return end
    if os.clock() - lastBlock < 0.08 then return end
    if not blockReady() then return end
    lastBlock = os.clock()

    if S.SSHTEnabled then startSSHT(hrp) end

    for i = 1, 3 do
        fireBlock()
        if i < 3 then task.wait(0.03) end
    end

    showBoxes(killer)
end

-- ============================================================
-- SOUND HOOK (SCOPED — killers only)
-- ============================================================
local hookedSounds = setmetatable({}, { __mode = "k" })
local trackedKillers = {}

local function isM1(sound)
    if not sound or not sound:IsA("Sound") then return false end
    local id = tostring(sound.SoundId):match("%d+")
    if not id or BAD_SOUNDS[id] then return false end
    return SOUND_DB[id] == true
end

local function findKillerFromSound(sound)
    local model = sound:FindFirstAncestorOfClass("Model")
    if not model then return nil end
    local parent = model.Parent
    if not parent or parent.Name ~= "Killers" then return nil end
    return model
end

local function hookSound(sound)
    if not sound or not sound:IsA("Sound") then return end
    if hookedSounds[sound] then return end
    hookedSounds[sound] = true

    local function check()
        if not S.AutoBlock then return end
        if not isM1(sound) then return end
        local killer = findKillerFromSound(sound)
        if not killer then return end
        local hrp = getKillerHRP(killer)
        if not hrp then return end
        tryBlock(killer, hrp)
    end

    sound.Played:Connect(check)
    sound:GetPropertyChangedSignal("IsPlaying"):Connect(function()
        if sound.IsPlaying then check() end
    end)
    if sound.IsPlaying then check() end
end

local function trackKiller(killer)
    if not killer or trackedKillers[killer] then return end
    if not killer:IsA("Model") then return end
    trackedKillers[killer] = true

    for _, d in ipairs(killer:GetDescendants()) do
        if d:IsA("Sound") then hookSound(d) end
    end

    killer.DescendantAdded:Connect(function(d)
        if d:IsA("Sound") then
            task.defer(function() hookSound(d) end)
        end
    end)

    killer.Destroying:Connect(function()
        trackedKillers[killer] = nil
    end)
end

local function scanAllKillers()
    local folder = getKillersFolder()
    if not folder then return 0 end
    local count = 0
    for _, k in ipairs(folder:GetChildren()) do
        if k:IsA("Model") then
            trackKiller(k)
            count = count + 1
        end
    end
    return count
end

local function waitAndScan()
    local folder = getKillersFolder()
    if folder then
        scanAllKillers()
        folder.ChildAdded:Connect(function(k)
            if k:IsA("Model") then
                task.defer(function() trackKiller(k) end)
            end
        end)
        return
    end
    task.spawn(function()
        while true do
            task.wait(1)
            if getKillersFolder() then
                scanAllKillers()
                getKillersFolder().ChildAdded:Connect(function(k)
                    if k:IsA("Model") then
                        task.defer(function() trackKiller(k) end)
                    end
                end)
                return
            end
        end
    end)
end

waitAndScan()

-- ============================================================
-- AUTO PARRY
-- ============================================================
local parryConn = nil
local lastParry = 0

local function onParry()
    if os.clock() - lastParry < 0.2 then return end
    lastParry = os.clock()

    if S.ParryAim then
        local nearest = getNearestKiller(S.BlockRange * 1.5)
        if nearest then faceTarget(getKillerHRP(nearest)) end
    end

    if S.ParryPunch then
        task.wait(0.02)
        firePunch()
    end

    stopSSHT()
end

local function installParryHook()
    if parryConn then return end
    if not networkModule then
        task.spawn(function()
            local t0 = tick()
            while not networkModule and tick() - t0 < 15 do
                task.wait(0.5)
                pcall(function()
                    networkModule = require(ReplicatedStorage.Modules.Network.Network)
                end)
            end
            if networkModule then installParryHook() end
        end)
        return
    end
    pcall(function()
        parryConn = networkModule:SetConnection(
            ("%*1337ParryIcon"):format(LocalPlayer.Name),
            "REMOTE_EVENT",
            function(blocked)
                if blocked == true then onParry() end
            end
        )
    end)
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1.5)
    parryConn = nil
    installParryHook()
end)
task.delay(2, installParryHook)

-- ============================================================
-- DETECT
-- ============================================================
local function detectKillerNames()
    local killers = getKillers()
    local names = {}
    for _, k in ipairs(killers) do
        names[#names+1] = k.Name
    end
    return names
end

-- ============================================================
-- CRIMSON UI
-- ============================================================
local ACCENT     = Color3.fromRGB(220, 30, 60)
local ACCENT_DIM = Color3.fromRGB(120, 20, 35)
local BG_DARK    = Color3.fromRGB(16, 14, 16)
local BG_MID     = Color3.fromRGB(24, 20, 22)
local BG_ROW     = Color3.fromRGB(34, 28, 30)
local TEXT_MAIN  = Color3.fromRGB(240, 235, 238)
local TEXT_DIM   = Color3.fromRGB(150, 140, 145)
local STROKE     = Color3.fromRGB(48, 38, 42)

if _G._HunterBlockGui and _G._HunterBlockGui.Parent then
    pcall(function() _G._HunterBlockGui:Destroy() end)
end

local gui = Instance.new("ScreenGui")
gui.Name = "HunterBlockPanel"
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.IgnoreGuiInset = true
pcall(function() gui.Parent = (gethui and gethui()) or CoreGui end)
if not gui.Parent then gui.Parent = PlayerGui end
_G._HunterBlockGui = gui

-- Floating button
local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.fromOffset(46, 46)
toggleBtn.Position = UDim2.new(1, -100, 0.5, -23)
toggleBtn.BackgroundColor3 = BG_MID
toggleBtn.Text = "HB"
toggleBtn.TextColor3 = ACCENT
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextSize = 14
toggleBtn.AutoButtonColor = false
toggleBtn.Active = true
toggleBtn.Draggable = true
toggleBtn.Parent = gui
Instance.new("UICorner", toggleBtn).CornerRadius = UDim.new(1, 0)
local tbs = Instance.new("UIStroke", toggleBtn)
tbs.Color = ACCENT; tbs.Thickness = 1.5

-- Window
local window = Instance.new("Frame")
window.Size = UDim2.fromOffset(500, 400)
window.Position = UDim2.fromScale(0.5, 0.5)
window.AnchorPoint = Vector2.new(0.5, 0.5)
window.BackgroundColor3 = BG_DARK
window.BorderSizePixel = 0
window.Visible = false
window.ClipsDescendants = true
window.Parent = gui
Instance.new("UICorner", window).CornerRadius = UDim.new(0, 12)
local ws = Instance.new("UIStroke", window)
ws.Color = STROKE; ws.Thickness = 1

-- Header
local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 40)
header.BackgroundColor3 = BG_MID
header.BorderSizePixel = 0
header.Parent = window
Instance.new("UICorner", header).CornerRadius = UDim.new(0, 12)
local headerFix = Instance.new("Frame")
headerFix.Size = UDim2.new(1, 0, 0, 12)
headerFix.Position = UDim2.new(0, 0, 1, -12)
headerFix.BackgroundColor3 = BG_MID
headerFix.BorderSizePixel = 0
headerFix.Parent = header

local dot = Instance.new("Frame")
dot.Size = UDim2.fromOffset(10, 10)
dot.Position = UDim2.new(0, 14, 0.5, -5)
dot.BackgroundColor3 = ACCENT
dot.BorderSizePixel = 0
dot.Parent = header
Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)

local titleLbl = Instance.new("TextLabel")
titleLbl.Size = UDim2.new(0, 200, 1, 0)
titleLbl.Position = UDim2.fromOffset(34, 0)
titleLbl.BackgroundTransparency = 1
titleLbl.Text = "HunterBlock"
titleLbl.TextColor3 = TEXT_MAIN
titleLbl.Font = Enum.Font.GothamBold
titleLbl.TextSize = 14
titleLbl.TextXAlignment = Enum.TextXAlignment.Left
titleLbl.Parent = header

local verLbl = Instance.new("TextLabel")
verLbl.Size = UDim2.new(0, 60, 1, 0)
verLbl.Position = UDim2.new(0, 130, 0, 0)
verLbl.BackgroundTransparency = 1
verLbl.Text = "v8"
verLbl.TextColor3 = TEXT_DIM
verLbl.Font = Enum.Font.GothamMedium
verLbl.TextSize = 11
verLbl.TextXAlignment = Enum.TextXAlignment.Left
verLbl.Parent = header

local closeBtn = Instance.new("TextButton")
closeBtn.Size = UDim2.fromOffset(24, 24)
closeBtn.Position = UDim2.new(1, -34, 0.5, -12)
closeBtn.BackgroundColor3 = BG_DARK
closeBtn.Text = "×"
closeBtn.TextColor3 = ACCENT
closeBtn.Font = Enum.Font.GothamBold
closeBtn.TextSize = 16
closeBtn.AutoButtonColor = false
closeBtn.Parent = header
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 6)
closeBtn.MouseEnter:Connect(function() closeBtn.BackgroundColor3 = ACCENT end)
closeBtn.MouseLeave:Connect(function() closeBtn.BackgroundColor3 = BG_DARK end)

-- Sidebar
local sidebar = Instance.new("Frame")
sidebar.Size = UDim2.new(0, 110, 1, -40)
sidebar.Position = UDim2.new(0, 0, 0, 40)
sidebar.BackgroundColor3 = BG_MID
sidebar.BorderSizePixel = 0
sidebar.Parent = window

local sideFix = Instance.new("Frame")
sideFix.Size = UDim2.new(0, 12, 1, -40)
sideFix.Position = UDim2.new(1, -12, 0, 0)
sideFix.BackgroundColor3 = BG_MID
sideFix.BorderSizePixel = 0
sideFix.Parent = sidebar

local tabList = Instance.new("Frame")
tabList.Size = UDim2.new(1, 0, 1, -20)
tabList.Position = UDim2.new(0, 0, 0, 10)
tabList.BackgroundTransparency = 1
tabList.Parent = sidebar

local tabLayout = Instance.new("UIListLayout", tabList)
tabLayout.Padding = UDim.new(0, 6)
tabLayout.SortOrder = Enum.SortOrder.LayoutOrder
tabLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

-- Content
local content = Instance.new("Frame")
content.Size = UDim2.new(1, -110, 1, -40)
content.Position = UDim2.new(0, 110, 0, 40)
content.BackgroundTransparency = 1
content.Parent = window

local pages = {}
local tabButtons = {}

local function makePage(name)
    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.new(1, -20, 1, -20)
    page.Position = UDim2.new(0, 10, 0, 10)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = 3
    page.ScrollBarImageColor3 = ACCENT
    page.CanvasSize = UDim2.new(0, 0, 0, 0)
    page.AutomaticCanvasSize = Enum.AutomaticSize.Y
    page.Visible = false
    page.Parent = content

    local lay = Instance.new("UIListLayout", page)
    lay.Padding = UDim.new(0, 6)
    lay.SortOrder = Enum.SortOrder.LayoutOrder

    pages[name] = page
    return page
end

local function makeTab(name)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -20, 0, 32)
    b.BackgroundColor3 = BG_DARK
    b.Text = name
    b.TextColor3 = TEXT_DIM
    b.Font = Enum.Font.GothamMedium
    b.TextSize = 12
    b.AutoButtonColor = false
    b.Parent = tabList
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)

    tabButtons[name] = b

    b.MouseButton1Click:Connect(function()
        for _, p in pairs(pages) do p.Visible = false end
        for _, tb in pairs(tabButtons) do
            tb.TextColor3 = TEXT_DIM
            tb.BackgroundColor3 = BG_DARK
        end
        if pages[name] then pages[name].Visible = true end
        b.TextColor3 = ACCENT
        b.BackgroundColor3 = BG_ROW
    end)
    return b
end

local function sectionLabel(page, text)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 0, 20)
    l.BackgroundTransparency = 1
    l.Text = string.upper(text)
    l.TextColor3 = TEXT_DIM
    l.Font = Enum.Font.GothamBold
    l.TextSize = 10
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = page
    return l
end

local function toggleRow(page, name, initial, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 38)
    row.BackgroundColor3 = BG_ROW
    row.BorderSizePixel = 0
    row.Parent = page
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 6)

    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, -70, 1, 0)
    l.Position = UDim2.fromOffset(12, 0)
    l.BackgroundTransparency = 1
    l.Text = name
    l.TextColor3 = TEXT_MAIN
    l.Font = Enum.Font.GothamMedium
    l.TextSize = 12
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = row

    local state = initial

    local switch = Instance.new("TextButton")
    switch.Size = UDim2.fromOffset(38, 20)
    switch.Position = UDim2.new(1, -50, 0.5, -10)
    switch.BackgroundColor3 = state and ACCENT or Color3.fromRGB(55, 45, 48)
    switch.Text = ""
    switch.AutoButtonColor = false
    switch.Parent = row
    Instance.new("UICorner", switch).CornerRadius = UDim.new(1, 0)

    local knob = Instance.new("Frame")
    knob.Size = UDim2.fromOffset(14, 14)
    knob.Position = state and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7)
    knob.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    knob.BorderSizePixel = 0
    knob.Parent = switch
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)

    local api = {}
    api.set = function(v)
        state = v
        switch.BackgroundColor3 = v and ACCENT or Color3.fromRGB(55, 45, 48)
        knob.Position = v and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7)
    end

    switch.MouseButton1Click:Connect(function()
        state = not state
        api.set(state)
        if callback then callback(state) end
    end)

    return api
end

local function sliderRow(page, name, minV, maxV, initial, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 46)
    row.BackgroundColor3 = BG_ROW
    row.BorderSizePixel = 0
    row.Parent = page
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 6)

    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, -70, 0, 18)
    l.Position = UDim2.fromOffset(12, 4)
    l.BackgroundTransparency = 1
    l.Text = name
    l.TextColor3 = TEXT_MAIN
    l.Font = Enum.Font.GothamMedium
    l.TextSize = 12
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = row

    local vl = Instance.new("TextLabel")
    vl.Size = UDim2.new(0, 60, 0, 18)
    vl.Position = UDim2.new(1, -72, 0, 4)
    vl.BackgroundTransparency = 1
    vl.Text = tostring(math.floor(initial * 100) / 100)
    vl.TextColor3 = ACCENT
    vl.Font = Enum.Font.GothamBold
    vl.TextSize = 12
    vl.TextXAlignment = Enum.TextXAlignment.Right
    vl.Parent = row

    local bg = Instance.new("Frame")
    bg.Size = UDim2.new(1, -24, 0, 6)
    bg.Position = UDim2.new(0, 12, 0, 32)
    bg.BackgroundColor3 = Color3.fromRGB(50, 42, 45)
    bg.BorderSizePixel = 0
    bg.Parent = row
    Instance.new("UICorner", bg).CornerRadius = UDim.new(1, 0)

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(0, 0, 1, 0)
    fill.BackgroundColor3 = ACCENT
    fill.BorderSizePixel = 0
    fill.Parent = bg
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

    local function setV(v)
        local pct = (v - minV) / (maxV - minV)
        fill.Size = UDim2.new(pct, 0, 1, 0)
        vl.Text = tostring(math.floor(v * 100) / 100)
    end
    setV(initial)

    local dragging = false
    local function apply(x)
        local abs = bg.AbsolutePosition.X
        local w = bg.AbsoluteSize.X
        local pct = math.clamp((x - abs) / w, 0, 1)
        local v = minV + pct * (maxV - minV)
        setV(v)
        if callback then callback(v) end
    end

    bg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
           or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            apply(input.Position.X)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement
           or input.UserInputType == Enum.UserInputType.Touch then
            apply(input.Position.X)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
           or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

local function dropdownRow(page, name, options, initial, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 38)
    row.BackgroundColor3 = BG_ROW
    row.BorderSizePixel = 0
    row.Parent = page
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 6)

    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, -130, 1, 0)
    l.Position = UDim2.fromOffset(12, 0)
    l.BackgroundTransparency = 1
    l.Text = name
    l.TextColor3 = TEXT_MAIN
    l.Font = Enum.Font.GothamMedium
    l.TextSize = 12
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = row

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.fromOffset(100, 24)
    btn.Position = UDim2.new(1, -112, 0.5, -12)
    btn.BackgroundColor3 = BG_DARK
    btn.Text = initial
    btn.TextColor3 = ACCENT
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.AutoButtonColor = false
    btn.Parent = row
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 5)

    local idx = 1
    for i, opt in ipairs(options) do
        if opt == initial then idx = i break end
    end

    btn.MouseButton1Click:Connect(function()
        idx = idx + 1
        if idx > #options then idx = 1 end
        btn.Text = options[idx]
        if callback then callback(options[idx]) end
    end)
    return btn
end

local function buttonRow(page, name, callback, color)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, 0, 0, 34)
    b.BackgroundColor3 = color or BG_ROW
    b.Text = name
    b.TextColor3 = color and Color3.new(1, 1, 1) or TEXT_MAIN
    b.Font = Enum.Font.GothamMedium
    b.TextSize = 12
    b.AutoButtonColor = false
    b.Parent = page
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)

    b.MouseEnter:Connect(function()
        b.BackgroundColor3 = (color or BG_ROW):Lerp(Color3.new(1,1,1), 0.12)
    end)
    b.MouseLeave:Connect(function()
        b.BackgroundColor3 = color or BG_ROW
    end)
    b.MouseButton1Click:Connect(function()
        if callback then callback() end
    end)
    return b
end

-- ============================================================
-- BUILD TABS
-- ============================================================
makeTab("Block")
makeTab("SSHT")
makeTab("Info")

local blockPage = makePage("Block")
local sshtPage = makePage("SSHT")
local infoPage = makePage("Info")

-- BLOCK page
sectionLabel(blockPage, "Auto Block")
toggleRow(blockPage, "Enable Auto Block", S.AutoBlock, function(v) S.AutoBlock = v end)
sliderRow(blockPage, "Block Range", 5, 40, S.BlockRange, function(v) S.BlockRange = v end)

sectionLabel(blockPage, "Auto Parry")
toggleRow(blockPage, "Aim on Parry", S.ParryAim, function(v) S.ParryAim = v end)
toggleRow(blockPage, "Punch on Parry", S.ParryPunch, function(v) S.ParryPunch = v end)

sectionLabel(blockPage, "Box Visuals")
toggleRow(blockPage, "Show Range Box", S.ShowRangeBox, function(v) S.ShowRangeBox = v end)
toggleRow(blockPage, "Show Facing Box", S.ShowFacingBox, function(v) S.ShowFacingBox = v end)

-- SSHT page
sectionLabel(sshtPage, "SSHT")
toggleRow(sshtPage, "Enable SSHT", S.SSHTEnabled, function(v) S.SSHTEnabled = v end)
dropdownRow(sshtPage, "SSHT Mode", {"Legit", "Blatant"}, S.SSHTMode, function(v) S.SSHTMode = v end)
sliderRow(sshtPage, "SSHT Speed", 5, 100, S.SSHTSpeed, function(v) S.SSHTSpeed = v end)

-- INFO page
sectionLabel(infoPage, "Killer Detection")
dropdownRow(infoPage, "Manual Killer", {"Auto", "c00lkidd", "Slasher", "Noli", "JohnDoe", "1x1x1x1", "Sixer", "Nosferatu", "Azure", "All"}, S.ManualKiller, function(v)
    S.ManualKiller = v
end)

local statusLbl = Instance.new("TextLabel")
statusLbl.Size = UDim2.new(1, 0, 0, 22)
statusLbl.BackgroundTransparency = 1
statusLbl.Text = "Scanning..."
statusLbl.TextColor3 = Color3.fromRGB(255, 220, 100)
statusLbl.Font = Enum.Font.GothamMedium
statusLbl.TextSize = 11
statusLbl.TextXAlignment = Enum.TextXAlignment.Left
statusLbl.Parent = infoPage

buttonRow(infoPage, "Refresh Killers", function()
    local count = scanAllKillers()
    if count > 0 then
        local names = detectKillerNames()
        statusLbl.Text = "Refreshed — " .. count .. " killers: " .. table.concat(names, ", ")
        statusLbl.TextColor3 = Color3.fromRGB(90, 255, 150)
    else
        statusLbl.Text = "Refreshed — 0 killers found"
        statusLbl.TextColor3 = Color3.fromRGB(255, 100, 100)
    end
end, ACCENT_DIM)

-- status updater
task.spawn(function()
    while gui.Parent do
        task.wait(2)
        pcall(function()
            local names = detectKillerNames()
            if #names > 0 then
                statusLbl.Text = #names .. " killers: " .. table.concat(names, ", ")
                statusLbl.TextColor3 = Color3.fromRGB(90, 255, 150)
            else
                statusLbl.Text = "No killers yet — waiting"
                statusLbl.TextColor3 = Color3.fromRGB(255, 180, 60)
            end
        end)
    end
end)

-- Default tab
for _, tb in pairs(tabButtons) do
    tb.TextColor3 = TEXT_DIM
    tb.BackgroundColor3 = BG_DARK
end
tabButtons["Block"].TextColor3 = ACCENT
tabButtons["Block"].BackgroundColor3 = BG_ROW
pages["Block"].Visible = true

-- ============================================================
-- OPEN / CLOSE
-- ============================================================
local windowOpen = false
local function setOpen(v)
    windowOpen = v
    window.Visible = v
    toggleBtn.Visible = not v
end

toggleBtn.MouseButton1Click:Connect(function() setOpen(true) end)
closeBtn.MouseButton1Click:Connect(function() setOpen(false) end)

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.K then
        setOpen(not windowOpen)
    end
end)

-- ============================================================
-- DRAG
-- ============================================================
local dragging, dragStart, startPos = false, nil, nil

header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
       or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = window.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not dragging then return end
    if input.UserInputType == Enum.UserInputType.MouseMovement
       or input.UserInputType == Enum.UserInputType.Touch then
        local delta = input.Position - dragStart
        window.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + delta.X,
            startPos.Y.Scale, startPos.Y.Offset + delta.Y
        )
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
       or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
        dragStart = nil
    end
end)

print("[HunterBlock] v8 loaded.")
