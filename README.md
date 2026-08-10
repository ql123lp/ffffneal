--- Services ---
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

--- ppp ---
local AttackOffset = 2.5       -- 攻击偏移距离
local AttackInterval = 0.12     -- 攻击间隔时间(秒)
local AutoBananaActive = false
local AutoBananaThread = nil
local HasEquippedBanana = false

--- qqq ---
local function findBananaGun()
    local char = LocalPlayer.Character
    if char then
        for _, tool in pairs(char:GetChildren()) do
            if tool:IsA("Tool") and string.find(string.lower(tool.Name), "banana gun") then
                return tool
            end
        end
    end
    for _, tool in pairs(LocalPlayer.Backpack:GetChildren()) do
        if tool:IsA("Tool") and string.find(string.lower(tool.Name), "banana gun") then
            return tool
        end
    end
    return nil
end

--- jjj ---
local function getTargetRoot()
    local entitiesFolder = Workspace:FindFirstChild("Entities") or Workspace
    for _, ent in ipairs(entitiesFolder:GetChildren()) do
        if ent:IsA("Model") and ent:FindFirstChild("Humanoid") then
            local isPlayer = Players:GetPlayerFromCharacter(ent)
            if not isPlayer and ent:FindFirstChild("HumanoidRootPart") then
                local hum = ent.Humanoid
                if hum.Health > 0 then
                    return ent.HumanoidRootPart
                end
            end
        end
    end
    return nil
end

--- jjj ---
local function getTargetDirection(targetRoot)
    if not targetRoot then return Vector3.new(0, 0, -1) end
    local velocity = targetRoot.AssemblyLinearVelocity
    local horizontalVelocity = Vector3.new(velocity.X, 0, velocity.Z)
    if horizontalVelocity.Magnitude > 0.5 then
        return horizontalVelocity.Unit
    else
        return targetRoot.CFrame.LookVector.Unit
    end
end

--- qqq ---
local function bananaGunShoot()
    local targetRoot = getTargetRoot()
    if not targetRoot then return end

    local tool = findBananaGun()
    if not tool then return end
    local char = LocalPlayer.Character

    -- 刷新
    if not HasEquippedBanana then
        if char and tool.Parent ~= char then
            local hum = char:FindFirstChild("Humanoid")
            if hum then
                hum:EquipTool(tool)
                task.wait(0.3)
                hum:UnequipTools()
            end
        end
        HasEquippedBanana = true
    end

    local moveDir = getTargetDirection(targetRoot)
    local shootPos = targetRoot.Position + (moveDir * AttackOffset)
    local cf = CFrame.new(shootPos, shootPos + moveDir)

    local remote = tool:FindFirstChild("RemoteEvent")
    if remote then pcall(function() remote:FireServer(cf) end) end

    local really = tool:FindFirstChild("REALLY")
    if really then pcall(function() really:FireServer(cf) end) end

    local ability = tool:FindFirstChild("Ability")
    if ability then
        local airstrikeRemote = ability:FindFirstChild("Airstrike")
        if airstrikeRemote then
            local prePos = targetRoot.Position + (moveDir * (AttackOffset + 10))
            local pos = Vector3.new(prePos.X, prePos.Y - 1.5, prePos.Z)
            pcall(function() airstrikeRemote:FireServer("AirstrikeCD", "TAP", pos) end)
        end
    end
end

--- UI 界面生成 ---
local BananaGui = Instance.new("ScreenGui")
BananaGui.Name = "BananaGunAuto_Gui"
BananaGui.ResetOnSpawn = false

pcall(function() BananaGui.Parent = CoreGui end)
if not BananaGui.Parent then BananaGui.Parent = PlayerGui end

-- 独立按钮
local AttackBtn = Instance.new("TextButton")
AttackBtn.Size = UDim2.new(0, 160, 0, 45)
AttackBtn.Position = UDim2.new(0.8, 0, 0.5, 0)
AttackBtn.Text = "🍌 : 关"
AttackBtn.Font = Enum.Font.GothamBold
AttackBtn.TextSize = 13
AttackBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
AttackBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
AttackBtn.BorderSizePixel = 0
AttackBtn.Parent = BananaGui

local BtnCorner = Instance.new("UICorner")
BtnCorner.CornerRadius = UDim.new(0, 8)
BtnCorner.Parent = AttackBtn

local BtnStroke = Instance.new("UIStroke")
BtnStroke.Thickness = 2
BtnStroke.Color = Color3.fromRGB(255, 200, 0)
BtnStroke.Parent = AttackBtn

--- 拖拽逻辑 ---
local dragging, dragStart, startPos
AttackBtn.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = AttackBtn.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        AttackBtn.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + delta.X,
            startPos.Y.Scale, startPos.Y.Offset + delta.Y
        )
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

--- 按钮点击切换 ---
AttackBtn.MouseButton1Click:Connect(function()
    AutoBananaActive = not AutoBananaActive

    if AutoBananaActive then
        AttackBtn.Text = "🍌 : 开"
        AttackBtn.BackgroundColor3 = Color3.fromRGB(0, 180, 100)
        HasEquippedBanana = false

        if AutoBananaThread then task.cancel(AutoBananaThread) end
        AutoBananaThread = task.spawn(function()
            while AutoBananaActive do
                bananaGunShoot()
                task.wait(AttackInterval)
            end
        end)
    else
        AttackBtn.Text = "🍌 : 关"
        AttackBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
        if AutoBananaThread then
            task.cancel(AutoBananaThread)
            AutoBananaThread = nil
        end
    end
end)

-- 角色死亡复位状态
LocalPlayer.CharacterAdded:Connect(function()
    HasEquippedBanana = false
    if AutoBananaActive then
        if AutoBananaThread then task.cancel(AutoBananaThread) end
        AutoBananaThread = task.spawn(function()
            while AutoBananaActive do
                bananaGunShoot()
                task.wait(AttackInterval)
            end
        end)
    end
end)