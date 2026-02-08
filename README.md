--------------------------------------------------
-- SERVICES
--------------------------------------------------
local RunSvc = game:GetService("RunService")
local Players = game:GetService("Players")
local RepStorage = game:GetService("ReplicatedStorage")

--------------------------------------------------
-- REMOTES
--------------------------------------------------
local PlayAnimationState = RepStorage:FindFirstChild("PlayAnimation")
if not PlayAnimationState then
    PlayAnimationState = Instance.new("RemoteEvent")
    PlayAnimationState.Name = "PlayAnimation"
    PlayAnimationState.Parent = RepStorage
end

local DashEvent = RepStorage:FindFirstChild("DashEvent")
if not DashEvent then
    DashEvent = Instance.new("RemoteEvent")
    DashEvent.Name = "DashEvent"
    DashEvent.Parent = RepStorage
end

--------------------------------------------------
-- CONFIG
--------------------------------------------------
local MOVEMENT_THRESHOLD = 0.01
local DASH_TIME = 0.45
local CharacterData = {}

--------------------------------------------------
-- HELPERS
--------------------------------------------------
local function FireToAllExceptOwner(playerToExclude, state)
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= playerToExclude then
            PlayAnimationState:FireClient(player, playerToExclude, state)
        end
    end
end

local function fire(player, state)
    PlayAnimationState:FireClient(player, player, state)
    FireToAllExceptOwner(player, state)
end

--------------------------------------------------
-- BASE STATE
--------------------------------------------------
local function setBaseState(player, data, newState)
    if data.BaseState == newState then return end
    data.BaseState = newState
    
    if not data.OverlayLock then
        fire(player, newState)
    end
end

--------------------------------------------------
-- OVERLAY / GLOBAL
--------------------------------------------------
local function playOverlay(player, data, overlayState, duration)
    data.OverlayLock = true
    data.OverlayState = overlayState
    
    fire(player, overlayState)
    
    task.delay(duration, function()
        if not data then return end
        data.OverlayLock = false
        data.OverlayState = nil
        
        -- REAVALIA ESTADO APÓS GLOBAL
        if data.Falling then
            fire(player, "Fall")
        else
            fire(player, data.BaseState)
        end
    end)
end

--------------------------------------------------
-- CHARACTER HANDLER
--------------------------------------------------
local function handleCharacterAdded(character)
    local player = Players:GetPlayerFromCharacter(character)
    if not player then return end
    
    local humanoid = character:WaitForChild("Humanoid")
    
    local data = {
    Humanoid = humanoid,
    BaseState = "Idle",
    OverlayState = nil,
    OverlayLock = false,
    Jumping = false,
    Falling = false,
    DashLock = false,
    DashCount = 0
    }
    
    CharacterData[player] = data
    fire(player, "Idle")
    
    humanoid.StateChanged:Connect(function(_, newState)
        if newState == Enum.HumanoidStateType.Jumping then
            data.Jumping = true
            data.Falling = false
            setBaseState(player, data, "Jump")
            
        elseif newState == Enum.HumanoidStateType.Freefall then
            data.Jumping = false
            data.Falling = true
            setBaseState(player, data, "Fall")
            
        elseif newState == Enum.HumanoidStateType.Landed then
            data.Jumping = false
            data.Falling = false
            
            if humanoid.MoveDirection.Magnitude > MOVEMENT_THRESHOLD then
                setBaseState(player, data, "Walk")
            else
                setBaseState(player, data, "Idle")
            end
        end
    end)
end

Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(handleCharacterAdded)
end)

for _, p in ipairs(Players:GetPlayers()) do
    if p.Character then
        handleCharacterAdded(p.Character)
    end
end

--------------------------------------------------
-- DASH (GLOBAL)
--------------------------------------------------
DashEvent.OnServerEvent:Connect(function(player)
    local data = CharacterData[player]
    if not data or data.DashLock then return end
    
    data.DashLock = true
    
    local dashState = (data.DashCount % 2 == 1) and "Dash2" or "Dash"
    data.DashCount += 1
    
    playOverlay(player, data, dashState, DASH_TIME)
    
    task.delay(DASH_TIME, function()
        if data then
            data.DashLock = false
        end
    end)
end)

--------------------------------------------------
-- LOOP BASE (IDLE / WALK)
--------------------------------------------------
RunSvc.Heartbeat:Connect(function()
    for player, data in pairs(CharacterData) do
        if not data.OverlayLock and not data.Jumping and not data.Falling then
            local hum = data.Humanoid
            if local speed = hum.WalkSpeed

if speed >= 20 then
    setBaseState(player, data, "Run")
elseif hum.MoveDirection.Magnitude > MOVEMENT_THRESHOLD then
    setBaseState(player, data, "Walk")
else
    setBaseState(player, data, "Idle")
end
            else
                setBaseState(player, data, "Idle")
            end
        end
    end
end)



