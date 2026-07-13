local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")
local VirtualUser = game:GetService("VirtualUser")

local player = Players.LocalPlayer
local farmEnabled = false
local antiAfkEnabled = false
local antiAfkConnection = nil
local menuVisible = false
local timerVisible = false
local farmStartTime = 0
local timerRunning = false
local timerUpdateConnection = nil
local farmThread = nil
local lastText = "00:00:00"

-- Загрузка сохранённых настроек
local function loadSettings()
    local saved = player:GetAttribute("FarmSettings")
    if saved then
        local data = game:GetService("HttpService"):JSONDecode(saved)
        return data
    end
    return nil
end

local function saveSettings(data)
    local json = game:GetService("HttpService"):JSONEncode(data)
    player:SetAttribute("FarmSettings", json)
end

local settings = loadSettings() or {}

local waypoints = {
    Vector3.new(-1457, -161, -913),
    Vector3.new(-1465, -160, -938),
    Vector3.new(-1483, -135, -764),
    Vector3.new(-1476, -87, -591),
    Vector3.new(-1465, -62, -414),
    Vector3.new(-1464, -57, -255),
    Vector3.new(-1461, -90, -122),
    Vector3.new(-1482, -56, -16)
}

local tracks = {
    ADMIN = Vector3.new(-1505, -157, -1061),
    CANDY = Vector3.new(-1480, -157, -1061),
    DIAMOND = Vector3.new(-1401, -157, -1061),
    GOLD = Vector3.new(-1421, -158, -1061),
    DEFAULT = Vector3.new(-1439, -158, -1061)
}

local selectedTrack = settings.selectedTrack or "DEFAULT"
local bindKey = settings.bindKey and Enum.KeyCode[settings.bindKey] or Enum.KeyCode.X
local farmEnabledState = settings.farmEnabled or false
local antiAfkEnabledState = settings.antiAfkEnabled or false
local timerVisibleState = settings.timerVisible or false

local function makeDraggable(frame)
    local dragging, dragInput, dragStart, startPos
    frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
            if frame.Name == "MenuButton" then
                TweenService:Create(frame, TweenInfo.new(0.15, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                    Size = UDim2.new(0, 72, 0, 72),
                    Rotation = 5
                }):Play()
            end
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then 
                    dragging = false
                    if frame.Name == "MenuButton" then
                        TweenService:Create(frame, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                            Size = UDim2.new(0, 65, 0, 65),
                            Rotation = 0
                        }):Play()
                    end
                end
            end)
        end
    end)
    frame.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

if CoreGui:FindFirstChild("BeautifulBlueGui") then
    CoreGui.BeautifulBlueGui:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BeautifulBlueGui"
ScreenGui.Parent = CoreGui
ScreenGui.ResetOnSpawn = false

----------------------------------------------------------------
-- ТАЙМЕР (С РАБОТАЮЩЕЙ АНИМАЦИЕЙ ЦИФР)
----------------------------------------------------------------
local TimerFrame = Instance.new("Frame")
TimerFrame.Name = "TimerFrame"
TimerFrame.Parent = ScreenGui
TimerFrame.Size = UDim2.new(0, 180, 0, 50)
TimerFrame.Position = UDim2.new(0.85, -90, 0.05, 0)
TimerFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
TimerFrame.BackgroundTransparency = 0.01
TimerFrame.Active = true
TimerFrame.Visible = timerVisibleState
makeDraggable(TimerFrame)

local TimerCorner = Instance.new("UICorner")
TimerCorner.CornerRadius = UDim.new(0, 10)
TimerCorner.Parent = TimerFrame

local TimerLabel = Instance.new("TextLabel")
TimerLabel.Name = "TimerLabel"
TimerLabel.Parent = TimerFrame
TimerLabel.Size = UDim2.new(1, 0, 1, 0)
TimerLabel.BackgroundTransparency = 1
TimerLabel.Text = "00:00:00"
TimerLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TimerLabel.TextSize = 22
TimerLabel.Font = Enum.Font.GothamBold
TimerLabel.TextScaled = false
TimerLabel.TextWrapped = true

