local SOURCE = [=[
local Rayfield = loadstring(game:HttpGet("https://sirius.menu/gen2"))()
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer
pcall(function()
	if type(getgenv().ThaBooSource) ~= "string" and isfile and isfile("ThaBoo/Dimensions.lua") then
		getgenv().ThaBooSource = readfile("ThaBoo/Dimensions.lua")
	end
end)
if getgenv().ThaBooRunning then
	return
end
getgenv().ThaBooRunning = true
local Window = Rayfield:CreateWindow({
	name = "Tha Boo | Dimensions",
	subtitle = "Farm + Lobby",
	sidebarLayout = true,
	theme = "cobalt",
	configuration = {
		autoSave = true,
		autoLoad = true,
		fileName = "Dimensions",
		customFolder = "ThaBoo",
	},
})
local Farm = Window:CreateTab({ name = "Farm", icon = 4483362458 })
local Lobby = Window:CreateTab({ name = "Lobby", icon = 4483362458 })
local Priority = Window:CreateTab({ name = "Priority", icon = 4483362458 })
local Misc = Window:CreateTab({ name = "Misc", icon = 4483362458 })
local alive = true
local conns = {}
local autoFarm = false
local autoDodge = false
local autoRetry = false
local autoReconnect = false
local reconnecting = false
local distance = 6
local method = "Behind"
local selected = { Basic = true }
local currentMob
local hardcore = false
local friendsOnly = false
local nextAttack = 0
local nextBasic = 0
local dodgePos
local threats = {}
local debrisHooked = false
local cachedBoss
local bossScan = 0
local startDelay = 0
local circleAngle = 0
local CIRCLE_SPEED = 1.8
local autoRaid = false
local autoSpeedRaid = false
local autoBossRush = false
local autoExtreme = false
local priorityEnabled = false
local lastRaidAttempt = 0
local lastSpeedAttempt = 0
local lastBossAttempt = 0
local leaveAfter = 0
local runsDone = 0
local leaveAfterLoaded = false
local lastResultCount = 0
local priorities = { Dimension = 4, SpeedRaid = 5, BossRush = 6, Raid = 8 }
local usedMains = {}
local RAID_LOBBY_ID = 7338881230
local afterSpeedLoadout = 0
local FARM_SPEED = 80
local DODGE_SPEED = 150
local FARM_SMOOTH_TIME = 0.08
local DODGE_SMOOTH_TIME = 0.05
local FARM_GOAL_RESPONSE = 26
local DODGE_GOAL_RESPONSE = 32
local TURN_RESPONSE = 18
local MELEE_DODGE_RANGE = 14
local function track(conn)
	table.insert(conns, conn)
	return conn
end
local function flatLook(fromPos, toPos, fallback)
	local dir = Vector3.new(toPos.X - fromPos.X, 0, toPos.Z - fromPos.Z)
	if dir.Magnitude < 0.2 then
		return fallback
	end
	return dir.Unit
end
local motionGoal
local motionVelocity = Vector3.zero
local motionRoot
local function resetMotion()
	motionGoal = nil
	motionVelocity = Vector3.zero
	motionRoot = nil
end
local function smoothDamp(current, target, velocity, smoothTime, topSpeed, dt)
	smoothTime = math.max(0.04, smoothTime)
	local omega = 2 / smoothTime
	local x = omega * dt
	local decay = 1 / (1 + x + 0.48 * x * x + 0.235 * x * x * x)
	local change = current - target
	local maxChange = topSpeed * smoothTime
	if change.Magnitude > maxChange then
		change = change.Unit * maxChange
	end
	local adjustedTarget = current - change
	local temp = (velocity + omega * change) * dt
	local nextVelocity = (velocity - omega * temp) * decay
	local nextPosition = adjustedTarget + (change + temp) * decay
	if (target - current):Dot(nextPosition - target) > 0 then
		nextPosition = target
		nextVelocity = Vector3.zero
	end
	return nextPosition, nextVelocity
end
local function stepToward(root, goalPos, lookAt, topSpeed, dt, dodging)
	dt = math.clamp(dt, 1 / 240, 0.05)
	if motionRoot ~= root then
		resetMotion()
		motionRoot = root
		motionGoal = root.Position
	end
	local response = dodging and DODGE_GOAL_RESPONSE or FARM_GOAL_RESPONSE
	local goalAlpha = 1 - math.exp(-response * dt)
	motionGoal = motionGoal:Lerp(goalPos, goalAlpha)
	local smoothTime = dodging and DODGE_SMOOTH_TIME or FARM_SMOOTH_TIME
	local nextPos, nextVelocity = smoothDamp(
		root.Position, motionGoal, motionVelocity, smoothTime, topSpeed, dt
	)
	motionVelocity = nextVelocity
	if (nextPos - root.Position).Magnitude < 0.004 then
		nextPos = root.Position
	end
	local currentLook = flatLook(Vector3.zero, root.CFrame.LookVector, Vector3.new(0, 0, -1))
	local wantedLook = flatLook(nextPos, lookAt, currentLook)
	local currentFrame = CFrame.lookAt(nextPos, nextPos + currentLook)
	local wantedFrame = CFrame.lookAt(nextPos, nextPos + wantedLook)
	local turnAlpha = 1 - math.exp(-TURN_RESPONSE * dt)
	root.CFrame = currentFrame:Lerp(wantedFrame, turnAlpha)
	root.AssemblyLinearVelocity = Vector3.zero
	root.AssemblyAngularVelocity = Vector3.zero
end
local dimensionReqs = {
	["Titan Dimension"] = { Easy = 1, Hard = 6, Nightmare = 11 },
	["Demon Dimension"] = { Easy = 16, Hard = 21, Nightmare = 26 },
	["Curse Dimension"] = { Easy = 31, Hard = 36, Nightmare = 41 },
	["Villain Dimension"] = { Easy = 46, Hard = 51, Nightmare = 56 },
	["Sword Dimension"] = { Easy = 61, Nightmare = 68 },
	["Ghoul Dimension"] = { Easy = 76, Nightmare = 83 },
	["Fate Dimension"] = { Easy = 91, Nightmare = 98 },
	["Slime Dimension"] = { Easy = 106, Nightmare = 113 },
	["Devil Dimension"] = { Nightmare = 121 },
	["Pirate Dimension"] = { Nightmare = 131 },
	["Alien Dimension"] = { Nightmare = 141 },
	["Level Dimension"] = { Nightmare = 151 },
	["Death Dimension"] = { Nightmare = 161 },
	["Halloween Dimension"] = { Nightmare = 20 },
}
local raidNames = {
	"Black Pasta Raid", "Blue Devil Raid", "Buff Boy Raid", "Chainsaw Raid", "Christmas Raid",
	"Combat Titan Raid", "Cosmic Wolfman Raid", "Cursed Sage Raid", "Dark One Hero Raid",
	"Demon Lord Raid", "Divine Guardian Raid", "Esper Raid", "Gear 5 Fluffy Raid", "Halloween Raid",
	"Hirito Raid", "Ichini Fullbring Raid", "Infinity Nojo Raid", "Joto Raid", "King of Curses Raid",
	"Matsuri Raid", "Nardo Beast Raid", "One Hero 100 Raid", "Psycho Student Raid", "Red Emperor Raid",
	"Roku Ultra Instinct Raid", "Shadow Christmas Raid", "Shadow Combat Titan Raid", "Shadow Demon Lord Raid",
	"Shadow Esper Raid", "Shadow Nojo Raid", "Shadow Shanks Raid", "Shadow Tengu Raid", "Shadow Yomiichi Raid",
	"Tengoku Raid", "Tengu Raid", "Titan Raid", "Tonjuro Sun God Raid", "Vio Raid", "Warlord Raid", "Yomiichi Raid",
}
local dimensionOptions = { "Hardest" }
for name in pairs(dimensionReqs) do
	table.insert(dimensionOptions, name)
end
table.sort(dimensionOptions, function(a, b)
	if a == "Hardest" then return true end
	if b == "Hardest" then return false end
	return a < b
end)
local dimensionPick = "Hardest"
local dimensionDifficulty = "Hardest"
local raidPick = raidNames[1]
local raidDifficulty = "Easy"
local extremeDifficulty = "Extreme"
local diffRank = { Easy = 1, Hard = 2, Nightmare = 3 }
local function notify(text)
	Window:Notify({ title = "Tha Boo", content = tostring(text), duration = 4 })
end
local function flagOn(name)
	local ok, text = pcall(function()
		return isfile and isfile("ThaBoo/" .. name .. ".txt") and readfile("ThaBoo/" .. name .. ".txt")
	end)
	return ok and text == "1"
end
local function setFlag(name, on)
	pcall(function()
		if makefolder and not (isfolder and isfolder("ThaBoo")) then
			makefolder("ThaBoo")
		end
		writefile("ThaBoo/" .. name .. ".txt", on and "1" or "0")
	end)
end
local function saveFarmConfig()
	pcall(function()
		if makefolder and not (isfolder and isfolder("ThaBoo")) then
			makefolder("ThaBoo")
		end
		local data = {
			dimensionPick = dimensionPick,
			dimensionDifficulty = dimensionDifficulty,
			raidPick = raidPick,
			raidDifficulty = raidDifficulty,
			extremeDifficulty = extremeDifficulty,
			hardcore = hardcore,
			friendsOnly = friendsOnly,
		}
		writefile("ThaBoo/FarmConfig.json", HttpService:JSONEncode(data))
	end)
end
local function loadFarmConfig()
	pcall(function()
		if isfile and isfile("ThaBoo/FarmConfig.json") then
			local raw = readfile("ThaBoo/FarmConfig.json")
			local data = HttpService:JSONDecode(raw)
			if data.dimensionPick then dimensionPick = data.dimensionPick end
			if data.dimensionDifficulty then dimensionDifficulty = data.dimensionDifficulty end
			if data.raidPick then raidPick = data.raidPick end
			if data.raidDifficulty then raidDifficulty = data.raidDifficulty end
			if data.extremeDifficulty then extremeDifficulty = data.extremeDifficulty end
			if data.hardcore ~= nil then hardcore = data.hardcore end
			if data.friendsOnly ~= nil then friendsOnly = data.friendsOnly end
		end
	end)
end
loadFarmConfig()
local function saveRuns()
	pcall(function()
		if makefolder and not (isfolder and isfolder("ThaBoo")) then
			makefolder("ThaBoo")
		end
		writefile("ThaBoo/runsDone.txt", tostring(runsDone))
	end)
	getgenv().ThaBooRunsDone = runsDone
end
local function loadRuns()
	local fromFile = 0
	pcall(function()
		if isfile and isfile("ThaBoo/runsDone.txt") then
			fromFile = tonumber(readfile("ThaBoo/runsDone.txt")) or 0
		end
	end)
	local fromGen = tonumber(getgenv().ThaBooRunsDone) or 0
	runsDone = math.max(fromFile, fromGen)
	if runsDone > 0 then
		notify("Loaded runs: " .. runsDone)
	end
end
loadRuns()
local function saveUsedMains()
	pcall(function()
		if makefolder and not (isfolder and isfolder("ThaBoo")) then
			makefolder("ThaBoo")
		end
		writefile("ThaBoo/usedMains.json", HttpService:JSONEncode(usedMains))
	end)
end
local function loadUsedMains()
	pcall(function()
		if isfile and isfile("ThaBoo/usedMains.json") then
			local data = HttpService:JSONDecode(readfile("ThaBoo/usedMains.json"))
			if type(data) == "table" then
				usedMains = data
			end
		end
	end)
end
loadUsedMains()
local function getStat(name)
	local ok, stats = pcall(function()
		return require(LocalPlayer.PlayerGui:WaitForChild("Modules"):WaitForChild("LocalStatsModule"))
	end)
	if ok and stats then
		local v = stats.returnStat(LocalPlayer, name)
		if typeof(v) == "Instance" then v = v.Value end
		return tonumber(v) or v or 0
	end
	return 0
end
local function getOwnedCharacters()
	local ok, mod = pcall(function()
		return require(LocalPlayer.PlayerGui:WaitForChild("Modules"):WaitForChild("CharacterModule"))
	end)
	if ok and mod and mod.returnCharacterCollectedTable then
		return mod.returnCharacterCollectedTable(true) or {}
	end
	return {}
end
local function pickSpeedCharacter()
	local owned = getOwnedCharacters()
	local current = getStat("CharacterName")
	if type(current) == "string" and current ~= "" and owned[current] and not usedMains[current] then
		usedMains[current] = true
		saveUsedMains()
		return current
	end
	for name in pairs(owned) do
		if not usedMains[name] then
			usedMains[name] = true
			saveUsedMains()
			return name
		end
	end
	return nil
end
local function speedRaidReady()
	local timer = getStat("ShadowRaidTimer") or 0
	local now = workspace:GetServerTimeNow()
	return (now - timer) >= 1800
end
local function bossRushHasTickets()
	local free = getStat("BossRushFreeEntry") or 0
	local tickets = getStat("BossRushEntryPass") or 0
	return free > 0 or tickets > 0
end
local function highestAvailable()
	local best, score = nil, -1
	if autoRaid and raidIsOpen() and priorities.Raid > score then
		best, score = "Raid", priorities.Raid
	end
	if autoSpeedRaid and speedRaidReady() and priorities.SpeedRaid > score then
		best, score = "SpeedRaid", priorities.SpeedRaid
	end
	if autoBossRush and bossRushHasTickets() and priorities.BossRush > score then
		best, score = "BossRush", priorities.BossRush
	end
	if priorities.Dimension > score then
		best, score = "Dimension", priorities.Dimension
	end
	return best
end
local function queueFunc()
	if type(queue_on_teleport) == "function" then
		return queue_on_teleport
	end
	if type(queueonteleport) == "function" then
		return queueonteleport
	end
	local genv = getgenv()
	if type(genv.queue_on_teleport) == "function" then
		return genv.queue_on_teleport
	end
	if syn and type(syn.queue_on_teleport) == "function" then
		return syn.queue_on_teleport
	end
	if fluxus and type(fluxus.queue_on_teleport) == "function" then
		return fluxus.queue_on_teleport
	end
	if krnl and type(krnl.queue_on_teleport) == "function" then
		return krnl.queue_on_teleport
	end
end
local function armTeleport(silent)
	local queue = queueFunc()
	if not queue then
		if not silent then
			notify("This executor has no queue_on_teleport")
		end
		return
	end
	local body = table.concat({
		"repeat task.wait() until game:IsLoaded()",
		"task.wait(2)",
		"local plr = game:GetService('Players').LocalPlayer",
		"local t0 = os.clock()",
		"while plr and not plr.Character and os.clock() - t0 < 10 do task.wait(0.2) end",
		"task.wait(1)",
		"if getgenv().ThaBooRunning then return end",
		"if not (isfile and readfile and isfile('ThaBoo/exec_on_tp.txt') and readfile('ThaBoo/exec_on_tp.txt') == '1') then return end",
		"loadstring(game:HttpGet('https://raw.githubusercontent.com/S1n-script/Demension/refs/heads/main/README.md'))()",
	}, "\n")
	local ok, err = pcall(queue, body)
	if silent then
		return
	end
	if ok then
		notify("Queued. It will pull the latest script after teleport.")
	else
		notify("Queue failed: " .. tostring(err))
	end
end
local function refreshQueue()
	if flagOn("exec_on_tp") then
		armTeleport(true)
	end
end
local function playerLevel()
	local folder = ReplicatedStorage:FindFirstChild(LocalPlayer.Name .. "StatDisplay")
	if folder then
		local obj = folder:FindFirstChild("Level")
		if obj and obj:IsA("ValueBase") then
			return tonumber(obj.Value) or 1
		end
	end
	local ok, stats = pcall(function()
		return require(LocalPlayer.PlayerGui:WaitForChild("Modules"):WaitForChild("LocalStatsModule"))
	end)
	if ok and stats then
		local level = stats.returnStat(LocalPlayer, "Level")
		if typeof(level) == "Instance" then
			level = level.Value
		end
		return tonumber(level) or 1
	end
	return 1
end
local function resolveDimension(map, diff)
	local level = playerLevel()
	if map == "Hardest" and diff == "Hardest" then
		local bestMap, bestDiff, bestReq
		for name, reqs in pairs(dimensionReqs) do
			for difficulty, req in pairs(reqs) do
				if level >= req and (not bestReq or req > bestReq) then
					bestMap, bestDiff, bestReq = name, difficulty, req
				end
			end
		end
		return bestMap, bestDiff, level
	end
	if map == "Hardest" then
		local bestMap, bestReq
		for name, reqs in pairs(dimensionReqs) do
			local req = reqs[diff]
			if req and level >= req and (not bestReq or req > bestReq) then
				bestMap, bestReq = name, req
			end
		end
		return bestMap, diff, level
	end
	if diff == "Hardest" then
		local bestDiff, bestRank
		for difficulty, req in pairs(dimensionReqs[map] or {}) do
			local rank = diffRank[difficulty] or 0
			if level >= req and (not bestRank or rank > bestRank) then
				bestDiff, bestRank = difficulty, rank
			end
		end
		return map, bestDiff, level
	end
	return map, diff, level
end
local function invoke(name, ...)
	if tostring(name):find("Teleport") then
		refreshQueue()
	end
	local remote = ReplicatedStorage:WaitForChild("RemoteFunctions"):WaitForChild("MainRemoteFunction")
	local args = { ... }
	local ok, result = pcall(function()
		return remote:InvokeServer(name, table.unpack(args))
	end)
	if not ok then
		notify(result)
		return nil
	end
	if type(result) == "table" and result.status == "Fail" then
		notify(result.message or "Failed")
		return nil
	end
	return result or true
end
local function startMap(mapName, difficulty, useHardcore)
	refreshQueue()
	if not invoke("CreateRoom", {
		MapName = mapName,
		Difficulty = difficulty,
		Hardcore = useHardcore and true or false,
		FriendsOnly = friendsOnly and true or false,
	}) then
		return false
	end
	task.wait(0.4)
	invoke("TeleportPlayers")
	return true
end
local function equipLoadout(index)
	if index < 1 or index > 3 then return end
	pcall(function()
		invoke("SwitchLoadout", index)
		notify("Equipped Loadout " .. index)
	end)
end
local function getHumanoid()
	local character = LocalPlayer.Character
	return character and character:FindFirstChildOfClass("Humanoid")
end
local function getRoot()
	local character = LocalPlayer.Character
	return character and character:FindFirstChild("HumanoidRootPart")
end
local cachedSkillModule
local function getSkillModule()
	if cachedSkillModule then
		return cachedSkillModule
	end
	local gui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
	if not gui then
		return nil
	end
	local libObject = gui:FindFirstChild("ClientLibrary")
	if libObject and libObject:IsA("ModuleScript") then
		local ok, lib = pcall(require, libObject)
		if ok and lib and lib.ClientSkillModule then
			cachedSkillModule = lib.ClientSkillModule
			return cachedSkillModule
		end
	end
	local modules = gui:FindFirstChild("Modules")
	local mod = modules and modules:FindFirstChild("ClientSkillModule")
	if mod and mod:IsA("ModuleScript") then
		local ok, result = pcall(require, mod)
		if ok then
			cachedSkillModule = result
			return cachedSkillModule
		end
	end
	return nil
end
local function isAliveMob(mob)
	if not (mob and mob.Parent) then
		return false
	end
	local monster = mob:FindFirstChild("Monster")
	local root = mob:FindFirstChild("HumanoidRootPart")
	return monster and monster:IsA("Humanoid") and monster.Health > 0 and root ~= nil
end
local function mobSize(mob)
	local root = mob.HumanoidRootPart
	return math.max(root.Size.X, root.Size.Y, root.Size.Z)
end
local function bossFrame()
	if cachedBoss and cachedBoss.Parent and os.clock() - bossScan < 0.4 then
		return cachedBoss
	end
	local found
	for _, desc in ipairs(LocalPlayer.PlayerGui:GetDescendants()) do
		if desc.Name == "BossHealthFrame" and desc:FindFirstChild("BossName") then
			if desc.Visible then
				found = desc
				break
			end
			found = desc
		end
	end
	cachedBoss = found
	bossScan = os.clock()
	return found
end
local function livingMobs()
	local folders = workspace:FindFirstChild("Folders")
	local folder = folders and folders:FindFirstChild("Monsters")
	local list = {}
	if folder then
		for _, mob in ipairs(folder:GetChildren()) do
			if isAliveMob(mob) then
				table.insert(list, mob)
			end
		end
	end
	return list
end
local function nearbyThreatMob()
	local root = getRoot()
	if not root then return nil end
	local folders = workspace:FindFirstChild("Folders")
	local monsters = folders and folders:FindFirstChild("Monsters")
	if not monsters then return nil end
	local closest, closestDist
	for _, mob in ipairs(monsters:GetChildren()) do
		if isAliveMob(mob) and mob ~= currentMob then
			local mobRoot = mob:FindFirstChild("HumanoidRootPart")
			if mobRoot then
				local dist = (mobRoot.Position - root.Position).Magnitude
				if dist < MELEE_DODGE_RANGE and (not closestDist or dist < closestDist) then
					closest = mob
					closestDist = dist
				end
			end
		end
	end
	return closest
end
local function findBoss()
	local frame = bossFrame()
	local barOn = frame and frame.Visible
	local bossName = (barOn and frame.BossName.Text) or ""
	local named, largest, largestSize, second = nil, nil, 0, 0
	for _, mob in ipairs(livingMobs()) do
		local size = mobSize(mob)
		if bossName ~= "" and (mob.Name == bossName or mob.Name:find(bossName, 1, true)) then
			named = mob
		end
		if size > largestSize then
			second = largestSize
			largest = mob
			largestSize = size
		elseif size > second then
			second = size
		end
	end
	if named then
		return named
	end
	if largest and largestSize >= 8 then
		return largest
	end
	if barOn and largest and largestSize >= second + 4 then
		return largest
	end
end
local function nearestMob()
	local root = getRoot()
	if not root then
		return nil
	end
	local best, bestDist
	for _, mob in ipairs(livingMobs()) do
		local dist = (mob.HumanoidRootPart.Position - root.Position).Magnitude
		if not bestDist or dist < bestDist then
			best = mob
			bestDist = dist
		end
	end
	return best
end
local nextBossProbe = 0
local function farmTarget()
	if isAliveMob(currentMob) then
		if os.clock() >= nextBossProbe then
			nextBossProbe = os.clock() + 0.3
			local boss = findBoss()
			if isAliveMob(boss) then
				currentMob = boss
			end
		end
		return currentMob
	end
	currentMob = findBoss() or nearestMob()
	nextBossProbe = os.clock() + 0.3
	return currentMob
end
local function getRaidCenter()
	local map = workspace:FindFirstChild("GameMap")
	if not map then return nil end
	for _, child in ipairs(map:GetChildren()) do
		local pos = child:FindFirstChild("CenterAttackPositioner")
		if pos and pos:IsA("BasePart") then
			return pos.Position
		end
	end
end
local function preferredPos(enemyRoot, dt)
	if method == "Above" then
		return enemyRoot.Position + Vector3.new(0, math.max(distance, 12), 0)
	elseif method == "Circling" then
		circleAngle = circleAngle + (CIRCLE_SPEED * (dt or 0.016))
		local x = math.cos(circleAngle) * distance
		local z = math.sin(circleAngle) * distance
		return enemyRoot.Position + Vector3.new(x, 0, z)
	end
	return (enemyRoot.CFrame * CFrame.new(0, 0, distance)).Position
end
local originalCollisions = setmetatable({}, { __mode = "k" })
local function noclip(on)
	if on then
		local character = LocalPlayer.Character
		if not character then
			return
		end
		for _, part in ipairs(character:GetDescendants()) do
			if part:IsA("BasePart") then
				if originalCollisions[part] == nil then
					originalCollisions[part] = part.CanCollide
				end
				part.CanCollide = false
			end
		end
	else
		for part, previous in pairs(originalCollisions) do
			if part.Parent then
				part.CanCollide = previous
			end
			originalCollisions[part] = nil
		end
	end
end
local function releaseStick()
	currentMob = nil
	dodgePos = nil
	resetMotion()
	noclip(false)
	local humanoid = getHumanoid()
	if humanoid then
		humanoid.PlatformStand = false
		humanoid.AutoRotate = true
	end
end
local function flat(v)
	return Vector3.new(v.X, 0, v.Z)
end
local function flatDir(dir)
	local horizontal = flat(dir)
	if horizontal.Magnitude < 0.05 then
		return Vector3.zAxis
	end
	return horizontal.Unit
end
local function isMarker(inst)
	if not (inst:IsA("BasePart") and inst.Anchored) then
		return false
	end
	local c = inst.Color
	local isPurple = c.R > 0.35 and c.B > 0.35 and c.G < 0.35
	local isNeon = inst.Material == Enum.Material.Neon
	if not (isNeon or isPurple) then
		return false
	end
	if inst.Name == "CirclePart" then
		return true
	end
	local size = inst.Size
	if size.X <= 2 and size.Y <= 2 and size.Z <= 1 then
		return true
	end
	return size.Y <= 0.3 and size.X >= 5
end
local function registerThreat(inst)
	if not isMarker(inst) then
		return
	end
	local size = inst.Size
	threats[inst] = {
		origin = inst.Position,
		born = os.clock(),
		bornSize = size,
		look = flatDir(inst.CFrame.LookVector),
		circle = inst.Name == "CirclePart" or (size.X <= 2 and size.Y <= 2 and size.Z <= 1),
	}
end
local function watchDebris()
	if debrisHooked then
		return
	end
	local folders = workspace:FindFirstChild("Folders")
	local debris = folders and folders:FindFirstChild("Debris")
	if not debris then
		return
	end
	debrisHooked = true
	track(debris.ChildAdded:Connect(registerThreat))
	track(debris.ChildRemoved:Connect(function(inst)
		threats[inst] = nil
	end))
	for _, inst in ipairs(debris:GetChildren()) do
		registerThreat(inst)
	end
end
local function circleRadius(part, rec)
	local age = math.max(os.clock() - rec.born, 0.03)
	local now = math.max(part.Size.X, part.Size.Z, part.Size.Y)
	local rate = math.max(now - math.max(rec.bornSize.X, rec.bornSize.Z, rec.bornSize.Y), 0) / age
	return math.clamp(now * 0.5 + rate * 0.5 + 6, 8, 90)
end
local function threatHits(point, part, rec)
	if not part.Parent then
		return false
	end
	local flatPoint = flat(point)
	if rec.circle or part.Name == "CirclePart" then
		return (flatPoint - flat(part.Position)).Magnitude <= circleRadius(part, rec)
	end
	local look = rec.look
	local visualZ = math.max(rec.bornSize.Z, 0.2)
	local length = visualZ >= 19.5 and 340 or math.max(visualZ * 15, 36)
	local rear = flat(rec.origin) - look * (visualZ * 0.5)
	local delta = flatPoint - rear
	local along = delta:Dot(look)
	local side = (delta - look * along).Magnitude
	return along >= -8 and along <= length + 8 and side <= part.Size.X * 0.5 + 5
end
local function threatsBlocking(point)
	for part, rec in pairs(threats) do
		if part.Parent and threatHits(point, part, rec) then
			return true
		end
	end
	return false
end
local function hitCount(point)
	local count = 0
	for part, rec in pairs(threats) do
		if part.Parent and threatHits(point, part, rec) then
			count = count + 1
		end
	end
	return count
end
local function dodgeSpot(anchor, preferred)
	if not threatsBlocking(preferred) then
		dodgePos = nil
		return preferred
	end
	if dodgePos and not threatsBlocking(dodgePos) and (dodgePos - anchor).Magnitude <= 70 then
		return dodgePos
	end
	local bestSafe, bestSafeDist
	local bestAny, bestAnyCount, bestAnyDist
	local function consider(pos)
		local minY = anchor.Y
		if pos.Y < minY then
			pos = Vector3.new(pos.X, minY, pos.Z)
		else
			pos = Vector3.new(pos.X, math.max(pos.Y, minY), pos.Z)
		end
		if (Vector3.new(pos.X, anchor.Y, pos.Z) - anchor).Magnitude > 86 then
			return
		end
		local hits = hitCount(pos)
		local dist = (pos - preferred).Magnitude
		if hits == 0 and (not bestSafeDist or dist < bestSafeDist) then
			bestSafe, bestSafeDist = pos, dist
		end
		if not bestAnyCount or hits < bestAnyCount or (hits == bestAnyCount and dist < bestAnyDist) then
			bestAny, bestAnyCount, bestAnyDist = pos, hits, dist
		end
	end
	consider(preferred)
	for _, radius in ipairs({ 16, 28, 40, 54, 68 }) do
		for i = 0, 15 do
			local angle = (i / 16) * math.pi * 2
			consider(anchor + Vector3.new(math.cos(angle) * radius, 0, math.sin(angle) * radius))
		end
	end
	local angles = {}
	for part, rec in pairs(threats) do
		if part.Parent and not rec.circle and part.Name ~= "CirclePart" then
			local look = rec.look
			local along = math.clamp((flat(preferred) - flat(rec.origin)):Dot(look), 0, 80)
			local center = flat(rec.origin) + look * along
			local right = look:Cross(Vector3.yAxis)
			if right.Magnitude > 0.05 then
				right = right.Unit
				local step = part.Size.X * 0.5 + 8
				consider(center + right * step)
				consider(center - right * step)
			end
			table.insert(angles, math.atan2(look.Z, look.X))
		elseif part.Parent then
			local radius = circleRadius(part, rec) + 4
			local away = flat(preferred) - flat(part.Position)
			if away.Magnitude < 1 then
				away = Vector3.zAxis * radius
			else
				away = away.Unit * radius
			end
			consider(flat(part.Position) + away)
		end
	end
	table.sort(angles)
	for i = 1, #angles do
		local first = angles[i]
		local second = angles[(i % #angles) + 1]
		local gap = second - first
		if i == #angles then
			gap = (second + math.pi * 2) - first
		end
		local mid = first + gap * 0.5
		local dir = Vector3.new(math.cos(mid), 0, math.sin(mid))
		consider(anchor + dir * 40)
		consider(anchor + dir * 56)
		consider(anchor + dir * 72)
	end
	dodgePos = bestSafe or bestAny or preferred
	return dodgePos
end
local function resultsOpen()
	local gui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
	if not gui then
		return false
	end
	local universal = gui:FindFirstChild("UniversalGui")
	if universal then
		local center = universal:FindFirstChild("UniversalCenterUIFrame")
		if center then
			local result = center:FindFirstChild("ResultUI")
			if result and result.Visible then
				return true
			end
		end
	end
	return false
end
local function raidIsOpen()
	local circles = workspace:FindFirstChild("InteractionCircles")
	local portal = circles and circles:FindFirstChild("RaidPortal")
	local timer = portal and portal:FindFirstChild("RaidTimer")
	local frame = timer and timer:FindFirstChild("Frame")
	local text = frame and frame:FindFirstChild("RaidTimerText")
	if text and text:IsA("TextLabel") then
		local t = text.Text:lower()
		if t:find("open") or t:find("available") or t == "0:00" or t == "00:00" then
			return true
		end
		local mins = t:match("(%d+):")
		if mins and tonumber(mins) and tonumber(mins) < 2 then
			return true
		end
	end
	return false
end
local function setSelected(option)
	selected = {}
	local list = type(option) == "table" and option or { option }
	for _, name in ipairs(list) do
		selected[name] = true
	end
end
local function one(option)
	return type(option) == "table" and option[1] or option
end
local lastCast = {}
local function ready(skillModule, key)
	if skillModule.returnCooldown and skillModule.returnCooldown(key) > 0 then
		return false
	end
	if skillModule.returnSkillIsInUse and skillModule.returnSkillIsInUse() then
		return false
	end
	return os.clock() - (lastCast[key] or 0) >= 0.12
end
local function tryCast(skillModule, name)
	local number = tonumber(name:match("%d+"))
	if name:find("Skill") then
		if ready(skillModule, number) then
			lastCast[number] = os.clock()
			skillModule.useSkill(number)
			return true
		end
	elseif name:find("Assist") then
		local key = "Assist" .. number
		if ready(skillModule, key) then
			lastCast[key] = os.clock()
			skillModule.useAssistSkill(number)
			return true
		end
	elseif name:find("Fruit") then
		local key = "Fruit" .. number
		if ready(skillModule, key) then
			lastCast[key] = os.clock()
			skillModule.useFruitSkill(number)
			return true
		end
	end
	return false
end
local MELEE_RANGE = 12
local SKILL_RANGE_BUFFER = 12
local allAbilities = {
	"Skill 1", "Skill 2", "Skill 3", "Skill 4", "Skill 5",
	"Assist 1", "Assist 2",
	"Fruit 1", "Fruit 2"
}
local function attackSkills()
	local skillModule = getSkillModule()
	local root = getRoot()
	if not (autoFarm and skillModule and root and currentMob and isAliveMob(currentMob)) then
		return
	end
	local enemyRoot = currentMob.HumanoidRootPart
	local dist = (root.Position - enemyRoot.Position).Magnitude
	if dist <= (distance + SKILL_RANGE_BUFFER) then
		for _, name in ipairs(allAbilities) do
			if selected[name] then
				pcall(tryCast, skillModule, name)
			end
		end
	end
end
local function attackBasic()
	if not selected.Basic then return end
	local skillModule = getSkillModule()
	local root = getRoot()
	if not (autoFarm and skillModule and root and currentMob and isAliveMob(currentMob)) then
		return
	end
	local enemyRoot = currentMob.HumanoidRootPart
	local dist = (root.Position - enemyRoot.Position).Magnitude
	if dist <= MELEE_RANGE then
		pcall(function()
			skillModule.basicAttack()
		end)
	end
end
local function tryReconnect()
	if not autoReconnect or reconnecting then
		return
	end
	reconnecting = true
	task.delay(10, function()
		reconnecting = false
	end)
	refreshQueue()
	TeleportService:Teleport(game.PlaceId, LocalPlayer)
end
local function unload()
	alive = false
	autoFarm = false
	autoDodge = false
	autoRetry = false
	autoRaid = false
	autoSpeedRaid = false
	autoBossRush = false
	autoExtreme = false
	priorityEnabled = false
	releaseStick()
	table.clear(threats)
	for _, conn in ipairs(conns) do
		pcall(function()
			conn:Disconnect()
		end)
	end
	pcall(function()
		Window:Unload()
	end)
	getgenv().ThaBooRunning = false
end
local retryHooked = false
local function hookRetry()
	if retryHooked then return end
	local gui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
	if not gui then return end
	local universal = gui:FindFirstChild("UniversalGui")
	if not universal then return end
	local center = universal:FindFirstChild("UniversalCenterUIFrame")
	if not center then return end
	local result = center:FindFirstChild("ResultUI")
	if not result then return end
	retryHooked = true
	local function onResults()
		if not result.Visible then return end
		local now = os.clock()
		if now - lastResultCount < 2 then return end
		lastResultCount = now
		runsDone = runsDone + 1
		saveRuns()
		if leaveAfter > 0 then
			notify("Run " .. runsDone .. " / " .. leaveAfter)
		end
		if flagOn("was_speed_raid") and afterSpeedLoadout > 0 then
			setFlag("was_speed_raid", false)
			task.wait(0.5)
			equipLoadout(afterSpeedLoadout)
		end
		if leaveAfter > 0 and runsDone >= leaveAfter then
			task.wait(1.2)
			if not alive then return end
			refreshQueue()
			invoke("TeleportToLobby")
			notify("Left after " .. runsDone .. " runs")
			runsDone = 0
			saveRuns()
			return
		end
		if autoRetry then
			task.wait(1.5)
			if not alive or not result.Visible then return end
			local remote = ReplicatedStorage:FindFirstChild("RemoteEvents")
			remote = remote and remote:FindFirstChild("MainRemoteEvent")
			if remote then
				remote:FireServer("RetryDungeon")
				notify("Retry sent")
			end
		end
	end
	track(result:GetPropertyChangedSignal("Visible"):Connect(function()
		if result.Visible then
			task.spawn(onResults)
		end
	end))
	if result.Visible then
		task.spawn(onResults)
	end
end
if flagOn("pending_raid") and game.PlaceId == RAID_LOBBY_ID then
	task.delay(6, function()
		local circles = workspace:WaitForChild("InteractionCircles", 10)
		local portal2 = circles and circles:WaitForChild("RaidPortal2", 10)
		if portal2 then
			local ok = pcall(function()
				return startMap(raidPick, raidDifficulty, false)
			end)
			if ok then
				setFlag("pending_raid", false)
				notify("Starting raid: " .. tostring(raidPick))
			else
				notify("Raid start failed, will retry")
			end
		else
			notify("Raid lobby not ready")
		end
	end)
end
Farm:CreateSection({ name = "Farm" })
Farm:CreateSlider({
	name = "Farm Distance",
	range = { 2, 40 },
	increment = 1,
	suffix = " studs",
	value = 6,
	flag = "FarmDistance",
	callback = function(value)
		distance = value
	end,
})
Farm:CreateSlider({
	name = "Farm Speed",
	range = { 20, 200 },
	increment = 5,
	suffix = " studs/s",
	value = 80,
	flag = "FarmSpeed",
	callback = function(value)
		FARM_SPEED = value
	end,
})
Farm:CreateDropdown({
	name = "Farm Position",
	options = { "Behind", "Above", "Circling" },
	value = "Behind",
	multiSelect = false,
	flag = "FarmPosition",
	callback = function(option)
		method = one(option)
		if method ~= "Above" and method ~= "Circling" then
			method = "Behind"
		end
	end,
})
Farm:CreateDropdown({
	name = "Abilities",
	options = { "Basic", "Skill 1", "Skill 2", "Skill 3", "Skill 4", "Skill 5", "Assist 1", "Assist 2", "Fruit 1", "Fruit 2" },
	value = { "Basic" },
	multiSelect = true,
	flag = "Abilities",
	callback = function(option)
		setSelected(option)
	end,
})
Farm:CreateToggle({ name = "Auto Farm", value = false, flag = "AutoFarm", callback = function(value)
	autoFarm = value
	if not value then
		releaseStick()
	end
end })
Farm:CreateToggle({ name = "Auto Dodge", value = false, flag = "SmartDodge", callback = function(value)
	autoDodge = value
	dodgePos = nil
end })
Farm:CreateToggle({ name = "Auto Retry", value = false, flag = "AutoRetry", callback = function(value)
	autoRetry = value
end })
Farm:CreateInput({
	name = "Leave After Runs",
	placeholder = "0 = off",
	value = "",
	flag = "LeaveAfterRuns",
	callback = function(text)
		local number = tonumber(text)
		local newVal = number and math.max(0, math.floor(number)) or 0
		if leaveAfterLoaded and newVal ~= leaveAfter then
			runsDone = 0
			saveRuns()
		end
		leaveAfter = newVal
		leaveAfterLoaded = true
	end,
})
Lobby:CreateSection({ name = "Dimension" })
Lobby:CreateDropdown({
	name = "Dimension",
	options = dimensionOptions,
	value = dimensionPick,
	multiSelect = false,
	flag = "Dimension",
	callback = function(option)
		dimensionPick = one(option)
		saveFarmConfig()
	end,
})
Lobby:CreateDropdown({
	name = "Dimension Difficulty",
	options = { "Hardest", "Easy", "Hard", "Nightmare" },
	value = dimensionDifficulty,
	multiSelect = false,
	flag = "DimensionDifficulty",
	callback = function(option)
		dimensionDifficulty = one(option)
		saveFarmConfig()
	end,
})
Lobby:CreateToggle({ name = "Hardcore", value = hardcore, flag = "Hardcore", callback = function(value)
	hardcore = value
	saveFarmConfig()
end })
Lobby:CreateToggle({ name = "Friends Only", value = friendsOnly, flag = "FriendsOnly", callback = function(value)
	friendsOnly = value
	saveFarmConfig()
end })
Lobby:CreateSlider({
	name = "Before Starting",
	range = { 0, 20 },
	increment = 1,
	suffix = "s",
	value = 0,
	flag = "StartDelay",
	callback = function(value)
		startDelay = value
	end,
})
Lobby:CreateToggle({
	name = "Start Dimension",
	value = false,
	flag = "StartDimension",
	callback = function(value)
		if not value then
			local remote = ReplicatedStorage:FindFirstChild("RemoteEvents")
			remote = remote and remote:FindFirstChild("MainRemoteEvent")
			if remote then
				remote:FireServer("RemovePlayerFromParty")
			end
			return
		end
		task.spawn(function()
			task.wait(0.3)
			local delay = startDelay
			if delay > 0 then
				notify("Starting in " .. delay .. "s")
				task.wait(delay)
			end
			if not alive then return end
			local map, diff, level = resolveDimension(dimensionPick, dimensionDifficulty)
			if not map or not diff then
				notify("Nothing unlocked at level " .. tostring(level))
				return
			end
			local req = dimensionReqs[map] and dimensionReqs[map][diff]
			local isAuto = dimensionPick == "Hardest" or dimensionDifficulty == "Hardest"
			if req and level < req and isAuto then
				notify(map .. " " .. diff .. " needs level " .. req)
				return
			end
			notify(map .. " · " .. diff .. " · Lv " .. level)
			startMap(map, diff, hardcore)
		end)
	end,
})
Lobby:CreateSection({ name = "Raid" })
Lobby:CreateDropdown({
	name = "Raid",
	options = raidNames,
	value = raidPick,
	multiSelect = false,
	flag = "Raid",
	callback = function(option)
		raidPick = one(option)
		saveFarmConfig()
	end,
})
Lobby:CreateDropdown({
	name = "Raid Difficulty",
	options = { "Easy" },
	value = raidDifficulty,
	multiSelect = false,
	flag = "RaidDifficulty",
	callback = function(option)
		raidDifficulty = one(option)
		saveFarmConfig()
	end,
})
Lobby:CreateToggle({
	name = "Auto Start Raid",
	value = false,
	flag = "AutoRaid",
	callback = function(value)
		autoRaid = value
	end,
})
Lobby:CreateToggle({
	name = "Auto Speed Raid",
	value = false,
	flag = "AutoSpeedRaid",
	callback = function(value)
		autoSpeedRaid = value
	end,
})
Lobby:CreateDropdown({
	name = "After Speed Raid Loadout",
	options = { "Off", "Loadout 1", "Loadout 2", "Loadout 3" },
	value = "Off",
	multiSelect = false,
	flag = "AfterSpeedLoadout",
	callback = function(option)
		local pick = one(option)
		if pick == "Loadout 1" then
			afterSpeedLoadout = 1
		elseif pick == "Loadout 2" then
			afterSpeedLoadout = 2
		elseif pick == "Loadout 3" then
			afterSpeedLoadout = 3
		else
			afterSpeedLoadout = 0
		end
	end,
})
Lobby:CreateSection({ name = "Other" })
Lobby:CreateToggle({
	name = "Auto Boss Rush",
	value = false,
	flag = "AutoBossRush",
	callback = function(value)
		autoBossRush = value
	end,
})
Lobby:CreateDropdown({
	name = "Extreme Difficulty",
	options = { "Easy", "Normal", "Nightmare", "Extreme" },
	value = extremeDifficulty,
	multiSelect = false,
	flag = "ExtremeDifficulty",
	callback = function(option)
		extremeDifficulty = one(option)
		saveFarmConfig()
	end,
})
Lobby:CreateToggle({
	name = "Auto Extreme Dimension",
	value = false,
	flag = "AutoExtreme",
	callback = function(value)
		autoExtreme = value
	end,
})
Lobby:CreateButton({ name = "Infinite Mode", callback = function() startMap("Infinite Mode", "Infinite", false) end })
Lobby:CreateButton({ name = "Time Challenge", callback = function() invoke("TeleportToTimeChallenge") end })
Lobby:CreateButton({ name = "Daily Raid Challenge", callback = function() invoke("TeleportToRaidChallenge") end })
Lobby:CreateButton({ name = "Raid Lobby", callback = function() invoke("TeleportToRaidLobby") end })
Lobby:CreateButton({ name = "High Level Lobby", callback = function() invoke("TeleportToHighLevelLobby") end })
Lobby:CreateButton({ name = "AFK", callback = function() invoke("TeleportToAFK") end })
Lobby:CreateButton({ name = "Return to Lobby", callback = function() invoke("TeleportToLobby") end })
Priority:CreateSection({ name = "Priority System" })
Priority:CreateToggle({
	name = "Enable Priority",
	value = false,
	flag = "PriorityEnabled",
	callback = function(value)
		priorityEnabled = value
	end,
})
Priority:CreateSection({ name = "Priority Values (0 = low, 8 = high)" })
Priority:CreateSlider({ name = "Raid", range = {0, 8}, increment = 1, value = 8, flag = "PrioRaid", callback = function(v) priorities.Raid = v end })
Priority:CreateSlider({ name = "Boss Rush", range = {0, 8}, increment = 1, value = 6, flag = "PrioBoss", callback = function(v) priorities.BossRush = v end })
Priority:CreateSlider({ name = "Speed Raid", range = {0, 8}, increment = 1, value = 5, flag = "PrioSpeed", callback = function(v) priorities.SpeedRaid = v end })
Priority:CreateSlider({ name = "Dimension", range = {0, 8}, increment = 1, value = 4, flag = "PrioDim", callback = function(v) priorities.Dimension = v end })
Misc:CreateSection({ name = "Misc" })
Misc:CreateButton({
	name = "Save Config",
	callback = function()
		saveFarmConfig()
		notify("Config saved")
	end,
})
Misc:CreateToggle({
	name = "Execute on Teleport",
	value = flagOn("exec_on_tp"),
	flag = "ExecuteOnTeleport",
	callback = function(value)
		setFlag("exec_on_tp", value)
		if value then
			armTeleport(false)
		end
	end,
})
Misc:CreateToggle({ name = "Auto Reconnect", value = false, flag = "AutoReconnect", callback = function(value)
	autoReconnect = value
end })
Misc:CreateButton({ name = "Unload UI", callback = unload })
track(LocalPlayer.CharacterAdded:Connect(function()
	currentMob = nil
	dodgePos = nil
	cachedBoss = nil
	cachedSkillModule = nil
	resetMotion()
	noclip(false)
	retryHooked = false
	task.delay(2, hookRetry)
end))
local attackBusy = false
track(RunService.Heartbeat:Connect(function(dt)
	if not alive then
		return
	end
	watchDebris()
	for part in pairs(threats) do
		if not part.Parent then
			threats[part] = nil
		end
	end
	local humanoid = getHumanoid()
	local root = getRoot()
	if not (autoFarm and humanoid and root and humanoid.Health > 0) then
		return
	end
	local target = farmTarget()
	if not isAliveMob(target) then
		if motionRoot or humanoid.PlatformStand then
			releaseStick()
		end
		currentMob = nil
		return
	end
	local enemyRoot = target.HumanoidRootPart
	local stand = preferredPos(enemyRoot, dt)
	local center = getRaidCenter()
	if center then
		local minY = center.Y + 15
		if stand.Y < minY then
			stand = Vector3.new(stand.X, minY, stand.Z)
		end
	end
	local rootY = root.Position.Y
	if stand.Y < rootY - 4 then
		stand = Vector3.new(stand.X, rootY - 2, stand.Z)
	end
	local dodging = false
	local speed = FARM_SPEED
	if autoDodge then
		local dodgeAnchor = Vector3.new(enemyRoot.Position.X, stand.Y, enemyRoot.Position.Z)
		local safePos = dodgeSpot(dodgeAnchor, stand)
		dodging = (safePos - stand).Magnitude > 3
		if dodging then
			speed = DODGE_SPEED
		end
		stand = safePos
		if center then
			local minY = center.Y + 15
			if stand.Y < minY then
				stand = Vector3.new(stand.X, minY, stand.Z)
			end
		end
		local threatMob = nearbyThreatMob()
		if threatMob then
			local mobRoot = threatMob.HumanoidRootPart
			local away = root.Position - mobRoot.Position
			if away.Magnitude < 1 then
				away = Vector3.new(1, 0, 0)
			end
			local dodgeTarget = root.Position + away.Unit * 18
			stand = dodgeTarget
			speed = DODGE_SPEED
			dodging = true
		end
	else
		dodgePos = nil
	end
	if not humanoid.PlatformStand then
		humanoid.PlatformStand = true
	end
	humanoid.AutoRotate = false
	noclip(true)
	stepToward(root, stand, enemyRoot.Position, speed, dt, dodging)
	local now = os.clock()
	if not attackBusy and now >= nextAttack then
		nextAttack = now + 0.07
		attackBusy = true
		task.spawn(function()
			pcall(attackSkills)
			attackBusy = false
		end)
	end
	if now >= nextBasic then
		nextBasic = now + 0.05
		task.spawn(attackBasic)
	end
end))
task.spawn(function()
	while alive do
		local now = os.clock()
		local inRaidLobby = game.PlaceId == RAID_LOBBY_ID
		if priorityEnabled then
			local chosen = highestAvailable()
			if chosen == "Raid" and not resultsOpen() and not flagOn("pending_raid") and not inRaidLobby and (now - lastRaidAttempt) > 12 then
				lastRaidAttempt = now
				setFlag("pending_raid", true)
				notify("Going to raid lobby...")
				pcall(function()
					invoke("TeleportToRaidLobby")
				end)
			elseif chosen == "SpeedRaid" and not resultsOpen() and (now - lastSpeedAttempt) > 8 then
				lastSpeedAttempt = now
				local char = pickSpeedCharacter()
				if char then
					setFlag("was_speed_raid", true)
					pcall(function()
						invoke("TeleportToShadowRaid", char)
					end)
				else
					notify("No unused characters for Speed Raid")
				end
			elseif chosen == "BossRush" and (now - lastBossAttempt) > 6 then
				lastBossAttempt = now
				if bossRushHasTickets() then
					pcall(function()
						invoke("TeleportToBossRush")
					end)
				end
			elseif chosen == "Dimension" and autoExtreme and (now - lastBossAttempt) > 6 then
				lastBossAttempt = now
				pcall(function()
					invoke("TeleportToExtremeDimension", extremeDifficulty)
				end)
			end
		else
			if autoRaid and raidIsOpen() and not resultsOpen() and not flagOn("pending_raid") and not inRaidLobby and (now - lastRaidAttempt) > 12 then
				lastRaidAttempt = now
				setFlag("pending_raid", true)
				notify("Going to raid lobby...")
				pcall(function()
					invoke("TeleportToRaidLobby")
				end)
			end
			if autoSpeedRaid and speedRaidReady() and not resultsOpen() and (now - lastSpeedAttempt) > 8 then
				lastSpeedAttempt = now
				local char = pickSpeedCharacter()
				if char then
					setFlag("was_speed_raid", true)
					pcall(function()
						invoke("TeleportToShadowRaid", char)
					end)
				else
					notify("No unused characters for Speed Raid")
				end
			end
			if autoBossRush and bossRushHasTickets() and (now - lastBossAttempt) > 6 then
				lastBossAttempt = now
				pcall(function()
					invoke("TeleportToBossRush")
				end)
			end
			if autoExtreme and (now - lastRaidAttempt) > 6 then
				lastRaidAttempt = now
				pcall(function()
					invoke("TeleportToExtremeDimension", extremeDifficulty)
				end)
			end
		end
		task.wait(1.5)
	end
end)
task.spawn(function()
	local ok, overlay = pcall(function()
		return game:GetService("CoreGui"):WaitForChild("RobloxPromptGui", 15):WaitForChild("promptOverlay", 15)
	end)
	if ok and overlay then
		track(overlay.ChildAdded:Connect(function(obj)
			if obj.Name == "ErrorPrompt" then
				task.wait(1.2)
				tryReconnect()
			end
		end))
	end
end)
track(TeleportService.TeleportInitFailed:Connect(function()
	task.wait(2)
	tryReconnect()
end))
if flagOn("exec_on_tp") then
	task.defer(function()
		armTeleport(true)
	end)
end
task.delay(3, hookRetry)
]=]
getgenv().ThaBooSource = SOURCE
loadstring(SOURCE)()
