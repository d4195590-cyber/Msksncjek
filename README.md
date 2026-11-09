if not game:IsLoaded() then
    game.Loaded:Wait()
end

local okUi, ui = pcall(function()
    return loadstring(game:HttpGet("https://pastebin.com/raw/YDjtnjyE"))()
end)
if not okUi or not ui then
    warn("[Egg] Не удалось загрузить UI с pastebin")
    return
end

local plrs = game:GetService("Players")
local ws = game:GetService("Workspace")
local rs = game:GetService("RunService")
local lp = plrs.LocalPlayer

local on = false
local quickOn = false
local taskId = nil
local stepConn = nil
local childAddedConn = nil
local promptAddedConn = nil

local eggFolder = nil
local knownEggs = {}
local knownPrompts = {}
local currentModel = nil
local moveTarget = nil
local moveSpeed = 16
local isGoingToEgg = false

local lvAttachment = nil
local linearVel = nil

local holdDuration = 5
local checkInterval = 0.3
local arriveDist = 3
local playerRadius = 13
local collectPoint = Vector3.new(8.7, 3.0, -13.7)
local heightOffset = 2
local promptTimeout = 10
local promptRadius = 20
local holdInterval = 0.070

local quickConn = nil
local quickTriggerConns = {}
local quickCooldown = 0
local quickCooldownTime = 3
local quickRiseHeight = 13

local playerCheckCache = {}
local playerCheckTime = 0

local function getHrp()
    local c = lp.Character
    if not c then return nil end
    return c:FindFirstChild("HumanoidRootPart")
end

local function getHum()
    local c = lp.Character
    if not c then return nil end
    return c:FindFirstChildOfClass("Humanoid")
end

local function getEggFolder()
    if eggFolder and eggFolder.Parent then return eggFolder end
    eggFolder = ws:FindFirstChild("WorldEggs")
    return eggFolder
end

local function computePlayerNear()
    local result = {}
    for _, plr in ipairs(plrs:GetPlayers()) do
        if plr ~= lp and plr.Character and plr.Character ~= lp.Character then
            local pPart = plr.Character:FindFirstChild("HumanoidRootPart") or plr.Character:FindFirstChildWhichIsA("BasePart", true)
            if pPart then
                table.insert(result, pPart.Position)
            end
        end
    end
    return result
end

local function hasPlayerNear(model, radius)
    if os.clock() - playerCheckTime > 0.3 then
        playerCheckCache = computePlayerNear()
        playerCheckTime = os.clock()
    end
    local modelPart = model.PrimaryPart or model:FindFirstChildWhichIsA("BasePart", true)
    if not modelPart then return false end
    for _, pos in ipairs(playerCheckCache) do
        if (pos - modelPart.Position).Magnitude <= radius then
            return true
        end
    end
    return false
end

local function getPromptPos(obj)
    local par = obj.Parent
    if par and par:IsA("BasePart") then return par.Position end
    if par and par:IsA("Attachment") and par.Parent and par.Parent:IsA("BasePart") then return par.Parent.Position end
    if par and par:IsA("Model") then
        local pp = par:FindFirstChildWhichIsA("BasePart", true)
        if pp then return pp.Position end
    end
    local ok, wp = pcall(function() return obj:GetWorldPosition() end)
    if ok then return wp end
    return nil
end

local function findNearestPrompt(pos, radius, eggModel)
    local best = nil
    local bestD = math.huge

    if eggModel then
        for _, obj in ipairs(eggModel:GetDescendants()) do
            if obj:IsA("ProximityPrompt") then
                local wp = getPromptPos(obj)
                if wp then
                    local d = (wp - pos).Magnitude
                    if d < bestD then
                        bestD = d
                        best = obj
                    end
                end
            end
        end
    end

    if best then return best, bestD end

    local toRemove = {}
    for prompt in pairs(knownPrompts) do
        if prompt and prompt.Parent then
            local wp = getPromptPos(prompt)
            if wp then
                local d = (wp - pos).Magnitude
                if d < bestD and d <= radius then
                    bestD = d
                    best = prompt
                end
            end
        else
            table.insert(toRemove, prompt)
        end
    end
    for _, p in ipairs(toRemove) do
        knownPrompts[p] = nil
    end
    return best, bestD
end

local function destroyLinearVel()
    if linearVel then
        pcall(function() linearVel:Destroy() end)
        linearVel = nil
    end
    if lvAttachment then
        pcall(function() lvAttachment:Destroy() end)
        lvAttachment = nil
    end
