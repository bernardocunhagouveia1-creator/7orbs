-- Script de executor (LocalScript). Cole e execute com o jogo aberto. Nada vai para o servidor.
-- F voar, E explodir, T teleportar, C clonar, X apagar clones, M mudar forma, Q recolher,
-- B impulso (orbe dourada), N ponte (orbe verde), Delete descarregar tudo.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local StarterGui = game:GetService("StarterGui")
local Debris = game:GetService("Debris")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local SPHERE_COUNT = 5
local BOOST_INDEX = SPHERE_COUNT + 1
local BRIDGE_INDEX = SPHERE_COUNT + 2
local ORBIT_COUNT = SPHERE_COUNT + 2
local ORBIT_RADIUS = 5
local ORBIT_SPEED = 1.1
local FEET_OFFSET = 3
local FLY_SPEED = 70
local LAUNCH_SPEED = 150
local MAX_RANGE = 400
local MAX_CLONES = 3
local CLONE_LIFETIME = 120
local PUSH_RADIUS = 10
local PUSH_FORCE = 200
local PUSH_LIFT = 40
local PUSH_MIN_STRENGTH = 0.45
local PUSH_SPIN = 12
local PUSH_MAX_SIZE = 150
local PUSH_DISTANCE = 6
local PUSH_DURATION = 2
local MAX_PUSH_PARTS = 120
local FLOOR_MIN_SPAN = 12
local FLOOR_MAX_THICKNESS = 4
local FOOTING_DEPTH = 12
local BOOST_JUMP_POWER = 120
local BOOST_WALK_SPEED = 32
local BOOST_DURATION = 8
local BOOST_LAUNCH_SPEED = 90
local BOOST_FORWARD_SPEED = 25
local PLANK_SIZE = Vector3.new(5, 0.8, 5)
local PLANK_SPACING = 4.5
local PLANK_LIFETIME = 8
local PLANK_FADE_TIME = 0.8
local MAX_PLANKS = 40
local SIZES = { 1.4, 2.2, 1.0 }
local SHAPES = { Enum.PartType.Ball, Enum.PartType.Block, Enum.PartType.Cylinder }
local COLORS = {
Color3.fromRGB(120, 200, 255),
Color3.fromRGB(255, 120, 200),
Color3.fromRGB(160, 255, 140),
Color3.fromRGB(255, 220, 120),
Color3.fromRGB(200, 160, 255),
}
local COLOR_BOOST = Color3.fromRGB(255, 200, 80)
local COLOR_BRIDGE = Color3.fromRGB(120, 255, 200)

local character = player.Character or player.CharacterAdded:Wait()
local flying = false
local unloaded = false
local spheres = {}
local clones = {}
local anchoredPushes = {}
local connections = {}
local bridgeActive = false
local lastPlankPos = nil
local boostEndsAt = 0
local baseWalkSpeed = nil
local baseJumpPower = nil

local folder = Instance.new("Folder")
folder.Name = "OrbitSpheres"
folder.Parent = workspace

local cloneFolder = Instance.new("Folder")
cloneFolder.Name = "OrbitClones"
cloneFolder.Parent = workspace

local planksFolder = Instance.new("Folder")
planksFolder.Name = "BridgePlanks"
planksFolder.Parent = workspace

local function notify(text)
pcall(function()
StarterGui:SetCore("SendNotification", { Title = "Esferas", Text = text, Duration = 3 })
end)
end

local function getRoot()
return character and character:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
return character and character:FindFirstChildOfClass("Humanoid")
end

local function orbitPosition(root, index, now)
local angle = now ORBIT_SPEED + index (math.pi 2 / ORBIT_COUNT)
local bob = math.sin(now 2.5 + index) 0.6
return root.Position + Vector3.new(math.cos(angle) ORBIT_RADIUS, 3 + bob, math.sin(angle) *ORBIT_RADIUS)
end

local function makeOrb(name, color, size)
local part = Instance.new("Part")
part.Name = name
part.Shape = Enum.PartType.Ball
part.Size = Vector3.new(size, size, size)
part.Material = Enum.Material.Neon
part.Color = color
part.Anchored = true
part.CanCollide = false
part.CanTouch = false
part.CanQuery = false
part.CastShadow = false
part.Massless = true
part.TopSurface = Enum.SurfaceType.Smooth
part.BottomSurface = Enum.SurfaceType.Smooth
local root = getRoot()
part.CFrame = CFrame.new(root and root.Position or Vector3.new(0, 10, 0))
part.Parent = folder

