-- ============================================
-- DRAXZS PVP MENU V2.3 - FLUENT INTERFACE
-- COM SPINBOT + SPEED HACK + FPS BOOST + ESP DISTANCE
-- ============================================

-- ============================================
-- CARREGAR BIBLIOTECAS
-- ============================================

local Library = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
local SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua"))()
local InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua"))()

-- ============================================
-- CRIAR JANELA PRINCIPAL
-- ============================================

local Window = Library:CreateWindow({
    Title = "⚡ DRAXZS PVP MENU ⚡",
    SubTitle = "by DRAXZS",
    TabWidth = 160,
    Size = UDim2.fromOffset(620, 480),
    Acrylic = false,
    Theme = "Darker",
    AccentColor = Color3.fromRGB(255, 0, 0),
    MinimizeKey = Enum.KeyCode.LeftControl
})

-- ============================================
-- CRIAR ABAS
-- ============================================

local Tabs = {
    Main = Window:AddTab({ Title = "🏠 Welcome", Icon = "home" }),
    Aim = Window:AddTab({ Title = "🎯 Aim Assist", Icon = "crosshair" }),
    ESPTab = Window:AddTab({ Title = "👁️ ESP", Icon = "eye" }),
    Movement = Window:AddTab({ Title = "🌀 Movement", Icon = "move" }),
    Performance = Window:AddTab({ Title = "⚡ Performance", Icon = "zap" }),
    Settings = Window:AddTab({ Title = "⚙️ Settings", Icon = "settings" }),
}

-- ============================================
-- VARIÁVEIS GLOBAIS
-- ============================================

local player = game.Players.LocalPlayer
local camera = game.Workspace.CurrentCamera
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local Mouse = player:GetMouse()

-- Configurações do Aimbot
local aimbotConfig = {
    enabled = false,
    holdMode = true,
    holdKey = Enum.KeyCode.ButtonR2,
    toggleKey = Enum.KeyCode.F,
    fov = 120,
    smoothness = 0.35,
    teamCheck = false,
    wallCheck = false,
    blatantMode = false,
    hitbox = "Cabeça",
    showFOV = false,
    aimKey = Enum.KeyCode.F,
}

-- Configurações do ESP
local espConfig = {
    enabled = false,
    boxes = false,
    healthbars = false,
    names = false,
    distance = false,
    teamcheck = false,
    skeleton = false,
    showSelf = false,
    maxDistance = 500,
}

-- Configurações de Movimento (SPINBOT + SPEED)
local movementConfig = {
    spinbotEnabled = false,
    spinSpeed = 30,
    spinMode = "Horizontal",
    spinTarget = "HumanoidRootPart",
    randomJitter = false,
    -- SPEED
    speedEnabled = false,
    speedValue = 16,
    speedMode = "Normal",
    speedKey = Enum.KeyCode.LeftShift,
    speedKeyHeld = false,
}

-- Configurações de FPS Boost
local fpsConfig = {
    enabled = false,
    lowGraphics = true,
    noEffects = true,
    noShadows = true,
    noTextures = false,
    lowRenderDistance = true,
}

-- Variáveis de estado
local holdAimActive = false
local drawingObjects = {}

-- ============================================
-- FUNÇÕES DO AIMBOT
-- ============================================

local function IsVisible(targetPos)
    local origin = camera.CFrame.Position
    local direction = (targetPos - origin).unit * (targetPos - origin).magnitude
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {player.Character, camera}
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    local result = game.Workspace:Raycast(origin, direction, raycastParams)
    return result == nil or result.Instance:IsDescendantOf(targetPos.Parent)
end

local function GetClosestTarget()
    if not aimbotConfig.enabled then return nil end

    local char = player.Character
    if not char then return nil end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end

    local myPos = hrp.Position
    local cameraCF = camera.CFrame
    local myLook = cameraCF.LookVector

    local best = nil
    local bestScore = math.huge
    local fovRad = math.rad(aimbotConfig.fov)

    for _, plr in pairs(game.Players:GetPlayers()) do
        if plr == player then continue end
        if not plr.Character then continue end

        local root = plr.Character:FindFirstChild("HumanoidRootPart")
        local humanoid = plr.Character:FindFirstChild("Humanoid")
        if not root or not humanoid or humanoid.Health <= 0 then continue end

        if aimbotConfig.teamCheck and plr.Team == player.Team then
            continue
        end

        local targetPart
        if aimbotConfig.hitbox == "Cabeça" then
            targetPart = plr.Character:FindFirstChild("Head")
        elseif aimbotConfig.hitbox == "Peito" then
            targetPart = plr.Character:FindFirstChild("UpperTorso") or plr.Character:FindFirstChild("Torso")
        else
            targetPart = root
        end
        if not targetPart then continue end

        local targetPos = targetPart.Position
        local direction = (targetPos - myPos).unit
        local angle = math.acos(math.clamp(myLook:Dot(direction), -1, 1))

        if angle > fovRad then
            continue
        end

        if aimbotConfig.wallCheck and not IsVisible(targetPart) then
            continue
        end

        local dist = (targetPos - myPos).Magnitude
        local score = dist + angle * 10
        if score < bestScore then
            bestScore = score
            best = {
                Player = plr,
                Part = targetPart,
                Position = targetPos,
                Distance = dist,
                Angle = angle
            }
        end
    end

    return best
