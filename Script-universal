local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local BACKPACK_DISTANCE = 1
local CHASE_DISTANCE = 3.5
local TELEPORT_DISTANCE = 20
local UPDATE_RATE = 0.12
local JUMP_COOLDOWN = 0.8
local MOVE_THRESHOLD = 0.45

local ORBIT_SPEED = math.rad(90)
local ORBIT_RADIUS = 3.5
local ORBIT_HEIGHT = 0.5

local MODE_NAMES = {
    [1] = "SMART BACKPACK",
    [2] = "PLAYER CHASE",
    [3] = "PLAYER ORBIT",
    [4] = "ATTACH TO OBJECT"
}

local selectedMode = 1
local active = false
local closed = false
local target = nil

local loopConnection
local characterConnection
local seatedConnection
local focusConnection

local character
local humanoid
local rootPart

local elapsed = 0
local jumpCooldown = 0
local lastTargetPosition
local targetMoving = false
local orbitAngle = 0

local originalCollisions = {}

local attachedPart = nil
local attachWeld = nil
local originalAttachedAnchored = nil
local originalAttachedCanCollide = nil

local status

local function stopAttachMode()
    if attachWeld then
        pcall(function()
            attachWeld:Destroy()
        end)
        attachWeld = nil
    end

    if attachedPart and attachedPart.Parent then
        pcall(function()
            attachedPart.CanCollide = originalAttachedCanCollide
            attachedPart.Anchored = originalAttachedAnchored
        end)
    end

    attachedPart = nil
    originalAttachedAnchored = nil
    originalAttachedCanCollide = nil
end

local function startAttachMode()
    if not character or not rootPart then
        status.Text = "Персонаж не готов"
        return false
    end

    stopAttachMode()

    local rayParams = RaycastParams.new()
    rayParams.FilterType = Enum.RaycastFilterType.Exclude

    local excluded = {character}
    local part = nil

    for i = 1, 100 do
        rayParams.FilterDescendantsInstances = excluded

        local result = workspace:Raycast(
            rootPart.Position,
            Vector3.new(0, -10, 0),
            rayParams
        )

        if not result then
            break
        end

        local hit = result.Instance

        if hit:IsA("BasePart") and hit.Anchored == false then
            part = hit
            break
        end

        table.insert(excluded, hit)
    end

    if not part then
        status.Text = "Незаякоренный объект не найден"
        return false
    end

    attachedPart = part
    originalAttachedAnchored = part.Anchored
    originalAttachedCanCollide = part.CanCollide

    part.CanCollide = false

    attachWeld = Instance.new("WeldConstraint")
    attachWeld.Part0 = part
    attachWeld.Part1 = rootPart
    attachWeld.Parent = part

    status.Text = "Прикреплено к объекту"

    return true
end

local function restoreCollisions()
    for part, originalValue in pairs(originalCollisions) do
        if part and part.Parent then
            part.CanCollide = originalValue
        end
    end

    table.clear(originalCollisions)
end

local function setupCharacter(char)
    if active then
        active = false
    end

    stopAttachMode()
    restoreCollisions()

    character = char
    humanoid = char:WaitForChild("Humanoid")
    rootPart = char:WaitForChild("HumanoidRootPart")

    if seatedConnection then
        seatedConnection:Disconnect()
    end

    seatedConnection = humanoid.Seated:Connect(function(isSeated)
        if active and selectedMode == 3 and isSeated then
            humanoid.Sit = false
            humanoid:ChangeState(
                Enum.HumanoidStateType.GettingUp
            )
        end
    end)
end

if player.Character then
    task.spawn(setupCharacter, player.Character)
end

characterConnection = player.CharacterAdded:Connect(setupCharacter)

local function getRoot(p)
    local char = p and p.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

-- ИЗМЕНЕННАЯ ФУНКЦИЯ ПОИСКА (строгая логика по первым 3 буквам + проверка)
local function findPlayer(inputName)
    inputName = string.lower(inputName:match("^%s*(.-)%s*$"))

    if #inputName < 3 then
        return nil, "Введи минимум 3 буквы"
    end

    local prefix = string.sub(inputName, 1, 3)
    local matches = {}

    for _, other in ipairs(Players:GetPlayers()) do
        if other ~= player then
            local username = string.lower(other.Name)
            local displayName = string.lower(other.DisplayName)

            if string.sub(username, 1, 3) == prefix
                or string.sub(displayName, 1, 3) == prefix then

                table.insert(matches, other)
            end
        end
    end

    if #matches == 0 then
        return nil, "Игрок не найден"
    end

    if #matches > 1 then
        return nil, "Несколько совпадений: уточни ник"
    end

    return matches[1], nil
end

local oldGui = playerGui:FindFirstChild("PlayerControlGUI")

if oldGui then
    oldGui:Destroy()
