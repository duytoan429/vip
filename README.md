-- ===== Services =====
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ===== Load Dtoan GUI =====
local Dtoan = loadstring(game:HttpGet('https://raw.githubusercontent.com/duytoan429/Menu/refs/heads/main/sourcebydtoan.lua.txt'))()
local Window = Dtoan:CreateWindow({
Name = "Dtoan",
Icon = 0,
LoadingTitle = "Dtoan",
Theme = "Default",
ToggleUIKeybind = "K",
DisableDtoanPrompts = false,
DisableBuildWarnings = false,
ConfigurationSaving = {Enabled = true, FolderName = nil, FileName = "Dtoan Hub"},
Discord = {Enabled = false, Invite = "https://discord.gg/9Xhw2AMKw", RememberJoins = true},
KeySystem = false,
KeySettings = {Title = "Untitled", FileName = "Key", SaveKey = true, GrabKeyFromSite = false, Key = {"Hello"}}
})

-- ===== Aim bot Tab =====
local aimEnabled = false
local fovRadius = 120
local aimSmooth = 0.35
local wallCheckEnabled = true
local whitelist = {workspace.Terrain}

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AimBot_FOVGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
local fovFrame = Instance.new("Frame", screenGui)
fovFrame.AnchorPoint = Vector2.new(0.5,0.5)
fovFrame.Position = UDim2.new(0.5,0,0.5,0)
fovFrame.BackgroundTransparency = 1
fovFrame.Visible = false
local fovStroke = Instance.new("UIStroke", fovFrame)
fovStroke.Thickness = 1 -- Giảm từ 2 xuống 1 để mỏng hơn
fovStroke.Transparency = 0.3
fovStroke.Color = Color3.fromRGB(0, 255, 0) -- Màu xanh lá cố định
local fovCorner = Instance.new("UICorner", fovFrame)
fovCorner.CornerRadius = UDim.new(1,0)

local function updateFOVVisual()
local diameter = math.clamp(fovRadius*2, 20, 1000)
fovFrame.Size = UDim2.new(0, diameter, 0, diameter)
end
updateFOVVisual()

-- Đã xóa phần màu chuyển sắc và thay bằng màu xanh lá cố định

local function isVisible(part)
if not part or not part.Parent then return false end
if not wallCheckEnabled then return true end
local origin = Camera.CFrame.Position
local direction = part.Position - origin
local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Blacklist
rayParams.FilterDescendantsInstances = {LocalPlayer.Character, unpack(whitelist)}
local result = workspace:Raycast(origin,direction,rayParams)
if result then return result.Instance:IsDescendantOf(part.Parent) end
return true
end

RunService.RenderStepped:Connect(function()
if aimEnabled then
local center=Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
local bestPart,bestDist=nil,math.huge
for _,player in pairs(Players:GetPlayers()) do
if player~=LocalPlayer and player.Character and player.Character:FindFirstChild("Head") and player.Character:FindFirstChild("Humanoid") and player.Character.Humanoid.Health>0 then
local head = player.Character.Head
local screenPos,onScreen=Camera:WorldToViewportPoint(head.Position)
local d=(Vector2.new(screenPos.X,screenPos.Y)-center).Magnitude
if onScreen and d<=fovRadius and d<bestDist and isVisible(head) then bestDist=d; bestPart=head end
end
end
if bestPart then Camera.CFrame=Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position,bestPart.Position),aimSmooth) end
end
end)

local AimTab = Window:CreateTab("Aim bot", 4483362458)
AimTab:CreateToggle({Name="Aimbot", CurrentValue=aimEnabled, Callback=function(v) aimEnabled=v; fovFrame.Visible=v end})
AimTab:CreateToggle({Name="Wall Check", CurrentValue=wallCheckEnabled, Callback=function(v) wallCheckEnabled=v end})
AimTab:CreateButton({Name="Reset Defaults", Callback=function() aimEnabled=false; fovRadius=120; aimSmooth=0.35; wallCheckEnabled=true; fovFrame.Visible=false; updateFOVVisual() end})
AimTab:CreateSlider({Name="FOV (pixels)", Range={20,1000}, Increment=10, CurrentValue=fovRadius, Callback=function(v) fovRadius=v; updateFOVVisual() end})
AimTab:CreateSlider({Name="Aim Smooth", Range={0.01,1}, Increment=0.01, CurrentValue=aimSmooth, Callback=function(v) aimSmooth=v end})