end

local function AimAtTarget(target)
    if not target or not target.Part then return end

    local targetPos = target.Position

    if aimbotConfig.blatantMode then
        camera.CFrame = CFrame.new(camera.CFrame.Position, targetPos)
        return
    end

    local smooth = aimbotConfig.smoothness
    if smooth < 0.01 then smooth = 0.01 end
    if smooth > 1 then smooth = 1 end

    local currentCF = camera.CFrame
    local targetCF = CFrame.new(currentCF.Position, targetPos)
    local newCF = currentCF:lerp(targetCF, smooth)
    camera.CFrame = newCF
end

-- ============================================
-- TECLAS DE ATIVAÇÃO DO AIMBOT
-- ============================================

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == aimbotConfig.aimKey then
        if aimbotConfig.holdMode then
            holdAimActive = true
        else
            aimbotConfig.enabled = not aimbotConfig.enabled
            Library:Notify({
                Title = "🎯 Aim Assist",
                Content = aimbotConfig.enabled and "✅ Ativado!" or "❌ Desativado!",
                Duration = 1.5
            })
        end
    end
end)

UserInputService.InputEnded:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == aimbotConfig.aimKey and aimbotConfig.holdMode then
        holdAimActive = false
    end
end)

RunService.RenderStepped:Connect(function()
    local aimActive = false
    if aimbotConfig.holdMode then
        aimActive = aimbotConfig.enabled and holdAimActive
    else
        aimActive = aimbotConfig.enabled
    end
    if aimActive then
        local target = GetClosestTarget()
        if target then
            AimAtTarget(target)
        end
    end
end)

-- ============================================
-- FOV CIRCLE
-- ============================================

local FOVGui = Instance.new("ScreenGui")
FOVGui.Name = "DRAXZS_FOV"
FOVGui.Parent = player:WaitForChild("PlayerGui")
FOVGui.ResetOnSpawn = false
FOVGui.Enabled = false

local FOVFrame = Instance.new("Frame")
FOVFrame.Size = UDim2.new(0, 1, 0, 1)
FOVFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
FOVFrame.BackgroundTransparency = 1
FOVFrame.BorderSizePixel = 0
FOVFrame.Parent = FOVGui

local FOVCircle = Instance.new("Frame")
FOVCircle.BackgroundTransparency = 1
FOVCircle.BorderSizePixel = 0
FOVCircle.Parent = FOVFrame

local CircleCorner = Instance.new("UICorner")
CircleCorner.CornerRadius = UDim.new(1, 0)
CircleCorner.Parent = FOVCircle

local CircleStroke = Instance.new("UIStroke")
CircleStroke.Thickness = 2
CircleStroke.Color = Color3.fromRGB(255, 0, 0)
CircleStroke.Transparency = 0.5
CircleStroke.Parent = FOVCircle

local Dot = Instance.new("Frame")
Dot.Size = UDim2.new(0, 4, 0, 4)
Dot.Position = UDim2.new(0.5, -2, 0.5, -2)
Dot.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
Dot.BackgroundTransparency = 0.3
Dot.BorderSizePixel = 0
Dot.Parent = FOVFrame

local DotCorner = Instance.new("UICorner")
DotCorner.CornerRadius = UDim.new(1, 0)
DotCorner.Parent = Dot

local function UpdateFOVCircle()
    if aimbotConfig.showFOV then
        FOVGui.Enabled = true
        local fovPixels = aimbotConfig.fov * 2.5
        FOVCircle.Size = UDim2.new(0, fovPixels, 0, fovPixels)
        FOVCircle.Position = UDim2.new(0.5, -fovPixels/2, 0.5, -fovPixels/2)
    else
        FOVGui.Enabled = false
    end
end
UpdateFOVCircle()

-- ============================================
-- FUNÇÕES DE DESENHO DO ESP
-- ============================================

local function WorldToScreen(worldPos)
    local screenPos, onScreen = camera:WorldToViewportPoint(worldPos)
    if onScreen then
        return Vector2.new(screenPos.X, screenPos.Y)
    end
    return nil
end

local function ClearESPObjects()
    for _, obj in pairs(drawingObjects) do
        if obj and obj.Remove then
            obj:Remove()
        end
    end
    drawingObjects = {}
end

local function DrawLine(p1, p2, color, thickness)
    local p1s = WorldToScreen(p1)
    local p2s = WorldToScreen(p2)
    if p1s and p2s then
        local line = Drawing.new("Line")
        line.From = p1s
        line.To = p2s
        line.Thickness = thickness or 2
        line.Color = color or Color3.fromRGB(255, 255, 255)
        line.Visible = true
        table.insert(drawingObjects, line)
        return line
    end
    return nil
end

