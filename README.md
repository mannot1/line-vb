-- TorsoLine.local.lua
-- Place this LocalScript in StarterPlayerScripts (client-side)
-- Creates a movable GUI to configure a Beam that comes out of other-team players' torso area.
-- The Beam is attached to the HumanoidRootPart (fallbacks included) so it's not thrown off by upper-body animations.
-- The beam now extends forward from the torso (negative local Z), so it points from the player's torso in their facing direction.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- Config with sensible defaults
local config = {
    color = Color3.fromRGB(255, 80, 60), -- default red-ish
    emission = 1, -- LightEmission 0..1 (neon brightness)
    length = 6,  -- studs out from torso (default not too short)
    thickness = 0.25, -- beam width (not too thin)
    torsoYOffset = 0.5, -- vertical offset from HumanoidRootPart to approximate torso area (lower than head)
    enabled = true
}

-- Folder for beams/attachments
local beamFolder = Instance.new("Folder")
beamFolder.Name = "TorsoLines_" .. tostring(LocalPlayer.UserId)
beamFolder.Parent = workspace

-- Utility: find a robust root part for attaching
local function getRootPart(character)
    if not character then return nil end
    local hrp = character:FindFirstChild("HumanoidRootPart")
    if hrp and hrp:IsA("BasePart") then return hrp end
    local upperTorso = character:FindFirstChild("UpperTorso")
    if upperTorso and upperTorso:IsA("BasePart") then return upperTorso end
    local torso = character:FindFirstChild("Torso")
    if torso and torso:IsA("BasePart") then return torso end
    return nil
end

-- Table of tracked beams: player -> {attachment0, attachment1, beam}
local tracked = {}

local function destroyTrackedForPlayer(player)
    local t = tracked[player]
    if not t then return end
    if t.beam and t.beam.Parent then t.beam:Destroy() end
    if t.attachment0 and t.attachment0.Parent then t.attachment0:Destroy() end
    if t.attachment1 and t.attachment1.Parent then t.attachment1:Destroy() end
    tracked[player] = nil
end

local function createBeamForCharacter(player, character)
    if not config.enabled then return end
    if not player or player == LocalPlayer then return end
    if player.Team and LocalPlayer.Team and player.Team == LocalPlayer.Team then
        return
    end

    local root = getRootPart(character)
    if not root then return end

    -- clean up if exists
    destroyTrackedForPlayer(player)

    -- Attachments parented to the root part (uses local coordinates so beam points along the local forward)
    local att0 = Instance.new("Attachment")
    att0.Name = "TorsoLine_Att0_" .. player.Name
    att0.Position = Vector3.new(0, config.torsoYOffset, 0)
    att0.Parent = root

    -- Use negative Z so the beam extends forward from the torso (front of character)
    local att1 = Instance.new("Attachment")
    att1.Name = "TorsoLine_Att1_" .. player.Name
    att1.Position = Vector3.new(0, config.torsoYOffset, -config.length)
    att1.Parent = root

    local beam = Instance.new("Beam")
    beam.Name = "TorsoLine_Beam_" .. player.Name
    beam.Attachment0 = att0
    beam.Attachment1 = att1
    beam.Color = ColorSequence.new(config.color)
    beam.Width0 = math.clamp(config.thickness, 0.05, 10)
    beam.Width1 = math.clamp(config.thickness, 0.05, 10)
    beam.LightEmission = math.clamp(config.emission, 0, 1)
    beam.Transparency = NumberSequence.new(0)
    beam.FaceCamera = false
    beam.Segments = 1
    beam.Parent = beamFolder

    tracked[player] = {attachment0 = att0, attachment1 = att1, beam = beam, root = root}
end

local function shouldShowForPlayer(player)
    if not player or player == LocalPlayer then return false end
    if player.Team and LocalPlayer.Team and player.Team == LocalPlayer.Team then
        return false
    end
    return true
end

-- (Re)create beams as needed for a player
local function ensureForPlayer(player)
    local character = player.Character
    if not character then return end
    if not shouldShowForPlayer(player) then
        destroyTrackedForPlayer(player)
        return
    end
    createBeamForCharacter(player, character)
end

-- Hook for when a character is added or respawns
local function onCharacterAdded(player, character)
    -- slight delay to let HumanoidRootPart spawn
    character:WaitForChild("HumanoidRootPart", 2)
    -- create beam if eligible
    if shouldShowForPlayer(player) then
        createBeamForCharacter(player, character)
    end
end