end

local function createLinearVel()
    local hrp = getHrp()
    if not hrp then return end
    if linearVel and linearVel.Parent then return end
    destroyLinearVel()
    lvAttachment = Instance.new("Attachment")
    lvAttachment.Parent = hrp

    linearVel = Instance.new("LinearVelocity")
    linearVel.Attachment0 = lvAttachment
    linearVel.MaxForce = math.huge
    linearVel.VelocityConstraintMode = Enum.VelocityConstraintMode.Vector
    linearVel.Parent = hrp
end

local function movementActive()
    return on or quickOn
end

local function onStepped(time, dt)
    if not movementActive() or not moveTarget then return end
    local hrp = getHrp()
    if not hrp then return end

    if isGoingToEgg and currentModel then
        local valid = currentModel.Parent ~= nil
        if valid then
            local folder = getEggFolder()
            if not folder or not currentModel:IsDescendantOf(folder) then
                valid = false
            end
        end
        if valid then
            local eggPart = currentModel.PrimaryPart or currentModel:FindFirstChildWhichIsA("BasePart", true)
            if not eggPart then
                valid = false
            else
                if not hasPlayerNear(currentModel, playerRadius) then
                    valid = false
                end
            end
        end
        if not valid then
            moveTarget = nil
            isGoingToEgg = false
            currentModel = nil
            if linearVel then
                pcall(function() linearVel.VectorVelocity = Vector3.zero end)
            end
            hrp.AssemblyLinearVelocity = Vector3.zero
            destroyLinearVel()
            return
        end
    end

    local targetPos = moveTarget + Vector3.new(0, heightOffset, 0)
    local delta = targetPos - hrp.Position
    local d = delta.Magnitude
    if d < arriveDist then
        moveTarget = nil
        if isGoingToEgg then
            isGoingToEgg = false
            if linearVel then
                pcall(function() linearVel.VectorVelocity = Vector3.zero end)
            end
            hrp.AssemblyLinearVelocity = Vector3.zero
        else
            if linearVel then
                pcall(function() linearVel.VectorVelocity = Vector3.zero end)
            end
            hrp.AssemblyLinearVelocity = Vector3.zero
            destroyLinearVel()
        end
        return
    end

    local dir = delta.Unit * moveSpeed
    if linearVel then
        linearVel.VectorVelocity = dir
    else
        hrp.AssemblyLinearVelocity = dir
    end
end

local function startMove(target, speed, goingToEgg)
    isGoingToEgg = goingToEgg and true or false
    moveTarget = target
    moveSpeed = speed or 16
    createLinearVel()
end

local function waitMoveDone()
    local start = os.clock()
    while movementActive() and moveTarget do
        if os.clock() - start > 30 then
            moveTarget = nil
            break
        end
        task.wait(0.05)
    end
    return movementActive()
end

local function bindEggFolder(folder)
    knownEggs = {}
    for _, obj in ipairs(folder:GetDescendants()) do
        if obj:IsA("Model") then
            knownEggs[obj] = true
        end
    end

    if childAddedConn then
        pcall(function() childAddedConn:Disconnect() end)
        childAddedConn = nil
    end

    childAddedConn = folder.ChildAdded:Connect(function(obj)
        if obj:IsA("Model") then
            knownEggs[obj] = true
        end
    end)
end

local function bindPrompts()
    knownPrompts = {}
    for _, obj in ipairs(ws:GetDescendants()) do
        if obj:IsA("ProximityPrompt") then
            knownPrompts[obj] = true
        end
    end
    if promptAddedConn then
        pcall(function() promptAddedConn:Disconnect() end)
        promptAddedConn = nil
    end
    promptAddedConn = ws.DescendantAdded:Connect(function(obj)
        if obj:IsA("ProximityPrompt") then
            knownPrompts[obj] = true
        end
    end)
end

