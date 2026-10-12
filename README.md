# HallowiteESP.lua.-

-- HALLOWITE ESP FINAL
-- METER + DRAG GUI + ON/OFF
-- LocalScript untuk pengalaman Roblox milik sendiri

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- SETTINGS
local TARGET_NAME = "Hallowite"
local ESP_COLOR = Color3.fromRGB(180, 0, 255)

local FILL_TRANSPARENCY = 0.8
local OUTLINE_TRANSPARENCY = 0

local SCAN_INTERVAL = 1.5
local DISTANCE_UPDATE_INTERVAL = 0.2
local DRAG_THRESHOLD = 6
local STUD_TO_METER = 0.28

local enabled = true
local running = true
local activeESP = {}
local connections = {}

-- HAPUS GUI LAMA
local oldGUI = playerGui:FindFirstChild("HallowiteESP_GUI")
if oldGUI then
    oldGUI:Destroy()
end

-- SCREEN GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "HallowiteESP_GUI"
screenGui.ResetOnSpawn = false
screenGui.DisplayOrder = 10
screenGui.Parent = playerGui

-- TEMPAT LABEL ESP
local espFolder = Instance.new("Folder")
espFolder.Name = "HallowiteESP_Objects"
espFolder.Parent = playerGui

-- TOMBOL GUI
local button = Instance.new("TextButton")
button.Name = "ToggleESP"
button.Size = UDim2.fromOffset(155, 46)
button.Position = UDim2.new(1, -170, 0.35, 0)
button.BackgroundColor3 = Color3.fromRGB(65, 0, 100)
button.TextColor3 = Color3.new(1, 1, 1)
button.TextSize = 16
button.Font = Enum.Font.GothamBold
button.Text = "ESP: ON"
button.Active = true
button.Selectable = true
button.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = button

local stroke = Instance.new("UIStroke")
stroke.Color = ESP_COLOR
stroke.Thickness = 2
stroke.Parent = button

-- DRAG GUI HP + PC
local dragging = false
local dragInput
local dragStart
local startPosition
local moved = false

button.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        moved = false
        dragStart = input.Position
        startPosition = button.Position

        local thisInput = input

        input.Changed:Connect(function()
            if thisInput.UserInputState == Enum.UserInputState.End then
                dragging = false
                dragInput = nil
            end
        end)
    end
end)

button.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

local dragConnection = UserInputService.InputChanged:Connect(
    function(input)
        if not running or not dragging then
            return
        end

        if input ~= dragInput
            and input.UserInputType ~= Enum.UserInputType.MouseMovement then
            return
        end

        local delta = input.Position - dragStart

        if math.abs(delta.X) > DRAG_THRESHOLD
            or math.abs(delta.Y) > DRAG_THRESHOLD then
            moved = true
        end

        if moved then
            button.Position = UDim2.new(
                startPosition.X.Scale,
                startPosition.X.Offset + delta.X,
                startPosition.Y.Scale,
                startPosition.Y.Offset + delta.Y
            )
        end
    end
)

-- DETEKSI TARGET
local function isTarget(obj)
    return obj:IsA("Model")
        and string.find(
            string.lower(obj.Name),
            string.lower(TARGET_NAME),
            1,
            true
        ) ~= nil
end

-- CARI BAGIAN MODEL
local function getAdornee(obj)
    return obj.PrimaryPart
        or obj:FindFirstChildWhichIsA("BasePart", true)
end

-- HAPUS ESP
local function removeESP(obj)
    local data = activeESP[obj]
    if not data then
        return
    end

    if data.highlight then
        data.highlight:Destroy()
    end

    if data.billboard then
        data.billboard:Destroy()
    end

    activeESP[obj] = nil

    if connections[obj] then
        connections[obj]:Disconnect()
        connections[obj] = nil
    end
end