local function DrawSkeleton(plr, color)
    local char = plr.Character
    if not char then return end
    local humanoid = char:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end

    local head = char:FindFirstChild("Head")
    local torso = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso")
    local leftArm = char:FindFirstChild("LeftUpperArm")
    local rightArm = char:FindFirstChild("RightUpperArm")
    local leftLeg = char:FindFirstChild("LeftUpperLeg")
    local rightLeg = char:FindFirstChild("RightUpperLeg")
    local leftHand = char:FindFirstChild("LeftHand") or char:FindFirstChild("LeftLowerArm")
    local rightHand = char:FindFirstChild("RightHand") or char:FindFirstChild("RightLowerArm")
    local leftFoot = char:FindFirstChild("LeftFoot") or char:FindFirstChild("LeftLowerLeg")
    local rightFoot = char:FindFirstChild("RightFoot") or char:FindFirstChild("RightLowerLeg")

    if head and torso then DrawLine(head.Position, torso.Position, color, 2) end
    if leftArm and torso then DrawLine(leftArm.Position, torso.Position, color, 2) end
    if rightArm and torso then DrawLine(rightArm.Position, torso.Position, color, 2) end
    if leftLeg and torso then DrawLine(leftLeg.Position, torso.Position, color, 2) end
    if rightLeg and torso then DrawLine(rightLeg.Position, torso.Position, color, 2) end
    if leftHand and leftArm then DrawLine(leftHand.Position, leftArm.Position, color, 2) end
    if rightHand and rightArm then DrawLine(rightHand.Position, rightArm.Position, color, 2) end
    if leftFoot and leftLeg then DrawLine(leftFoot.Position, leftLeg.Position, color, 2) end
    if rightFoot and rightLeg then DrawLine(rightFoot.Position, rightLeg.Position, color, 2) end
end

local function DrawBox(plr)
    local char = plr.Character
    if not char then return end
    local humanoid = char:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end

    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local minX, maxX, minY, maxY = math.huge, -math.huge, math.huge, -math.huge
    for _, part in pairs(char:GetChildren()) do
        if part:IsA("BasePart") then
            local pos = WorldToScreen(part.Position)
            if pos then
                local size = part.Size
                local corners = {
                    Vector3.new(-size.X/2, -size.Y/2, -size.Z/2),
                    Vector3.new( size.X/2, -size.Y/2, -size.Z/2),
                    Vector3.new(-size.X/2,  size.Y/2, -size.Z/2),
                    Vector3.new( size.X/2,  size.Y/2, -size.Z/2),
                    Vector3.new(-size.X/2, -size.Y/2,  size.Z/2),
                    Vector3.new( size.X/2, -size.Y/2,  size.Z/2),
                    Vector3.new(-size.X/2,  size.Y/2,  size.Z/2),
                    Vector3.new( size.X/2,  size.Y/2,  size.Z/2),
                }
                for _, offset in pairs(corners) do
                    local worldPos = part.CFrame:PointToWorldSpace(offset)
                    local screenPos = WorldToScreen(worldPos)
                    if screenPos then
                        minX = math.min(minX, screenPos.X)
                        maxX = math.max(maxX, screenPos.X)
                        minY = math.min(minY, screenPos.Y)
                        maxY = math.max(maxY, screenPos.Y)
                    end
                end
            end
        end
    end

    if minX ~= math.huge and maxX ~= -math.huge and minY ~= math.huge and maxY ~= -math.huge then
        local color = Color3.fromRGB(255, 0, 0)
        local lines = {
            {Vector2.new(minX, minY), Vector2.new(maxX, minY)},
            {Vector2.new(maxX, minY), Vector2.new(maxX, maxY)},
            {Vector2.new(maxX, maxY), Vector2.new(minX, maxY)},
            {Vector2.new(minX, maxY), Vector2.new(minX, minY)},
        }
        for _, pair in pairs(lines) do
            local line = Drawing.new("Line")
            line.From = pair[1]
            line.To = pair[2]
            line.Thickness = 2
            line.Color = color
            line.Visible = true
            table.insert(drawingObjects, line)
        end
        return minX, maxX, minY, maxY
    end
    return nil, nil, nil, nil
end

local function DrawHealthBar(plr, color, minX, maxX, minY, maxY)
    if not minX or not maxX or not minY or not maxY then return end

    local char = plr.Character
    if not char then return end
    local humanoid = char:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end

    local healthPercent = humanoid.Health / humanoid.MaxHealth
    local barWidth = 6
    local barHeight = maxY - minY

    local x = maxX + 8
    local y = minY

    local bg = Drawing.new("Line")
    bg.From = Vector2.new(x, y)
    bg.To = Vector2.new(x, y + barHeight)
    bg.Thickness = barWidth
    bg.Color = Color3.fromRGB(50, 50, 50)
    bg.Visible = true
    table.insert(drawingObjects, bg)

    local healthColor = color or Color3.fromRGB(0, 255, 0)
    local healthLine = Drawing.new("Line")
    local fillHeight = barHeight * healthPercent
    healthLine.From = Vector2.new(x, y + barHeight - fillHeight)
    healthLine.To = Vector2.new(x, y + barHeight)
    healthLine.Thickness = barWidth
    healthLine.Color = healthColor
    healthLine.Visible = true
    table.insert(drawingObjects, healthLine)