local function updateTimer()
    if not timerRunning then
        TimerLabel.Text = "00:00:00"
        return
    end
    local elapsed = os.time() - farmStartTime
    local hours = math.floor(elapsed / 3600)
    local minutes = math.floor((elapsed % 3600) / 60)
    local seconds = elapsed % 60
    local newText = string.format("%02d:%02d:%02d", hours, minutes, seconds)
    
    if newText ~= lastText then
        lastText = newText
        TimerLabel.Text = newText
        TweenService:Create(TimerLabel, TweenInfo.new(0.1, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            TextSize = 30
        }):Play()
        task.wait(0.08)
        TweenService:Create(TimerLabel, TweenInfo.new(0.1, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            TextSize = 22
        }):Play()
    end
end

local function startTimer()
    farmStartTime = os.time()
    timerRunning = true
    if timerUpdateConnection then timerUpdateConnection:Disconnect() end
    timerUpdateConnection = game:GetService("RunService").Heartbeat:Connect(updateTimer)
end

local function stopTimer()
    timerRunning = false
    if timerUpdateConnection then
        timerUpdateConnection:Disconnect()
        timerUpdateConnection = nil
    end
    TimerLabel.Text = "00:00:00"
    lastText = "00:00:00"
end

----------------------------------------------------------------
-- КНОПКА-КВАДРАТ
----------------------------------------------------------------
local MenuButton = Instance.new("ImageButton")
MenuButton.Name = "MenuButton"
MenuButton.Parent = ScreenGui
MenuButton.Size = UDim2.new(0, 65, 0, 65)
MenuButton.Position = UDim2.new(0.03, 0, 0.25, 0)
MenuButton.BackgroundColor3 = Color3.fromRGB(10, 10, 12)
MenuButton.Image = "rbxassetid://160264132"
MenuButton.ImageColor3 = Color3.fromRGB(15, 15, 18)
MenuButton.ScaleType = Enum.ScaleType.Crop
MenuButton.Visible = true
MenuButton.AutoButtonColor = false
MenuButton.ClipsDescendants = true
MenuButton.ZIndex = 1
makeDraggable(MenuButton)

local MenuCorner = Instance.new("UICorner")
MenuCorner.CornerRadius = UDim.new(0, 10)
MenuCorner.Parent = MenuButton

local FabricOverlay = Instance.new("ImageLabel")
FabricOverlay.Name = "FabricOverlay"
FabricOverlay.Parent = MenuButton
FabricOverlay.Size = UDim2.new(1, 0, 1, 0)
FabricOverlay.BackgroundTransparency = 1
FabricOverlay.Image = "rbxassetid://160264132"
FabricOverlay.ImageColor3 = Color3.fromRGB(5, 5, 8)
FabricOverlay.ImageTransparency = 0.3
FabricOverlay.ScaleType = Enum.ScaleType.Crop
FabricOverlay.ZIndex = 1

local LetterLabel = Instance.new("TextLabel")
LetterLabel.Name = "LetterLabel"
LetterLabel.Parent = MenuButton
LetterLabel.Size = UDim2.new(0.7, 0, 0.7, 0)
LetterLabel.Position = UDim2.new(0.15, 0, 0.15, 0)
LetterLabel.BackgroundTransparency = 1
LetterLabel.Text = "W"
LetterLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
LetterLabel.TextSize = 40
LetterLabel.Font = Enum.Font.GothamBlack
LetterLabel.TextScaled = true
LetterLabel.TextWrapped = true
LetterLabel.ZIndex = 2

local snowContainer = Instance.new("Frame")
snowContainer.Parent = MenuButton
snowContainer.Size = UDim2.new(1, 0, 1, 0)
snowContainer.BackgroundTransparency = 1
snowContainer.ZIndex = 3
snowContainer.ClipsDescendants = true

