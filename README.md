--------------------------------------------------
-- LACERDA HUB V3.3
-- Discord: https://discord.gg/xkZh24RHYN
--------------------------------------------------

--------------------------------------------------
-- SISTEMA DE KEY POR NOME (WHITELIST)
--------------------------------------------------
local Players = game:GetService("Players")
local player = Players.LocalPlayer

local WHITELIST = {
    "LipeLovesJesus",
    "souzaaazr7",
    "tong_ton66",
    "hacjdk9",
    "exiled57",
    "gorilatortinho67",
    "Secundaria_614",
    "Roubeumbrainrot98798",
}

local autorizado = false
for _, nome in ipairs(WHITELIST) do
    if player.Name == nome then
        autorizado = true
        break
    end
end

if not autorizado then
    warn("❌ Acesso negado para: " .. player.Name)
    task.wait(1)
    player:Kick("❌ Você não está autorizado a usar este script.")
    return
end

print("✅ Acesso autorizado para:", player.Name)

--------------------------------------------------
-- SERVIÇOS
--------------------------------------------------
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local HttpService = game:GetService("HttpService")

--------------------------------------------------
-- NAMESPACE ÚNICO PARA AUTO GRAB
--------------------------------------------------
local AG_KEY = "LacerdaHub_AG_" .. player.UserId
_G[AG_KEY] = false

--------------------------------------------------
-- SISTEMA DE GERENCIAMENTO DE CONEXÕES
--------------------------------------------------
local ConnectionManager = {}
ConnectionManager.__index = ConnectionManager

function ConnectionManager.new()
    local self = setmetatable({}, ConnectionManager)
    self.connections = {}
    return self
end

function ConnectionManager:Add(name, connection)
    if self.connections[name] then
        pcall(function() self.connections[name]:Disconnect() end)
    end
    self.connections[name] = connection
end

function ConnectionManager:Remove(name)
    if self.connections[name] then
        pcall(function() self.connections[name]:Disconnect() end)
        self.connections[name] = nil
    end
end

function ConnectionManager:DisconnectAll()
    for _, conn in pairs(self.connections) do
        pcall(function() conn:Disconnect() end)
    end
    self.connections = {}
end

local globalConnections = ConnectionManager.new()

--------------------------------------------------
-- CACHE DE PERSONAGEM OTIMIZADO
--------------------------------------------------
local CharacterCache = {
    character = nil,
    hrp = nil,
    hum = nil,
    lastUpdate = 0,
    updateInterval = 0.5
}

local function UpdateCharacterCache(forceUpdate)
    local now = tick()
    if not forceUpdate and now - CharacterCache.lastUpdate < CharacterCache.updateInterval then
        return CharacterCache.character ~= nil
    end
    CharacterCache.lastUpdate = now
    CharacterCache.character = player.Character
    if CharacterCache.character then
        CharacterCache.hrp = CharacterCache.character:FindFirstChild("HumanoidRootPart")
        CharacterCache.hum = CharacterCache.character:FindFirstChild("Humanoid")
        return CharacterCache.hrp ~= nil and CharacterCache.hum ~= nil
    end
    CharacterCache.hrp = nil
    CharacterCache.hum = nil
    return false
end

player.CharacterAdded:Connect(function()
    task.wait(0.3)
    UpdateCharacterCache(true)
end)

--------------------------------------------------
-- CONFIGURAÇÕES GLOBAIS
--------------------------------------------------
local Config = {
    SpeedAutoActive = true,
    SPEED_NORMAL = 60,
    SPEED_STEAL = 30,
    Speed55Value = 60,
    Speed30Value = 30,
    NoAnimsEnabled = false,
    AutoBatEnabled = false,
    AntiRagdollEnabled = false,
    OptimizerXrayEnabled = false,
    AutoGrabEnabled = false,
    AutoGrabRange = 7, -- FIXO EM 7
    TrackerPlayerEnabled = false,
    GalaxyGravityPercent = 70,
    GalaxyEnabled = false,
    HopPower = 35,
    HopCooldown = 0.08,
}

local Values = {
    DEFAULT_GRAVITY = 196.2,
}

--------------------------------------------------
-- SISTEMA DE SALVAMENTO/CARREGAMENTO
--------------------------------------------------
local ConfigFileName = "lacerda_config.json"

local function SaveConfig()
    local data = {}
    for k, v in pairs(Config) do data[k] = v end
    for k, v in pairs(Values) do data[k] = v end

    local success = false
    if writefile then
        pcall(function()
            writefile(ConfigFileName, HttpService:JSONEncode(data))
            success = true
        end)
    end
    return success
end

local configLoaded = false
pcall(function()
    if readfile and isfile and isfile(ConfigFileName) then
        local data = HttpService:JSONDecode(readfile(ConfigFileName))
        if data then
            for k, v in pairs(data) do
                if Config[k] ~= nil then Config[k] = v end
            end
            for k, v in pairs(data) do
                if Values[k] ~= nil then Values[k] = v end
            end
            configLoaded = true
            print("✅ Configuração carregada")
        end
    end
end)

Config.AutoGrabRange = 7 -- garante fixo após load

--------------------------------------------------
-- SPEED AUTOMÁTICO
--------------------------------------------------
local function SetSpeedAutoEnabled(state)
    globalConnections:Remove("SpeedAuto")
    if state then
        globalConnections:Add("SpeedAuto", RunService.Heartbeat:Connect(function()
            if not UpdateCharacterCache() then return end
            local moveDir = CharacterCache.hum.MoveDirection
            if moveDir.Magnitude > 0.1 then
                local vel = (CharacterCache.hum.WalkSpeed < 25) and Config.SPEED_STEAL or Config.SPEED_NORMAL
                CharacterCache.hrp.AssemblyLinearVelocity = Vector3.new(
                    moveDir.X * vel,
                    CharacterCache.hrp.AssemblyLinearVelocity.Y,
                    moveDir.Z * vel
                )
            end
        end))
    end
    Config.SpeedAutoActive = state
    SaveConfig()
end

SetSpeedAutoEnabled(Config.SpeedAutoActive)

--------------------------------------------------
-- GALAXY MODE COM SUPORTE MOBILE
--------------------------------------------------
local galaxyVectorForce = nil
local galaxyAttachment   = nil
local galaxyEnabled      = false
local hopsEnabled        = false
local lastHopTime        = 0
local spaceHeld          = false
local originalJumpPower  = 50
local jumpRequestConn    = nil

local function captureOriginalJumpPower()
    local c = player.Character
    if not c then return end
    local hum = c:FindFirstChildOfClass("Humanoid")
    if hum and hum.JumpPower > 0 then
        originalJumpPower = hum.JumpPower
    end
end

task.spawn(function() task.wait(1); captureOriginalJumpPower() end)

local function setupGalaxyForce()
    pcall(function()
        local c = player.Character
        if not c then return end
        local h = c:FindFirstChild("HumanoidRootPart")
        if not h then return end
        if galaxyVectorForce then galaxyVectorForce:Destroy() end
        if galaxyAttachment then galaxyAttachment:Destroy() end
        galaxyAttachment = Instance.new("Attachment")
        galaxyAttachment.Parent = h
        galaxyVectorForce = Instance.new("VectorForce")
        galaxyVectorForce.Attachment0 = galaxyAttachment
        galaxyVectorForce.ApplyAtCenterOfMass = true
        galaxyVectorForce.RelativeTo = Enum.ActuatorRelativeTo.World
        galaxyVectorForce.Force = Vector3.new(0, 0, 0)
        galaxyVectorForce.Parent = h
    end)
end

local function updateGalaxyForce()
    if not galaxyEnabled or not galaxyVectorForce then return end
    local c = player.Character
    if not c then return end
    local mass = 0
    for _, p in ipairs(c:GetDescendants()) do
        if p:IsA("BasePart") then mass = mass + p:GetMass() end
    end
    local tg = Values.DEFAULT_GRAVITY * (Config.GalaxyGravityPercent / 100)
    galaxyVectorForce.Force = Vector3.new(0, mass * (Values.DEFAULT_GRAVITY - tg) * 0.95, 0)
end

local function adjustGalaxyJump()
    pcall(function()
        local c = player.Character
        if not c then return end
        local hum = c:FindFirstChildOfClass("Humanoid")
        if not hum then return end
        if not galaxyEnabled then
            hum.JumpPower = originalJumpPower
            return
        end
        local ratio = math.sqrt((Values.DEFAULT_GRAVITY * (Config.GalaxyGravityPercent / 100)) / Values.DEFAULT_GRAVITY)
        hum.JumpPower = originalJumpPower * ratio
    end)
end

