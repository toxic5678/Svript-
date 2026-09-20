--[[
    Savannah Life OP Hub v2 — Fixed
    Language: Luau (Roblox)
    Runtime: Executor
    Fixes: Inf Stats capped at 99%, auto-eat grass via touch events
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local VirtualUser = game:GetService("VirtualUser")
local GuiService = game:GetService("GuiService")

local player = Players.LocalPlayer

local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
    Name = "☀ SAVANNAH OP HUB v2",
    LoadingTitle = "Savannah Life OP",
    LoadingSubtitle = "fixed stats + auto eat",
    ConfigurationSaving = { Enabled = true, FolderName = "SavannahOP", FileName = "Config" },
    KeySystem = false,
    Theme = "Amethyst"
})

-- ============================================================
-- STATE
-- ============================================================
local state = {
    espEnabled = false,
    lockEnabled = false,
    flyEnabled = false,
    flySpeed = 80,
    infStamina = false,
    infFood = false,
    infWater = false,
    autoEat = false,
    espObjects = {},
    espConn = nil,
    flyConn = nil,
    lockConn = nil,
    autoEatConn = nil
}

-- ============================================================
-- CHARACTER HELPERS
-- ============================================================
local function getChar() return player.Character or player.CharacterAdded:Wait() end
local function getHum() local c = getChar(); return c and c:FindFirstChildOfClass("Humanoid") end
local function getRoot() local c = getChar(); return c and c:FindFirstChild("HumanoidRootPart") end

-- ============================================================
-- STAT FINDER (finds water/food/stamina values)
-- ============================================================
local function findStatValue(keywordList)
    -- search in PlayerGui first (Savannah shows water as text)
    local pg = player:FindFirstChild("PlayerGui")
    if pg then
        for _, obj in ipairs(pg:GetDescendants()) do
            if obj:IsA("ValueBase") then
                local n = obj.Name:lower()
                for _, k in ipairs(keywordList) do
                    if n:find(k) then return obj end
                end
            end
        end
    end
    -- search in Player
    for _, obj in ipairs(player:GetDescendants()) do
        if obj:IsA("ValueBase") then
            local n = obj.Name:lower()
            for _, k in ipairs(keywordList) do
                if n:find(k) then return obj end
            end
        end
    end
    -- search in Character
    local char = player.Character
    if char then
        for _, obj in ipairs(char:GetDescendants()) do
            if obj:IsA("ValueBase") then
                local n = obj.Name:lower()
                for _, k in ipairs(keywordList) do
                    if n:find(k) then return obj end
                end
            end
        end
    end
    -- search whole game (fallback)
    for _, obj in ipairs(game:GetDescendants()) do
        if obj:IsA("ValueBase") then
            local n = obj.Name:lower()
            for _, k in ipairs(keywordList) do
                if n:find(k) then return obj end
            end
        end
    end
    return nil
end

-- Find stat values once and cache
local statCache = {
    water = nil,
    food = nil,
    stamina = nil
}

local function getWaterStat()
    if statCache.water and statCache.water.Parent then return statCache.water end
    statCache.water = findStatValue({"water", "thirst", "hydration"})
    return statCache.water
end

local function getFoodStat()
    if statCache.food and statCache.food.Parent then return statCache.food end
    statCache.food = findStatValue({"food", "hunger", "satiety"})
    return statCache.food
end

local function getStaminaStat()
    if statCache.stamina and statCache.stamina.Parent then return statCache.stamina end
    statCache.stamina = findStatValue({"stamina", "energy", "endurance"})
    return statCache.stamina
end

-- ============================================================
-- INFINITE STATS — CAPPED AT 99%
-- ============================================================
local function keepStatAt99(getter)
    return task.spawn(function()
        while true do
            local flag = state.infWater or state.infFood or state.infStamina
            if not flag then task.wait(0.3); continue end

            local v = getter()
            if v then
                pcall(function()
                    if v:IsA("NumberValue") or v:IsA("IntValue") then
                        if v.Value < 99 then
                            v.Value = 99
                        elseif v.Value > 99 then
                            v.Value = 99
                        end
                    end
                end)
            end
            task.wait(0.4)
        end
    end)
end

-- Start three keepers
keepStatAt99(function() return state.infWater and getWaterStat() or nil end)
keepStatAt99(function() return state.infFood and getFoodStat() or nil end)
keepStatAt99(function() return state.infStamina and getStaminaStat() or nil end)

-- ============================================================
-- AUTO EAT GRASS — simulate taps on "Tap to eat"
-- ============================================================
-- Savannah Life uses a "Tap to eat" UI button that appears when
-- standing on grass. We detect it in PlayerGui and fire the same
-- touch/click event the game listens to.
-- ============================================================