local snowflakes = {}
for i = 1, 50 do
    local snow = Instance.new("TextLabel")
    snow.Parent = snowContainer
    snow.Size = UDim2.new(0, 4, 0, 4)
    snow.Position = UDim2.new(math.random() * 0.95, 0, math.random() * 0.95, 0)
    snow.BackgroundTransparency = 1
    snow.Text = "•"
    snow.TextColor3 = Color3.fromRGB(200, 200, 200)
    snow.TextSize = 6
    snow.Font = Enum.Font.GothamBold
    snow.TextScaled = true
    snow.TextWrapped = true
    snow.ZIndex = 4
    table.insert(snowflakes, snow)
end

task.spawn(function()
    while MenuButton and MenuButton.Parent do
        for _, snow in ipairs(snowflakes) do
            local pos = snow.Position
            local newY = pos.Y.Scale + 0.008
            if newY > 1 then newY = 0 end
            snow.Position = UDim2.new(pos.X.Scale, 0, newY, 0)
        end
        task.wait(0.03)
    end
end)

----------------------------------------------------------------
-- БОЛЬШИЕ СНЕЖИНКИ (200 штук)
----------------------------------------------------------------
local bigSnowContainer = Instance.new("Frame")
bigSnowContainer.Name = "BigSnowContainer"
bigSnowContainer.Parent = ScreenGui
bigSnowContainer.Size = UDim2.new(1, 0, 1, 0)
bigSnowContainer.BackgroundTransparency = 1
bigSnowContainer.ZIndex = 5
bigSnowContainer.Visible = false
bigSnowContainer.ClipsDescendants = false

local bigSnowflakes = {}
for i = 1, 200 do
    local snow = Instance.new("TextLabel")
    snow.Parent = bigSnowContainer
    snow.Size = UDim2.new(0, 18, 0, 18)
    snow.Position = UDim2.new(math.random() * 0.98, 0, math.random() * 0.98, 0)
    snow.BackgroundTransparency = 1
    snow.Text = "*"
    snow.TextColor3 = Color3.fromRGB(255, 255, 255)
    snow.TextSize = 22
    snow.Font = Enum.Font.GothamBold
    snow.TextTransparency = 0
    snow.ZIndex = 5
    table.insert(bigSnowflakes, snow)
end

task.spawn(function()
    while bigSnowContainer and bigSnowContainer.Parent do
        for _, snow in ipairs(bigSnowflakes) do
            local pos = snow.Position
            local newY = pos.Y.Scale + 0.005
            if newY > 1 then newY = 0 end
            snow.Position = UDim2.new(pos.X.Scale, 0, newY, 0)
        end
        task.wait(0.05)
    end
end)

----------------------------------------------------------------
-- ГЛАВНОЕ МЕНЮ (420x360)
----------------------------------------------------------------
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.Size = UDim2.new(0, 420, 0, 360)
MainFrame.Position = UDim2.new(0.5, -210, 0.5, -180)
MainFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
MainFrame.Active = true
MainFrame.Visible = false
MainFrame.BackgroundTransparency = 1
MainFrame.ZIndex = 10
makeDraggable(MainFrame)

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 12)
MainCorner.Parent = MainFrame