local function doMiniHop()
    if not hopsEnabled then return end
    pcall(function()
        local c = player.Character
        if not c then return end
        local h = c:FindFirstChild("HumanoidRootPart")
        local hum = c:FindFirstChildOfClass("Humanoid")
        if not h or not hum then return end
        if tick() - lastHopTime < Config.HopCooldown then return end
        lastHopTime = tick()
        h.AssemblyLinearVelocity = Vector3.new(
            h.AssemblyLinearVelocity.X,
            Config.HopPower,
            h.AssemblyLinearVelocity.Z
        )
    end)
end

local function startGalaxy()
    galaxyEnabled = true
    hopsEnabled   = true
    setupGalaxyForce()
    adjustGalaxyJump()
    if jumpRequestConn then jumpRequestConn:Disconnect() end
    jumpRequestConn = UserInputService.JumpRequest:Connect(function()
        if hopsEnabled then
            spaceHeld = true
            doMiniHop()
            task.delay(0.15, function() spaceHeld = false end)
        end
    end)
    print("🌌 Galaxy ATIVADO")
end

local function stopGalaxy()
    galaxyEnabled = false
    hopsEnabled   = false
    if jumpRequestConn then jumpRequestConn:Disconnect(); jumpRequestConn = nil end
    if galaxyVectorForce then galaxyVectorForce:Destroy(); galaxyVectorForce = nil end
    if galaxyAttachment  then galaxyAttachment:Destroy();  galaxyAttachment  = nil end
    adjustGalaxyJump()
    print("🌍 Galaxy DESATIVADO")
end

local function SetGalaxyGravity(percent)
    Config.GalaxyGravityPercent = percent
    if galaxyEnabled then
        updateGalaxyForce()
        adjustGalaxyJump()
    end
    SaveConfig()
end

local function ToggleGalaxy(state)
    Config.GalaxyEnabled = state
    if state then startGalaxy() else stopGalaxy() end
    SaveConfig()
end

globalConnections:Add("GalaxyHeartbeat", RunService.Heartbeat:Connect(function()
    if hopsEnabled and spaceHeld then doMiniHop() end
    if galaxyEnabled then updateGalaxyForce() end
end))

UserInputService.InputBegan:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.Space then
        spaceHeld = true
        if hopsEnabled then doMiniHop() end
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.Space then spaceHeld = false end
end)

player.CharacterAdded:Connect(function()
    task.wait(1)
    captureOriginalJumpPower()
    if galaxyEnabled then
        setupGalaxyForce()
        adjustGalaxyJump()
    end
end)

--------------------------------------------------
-- ANTI RAGDOLL
--------------------------------------------------
local arIsRunning = false
local arConnections = {}
local arCachedChar = {}

local function arCacheCharacter()
    local char = player.Character
    if not char then return false end
    local hum  = char:FindFirstChildOfClass("Humanoid")
    local root = char:FindFirstChild("HumanoidRootPart")
    if not hum or not root then return false end
    arCachedChar = { character = char, humanoid = hum, root = root }
    return true
end

local function arIsRagdolled()
    if not arCachedChar.humanoid then return false end
    local state = arCachedChar.humanoid:GetState()
    local ragdollStates = {
        [Enum.HumanoidStateType.Physics]     = true,
        [Enum.HumanoidStateType.Ragdoll]     = true,
        [Enum.HumanoidStateType.FallingDown] = true,
    }
    if ragdollStates[state] then return true end
    local endTime = player:GetAttribute("RagdollEndTime")
    if endTime then
        local now = workspace:GetServerTimeNow()
        if (endTime - now) > 0 then return true end
    end
    return false
end

local function arPreventRagdoll()
    if not arCachedChar.humanoid or not arCachedChar.root then return end
    local hum  = arCachedChar.humanoid
    local root = arCachedChar.root
    pcall(function()
        player:SetAttribute("RagdollEndTime", workspace:GetServerTimeNow() - 1)
    end)
    if hum.Health > 0 then
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
    end
    pcall(function()
        if root.Anchored then root.Anchored = false end
        root.AssemblyLinearVelocity  = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
    end)
end

local function arKeepMotors()
    if not arCachedChar.character then return end
    for _, obj in ipairs(arCachedChar.character:GetDescendants()) do
        if obj:IsA("Motor6D") and not obj.Enabled then
            pcall(function() obj.Enabled = true end)
        end
    end
end

local function arSetupCamera()
    local camConn = RunService.Heartbeat:Connect(function()
        if not arCachedChar.humanoid then return end
        local cam = workspace.CurrentCamera
        if cam and cam.CameraSubject ~= arCachedChar.humanoid then
            cam.CameraSubject = arCachedChar.humanoid
        end
    end)
    table.insert(arConnections, camConn)
end

local function arHeartbeatLoop()
    while arIsRunning do
        task.wait()
        if not Config.AntiRagdollEnabled then task.wait(0.5); continue end
        if arIsRagdolled() then
            arPreventRagdoll()
            arKeepMotors()
        end
    end
end

local function arOnCharacterAdded()
    for _, conn in ipairs(arConnections) do
        pcall(function() conn:Disconnect() end)
    end
    arConnections = {}
    task.wait(0.5)
    if arCacheCharacter() then
        arSetupCamera()
        arIsRunning = true
        task.spawn(arHeartbeatLoop)
    end
end

local function toggleAntiRagdoll(state)
    Config.AntiRagdollEnabled = state
    SaveConfig()
end

task.spawn(function()
    task.wait(1)
    if arCacheCharacter() then
        arSetupCamera()
        arIsRunning = true
        task.spawn(arHeartbeatLoop)
    end
end)

player.CharacterAdded:Connect(arOnCharacterAdded)

if Config.GalaxyEnabled then
    task.spawn(function() task.wait(1); startGalaxy() end)
end

--------------------------------------------------
-- TRACKER PLAYER
--------------------------------------------------
local TrackerPlayerModule = {}
TrackerPlayerModule.ativo = false
TrackerPlayerModule.followConnection = nil
TrackerPlayerModule.lookConnection = nil
TrackerPlayerModule.align = nil
TrackerPlayerModule.attachment = nil

local TRACKER_SPEED = 60
local TRACKER_TARGET_OFFSET = Vector3.new(0, 1.5, -1.8)

local function getTrackerNearestEnemy()
    if not UpdateCharacterCache() then return nil end
    local nearest, dist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player and plr.Character then
            local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if hrp and hum and hum.Health > 0 then
                local d = (CharacterCache.hrp.Position - hrp.Position).Magnitude
                if d < dist then dist = d; nearest = hrp end
            end
        end
    end
    return nearest
end

local function startTrackerLookAt()
    if not UpdateCharacterCache() then return end
    CharacterCache.hum.AutoRotate = false
    TrackerPlayerModule.attachment = Instance.new("Attachment", CharacterCache.hrp)
    TrackerPlayerModule.align = Instance.new("AlignOrientation")
    TrackerPlayerModule.align.Attachment0 = TrackerPlayerModule.attachment
    TrackerPlayerModule.align.Mode = Enum.OrientationAlignmentMode.OneAttachment
    TrackerPlayerModule.align.Responsiveness = 200
    TrackerPlayerModule.align.MaxTorque = math.huge
    TrackerPlayerModule.align.Parent = CharacterCache.hrp
    TrackerPlayerModule.lookConnection = RunService.Heartbeat:Connect(function()
        if not TrackerPlayerModule.ativo then return end
        if not UpdateCharacterCache() then return end
        local target = getTrackerNearestEnemy()
        if not target then return end
        pcall(function()
            local lookPos = Vector3.new(target.Position.X, CharacterCache.hrp.Position.Y, target.Position.Z)
            TrackerPlayerModule.align.CFrame = CFrame.lookAt(CharacterCache.hrp.Position, lookPos)
        end)
    end)
end

local function stopTrackerLookAt()
    if UpdateCharacterCache() and CharacterCache.hum then
        CharacterCache.hum.AutoRotate = true
    end
    if TrackerPlayerModule.lookConnection then
        TrackerPlayerModule.lookConnection:Disconnect()
        TrackerPlayerModule.lookConnection = nil
    end
    if TrackerPlayerModule.align then pcall(function() TrackerPlayerModule.align:Destroy() end); TrackerPlayerModule.align = nil end
    if TrackerPlayerModule.attachment then pcall(function() TrackerPlayerModule.attachment:Destroy() end); TrackerPlayerModule.attachment = nil end
end

