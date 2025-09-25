local BulletTrackingLib = {}

local Players           = game:GetService("Players")
local Workspace         = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService        = game:GetService("RunService")
local UserInputService  = game:GetService("UserInputService")

local LocalPlayer       = Players.LocalPlayer
local Camera            = Workspace.CurrentCamera

if not Camera then
    wait(1)
    Camera = Workspace.CurrentCamera
end

BulletTrackingLib.Enabled = false
BulletTrackingLib.TeamCheck = true
BulletTrackingLib.PerformanceMode = false
BulletTrackingLib.FOV_Radius = 150
BulletTrackingLib.TargetPart = "Head"
BulletTrackingLib.CacheRefreshRate = 1

local FOV_Circle = Drawing.new("Circle")
FOV_Circle.Visible = false
FOV_Circle.Color = Color3.new(1, 1, 1)
FOV_Circle.Thickness = 2
FOV_Circle.NumSides = 64
FOV_Circle.Filled = false

local SemiCircle = Drawing.new("Circle")
SemiCircle.Visible = false
SemiCircle.Thickness = 3
SemiCircle.NumSides = 64
SemiCircle.Filled = false
SemiCircle.Radius = 30
SemiCircle.ArcStart = 180
SemiCircle.ArcEnd = 360

local TargetText = Drawing.new("Text")
TargetText.Visible = false
TargetText.Text = "目标"
TargetText.Color = Color3.new(1, 1, 1)
TargetText.Size = 10
TargetText.Center = true
TargetText.Outline = true

local CachedPlayers = {}
local LastUpdateTime = 0
local UpdateInterval = 0

local function getCamera()
    if not Camera or not Camera:IsA("Camera") then
        Camera = Workspace.CurrentCamera
        if not Camera then
            wait(0.1)
            Camera = Workspace.CurrentCamera
        end
    end
    return Camera
end

local function getTargetPart(character)
    return character and character:FindFirstChild(BulletTrackingLib.TargetPart)
end

local function isTeammate(player)
    if not BulletTrackingLib.TeamCheck then return false end
    
    local localTeam = LocalPlayer.Team
    local targetTeam = player.Team
    
    if not localTeam or not targetTeam then return false end
    
    return localTeam == targetTeam
end

local function safeWorldToViewportPoint(camera, position)
    if not camera or not camera:IsA("Camera") then
        return nil, false
    end
    local success, result = pcall(function()
        return camera:WorldToViewportPoint(position)
    end)
    if success then
        return result, result.Z > 0
    end
    return nil, false
end

local function getClosestPlayerInFOV()
    local camera = getCamera()
    if not camera then return nil end
    
    local currentTime = tick()
    
    if currentTime - LastUpdateTime > BulletTrackingLib.CacheRefreshRate then
        CachedPlayers = Players:GetPlayers()
        LastUpdateTime = currentTime
    end
    
    local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end

    local screenCenter = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    local closestPlayer, closestDistance

    for _, plr in ipairs(CachedPlayers) do
        if plr ~= LocalPlayer and plr.Character and not isTeammate(plr) then
            local targetPart = getTargetPart(plr.Character)
            local humanoid = plr.Character:FindFirstChildOfClass("Humanoid")

            if targetPart and humanoid and humanoid.Health > 0 then
                local screenPos, onScreen = safeWorldToViewportPoint(camera, targetPart.Position)
                if onScreen then
                    local dist = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
                    if dist <= BulletTrackingLib.FOV_Radius and (not closestDistance or dist < closestDistance) then
                        if not BulletTrackingLib.PerformanceMode or dist < BulletTrackingLib.FOV_Radius / 2 then
                            local ray = Ray.new(myRoot.Position, (targetPart.Position - myRoot.Position).Unit * 5000)
                            local hit, pos = Workspace:FindPartOnRayWithIgnoreList(ray, {LocalPlayer.Character})
                            if hit and hit:IsDescendantOf(plr.Character) then
                                closestPlayer = plr
                                closestDistance = dist
                            end
                        else
                            closestPlayer = plr
                            closestDistance = dist
                        end
                    end
                end
            end
        end
    end
    return closestPlayer
end