end

local function DrawName(plr, color)
    local char = plr.Character
    if not char then return end
    local humanoid = char:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end

    local head = char:FindFirstChild("Head")
    if not head then return end
    local headPos = WorldToScreen(head.Position)
    if not headPos then return end

    local txt = Drawing.new("Text")
    txt.Text = plr.Name
    txt.Size = 14
    txt.Font = 0
    txt.Position = Vector2.new(headPos.X - 40, headPos.Y - 35)
    txt.Color = color or Color3.fromRGB(255, 255, 255)
    txt.Outlined = true
    txt.Visible = true
    table.insert(drawingObjects, txt)
end

local function DrawDistance(plr, color)
    local char = plr.Character
    if not char then return end
    local humanoid = char:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end

    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local head = char:FindFirstChild("Head")
    if not head then return end
    local headPos = WorldToScreen(head.Position)
    if not headPos then return end

    local dist = (hrp.Position - camera.CFrame.Position).Magnitude
    local txt = Drawing.new("Text")
    txt.Text = math.floor(dist) .. "m"
    txt.Size = 12
    txt.Font = 0
    txt.Position = Vector2.new(headPos.X - 20, headPos.Y - 20)
    txt.Color = color or Color3.fromRGB(200, 200, 200)
    txt.Outlined = true
    txt.Visible = true
    table.insert(drawingObjects, txt)
end

-- ============================================
-- LOOP PRINCIPAL DO ESP (com distância máxima)
-- ============================================

RunService.Heartbeat:Connect(function()
    ClearESPObjects()
    if not espConfig.enabled then return end

    local myChar = player.Character
    local myPos
    if myChar then
        local myHrp = myChar:FindFirstChild("HumanoidRootPart")
        if myHrp then myPos = myHrp.Position end
    end

    local playersToDraw = {}
    if espConfig.showSelf then
        table.insert(playersToDraw, player)
    end
    for _, plr in pairs(game.Players:GetPlayers()) do
        if plr ~= player then
            table.insert(playersToDraw, plr)
        end
    end

    for _, plr in pairs(playersToDraw) do
        if not plr.Character then continue end

        local isSelf = (plr == player)
        local isAlly = (not isSelf and plr.Team == player.Team)

        if espConfig.teamcheck and isAlly then
            continue
        end

        -- Checagem de distância máxima do ESP
        if not isSelf and myPos then
            local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                local dist = (hrp.Position - myPos).Magnitude
                if dist > espConfig.maxDistance then
                    continue
                end
            end
        end

        local minX, maxX, minY, maxY = DrawBox(plr)

        if espConfig.healthbars and minX then
            DrawHealthBar(plr, Color3.fromRGB(0, 255, 0), minX, maxX, minY, maxY)
        end

        if espConfig.names then
            DrawName(plr, Color3.fromRGB(255, 255, 255))
        end

        if espConfig.distance then
            DrawDistance(plr, Color3.fromRGB(200, 200, 200))
        end

        if espConfig.skeleton then
            DrawSkeleton(plr, Color3.fromRGB(255, 255, 255))
        end
    end
end)

-- ============================================
-- SPINBOT (MOVIMENTO)
-- ============================================

RunService.Heartbeat:Connect(function()
    if not movementConfig.spinbotEnabled then return end

    local char = player.Character
    if not char then return end

    local hrp = char:FindFirstChild(movementConfig.spinTarget)
    if not hrp then return end

    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end

    local speed = math.rad(movementConfig.spinSpeed * 0.6)

    if movementConfig.spinMode == "Horizontal" then
        hrp.CFrame = hrp.CFrame * CFrame.Angles(0, speed, 0)
    elseif movementConfig.spinMode == "Vertical" then
        hrp.CFrame = hrp.CFrame * CFrame.Angles(speed, 0, 0)
    elseif movementConfig.spinMode == "Random" then
        hrp.CFrame = hrp.CFrame * CFrame.Angles(
            speed * (math.random() - 0.5),
            speed * (math.random() - 0.5),
            speed * (math.random() - 0.5)
        )
    end

    if movementConfig.randomJitter then
        local jitter = Vector3.new(
            (math.random() - 0.5) * 0.5,
            0,
            (math.random() - 0.5) * 0.5
        )
        hrp.CFrame = hrp.CFrame + jitter
    end
end)

-- ============================================
-- SPEED HACK (MOVIMENTO)
-- ============================================

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == movementConfig.speedKey then
        movementConfig.speedKeyHeld = true
    end
end)

UserInputService.InputEnded:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == movementConfig.speedKey then
        movementConfig.speedKeyHeld = false
    end
end)