-- ===== ESP Tab =====
local ESPTab = Window:CreateTab("ESP", 4483362458)
local espEnabled = false
local espBoxEnabled = false
local espLineEnabled = false
local espHealthEnabled = false
local espTable = {}
local linesTable = {}

local function getTeamColor(player)
if player.Team and player.Team.TeamColor then
return player.Team.TeamColor.Color
end
return Color3.fromRGB(255,255,255)
end

local function createESPBox(player)
if player == LocalPlayer or espTable[player] then return end
local box = Drawing.new("Square")
box.Thickness = 1
box.Filled = false
local name = Drawing.new("Text")
name.Size = 12
name.Center = true
name.Outline = true
local distTxt = Drawing.new("Text")
distTxt.Size = 14
distTxt.Center = true
distTxt.Outline = true
distTxt.Color = Color3.fromRGB(255,255,255)
local healthBar = Drawing.new("Square")
healthBar.Filled = true
healthBar.Thickness = 1
espTable[player] = {box = box, name = name, dist = distTxt, healthBar = healthBar}
end

local function removeESPBox(player)
local data = espTable[player]
if not data then return end
pcall(function() data.box:Remove() end)
pcall(function() data.name:Remove() end)
pcall(function() data.dist:Remove() end)
pcall(function() data.healthBar:Remove() end)
espTable[player] = nil
end

for _, pl in ipairs(Players:GetPlayers()) do
createESPBox(pl)
end
Players.PlayerAdded:Connect(createESPBox)
Players.PlayerRemoving:Connect(removeESPBox)

local function worldToViewportVec(pos)
local p, onScreen = Camera:WorldToViewportPoint(pos)
return Vector2.new(p.X, p.Y), onScreen, p.Z
end

local function createLine()
local line = Drawing.new("Line")
line.Thickness = 2
line.Visible = false
return line
end

local function removeLine(player)
if linesTable[player] then
linesTable[player]:Remove()
linesTable[player] = nil
end
end
Players.PlayerRemoving:Connect(removeLine)

-- Player Count
local playerCountText = Drawing.new("Text")
playerCountText.Size = 60
playerCountText.Center = true
playerCountText.Outline = true
playerCountText.Position = Vector2.new(Camera.ViewportSize.X/2, 50)
playerCountText.Color = Color3.fromRGB(255,0,0)
playerCountText.Visible = false

RunService.RenderStepped:Connect(function()
-- Player Count
if espEnabled then
local aliveCount = 0
for _, player in pairs(Players:GetPlayers()) do
if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Humanoid") and player.Character.Humanoid.Health > 0 then
aliveCount = aliveCount + 1
end
end
playerCountText.Text = aliveCount > 0 and tostring(aliveCount) or "Clear"
playerCountText.Visible = true
else
playerCountText.Visible = false
end

-- ESP Box / Health  
for player, data in pairs(espTable) do  
    local char = player.Character  
    if not espEnabled or not char or not char:FindFirstChild("HumanoidRootPart") or not char:FindFirstChild("Head") or not char:FindFirstChild("Humanoid") then  
        data.box.Visible, data.name.Visible, data.dist.Visible, data.healthBar.Visible = false, false, false, false  
    else  
        local root, head, humanoid = char.HumanoidRootPart, char.Head, char:FindFirstChild("Humanoid")  
        local topPos, topVis = worldToViewportVec(head.Position + Vector3.new(0,0.5,0))  
        local bottomPos, botVis = worldToViewportVec(root.Position - Vector3.new(0,3,0))  
        local centerPos, cenVis = worldToViewportVec(root.Position)  

        if topVis and botVis and cenVis then  
            local height = math.abs(bottomPos.Y - topPos.Y) * 1.5  
            local width = height / 2  
            local x, y = centerPos.X - width/2, topPos.Y  

            data.box.Visible = espBoxEnabled  
            data.box.Size = Vector2.new(width, height)  
            data.box.Position = Vector2.new(x, y)  
            data.box.Color = getTeamColor(player)  

            data.name.Visible = espBoxEnabled  
            data.name.Text = player.Name  
            data.name.Position = Vector2.new(centerPos.X, topPos.Y - 14)  
            data.name.Color = getTeamColor(player)  

            local dist = math.floor((LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")  
                and (LocalPlayer.Character.HumanoidRootPart.Position - root.Position).Magnitude) or 0)  
            data.dist.Visible = espBoxEnabled  
            data.dist.Text = "["..dist.." studs]"  
            data.dist.Position = Vector2.new(centerPos.X, bottomPos.Y + 14)  

            if espHealthEnabled then  
                local healthPercent = humanoid.Health / humanoid.MaxHealth  
                local maxBarWidth = 4  
                local minBarWidth = 1  
                local barWidth = math.clamp(maxBarWidth - (dist/50), minBarWidth, maxBarWidth)  
                data.healthBar.Size = Vector2.new(barWidth, height * healthPercent)  
                data.healthBar.Position = Vector2.new(x + width + 2, y + height*(1 - healthPercent))  
                data.healthBar.Color = Color3.fromRGB(255*(1-healthPercent), 255*healthPercent, 0)  
                data.healthBar.Visible = true  
            else  
                data.healthBar.Visible = false  
            end  
        else  
            data.box.Visible, data.name.Visible, data.dist.Visible, data.healthBar.Visible = false, false, false, false  
        end  
    end  
end  

-- ESP Lines  
for _, player in pairs(Players:GetPlayers()) do  
    if espEnabled and espLineEnabled and player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") then  
        if not linesTable[player] then linesTable[player] = createLine() end  
        local headPos, onScreen = Camera:WorldToViewportPoint(player.Character.Head.Position + Vector3.new(0,0.5,0))  
        if onScreen then  
            linesTable[player].From = Vector2.new(Camera.ViewportSize.X/2, 50)  
            linesTable[player].To = Vector2.new(headPos.X, headPos.Y)  
            linesTable[player].Color = getTeamColor(player)  
            linesTable[player].Visible = true  
        else  
            linesTable[player].Visible = false  
        end  
    elseif linesTable[player] then  
        linesTable[player].Visible = false  
    end  
end

end)