local function startTrackerFollow()
    if not UpdateCharacterCache() then return end
    startTrackerLookAt()
    pcall(function() CharacterCache.hrp.CFrame = CharacterCache.hrp.CFrame + Vector3.new(0, 2, 0) end)
    TrackerPlayerModule.followConnection = RunService.Heartbeat:Connect(function()
        if not TrackerPlayerModule.ativo then return end
        if not UpdateCharacterCache() then return end
        local target = getTrackerNearestEnemy()
        if not target then CharacterCache.hrp.Velocity = Vector3.zero; return end
        pcall(function()
            local desiredPos = target.CFrame:PointToWorldSpace(TRACKER_TARGET_OFFSET)
            local dir = desiredPos - CharacterCache.hrp.Position
            if dir.Magnitude > 0.05 then
                CharacterCache.hrp.Velocity = dir.Unit * TRACKER_SPEED
            else
                CharacterCache.hrp.Velocity = Vector3.zero
            end
        end)
    end)
end

local function stopTrackerFollow()
    if TrackerPlayerModule.followConnection then
        TrackerPlayerModule.followConnection:Disconnect()
        TrackerPlayerModule.followConnection = nil
    end
    stopTrackerLookAt()
    if UpdateCharacterCache() and CharacterCache.hrp then
        CharacterCache.hrp.Velocity = Vector3.zero
    end
end

local function toggleTrackerPlayer(newState)
    TrackerPlayerModule.ativo = newState
    Config.TrackerPlayerEnabled = newState
    if TrackerPlayerModule.ativo then
        UpdateCharacterCache(true)
        startTrackerFollow()
    else
        stopTrackerFollow()
    end
    SaveConfig()
end

--------------------------------------------------
-- AUTO BAT
--------------------------------------------------
local AutoBatModule = {}
AutoBatModule.lastAttack = 0
AutoBatModule.attackInterval = 0.1
AutoBatModule.cachedTarget = nil
AutoBatModule.lastTargetSearch = 0
AutoBatModule.searchInterval = 0.3

function AutoBatModule.FindTarget()
    local now = tick()
    if now - AutoBatModule.lastTargetSearch < AutoBatModule.searchInterval and AutoBatModule.cachedTarget then
        return AutoBatModule.cachedTarget
    end
    AutoBatModule.lastTargetSearch = now
    if not UpdateCharacterCache() then AutoBatModule.cachedTarget = nil; return nil end
    local target, dist = nil, 18
    for _, v in pairs(workspace:GetChildren()) do
        if v:IsA("Model") and v ~= CharacterCache.character and v:FindFirstChild("HumanoidRootPart") then
            local d = (CharacterCache.hrp.Position - v.HumanoidRootPart.Position).Magnitude
            if d < dist then dist = d; target = v end
        end
    end
    AutoBatModule.cachedTarget = target
    return target
end

function AutoBatModule.Toggle(state)
    Config.AutoBatEnabled = state
    if state then
        globalConnections:Add("AutoBat", RunService.Heartbeat:Connect(function()
            if not Config.AutoBatEnabled or not UpdateCharacterCache() then return end
            local now = tick()
            if now - AutoBatModule.lastAttack < AutoBatModule.attackInterval then return end
            local tool = CharacterCache.character:FindFirstChild("Bat") or CharacterCache.character:FindFirstChild("Taco")
            if not tool then
                local bpTool = player.Backpack:FindFirstChild("Bat") or player.Backpack:FindFirstChild("Taco")
                if bpTool then bpTool.Parent = CharacterCache.character; tool = bpTool end
            end
            local target = AutoBatModule.FindTarget()
            if target then
                CharacterCache.hrp.CFrame = CFrame.new(
                    CharacterCache.hrp.Position,
                    Vector3.new(target.HumanoidRootPart.Position.X, CharacterCache.hrp.Position.Y, target.HumanoidRootPart.Position.Z)
                )
            end
            if tool then tool:Activate(); AutoBatModule.lastAttack = now end
        end))
    else
        globalConnections:Remove("AutoBat")
        AutoBatModule.cachedTarget = nil
    end
    SaveConfig()
end

--------------------------------------------------
-- AUTO GRAB COM NAMESPACE ÚNICO
--------------------------------------------------
local AutoGrabModule = {}
AutoGrabModule.CurrentBrainrot = nil
AutoGrabModule.StealCycleStart = 0
AutoGrabModule.StealCycleDuration = 0.15
AutoGrabModule.IsStealingInProgress = false

local agLoopActive = false

local function agGetPos(prompt)
    local p = prompt.Parent
    if p:IsA("BasePart") then return p.Position end
    if p:IsA("Model") then
        local prim = p.PrimaryPart or p:FindFirstChildWhichIsA("BasePart")
        return prim and prim.Position
    end
    if p:IsA("Attachment") then return p.WorldPosition end
    local part = p:FindFirstChildWhichIsA("BasePart", true)
    return part and part.Position
end

local function agFindNearest()
    local c = player.Character
    if not c then return nil end
    local hrp = c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("UpperTorso")
    if not hrp then return nil end
    local plots = workspace:FindFirstChild("Plots")
    if not plots then return nil end
    local myPos = hrp.Position
    local nearest, nearestDist = nil, Config.AutoGrabRange
    for _, plot in ipairs(plots:GetChildren()) do
        for _, obj in ipairs(plot:GetDescendants()) do
            if obj:IsA("ProximityPrompt") and obj.Enabled and obj.ActionText == "Steal" then
                local pos = agGetPos(obj)
                if pos then
                    local dist = (myPos - pos).Magnitude
                    if dist <= obj.MaxActivationDistance and dist < nearestDist then
                        nearest = obj
                        nearestDist = dist
                    end
                end
            end
        end
    end
    return nearest
end

local function agGetAnimalName(prompt)
    if not prompt then return "Procurando brainrot..." end
    local function isIgnored(text)
        if not text or text == "" then return true end
        local lower = text:lower()
        if lower:match("%$") then return true end
        if lower:match("%d+%.?%d*[kmb]") then return true end
        if lower:match("/s") then return true end
        if lower:match("^%d") then return true end
        local ignoreList = {
            "steal","coletar","epico","épico","raro","comum","lendario","lendário",
            "secreto","incomum","especial","offline","dinheiro","grab",
            "base","plot","part","model","mesh","union","handle","hitbox",
            "platform","floor","ground","spawn","stock","projectiles",
        }
        for _, word in ipairs(ignoreList) do
            if lower == word then return true end
        end
        if lower:match("^%x+%-%x+%-%x+%-%x+%-%x+$") then return true end
        return false
    end
    local promptPos = nil
    local pp = prompt.Parent
    if pp:IsA("BasePart") then promptPos = pp.Position
    elseif pp:IsA("Model") then
        local prim = pp.PrimaryPart or pp:FindFirstChildWhichIsA("BasePart")
        if prim then promptPos = prim.Position end
    end
    local plot = prompt.Parent
    for _ = 1, 8 do
        if plot.Parent then plot = plot.Parent end
        if plot.Parent and (plot.Parent.Name == "Plots" or plot.Parent:IsA("Workspace")) then break end
    end
    local bestName, bestDist = nil, 50
    for _, child in ipairs(plot:GetDescendants()) do
        if child:IsA("BillboardGui") then
            for _, label in ipairs(child:GetDescendants()) do
                if label:IsA("TextLabel") and not isIgnored(label.Text) then
                    if promptPos then
                        local modelRoot = child:FindFirstAncestorWhichIsA("Model")
                        if modelRoot then
                            local part = modelRoot.PrimaryPart or modelRoot:FindFirstChildWhichIsA("BasePart")
                            if part then
                                local dist = (part.Position - promptPos).Magnitude
                                if dist < bestDist then bestDist = dist; bestName = label.Text end
                            end
                        end
                    else
                        bestName = label.Text
                    end
                    break
                end
            end
        end
    end
    return bestName or "Animal"
end

local function agFirePrompt(prompt)
    if not prompt then return end
    task.spawn(function()
        pcall(function()
            fireproximityprompt(prompt, 10000)
            prompt:InputHoldBegin()
            task.wait(0.04)
            prompt:InputHoldEnd()
        end)
    end)
end

local function agGrabLoop()
    while _G[AG_KEY] do
        local c = player.Character
        local hum = c and c:FindFirstChildOfClass("Humanoid")
        if hum and hum.WalkSpeed > 29 then
            local nearest = agFindNearest()
            if nearest then
                AutoGrabModule.IsStealingInProgress = true
                AutoGrabModule.StealCycleStart = tick()
                AutoGrabModule.CurrentBrainrot = {
                    Info = { DisplayName = agGetAnimalName(nearest), GenerationText = "$?/s" },
                    Prompt = nearest
                }
                task.wait(0.07)
                agFirePrompt(nearest)
                task.wait(AutoGrabModule.StealCycleDuration)
                AutoGrabModule.IsStealingInProgress = false
                AutoGrabModule.CurrentBrainrot = nil
            end
        end
        task.wait(0.3)
    end
    agLoopActive = false