RunService.Heartbeat:Connect(function()
    local char = player.Character
    if not char then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return end

    if not movementConfig.speedEnabled then
        humanoid.WalkSpeed = 16
        return
    end

    local applySpeed = false
    if movementConfig.speedMode == "Normal" then
        applySpeed = true
    elseif movementConfig.speedMode == "Shift" then
        applySpeed = movementConfig.speedKeyHeld
    elseif movementConfig.speedMode == "Always" then
        applySpeed = true
    end

    if applySpeed then
        humanoid.WalkSpeed = movementConfig.speedValue
    else
        humanoid.WalkSpeed = 16
    end
end)

-- ============================================
-- FPS BOOST
-- ============================================

local function ApplyFPSBoost()
    local Terrain = workspace:FindFirstChildOfClass("Terrain")

    if fpsConfig.lowGraphics then
        pcall(function()
            settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
            Lighting.GlobalShadows = false
            Lighting.FogEnd = 100000
            Lighting.Brightness = 0
        end)
    end

    if fpsConfig.noEffects then
        pcall(function()
            for _, v in pairs(Lighting:GetChildren()) do
                if v:IsA("PostEffect") or v:IsA("Atmosphere") or v:IsA("Sky") then
                    v.Enabled = false
                end
            end
        end)
    end

    if fpsConfig.noShadows then
        pcall(function()
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("BasePart") then
                    v.CastShadow = false
                end
            end
        end)
    end

    if fpsConfig.noTextures then
        pcall(function()
            for _, v in pairs(workspace:GetDescendants()) do
                if v:IsA("Decal") or v:IsA("Texture") then
                    v.Transparency = 1
                end
            end
        end)
    end

    if fpsConfig.lowRenderDistance then
        pcall(function()
            if Terrain then
                Terrain.WaterWaveSize = 0
                Terrain.WaterWaveSpeed = 0
                Terrain.WaterReflectance = 0
                Terrain.WaterTransparency = 1
            end
            player.CameraMaxZoomDistance = 50
        end)
    end
end

local function RestoreFPSBoost()
    pcall(function()
        settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
        Lighting.GlobalShadows = true
        Lighting.Brightness = 2
    end)
end

RunService.Heartbeat:Connect(function()
    if fpsConfig.enabled then
        ApplyFPSBoost()
    end
end)

-- ============================================
-- 1. WELCOME TAB
-- ============================================

Tabs.Main:AddParagraph({
    Title = "🔥 Bem-vindo, " .. player.Name .. "!",
    Content = "Menu PvP definitivo do DRAXZS.\n\n🎯 Aimbot com FOV ajustável\n👁️ ESP completo (Box vermelha, Health lateral, Nome, Distância, Skeleton)\n📏 Distância máxima configurável no ESP\n🌀 Spinbot na aba Movement\n⚡ Speed Hack na aba Movement\n🚀 FPS Boost na aba Performance\n⚙️ Totalmente personalizável\n\nPressione Insert para abrir/fechar."
})

-- ============================================
-- 2. AIM ASSIST TAB
-- ============================================

Tabs.Aim:AddToggle("AimbotToggle", {
    Title = "🎯 Aimbot",
    Description = "Ativa/Desativa o aimbot",
    Default = false,
    Callback = function(value)
        aimbotConfig.enabled = value
        Library:Notify({
            Title = "🎯 Aimbot",
            Content = value and "✅ Ativado!" or "❌ Desativado!",
            Duration = 1.5
        })
    end
})

local bindButton = Tabs.Aim:AddButton({
    Title = "🔑 Tecla atual: " .. tostring(aimbotConfig.aimKey):gsub("Enum.KeyCode.", ""),
    Description = "Clique para alterar a tecla do aimbot",
    Callback = function()
        Library:Notify({
            Title = "⌨️ Aguardando tecla...",
            Content = "Pressione uma tecla.",
            Duration = 2
        })
        local input = UserInputService.InputBegan:Wait()
        if input then
            local key = input.KeyCode
            if key ~= Enum.KeyCode.Unknown then
                aimbotConfig.aimKey = key
                bindButton:SetTitle("🔑 Tecla atual: " .. tostring(key):gsub("Enum.KeyCode.", ""))
                Library:Notify({
                    Title = "✅ Tecla alterada!",
                    Content = "Aimbot agora ativa com: " .. tostring(key):gsub("Enum.KeyCode.", ""),
                    Duration = 2
                })
            end
        end
    end
})

Tabs.Aim:AddDropdown("AimMode", {
    Title = "🔑 Modo de Ativação",
    Description = "Hold = segurar, Toggle = clicar",
    Values = {"Hold", "Toggle"},
    Default = 1,
    Callback = function(value)
        aimbotConfig.holdMode = (value == "Hold")
    end
})

Tabs.Aim:AddSlider("FOVSlider", {
    Title = "🎯 FOV (Campo de Visão)",
    Description = "Ângulo de busca (30° a 360°)",
    Default = 120,
    Min = 30,
    Max = 360,
    Rounding = 1,
    Callback = function(value)
        aimbotConfig.fov = value
        UpdateFOVCircle()
    end
})

Tabs.Aim:AddSlider("SmoothnessSlider", {
    Title = "🔄 Suavidade (Smoothness)",
    Description = "0 = instantâneo, 100 = muito suave",
    Default = 35,
    Min = 0,
    Max = 100,
    Rounding = 1,
    Callback = function(value)
        aimbotConfig.smoothness = math.max(0.01, value / 100)
    end
})