local light = Instance.new("PointLight")
light.Color = color
light.Range = 10
light.Brightness = 1.5
light.Parent = part

return part

end

local function makeSphere(index)
local part = makeOrb("Sphere" .. index, COLORS[index], SIZES[1])
return {
part = part,
index = index,
state = "orbit",
mode = nil,
target = nil,
startedAt = 0,
size = SIZES[1],
sizeIndex = 1,
shapeIndex = 1,
}
end

local boostOrb = { part = makeOrb("BoostOrb", COLOR_BOOST, 1.6), index = BOOST_INDEX, state = "orbit" }
local bridgeOrb = { part = makeOrb("BridgeOrb", COLOR_BRIDGE, 1.6), index = BRIDGE_INDEX, state = "orbit" }
local specialOrbs = { boostOrb, bridgeOrb }

local function pulse(sphere)
local part = sphere.part
TweenService:Create(part, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
Size = Vector3.new(sphere.size* 1.5, sphere.size 1.5, sphere.size 1.5),
}):Play()
task.delay(0.12, function()
if part.Parent then
TweenService:Create(part, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
Size = Vector3.new(sphere.size, sphere.size, sphere.size),
}):Play()
end
end)
end

local function groundUnder(root)
local params = RaycastParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude
params.FilterDescendantsInstances = { folder, cloneFolder, planksFolder, character }
local hit = workspace:Raycast(root.Position, Vector3.new(0, -FOOTING_DEPTH, 0), params)
return hit and hit.Instance
end

local function isFloorLike(part, root)
local size = part.Size
local span = math.max(size.X, size.Z)
if span > FLOOR_MIN_SPAN and size.Y < FLOOR_MAX_THICKNESS then
return true
end
if root and span > FLOOR_MIN_SPAN and part.Position.Y + size.Y / 2 <= root.Position.Y - 2 then
return true
end
return false
end

local function isPushable(part, footing, root)
if not part.Parent or part:IsA("Terrain") then
return false
end
if part == footing or isFloorLike(part, root) then
return false
end
if part:IsDescendantOf(folder) or part:IsDescendantOf(cloneFolder) or part:IsDescendantOf(planksFolder) then
return false
end
if character and part:IsDescendantOf(character) then
return false
end
local model = part:FindFirstAncestorWhichIsA("Model")
if model and model:FindFirstChildOfClass("Humanoid") then
return false
end
local size = part.Size
return math.max(size.X, size.Y, size.Z) <= PUSH_MAX_SIZE
end

local function pushUnanchored(part, direction, strength)
local velocity = direction PUSH_FORCE strength + Vector3.new(0, PUSH_LIFT strength, 0)
local spin = Vector3.new(math.random() - 0.5, math.random() - 0.5, math.random() - 0.5) PUSH_SPIN *strength
pcall(function()
part.AssemblyLinearVelocity = velocity
part.AssemblyAngularVelocity = spin
end)
end

local function pushAnchored(part, direction, strength)
if anchoredPushes[part] then
return
end
anchoredPushes[part] = {
origin = part.CFrame,
direction = direction,
distance = PUSH_DISTANCE* strength,
startedAt = os.clock(),
}
end

local function shockwave(position)
local ring = Instance.new("Part")
ring.Shape = Enum.PartType.Ball
ring.Anchored = true
ring.CanCollide = false
ring.CanTouch = false
ring.CanQuery = false
ring.CastShadow = false
ring.Material = Enum.Material.Neon
ring.Color = COLORS[2]
ring.Transparency = 0.3
ring.Size = Vector3.new(2, 2, 2)
ring.CFrame = CFrame.new(position)
ring.Parent = folder
TweenService:Create(ring, TweenInfo.new(0.45, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
Size = Vector3.new(PUSH_RADIUS 2, PUSH_RADIUS 2, PUSH_RADIUS *2),
Transparency = 1,
}):Play()
Debris:AddItem(ring, 0.6)

local root = getRoot()
local footing = root and groundUnder(root)

local params = OverlapParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude
params.FilterDescendantsInstances = { folder, cloneFolder, planksFolder, character }

local candidates = {}
for _, part in ipairs(workspace:GetPartBoundsInRadius(position, PUSH_RADIUS, params)) do
	if isPushable(part, footing, root) then
		local offset = part.Position - position
		local distance = offset.Magnitude
		if distance > 0.1 then
			table.insert(candidates, { part = part, offset = offset, distance = distance })
		end
	end
end
table.sort(candidates, function(a, b)
	return a.distance < b.distance
end)