end

function AutoGrabModule.Toggle(state)
    Config.AutoGrabEnabled = state
    _G[AG_KEY] = state
    if state then
        if not agLoopActive then agLoopActive = true; task.spawn(agGrabLoop) end
    else
        AutoGrabModule.CurrentBrainrot = nil
        AutoGrabModule.IsStealingInProgress = false
    end
    SaveConfig()
end

--------------------------------------------------
-- NO ANIMATIONS
--------------------------------------------------
local function ToggleNoAnims(state)
    Config.NoAnimsEnabled = state
    if state then
        local lastCheck = 0
        globalConnections:Add("NoAnims", RunService.Heartbeat:Connect(function()
            local now = tick()
            if now - lastCheck < 0.2 then return end
            lastCheck = now
            if not UpdateCharacterCache() then return end
            for _, track in pairs(CharacterCache.hum:GetPlayingAnimationTracks()) do track:Stop() end
            local animate = CharacterCache.character:FindFirstChild("Animate")
            if animate and animate.Enabled then animate.Enabled = false end
        end))
    else
        globalConnections:Remove("NoAnims")
        if UpdateCharacterCache() then
            local animate = CharacterCache.character:FindFirstChild("Animate")
            if animate then animate.Enabled = true end
        end
    end
    SaveConfig()
end

--------------------------------------------------
-- OPTIMIZER + XRAY
--------------------------------------------------
local originalTransparency = {}
local xrayEnabled = false

local function ToggleOptimizerXray(state)
    Config.OptimizerXrayEnabled = state
    if state then
        if getgenv then getgenv().OPTIMIZER_ACTIVE = true end
        pcall(function()
            settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
            Lighting.GlobalShadows = false
            Lighting.Brightness = 3
            Lighting.FogEnd = 9e9
        end)
        pcall(function()
            for _, obj in ipairs(workspace:GetDescendants()) do
                pcall(function()
                    if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") then
                        obj:Destroy()
                    elseif obj:IsA("BasePart") then
                        obj.CastShadow = false
                        obj.Material = Enum.Material.Plastic
                    end
                end)
            end
        end)
        xrayEnabled = true
        pcall(function()
            for _, obj in ipairs(workspace:GetDescendants()) do
                if obj:IsA("BasePart") and obj.Anchored and
                   (obj.Name:lower():find("base") or (obj.Parent and obj.Parent.Name:lower():find("base"))) then
                    originalTransparency[obj] = obj.LocalTransparencyModifier
                    obj.LocalTransparencyModifier = 0.85
                end
            end
        end)
    else
        if getgenv then getgenv().OPTIMIZER_ACTIVE = false end
        if xrayEnabled then
            for part, value in pairs(originalTransparency) do
                if part and part.Parent then part.LocalTransparencyModifier = value end
            end
            originalTransparency = {}
            xrayEnabled = false
        end
    end
    SaveConfig()
end

--------------------------------------------------
-- INTERFACE DO USUÁRIO
--------------------------------------------------
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "LacerdaHub"
ScreenGui.Parent = player:WaitForChild("PlayerGui")
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true

local CIANO        = Color3.fromRGB(0, 255, 255)
local CIANO_CLARO  = Color3.fromRGB(100, 255, 255)
local CIANO_ESCURO = Color3.fromRGB(0, 180, 180)
local PRETO_FUNDO  = Color3.fromRGB(0, 0, 0)
local PRETO_PAINEL = Color3.fromRGB(0, 0, 0)
local CINZA_ESCURO = Color3.fromRGB(40, 40, 40)
local CINZA_MEDIO  = Color3.fromRGB(60, 60, 60)
local BRANCO       = Color3.fromRGB(255, 255, 255)

local isMobile = UserInputService.TouchEnabled
local Scale    = isMobile and 0.38 or 0.62
local iconSize = isMobile and 44 or 48

local Toggles = {}

--------------------------------------------------
-- BOTÃO OPEN/CLOSE
--------------------------------------------------
local OpenButton = Instance.new("TextButton", ScreenGui)
OpenButton.Size = UDim2.new(0, iconSize, 0, iconSize)
OpenButton.Position = UDim2.new(0, 10, 0, 60)
OpenButton.BackgroundColor3 = CINZA_ESCURO
OpenButton.Text = "LCH"
OpenButton.Font = Enum.Font.GothamBlack
OpenButton.TextSize = isMobile and 13 or 15
OpenButton.TextColor3 = CIANO
Instance.new("UICorner", OpenButton).CornerRadius = UDim.new(0, 12)

task.spawn(function()
    while true do
        local t1 = TweenService:Create(OpenButton, TweenInfo.new(0.8, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut), {BackgroundTransparency = 0.2})
        t1:Play(); t1.Completed:Wait()
        local t2 = TweenService:Create(OpenButton, TweenInfo.new(0.8, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut), {BackgroundTransparency = 0.1})
        t2:Play(); t2.Completed:Wait()
    end
end)

local obDragging, obDragStart, obStartPos
OpenButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        obDragging = true; obDragStart = input.Position; obStartPos = OpenButton.Position
    end
end)
OpenButton.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        obDragging = false
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if obDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - obDragStart
        OpenButton.Position = UDim2.new(obStartPos.X.Scale, obStartPos.X.Offset + delta.X, obStartPos.Y.Scale, obStartPos.Y.Offset + delta.Y)
    end
end)

--------------------------------------------------
-- MAIN HOLDER
--------------------------------------------------
local guiW = math.floor(460 * Scale)
local guiH = math.floor(665 * Scale)

local MainHolder = Instance.new("Frame", ScreenGui)
MainHolder.BackgroundColor3 = PRETO_FUNDO
MainHolder.BackgroundTransparency = 0
MainHolder.Visible = true
MainHolder.Size = UDim2.new(0, guiW, 0, guiH)
MainHolder.Position = UDim2.new(1, -(guiW + 8), 0, 10)
MainHolder.ZIndex = 10
Instance.new("UICorner", MainHolder).CornerRadius = UDim.new(0, math.floor(15 * Scale))

local Gradient = Instance.new("UIGradient", MainHolder)
Gradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(0,0,0)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0,0,0)),
})
Gradient.Rotation = 135

local function MakeDraggable(frame, dragHandle)
    local isDragging, dStart, dStartPos = false, Vector2.new(), UDim2.new()
    dragHandle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            isDragging = true; dStart = Vector2.new(input.Position.X, input.Position.Y); dStartPos = frame.Position
        end
    end)
    dragHandle.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            isDragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if isDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = Vector2.new(input.Position.X, input.Position.Y) - dStart
            frame.Position = UDim2.new(dStartPos.X.Scale, dStartPos.X.Offset + delta.X, dStartPos.Y.Scale, dStartPos.Y.Offset + delta.Y)
        end
    end)
end

OpenButton.MouseButton1Click:Connect(function()
    MainHolder.Visible = not MainHolder.Visible
end)

local function CreateMainPanel(pos, size, parent)
    local frame = Instance.new("Frame")
    frame.Size = size; frame.Position = pos
    frame.BackgroundColor3 = PRETO_PAINEL
    frame.BackgroundTransparency = 0
    frame.BorderSizePixel = 0
    frame.Parent = parent
    frame.ZIndex = 11
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, math.floor(10 * Scale))
    return frame
end

local panelW = math.floor(215 * Scale)
local panelH = math.floor(550 * Scale)
local LeftSide  = CreateMainPanel(UDim2.new(0, math.floor(10*Scale),  0, math.floor(70*Scale)), UDim2.new(0, panelW, 0, panelH), MainHolder)
local RightSide = CreateMainPanel(UDim2.new(0, math.floor(235*Scale), 0, math.floor(70*Scale)), UDim2.new(0, panelW, 0, panelH), MainHolder)

--------------------------------------------------
-- HEADER
--------------------------------------------------
local HeaderFrame = Instance.new("Frame", MainHolder)
HeaderFrame.Size = UDim2.new(1, 0, 0, math.floor(60 * Scale))
HeaderFrame.Position = UDim2.new(0, 0, 0, 0)
HeaderFrame.BackgroundTransparency = 1
HeaderFrame.ZIndex = 11

local TitleLabel = Instance.new("TextLabel", HeaderFrame)
TitleLabel.Text = "Lacerda Hub"
TitleLabel.Font = Enum.Font.GothamBlack
TitleLabel.TextSize = math.floor(28 * Scale)
TitleLabel.TextColor3 = CIANO
TitleLabel.Size = UDim2.new(1, 0, 0.6, 0)
TitleLabel.Position = UDim2.new(0, 0, 0, 0)
TitleLabel.BackgroundTransparency = 1
TitleLabel.ZIndex = 11
TitleLabel.TextStrokeTransparency = 0.5
TitleLabel.TextStrokeColor3 = Color3.fromRGB(0,0,0)