local function waitForNewEgg(timeout)
    local folder = getEggFolder()
    if not folder then return nil end

    local startTime = os.clock()
    while on and (os.clock() - startTime) < timeout do
        if moveTarget then
            task.wait(0.1)
            continue
        end

        local hrp = getHrp()
        local myPos = hrp and hrp.Position
        if not myPos then
            task.wait(0.1)
            continue
        end

        local bestEgg = nil
        local bestDist = math.huge
        local toRemove = {}

        for egg in pairs(knownEggs) do
            if egg and egg.Parent and egg:IsA("Model") and egg:IsDescendantOf(folder) and egg ~= currentModel then
                if hasPlayerNear(egg, playerRadius) then
                    local eggPart = egg.PrimaryPart or egg:FindFirstChildWhichIsA("BasePart", true)
                    if eggPart then
                        local d = (eggPart.Position - myPos).Magnitude
                        if d < bestDist then
                            bestDist = d
                            bestEgg = egg
                        end
                    end
                end
            else
                table.insert(toRemove, egg)
            end
        end

        for _, egg in ipairs(toRemove) do
            knownEggs[egg] = nil
        end

        if bestEgg then
            return bestEgg
        end

        task.wait(checkInterval)
    end
    return nil
end

local function run()
    while on do
        local folder = getEggFolder()
        if not folder then
            task.wait(1)
            continue
        end

        local egg = waitForNewEgg(60)
        if not egg then
            task.wait(0.3)
            continue
        end

        currentModel = egg
        local eggPart = egg.PrimaryPart or egg:FindFirstChildWhichIsA("BasePart", true)
        if not eggPart then
            currentModel = nil
            task.wait(0.3)
            continue
        end

        local hum = getHum()
        local speed = hum and hum.WalkSpeed or 16
        if speed <= 0 then speed = 16 end

        startMove(eggPart.Position, speed, true)
        if not waitMoveDone() then break end

        if not currentModel or not currentModel.Parent then
            currentModel = nil
            destroyLinearVel()
            task.wait(0.2)
            continue
        end

        task.wait(0.1)

        local promptStart = os.clock()
        local prompt = nil
        while on and (os.clock() - promptStart) < promptTimeout do
            local hrp = getHrp()
            if not hrp then break end
            local p = findNearestPrompt(hrp.Position, promptRadius, currentModel)
            if p then
                prompt = p
                break
            else
                task.wait(0.1)
            end
        end

        local success = false
        if prompt then
            local myHoldActive = true
            local triggered = false

            local conn
            conn = prompt.Triggered:Connect(function(player)
                if not myHoldActive then return end
                if player ~= lp then return end
                if not prompt or not prompt.Parent then return end

                local pos = getPromptPos(prompt)
                local hrp = getHrp()
                if pos and hrp then
                    local maxDist = prompt.MaxActivationDistance or 10
                    if (pos - hrp.Position).Magnitude > (maxDist + 5) then
                        return
                    end
                end

                if currentModel and not currentModel:IsDescendantOf(getEggFolder()) then
                    return
                end

                triggered = true
            end)

            local timeout = os.clock() + holdDuration
            while on and not triggered and os.clock() < timeout do
                if prompt and prompt.Parent then
                    pcall(function() prompt:InputHoldBegin() end)
                end
                task.wait(holdInterval)
            end

            if prompt and prompt.Parent then
                pcall(function() prompt:InputHoldEnd() end)
            end

            myHoldActive = false
            if conn then
                pcall(function() conn:Disconnect() end)
            end
            success = triggered
        end

        if success then
            local hrp = getHrp()
            if hrp and on then
                local upTarget = hrp.Position + Vector3.new(0, 13, 0)
                startMove(upTarget, speed, false)
                if not waitMoveDone() then break end
            end

            startMove(collectPoint, speed, false)
            if not waitMoveDone() then break end
            task.wait(0.5)
        else
            destroyLinearVel()
            task.wait(0.2)
        end

        currentModel = nil
        task.wait(0.3)
    end
end

local function stopQuick()
    quickOn = false
    if quickConn then
        pcall(function() quickConn:Disconnect() end)
        quickConn = nil
    end
    for conn in pairs(quickTriggerConns) do
        pcall(function() conn:Disconnect() end)
        quickTriggerConns[conn] = nil
    end
end

local function onQuickTriggered(prompt, player)
    if not quickOn then return end
    if player ~= lp then return end
    if not prompt or not prompt.Parent then return end
    if os.clock() < quickCooldown then return end

    local pos = getPromptPos(prompt)
    local hrp = getHrp()
    if not pos or not hrp then return end
    local maxDist = prompt.MaxActivationDistance or 10
    if (pos - hrp.Position).Magnitude > (maxDist + 5) then return end

    quickCooldown = os.clock() + quickCooldownTime

    local hum = getHum()
    local speed = hum and hum.WalkSpeed or 16
    if speed <= 0 then speed = 16 end

    moveTarget = nil
    isGoingToEgg = false
    currentModel = nil
    if linearVel then
        pcall(function() linearVel.VectorVelocity = Vector3.zero end)
    end
    destroyLinearVel()

    task.spawn(function()
        if not quickOn then return end
        local hrp2 = getHrp()
        if not hrp2 then return end

        startMove(hrp2.Position + Vector3.new(0, quickRiseHeight, 0), speed, false)
        if not waitMoveDone() then return end
        if not quickOn then return end

        startMove(collectPoint, speed, false)
        waitMoveDone()
    end)