-- BUAT ESP
local function createESP(obj)
    if not running or not enabled then
        return
    end

    if not obj:IsDescendantOf(Workspace) then
        return
    end

    if not isTarget(obj) or activeESP[obj] then
        return
    end

    local adornee = getAdornee(obj)
    if not adornee then
        return
    end

    local highlight = Instance.new("Highlight")
    highlight.Name = "HallowiteESP_Highlight"
    highlight.Adornee = obj
    highlight.FillColor = ESP_COLOR
    highlight.FillTransparency = FILL_TRANSPARENCY
    highlight.OutlineColor = ESP_COLOR
    highlight.OutlineTransparency = OUTLINE_TRANSPARENCY
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Parent = obj

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "HallowiteESP_Label"
    billboard.Adornee = adornee
    billboard.Size = UDim2.fromOffset(200, 35)
    billboard.StudsOffsetWorldSpace = Vector3.new(0, 4, 0)
    billboard.AlwaysOnTop = true
    billboard.MaxDistance = 2000
    billboard.Parent = espFolder

    local label = Instance.new("TextLabel")
    label.Name = "DistanceLabel"
    label.Size = UDim2.fromScale(1, 1)
    label.BackgroundTransparency = 1
    label.Text = "Hallowite | -- meter"
    label.TextColor3 = ESP_COLOR
    label.TextSize = 16
    label.Font = Enum.Font.SourceSansBold
    label.TextStrokeTransparency = 0.2
    label.Parent = billboard

    activeESP[obj] = {
        highlight = highlight,
        billboard = billboard,
        label = label
    }

    connections[obj] = obj.AncestryChanged:Connect(function()
        if not obj:IsDescendantOf(Workspace) then
            removeESP(obj)
        end
    end)
end

-- SCAN WORKSPACE
local function scanMap()
    if not running or not enabled then
        return
    end

    for _, obj in ipairs(Workspace:GetDescendants()) do
        if isTarget(obj) then
            createESP(obj)
        end
    end
end

-- UPDATE TOMBOL
local function updateButton()
    button.Text = enabled and "ESP: ON" or "ESP: OFF"

    button.BackgroundColor3 = enabled
        and Color3.fromRGB(65, 0, 100)
        or Color3.fromRGB(55, 55, 55)

    for _, data in pairs(activeESP) do
        data.highlight.Enabled = enabled
        data.billboard.Enabled = enabled
    end
end

-- TOGGLE ON/OFF
button.Activated:Connect(function()
    if moved then
        moved = false
        return
    end

    enabled = not enabled
    updateButton()

    if enabled then
        scanMap()
    end
end)

-- DETEKSI MODEL BARU
local addedConnection = Workspace.DescendantAdded:Connect(function(obj)
    task.defer(function()
        if not running or not enabled then
            return
        end

        if isTarget(obj) then
            createESP(obj)
        end

        local ancestor = obj.Parent

        while ancestor and ancestor ~= Workspace do
            if isTarget(ancestor) then
                createESP(ancestor)
            end

            ancestor = ancestor.Parent
        end
    end)
end)

-- UPDATE JARAK DALAM METER
task.spawn(function()
    while running and screenGui.Parent do
        task.wait(DISTANCE_UPDATE_INTERVAL)

        local character = player.Character
        local root = character
            and character:FindFirstChild("HumanoidRootPart")

        for obj, data in pairs(activeESP) do
            if not obj:IsDescendantOf(Workspace) then
                removeESP(obj)

            elseif enabled and root
                and data.billboard.Parent then

                local part = getAdornee(obj)

                if part and data.label.Parent then
                    local distanceStuds =
                        (root.Position - part.Position).Magnitude

                    local distanceMeters =
                        distanceStuds * STUD_TO_METER

                    data.label.Text = string.format(
                        "Hallowite | %.1f meter",
                        distanceMeters
                    )

                    if data.billboard.Adornee ~= part then
                        data.billboard.Adornee = part
                    end
                end
            end
        end
    end
end)

-- PEMINDAIAN AWAL
scanMap()

-- PEMINDAIAN BERKALA
task.spawn(function()
    while running and screenGui.Parent do
        task.wait(SCAN_INTERVAL)
        scanMap()
    end
end)

-- PEMBERSIHAN
screenGui.Destroying:Connect(function()
    running = false

    dragConnection:Disconnect()
    addedConnection:Disconnect()

    for obj in pairs(activeESP) do
        removeESP(obj)
    end

    if espFolder.Parent then
        espFolder:Destroy()
    end
end)

print("HALLOWITE ESP FINAL + METER siap!")