local TitleGlow = Instance.new("UIGradient", TitleLabel)
TitleGlow.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0,   CIANO),
    ColorSequenceKeypoint.new(0.5, CIANO_CLARO),
    ColorSequenceKeypoint.new(1,   CIANO)
})
TitleGlow.Rotation = 0

local DiscordButton = Instance.new("TextButton", HeaderFrame)
DiscordButton.Size = UDim2.new(0, math.floor(300*Scale), 0, math.floor(32*Scale))
DiscordButton.Position = UDim2.new(0.5, -math.floor(150*Scale), 1, -math.floor(36*Scale))
DiscordButton.BackgroundTransparency = 1
DiscordButton.Text = "Discord: discord.gg/xkZh24RHYN"
DiscordButton.Font = Enum.Font.GothamBlack
DiscordButton.TextSize = math.floor(15 * Scale)
DiscordButton.TextColor3 = Color3.fromRGB(0, 200, 200)
DiscordButton.ZIndex = 11
DiscordButton.AutoButtonColor = false
DiscordButton.TextXAlignment = Enum.TextXAlignment.Center

DiscordButton.MouseButton1Click:Connect(function()
    if setclipboard then
        setclipboard("https://discord.gg/xkZh24RHYN")
        local orig = DiscordButton.Text
        DiscordButton.Text = "✓ LINK COPIADO!"
        DiscordButton.TextColor3 = Color3.fromRGB(0, 255, 100)
        task.wait(1.5)
        DiscordButton.Text = orig
        DiscordButton.TextColor3 = Color3.fromRGB(0, 200, 200)
    end
end)

MakeDraggable(MainHolder, HeaderFrame)

--------------------------------------------------
-- SISTEMA DE TEMA
--------------------------------------------------
local ThemeCallbacks = {}
local function RegisterThemeCallback(fn) table.insert(ThemeCallbacks, fn) end
local gradientElements = {}

local function buildGradientFromColor(cor)
    local isWhite = cor.R > 0.8 and cor.G > 0.8 and cor.B > 0.8
    local isBlack = cor.R < 0.1 and cor.G < 0.1 and cor.B < 0.1
    local c1, c2
    if isWhite then c1 = Color3.fromRGB(0,0,0); c2 = Color3.fromRGB(220,220,220)
    elseif isBlack then c1 = Color3.fromRGB(200,200,200); c2 = Color3.fromRGB(10,10,10)
    else c1 = Color3.fromRGB(0,0,0); c2 = Color3.new(cor.R*0.75, cor.G*0.75, cor.B*0.75) end
    return ColorSequence.new({
        ColorSequenceKeypoint.new(0,   c1),
        ColorSequenceKeypoint.new(0.5, cor),
        ColorSequenceKeypoint.new(1,   c2),
    })
end

RegisterThemeCallback(function(cor)
    local seq = buildGradientFromColor(cor)
    for _, g in ipairs(gradientElements) do pcall(function() g.Color = seq end) end
end)

--------------------------------------------------
-- ATUALIZAR BOTÕES DE SPEED
--------------------------------------------------
local function AtualizarBotoesSpeed()
    if not Toggles or not Toggles[30.0] or not Toggles[55] then return end
    if not UpdateCharacterCache() then return end
    local steal = CharacterCache.hum.WalkSpeed < 25
    if steal then
        Toggles[30.0].update(true); Toggles[55].update(false)
    else
        Toggles[55].update(true); Toggles[30.0].update(false)
    end
end

task.spawn(function()
    repeat task.wait(0.1) until Toggles and Toggles[30.0] and Toggles[55]
    AtualizarBotoesSpeed()
    while true do
        task.wait(0.3)
        if UpdateCharacterCache() then pcall(AtualizarBotoesSpeed) end
    end
end)

player.CharacterAdded:Connect(function()
    task.wait(0.5); AtualizarBotoesSpeed()
end)

--------------------------------------------------
-- SISTEMA DE BIND CORRIGIDO
--------------------------------------------------
local BindSystem = {}
BindSystem.listeningFor = nil

local function keyCodeToString(keyCode)
    local name = keyCode.Name
    local map = {
        LeftShift="LShift", RightShift="RShift",
        LeftControl="LCtrl", RightControl="RCtrl",
        LeftAlt="LAlt", RightAlt="RAlt",
        Space="Space", Return="Enter",
        BackSpace="Back", Delete="Del", Tab="Tab",
    }
    return map[name] or name
end