local HeaderFrame = Instance.new("Frame")
HeaderFrame.Name = "HeaderFrame"
HeaderFrame.Parent = MainFrame
HeaderFrame.Size = UDim2.new(1, 0, 0, 50)
HeaderFrame.BackgroundTransparency = 1
HeaderFrame.ZIndex = 11

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Parent = HeaderFrame
TitleLabel.Size = UDim2.new(1, 0, 0, 30)
TitleLabel.Position = UDim2.new(0, 0, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.Text = "BEST FARM"
TitleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleLabel.TextSize = 24
TitleLabel.Font = Enum.Font.GothamBlack
TitleLabel.TextXAlignment = Enum.TextXAlignment.Center
TitleLabel.ZIndex = 12

local smallText = Instance.new("TextLabel")
smallText.Parent = HeaderFrame
smallText.Size = UDim2.new(1, 0, 0, 20)
smallText.Position = UDim2.new(0, 0, 0, 30)
smallText.BackgroundTransparency = 1
smallText.Text = "3 WORLD"
smallText.TextColor3 = Color3.fromRGB(255, 255, 255)
smallText.TextSize = 14
smallText.Font = Enum.Font.Gotham
smallText.TextXAlignment = Enum.TextXAlignment.Center
smallText.ZIndex = 12

local Line = Instance.new("Frame")
Line.Parent = MainFrame
Line.Size = UDim2.new(0.9, 0, 0, 1)
Line.Position = UDim2.new(0.05, 0, 0, 55)
Line.BackgroundColor3 = Color3.fromRGB(150, 150, 150)
Line.ZIndex = 13

local buttonWidth = 400
local startX = 0.5 - (buttonWidth / 2 / 420)

local btnH = 45
local gap = 5

local function createButton(text, yPos, bgColor, textColor)
    local btn = Instance.new("TextButton")
    btn.Parent = MainFrame
    btn.Size = UDim2.new(0, buttonWidth, 0, btnH)
    btn.Position = UDim2.new(startX, 0, 0, yPos)
    btn.BackgroundColor3 = bgColor
    btn.Text = text
    btn.TextColor3 = textColor
    btn.TextSize = 13
    btn.Font = Enum.Font.GothamBold
    btn.ZIndex = 20
    btn.BackgroundTransparency = 0
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = btn
    return btn
end

local ToggleButton = createButton("AUTO FARM: OFF", 62, Color3.fromRGB(80, 80, 80), Color3.fromRGB(255, 255, 255))
local AntiAfkButton = createButton("ANTI AFK: OFF", 62 + btnH + gap, Color3.fromRGB(80, 80, 80), Color3.fromRGB(255, 255, 255))
local BindButton = createButton("BIND: " .. tostring(bindKey):gsub("Enum.KeyCode.", "") .. " (click to change it)", 62 + (btnH + gap) * 2, Color3.fromRGB(80, 80, 80), Color3.fromRGB(255, 255, 255))
local TimerToggleButton = createButton("TIMER FARM: OFF", 62 + (btnH + gap) * 3, Color3.fromRGB(80, 80, 80), Color3.fromRGB(255, 255, 255))
local TeleportButton = createButton("TREADMILL FARM", 62 + (btnH + gap) * 4, Color3.fromRGB(80, 80, 80), Color3.fromRGB(255, 255, 255))

local TrackSelectButton = Instance.new("TextButton")
TrackSelectButton.Name = "TrackSelectButton"
TrackSelectButton.Parent = MainFrame
TrackSelectButton.Size = UDim2.new(0, buttonWidth, 0, btnH - 2)
TrackSelectButton.Position = UDim2.new(startX, 0, 0, 62 + (btnH + gap) * 5)
TrackSelectButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TrackSelectButton.Text = "TREADMILL: " .. selectedTrack
TrackSelectButton.TextColor3 = Color3.fromRGB(120, 120, 120)
TrackSelectButton.TextSize = 12
TrackSelectButton.Font = Enum.Font.GothamBold
TrackSelectButton.ZIndex = 20
TrackSelectButton.BackgroundTransparency = 0
local TrackCorner = Instance.new("UICorner")
TrackCorner.CornerRadius = UDim.new(0, 6)
TrackCorner.Parent = TrackSelectButton

if farmEnabledState then
    ToggleButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ToggleButton.TextColor3 = Color3.fromRGB(120, 120, 120)
    ToggleButton.Text = "AUTO FARM: ON"
end
if antiAfkEnabledState then
    AntiAfkButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    AntiAfkButton.TextColor3 = Color3.fromRGB(120, 120, 120)
    AntiAfkButton.Text = "ANTI AFK: ON"
end
if timerVisibleState then
    TimerToggleButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    TimerToggleButton.TextColor3 = Color3.fromRGB(120, 120, 120)
    TimerToggleButton.Text = "TIMER FARM: ON"
end

local childElements = {}
for _, child in ipairs(MainFrame:GetChildren()) do
    if child:IsA("TextButton") or child:IsA("Frame") or child:IsA("TextLabel") then
        table.insert(childElements, child)
    end
end

----------------------------------------------------------------
-- АНИМАЦИЯ МЕНЮ
----------------------------------------------------------------
local function animateMenu(open)
    if open then
        TweenService:Create(MenuButton, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Size = UDim2.new(0, 80, 0, 80),
            Rotation = -10
        }):Play()
        task.wait(0.1)
        TweenService:Create(MenuButton, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Size = UDim2.new(0, 55, 0, 55),
            Rotation = 5
        }):Play()
        task.wait(0.1)
        TweenService:Create(MenuButton, TweenInfo.new(0.15, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Size = UDim2.new(0, 65, 0, 65),
            Rotation = 0
        }):Play()
        
        bigSnowContainer.Visible = true
        MainFrame.Visible = true
        MainFrame.BackgroundTransparency = 1
        MainFrame.Size = UDim2.new(0, 0, 0, 0)
        MainFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
        for _, child in ipairs(childElements) do
            child.Visible = false
        end
        local tween1 = TweenService:Create(MainFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Position = UDim2.new(0.5, -210, 0.5, -180),
            Size = UDim2.new(0, 420, 0, 360),
            BackgroundTransparency = 0
        })
        tween1:Play()
        tween1.Completed:Wait()
        for _, child in ipairs(childElements) do
            child.Visible = true
        end
    else
        bigSnowContainer.Visible = false
        for _, child in ipairs(childElements) do
            child.Visible = false
        end
        local tween1 = TweenService:Create(MainFrame, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Position = UDim2.new(0.5, 0, 0.5, 0),
            Size = UDim2.new(0, 0, 0, 0),
            BackgroundTransparency = 1
        })
        tween1:Play()
        tween1.Completed:Wait()
        MainFrame.Visible = false
        MainFrame.Size = UDim2.new(0, 420, 0, 360)
        MainFrame.BackgroundTransparency = 0
        MainFrame.Position = UDim2.new(0.5, -210, 0.5, -180)
    end