local function findEatButton()
    local pg = player:FindFirstChild("PlayerGui")
    if not pg then return nil end

    for _, obj in ipairs(pg:GetDescendants()) do
        if obj:IsA("TextButton") or obj:IsA("ImageButton") then
            local txt = (obj.Text or ""):lower()
            local nm = obj.Name:lower()
            if txt:find("tap to eat") or txt:find("eat") or 
               nm:find("eat") or nm:find("consume") then
                return obj
            end
        elseif obj:IsA("TextLabel") then
            local txt = (obj.Text or ""):lower()
            if txt:find("tap to eat") then
                -- label likely sits on top of a button — look at parent
                local parent = obj.Parent
                if parent and (parent:IsA("TextButton") or parent:IsA("ImageButton")) then
                    return parent
                end
                -- or find sibling button
                if parent then
                    for _, sib in ipairs(parent:GetChildren()) do
                        if sib:IsA("TextButton") or sib:IsA("ImageButton") then
                            return sib
                        end
                    end
                end
            end
        end
    end
    return nil
end

local function fireButtonClick(button)
    if not button then return false end

    -- Try each input method — one of these will match what the game listens for
    local methods = {
        function() button:Activate() end,
        function()
            button.MouseButton1Click:Fire()
        end,
        function()
            button.MouseButton1Down:Fire()
            task.wait(0.05)
            button.MouseButton1Up:Fire()
        end,
    }

    for _, m in ipairs(methods) do
        local ok = pcall(m)
        if ok then return true end
    end

    -- Touch simulation (mobile-style games listen to TouchTap)
    local ok = pcall(function()
        local vu = game:GetService("VirtualUser")
        vu:Button1Down(Vector2.new(button.AbsolutePosition.X + 10, button.AbsolutePosition.Y + 10))
        task.wait(0.05)
        vu:Button1Up(Vector2.new(button.AbsolutePosition.X + 10, button.AbsolutePosition.Y + 10))
    end)
    if ok then return true end

    -- Last resort: fire InputBegan/Ended on the button (mobile touch)
    pcall(function()
        local input = Instance.new("InputObject")
        -- cannot create InputObject directly; use VirtualInputManager instead
    end)

    return false
end

-- Alternate auto-eat: find all grass parts near player and touch them
local function findGrassNearby()
    local root = getRoot()
    if not root then return nil end

    local closest = nil
    local closestDist = 30 -- studs

    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") then
            local n = obj.Name:lower()
            if n:find("grass") or n:find("plant") or n:find("food") or 
               n:find("bush") or n:find("leaf") or n:find("herb") then
                local dist = (obj.Position - root.Position).Magnitude
                if dist < closestDist then
                    closestDist = dist
                    closest = obj
                end
            end
        end
    end
    return closest
end

local function autoEatLoop()
    return task.spawn(function()
        while state.autoEat do
            -- 1. try clicking the eat button
            local btn = findEatButton()
            if btn and btn.Visible then
                fireButtonClick(btn)
                task.wait(0.8)
            else
                -- 2. no button visible — walk toward nearest grass
                local grass = findGrassNearby()
                local hum = getHum()
                if grass and hum then
                    hum:MoveTo(grass.Position)
                    task.wait(0.6)
                else
                    task.wait(0.4)
                end
            end

            -- Refresh food value
            local food = getFoodStat()
            if food and food.Value >= 99 then
                -- fully fed — pause briefly
                task.wait(1)
            end
        end
    end)
end

