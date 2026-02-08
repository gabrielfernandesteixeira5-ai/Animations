---------------------------------------------------------------------
-- CLIENT PROFESSIONAL ANIMATION HANDLER (BASE + GLOBAL g_)
-- MULTIPLAYER REAL FIX | NADA REMOVIDO | R6 | 300+ g_
---------------------------------------------------------------------

---------------------------------------------------------------------
-- SERVICES
---------------------------------------------------------------------
local RunSvc = game:GetService("RunService")
local RepStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")

---------------------------------------------------------------------
-- REMOTE
---------------------------------------------------------------------
local PlayAnimationState = RepStorage:WaitForChild("PlayAnimation")

---------------------------------------------------------------------
-- CONFIG
---------------------------------------------------------------------
local FRAME_DURATION = 0.1
local FADE_TIME = 0.15
local EASING_STYLE = "Sine"
local RUN_SPEED_THRESHOLD = 20

---------------------------------------------------------------------
-- SPEED TABLE
---------------------------------------------------------------------
local AnimationSpeed = {
	Idle  = 0.45,
	Walk  = 1,
	Run   = 1,
	Jump  = 0.45,
	Fall  = 0.45,
	Climb = 1,
	Sit   = 1,
	Dash  = 0.45,
	Dash2 = 0.45,
}

---------------------------------------------------------------------
-- EASING
---------------------------------------------------------------------
local Easing = {
	Linear = function(t) return t end,
	Quad   = function(t) return t*t end,
	Cubic  = function(t) return t^3 end,
	Quart  = function(t) return t^4 end,
	Quint  = function(t) return t^5 end,
	Sine   = function(t) return 1 - math.cos(t * math.pi/2) end,
}
local Ease = Easing[EASING_STYLE] or Easing.Linear

---------------------------------------------------------------------
-- ANIMS
---------------------------------------------------------------------
local AnimFolder = RepStorage:WaitForChild("Animations")

local AnimSequences = {
	Idle  = AnimFolder:FindFirstChild("IdleAnim"),
	Walk  = AnimFolder:FindFirstChild("WalkAnim"),
	Run   = AnimFolder:FindFirstChild("RunAnim"),
	Jump  = AnimFolder:FindFirstChild("JumpAnim"),
	Fall  = AnimFolder:FindFirstChild("FallAnim"),
	Climb = AnimFolder:FindFirstChild("ClimbAnim"),
	Sit   = AnimFolder:FindFirstChild("SitAnim"),
	Dash  = AnimFolder:FindFirstChild("DashAnim"),
	Dash2 = AnimFolder:FindFirstChild("DashAnim2"),
}

---------------------------------------------------------------------
-- R6 SEQUENCE PLAYER (INALTERADO)
---------------------------------------------------------------------
local R6SequencePlayer = {}
R6SequencePlayer.__index = R6SequencePlayer

function R6SequencePlayer.new(character, sequence, isLooping)
	local self = setmetatable({}, R6SequencePlayer)
	self.Character = character
	self.Sequence = sequence
	self.IsLooping = isLooping
	self.MotorCache = {}
	self.Keyframes = {}
	self.CurrentIndex = 1
	self.TimeAccumulator = 0
	self.IsPlaying = false
	self.FadeAlpha = 0
	self.FadeGoal = 0
	return self
end

function R6SequencePlayer:FindMotor(pose)
	for _, m in ipairs(self.Character:GetDescendants()) do
		if m:IsA("Motor6D") and m.Part1.Name:lower() == pose.Name:lower() then
			return m
		end
	end
end

function R6SequencePlayer:Load()
	local kfs = self.Sequence:GetKeyframes()
	table.sort(kfs, function(a,b) return a.Time < b.Time end)

	local cache = {}
	self.Keyframes = {}

	for i, kf in ipairs(kfs) do
		local poses = {}
		for _, p in ipairs(kf:GetDescendants()) do
			if p:IsA("Pose") and p.Weight > 0 then
				local key = p.Name.."."..p.Parent.Name
				if not cache[key] then
					cache[key] = self:FindMotor(p)
					self.MotorCache[key] = cache[key]
				end
				if cache[key] then
					poses[key] = p
				end
			end
		end
		self.Keyframes[i] = poses
	end
end

function R6SequencePlayer:SetPlaying(state)
	self.IsPlaying = state
	self.FadeGoal = state and 1 or 0
end

function R6SequencePlayer:Play()
	if not self.IsPlaying then
		self.CurrentIndex = 1
		self.TimeAccumulator = 0
		self.FadeAlpha = 0
		self:SetPlaying(true)
	end
end

function R6SequencePlayer:Stop()
	self:SetPlaying(false)
end

function R6SequencePlayer:GetAnimName()
	return self.Sequence.Name:gsub("Anim","")
end