-- ESP Toggles
ESPTab:CreateToggle({Name="Enable ESP", CurrentValue=false, Callback=function(v) espEnabled=v end})
ESPTab:CreateToggle({Name="ESP Box", CurrentValue=false, Callback=function(v) espBoxEnabled=v end})
ESPTab:CreateToggle({Name="ESP Line", CurrentValue=false, Callback=function(v) espLineEnabled=v end})
ESPTab:CreateToggle({Name="ESP Health", CurrentValue=false, Callback=function(v) espHealthEnabled=v end})

-- ===== Other Tab =====
local OtherTab = Window:CreateTab("Khác", 4483362458)
local InfiniteJumpEnabled=false
OtherTab:CreateToggle({Name="Infinity Jump", CurrentValue=false, Callback=function(v) InfiniteJumpEnabled=v end})
UserInputService.JumpRequest:Connect(function()
if InfiniteJumpEnabled and LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
LocalPlayer.Character:FindFirstChildOfClass("Humanoid"):ChangeState("Jumping")
end
end)

OtherTab:CreateSlider({Name="Player Speed", Range={16,200}, Increment=5, CurrentValue=16, Callback=function(v)
if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed=v
end
end})

local noclipEnabled=false
local noclipConnection
OtherTab:CreateToggle({Name="Noclip (xuyên vật thể)", CurrentValue=false, Callback=function(v)
noclipEnabled=v
if noclipEnabled then
noclipConnection=RunService.Stepped:Connect(function()
if LocalPlayer.Character then
for _,p in pairs(LocalPlayer.Character:GetChildren()) do
if p:IsA("BasePart") then
p.CanCollide=false
end
end
end
end)
elseif noclipConnection then
noclipConnection:Disconnect()
noclipConnection=nil
end
end})

OtherTab:CreateToggle({Name="Fly Gui", CurrentValue=false, Callback=function(v)
if v then loadstring(game:HttpGet("https://raw.githubusercontent.com/XNEOFF/FlyGuiV3/main/FlyGuiV3.txt"))() end
end})

-- ===== Script Tổng Hợp Tab =====
local ScriptTab = Window:CreateTab("change the script ", 4483362458)

local Toggle = ScriptTab:CreateToggle({
Name = "Dtoan VIP",
CurrentValue = false,
Flag = "Toggle1",
Callback = function(Value)
if Value then
loadstring(game:HttpGet("https://raw.githubusercontent.com/duytoan429/vip/refs/heads/main/by%20dtoan"))()
end
end,
})

-- ===== END OF SCRIPT =====