end

local gui = Instance.new("ScreenGui")
gui.Name = "PlayerControlGUI"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.DisplayOrder = 100
gui.Parent = playerGui

local container = Instance.new("Frame")
container.Name = "Container"
container.Size = UDim2.fromOffset(250, 225)
container.Position = UDim2.new(0, 12, 1, -245)
container.BackgroundTransparency = 1
container.Parent = gui

local square = Instance.new("TextButton")
square.Name = "YellowSquare"
square.Size = UDim2.fromOffset(30, 30)
square.Position = UDim2.fromOffset(0, 95)
square.BackgroundColor3 = Color3.fromRGB(255, 210, 35)
square.BorderSizePixel = 0
square.Text = ""
square.AutoButtonColor = false
square.Parent = container

Instance.new("UICorner", square).CornerRadius = UDim.new(0, 5)

local panel = Instance.new("Frame")
panel.Name = "MainPanel"
panel.Size = UDim2.fromOffset(210, 225)
panel.Position = UDim2.fromOffset(38, 0)
panel.BackgroundColor3 = Color3.fromRGB(27, 27, 33)
panel.BorderSizePixel = 0
panel.Parent = container

Instance.new("UICorner", panel).CornerRadius = UDim.new(0, 8)

local function makeButton(name, text, size, position, color)
    local button = Instance.new("TextButton")
    button.Name = name
    button.Text = text
    button.Size = size
    button.Position = position
    button.BackgroundColor3 = color
    button.TextColor3 = Color3.new(1, 1, 1)
    button.TextSize = 12
    button.Font = Enum.Font.Gotham
    button.BorderSizePixel = 0
    button.Parent = panel

    Instance.new("UICorner", button).CornerRadius = UDim.new(0, 4)

    return button
end

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -30, 0, 23)
title.Position = UDim2.fromOffset(7, 3)
title.BackgroundTransparency = 1
title.Text = "PLAYER CONTROL"
title.TextColor3 = Color3.new(1, 1, 1)
title.TextSize = 13
title.Font = Enum.Font.GothamBold
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = panel

local closeButton = makeButton(
    "Close", "X",
    UDim2.fromOffset(20, 20),
    UDim2.new(1, -24, 0, 3),
    Color3.fromRGB(170, 45, 45)
)

local nameBox = Instance.new("TextBox")
nameBox.Name = "TargetName"
nameBox.Size = UDim2.new(1, -14, 0, 23)
nameBox.Position = UDim2.fromOffset(7, 30)
nameBox.BackgroundColor3 = Color3.fromRGB(45, 45, 55)
nameBox.TextColor3 = Color3.new(1, 1, 1)
nameBox.PlaceholderText = "Первые 3 буквы ника"
nameBox.PlaceholderColor3 = Color3.fromRGB(160, 160, 160)
nameBox.Text = ""
nameBox.TextSize = 11
nameBox.Font = Enum.Font.Gotham
nameBox.ClearTextOnFocus = false
nameBox.Parent = panel

Instance.new("UICorner", nameBox).CornerRadius = UDim.new(0, 4)

local findButton = makeButton(
    "FindPlayer", "OK",
    UDim2.new(1, -14, 0, 22),
    UDim2.fromOffset(7, 58),
    Color3.fromRGB(55, 95, 155)
)

local startButton = makeButton(
    "Start", "Start",
    UDim2.new(0.48, -7, 0, 24),
    UDim2.fromOffset(7, 86),
    Color3.fromRGB(40, 145, 75)
)

local endButton = makeButton(
    "End", "End",
    UDim2.new(0.48, -7, 0, 24),
    UDim2.new(0.52, 0, 0, 86),
    Color3.fromRGB(165, 55, 55)
)

local modeLabel = Instance.new("TextLabel")
modeLabel.Size = UDim2.new(1, -14, 0, 17)
modeLabel.Position = UDim2.fromOffset(7, 116)
modeLabel.BackgroundTransparency = 1
modeLabel.TextColor3 = Color3.fromRGB(230, 210, 120)
modeLabel.TextSize = 11
modeLabel.Font = Enum.Font.GothamBold
modeLabel.Parent = panel

local previousButton = makeButton(
    "Previous", "◀",
    UDim2.fromOffset(42, 25),
    UDim2.fromOffset(47, 143),
    Color3.fromRGB(65, 65, 80)
)

local nextButton = makeButton(
    "Next", "▶",
    UDim2.fromOffset(42, 25),
    UDim2.fromOffset(121, 143),
    Color3.fromRGB(65, 65, 80)
)

local numberLabel = Instance.new("TextLabel")
numberLabel.Size = UDim2.fromOffset(30, 25)
numberLabel.Position = UDim2.fromOffset(90, 143)
numberLabel.BackgroundTransparency = 1
numberLabel.TextColor3 = Color3.new(1, 1, 1)
numberLabel.TextSize = 17
numberLabel.Font = Enum.Font.GothamBold
numberLabel.Parent = panel

