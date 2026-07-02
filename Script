

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local RootPart = Character:WaitForChild("HumanoidRootPart")

local cash = LocalPlayer:WaitForChild("leaderstats"):WaitForChild("Cash")
local ProducerAction = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("ProducerAction")

local MACHINES = {
    "FruitBasket",
    "CashRegister",
    "SelfCheckouts",
    "ProduceSection",
    "FreezerSection",
    "OrderPickoutCheckInHereCounter",
    "WarehouseOrderStation",
    "DockProducer",
    "BoxMakerMachine",
    "ColdStorage",
}

-- state
local autoBuyRunning = false
local autoUpgradeRunning = false
local autoCycleRunning = false
local excludedMachines = {}
local availableButtons = {}
local buttonEventConnections = {}
local lastBuyTime = 0
local BUY_COOLDOWN = 0.4
local CYCLE_COOLDOWN = 0.4

-- tycoon state
local buttonsFolder = nil
local currentTycoonNumber = nil

-- tween sync
local activeTween = nil
local isTeleporting = false

-- parse cash string
local function parseCash(str)
    local s = tostring(str):gsub("%$", ""):gsub(",", ""):gsub("%s+", " "):lower():match("^%s*(.-)%s*$")
    local multipliers = {
        k=1e3, thousand=1e3,
        m=1e6, million=1e6,
        b=1e9, billion=1e9,
        t=1e12, trillion=1e12,
        q=1e15, quadrillion=1e15,
    }
    local num, suffix = s:match("^([%d%.]+)%s*([a-z]*)$")
    if not num then return 0 end
    return math.floor((tonumber(num) or 0) * (multipliers[suffix] or 1))
end

-- button helpers
local function getButtonPart(button)
    return button:FindFirstChild("ButtonPart") or button:FindFirstChild("Part")
end

local function isButtonAvailable(button)
    local part = getButtonPart(button)
    return part and part.CanCollide == true
end

local function getButtonPrice(button)
    local part = getButtonPart(button)
    if part then
        local label = part:FindFirstChild("BillboardGui") and part.BillboardGui:FindFirstChild("Frame") and part.BillboardGui.Frame:FindFirstChild("PriceLabel")
        if label and label:IsA("TextLabel") then
            local price = parseCash(label.Text)
            if price > 0 then return price end
        end
    end
    for _, desc in ipairs(button:GetDescendants()) do
        if desc:IsA("TextLabel") and desc.Name == "PriceLabel" then
            local price = parseCash(desc.Text)
            if price > 0 then return price end
        end
    end
    return nil
end

-- smooth teleport with overlap prevention
local function teleportToButtonSmooth(button)
    if isTeleporting then
        return -- already moving, skip this call
    end
    local part = getButtonPart(button)
    if not part then return end

    -- cancel any ongoing tween
    if activeTween then
        activeTween:Cancel()
        activeTween = nil
    end

    isTeleporting = true
    local targetPos = part.Position + Vector3.new(0, 5, 0)
    local tweenInfo = TweenInfo.new(0.125, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)
    local tween = TweenService:Create(RootPart, tweenInfo, {CFrame = CFrame.new(targetPos)})
    activeTween = tween
    tween:Play()
    tween.Completed:Wait()
    activeTween = nil
    isTeleporting = false
end

-- ----------------------------------------------------------------------
-- TYCOON DETECTION
-- ----------------------------------------------------------------------
local function getOwnedTycoon()
    local tycoons = workspace:FindFirstChild("Tycoons")
    if not tycoons then return nil end

    for _, tycoon in ipairs(tycoons:GetChildren()) do
        local owner = tycoon:GetAttribute("OwnerName")
        if not owner then
            local ownerValue = tycoon:FindFirstChild("OwnerName")
            if ownerValue and ownerValue:IsA("StringValue") then
                owner = ownerValue.Value
            end
        end
        if owner and owner == LocalPlayer.Name then
            return tycoon
        end
    end
    return nil
end

-- ----------------------------------------------------------------------
-- BUTTON CACHE MANAGEMENT
-- ----------------------------------------------------------------------
local function rebuildButtonCache()
    for _, conn in pairs(buttonEventConnections) do
        conn:Disconnect()
    end
    table.clear(buttonEventConnections)

    if not buttonsFolder then
        availableButtons = {}
        return
    end

    availableButtons = {}
    for _, button in ipairs(buttonsFolder:GetChildren()) do
        if isButtonAvailable(button) then
            table.insert(availableButtons, button)
        end
        local part = getButtonPart(button)
        if part then
            local conn = part:GetPropertyChangedSignal("CanCollide"):Connect(function()
                task.wait(0.1)
                rebuildButtonCache()
            end)
            table.insert(buttonEventConnections, conn)
        end
    end

    local childConn = buttonsFolder.ChildAdded:Connect(function(child)
        task.wait(0.1)
        rebuildButtonCache()
    end)
    table.insert(buttonEventConnections, childConn)