function R6SequencePlayer:Step(dt)
	if self.FadeAlpha ~= self.FadeGoal then
		local step = dt / FADE_TIME
		self.FadeAlpha += (self.FadeGoal > self.FadeAlpha and step or -step)
		self.FadeAlpha = math.clamp(self.FadeAlpha, 0, 1)
	end

	if self.FadeAlpha <= 0.001 and not self.IsPlaying then return end

	local posesA = self.Keyframes[self.CurrentIndex]
	if not posesA then return end

	local speed = AnimationSpeed[self:GetAnimName()] or 1
	self.TimeAccumulator += dt

	local nextIndex = self.CurrentIndex + 1
	if nextIndex > #self.Keyframes then
		nextIndex = self.IsLooping and 1 or self.CurrentIndex
	end

	local posesB = self.Keyframes[nextIndex]
	local t = math.clamp(self.TimeAccumulator / (FRAME_DURATION / speed), 0, 1)
	local alpha = Ease(self.FadeAlpha)

	for key, poseA in pairs(posesA) do
		local motor = self.MotorCache[key]
		if motor then
			local blended = posesB and posesB[key]
				and poseA.CFrame:Lerp(posesB[key].CFrame, t)
				or poseA.CFrame
			motor.Transform = motor.Transform:Lerp(blended, alpha)
		end
	end

	if t >= 1 then
		self.CurrentIndex = nextIndex
		self.TimeAccumulator = 0
	end
end

---------------------------------------------------------------------
-- CHARACTER DATA (MULTIPLAYER REAL)
---------------------------------------------------------------------
local CharacterData = {}

local function DisableDefaultAnims(char)
	local h = char:FindFirstChild("Humanoid")
	if h then
		local a = h:FindFirstChildOfClass("Animator")
		if a then a:Destroy() end
	end
	local anim = char:FindFirstChild("Animate")
	if anim then anim:Destroy() end
end

local function SetupCharacter(char)
	DisableDefaultAnims(char)

	local hum = char:WaitForChild("Humanoid")

	local data = {
		Character = char,
		Humanoid = hum,
		Airborne = false,
		GlobalLock = false,
		SpecialStateLock = false,

		DashLockId = 0,
		ActionLock = false,
		BaseAnims = {},
		GlobalAnims = {},
	}

	for name, seq in pairs(AnimSequences) do
		if seq then
			local loop = not (name=="Dash" or name=="Dash2" or name=="Jump" or name=="Fall")
			local p = R6SequencePlayer.new(char, seq, loop)
			p:Load()
			data.BaseAnims[name] = p
		end
	end

	for _, seq in ipairs(AnimFolder:GetChildren()) do
		if seq:IsA("KeyframeSequence") and seq.Name:sub(1,2) == "g_" then
			local p = R6SequencePlayer.new(char, seq, false)
			p:Load()
			data.GlobalAnims[seq.Name] = p
		end
	end

	CharacterData[char] = data
end

---------------------------------------------------------------------
-- APPLY BASE
---------------------------------------------------------------------
local function ApplyState(data, state)
	if data.ActionLock or data.GlobalLock then return end
	for name, p in pairs(data.BaseAnims) do
		if name ~= state then p:Stop() end
	end
	if data.BaseAnims[state] then
		data.BaseAnims[state]:Play()
	end
end

---------------------------------------------------------------------
-- GLOBAL API (MULTIPLAYER)
---------------------------------------------------------------------
_G.PlayGlobal = function(character, name)
	local data = CharacterData[character]
	if not data then return end

	local anim = data.GlobalAnims["g_"..name]
	if not anim then return end

	data.GlobalLock = true
	data.SpecialStateLock = true

	for _, p in pairs(data.BaseAnims) do p:Stop() end
	anim:Play()
end

_G.StopGlobal = function(character, name)
	local data = CharacterData[character]
	if not data then return end

	local anim = data.GlobalAnims["g_"..name]
	if anim then anim:Stop() end

	data.GlobalLock = false
	data.SpecialStateLock = false

	if not data.Airborne then
		ApplyState(data, data.Humanoid.MoveDirection.Magnitude > 0.01 and "Walk" or "Idle")
	end
end

---------------------------------------------------------------------
-- HEARTBEAT
---------------------------------------------------------------------
RunSvc.Heartbeat:Connect(function(dt)
	for _, data in pairs(CharacterData) do
		for _, p in pairs(data.BaseAnims) do
			p:Step(dt)
		end
		for _, p in pairs(data.GlobalAnims) do
			if p.IsPlaying or p.FadeAlpha > 0 then
				p:Step(dt)
			end
		end
	end
end)

---------------------------------------------------------------------
-- SERVER STATE EVENTS
---------------------------------------------------------------------
PlayAnimationState.OnClientEvent:Connect(function(player, state)
	local char = player.Character
	local data = char and CharacterData[char]
	if not data then return end

	if state == "Dash" or state == "Dash2" then
		data.DashLockId += 1
		local id = data.DashLockId
		data.SpecialStateLock = true
		ApplyState(data, state)

		task.delay(0.45, function()
			if data.DashLockId == id then
				data.SpecialStateLock = false
			end
		end)

	elseif state == "Jump" or state == "Fall" then
		data.SpecialStateLock = true
		data.Airborne = true
		ApplyState(data, state)

	else
    data.SpecialStateLock = false
    data.Airborne = false
    ApplyState(data, state)
end
end)

---------------------------------------------------------------------
-- INIT
---------------------------------------------------------------------
local function OnCharacter(char)
	SetupCharacter(char)
	ApplyState(CharacterData[char], "Idle")
end

Players.PlayerAdded:Connect(function(p)
	p.CharacterAdded:Connect(OnCharacter)
end)

for _, p in ipairs(Players:GetPlayers()) do
	if p.Character then
		OnCharacter(p.Character)
	end
end