status = Instance.new("TextLabel")
status.Size = UDim2.new(1, -14, 0, 17)
status.Position = UDim2.fromOffset(7, 176)
status.BackgroundTransparency = 1
status.Text = "Ожидание"
status.TextColor3 = Color3.fromRGB(190, 190, 190)
status.TextSize = 10
status.Parent = panel

findButton.Activated:Connect(function()
    local found, errorMessage = findPlayer(nameBox.Text)

    if not found then
        target = nil
        status.Text = errorMessage or "Игрок не найден"
        return
    end

    target = found
    status.Text = "Цель: " .. found.Name
end)

nameBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then
        local found, errorMessage = findPlayer(nameBox.Text)

        if not found then
            target = nil
            status.Text = errorMessage or "Игрок не найден"
            return
        end

        target = found
        status.Text = "Цель: " .. found.Name
    end
end)

local dragging = false
local moved = false
local dragStart
local startPosition

square.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        moved = false
        dragStart = input.Position
        startPosition = container.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if not dragging then return end

    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then

        local delta = input.Position - dragStart

        if delta.Magnitude > 5 then
            moved = true
        end

        container.Position = UDim2.new(
            startPosition.X.Scale,
            startPosition.X.Offset + delta.X,
            startPosition.Y.Scale,
            startPosition.Y.Offset + delta.Y
        )
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = false
    end
end)

local minimized = false

square.Activated:Connect(function()
    if moved then
        moved = false
        return
    end

    minimized = not minimized
    panel.Visible = not minimized
end)

local function disableCollisions()
    if not character then return end

    for _, object in ipairs(character:GetDescendants()) do
        if object:IsA("BasePart") then
            if originalCollisions[object] == nil then
                originalCollisions[object] = object.CanCollide
            end

            object.CanCollide = false
        end
    end
end

local function freezeMovementAnimations()
    if not humanoid then return end

    local animator = humanoid:FindFirstChildOfClass("Animator")

    if not animator then return end

    for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
        local trackName = string.lower(track.Name or "")
        local animationName = ""

        if track.Animation then
            animationName = string.lower(track.Animation.Name or "")
        end

        local isMovement =
            string.find(trackName, "walk")
            or string.find(trackName, "run")
            or string.find(trackName, "jump")
            or string.find(animationName, "walk")
            or string.find(animationName, "run")
            or string.find(animationName, "jump")

        if isMovement then
            track:Stop()
        end
    end
end

local function stopMode(message)
    active = false

    if loopConnection then
        loopConnection:Disconnect()
        loopConnection = nil
    end

    stopAttachMode()
    restoreCollisions()

    if humanoid and humanoid.Parent then
        humanoid:Move(Vector3.zero)
    end

    status.Text = message or "Остановлено"
end

local function runBackpack(dt)
    local myRoot = rootPart
    local targetRoot = getRoot(target)

    if not myRoot or not targetRoot or not humanoid then
        stopMode("Персонаж недоступен")
        return
    end

    if humanoid.Health <= 0 then
        stopMode("Персонаж недоступен")
        return
    end

    disableCollisions()
    freezeMovementAnimations()

    local destination =
        targetRoot.Position
        - targetRoot.CFrame.LookVector * BACKPACK_DISTANCE

    myRoot.CFrame = CFrame.lookAt(
        destination,
        targetRoot.Position
    )

    myRoot.AssemblyLinearVelocity =
        targetRoot.AssemblyLinearVelocity

    myRoot.AssemblyAngularVelocity =
        targetRoot.AssemblyAngularVelocity
end

local function runChase(dt)
    elapsed += dt
    jumpCooldown = math.max(0, jumpCooldown - dt)

    if elapsed < UPDATE_RATE then return end

    elapsed = 0

    local myRoot = rootPart
    local targetRoot = getRoot(target)

    if not myRoot or not targetRoot or not humanoid then
        stopMode("Персонаж недоступен")
        return
    end

    if humanoid.Health <= 0 then
        stopMode("Персонаж недоступен")
        return
    end

    local actualDistance =
        (targetRoot.Position - myRoot.Position).Magnitude

    if actualDistance > TELEPORT_DISTANCE then
        local destination =
            targetRoot.Position
            - targetRoot.CFrame.LookVector * CHASE_DISTANCE

        myRoot.CFrame = CFrame.lookAt(
            destination,
            targetRoot.Position
        )

        lastTargetPosition = targetRoot.Position
        status.Text = "Цель далеко"
        return
    end

    local currentPosition = targetRoot.Position

    if lastTargetPosition then
        targetMoving =
            (currentPosition - lastTargetPosition).Magnitude
            > MOVE_THRESHOLD
    end

    lastTargetPosition = currentPosition

    if not targetMoving then
        humanoid:Move(Vector3.zero)
        status.Text = "Цель стоит"
        return
    end

    local destination =
        targetRoot.Position
        - targetRoot.CFrame.LookVector * CHASE_DISTANCE

    if (destination - myRoot.Position).Magnitude > 1.2 then
        humanoid:MoveTo(destination)
    else
        humanoid:Move(Vector3.zero)
    end

    local heightDifference =
        targetRoot.Position.Y - myRoot.Position.Y

    if humanoid.FloorMaterial ~= Enum.Material.Air
        and jumpCooldown <= 0
        and heightDifference > 2.5 then

        humanoid.Jump = true
        jumpCooldown = JUMP_COOLDOWN
    end

    status.Text = "Следую за " .. target.Name