end

-- ----------------------------------------------------------------------
-- SET TYCOON
-- ----------------------------------------------------------------------
local function setTycoon(tycoon)
    if not tycoon then
        buttonsFolder = nil
        currentTycoonNumber = nil
        rebuildButtonCache()
        if tycoonStatusLabel then
            tycoonStatusLabel.Text = "No owned tycoon"
        end
        return
    end

    local num = tonumber(tycoon.Name:match("(%d+)$"))
    currentTycoonNumber = num or 0
    buttonsFolder = tycoon:FindFirstChild("Buttons")
    if not buttonsFolder then
        buttonsFolder = nil
        if tycoonStatusLabel then
            tycoonStatusLabel.Text = "Tycoon"..tostring(num).." (no Buttons folder)"
        end
        return
    end
    rebuildButtonCache()
    if tycoonStatusLabel then
        tycoonStatusLabel.Text = "Tycoon"..tostring(num).."  |  "..#availableButtons.." buttons"
    end
end

-- auto-find on load
local initialTycoon = getOwnedTycoon()
if initialTycoon then
    setTycoon(initialTycoon)
else
    setTycoon(nil)
end

-- ----------------------------------------------------------------------
-- HEARTBEAT LOOPS
-- ----------------------------------------------------------------------
RunService.Heartbeat:Connect(function()
    if not autoBuyRunning or not buttonsFolder then return end
    local now = tick()
    if now - lastBuyTime < BUY_COOLDOWN then return end
    lastBuyTime = now

    -- skip if we're currently in a teleport
    if isTeleporting then return end

    local buttons = availableButtons
    for _, button in ipairs(buttons) do
        if not autoBuyRunning then break end
        local price = getButtonPrice(button)
        if not price or price == 0 then continue end
        local currentCash = parseCash(cash.Value)
        if currentCash < price then continue end

        teleportToButtonSmooth(button)
        -- after teleport, if we got interrupted, skip
        if isTeleporting then
            -- actually teleportToButtonSmooth sets isTeleporting false after completion
            -- but we need to check if the button is still valid
            if not button.Parent then continue end
        end
        local part = getButtonPart(button)
        if part then
            local cd = part:FindFirstChildOfClass("ClickDetector")
            if cd then
                fireclickdetector(cd)
            end
        end
        task.wait(0.2)
    end
end)

-- auto upgrade loop
task.spawn(function()
    while true do
        task.wait(0.5)
        if not autoUpgradeRunning then
            task.wait(0.5)
            continue
        end
        for _, machineName in ipairs(MACHINES) do
            if not autoUpgradeRunning then break end
            if excludedMachines[machineName] then continue end
            pcall(function()
                ProducerAction:FireServer("Upgrade", machineName)
            end)
            task.wait(0.1)
        end
    end
end)

-- auto cycle loop
local lastCycleTime = 0
RunService.Heartbeat:Connect(function()
    if not autoCycleRunning or not buttonsFolder then return end
    local now = tick()
    if now - lastCycleTime < CYCLE_COOLDOWN then return end
    lastCycleTime = now

    for _, machineName in ipairs(MACHINES) do
        if not autoCycleRunning then break end
        if excludedMachines[machineName] then continue end
        pcall(function()
            ProducerAction:FireServer("ManualCycle", machineName)
        end)
    end
end)

-- character respawn
LocalPlayer.CharacterAdded:Connect(function(char)
    Character = char
    RootPart = char:WaitForChild("HumanoidRootPart")
    -- reset tween state
    if activeTween then
        activeTween:Cancel()
        activeTween = nil
    end
    isTeleporting = false
end)

-- ----------------------------------------------------------------------
-- UI CONSTRUCTION
-- ----------------------------------------------------------------------
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AutoUI"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = LocalPlayer.PlayerGui

local function destroyAll()
    autoBuyRunning = false
    autoUpgradeRunning = false
    autoCycleRunning = false
    for _, conn in pairs(buttonEventConnections) do
        conn:Disconnect()
    end
    if activeTween then
        activeTween:Cancel()
        activeTween = nil
    end
    isTeleporting = false
    screenGui:Destroy()
    print("[AutoBot] Closed.")
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.KeyCode == Enum.KeyCode.End then
        destroyAll()
    end
end)

-- main frame
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 230, 0, 460)
mainFrame.Position = UDim2.new(0, 20, 0, 20)
mainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui
Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 8)