-- Handle players joining / leaving
Players.PlayerAdded:Connect(function(player)
    -- initial if they already have a character (rare)
    if player.Character then
        onCharacterAdded(player, player.Character)
    end
    player.CharacterAdded:Connect(function(char) onCharacterAdded(player, char) end)
    -- watch team changes to add/remove beam
    player:GetPropertyChangedSignal("Team"):Connect(function()
        if shouldShowForPlayer(player) then
            if player.Character then
                createBeamForCharacter(player, player.Character)
            end
        else
            destroyTrackedForPlayer(player)
        end
    end)
end)

-- Existing players
for _, p in pairs(Players:GetPlayers()) do
    if p ~= LocalPlayer then
        if p.Character then
            onCharacterAdded(p, p.Character)
        end
        p.CharacterAdded:Connect(function(char) onCharacterAdded(p, char) end)
        p:GetPropertyChangedSignal("Team"):Connect(function()
            if shouldShowForPlayer(p) then
                if p.Character then
                    createBeamForCharacter(p, p.Character)
                end
            else
                destroyTrackedForPlayer(p)
            end
        end)
    end
end

-- Update beam visuals when config changes
local function applyConfigToAll()
    for player, t in pairs(tracked) do
        if t and t.beam and t.attachment1 and t.attachment0 then
            t.beam.Color = ColorSequence.new(config.color)
            t.beam.Width0 = math.clamp(config.thickness, 0.05, 10)
            t.beam.Width1 = math.clamp(config.thickness, 0.05, 10)
            t.beam.LightEmission = math.clamp(config.emission, 0, 1)
            t.attachment0.Position = Vector3.new(0, config.torsoYOffset, 0)
            t.attachment1.Position = Vector3.new(0, config.torsoYOffset, -config.length)
        end
    end
end

-- GUI Creation (simple, movable, numeric input)
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "TorsoLineGUI"
screenGui.Parent = PlayerGui
screenGui.ResetOnSpawn = false

local mainFrame = Instance.new("Frame")
mainFrame.Name = "Main"
mainFrame.Size = UDim2.new(0, 260, 0, 260)
mainFrame.Position = UDim2.new(0.02, 0, 0.7, 0)
mainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
mainFrame.BorderSizePixel = 0
mainFrame.Parent = screenGui
mainFrame.Active = true

-- Title bar / drag handle
local titleBar = Instance.new("Frame")
titleBar.Name = "TitleBar"
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
titleBar.BorderSizePixel = 0
titleBar.Parent = mainFrame

local titleLabel = Instance.new("TextLabel")
titleLabel.Text = "Torso Lines"
titleLabel.Size = UDim2.new(1, -8, 1, 0)
titleLabel.Position = UDim2.new(0, 8, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
titleLabel.Font = Enum.Font.SourceSansSemibold
titleLabel.TextSize = 20
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.Parent = titleBar

-- Draggable
local dragging = false
local dragInput, dragStart, startPos

titleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = mainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

titleBar.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

game:GetService("UserInputService").InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- Helper to create label + textbox row
local function createRow(y, labelText, defaultText)
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0, 120, 0, 28)
    label.Position = UDim2.new(0, 8, 0, y)
    label.BackgroundTransparency = 1
    label.Text = labelText
    label.TextColor3 = Color3.fromRGB(200, 200, 200)
    label.Font = Enum.Font.SourceSans
    label.TextSize = 14
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = mainFrame

    local box = Instance.new("TextBox")
    box.Size = UDim2.new(0, 110, 0, 26)
    box.Position = UDim2.new(0, 138, 0, y)
    box.BackgroundColor3 = Color3.fromRGB(44, 44, 44)
    box.TextColor3 = Color3.fromRGB(230, 230, 230)
    box.Text = defaultText or ""
    box.ClearTextOnFocus = false
    box.Font = Enum.Font.SourceSans
    box.TextSize = 14
    box.Parent = mainFrame
    return label, box
end

-- Rows: R, G, B, Emission, Length, Thickness, Torso Y Offset
local _, rBox = createRow(36, "R (0-255)", tostring(math.clamp(math.floor(config.color.R * 255), 0, 255)))
local _, gBox = createRow(70, "G (0-255)", tostring(math.clamp(math.floor(config.color.G * 255), 0, 255)))
local _, bBox = createRow(104, "B (0-255)", tostring(math.clamp(math.floor(config.color.B * 255), 0, 255)))
local _, emissionBox = createRow(138, "Emission (0-1)", tostring(config.emission))
local _, lengthBox = createRow(172, "Length (studs)", tostring(config.length))
local _, thicknessBox = createRow(206, "Thickness", tostring(config.thickness))
local _, yOffsetBox = createRow(240, "Torso Y Offset", tostring(config.torsoYOffset))