local function CreateItem(parent, name, yPos, targetSpeed, initialState, noBind)
    local item = Instance.new("TextButton")
    item.Size = UDim2.new(0.9, 0, 0, math.floor(45 * Scale))
    item.Position = UDim2.new(0.05, 0, 0, math.floor(yPos * Scale))
    item.BackgroundColor3 = PRETO_PAINEL
    item.Text = ""
    item.Parent = parent
    item.ZIndex = 12
    item.AutoButtonColor = false
    Instance.new("UICorner", item).CornerRadius = UDim.new(0, math.floor(10 * Scale))

    local originalColor = PRETO_PAINEL
    local hoverColor = Color3.fromRGB(15,15,15)
    item.MouseEnter:Connect(function()
        if item.BackgroundColor3 == originalColor then item.BackgroundColor3 = hoverColor end
    end)
    item.MouseLeave:Connect(function()
        if item.BackgroundColor3 == hoverColor then item.BackgroundColor3 = originalColor end
    end)

    local boundKey = nil
    local bindBtn = nil
    local namePosX = 0.07

    if not targetSpeed and not noBind then
        bindBtn = Instance.new("TextButton")
        bindBtn.Size = UDim2.new(0, math.floor(34*Scale), 0, math.floor(22*Scale))
        bindBtn.Position = UDim2.new(0, math.floor(6*Scale), 0.5, -math.floor(11*Scale))
        bindBtn.BackgroundColor3 = Color3.fromRGB(18,18,30)
        bindBtn.Text = ""
        bindBtn.AutoButtonColor = false
        bindBtn.ZIndex = 13
        bindBtn.Parent = item
        Instance.new("UICorner", bindBtn).CornerRadius = UDim.new(0, math.floor(6*Scale))

        local bindStroke = Instance.new("UIStroke", bindBtn)
        bindStroke.Color = CIANO
        bindStroke.Thickness = 1.5

        local bindLabel = Instance.new("TextLabel")
        bindLabel.Size = UDim2.new(1,0,1,0)
        bindLabel.Position = UDim2.new(0,0,0,0)
        bindLabel.BackgroundTransparency = 1
        bindLabel.Text = "---"
        bindLabel.Font = Enum.Font.GothamBlack
        bindLabel.TextSize = math.floor(10 * Scale)
        bindLabel.TextColor3 = CIANO
        bindLabel.ZIndex = 14
        bindLabel.Parent = bindBtn

        namePosX = 0.30

        bindBtn.MouseButton1Click:Connect(function()
            if BindSystem.listeningFor == bindBtn then
                BindSystem.listeningFor = nil
                bindLabel.Text = boundKey and keyCodeToString(boundKey) or "---"
                bindLabel.TextColor3 = CIANO
                bindStroke.Color = CIANO
            else
                BindSystem.listeningFor = bindBtn
                bindLabel.Text = "..."
                bindLabel.TextColor3 = Color3.fromRGB(255, 220, 0)
                bindStroke.Color = Color3.fromRGB(255, 220, 0)
            end
        end)

        UserInputService.InputBegan:Connect(function(input, gpe)
            if BindSystem.listeningFor ~= bindBtn then return end
            if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
            if input.KeyCode == Enum.KeyCode.Escape then
                boundKey = nil
                bindLabel.Text = "---"
                bindLabel.TextColor3 = CIANO
                bindStroke.Color = CIANO
                BindSystem.listeningFor = nil
                return
            end
            boundKey = input.KeyCode
            bindLabel.Text = keyCodeToString(boundKey)
            bindLabel.TextColor3 = CIANO
            bindStroke.Color = CIANO
            BindSystem.listeningFor = nil
        end)

        UserInputService.InputBegan:Connect(function(input, gpe)
            if gpe then return end
            if BindSystem.listeningFor ~= nil then return end
            if not boundKey then return end
            if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
            if input.KeyCode ~= boundKey then return end
            item:Activated()
        end)

        RegisterThemeCallback(function(cor)
            if BindSystem.listeningFor ~= bindBtn then
                bindLabel.TextColor3 = cor
                bindStroke.Color = cor
            end
        end)
    end

    local label = Instance.new("TextLabel")
    label.Text = name
    label.Font = Enum.Font.GothamBold
    label.TextSize = math.floor(12 * Scale)
    label.TextColor3 = BRANCO
    label.Position = UDim2.new(namePosX, 0, 0, 0)
    label.Size = UDim2.new(1 - namePosX - 0.35, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = item
    label.ZIndex = 12

    local switchBg = Instance.new("Frame")
    switchBg.Size = UDim2.new(0, math.floor(50*Scale), 0, math.floor(24*Scale))
    switchBg.Position = UDim2.new(1, -math.floor(58*Scale), 0.5, -math.floor(12*Scale))
    switchBg.BackgroundColor3 = CINZA_MEDIO
    switchBg.Parent = item
    switchBg.ZIndex = 12
    Instance.new("UICorner", switchBg).CornerRadius = UDim.new(0.5, 0)

    local circle = Instance.new("Frame")
    circle.Size = UDim2.new(0, math.floor(18*Scale), 0, math.floor(18*Scale))
    circle.Position = UDim2.new(0, math.floor(3*Scale), 0.5, -math.floor(9*Scale))
    circle.BackgroundColor3 = BRANCO
    circle.Parent = switchBg
    circle.ZIndex = 12
    Instance.new("UICorner", circle).CornerRadius = UDim.new(1, 0)

    local switchGradient = Instance.new("UIGradient", switchBg)
    switchGradient.Rotation = 0
    switchGradient.Enabled = false

    local active = initialState or false

    local function updateButtonAppearance(isActive)
        if isActive then
            circle.Position = UDim2.new(1, -math.floor(21*Scale), 0.5, -math.floor(9*Scale))
            switchBg.BackgroundColor3 = Color3.fromRGB(255,255,255)
            switchGradient.Color = buildGradientFromColor(CIANO)
            switchGradient.Enabled = true
            circle.BackgroundColor3 = BRANCO
            label.TextColor3 = CIANO
            item.BackgroundColor3 = Color3.fromRGB(15,15,15)
        else
            circle.Position = UDim2.new(0, math.floor(3*Scale), 0.5, -math.floor(9*Scale))
            switchGradient.Enabled = false
            switchBg.BackgroundColor3 = CINZA_ESCURO
            circle.BackgroundColor3 = BRANCO
            label.TextColor3 = BRANCO
            item.BackgroundColor3 = originalColor
        end
    end

    if targetSpeed == 55 then
        active = Config.SpeedAutoActive and Config.SPEED_NORMAL == Config.Speed55Value
        updateButtonAppearance(active)
    elseif targetSpeed == 30.0 then
        active = Config.SpeedAutoActive and Config.SPEED_STEAL == Config.Speed30Value
        updateButtonAppearance(active)
    else
        updateButtonAppearance(active)
    end

    if targetSpeed then
        Toggles[targetSpeed] = {
            update = function(s) active = s; updateButtonAppearance(s) end,
            value = targetSpeed == 55 and Config.Speed55Value or Config.Speed30Value
        }
    end

    RegisterThemeCallback(function(cor)
        if active then
            switchBg.BackgroundColor3 = Color3.fromRGB(255,255,255)
            switchGradient.Color = buildGradientFromColor(cor)
            switchGradient.Enabled = true
            label.TextColor3 = cor
        end
    end)

    local function doToggle()
        active = not active
        updateButtonAppearance(active)
        if name == "Optimizer + Xray" then ToggleOptimizerXray(active)
        elseif name == "Anti Ragdoll"   then toggleAntiRagdoll(active)
        elseif name == "Auto Bat"        then AutoBatModule.Toggle(active)
        elseif name == "Auto Grab"       then AutoGrabModule.Toggle(active)
        elseif name == "Tracker Player"  then toggleTrackerPlayer(active)
        elseif name == "No Anims"        then ToggleNoAnims(active)
        elseif name == "Inf Jump"        then ToggleGalaxy(active)
        end
    end

    item.MouseButton1Click:Connect(function()
        if targetSpeed then
            local targetValue = targetSpeed == 55 and Config.Speed55Value or Config.Speed30Value
            if targetSpeed == 55 then
                if Config.SpeedAutoActive and Config.SPEED_NORMAL == targetValue then
                    SetSpeedAutoEnabled(false)
                    if Toggles[55] then Toggles[55].update(false) end
                    if Toggles[30.0] then Toggles[30.0].update(false) end
                else
                    for k, v in pairs(Toggles) do if type(k) == "number" and v.update then v.update(false) end end
                    Config.SPEED_NORMAL = targetValue
                    SetSpeedAutoEnabled(true)
                    if Toggles[55] then Toggles[55].update(true) end
                end
            elseif targetSpeed == 30.0 then
                if Config.SpeedAutoActive and Config.SPEED_STEAL == targetValue then
                    SetSpeedAutoEnabled(false)
                    if Toggles[30.0] then Toggles[30.0].update(false) end
                    if Toggles[55] then Toggles[55].update(false) end
                else
                    for k, v in pairs(Toggles) do if type(k) == "number" and v.update then v.update(false) end end
                    Config.SPEED_STEAL = targetValue
                    SetSpeedAutoEnabled(true)
                    if Toggles[30.0] then Toggles[30.0].update(true) end
                end
            end
        else
            doToggle()
        end
    end)

    return item
end

--------------------------------------------------
-- CRIAR SLIDER
--------------------------------------------------
local function CreateConfigSlider(parent, yPos, title, minValue, maxValue, currentValue, callback, configKey)
    local container = Instance.new("Frame", parent)
    container.Size = UDim2.new(0.9, 0, 0, math.floor(60 * Scale))
    container.Position = UDim2.new(0.05, 0, 0, math.floor(yPos * Scale))
    container.BackgroundTransparency = 1
    container.ZIndex = 12

    local titleLabel = Instance.new("TextLabel", container)
    titleLabel.Text = title
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.TextSize = math.floor(13 * Scale)
    titleLabel.TextColor3 = BRANCO
    titleLabel.Size = UDim2.new(1, 0, 0, math.floor(20 * Scale))
    titleLabel.BackgroundTransparency = 1
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.ZIndex = 12

    local sliderContainer = Instance.new("Frame", container)
    sliderContainer.Size = UDim2.new(1, 0, 0, math.floor(30 * Scale))
    sliderContainer.Position = UDim2.new(0, 0, 0, math.floor(25 * Scale))
    sliderContainer.BackgroundTransparency = 1
    sliderContainer.ZIndex = 12

    local sliderTrack = Instance.new("Frame", sliderContainer)
    sliderTrack.Size = UDim2.new(1, 0, 0, math.floor(12 * Scale))
    sliderTrack.Position = UDim2.new(0, 0, 0.5, -math.floor(6 * Scale))
    sliderTrack.BackgroundColor3 = CINZA_ESCURO
    sliderTrack.ZIndex = 12
    Instance.new("UICorner", sliderTrack).CornerRadius = UDim.new(1, 0)

    local sliderFill = Instance.new("Frame", sliderTrack)
    sliderFill.Size = UDim2.new((currentValue - minValue) / (maxValue - minValue), 0, 1, 0)
    sliderFill.BackgroundColor3 = CIANO
    sliderFill.ZIndex = 12
    Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(1, 0)

    local thumb = Instance.new("Frame", sliderContainer)
    thumb.Size = UDim2.new(0, math.floor(24*Scale), 0, math.floor(24*Scale))
    thumb.Position = UDim2.new((currentValue - minValue) / (maxValue - minValue), -math.floor(12*Scale), 0.5, -math.floor(12*Scale))
    thumb.BackgroundColor3 = BRANCO
    thumb.ZIndex = 13
    Instance.new("UICorner", thumb).CornerRadius = UDim.new(1, 0)

    local valueDisplay = Instance.new("TextLabel", thumb)
    valueDisplay.Text = tostring(currentValue)
    valueDisplay.Font = Enum.Font.GothamBold
    valueDisplay.TextSize = math.floor(11 * Scale)
    valueDisplay.TextColor3 = PRETO_FUNDO
    valueDisplay.Size = UDim2.new(1,0,1,0)
    valueDisplay.BackgroundTransparency = 1
    valueDisplay.ZIndex = 14

    local sliderDragging = false

    local function updateSliderValue(value)
        value = math.floor(math.clamp(value, minValue, maxValue))
        local rel = (value - minValue) / (maxValue - minValue)
        sliderFill.Size = UDim2.new(rel, 0, 1, 0)
        thumb.Position = UDim2.new(rel, -math.floor(12*Scale), 0.5, -math.floor(12*Scale))
        valueDisplay.Text = tostring(value)
        if configKey then
            if Config[configKey] ~= nil then Config[configKey] = value end
            if Values[configKey] ~= nil then Values[configKey] = value end
        end
        if callback then callback(value) end
    end

    local function updateFromInput(input)
        if sliderDragging then
            local rel = (input.Position.X - sliderTrack.AbsolutePosition.X) / sliderTrack.AbsoluteSize.X
            updateSliderValue(math.floor(minValue + math.clamp(rel, 0, 1) * (maxValue - minValue)))
        end
    end

    thumb.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then sliderDragging = true end
    end)
    sliderTrack.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then sliderDragging = true; updateFromInput(input) end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            if sliderDragging then sliderDragging = false; SaveConfig() end
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if sliderDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then updateFromInput(input) end
    end)

    updateSliderValue(currentValue)

    RegisterThemeCallback(function(cor)
        local g = sliderFill:FindFirstChildOfClass("UIGradient")
        if not g then g = Instance.new("UIGradient", sliderFill); g.Rotation = 0; table.insert(gradientElements, g) end
        g.Color = buildGradientFromColor(cor)
        sliderFill.BackgroundColor3 = Color3.fromRGB(255,255,255)
    end)

    return container