-- title bar
local titleBar = Instance.new("Frame")
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
titleBar.BorderSizePixel = 0
titleBar.Parent = mainFrame
Instance.new("UICorner", titleBar).CornerRadius = UDim.new(0, 8)

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, -34, 1, 0)
titleLabel.Position = UDim2.new(0, 10, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "AutoBot  |  End to close"
titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
titleLabel.TextSize = 12
titleLabel.Font = Enum.Font.GothamBold
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.Parent = titleBar

local minimizeBtn = Instance.new("TextButton")
minimizeBtn.Size = UDim2.new(0, 24, 0, 24)
minimizeBtn.Position = UDim2.new(1, -28, 0, 3)
minimizeBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
minimizeBtn.Text = "-"
minimizeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
minimizeBtn.TextSize = 16
minimizeBtn.Font = Enum.Font.GothamBold
minimizeBtn.BorderSizePixel = 0
minimizeBtn.Parent = titleBar
Instance.new("UICorner", minimizeBtn).CornerRadius = UDim.new(0, 4)

-- content
local contentFrame = Instance.new("Frame")
contentFrame.Size = UDim2.new(1, 0, 1, -30)
contentFrame.Position = UDim2.new(0, 0, 0, 30)
contentFrame.BackgroundTransparency = 1
contentFrame.Parent = mainFrame

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 6)
layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
layout.Parent = contentFrame

local padding = Instance.new("UIPadding")
padding.PaddingTop = UDim.new(0, 8)
padding.Parent = contentFrame

-- status label
local tycoonStatusLabel = Instance.new("TextLabel")
tycoonStatusLabel.Size = UDim2.new(1, -20, 0, 20)
tycoonStatusLabel.BackgroundTransparency = 1
tycoonStatusLabel.Text = "Scanning..."
tycoonStatusLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
tycoonStatusLabel.TextSize = 11
tycoonStatusLabel.Font = Enum.Font.Gotham
tycoonStatusLabel.Parent = contentFrame
tycoonStatusLabel.TextXAlignment = Enum.TextXAlignment.Center

-- auto buy
local autoBuyBtn = Instance.new("TextButton")
autoBuyBtn.Size = UDim2.new(1, -20, 0, 34)
autoBuyBtn.BackgroundColor3 = Color3.fromRGB(40, 167, 69)
autoBuyBtn.Text = "AUTO BUY: OFF"
autoBuyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
autoBuyBtn.TextSize = 12
autoBuyBtn.Font = Enum.Font.GothamBold
autoBuyBtn.BorderSizePixel = 0
autoBuyBtn.Parent = contentFrame
Instance.new("UICorner", autoBuyBtn).CornerRadius = UDim.new(0, 6)

autoBuyBtn.MouseButton1Click:Connect(function()
    autoBuyRunning = not autoBuyRunning
    if autoBuyRunning then
        autoBuyBtn.Text = "AUTO BUY: ON"
        autoBuyBtn.BackgroundColor3 = Color3.fromRGB(220, 53, 69)
        if buttonsFolder then rebuildButtonCache() end
    else
        autoBuyBtn.Text = "AUTO BUY: OFF"
        autoBuyBtn.BackgroundColor3 = Color3.fromRGB(40, 167, 69)
    end
end)

-- auto upgrade
local autoUpgradeBtn = Instance.new("TextButton")
autoUpgradeBtn.Size = UDim2.new(1, -20, 0, 34)
autoUpgradeBtn.BackgroundColor3 = Color3.fromRGB(40, 167, 69)
autoUpgradeBtn.Text = "AUTO UPGRADE: OFF"
autoUpgradeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
autoUpgradeBtn.TextSize = 12
autoUpgradeBtn.Font = Enum.Font.GothamBold
autoUpgradeBtn.BorderSizePixel = 0
autoUpgradeBtn.Parent = contentFrame
Instance.new("UICorner", autoUpgradeBtn).CornerRadius = UDim.new(0, 6)

autoUpgradeBtn.MouseButton1Click:Connect(function()
    autoUpgradeRunning = not autoUpgradeRunning
    if autoUpgradeRunning then
        autoUpgradeBtn.Text = "AUTO UPGRADE: ON"
        autoUpgradeBtn.BackgroundColor3 = Color3.fromRGB(220, 53, 69)
    else
        autoUpgradeBtn.Text = "AUTO UPGRADE: OFF"
        autoUpgradeBtn.BackgroundColor3 = Color3.fromRGB(40, 167, 69)
    end
end)

-- auto cycle
local autoCycleBtn = Instance.new("TextButton")
autoCycleBtn.Size = UDim2.new(1, -20, 0, 34)
autoCycleBtn.BackgroundColor3 = Color3.fromRGB(40, 167, 69)
autoCycleBtn.Text = "AUTO CYCLE: OFF"
autoCycleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
autoCycleBtn.TextSize = 12
autoCycleBtn.Font = Enum.Font.GothamBold
autoCycleBtn.BorderSizePixel = 0
autoCycleBtn.Parent = contentFrame
Instance.new("UICorner", autoCycleBtn).CornerRadius = UDim.new(0, 6)