Tabs.Aim:AddDropdown("HitboxDropdown", {
    Title = "🎯 Hitbox",
    Description = "Parte do corpo",
    Values = {"Cabeça", "Peito", "Root"},
    Default = 1,
    Callback = function(value)
        aimbotConfig.hitbox = value
    end
})

Tabs.Aim:AddToggle("TeamCheckAim", {
    Title = "👥 Team Check (Aimbot)",
    Description = "Ignora membros do seu time",
    Default = false,
    Callback = function(value)
        aimbotConfig.teamCheck = value
    end
})

Tabs.Aim:AddToggle("WallCheckAim", {
    Title = "🧱 Wall Check",
    Description = "Só mira em alvos visíveis",
    Default = false,
    Callback = function(value)
        aimbotConfig.wallCheck = value
    end
})

Tabs.Aim:AddToggle("BlatantMode", {
    Title = "💥 Blatant Mode",
    Description = "Mira instantânea (sem suavidade)",
    Default = false,
    Callback = function(value)
        aimbotConfig.blatantMode = value
    end
})

Tabs.Aim:AddToggle("ShowFOV", {
    Title = "👁️ Mostrar FOV",
    Description = "Mostra o círculo do FOV na tela",
    Default = false,
    Callback = function(value)
        aimbotConfig.showFOV = value
        UpdateFOVCircle()
    end
})

-- ============================================
-- 3. ESP TAB
-- ============================================

Tabs.ESPTab:AddToggle("ESPGlobal", {
    Title = "👁️ ESP Global",
    Description = "Ativa/Desativa todos os ESP",
    Default = false,
    Callback = function(value)
        espConfig.enabled = value
        if not value then
            ClearESPObjects()
        end
    end
})

Tabs.ESPTab:AddToggle("ESPBoxes", {
    Title = "📦 Box ESP",
    Description = "Caixa vermelha ao redor dos jogadores",
    Default = false,
    Callback = function(value)
        espConfig.boxes = value
    end
})

Tabs.ESPTab:AddToggle("ESPHealthBar", {
    Title = "❤️ Health Bar (Lateral)",
    Description = "Barra de vida vertical ao lado do boneco (na altura dele)",
    Default = false,
    Callback = function(value)
        espConfig.healthbars = value
    end
})

Tabs.ESPTab:AddToggle("ESPNames", {
    Title = "📝 Player Names",
    Description = "Mostra o nome dos jogadores",
    Default = false,
    Callback = function(value)
        espConfig.names = value
    end
})

Tabs.ESPTab:AddToggle("ESPDistance", {
    Title = "📏 Distance",
    Description = "Mostra a distância até o jogador",
    Default = false,
    Callback = function(value)
        espConfig.distance = value
    end
})

Tabs.ESPTab:AddToggle("ESPTeamCheck", {
    Title = "👥 Team Check (ESP)",
    Description = "Ignora aliados (exceto se 'Mostrar em si' estiver ativo)",
    Default = false,
    Callback = function(value)
        espConfig.teamcheck = value
    end
})

Tabs.ESPTab:AddToggle("ESPSkeleton", {
    Title = "🦴 Skeleton ESP",
    Description = "Desenha o esqueleto dos jogadores",
    Default = false,
    Callback = function(value)
        espConfig.skeleton = value
    end
})

Tabs.ESPTab:AddToggle("ESPSelf", {
    Title = "👤 Mostrar em si mesmo",
    Description = "Ativa o ESP também no seu próprio personagem",
    Default = false,
    Callback = function(value)
        espConfig.showSelf = value
    end
})

Tabs.ESPTab:AddParagraph({
    Title = "📏 Otimização de Distância",
    Content = "Diminua a distância máxima se o ESP estiver travando o jogo.\nValores baixos (100-300) = melhor performance.\nValores altos (1000-2000) = mostra todo mundo, mas pesa mais."
})

Tabs.ESPTab:AddSlider("ESPMaxDistance", {
    Title = "📏 Distância Máxima do ESP",
    Description = "Só mostra jogadores até essa distância (metros)",
    Default = 500,
    Min = 10,
    Max = 2000,
    Rounding = 1,
    Callback = function(value)
        espConfig.maxDistance = value
    end
})

-- ============================================
-- 4. MOVEMENT TAB (SPINBOT + SPEED)
-- ============================================

Tabs.Movement:AddParagraph({
    Title = "🌀 Spinbot",
    Content = "Gira seu personagem em alta velocidade para dificultar a mira dos inimigos.\n\n⚠️ Velocidades muito altas podem te fazer tomar kick em jogos com anti-cheat."
})

Tabs.Movement:AddToggle("SpinbotToggle", {
    Title = "🌀 Spinbot",
    Description = "Ativa/Desativa o Spinbot",
    Default = false,
    Callback = function(value)
        movementConfig.spinbotEnabled = value
        Library:Notify({
            Title = "🌀 Spinbot",
            Content = value and "✅ Ativado!" or "❌ Desativado!",
            Duration = 1.5
        })
    end
})