local oldNamecall
oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
    local method = getnamecallmethod()
    local args = {...}

    if method == "Raycast" and not checkcaller() and BulletTrackingLib.Enabled then
        local camera = getCamera()
        if not camera then return oldNamecall(self, ...) end
        
        local targetPlr = getClosestPlayerInFOV()
        if targetPlr and not isTeammate(targetPlr) then
            local targetPart = getTargetPart(targetPlr.Character)
            if targetPart then
                local origin = args[1] or camera.CFrame.Position
                return {
                    Instance = targetPart,
                    Position = targetPart.Position,
                    Normal = (origin - targetPart.Position).Unit,
                    Material = Enum.Material.Plastic,
                    Distance = (targetPart.Position - origin).Magnitude
                }
            end
        end
    end
    return oldNamecall(self, ...)
end))

local RenderConnection
local function setupRenderLoop()
    if RenderConnection then
        RenderConnection:Disconnect()
    end
    
    RenderConnection = RunService.RenderStepped:Connect(function()
        local camera = getCamera()
        if not camera then return end
        
        UpdateInterval = UpdateInterval + 1
        
        if BulletTrackingLib.PerformanceMode and UpdateInterval % 2 ~= 0 then
            return
        end
        
        if FOV_Circle.Visible then
            FOV_Circle.Position = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
            FOV_Circle.Radius = BulletTrackingLib.FOV_Radius
        end

        local targetPlr = BulletTrackingLib.Enabled and getClosestPlayerInFOV()
        
        if targetPlr and not isTeammate(targetPlr) then
            local targetPart = getTargetPart(targetPlr.Character)
            if targetPart then
                local screenPos, onScreen = safeWorldToViewportPoint(camera, targetPart.Position + Vector3.new(0, 2.5, 0))
                if onScreen then
                    SemiCircle.Position = Vector2.new(screenPos.X, screenPos.Y)
                    SemiCircle.Visible = true
                    TargetText.Position = Vector2.new(screenPos.X, screenPos.Y + SemiCircle.Radius / 2)
                    TargetText.Visible = true
                    
                    if isTeammate(targetPlr) then
                        SemiCircle.Color = Color3.new(0, 1, 0)
                        TargetText.Color = Color3.new(0, 1, 0)
                    else
                        SemiCircle.Color = Color3.new(1, 0, 0)
                        TargetText.Color = Color3.new(1, 0, 0)
                    end
                else
                    SemiCircle.Visible = false
                    TargetText.Visible = false
                end
            else
                SemiCircle.Visible = false
                TargetText.Visible = false
            end
        else
            SemiCircle.Visible = false
            TargetText.Visible = false
        end
    end)
end

function BulletTrackingLib.Toggle(enabled)
    BulletTrackingLib.Enabled = enabled
    FOV_Circle.Visible = enabled
    SemiCircle.Visible = false
    TargetText.Visible = false
    
    if enabled then
        setupRenderLoop()
    elseif RenderConnection then
        RenderConnection:Disconnect()
        RenderConnection = nil
    end
end

function BulletTrackingLib.UpdateSettings(newSettings)
    for setting, value in pairs(newSettings) do
        if BulletTrackingLib[setting] ~= nil then
            BulletTrackingLib[setting] = value
        end
    end
end

function BulletTrackingLib.GetSettings()
    return {
        Enabled = BulletTrackingLib.Enabled,
        TeamCheck = BulletTrackingLib.TeamCheck,
        PerformanceMode = BulletTrackingLib.PerformanceMode,
        FOV_Radius = BulletTrackingLib.FOV_Radius,
        TargetPart = BulletTrackingLib.TargetPart,
        CacheRefreshRate = BulletTrackingLib.CacheRefreshRate
    }
end

game:GetService("Players").PlayerRemoving:Connect(function(player)
    for i, plr in ipairs(CachedPlayers) do
        if plr == player then
            table.remove(CachedPlayers, i)
            break
        end
    end
end)

game:GetService("Players").LocalPlayer:GetPropertyChangedSignal("UserId"):Connect(function()
    if RenderConnection then
        RenderConnection:Disconnect()
    end
    if oldNamecall then
        hookmetamethod(game, "__namecall", oldNamecall)
    end
end)

return BulletTrackingLib