autoCycleBtn.MouseButton1Click:Connect(function()
    autoCycleRunning = not autoCycleRunning
    if autoCycleRunning then
        autoCycleBtn.Text = "AUTO CYCLE: ON"
        autoCycleBtn.BackgroundColor3 = Color3.fromRGB(220, 53, 69)
    else
        autoCycleBtn.Text = "AUTO CYCLE: OFF"
        autoCycleBtn.BackgroundColor3 = Color3.fromRGB(40, 167, 69)
    end
end)

-- find tycoon
local findTycoonBtn = Instance.new("TextButton")
findTycoonBtn.Size = UDim2.new(1, -20, 0, 30)
findTycoonBtn.BackgroundColor3 = Color3.fromRGB(70, 70, 100)
findTycoonBtn.Text = "Find Tycoon"
findTycoonBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
findTycoonBtn.TextSize = 12
findTycoonBtn.Font = Enum.Font.GothamBold
findTycoonBtn.BorderSizePixel = 0
findTycoonBtn.Parent = contentFrame
Instance.new("UICorner", findTycoonBtn).CornerRadius = UDim.new(0, 6)

findTycoonBtn.MouseButton1Click:Connect(function()
    local tycoon = getOwnedTycoon()
    if tycoon then
        setTycoon(tycoon)
    else
        setTycoon(nil)
        tycoonStatusLabel.Text = "No owned tycoon found"
    end
    if buttonsFolder then
        rebuildButtonCache()
    end
end)

-- exclude label
local excludeLabel = Instance.new("TextLabel")
excludeLabel.Size = UDim2.new(1, -20, 0, 18)
excludeLabel.BackgroundTransparency = 1
excludeLabel.Text = "EXCLUDE MACHINES"
excludeLabel.TextColor3 = Color3.fromRGB(160, 160, 160)
excludeLabel.TextSize = 11
excludeLabel.Font = Enum.Font.GothamBold
excludeLabel.Parent = contentFrame

-- scroll frame
local scrollFrame = Instance.new("ScrollingFrame")
scrollFrame.Size = UDim2.new(1, -20, 0, 160)
scrollFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
scrollFrame.BorderSizePixel = 0
scrollFrame.ScrollBarThickness = 4
scrollFrame.ScrollBarImageColor3 = Color3.fromRGB(80, 80, 80)
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
scrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
scrollFrame.Parent = contentFrame
Instance.new("UICorner", scrollFrame).CornerRadius = UDim.new(0, 6)

local scrollLayout = Instance.new("UIListLayout")
scrollLayout.Padding = UDim.new(0, 4)
scrollLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
scrollLayout.Parent = scrollFrame

local scrollPad = Instance.new("UIPadding")
scrollPad.PaddingTop = UDim.new(0, 6)
scrollPad.PaddingBottom = UDim.new(0, 6)
scrollPad.Parent = scrollFrame

for _, machineName in ipairs(MACHINES) do
    local row = Instance.new("TextButton")
    row.Size = UDim2.new(1, -10, 0, 26)
    row.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    row.Text = machineName
    row.TextColor3 = Color3.fromRGB(255, 255, 255)
    row.TextSize = 11
    row.Font = Enum.Font.Gotham
    row.BorderSizePixel = 0
    row.Parent = scrollFrame
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 4)

    row.MouseButton1Click:Connect(function()
        if excludedMachines[machineName] then
            excludedMachines[machineName] = nil
            row.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
            row.TextColor3 = Color3.fromRGB(255, 255, 255)
        else
            excludedMachines[machineName] = true
            row.BackgroundColor3 = Color3.fromRGB(80, 30, 30)
            row.TextColor3 = Color3.fromRGB(220, 100, 100)
        end
    end)
end

-- minimize
local minimized = false
minimizeBtn.MouseButton1Click:Connect(function()
    minimized = not minimized
    if minimized then
        mainFrame.Size = UDim2.new(0, 230, 0, 30)
        contentFrame.Visible = false
        minimizeBtn.Text = "+"
    else
        mainFrame.Size = UDim2.new(0, 230, 0, 460)
        contentFrame.Visible = true
        minimizeBtn.Text = "-"
    end
end)

-- initial status
if currentTycoonNumber then
    tycoonStatusLabel.Text = "Tycoon"..tostring(currentTycoonNumber).."  |  "..#availableButtons.." buttons"
else
    tycoonStatusLabel.Text = "No owned tycoon"
end

print("[AutoBot] Loaded. Press End to close. Use 'Find Tycoon' to refresh.")