end

local function subscribePrompt(prompt)
    if not prompt or not prompt.Parent then return end
    if quickTriggerConns[prompt] then return end
    local conn = prompt.Triggered:Connect(function(player)
        onQuickTriggered(prompt, player)
    end)
    quickTriggerConns[prompt] = conn
end

local function startQuick()
    stopQuick()
    quickOn = true
    quickConn = ws.DescendantAdded:Connect(function(obj)
        if obj:IsA("ProximityPrompt") then
            subscribePrompt(obj)
        end
    end)
    for _, obj in ipairs(ws:GetDescendants()) do
        if obj:IsA("ProximityPrompt") then
            subscribePrompt(obj)
        end
    end
    if not stepConn then
        stepConn = rs.Stepped:Connect(function(time, dt)
            if movementActive() then
                local ok2, err2 = pcall(onStepped, time, dt)
                if not ok2 then
                    warn("[Egg] onStepped error:", err2)
                    moveTarget = nil
                    isGoingToEgg = false
                end
            end
        end)
    end
end

local function stop()
    on = false
    moveTarget = nil
    moveSpeed = 16
    isGoingToEgg = false
    currentModel = nil
    knownEggs = {}
    knownPrompts = {}
    playerCheckCache = {}
    playerCheckTime = 0
    destroyLinearVel()
    if childAddedConn then
        pcall(function() childAddedConn:Disconnect() end)
        childAddedConn = nil
    end
    if promptAddedConn then
        pcall(function() promptAddedConn:Disconnect() end)
        promptAddedConn = nil
    end
    if taskId then
        pcall(function() task.cancel(taskId) end)
        taskId = nil
    end
    if not quickOn and stepConn then
        pcall(function() stepConn:Disconnect() end)
        stepConn = nil
    end
    local hrp = getHrp()
    if hrp then
        pcall(function() hrp.AssemblyLinearVelocity = Vector3.zero end)
    end
end

local function start()
    stop()
    if on then return true end
    local hrp = getHrp()
    if not hrp then return false, "Нет HRP" end
    local folder = getEggFolder()
    if not folder then return false, "Нет WorldEggs" end

    local ok, err = pcall(function()
        bindEggFolder(folder)
        bindPrompts()
    end)
    if not ok then
        stop()
        return false, err
    end

    on = true
    if not stepConn then
        stepConn = rs.Stepped:Connect(function(time, dt)
            if movementActive() then
                local ok2, err2 = pcall(onStepped, time, dt)
                if not ok2 then
                    warn("[Egg] onStepped error:", err2)
                    moveTarget = nil
                    isGoingToEgg = false
                end
            end
        end)
    end
    taskId = task.spawn(run)
    return true
end

local win = ui:CreateWindow({ Title = "Egg Steal" })
local main = win:CreateTab("Main")

local sLbl = main:CreateLabel("Статус: OFF")
local function upS(t) sLbl:SetText(t) end

local tgl
tgl = main:CreateToggle("Кража яиц", false, function(v)
    if v then
        local ok, err = start()
        if ok then
            upS("Статус: ON")
            win:Notify("Egg", "Включён 🥚", "success", 3)
        else
            upS("Статус: OFF")
            win:Notify("Egg", tostring(err), "error", 5)
            if tgl then tgl:SetValue(false) end
        end
    else
        stop()
        upS("Статус: OFF")
        win:Notify("Egg", "Выключен 🛑", "warn", 3)
    end
end)

local tglQuick
tglQuick = main:CreateToggle("Добыть подобранное яйцо", false, function(v)
    if v then
        startQuick()
        win:Notify("Egg", "Быстрый подбор ON ⚡", "success", 3)
    else
        stopQuick()
        if not on and stepConn then
            pcall(function() stepConn:Disconnect() end)
            stepConn = nil
        end
        win:Notify("Egg", "Быстрый подбор OFF", "warn", 3)
    end
end)

win:Notify("Nyxen Hub", "Egg Steal загружен ✅", "success", 4)