-- ============================================================
-- ESP
-- ============================================================
local function createESP(targetPlayer)
    if not targetPlayer.Character or targetPlayer == player then return end
    local char = targetPlayer.Character
    local head = char:FindFirstChild("Head")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not head or not hum then return end

    if state.espObjects[targetPlayer] then state.espObjects[targetPlayer]:Destroy() end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "SavannahESP"
    billboard.Adornee = head
    billboard.Size = UDim2.new(0, 200, 0, 60)
    billboard.StudsOffset = Vector3.new(0, 3, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = head

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, 0, 0.5, 0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = targetPlayer.Name
    nameLabel.TextColor3 = Color3.fromRGB(255, 200, 50)
    nameLabel.TextStrokeTransparency = 0
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 14
    nameLabel.Parent = billboard

    local hpLabel = Instance.new("TextLabel")
    hpLabel.Size = UDim2.new(1, 0, 0.5, 0)
    hpLabel.Position = UDim2.new(0, 0, 0.5, 0)
    hpLabel.BackgroundTransparency = 1
    hpLabel.Text = "HP: " .. math.floor(hum.Health)
    hpLabel.TextColor3 = Color3.fromRGB(100, 255, 100)
    hpLabel.TextStrokeTransparency = 0
    hpLabel.Font = Enum.Font.Gotham
    hpLabel.TextSize = 12
    hpLabel.Parent = billboard

    hum.HealthChanged:Connect(function(hp)
        if hpLabel and hpLabel.Parent then
            hpLabel.Text = "HP: " .. math.floor(hp)
        end
    end)

    state.espObjects[targetPlayer] = billboard
end

local function removeAllESP()
    for _, b in pairs(state.espObjects) do if b then b:Destroy() end end
    state.espObjects = {}
end

-- ============================================================
-- FLY
-- ============================================================
local function startFly()
    local root, hum = getRoot(), getHum()
    if not root or not hum then return end
    hum.PlatformStand = true

    local bv = Instance.new("BodyVelocity")
    bv.Name = "SavFly"
    bv.MaxForce = Vector3.new(1e6, 1e6, 1e6)
    bv.Velocity = Vector3.zero
    bv.Parent = root

    local bg = Instance.new("BodyGyro")
    bg.Name = "SavFlyGyro"
    bg.MaxTorque = Vector3.new(1e6, 1e6, 1e6)
    bg.P = 9e4
    bg.Parent = root

    state.flyConn = RunService.Heartbeat:Connect(function()
        if not state.flyEnabled then return end
        local cam = workspace.CurrentCamera
        local dir = Vector3.zero
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then dir += cam.CFrame.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then dir -= cam.CFrame.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then dir -= cam.CFrame.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then dir += cam.CFrame.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir += Vector3.new(0,1,0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then dir -= Vector3.new(0,1,0) end
        bv.Velocity = dir.Magnitude > 0 and dir.Unit * state.flySpeed or Vector3.zero
        bg.CFrame = cam.CFrame
    end)
end

local function stopFly()
    state.flyEnabled = false
    local root = getRoot()
    if root then
        local bv = root:FindFirstChild("SavFly"); if bv then bv:Destroy() end
        local bg = root:FindFirstChild("SavFlyGyro"); if bg then bg:Destroy() end
    end
    local hum = getHum(); if hum then hum.PlatformStand = false end
    if state.flyConn then state.flyConn:Disconnect(); state.flyConn = nil end
end

-- ============================================================
-- LOCK
-- ============================================================
local function getNearestPlayer()
    local myRoot = getRoot()
    if not myRoot then return nil end
    local nearest, shortest = nil, math.huge
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player and p.Character then
            local r = p.Character:FindFirstChild("HumanoidRootPart")
            local h = p.Character:FindFirstChildOfClass("Humanoid")
            if r and h and h.Health > 0 then
                local d = (r.Position - myRoot.Position).Magnitude
                if d < shortest and d < 500 then shortest = d; nearest = p end
            end
        end
    end
    return nearest
end

-- ============================================================
-- UI
-- ============================================================
local MainTab = Window:CreateTab("Main", 4483362458)
local VisualTab = Window:CreateTab("Visual", 4483362458)
local StatsTab = Window:CreateTab("Stats", 4483362458)

-- MAIN
MainTab:CreateSection("Combat")
MainTab:CreateToggle({
    Name = "Lock Nearest Player",
    CurrentValue = false,
    Flag = "LockTarget",
    Callback = function(v)
        state.lockEnabled = v
        if v then
            state.lockConn = RunService.RenderStepped:Connect(function()
                if state.lockEnabled then
                    local t = getNearestPlayer()
                    if t and t.Character then
                        local head = t.Character:FindFirstChild("Head")
                        if head then
                            workspace.CurrentCamera.CFrame = CFrame.new(
                                workspace.CurrentCamera.CFrame.Position, head.Position)
                        end
                    end
                end
            end)
        else
            if state.lockConn then state.lockConn:Disconnect(); state.lockConn = nil end
        end
    end
})

MainTab:CreateSection("Movement")
MainTab:CreateToggle({
    Name = "Fly",
    CurrentValue = false,
    Flag = "FlyToggle",
    Callback = function(v)
        state.flyEnabled = v
        if v then startFly() else stopFly() end
    end
})
MainTab:CreateSlider({
    Name = "Fly Speed",
    Range = {20, 500}, Increment = 10, CurrentValue = 80, Flag = "FlySpeed",
    Callback = function(v) state.flySpeed = v end
})
MainTab:CreateSlider({
    Name = "WalkSpeed",
    Range = {16, 300}, Increment = 1, CurrentValue = 16, Flag = "WalkSpeed",
    Callback = function(v) local h = getHum(); if h then h.WalkSpeed = v end end
})
MainTab:CreateSlider({
    Name = "JumpPower",
    Range = {50, 300}, Increment = 1, CurrentValue = 50, Flag = "JumpPower",
    Callback = function(v) local h = getHum(); if h then h.JumpPower = v end end
})

-- VISUAL
VisualTab:CreateSection("ESP")
VisualTab:CreateToggle({
    Name = "Player ESP",
    CurrentValue = false,
    Flag = "ESPEnabled",
    Callback = function(v)
        state.espEnabled = v
        if v then
            for _, p in ipairs(Players:GetPlayers()) do
                if p ~= player then createESP(p) end
            end
            state.espConn = RunService.RenderStepped:Connect(function()
                if state.espEnabled then
                    for _, p in ipairs(Players:GetPlayers()) do
                        if p ~= player and not state.espObjects[p] then createESP(p) end
                    end
                end
            end)
        else
            if state.espConn then state.espConn:Disconnect(); state.espConn = nil end
            removeAllESP()
        end
    end
})

VisualTab:CreateSection("Lighting")
VisualTab:CreateToggle({
    Name = "Full Bright",
    CurrentValue = false,
    Flag = "FullBright",
    Callback = function(v)
        local L = game:GetService("Lighting")
        if v then
            L.Brightness = 2; L.ClockTime = 14; L.FogEnd = 1e5
            L.GlobalShadows = false; L.OutdoorAmbient = Color3.fromRGB(128,128,128)
        else
            L.Brightness = 1; L.ClockTime = 8; L.FogEnd = 1e5
            L.GlobalShadows = true; L.OutdoorAmbient = Color3.fromRGB(70,70,70)
        end
    end
})

-- STATS
StatsTab:CreateSection("Infinite Stats (capped at 99%)")
StatsTab:CreateToggle({
    Name = "Infinite Water (99%)",
    CurrentValue = false,
    Flag = "InfWater",
    Callback = function(v)
        state.infWater = v
        if v then
            local w = getWaterStat()
            Rayfield:Notify({
                Title = "Inf Water",
                Content = w and ("Found: " .. w:GetFullName()) or "Water value not found client-side",
                Duration = 4
            })
        end
    end
})
StatsTab:CreateToggle({
    Name = "Infinite Food (99%)",
    CurrentValue = false,
    Flag = "InfFood",
    Callback = function(v)
        state.infFood = v
        if v then
            local f = getFoodStat()
            Rayfield:Notify({
                Title = "Inf Food",
                Content = f and ("Found: " .. f:GetFullName()) or "Food value not found client-side",
                Duration = 4
            })
        end
    end
})
StatsTab:CreateToggle({
    Name = "Infinite Stamina (99%)",
    CurrentValue = false,
    Flag = "InfStamina",
    Callback = function(v)
        state.infStamina = v
        if v then
            local s = getStaminaStat()
            Rayfield:Notify({
                Title = "Inf Stamina",
                Content = s and ("Found: " .. s:GetFullName()) or "Stamina value not found client-side",
                Duration = 4
            })
        end
    end
})

StatsTab:CreateSection("Auto Eat")
StatsTab:CreateToggle({
    Name = "Auto Eat Grass",
    CurrentValue = false,
    Flag = "AutoEat",
    Callback = function(v)
        state.autoEat = v
        if v then
            autoEatLoop()
            Rayfield:Notify({
                Title = "Auto Eat",
                Content = "Hunting for 'Tap to eat' button...",
                Duration = 3
            })
        end
    end
})

StatsTab:CreateButton({
    Name = "Debug: Find Eat Button + Stats",
    Callback = function()
        local btn = findEatButton()
        local w = getWaterStat()
        local f = getFoodStat()
        local s = getStaminaStat()
        print("=== SAVANNAH DEBUG ===")
        print("Eat button:", btn and btn:GetFullName() or "NOT FOUND")
        print("Water:", w and (w:GetFullName() .. " = " .. tostring(w.Value)) or "NOT FOUND")
        print("Food:", f and (f:GetFullName() .. " = " .. tostring(f.Value)) or "NOT FOUND")
        print("Stamina:", s and (s:GetFullName() .. " = " .. tostring(s.Value)) or "NOT FOUND")
        Rayfield:Notify({
            Title = "Debug Printed",
            Content = "Check executor console (F9)",
            Duration = 5
        })
    end
})

-- ============================================================
-- PLAYER EVENTS
-- ============================================================
Players.PlayerRemoving:Connect(function(p)
    if state.espObjects[p] then state.espObjects[p]:Destroy(); state.espObjects[p] = nil end
end)

player.CharacterAdded:Connect(function()
    task.wait(1)
    if state.flyEnabled then stopFly() end
end)

Rayfield:Notify({
    Title = "☀ Savannah OP v2",
    Content = "Loaded. Stats capped at 99%. Auto-eat ready.",
    Duration = 5
})