end

MenuButton.MouseButton1Click:Connect(function()
    menuVisible = not menuVisible
    animateMenu(menuVisible)
end)

----------------------------------------------------------------
-- БИНД
----------------------------------------------------------------
local listeningForBind = false

local function toggleMenuByBind()
    menuVisible = not menuVisible
    animateMenu(menuVisible)
end

local bindConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if listeningForBind then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            bindKey = input.KeyCode
            listeningForBind = false
            BindButton.Text = "BIND: " .. tostring(bindKey):gsub("Enum.KeyCode.", "") .. " (click to change it)"
            BindButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
            settings.bindKey = tostring(bindKey)
            saveSettings(settings)
        end
        return
    end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == bindKey then
        toggleMenuByBind()
    end
end)

BindButton.MouseButton1Click:Connect(function()
    listeningForBind = true
    BindButton.BackgroundColor3 = Color3.fromRGB(200, 100, 50)
    BindButton.Text = "PRESS ANY KEY..."
end)

----------------------------------------------------------------
-- ТАЙМЕР ВКЛ/ВЫКЛ
----------------------------------------------------------------
TimerToggleButton.MouseButton1Click:Connect(function()
    timerVisible = not timerVisible
    TimerFrame.Visible = timerVisible
    settings.timerVisible = timerVisible
    saveSettings(settings)
    if timerVisible then
        TimerToggleButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        TimerToggleButton.TextColor3 = Color3.fromRGB(120, 120, 120)
        TimerToggleButton.Text = "TIMER FARM: ON"
    else
        TimerToggleButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
        TimerToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        TimerToggleButton.Text = "TIMER FARM: OFF"
    end
end)