-- Expand mainFrame to fit the extra row and buttons
mainFrame.Size = UDim2.new(0, 260, 0, 320)

-- Toggle button and presets
local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(0, 120, 0, 28)
toggleBtn.Position = UDim2.new(0, 8, 1, -48)
toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
toggleBtn.TextColor3 = Color3.fromRGB(230, 230, 230)
toggleBtn.Font = Enum.Font.SourceSansSemibold
toggleBtn.TextSize = 16
toggleBtn.Text = config.enabled and "Disable" or "Enable"
toggleBtn.Parent = mainFrame

local resetBtn = Instance.new("TextButton")
resetBtn.Size = UDim2.new(0, 120, 0, 28)
resetBtn.Position = UDim2.new(0, 132, 1, -48)
resetBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
resetBtn.TextColor3 = Color3.fromRGB(230, 230, 230)
resetBtn.Font = Enum.Font.SourceSansSemibold
resetBtn.TextSize = 16
resetBtn.Text = "Reset Default"
resetBtn.Parent = mainFrame

-- Input validation and application
local function parseNumber(text, minv, maxv, fallback)
    local n = tonumber(text)
    if not n then return fallback end
    if minv then n = math.max(minv, n) end
    if maxv then n = math.min(maxv, n) end
    return n
end

local function updateColorFromBoxes()
    local r = parseNumber(rBox.Text, 0, 255, math.floor(config.color.R * 255))
    local g = parseNumber(gBox.Text, 0, 255, math.floor(config.color.G * 255))
    local b = parseNumber(bBox.Text, 0, 255, math.floor(config.color.B * 255))
    config.color = Color3.fromRGB(r, g, b)
    applyConfigToAll()
end

local function updateNumbers()
    config.emission = parseNumber(emissionBox.Text, 0, 1, config.emission)
    config.length = parseNumber(lengthBox.Text, 0.5, 200, config.length)
    config.thickness = parseNumber(thicknessBox.Text, 0.05, 10, config.thickness)
    config.torsoYOffset = parseNumber(yOffsetBox.Text, -5, 5, config.torsoYOffset)
    applyConfigToAll()
end

-- Connect FocusLost to apply values
rBox.FocusLost:Connect(function(enterPressed) updateColorFromBoxes() end)
gBox.FocusLost:Connect(function() updateColorFromBoxes() end)
bBox.FocusLost:Connect(function() updateColorFromBoxes() end)
emissionBox.FocusLost:Connect(function() updateNumbers() end)
lengthBox.FocusLost:Connect(function() updateNumbers() end)
thicknessBox.FocusLost:Connect(function() updateNumbers() end)
yOffsetBox.FocusLost:Connect(function() updateNumbers() end)

-- Toggle button
toggleBtn.MouseButton1Click:Connect(function()
    config.enabled = not config.enabled
    toggleBtn.Text = config.enabled and "Disable" or "Enable"
    if not config.enabled then
        -- remove all beams
        for p, _ in pairs(tracked) do
            destroyTrackedForPlayer(p)
        end
    else
        -- re-create for current eligible players
        for _, p in pairs(Players:GetPlayers()) do
            if shouldShowForPlayer(p) and p.Character then
                createBeamForCharacter(p, p.Character)
            end
        end
    end
end)

-- Reset button
resetBtn.MouseButton1Click:Connect(function()
    config.color = Color3.fromRGB(255, 80, 60)
    config.emission = 1
    config.length = 6
    config.thickness = 0.25
    config.torsoYOffset = 0.5
    rBox.Text = tostring(math.floor(config.color.R * 255))
    gBox.Text = tostring(math.floor(config.color.G * 255))
    bBox.Text = tostring(math.floor(config.color.B * 255))
    emissionBox.Text = tostring(config.emission)
    lengthBox.Text = tostring(config.length)
    thicknessBox.Text = tostring(config.thickness)
    yOffsetBox.Text = tostring(config.torsoYOffset)
    applyConfigToAll()
end)

-- Keep tracked players up-to-date (clean up on player removal)
Players.PlayerRemoving:Connect(function(player)
    destroyTrackedForPlayer(player)
end)

-- If a tracked player's root part changes (rare), re-create beam robustly on Heartbeat if needed
RunService.Heartbeat:Connect(function()
    -- Update attachment positions to reflect live config (length, offset)
    applyConfigToAll()
end)

-- Initial creation for existing players
for _, p in pairs(Players:GetPlayers()) do
    if p ~= LocalPlayer and p.Character then
        if shouldShowForPlayer(p) then
            createBeamForCharacter(p, p.Character)
        end
    end
end

-- End of script