end

local function runOrbit(dt)
    local myRoot = rootPart
    local targetRoot = getRoot(target)

    if not myRoot or not targetRoot or not humanoid then
        stopMode("Персонаж недоступен")
        return
    end

    if humanoid.Health <= 0 then
        stopMode("Персонаж недоступен")
        return
    end

    if humanoid.Sit
        or humanoid:GetState() == Enum.HumanoidStateType.Seated
        or humanoid.SeatPart ~= nil then

        humanoid.Sit = false
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end

    orbitAngle =
        (orbitAngle + ORBIT_SPEED * dt) % (math.pi * 2)

    local targetPosition = targetRoot.Position

    local offset = Vector3.new(
        math.cos(orbitAngle) * ORBIT_RADIUS,
        ORBIT_HEIGHT,
        math.sin(orbitAngle) * ORBIT_RADIUS
    )

    local destination = targetPosition + offset

    myRoot.CFrame = CFrame.lookAt(
        destination,
        targetPosition
    )

    status.Text = "Орбита: " .. target.Name
end

local function startMode()
    if active then
        status.Text = "Уже работает"
        return
    end

    if not character or not humanoid or not rootPart then
        status.Text = "Персонаж не готов"
        return
    end

    if humanoid.Health <= 0 then
        stopMode("Персонаж недоступен")
        return
    end

    if selectedMode == 4 then
        if startAttachMode() then
            active = true
        end
        return
    end

    if not target then
        status.Text = "Цель не выбрана"
        return
    end

    local targetRoot = getRoot(target)

    if not targetRoot then
        status.Text = "Цель не готова"
        return
    end

    if loopConnection then
        loopConnection:Disconnect()
        loopConnection = nil
    end

    active = true
    elapsed = 0
    jumpCooldown = 0
    orbitAngle = 0
    lastTargetPosition = nil
    targetMoving = false

    status.Text = "Запущен: " .. MODE_NAMES[selectedMode]

    loopConnection = RunService.Heartbeat:Connect(function(dt)
        if not active then return end

        if not target or not target.Parent then
            stopMode("Цель покинула игру")
            target = nil
            return
        end

        if selectedMode == 1 then
            runBackpack(dt)
        elseif selectedMode == 2 then
            runChase(dt)
        elseif selectedMode == 3 then
            runOrbit(dt)
        end
    end)
end

local function updateModeDisplay()
    numberLabel.Text = tostring(selectedMode)
    modeLabel.Text = MODE_NAMES[selectedMode]
end

local function switchMode(direction)
    local wasActive = active

    if wasActive then
        stopMode("Переключение режима")
    end

    selectedMode += direction

    if selectedMode > 4 then
        selectedMode = 1
    elseif selectedMode < 1 then
        selectedMode = 4
    end

    updateModeDisplay()

    if wasActive then
        startMode()
    else
        status.Text = "Выбран режим " .. selectedMode
    end
end

previousButton.Activated:Connect(function()
    switchMode(-1)
end)

nextButton.Activated:Connect(function()
    switchMode(1)
end)

startButton.Activated:Connect(function()
    if active then
        status.Text = "Уже работает"
        return
    end

    startMode()
end)

endButton.Activated:Connect(function()
    stopMode("Остановлено пользователем")
end)

focusConnection =
    UserInputService.WindowFocusReleased:Connect(function()
        if active then
            stopMode("Автостоп: Roblox потерял фокус")
        end
    end)

closeButton.Activated:Connect(function()
    if closed then return end

    closed = true

    stopMode("Система закрыта")

    restoreCollisions()
    stopAttachMode()

    if seatedConnection then
        seatedConnection:Disconnect()
    end

    if characterConnection then
        characterConnection:Disconnect()
    end

    if focusConnection then
        focusConnection:Disconnect()
    end

    gui:Destroy()
end)

updateModeDisplay()

print("PLAYER CONTROL V1.6 successfully loaded")