----------------------------------------------------------------
-- ТЕЛЕПОРТ
----------------------------------------------------------------
local function teleportToTrack(trackName)
    local targetPos = tracks[trackName]
    if not targetPos then return end
    local character = player.Character
    if not character then return end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if rootPart then
        local safePos = Vector3.new(targetPos.X, targetPos.Y + 5, targetPos.Z)
        rootPart.CFrame = CFrame.new(safePos)
    end
end

TeleportButton.MouseButton1Click:Connect(function()
    teleportToTrack(selectedTrack)
end)

local trackList = {"ADMIN", "CANDY", "DIAMOND", "GOLD", "DEFAULT"}
local trackIndex = 5
for i, name in ipairs(trackList) do
    if name == selectedTrack then
        trackIndex = i
        break
    end
end

TrackSelectButton.MouseButton1Click:Connect(function()
    trackIndex = trackIndex % #trackList + 1
    selectedTrack = trackList[trackIndex]
    TrackSelectButton.Text = "TREADMILL: " .. selectedTrack
    settings.selectedTrack = selectedTrack
    saveSettings(settings)
end)

----------------------------------------------------------------
-- АВТО ФАРМ
----------------------------------------------------------------
local function startFarm()
    while farmEnabled do
        local character = player.Character
        local rootPart = character and character:FindFirstChild("HumanoidRootPart")
        if rootPart then
            for i, targetPos in ipairs(waypoints) do
                if not farmEnabled then break end
                local distance = (rootPart.Position - targetPos).Magnitude
                local duration = distance / 220
                local tween = TweenService:Create(rootPart, TweenInfo.new(duration, Enum.EasingStyle.Linear), {CFrame = CFrame.new(targetPos)})
                tween:Play()
                tween.Completed:Wait()
            end
            if farmEnabled then task.wait(0.5) end
        else
            task.wait(0.5)
        end
        task.wait(0.1)
    end
end

ToggleButton.MouseButton1Click:Connect(function()
    farmEnabled = not farmEnabled
    settings.farmEnabled = farmEnabled
    saveSettings(settings)
    if farmEnabled then
        ToggleButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        ToggleButton.TextColor3 = Color3.fromRGB(120, 120, 120)
        ToggleButton.Text = "AUTO FARM: ON"
        startTimer()
        farmThread = task.spawn(startFarm)
    else
        ToggleButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
        ToggleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        ToggleButton.Text = "AUTO FARM: OFF"
        stopTimer()
        farmThread = nil
    end
end)

AntiAfkButton.MouseButton1Click:Connect(function()
    antiAfkEnabled = not antiAfkEnabled
    settings.antiAfkEnabled = antiAfkEnabled
    saveSettings(settings)
    if antiAfkEnabled then
        AntiAfkButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        AntiAfkButton.TextColor3 = Color3.fromRGB(120, 120, 120)
        AntiAfkButton.Text = "ANTI AFK: ON"
        antiAfkConnection = player.Idled:Connect(function()
            VirtualUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
            task.wait(0.5)
            VirtualUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        end)
    else
        AntiAfkButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
        AntiAfkButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        AntiAfkButton.Text = "ANTI AFK: OFF"
        if antiAfkConnection then
            antiAfkConnection:Disconnect()
            antiAfkConnection = nil
        end
    end
end)

if farmEnabledState then
    farmEnabled = true
    ToggleButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ToggleButton.TextColor3 = Color3.fromRGB(120, 120, 120)
    ToggleButton.Text = "AUTO FARM: ON"
    startTimer()
    farmThread = task.spawn(startFarm)
end

if antiAfkEnabledState then
    antiAfkEnabled = true
    AntiAfkButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    AntiAfkButton.TextColor3 = Color3.fromRGB(120, 120, 120)
    AntiAfkButton.Text = "ANTI AFK: ON"
    antiAfkConnection = player.Idled:Connect(function()
        VirtualUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
        task.wait(0.5)
        VirtualUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    end)
end

print("[good] Анимация цифр работает (TextScaled выключен).")