local pushed = 0
for _, candidate in ipairs(candidates) do
	if pushed >= MAX_PUSH_PARTS then
		break
	end
	local ratio = math.min(1, candidate.distance / PUSH_RADIUS)
	local strength = PUSH_MIN_STRENGTH + (1 - PUSH_MIN_STRENGTH)* (1 - ratio)
	local direction = candidate.offset.Unit
	if candidate.part.Anchored then
		pushAnchored(candidate.part, direction, strength)
	else
		pushUnanchored(candidate.part, direction, strength)
	end
	pushed += 1
end
return pushed

end

local function arrive(sphere)
local position = sphere.part.Position
sphere.state = "returning"
if sphere.mode == "explode" then
shockwave(position)
elseif sphere.mode == "teleport" then
local root = getRoot()
if root then
root.AssemblyLinearVelocity = Vector3.zero
root.CFrame = CFrame.new(position + Vector3.new(0, 3, 0))
end
end
sphere.mode = nil
pulse(sphere)
end

local function launch(mode)
local root = getRoot()
if not root then
return
end
local mouse = UserInputService:GetMouseLocation()
local ray = camera:ViewportPointToRay(mouse.X, mouse.Y)
local params = RaycastParams.new()
params.FilterType = Enum.RaycastFilterType.Exclude
params.FilterDescendantsInstances = { folder, cloneFolder, planksFolder, character }
local hit = workspace:Raycast(ray.Origin, ray.Direction MAX_RANGE, params)
local aim = hit and hit.Position or ray.Origin + ray.Direction 120

for _, sphere in ipairs(spheres) do
	if sphere.state == "orbit" then
		sphere.state = "flying"
		sphere.mode = mode
		sphere.target = aim
		sphere.startedAt = os.clock()
		return
	end
end

end

local function cloneCharacter()
local root = getRoot()
if not character or not root then
notify("Personagem não encontrado.")
return
end

for index = #clones, 1, -1 do
	if not clones[index].Parent then
		table.remove(clones, index)
	end
end
if #clones >= MAX_CLONES then
	notify("Limite de " .. MAX_CLONES .. " clones. Aperte X para apagar.")
	return
end

local archivableStates = {}
for _, descendant in ipairs(character:GetDescendants()) do
	archivableStates[descendant] = descendant.Archivable
	descendant.Archivable = true
end
local characterArchivable = character.Archivable
character.Archivable = true
local ok, copy = pcall(function()
	return character:Clone()
end)
character.Archivable = characterArchivable
for descendant, value in pairs(archivableStates) do
	if descendant.Parent then
		descendant.Archivable = value
	end
end

if not ok or not copy then
	notify("O jogo bloqueou a cópia do personagem.")
	return
end

local slot = #clones
for _, descendant in ipairs(copy:GetDescendants()) do
	if descendant:IsA("Script") or descendant:IsA("LocalScript") then
		descendant:Destroy()
	elseif descendant:IsA("BasePart") then
		descendant.Anchored = true
		descendant.CanCollide = false
		descendant.CanQuery = false
	end