Tabs.Movement:AddSlider("SpinSpeedSlider", {
    Title = "⚡ Velocidade do Giro",
    Description = "Quão rápido o personagem gira (1 = lento, 100 = insano)",
    Default = 30,
    Min = 1,
    Max = 100,
    Rounding = 1,
    Callback = function(value)
        movementConfig.spinSpeed = value
    end
})

Tabs.Movement:AddDropdown("SpinModeDropdown", {
    Title = "🎯 Modo do Spin",
    Description = "Direção do giro",
    Values = {"Horizontal", "Vertical", "Random"},
    Default = 1,
    Callback = function(value)
        movementConfig.spinMode = value
    end
})

Tabs.Movement:AddDropdown("SpinTargetDropdown", {
    Title = "🦴 Parte que Gira",
    Description = "Qual parte do corpo gira (Root = corpo todo)",
    Values = {"HumanoidRootPart", "UpperTorso", "Head"},
    Default = 1,
    Callback = function(value)
        movementConfig.spinTarget = value
    end
})

Tabs.Movement:AddToggle("SpinJitterToggle", {
    Title = "📳 Tremida Aleatória (Jitter)",
    Description = "Adiciona um tremor aleatório ao giro (mais 'quebrado')",
    Default = false,
    Callback = function(value)
        movementConfig.randomJitter = value
    end
})

Tabs.Movement:AddParagraph({
    Title = "⚠️ Aviso (Spinbot)",
    Content = "Spinbot pode te fazer tomar kick em alguns jogos.\nSe o jogo detectar, diminua a velocidade ou desligue.\n\nUse por sua conta e risco."
})

-- SPEED HACK

Tabs.Movement:AddParagraph({
    Title = "⚡ Speed Hack",
    Content = "Aumenta a velocidade de caminhada do seu personagem.\n\n⚠️ Velocidades muito altas podem te fazer tomar kick em jogos com anti-cheat."
})

Tabs.Movement:AddToggle("SpeedToggle", {
    Title = "⚡ Speed Hack",
    Description = "Ativa/Desativa o Speed Hack",
    Default = false,
    Callback = function(value)
        movementConfig.speedEnabled = value
        Library:Notify({
            Title = "⚡ Speed Hack",
            Content = value and "✅ Ativado!" or "❌ Desativado!",
            Duration = 1.5
        })
    end
})

Tabs.Movement:AddSlider("SpeedSlider", {
    Title = "🏃 Velocidade (0-100)",
    Description = "Velocidade de caminhada (0 = parado, 100 = insano)",
    Default = 16,
    Min = 0,
    Max = 100,
    Rounding = 1,
    Callback = function(value)
        movementConfig.speedValue = value
    end
})

Tabs.Movement:AddDropdown("SpeedModeDropdown", {
    Title = "🎯 Modo do Speed",
    Description = "Normal = sempre ativo, Shift = segurar tecla, Always = sempre",
    Values = {"Normal", "Shift", "Always"},
    Default = 1,
    Callback = function(value)
        movementConfig.speedMode = value
    end
})

local speedBindButton = Tabs.Movement:AddButton({
    Title = "🔑 Tecla do Speed: " .. tostring(movementConfig.speedKey):gsub("Enum.KeyCode.", ""),
    Description = "Clique para alterar a tecla do Speed",
    Callback = function()
        Library:Notify({
            Title = "⌨️ Aguardando tecla...",
            Content = "Pressione uma tecla.",
            Duration = 2
        })
        local input = UserInputService.InputBegan:Wait()
        if input then
            local key = input.KeyCode
            if key ~= Enum.KeyCode.Unknown then
                movementConfig.speedKey = key
                speedBindButton:SetTitle("🔑 Tecla do Speed: " .. tostring(key):gsub("Enum.KeyCode.", ""))
                Library:Notify({
                    Title = "✅ Tecla alterada!",
                    Content = "Speed agora usa: " .. tostring(key):gsub("Enum.KeyCode.", ""),
                    Duration = 2
                })
            end
        end
    end
})

Tabs.Movement:AddParagraph({
    Title = "⚠️ Aviso (Speed)",
    Content = "Speed Hack pode te fazer tomar kick em alguns jogos.\nSe o jogo detectar, diminua a velocidade ou desligue.\n\nUse por sua conta e risco."
})

-- ============================================
-- 5. PERFORMANCE TAB (FPS BOOST)
-- ============================================

Tabs.Performance:AddParagraph({
    Title = "🚀 FPS Boost",
    Content = "Otimiza o jogo para melhorar o FPS, especialmente útil quando o ESP está ativo.\n\n⚠️ Alguns efeitos visuais serão desativados enquanto estiver ligado.\nAo desligar, as configurações voltam ao normal."
})

Tabs.Performance:AddToggle("FPSBoostToggle", {
    Title = "⚡ FPS Boost",
    Description = "Ativa/Desativa o boost de FPS (principal)",
    Default = false,
    Callback = function(value)
        fpsConfig.enabled = value
        if not value then
            RestoreFPSBoost()
        end
        Library:Notify({
            Title = "⚡ FPS Boost",
            Content = value and "✅ Ativado!" or "❌ Desativado!",
            Duration = 1.5
        })
    end
})