end

--------------------------------------------------
-- BOTÕES DA INTERFACE
--------------------------------------------------
CreateItem(LeftSide, "Speed Booster", 10, 55, Config.SpeedAutoActive and Config.SPEED_NORMAL == Config.Speed55Value)
CreateConfigSlider(LeftSide, 60, "Speed", 10, 100, Config.Speed55Value, function(value)
    Config.Speed55Value = value
    if Config.SpeedAutoActive then Config.SPEED_NORMAL = value end
end, "Speed55Value")

CreateItem(LeftSide, "Speed", 135, 30.0, Config.SpeedAutoActive and Config.SPEED_STEAL == Config.Speed30Value)
CreateConfigSlider(LeftSide, 185, "Speed", 10, 100, Config.Speed30Value, function(value)
    Config.Speed30Value = value
    if Config.SpeedAutoActive then Config.SPEED_STEAL = value end
end, "Speed30Value")

CreateItem(LeftSide, "Inf Jump", 265, nil, Config.GalaxyEnabled, true)
CreateConfigSlider(LeftSide, 315, "Gravity %", 10, 100, Config.GalaxyGravityPercent, function(value)
    SetGalaxyGravity(value)
end, "GalaxyGravityPercent")

CreateConfigSlider(LeftSide, 385, "Jump Power", 10, 80, Config.HopPower, function(value)
    Config.HopPower = value; SaveConfig()
end, "HopPower")

CreateItem(LeftSide, "No Anims", 460, nil, Config.NoAnimsEnabled, true)

CreateItem(RightSide, "Anti Ragdoll",    10,  nil, Config.AntiRagdollEnabled, true)
CreateItem(RightSide, "Optimizer + Xray", 85,  nil, Config.OptimizerXrayEnabled, true)
CreateItem(RightSide, "Auto Grab",        160, nil, Config.AutoGrabEnabled, true)
CreateItem(RightSide, "Tracker Player",   235, nil, Config.TrackerPlayerEnabled)
CreateItem(RightSide, "Auto Bat",         310, nil, Config.AutoBatEnabled)

-- BOTÃO SAVE CONFIG
do
    local saveBtn = Instance.new("TextButton")
    saveBtn.Size = UDim2.new(0.9, 0, 0, math.floor(40 * Scale))
    saveBtn.Position = UDim2.new(0.05, 0, 0, math.floor(400 * Scale))
    saveBtn.BackgroundColor3 = CIANO
    saveBtn.Text = "💾 Save Config"
    saveBtn.Font = Enum.Font.GothamBlack
    saveBtn.TextSize = math.floor(13 * Scale)
    saveBtn.TextColor3 = Color3.fromRGB(0,0,0)
    saveBtn.AutoButtonColor = false
    saveBtn.ZIndex = 12
    saveBtn.Parent = RightSide
    Instance.new("UICorner", saveBtn).CornerRadius = UDim.new(0, math.floor(10*Scale))

    local saveBtnGradient = Instance.new("UIGradient", saveBtn)
    saveBtnGradient.Color = buildGradientFromColor(CIANO)

    saveBtn.MouseButton1Click:Connect(function()
        SaveConfig()
        local orig = saveBtn.Text
        saveBtn.Text = "✅ Salvo!"
        task.wait(1.5)
        saveBtn.Text = orig
    end)

    RegisterThemeCallback(function(cor)
        saveBtn.BackgroundColor3 = Color3.fromRGB(255,255,255)
        saveBtnGradient.Color = buildGradientFromColor(cor)
    end)
end

--------------------------------------------------
-- SLIDER DE COR DO TEMA
--------------------------------------------------
RegisterThemeCallback(function(cor, corClaro, corEscuro)
    local isBlack = cor.R < 0.1 and cor.G < 0.1 and cor.B < 0.1
    local corVisivel = isBlack and Color3.fromRGB(200,200,200) or cor
    TitleLabel.TextColor3 = Color3.fromRGB(255,255,255)
    TitleGlow.Rotation = 0
    TitleGlow.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0,    Color3.fromRGB(0,0,0)),
        ColorSequenceKeypoint.new(0.25, Color3.fromRGB(0,0,0)),
        ColorSequenceKeypoint.new(0.45, corVisivel),
        ColorSequenceKeypoint.new(0.5,  Color3.new(math.min(corVisivel.R+0.25,1), math.min(corVisivel.G+0.25,1), math.min(corVisivel.B+0.25,1))),
        ColorSequenceKeypoint.new(0.55, corVisivel),
        ColorSequenceKeypoint.new(0.75, Color3.fromRGB(0,0,0)),
        ColorSequenceKeypoint.new(1,    Color3.fromRGB(0,0,0)),
    })
    local dGrad = DiscordButton:FindFirstChildOfClass("UIGradient")
    if not dGrad then dGrad = Instance.new("UIGradient", DiscordButton); dGrad.Rotation = 0 end
    dGrad.Color = buildGradientFromColor(corVisivel)
    DiscordButton.TextColor3 = Color3.fromRGB(255,255,255)
    OpenButton.TextColor3 = corVisivel
end)