end
copy.Name = player.Name .. "_Clone"
copy:PivotTo(root.CFrame *CFrame.new(slot* 4 - 2, 0, -7))
copy.Parent = cloneFolder
table.insert(clones, copy)
Debris:AddItem(copy, CLONE_LIFETIME)
notify("Clone " .. #clones .. " criado.")

end

local function clearClones()
for _, clone in ipairs(clones) do
clone:Destroy()
end
table.clear(clones)
end

local function morph(sphere, delay)
task.delay(delay, function()
local part = sphere.part
if not part.Parent then
return
end
TweenService:Create(part, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
Size = Vector3.new(0.2, 0.2, 0.2),
}):Play()
task.wait(0.18)
if not part.Parent then
return
end
sphere.shapeIndex = sphere.shapeIndex % #SHAPES + 1
sphere.sizeIndex = sphere.sizeIndex % #SIZES + 1
sphere.size = SIZES[sphere.sizeIndex]
part.Shape = SHAPES[sphere.shapeIndex]
part.Color = COLORS[(sphere.shapeIndex + sphere.index) % #COLORS + 1]
TweenService:Create(part, TweenInfo.new(0.6, Enum.EasingStyle.Elastic, Enum.EasingDirection.Out), {
Size = Vector3.new(sphere.size, sphere.size, sphere.size),
}):Play()
end)
end

local function flatDistance(a, b)
return (Vector3.new(a.X, 0, a.Z) - Vector3.new(b.X, 0, b.Z)).Magnitude
end

local function moveOrb(orb, cframe, duration, nextState)
orb.state = "busy"
local tween = TweenService:Create(orb.part, TweenInfo.new(duration, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
CFrame = cframe,
})
tween:Play()
task.spawn(function()
tween.Completed:Wait()
orb.state = nextState
end)
end

local function applyBoost(humanoid)
if not baseWalkSpeed then
baseWalkSpeed = humanoid.WalkSpeed
baseJumpPower = humanoid.JumpPower
end
humanoid.UseJumpPower = true
humanoid.WalkSpeed = BOOST_WALK_SPEED
humanoid.JumpPower = BOOST_JUMP_POWER
end

local function endBoost()
local humanoid = getHumanoid()
if humanoid and baseWalkSpeed then
humanoid.WalkSpeed = baseWalkSpeed
humanoid.JumpPower = baseJumpPower
end
baseWalkSpeed = nil
baseJumpPower = nil
boostEndsAt = 0
end

local function boost()
local root = getRoot()
local humanoid = getHumanoid()
if not root or not humanoid or boostOrb.state == "busy" then
return
end

local look = root.CFrame.LookVector
local flat = Vector3.new(look.X, 0, look.Z)
local forward = flat.Magnitude > 0 and flat.Unit or Vector3.zero

moveOrb(boostOrb, CFrame.new(root.Position - Vector3.new(0, 2, 0)), 0.15, "orbit")
task.delay(0.15, function()
	local current = getRoot()
	if current then
		local velocity = current.AssemblyLinearVelocity
		current.AssemblyLinearVelocity = Vector3.new(velocity.X, BOOST_LAUNCH_SPEED, velocity.Z) + forward *BOOST_FORWARD_SPEED
	end
end)

applyBoost(humanoid)
boostEndsAt = os.clock() + BOOST_DURATION

end

local function spawnPlank(position, yaw)
local planks = planksFolder:GetChildren()
if #planks >= MAX_PLANKS then
planks[1]:Destroy()
end

local tiny = Vector3.new(0.1, 0.1, 0.1)
local plank = Instance.new("Part")
plank.Name = "BridgePlank"
plank.Size = tiny
plank.Anchored = true
plank.CanCollide = true
plank.CanQuery = false
plank.CastShadow = false
plank.Material = Enum.Material.Neon
plank.Color = COLOR_BRIDGE
plank.Transparency = 1
plank.TopSurface = Enum.SurfaceType.Smooth
plank.BottomSurface = Enum.SurfaceType.Smooth
plank.CFrame = CFrame.new(position)* CFrame.Angles(0, yaw, 0)
plank.Parent = planksFolder

TweenService:Create(plank, TweenInfo.new(0.6, Enum.EasingStyle.Elastic, Enum.EasingDirection.Out), {
	Size = PLANK_SIZE,
	Transparency = 0.2,
}):Play()

task.delay(PLANK_LIFETIME, function()
	if not plank.Parent then
		return
	end
	local fade = TweenService:Create(plank, TweenInfo.new(PLANK_FADE_TIME, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Size = tiny,
		Transparency = 1,
	})
	fade:Play()
	fade.Completed:Wait()
	plank:Destroy()
end)

end

local function toggleBridge()
local root = getRoot()
if not root then
return
end

bridgeActive = not bridgeActive
if bridgeActive then
	lastPlankPos = nil
	moveOrb(bridgeOrb, CFrame.new(root.Position - Vector3.new(0, 3, 0)), 0.3, "bridge")
else
	moveOrb(bridgeOrb, CFrame.new(root.Position + Vector3.new(0, 3, 0)), 0.3, "orbit")
end

end

for index = 1, SPHERE_COUNT do
table.insert(spheres, makeSphere(index))
end

local function updateSpheres(root, now)
for _, sphere in ipairs(spheres) do
local part = sphere.part
if sphere.state == "orbit" then
part.CFrame = CFrame.new(orbitPosition(root, sphere.index, now))
elseif sphere.state == "flying" then
local offset = sphere.target - part.Position
if offset.Magnitude < 1.5 or now - sphere.startedAt > 4 then
arrive(sphere)
else
local dt = RunService.Heartbeat:Wait()
part.CFrame = CFrame.new(part.Position + offset.Unit math.min(offset.Magnitude, LAUNCH_SPEED dt))
end
elseif sphere.state == "returning" then
local target = orbitPosition(root, sphere.index, now)
part.CFrame = part.CFrame:Lerp(CFrame.new(target), math.min(1, 0.1 *8))
if (part.Position - target).Magnitude < 1.5 then
sphere.state = "orbit"
end
end
end
end

local function updatePushes(now)
for part, push in pairs(anchoredPushes) do
if not part.Parent then
anchoredPushes[part] = nil
else
local t = math.min(1, (now - push.startedAt) / PUSH_DURATION)
local eased = 1 - (1 - t) ^ 2
part.CFrame = push.origin + push.direction* push.distance *eased
if t >= 1 then
anchoredPushes[part] = nil
end
end
end
end

local function updateFlight(root, humanoid)
local camCF = camera.CFrame
local dir = Vector3.zero
if UserInputService:IsKeyDown(Enum.KeyCode.W) then
dir += camCF.LookVector
end
if UserInputService:IsKeyDown(Enum.KeyCode.S) then
dir -= camCF.LookVector
end
if UserInputService:IsKeyDown(Enum.KeyCode.D) then
dir += camCF.RightVector
end
if UserInputService:IsKeyDown(Enum.KeyCode.A) then
dir -= camCF.RightVector
end
if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
dir += Vector3.yAxis
end
if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
dir -= Vector3.yAxis
end
if dir.Magnitude > 0 then
dir = dir.Unit* FLY_SPEED
end
root.AssemblyLinearVelocity = dir
end

local function updateBridge(root, humanoid)
if bridgeActive and humanoid and humanoid.MoveDirection.Magnitude > 0 then
local planted = Vector3.new(root.Position.X, root.Position.Y - FEET_OFFSET - PLANK_SIZE.Y / 2, root.Position.Z)
if not lastPlankPos or flatDistance(planted, lastPlankPos) >= PLANK_SPACING then
spawnPlank(planted, math.rad(root.Orientation.Y))
lastPlankPos = planted
end
end
end

local function step(dt)
local now = os.clock()
local root = getRoot()
if not root then
return
end
local humanoid = getHumanoid()

if boostEndsAt > 0 and now >= boostEndsAt then
	endBoost()
end

updateSpheres(root, now)
updatePushes(now)

for _, orb in ipairs(specialOrbs) do
	if orb.state == "orbit" then
		orb.part.CFrame = CFrame.new(orbitPosition(root, orb.index, now))
	end
end
if bridgeOrb.state == "bridge" and lastPlankPos then
	bridgeOrb.part.CFrame = CFrame.new(lastPlankPos + Vector3.new(0, 1.5, 0))
end

if flying then
	updateFlight(root, humanoid)
end

updateBridge(root, humanoid)

end

local function onInput(input, gameProcessed)
if gameProcessed then
return
end
local key = input.KeyCode
if key == Enum.KeyCode.F then
flying = not flying
local humanoid = getHumanoid()
if humanoid then
humanoid.PlatformStand = flying
end
elseif key == Enum.KeyCode.E then
launch("explode")
elseif key == Enum.KeyCode.T then
launch("teleport")
elseif key == Enum.KeyCode.C then
cloneCharacter()
elseif key == Enum.KeyCode.X then
clearClones()
elseif key == Enum.KeyCode.M then
for position, sphere in ipairs(spheres) do
morph(sphere, (position - 1) * 0.07)
end
elseif key == Enum.KeyCode.Q then
for _, sphere in ipairs(spheres) do
if sphere.state == "flying" then
sphere.mode = nil
sphere.state = "returning"
end
end
elseif key == Enum.KeyCode.B then
boost()
elseif key == Enum.KeyCode.N then
toggleBridge()
elseif key == Enum.KeyCode.Delete then
unload()
end
end

local function onCharacterAdded(newCharacter)
character = newCharacter
flying = false
bridgeActive = false
lastPlankPos = nil
boostEndsAt = 0
baseWalkSpeed = nil
baseJumpPower = nil
boostOrb.state = "orbit"
bridgeOrb.state = "orbit"
end

function unload()
if unloaded then
return
end
unloaded = true
for _, connection in ipairs(connections) do
connection:Disconnect()
end
table.clear(connections)
clearClones()
anchoredPushes = {}
endBoost()
local humanoid = getHumanoid()
if humanoid then
humanoid.PlatformStand = false
end
folder:Destroy()
cloneFolder:Destroy()
planksFolder:Destroy()
notify("Esferas descarregadas.")
end

table.insert(connections, RunService.Heartbeat:Connect(function(dt)
local ok, err = pcall(step, dt)
if not ok then
warn("[Esferas] " .. tostring(err))
end
end))

table.insert(connections, UserInputService.InputBegan:Connect(function(input, gameProcessed)
local ok, err = pcall(onInput, input, gameProcessed)
if not ok then
warn("[Esferas] " .. tostring(err))
end
end))