Tabs.Performance:AddToggle("FPSLowGraphics", {
    Title = "🎨 Baixa Qualidade Gráfica",
    Description = "Reduz a qualidade dos gráficos ao mínimo",
    Default = true,
    Callback = function(value)
        fpsConfig.lowGraphics = value
    end
})

Tabs.Performance:AddToggle("FPSNoEffects", {
    Title = "✨ Remover Efeitos",
    Description = "Remove efeitos pós-processamento, atmosfera e céu",
    Default = true,
    Callback = function(value)
        fpsConfig.noEffects = value
    end
})

Tabs.Performance:AddToggle("FPSNoShadows", {
    Title = "🌑 Remover Sombras",
    Description = "Desativa todas as sombras do mapa",
    Default = true,
    Callback = function(value)
        fpsConfig.noShadows = value
    end
})

Tabs.Performance:AddToggle("FPSNoTextures", {
    Title = "🖼️ Remover Texturas",
    Description = "Remove texturas e decals (mais FPS, mas feio)",
    Default = false,
    Callback = function(value)
        fpsConfig.noTextures = value
    end
})

Tabs.Performance:AddToggle("FPSLowRenderDistance", {
    Title = "🔭 Reduzir Renderização",
    Description = "Reduz distância de renderização e qualidade da água",
    Default = true,
    Callback = function(value)
        fpsConfig.lowRenderDistance = value
    end
})

Tabs.Performance:AddParagraph({
    Title = "💡 Dicas",
    Content = "Se ainda estiver travando com ESP ativo:\n\n1. Diminua a 'Distância Máxima do ESP' na aba ESP (200-300m)\n2. Desative o Skeleton ESP (é o mais pesado)\n3. Desative o Health Bar\n4. Mantenha só Box + Names\n5. Ative FPS Boost com Low Graphics + No Shadows + No Effects"
})

-- ============================================
-- 6. SETTINGS TAB
-- ============================================

Tabs.Settings:AddButton({
    Title = "💾 Salvar Configurações",
    Description = "Salva todas as configurações atuais",
    Callback = function()
        SaveManager:Save()
        Library:Notify({ Title = "✅ Salvo!", Content = "Configurações salvas!", Duration = 2 })
    end
})

Tabs.Settings:AddButton({
    Title = "📂 Carregar Configurações",
    Description = "Carrega as configurações salvas",
    Callback = function()
        SaveManager:Load()
        Library:Notify({ Title = "✅ Carregado!", Content = "Configurações carregadas!", Duration = 2 })
    end
})

Tabs.Settings:AddButton({
    Title = "🎨 Gerenciar Interface",
    Description = "Abre o gerenciador de interface",
    Callback = function()
        InterfaceManager:ToggleUI()
    end
})

Tabs.Settings:AddParagraph({
    Title = "⚙️ Informações",
    Content = "DRAXZS PVP MENU v2.3\n\nDesenvolvido por DRAXZS\nMenu baseado na Fluent UI\n\n🎯 Aimbot com FOV ajustável\n👁️ ESP completo + distância máxima\n🌀 Spinbot na aba Movement\n⚡ Speed Hack na aba Movement\n🚀 FPS Boost na aba Performance\n⚡ Otimizado para PvP"
})

Tabs.Settings:AddButton({
    Title = "❌ Fechar Menu",
    Description = "Fecha completamente o menu",
    Callback = function()
        Window:Close()
        Library:Notify({ Title = "👋 Até logo!", Content = "Pressione Insert para reabrir.", Duration = 3 })
    end
})

-- ============================================
-- INICIALIZAR SAVEMANAGER
-- ============================================

SaveManager:SetLibrary(Library)
InterfaceManager:SetLibrary(Library)

SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({})

InterfaceManager:SetFolder("DRAXZS_PVP")
SaveManager:SetFolder("DRAXZS_PVP/saves")

InterfaceManager:BuildInterfaceSection(Tabs.Settings)
SaveManager:BuildConfigSection(Tabs.Settings)

-- ============================================
-- ABRIR/FECHAR MENU (INSERT)
-- ============================================

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        Window:Toggle()
    end
end)

-- ============================================
-- INÍCIO
-- ============================================

Library:Notify({
    Title = "⚡ DRAXZS PVP MENU V2.3 ⚡",
    Content = "Carregado com sucesso!\nPressione Insert para abrir.",
    Duration = 4
})

print("====================================")
print("⚡ DRAXZS PVP MENU V2.3 ⚡")
print("📌 Insert - Abrir/Fechar menu")
print("📌 Aimbot com FOV ajustável")
print("📌 Box vermelha | Health bar lateral")
print("📌 ESP com distância máxima configurável")
print("📌 Spinbot na aba Movement")
print("📌 Speed Hack na aba Movement")
print("📌 FPS Boost na aba Performance")
print("====================================")