do
    local colorBar = Instance.new("Frame", MainHolder)
    colorBar.Size = UDim2.new(1, -math.floor(20*Scale), 0, math.floor(28*Scale))
    colorBar.Position = UDim2.new(0, math.floor(10*Scale), 0, math.floor(628*Scale))
    colorBar.BackgroundTransparency = 1
    colorBar.ZIndex = 12

    local colorLabel = Instance.new("TextLabel", colorBar)
    colorLabel.Size = UDim2.new(0, math.floor(30*Scale), 1, 0)
    colorLabel.BackgroundTransparency = 1
    colorLabel.Text = "Cor:"
    colorLabel.Font = Enum.Font.GothamBold
    colorLabel.TextSize = math.floor(10*Scale)
    colorLabel.TextColor3 = BRANCO
    colorLabel.TextXAlignment = Enum.TextXAlignment.Left
    colorLabel.ZIndex = 12

    local hueTrack = Instance.new("Frame", colorBar)
    hueTrack.Size = UDim2.new(1, -math.floor(66*Scale), 0, math.floor(14*Scale))
    hueTrack.Position = UDim2.new(0, math.floor(34*Scale), 0.5, -math.floor(7*Scale))
    hueTrack.BackgroundColor3 = Color3.fromRGB(255,255,255)
    hueTrack.BorderSizePixel = 0
    hueTrack.ZIndex = 12
    Instance.new("UICorner", hueTrack).CornerRadius = UDim.new(1, 0)

    local hueGradient = Instance.new("UIGradient", hueTrack)
    hueGradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0,    Color3.fromRGB(0,0,0)),
        ColorSequenceKeypoint.new(0.1,  Color3.fromHSV(0,    1, 0.8)),
        ColorSequenceKeypoint.new(0.2,  Color3.fromHSV(0.08, 1, 1)),
        ColorSequenceKeypoint.new(0.3,  Color3.fromHSV(0.17, 1, 0.7)),
        ColorSequenceKeypoint.new(0.4,  Color3.fromHSV(0.33, 1, 1)),
        ColorSequenceKeypoint.new(0.5,  Color3.fromHSV(0.5,  1, 0.7)),
        ColorSequenceKeypoint.new(0.6,  Color3.fromHSV(0.58, 1, 1)),
        ColorSequenceKeypoint.new(0.7,  Color3.fromHSV(0.67, 1, 0.7)),
        ColorSequenceKeypoint.new(0.8,  Color3.fromHSV(0.75, 1, 1)),
        ColorSequenceKeypoint.new(0.88, Color3.fromHSV(0.83, 1, 0.8)),
        ColorSequenceKeypoint.new(0.95, Color3.fromRGB(200,200,200)),
        ColorSequenceKeypoint.new(1,    Color3.fromRGB(255,255,255)),
    })

    local hueThumb = Instance.new("Frame", colorBar)
    hueThumb.Size = UDim2.new(0, math.floor(16*Scale), 0, math.floor(16*Scale))
    hueThumb.BackgroundColor3 = CIANO
    hueThumb.ZIndex = 13
    Instance.new("UICorner", hueThumb).CornerRadius = UDim.new(1, 0)
    local thumbStroke = Instance.new("UIStroke", hueThumb)
    thumbStroke.Color = Color3.fromRGB(0,0,0); thumbStroke.Thickness = 1.5

    local colorPreview = Instance.new("Frame", colorBar)
    colorPreview.Size = UDim2.new(0, math.floor(18*Scale), 0, math.floor(18*Scale))
    colorPreview.Position = UDim2.new(1, -math.floor(18*Scale), 0.5, -math.floor(9*Scale))
    colorPreview.BackgroundColor3 = CIANO
    colorPreview.BorderSizePixel = 0
    colorPreview.ZIndex = 12
    Instance.new("UICorner", colorPreview).CornerRadius = UDim.new(0, math.floor(4*Scale))

    local currentHue = 0.5
    local draggingHue = false

    local function applyThemeColor(hue)
        local cor
        if hue <= 0.05 then cor = Color3.fromRGB(20,20,20)
        elseif hue >= 0.92 then cor = Color3.fromRGB(220,220,220)
        else cor = Color3.fromHSV(((hue-0.05)/(0.92-0.05))*0.83, 1, 1) end
        local corClaro  = Color3.new(math.min(cor.R+0.3,1), math.min(cor.G+0.3,1), math.min(cor.B+0.3,1))
        local corEscuro = Color3.new(cor.R*0.65, cor.G*0.65, cor.B*0.65)
        CIANO = cor; CIANO_CLARO = corClaro; CIANO_ESCURO = corEscuro
        colorPreview.BackgroundColor3 = cor
        hueThumb.BackgroundColor3 = cor
        for _, fn in ipairs(ThemeCallbacks) do pcall(fn, cor, corClaro, corEscuro) end
    end

    local function positionThumb(hue)
        local trackAbsX = hueTrack.AbsolutePosition.X
        local barAbsX   = colorBar.AbsolutePosition.X
        local offsetX   = (trackAbsX - barAbsX) + hue * hueTrack.AbsoluteSize.X - (hueThumb.AbsoluteSize.X / 2)
        hueThumb.Position = UDim2.new(0, offsetX, 0.5, -math.floor(8*Scale))
    end

    local function updateHueFromInput(input)
        if draggingHue then
            local rel = (input.Position.X - hueTrack.AbsolutePosition.X) / hueTrack.AbsoluteSize.X
            currentHue = math.clamp(rel, 0, 1)
            positionThumb(currentHue)
            applyThemeColor(currentHue)
        end
    end

    hueThumb.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then draggingHue = true end
    end)
    hueTrack.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then draggingHue = true; updateHueFromInput(input) end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then draggingHue = false end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if draggingHue and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then updateHueFromInput(input) end
    end)
    task.spawn(function() task.wait(0.15); positionThumb(currentHue); applyThemeColor(currentHue) end)
end

--------------------------------------------------
-- HUD AUTO GRAB - SEM SLIDER, RANGE FIXO EM 7
--------------------------------------------------
task.spawn(function()
    local agScreenGui = Instance.new("ScreenGui")
    agScreenGui.Name = "AutoStealProgressUI_" .. player.UserId
    agScreenGui.ResetOnSpawn = false
    agScreenGui.IgnoreGuiInset = true
    agScreenGui.Parent = player:WaitForChild("PlayerGui")

    local fw = isMobile and 200 or 280
    local fh = isMobile and 48 or 56

    local progressFrame = Instance.new("Frame")
    progressFrame.Size = UDim2.new(0, fw, 0, fh)
    progressFrame.Position = UDim2.new(0.5, -fw/2, 0, 20)
    progressFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 28)
    progressFrame.BorderSizePixel = 0
    progressFrame.Visible = false
    progressFrame.Parent = agScreenGui
    Instance.new("UICorner", progressFrame).CornerRadius = UDim.new(0, 10)

    local progressStroke = Instance.new("UIStroke")
    progressStroke.Color = CIANO
    progressStroke.Thickness = 2
    progressStroke.Parent = progressFrame

    local agTitleLabel = Instance.new("TextLabel")
    agTitleLabel.Size = UDim2.new(1, -16, 0, 18)
    agTitleLabel.Position = UDim2.new(0, 8, 0, 4)
    agTitleLabel.BackgroundTransparency = 1
    agTitleLabel.Text = "Auto Grab"
    agTitleLabel.TextColor3 = CIANO
    agTitleLabel.TextSize = isMobile and 11 or 12
    agTitleLabel.Font = Enum.Font.GothamBlack
    agTitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    agTitleLabel.Parent = progressFrame

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, -16, 0, 14)
    nameLabel.Position = UDim2.new(0, 8, 0, 20)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = "Procurando brainrot..."
    nameLabel.TextColor3 = Color3.fromRGB(220, 220, 220)
    nameLabel.TextSize = isMobile and 9 or 10
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left
    nameLabel.TextTruncate = Enum.TextTruncate.AtEnd
    nameLabel.Parent = progressFrame

    local barBackground = Instance.new("Frame")
    barBackground.Size = UDim2.new(1, -16, 0, 7)
    barBackground.Position = UDim2.new(0, 8, 1, -13)
    barBackground.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
    barBackground.BorderSizePixel = 0
    barBackground.Parent = progressFrame
    Instance.new("UICorner", barBackground).CornerRadius = UDim.new(1, 0)

    local barFill = Instance.new("Frame")
    barFill.Size = UDim2.new(0, 0, 1, 0)
    barFill.BackgroundColor3 = CIANO
    barFill.BorderSizePixel = 0
    barFill.Parent = barBackground
    Instance.new("UICorner", barFill).CornerRadius = UDim.new(1, 0)

    local fillGradient = Instance.new("UIGradient")
    fillGradient.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0,   CIANO_ESCURO),
        ColorSequenceKeypoint.new(0.5, CIANO),
        ColorSequenceKeypoint.new(1,   CIANO_CLARO)
    }
    fillGradient.Rotation = 90
    fillGradient.Parent = barFill

    RegisterThemeCallback(function(cor, corClaro, corEscuro)
        progressStroke.Color = cor
        agTitleLabel.TextColor3 = cor
        barFill.BackgroundColor3 = Color3.fromRGB(255,255,255)
        local g = barFill:FindFirstChildOfClass("UIGradient") or Instance.new("UIGradient", barFill)
        g.Rotation = 0; g.Color = buildGradientFromColor(cor)
        fillGradient.Color = ColorSequence.new{
            ColorSequenceKeypoint.new(0,   corEscuro or cor),
            ColorSequenceKeypoint.new(0.5, cor),
            ColorSequenceKeypoint.new(1,   corClaro or cor)
        }
    end)

    task.spawn(function()
        while true do
            for rot = 0, 360, 5 do fillGradient.Rotation = rot; task.wait(0.05) end
        end
    end)

    local agLastState = false
    RunService.RenderStepped:Connect(function()
        local currentState = _G[AG_KEY] == true
        if currentState ~= agLastState then
            agLastState = currentState
            progressFrame.Visible = currentState
            if not currentState then
                AutoGrabModule.CurrentBrainrot = nil
                AutoGrabModule.IsStealingInProgress = false
                nameLabel.Text = "Procurando brainrot..."
            end
        end
        if not currentState or not progressFrame.Visible then return end
        nameLabel.Text = AutoGrabModule.CurrentBrainrot
            and AutoGrabModule.CurrentBrainrot.Info.DisplayName
            or "Procurando brainrot..."
        if AutoGrabModule.IsStealingInProgress then
            barFill.Size = UDim2.new(math.clamp(
                (tick() - AutoGrabModule.StealCycleStart) / AutoGrabModule.StealCycleDuration,
                0, 1
            ), 0, 1, 0)
        else
            barFill.Size = UDim2.new(0, 0, 1, 0)
        end
    end)
end)

print("✅ LACERDA HUB V3.3 CARREGADO!")
