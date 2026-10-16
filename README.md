--- V.8.7.0 ABA Smart Server Browser Balancer
repeat task.wait(0.1) until game:IsLoaded()

local TeleportService = game:GetService("TeleportService")
local Players = game:GetService("Players")
local EarlyLP = Players.LocalPlayer

local MAIN_PLACE_ID = 1458767429
local AFK_WORLD_PLACE_ID = 5411459567

if game.PlaceId == AFK_WORLD_PLACE_ID then
	pcall(function()
		TeleportService:Teleport(MAIN_PLACE_ID, EarlyLP)
	end)
	return
end

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VUser   = game:GetService("VirtualUser")
local VIM     = game:GetService("VirtualInputManager")
local Http    = game:GetService("HttpService")
local UIS     = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local SoundService = game:GetService("SoundService")
local LP      = Players.LocalPlayer

-- ===== Full Client Mute =====
pcall(function()
	UserSettings().GameSettings.MasterVolume=0
end)

local function muteSound(obj)
	if not obj or not obj:IsA("Sound") then return end
	pcall(function()
		obj.Volume=0
		obj:Stop()
	end)
end

task.spawn(function()
	for _,obj in ipairs(game:GetDescendants()) do
		if obj:IsA("Sound") then muteSound(obj) end
	end
end)

game.DescendantAdded:Connect(function(obj)
	if obj:IsA("Sound") then
		task.defer(muteSound,obj)
	end
end)

-- ===== Hide ABA move slots 1-5 =====
local function hideMoveSlots()
	local pg=LP:FindFirstChild("PlayerGui")
	local hud=pg and pg:FindFirstChild("HUD")
	local moves=hud and hud:FindFirstChild("Moves")
	if not moves then return end

	for i=1,5 do
		local slot=moves:FindFirstChild(tostring(i))
		if slot and slot:IsA("GuiObject") then
			pcall(function()
				slot.Visible=false
			end)
		end
	end
end

task.spawn(function()
	while task.wait(0.20) do
		hideMoveSlots()
	end
end)

LP:WaitForChild("PlayerGui").DescendantAdded:Connect(function(obj)
	if obj.Name=="Moves" or (obj.Parent and obj.Parent.Name=="Moves") then
		task.defer(hideMoveSlots)
	end
end)

local resetSequence = 0
local actionNoticeLabel = nil
local actionNoticeToken = 0
local function notifyAction(title, message, duration)
	local msg=tostring(message or "")
	pcall(function()
		StarterGui:SetCore("SendNotification",{
			Title = title or "WWHub",
			Text = msg,
			Duration = duration or 5
		})
	end)
	warn(("[WWHub] %s | %s"):format(tostring(title or "WWHub"),msg))
	if actionNoticeLabel then
		actionNoticeToken+=1
		local token=actionNoticeToken
		actionNoticeLabel.Text=tostring(title or "WWHub").." • "..msg
		actionNoticeLabel.Visible=true
		task.delay(duration or 5,function()
			if actionNoticeLabel and token==actionNoticeToken then
				actionNoticeLabel.Visible=false
			end
		end)
	end
end

local myName = LP.Name

-- ===== Robust external config bridge =====
-- Priority:
--   1) Fresh _G values visible to this BODY
--   2) Explicit getgenv().WWHubConfig written by the loader
--   3) Legacy getgenv().main/afk fallback
-- This prevents stale getgenv() values from overriding a newly-run loader.
local ENV=(getgenv and getgenv()) or _G
local BRIDGE=(type(ENV.WWHubConfig)=="table" and ENV.WWHubConfig) or {}

local function normalizeList(value)
	local out={}

	if type(value)=="string" then
		for part in string.gmatch(value,"[^,]+") do
			local s=part:gsub("^%s+",""):gsub("%s+$","")
			if s~="" then table.insert(out,s) end
		end
		return out
	end

	if type(value)=="table" then
		for _,v in ipairs(value) do
			if type(v)=="string" then
				local s=v:gsub("^%s+",""):gsub("%s+$","")
				if s~="" then table.insert(out,s) end
			end
		end
	end

	return out
end

local function usableListValue(v)
	if type(v)=="string" then
		return v:gsub("%s+","")~=""
	end
	if type(v)=="table" then
		return next(v)~=nil
	end
	return false
end

local function chooseConfigValue(localValue,bridgeValue,envValue)
	if usableListValue(localValue) then return localValue,"_G" end
	if usableListValue(bridgeValue) then return bridgeValue,"WWHubConfig" end
	if usableListValue(envValue) then return envValue,"getgenv" end
	return nil,"EMPTY"
end

local rawMain,mainSource=chooseConfigValue(
	_G and _G.main,
	BRIDGE.main,
	ENV.main
)
local rawAfk,afkSource=chooseConfigValue(
	_G and _G.afk,
	BRIDGE.afk,
	ENV.afk
)

local mainCfg=normalizeList(rawMain)
local afkCfg=normalizeList(rawAfk)

_G.main=mainCfg
_G.afk=afkCfg
ENV.main=mainCfg
ENV.afk=afkCfg
BRIDGE.main=mainCfg
BRIDGE.afk=afkCfg
ENV.WWHubConfig=BRIDGE

local function chooseScalar(localValue,bridgeValue,envValue,defaultValue)
	if localValue~=nil then return localValue end
	if bridgeValue~=nil then return bridgeValue end
	if envValue~=nil then return envValue end
	return defaultValue
end

local externalCap=tonumber(chooseScalar(
	_G and _G.pointcap,
	BRIDGE.pointcap,
	ENV.pointcap,
	1000
)) or 1000
_G.pointcap=externalCap
ENV.pointcap=externalCap
BRIDGE.pointcap=externalCap

local externalFpsCap=math.max(1,math.floor(tonumber(chooseScalar(
	_G and _G.fpscap,
	BRIDGE.fpscap,
	ENV.fpscap,
	20
)) or 20))
_G.fpscap=externalFpsCap
ENV.fpscap=externalFpsCap
BRIDGE.fpscap=externalFpsCap

local antiAfkJumpSec=math.max(60,math.floor(tonumber(chooseScalar(
	_G and _G.antiAfkJumpSec,
	BRIDGE.antiAfkJumpSec,
	ENV.antiAfkJumpSec,
	300
)) or 300))
_G.antiAfkJumpSec=antiAfkJumpSec
ENV.antiAfkJumpSec=antiAfkJumpSec
BRIDGE.antiAfkJumpSec=antiAfkJumpSec

pcall(function()
	if setfpscap then setfpscap(externalFpsCap) end
end)

warn("[WWHub] Config source MAIN="..tostring(mainSource).." AFK="..tostring(afkSource))

local function sameUser(a,b)
	a=tostring(a or ""):gsub("^%s+",""):gsub("%s+$",""):lower()
	b=tostring(b or ""):gsub("^%s+",""):gsub("%s+$",""):lower()
	return a~="" and a==b
end

local IS_MAIN=false
local IS_AFK=false

for _,n in ipairs(_G.main) do
	if sameUser(n,myName) then IS_MAIN=true break end
end

for _,n in ipairs(_G.afk) do
	if sameUser(n,myName) and not IS_MAIN then IS_AFK=true break end
end

local IS_FARMER=IS_MAIN or IS_AFK
local FARM_ROLE=IS_AFK and "AFK" or (IS_MAIN and "MAIN" or "UNLISTED")
local manualOverride=false
local lastStartMode=IS_FARMER and "AUTO" or "MANUAL"
local autoStartAttempts=0

warn("[WWHub] User: "..tostring(myName))
warn("[WWHub] Role: "..FARM_ROLE)
warn("[WWHub] Main count: "..tostring(#_G.main))
warn("[WWHub] AFK count: "..tostring(#_G.afk))
warn("[WWHub] Main config: "..table.concat(_G.main,", "))
warn("[WWHub] AFK config: "..table.concat(_G.afk,", "))
if not IS_FARMER then
	warn("[WWHub] UNLISTED diagnostic: local username did not match any loaded MAIN/AFK entry")
end

-- ===== Scoped Bakugou-Q workspace.Explosion remover =====
-- Do not delete every workspace.Explosion globally because other ABA FX modules may use it.
-- Only suppress explosions created immediately after this script fires Bakugou Q.
local qExplosionSuppressUntil=0

local function removeWorkspaceExplosion(obj)
	if not obj or obj.Parent~=workspace then return end
	if obj.Name~="Explosion" then return end
	if os.clock()>qExplosionSuppressUntil then return end
	pcall(function() obj:Destroy() end)
end

workspace.ChildAdded:Connect(function(obj)
	if obj.Name=="Explosion" and os.clock()<=qExplosionSuppressUntil then
		task.defer(removeWorkspaceExplosion,obj)
	end
end)

-- ===== Listed-player visual cloak =====
-- Client-side only: hides listed-player cosmetics and skill/Q FX; M1/block animations stay enabled.
local CLOAK_TAG="WWHub_CloakName"

local function isConfiguredFarmUser(name)
	for _,n in ipairs(_G.main) do if sameUser(n,name) then return true end end
	for _,n in ipairs(_G.afk) do if sameUser(n,name) then return true end end
	return false
end

local function isVisualFx(obj)
	return obj:IsA("ParticleEmitter")
		or obj:IsA("Trail")
		or obj:IsA("Beam")
		or obj:IsA("Fire")
		or obj:IsA("Smoke")
		or obj:IsA("Sparkles")
		or obj:IsA("Highlight")
end

local SKILL_FX_EXACT={
	["exploddodge"]=true,
	["explosion1"]=true,
	["explosion2"]=true,
	["rocketexplode"]=true,
	["bluehexplode"]=true,
	["redexplosion2"]=true,
	["explosiveflurryfx"]=true,
	["jojoexplosionfx"]=true,
}

local function skillFxName(name)
	local n=tostring(name or ""):lower()
	if SKILL_FX_EXACT[n] then return true end
	return n:find("exploddodge",1,true)
		or n:find("explosionfx",1,true)
		or n:find("blastfx",1,true)
		or n:find("dodgefx",1,true)
		or n:find("skillfx",1,true)
		or n:find("abilityfx",1,true)
		or n:find("bakugou",1,true)
		or n:find("bakugo",1,true)
		or n:find("nitro",1,true)
end

local function removeSkillFxObject(obj)
	if not obj or not obj.Parent then return end
	pcall(function()
		-- Direct skill container, e.g. workspace.Live.<user>.ExplodDodge.
		if skillFxName(obj.Name) then
			obj:Destroy()
			return
		end

		-- Visual descendants inside a known skill-effect container.
		local p=obj.Parent
		local insideSkill=false
		for _=1,4 do
			if not p then break end
			if skillFxName(p.Name) then
				insideSkill=true
				break
			end
			p=p.Parent
		end
		if not insideSkill then return end

		if isVisualFx(obj) then
			obj.Enabled=false
		elseif obj:IsA("Explosion") then
			obj:Destroy()
		elseif obj:IsA("Sound") then
			obj.Volume=0
			obj:Stop()
		end
	end)
end

local function cleanSkillFxInCharacter(char)
	if not char then return end
	-- Snapshot first because destroying while iterating can mutate descendants.
	local list=char:GetDescendants()
	for _,obj in ipairs(list) do
		if obj.Parent then
			if skillFxName(obj.Name) then
				removeSkillFxObject(obj)
			else
				local p=obj.Parent
				for _=1,4 do
					if not p then break end
					if skillFxName(p.Name) then
						removeSkillFxObject(obj)
						break
					end
					p=p.Parent
				end
			end
		end
	end
end

local CORE_BODY_NAMES={
	["Head"]=true,["HumanoidRootPart"]=true,["Torso"]=true,
	["UpperTorso"]=true,["LowerTorso"]=true,
	["Left Arm"]=true,["Right Arm"]=true,["Left Leg"]=true,["Right Leg"]=true,
	["LeftUpperArm"]=true,["LeftLowerArm"]=true,["LeftHand"]=true,
	["RightUpperArm"]=true,["RightLowerArm"]=true,["RightHand"]=true,
	["LeftUpperLeg"]=true,["LeftLowerLeg"]=true,["LeftFoot"]=true,
	["RightUpperLeg"]=true,["RightLowerLeg"]=true,["RightFoot"]=true,
	["Humanoid"]=true,
}

local GAMEPLAY_KEEP_NAMES={
	["ServerTraits"]=true,["Input"]=true,["Animate"]=true,["Health"]=true,
	["Stats"]=true,["Charge"]=true,
}

local function isCosmeticDirectChild(obj)
	if not obj then return false end
	if CORE_BODY_NAMES[obj.Name] or GAMEPLAY_KEEP_NAMES[obj.Name] then return false end

	local n=tostring(obj.Name or ""):lower()

	-- ABA morph/outfit containers.
	if n=="morph"
		or n:find("morph",1,true)
		or n:find("outfit",1,true)
		or n:find("costume",1,true)
		or n:find("cosmetic",1,true)
		or n:find("accessory",1,true)
		or n:find("hair",1,true) then
		return true
	end

	-- Clothing/accessories only. Keep CharacterMesh/SpecialMesh/base body geometry.
	if obj:IsA("Accessory") or obj:IsA("Hat")
		or obj:IsA("Shirt") or obj:IsA("Pants") or obj:IsA("ShirtGraphic")
		or obj:IsA("Decal") or obj:IsA("Texture") then
		return true
	end

	return false
end

local function stripCharacterCosmetics(char)
	if not char then return end

	-- Direct ABA cosmetics such as:
	-- workspace.Live.<user>.Pants
	-- workspace.Live.<user>.Morph
	-- Accessory / hair / outfit containers.
	for _,child in ipairs(char:GetChildren()) do
		if isCosmeticDirectChild(child) then
			pcall(function() child:Destroy() end)
		end
	end

	-- Remove visual textures only; preserve CharacterMesh/SpecialMesh/MeshPart geometry.
	for _,obj in ipairs(char:GetDescendants()) do
		pcall(function()
			if obj:IsA("Decal") or obj:IsA("Texture") then
				obj:Destroy()
			elseif obj:IsA("SurfaceAppearance") then
				-- SurfaceAppearance is texture/material data, not the mesh shape itself.
				obj:Destroy()
			elseif obj:IsA("SpecialMesh") then
				-- Keep MeshId/Scale so head/body does not become blocky.
				obj.TextureId=""
			elseif obj:IsA("MeshPart") then
				-- Keep MeshId; clear only the texture.
				obj.TextureID=""
			end
		end)
	end
end

local function blackenCharacterObject(obj)
	pcall(function()
		if obj:IsA("Accessory") or obj:IsA("Hat") then
			obj:Destroy()

		elseif obj:IsA("Shirt") or obj:IsA("Pants") or obj:IsA("ShirtGraphic") then
			obj:Destroy()

		elseif obj:IsA("CharacterMesh") then
			-- KEEP: CharacterMesh defines package/body shape.
			-- Destroying it was what made some avatars blocky.

		elseif obj:IsA("SpecialMesh") then
			-- KEEP MeshId/Scale. Remove texture only.
			obj.TextureId=""

		elseif obj:IsA("SurfaceAppearance") then
			-- Texture layer only; safe to remove while keeping MeshPart shape.
			obj:Destroy()

		elseif obj:IsA("Decal") or obj:IsA("Texture") then
			obj:Destroy()

		elseif obj:IsA("BasePart") then
			if obj.Name=="HumanoidRootPart" then
				obj.Transparency=1
			else
				obj.Color=Color3.new(0,0,0)
				obj.Reflectance=0
				obj.Transparency=0

				-- Keep the original geometry/material as much as possible.
				if obj:IsA("MeshPart") then
					obj.TextureID=""
				end
			end

		elseif obj:IsA("BodyColors") then
			obj.HeadColor3=Color3.new(0,0,0)
			obj.LeftArmColor3=Color3.new(0,0,0)
			obj.RightArmColor3=Color3.new(0,0,0)
			obj.LeftLegColor3=Color3.new(0,0,0)
			obj.RightLegColor3=Color3.new(0,0,0)
			obj.TorsoColor3=Color3.new(0,0,0)
		end
	end)
end

local function makeBlackNameTag(plr,char)
	local head=char:FindFirstChild("Head")
	if not head then return end
	local old=head:FindFirstChild(CLOAK_TAG)
	if old then old:Destroy() end

	local guiTag=Instance.new("BillboardGui")
	guiTag.Name=CLOAK_TAG
	guiTag.AlwaysOnTop=true
	guiTag.Size=UDim2.fromOffset(180,28)
	guiTag.StudsOffset=Vector3.new(0,2.5,0)
	guiTag.MaxDistance=120
	guiTag.Parent=head

	local lbl=Instance.new("TextLabel")
	lbl.Size=UDim2.fromScale(1,1)
	lbl.BackgroundTransparency=1
	lbl.Text=tostring(plr.Name)
	lbl.TextColor3=Color3.new(0,0,0)
	lbl.TextStrokeTransparency=1
	lbl.Font=Enum.Font.GothamBold
	lbl.TextScaled=true
	lbl.Parent=guiTag
end

local cloakConnections={}
local function cloakListedCharacter(plr,char)
	if not plr or not char or not isConfiguredFarmUser(plr.Name) then return end

	stripCharacterCosmetics(char)

	local hum=char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid",5)
	if hum then
		pcall(function()
			hum.DisplayDistanceType=Enum.HumanoidDisplayDistanceType.None
			hum.NameDisplayDistance=0
			hum.HealthDisplayDistance=0
		end)

	end

	for _,obj in ipairs(char:GetDescendants()) do
		blackenCharacterObject(obj)
	end
	cleanSkillFxInCharacter(char)
	makeBlackNameTag(plr,char)

	local childConn=char.ChildAdded:Connect(function(obj)
		if not isConfiguredFarmUser(plr.Name) then return end
		task.defer(function()
			-- Example: workspace.Live.<user>.ExplodDodge
			if skillFxName(obj.Name) then
				removeSkillFxObject(obj)
				return
			end
			if isCosmeticDirectChild(obj) then
				pcall(function() obj:Destroy() end)
				return
			end
			blackenCharacterObject(obj)
		end)
	end)
	table.insert(cloakConnections,childConn)

	local c=char.DescendantAdded:Connect(function(obj)
		if not isConfiguredFarmUser(plr.Name) then return end
		task.defer(function()
			if skillFxName(obj.Name) then
				removeSkillFxObject(obj)
				return
			end

			local p=obj.Parent
			for _=1,4 do
				if not p then break end
				if skillFxName(p.Name) then
					removeSkillFxObject(obj)
					return
				end
				p=p.Parent
			end

			blackenCharacterObject(obj)
		end)
	end)
	table.insert(cloakConnections,c)
end

local function hookListedPlayer(plr)
	if not isConfiguredFarmUser(plr.Name) then return end
	if plr.Character then task.defer(cloakListedCharacter,plr,plr.Character) end
	local c=plr.CharacterAdded:Connect(function(char)
		task.wait(0.15)
		cloakListedCharacter(plr,char)
	end)
	table.insert(cloakConnections,c)
end

for _,plr in ipairs(Players:GetPlayers()) do hookListedPlayer(plr) end
table.insert(cloakConnections,Players.PlayerAdded:Connect(function(plr)
	task.defer(hookListedPlayer,plr)
end))

-- Keep removing skill/Q FX for listed MAIN/AFK characters.
-- This intentionally does NOT stop M1/block/movement AnimationTracks.
local function cleanConfiguredSkillFx()
	local live=workspace:FindFirstChild("Live")
	if not live then return end
	for _,plr in ipairs(Players:GetPlayers()) do
		if isConfiguredFarmUser(plr.Name) then
			local char=live:FindFirstChild(plr.Name) or plr.Character
			if char then cleanSkillFxInCharacter(char) end
		end
	end
end

task.spawn(function()
	while task.wait(0.12) do
		cleanConfiguredSkillFx()
	end
end)

table.insert(cloakConnections,workspace.DescendantAdded:Connect(function(obj)
	local live=workspace:FindFirstChild("Live")
	if not live then return end

	-- Only touch FX belonging to a configured farm character.
	local cur=obj
	local ownerName=nil
	while cur and cur~=live do
		if cur.Parent==live then
			ownerName=cur.Name
			break
		end
		cur=cur.Parent
	end

	if ownerName and isConfiguredFarmUser(ownerName) then
		if skillFxName(obj.Name) then
			task.defer(removeSkillFxObject,obj)
		else
			local p=obj.Parent
			for _=1,4 do
				if not p then break end
				if skillFxName(p.Name) then
					task.defer(removeSkillFxObject,obj)
					break
				end
				p=p.Parent
			end
		end
	end
end))

-- ===== ABA Smart Server Browser Balancer =====
-- Uses ABA's own ReplicatedStorage.Servers + ServerTP.
--
-- Goals:
--   * MAIN accounts keep farming while the balancer works in background.
--   * Prefer joinable Special servers whose Mode contains "FFA".
--   * Spread active MAIN accounts in balanced groups of 2-4.
--   * If Special FFA capacity is genuinely insufficient, add "Kills Team" servers.
--   * Missing/offline MAIN accounts are removed from the active plan after a TTL.
--   * When an account returns, all clients deterministically rebalance again.
--   * Never blind-hop when server data is unavailable.
--
-- No shared JSON/coordinator is required: ABA's Server Browser already replicates
-- JobIds and player UserIds to every client.

local SMART_SCAN_INTERVAL=15
local SMART_ACTIVE_TTL=45
local SMART_DISCOVERY_WAIT=12
local SMART_MOVE_COOLDOWN=45
local SMART_STABLE_CHECKS=2
local SMART_MAX_GROUP_SIZE=4

local ServersFolder=ReplicatedStorage:FindFirstChild("Servers")
local ServerTP=ReplicatedStorage:FindFirstChild("ServerTP")
local StaffServerList=ReplicatedStorage:FindFirstChild("StaffServerList")

local smartServerText="SMART: STARTING"
local smartMoveBusy=false
local smartPendingJob=nil
local smartPendingChecks=0
local smartPlanSerial=0
local smartResolvedCount=0

local smartMainNames={}
local smartMainCount=0
local smartNameByKey={}
local smartIdByKey={}
local smartKeyById={}
local smartLastSeen={}

local function smartKey(name)
	return tostring(name or ""):gsub("^%s+",""):gsub("%s+$",""):lower()
end

do
	local seen={}
	for _,name in ipairs(_G.main) do
		local key=smartKey(name)
		if key~="" and not seen[key] then
			seen[key]=true
			smartMainCount+=1
			smartMainNames[smartMainCount]=tostring(name)
			smartNameByKey[key]=tostring(name)
		end
	end
end

local mySmartKey=smartKey(myName)

local function requestServerBrowserWatch()
	if StaffServerList and StaffServerList:IsA("RemoteFunction") then
		task.spawn(function()
			pcall(function()
				StaffServerList:InvokeServer("watch")
			end)
		end)
	end
end

local function resolveMainUserId(name)
	local key=smartKey(name)
	if key=="" or smartIdByKey[key] then return end

	if sameUser(name,myName) then
		smartIdByKey[key]=LP.UserId
		smartKeyById[LP.UserId]=key
		smartResolvedCount+=1
		return
	end

	for _=1,3 do
		local ok,userId=pcall(function()
			return Players:GetUserIdFromNameAsync(name)
		end)
		if ok and tonumber(userId) then
			userId=tonumber(userId)
			smartIdByKey[key]=userId
			smartKeyById[userId]=key
			smartResolvedCount+=1
			return
		end
		task.wait(1)
	end
end

for i=1,smartMainCount do
	local name=smartMainNames[i]
	task.spawn(resolveMainUserId,name)
end

local SMART_HOP_FILE="WWHub_smart_hop_"..tostring(LP.UserId)..".txt"
local smartLastMoveEpoch=0

pcall(function()
	if type(readfile)=="function" and isfile and isfile(SMART_HOP_FILE) then
		smartLastMoveEpoch=tonumber(readfile(SMART_HOP_FILE)) or 0
	end
end)

local function saveSmartMoveTime()
	smartLastMoveEpoch=os.time()
	pcall(function()
		if type(writefile)=="function" then
			writefile(SMART_HOP_FILE,tostring(smartLastMoveEpoch))
		end
	end)
end

local function currentBrowserJobId()
	return tostring(workspace:GetAttribute("ServerBrowserJobId") or game.JobId or "")
end

local function readServerFolder(folder)
	if not folder or not folder:IsA("Folder") then return nil end

	local attrs=folder:GetAttributes()
	local users={}
	local userCount=0
	local listedCount=0

	for _,child in ipairs(folder:GetChildren()) do
		local uid=tonumber(child.Name)
		if uid then
			userCount+=1
			users[userCount]=uid
			if smartKeyById[uid] then
				listedCount+=1
				smartLastSeen[smartKeyById[uid]]=os.clock()
			end
		end
	end

	local playerCount=tonumber(attrs.PlayerCount) or userCount
	local maxPlayers=tonumber(attrs.MaxPlayers) or 16
	if maxPlayers<1 then maxPlayers=16 end

	local outsiderCount=math.max(0,playerCount-listedCount)
	local botCapacity=math.max(0,maxPlayers-outsiderCount)

	return {
		JobId=tostring(folder.Name),
		Type=tostring(attrs.Type or ""),
		Mode=tostring(attrs.Mode or ""),
		Map=tostring(attrs.Map or ""),
		Region=tostring(attrs.ServerRegion or ""),
		PlayerCount=playerCount,
		MaxPlayers=maxPlayers,
		ListedCount=listedCount,
		OutsiderCount=outsiderCount,
		BotCapacity=botCapacity,
		Users=users
	}
end

local function scanServerBrowser()
	local ffa={}
	local killsTeam={}
	local ffaCount=0
	local teamCount=0

	if not ServersFolder or not ServersFolder.Parent then
		ServersFolder=ReplicatedStorage:FindFirstChild("Servers")
	end
	if not ServerTP or not ServerTP.Parent then
		ServerTP=ReplicatedStorage:FindFirstChild("ServerTP")
	end
	if not ServersFolder then
		return ffa,ffaCount,killsTeam,teamCount
	end

	for _,folder in ipairs(ServersFolder:GetChildren()) do
		local data=readServerFolder(folder)
		if data then
			local typeLower=data.Type:lower()
			local modeLower=data.Mode:lower()
			local joinable=(data.JobId==currentBrowserJobId()) or data.PlayerCount<data.MaxPlayers

			if typeLower=="special" and joinable and data.BotCapacity>=2 then
				if modeLower:find("ffa",1,true) then
					ffaCount+=1
					ffa[ffaCount]=data
				elseif modeLower:find("kills team",1,true) or modeLower:find("kill team",1,true) then
					teamCount+=1
					killsTeam[teamCount]=data
				end
			end
		end
	end

	local function better(a,b)
		-- Keep servers already containing our MAIN accounts whenever possible.
		if a.ListedCount~=b.ListedCount then
			return a.ListedCount>b.ListedCount
		end
		-- Then prefer fewer outsiders / lower population.
		if a.OutsiderCount~=b.OutsiderCount then
			return a.OutsiderCount<b.OutsiderCount
		end
		if a.PlayerCount~=b.PlayerCount then
			return a.PlayerCount<b.PlayerCount
		end
		return a.JobId<b.JobId
	end

	table.sort(ffa,better)
	table.sort(killsTeam,better)

	return ffa,ffaCount,killsTeam,teamCount
end

local function activeMainKeys()
	local now=os.clock()
	local active={}
	local activeCount=0

	-- Current client is always active.
	smartLastSeen[mySmartKey]=now

	-- Also touch MAIN accounts that are directly visible in this server by name.
	for _,plr in ipairs(Players:GetPlayers()) do
		local key=smartKey(plr.Name)
		if smartNameByKey[key] then
			smartLastSeen[key]=now
		end
	end

	for i=1,smartMainCount do
		local name=smartMainNames[i]
		local key=smartKey(name)
		local seenAt=smartLastSeen[key]

		if key==mySmartKey or (seenAt and now-seenAt<=SMART_ACTIVE_TTL) then
			activeCount+=1
			active[activeCount]=key
		end
	end

	return active,activeCount
end

local function addSelected(selected,selectedSet,server)
	if not server or selectedSet[server.JobId] then return false end
	selectedSet[server.JobId]=true
	server.Assigned=0
	server.AssignCapacity=math.max(2,math.min(SMART_MAX_GROUP_SIZE,server.BotCapacity))
	selected[selected.count+1]=server
	selected.count+=1
	return true
end

local function totalSelectedCapacity(selected)
	local total=0
	for i=1,selected.count do
		total+=tonumber(selected[i].AssignCapacity) or 0
	end
	return total
end

local function makeSmartPlan()
	requestServerBrowserWatch()

	local ffa,ffaCount,killsTeam,teamCount=scanServerBrowser()
	local active,activeCount=activeMainKeys()

	if activeCount<=0 then
		return nil,{
			reason="NO ACTIVE MAIN",
			activeCount=activeCount,
			ffaCount=ffaCount,
			teamCount=teamCount
		}
	end

	if activeCount==1 then
		-- Nothing useful to merge with. Keep farming current server; do not blind-hop.
		return nil,{
			reason="SOLO KEEP FARMING",
			activeCount=activeCount,
			ffaCount=ffaCount,
			teamCount=teamCount
		}
	end

	local idealGroups=math.max(1,math.floor(activeCount/2))
	local minGroups=math.max(1,math.ceil(activeCount/SMART_MAX_GROUP_SIZE))

	local selected={count=0}
	local selectedSet={}

	if ffaCount>0 then
		local desiredFFA=math.min(idealGroups,ffaCount)
		for i=1,desiredFFA do
			addSelected(selected,selectedSet,ffa[i])
		end

		-- Only use Kills Team when FFA alone cannot provide enough 2-4 player groups.
		local j=1
		while selected.count<minGroups and j<=teamCount do
			addSelected(selected,selectedSet,killsTeam[j])
			j+=1
		end

		-- Capacity check: if selected FFA servers are too crowded, add spare FFA,
		-- then Kills Team, but never exceed the ideal number of groups.
		local extraFFA=desiredFFA+1
		while totalSelectedCapacity(selected)<activeCount and selected.count<idealGroups and extraFFA<=ffaCount do
			addSelected(selected,selectedSet,ffa[extraFFA])
			extraFFA+=1
		end
		while totalSelectedCapacity(selected)<activeCount and selected.count<idealGroups and j<=teamCount do
			addSelected(selected,selectedSet,killsTeam[j])
			j+=1
		end
	else
		-- No Special FFA visible: Kills Team is the fallback.
		local desiredTeam=math.min(idealGroups,teamCount)
		for i=1,desiredTeam do
			addSelected(selected,selectedSet,killsTeam[i])
		end
	end

	if selected.count<=0 then
		return nil,{
			reason="NO SPECIAL FFA/KILLS TEAM",
			activeCount=activeCount,
			ffaCount=ffaCount,
			teamCount=teamCount
		}
	end

	-- Stable server ordering makes every Roblox instance compute the same mapping.
	table.sort(selected,function(a,b)
		return a.JobId<b.JobId
	end)

	local targetByKey={}
	local groupSizeByJob={}
	local assignedCount=0
	local cursor=1

	-- Round-robin keeps group sizes balanced while respecting per-server capacity.
	for i=1,activeCount do
		local key=active[i]
		local placed=false
		local tries=0

		while tries<selected.count do
			local server=selected[cursor]
			if server.Assigned<server.AssignCapacity then
				server.Assigned+=1
				targetByKey[key]=server
				groupSizeByJob[server.JobId]=server.Assigned
				assignedCount+=1
				placed=true
				cursor=(cursor%selected.count)+1
				break
			end

			cursor=(cursor%selected.count)+1
			tries+=1
		end

		if not placed then
			-- No safe 2-4 slot remains. This account simply keeps farming its
			-- current server until the browser shows enough capacity.
			targetByKey[key]=nil
		end
	end

	smartPlanSerial+=1

	return targetByKey,{
		reason="OK",
		activeCount=activeCount,
		configuredCount=smartMainCount,
		ffaCount=ffaCount,
		teamCount=teamCount,
		selectedCount=selected.count,
		assignedCount=assignedCount,
		groupSizeByJob=groupSizeByJob,
		serial=smartPlanSerial
	}
end

local function smartCanMove()
	return (os.time()-smartLastMoveEpoch)>=SMART_MOVE_COOLDOWN
end

local function smartMoveTo(server,meta)
	if not server or not ServerTP then return false end
	if smartMoveBusy then return false end
	if server.JobId==currentBrowserJobId() then return false end
	if not smartCanMove() then return false end

	smartMoveBusy=true
	saveSmartMoveTime()

	local groupSize=(meta.groupSizeByJob and meta.groupSizeByJob[server.JobId]) or server.Assigned or 0
	local modeLabel=server.Mode~="" and server.Mode or "Special"

	notifyAction(
		"Smart Server",
		("Move -> %s | group %d | %d/%d"):format(
			modeLabel,
			groupSize,
			meta.activeCount or 0,
			meta.configuredCount or smartMainCount
		),
		5
	)

	warn(("[WWHub] Smart ServerTP -> %s | %s | group=%d"):format(
		server.JobId,
		modeLabel,
		groupSize
	))

	local ok,err=pcall(function()
		ServerTP:FireServer(server.JobId)
	end)
	if not ok then
		warn("[WWHub] ServerTP failed: "..tostring(err))
	end

	task.delay(12,function()
		smartMoveBusy=false
	end)

	return ok
end

task.spawn(function()
	if not IS_MAIN then
		smartServerText="SMART: MAIN ONLY"
		return
	end

	if smartMainCount<=1 then
		smartServerText="SMART: OFF (1 MAIN)"
		return
	end

	requestServerBrowserWatch()

	-- Let ABA populate ReplicatedStorage.Servers and resolve configured usernames.
	task.wait(SMART_DISCOVERY_WAIT)

	while LP and LP.Parent do
		requestServerBrowserWatch()

		-- scanServerBrowser touches lastSeen before activeMainKeys() is evaluated.
		local plan,meta=makeSmartPlan()
		local target=plan and plan[mySmartKey] or nil

		if not target then
			smartPendingJob=nil
			smartPendingChecks=0
			smartServerText=("SMART: %s | %d/%d | FFA:%d TEAM:%d"):format(
				tostring(meta.reason or "WAIT"),
				tonumber(meta.activeCount) or 0,
				smartMainCount,
				tonumber(meta.ffaCount) or 0,
				tonumber(meta.teamCount) or 0
			)
		else
			local current=currentBrowserJobId()
			local groupSize=(meta.groupSizeByJob and meta.groupSizeByJob[target.JobId]) or target.Assigned or 0
			local shortMode=target.Mode:lower():find("ffa",1,true) and "FFA" or "TEAM"

			if target.JobId==current then
				smartPendingJob=nil
				smartPendingChecks=0
				smartServerText=("SMART: %s G%d | %d/%d | BALANCED"):format(
					shortMode,
					groupSize,
					meta.activeCount or 0,
					smartMainCount
				)
			else
				if smartPendingJob==target.JobId then
					smartPendingChecks+=1
				else
					smartPendingJob=target.JobId
					smartPendingChecks=1
				end

				local cooldownLeft=math.max(0,SMART_MOVE_COOLDOWN-(os.time()-smartLastMoveEpoch))
				smartServerText=("SMART: %s G%d | %d/%d | MOVE %d/%d"):format(
					shortMode,
					groupSize,
					meta.activeCount or 0,
					smartMainCount,
					smartPendingChecks,
					SMART_STABLE_CHECKS
				)

				if smartPendingChecks>=SMART_STABLE_CHECKS and cooldownLeft<=0 then
					smartMoveTo(target,meta)
				end
			end
		end

		task.wait(SMART_SCAN_INTERVAL)
	end
end)

-- ===== Vars =====
local FARM_BASE = Vector3.new(200, 2000, 200)
local FARM_PAIR_SPACING = 0.15
local FARM_ALT_OFFSET = 0.15

local function getRoleIndex(list)
	for i,n in ipairs(list) do
		if sameUser(n,myName) then return i end
	end
	return 1
end

local farmIndex = IS_AFK and getRoleIndex(_G.afk) or getRoleIndex(_G.main)
local pairX = (farmIndex - 1) * FARM_PAIR_SPACING
local farmPos = FARM_BASE + Vector3.new(pairX, 0, IS_AFK and FARM_ALT_OFFSET or 0)
local altCFrame = CFrame.new(farmPos)
local pauseCFrame = CFrame.new(156, 1, -43)
local baseName    = "WWHub_BasePlate"
local tpDist      = 0.55
local safeLimit   = 8


local loopMain      = false
local starting      = false
local farmRequested = false
local farmRunToken  = 0
local selectingTeam = false
local pointsCapped  = false
local roundPaused   = false
local roundPauseReason = nil
local roundResetting   = false
local handledChar   = nil
local timerTpDone   = false
local endRoundResetDone = false
local gui           = nil
_G.pointcap = math.max(100, tonumber(_G.pointcap) or 1000)

-- ===== Gold Progress Tracking =====
local scriptStartTime = os.time()
local startGold       = nil
local farmStartTime   = nil
local farmStartGold   = nil

local WebhookURL = "https://discord.com/api/webhooks/1453628734090514533/ddACObJX5Iuv966TcspBAEmkd5Er2ZfiVCMdoHzyONWLJ1CoqlDaAn3vg9D1GiZkvPoR"
local _request
if not pcall(function() _request = request or http_request or http.request end) then
	_request = function() end
end

local avatarUrl = ""
task.spawn(function()
	pcall(function()
		local url = ("https://thumbnails.roblox.com/v1/users/avatar-headshot?userIds=%d&size=420x420&format=Png&isCircular=false"):format(LP.UserId)
		local r = Http:JSONDecode(game:HttpGet(url))
		if r and r.data and r.data[1] then avatarUrl = r.data[1].imageUrl or "" end
	end)
end)

-- capture starting Gold as soon as it's available (retry up to ~50s)
task.spawn(function()
	for _ = 1, 50 do
		local ok, v = pcall(function()
			return LP:WaitForChild("ReplicatedStats"):WaitForChild("Gold").Value
		end)
		if ok and v then startGold = v break end
		task.wait(1)
	end
	if startGold == nil then startGold = 0 end
end)

-- ===== Helpers =====
local function getChar() return LP.Character end
local function getHRP()  local c = getChar() return c and c:FindFirstChild("HumanoidRootPart") end
local function getHum()  local c = getChar() return c and c:FindFirstChildOfClass("Humanoid") end
local function getInput()
	local bp = LP:FindFirstChild("Backpack") local c = getChar()
	return (bp and bp:FindFirstChild("Input")) or (c and c:FindFirstChild("Input")) or LP:FindFirstChild("Input")
end
local function fireInput(action, data)
	local inp = getInput() if not inp then return end
	pcall(function() if data then inp:FireServer(action, data) else inp:FireServer(action) end end)
end
local function makeBase()
	local b = workspace:FindFirstChild(baseName)
	if not b then
		b = Instance.new("Part") b.Name = baseName b.Anchored = true
		b.Color = Color3.fromRGB(35,35,35) b.Material = Enum.Material.SmoothPlastic b.Parent = workspace
	end
	b.Size = Vector3.new(500, 0.4, 500)
	b.CFrame = CFrame.new(altCFrame.Position - Vector3.new(0, 3, 0))
end
local function isNearCF(cf, limit)
	local hrp = getHRP() if not hrp or not cf then return false end
	return (hrp.Position - cf.Position).Magnitude <= (limit or tpDist)
end
local function tpToCF(cf)
	if not cf then return false end makeBase()
	local hrp = getHRP() if not hrp then return false end
	pcall(function()
		hrp.AssemblyLinearVelocity  = Vector3.new(0,0,0)
		hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
		hrp.CFrame = cf
	end)
	return true
end
local function getMainCF()
	local p = altCFrame.Position
	return CFrame.lookAt(p, p + Vector3.new(0,0,1))
end
local function tpToSafeZone()
	local cf = pauseCFrame
	task.spawn(function()
		for _ = 1, 20 do
			if not gui or not gui.Parent then break end
			local hrp = getHRP()
			if hrp then
				pcall(function()
					hrp.AssemblyLinearVelocity  = Vector3.new(0,0,0)
					hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
					hrp.CFrame = cf
				end)
				task.wait(0.25) hrp = getHRP()
				if hrp and (hrp.Position - cf.Position).Magnitude <= safeLimit then break end
			end
			task.wait(0.25)
		end
	end)
end

-- ===== Bakugou Q Remote =====
local function fireBakugouQ()
	if not IS_MAIN then return end
	local inp=getInput()
	local hrp=getHRP()
	if not inp or not hrp then return end

	-- Q visual objects are short-lived; 0.35s catches the spawned workspace.Explosion
	-- without touching unrelated effects for the rest of the game.
	qExplosionSuppressUntil=os.clock()+0.35

	pcall(function()
		inp:FireServer("dodge",{
			TagName="ExplodDodge",
			explod=Enum.KeyCode.W,
			ServerSwoosh=false,
			pos=hrp.CFrame
		})
	end)
end

local function forceFieldOff()
	fireInput("ForceFieldOff")
end

-- Anti-AFK
pcall(function()
	LP.Idled:Connect(function()
		pcall(function()
			VUser:CaptureController()
			VUser:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
			task.wait(0.1)
			VUser:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
		end)
	end)
end)

-- Game-level anti-AFK pulse:
-- every 5 minutes (default), send a real Space input + Humanoid jump.
-- Skip obvious team-pad selection moments so it does not knock the player off a pad.
task.spawn(function()
	while task.wait(antiAfkJumpSec) do
		if not gui or not gui.Parent then break end
		if not (farmRequested or loopMain) then continue end

		local teamPadPresent=workspace:FindFirstChild("Red Team")
			or workspace:FindFirstChild("Blue Team")
			or workspace:FindFirstChild("Green Team")
			or workspace:FindFirstChild("Yellow Team")
		if selectingTeam or teamPadPresent then
			continue
		end

		local char=LP.Character
		local hum=char and char:FindFirstChildOfClass("Humanoid")
		if hum and hum.Health>0 then
			pcall(function()
				VIM:SendKeyEvent(true,Enum.KeyCode.Space,false,game)
				hum.Jump=true
				hum:ChangeState(Enum.HumanoidStateType.Jumping)
				task.wait(0.08)
				VIM:SendKeyEvent(false,Enum.KeyCode.Space,false,game)
			end)
			warn("[WWHub] Anti-AFK jump pulse")
		end
	end
end)

-- ===== Farm Character Verify =====
local characterFixing = false

local function characterKey(name)
	return tostring(name or ""):lower():gsub("[^%w]","")
end

local function readEquippedCharacter()
	local char = LP.Character
	for _,container in ipairs({char or false,LP:FindFirstChild("ReplicatedStats") or false,LP}) do
		if container then
			for _,field in ipairs({"CharacterName","CurrentCharacter","EquippedCharacter"}) do
				local attr = container:GetAttribute(field)
				if type(attr) == "string" and attr ~= "" then return attr end
				local value = container:FindFirstChild(field)
				if value and value:IsA("StringValue") and value.Value ~= "" then return value.Value end
			end
		end
	end
	return nil
end

local function getChooseRemote()
	local bp = LP:FindFirstChild("Backpack")
	local c = LP.Character
	local st = (bp and bp:FindFirstChild("ServerTraits")) or (c and c:FindFirstChild("ServerTraits")) or LP:FindFirstChild("ServerTraits")
	return st and st:FindFirstChild("Choose")
end

local function fireSelectRemote(charName)
	local inp = getInput()
	if inp then
		local ok = pcall(function()
			inp:FireServer("CharacterButton",charName)
			task.wait(0.15)
			local fresh = getInput()
			if fresh then fresh:FireServer("ClickPlay") end
		end)
		if ok then return true end
	end

	local choose = getChooseRemote()
	if choose then
		return pcall(function()
			choose:FireServer(charName)
			task.wait(0.15)
			local fresh = getChooseRemote()
			if fresh then fresh:FireServer("PLAY") end
		end)
	end
	return false
end

local function farmCharacterName()
	return IS_MAIN and "Bakugou" or "Ichigo"
end

local function isKnownWrongFarmCharacter()
	local equipped = readEquippedCharacter()
	if not equipped then return false,nil end
	local target = farmCharacterName()
	return characterKey(equipped) ~= characterKey(target),equipped
end

local function ensureFarmCharacter(reason)
	if characterFixing then return false end
	if not farmRequested and not loopMain then return false end

	characterFixing=true
	local target=farmCharacterName()
	local equipped=readEquippedCharacter()

	if equipped and characterKey(equipped)==characterKey(target) then
		characterFixing=false
		return true
	end

	if equipped then
		notifyAction("Character Fix","Wrong character: "..tostring(equipped).." -> "..target.." | "..tostring(reason or ""),5)
	end

	-- ABA often does not expose the equipped character name.
	-- In that case don't burn 8+ seconds waiting for a value that never appears.
	if not equipped then
		local sent=false
		for _=1,2 do
			if not farmRequested and not loopMain then
				characterFixing=false
				return false
			end
			sent=fireSelectRemote(target) or sent
			pcall(function()
				local pg=LP:FindFirstChild("PlayerGui")
				local respawning=pg and pg:FindFirstChild("Respawning")
				local done=respawning and respawning:FindFirstChild("Done")
				if done then done:FireServer() end
			end)
			task.wait(0.18)
		end
		characterFixing=false
		return sent
	end

	local deadline=os.clock()+3
	while os.clock()<deadline do
		if not farmRequested and not loopMain then
			characterFixing=false
			return false
		end

		fireSelectRemote(target)
		task.wait(0.2)
		equipped=readEquippedCharacter()
		if equipped and characterKey(equipped)==characterKey(target) then
			notifyAction("Character Ready",target.." confirmed",3)
			characterFixing=false
			return true
		end
	end

	notifyAction("Character Error","Could not confirm "..target.." | observed "..tostring(readEquippedCharacter()),5)
	characterFixing=false
	return false
end

-- ===== Bakugou Auto Buy =====
local bakugouBuyTried = false

local function buyBakugou()
	if not IS_MAIN or bakugouBuyTried then return end
	local ok = pcall(function()
		local backpack = LP:WaitForChild("Backpack",10)
		local serverTraits = backpack and backpack:WaitForChild("ServerTraits",10)
		local choose = serverTraits and serverTraits:WaitForChild("Choose",10)
		if not choose then error("Choose remote missing") end
		choose:FireServer("Bakugou")
		task.wait(0.05)
		choose:FireServer("PLAY")
		task.wait(0.12)
	end)
	if ok then bakugouBuyTried = true end
end

-- ===== Character =====
local function fireRespawnDone()
	local r = LP:WaitForChild("PlayerGui"):FindFirstChild("Respawning")
	if r and r:FindFirstChild("Done") then pcall(function() r.Done:FireServer() end) end
end
local function afterCharLoaded(char)
	if not char or handledChar == char then return end
	handledChar = char
	char:WaitForChild("HumanoidRootPart",10)
	char:WaitForChild("Humanoid",10)
	task.wait(0.15)
	fireRespawnDone()
	task.wait(0.05)
	forceFieldOff()
	if isConfiguredFarmUser(LP.Name) then
		task.defer(cloakListedCharacter,LP,char)
	end
end
local function resetChar(reason)
	if not farmRequested and not loopMain then return false end
	resetSequence+=1
	notifyAction("WWHub Reset",("#%d | %s"):format(resetSequence,reason or "Unknown reason"),5)

	local c=getChar()
	if c then
		local h=c:FindFirstChildOfClass("Humanoid")
		if h and h.Health>0 then h.Health=0 end
	end

	pcall(function()
		game:GetService("ReplicatedStorage"):WaitForChild("Loaded"):FireServer()
	end)

	task.wait(0.35)
	if not farmRequested and not loopMain then return false end

	local nc=getChar()
	if nc then afterCharLoaded(nc) end
	return true
end

-- ===== Stats =====
local function getPoints() local ok,v = pcall(function() return LP.leaderstats.Points.Value end) return ok and v or 0 end
local function getTimerValue()
	local ok,v = pcall(function()
		local hud = LP.PlayerGui:FindFirstChild("HUD") if not hud then return 0 end
		local t = hud:FindFirstChild("Timer") if not t then return 0 end
		return (t:IsA("TextLabel") or t:IsA("TextBox")) and (tonumber(t.Text) or 0) or (tonumber(t.Value) or 0)
	end)
	return ok and v or 0
end
local lastKnownLevel = nil
local function getLevel()
	local ok,v = pcall(function()
		-- Try replicated/non-GUI values first.
		local attr = LP:GetAttribute("Level")
		if tonumber(attr) then return tonumber(attr) end

		local replicatedStats = LP:FindFirstChild("ReplicatedStats")
		local leaderstats = LP:FindFirstChild("leaderstats")
		local char = LP.Character
		local charStats = char and char:FindFirstChild("Stats")

		for _,root in ipairs({replicatedStats or false,leaderstats or false,LP,char or false,charStats or false}) do
			if root then
				local level = root:FindFirstChild("Level")
				if level and (level:IsA("IntValue") or level:IsA("NumberValue")) then
					return tonumber(level.Value)
				end
			end
		end

		for _,obj in ipairs(LP:GetDescendants()) do
			if obj.Name == "Level" and (obj:IsA("IntValue") or obj:IsA("NumberValue")) then
				return tonumber(obj.Value)
			end
		end

		-- ABA currently exposes the actual displayed level through HUD.LocalHUDScript.
		local pg = LP:FindFirstChild("PlayerGui")
		local hud = pg and pg:FindFirstChild("HUD")
		local rb = hud and hud:FindFirstChild("RightBotCorner")
		local line2 = rb and rb:FindFirstChild("Line2")
		local lvl = line2 and line2:FindFirstChild("Lvl")
		if lvl and (lvl:IsA("TextLabel") or lvl:IsA("TextBox")) then
			local n = tonumber(tostring(lvl.Text):match("[Ll]vl%.?%s*(%d+)") or tostring(lvl.Text):match("(%d+)"))
			if n then return n end
		end

		return lastKnownLevel
	end)
	if ok and tonumber(v) then lastKnownLevel = tonumber(v) end
	return lastKnownLevel
end

local PRESTIGE_POOL = {
	["Achilles"]=true,["Chiaotzu"]=true,["Deidara"]=true,["Fubuki"]=true,
	["Hercule Satan"]=true,["Mr President"]=true,["Ritsu"]=true,
	["Satsuki Kiryuin"]=true,["Shadow DIO"]=true,["Shisui"]=true,
	["Super Dummy"]=true,["Tobi"]=true,["Tokita Ohma"]=true,
	["Zamasu [Fused]"]=true,["Zenitsu"]=true
}

local function getPrestigeCount()
	local ok,count = pcall(function()
		local rs = LP:FindFirstChild("ReplicatedStats")
		local unlocked = rs and rs:FindFirstChild("Unlocked")
		if not unlocked or type(unlocked.Value) ~= "string" then return 0 end
		local data = Http:JSONDecode(unlocked.Value)
		if type(data) ~= "table" then return 0 end
		local n = 0
		for name in pairs(PRESTIGE_POOL) do
			if data[name] ~= nil then n += 1 end
		end
		return n
	end)
	return ok and tonumber(count) or 0
end

-- ===== Point Cap Visibility Safety =====
local function isFarmAccount(name)
	for _,n in ipairs(_G.main) do
		if sameUser(n,name) then return true end
	end
	for _,n in ipairs(_G.afk) do
		if sameUser(n,name) then return true end
	end
	return false
end

local function hasOutsider()
	for _, plr in ipairs(Players:GetPlayers()) do
		if not isFarmAccount(plr.Name) then
			return true
		end
	end
	return false
end

-- ===== Gold / Single Point Cap =====
local function getGold()
	local ok,v=pcall(function()
		return LP:WaitForChild("ReplicatedStats"):WaitForChild("Gold").Value
	end)
	return ok and v or 0
end

local function getEffectiveCap()
	return math.max(100,tonumber(_G.pointcap) or 1000)
end

local function adjustEffectiveCap(delta)
	_G.pointcap=math.max(100,getEffectiveCap()+delta)
	ENV.pointcap=_G.pointcap
	if getPoints()<getEffectiveCap() then
		pointsCapped=false
	end
end

-- Farm profit/time is measured from the moment a farm run actually starts.
local function markFarmStart()
	farmStartTime=os.time()
	farmStartGold=getGold()
end

local function getFarmProgress()
	local current=getGold()
	local base=farmStartGold or current
	local gained=current-base
	local elapsed=farmStartTime and math.max(0,os.time()-farmStartTime) or 0
	return gained,elapsed
end

local function getGoldProgressText()
	local gained,elapsed=getFarmProgress()
	local hh=math.floor(elapsed/3600)
	local mm=math.floor((elapsed%3600)/60)
	local sign=gained>=0 and "+" or ""
	return string.format("%s%d Gold | Farm %02d:%02d",sign,gained,hh,mm)
end

-- ===== Webhook =====
local function sendWebhook(label)
	task.spawn(function()
		pcall(function()
			local money, lvl, pts = "N/A","N/A","N/A"
			pcall(function()
				money = tostring(LP:WaitForChild("ReplicatedStats"):WaitForChild("Gold").Value)
				local realLevel = getLevel()
				if realLevel ~= nil then lvl = tostring(realLevel) else lvl = "?" end
				pts = tostring(LP.leaderstats.Points.Value)
			end)
			-- ถ้า level >= 100 ให้ติดตรา max level ต่อท้าย
			pcall(function()
				local lvlNum = tonumber(lvl:match("%d+"))
				if lvlNum and lvlNum >= 100 then
					lvl = lvl .. " 🟢"
				end
			end)
			local progressText = "N/A"
			pcall(function() progressText = getGoldProgressText() end)
			_request({
				Url = WebhookURL, Method = "POST",
				Headers = {["Content-Type"] = "application/json"},
				Body = Http:JSONEncode({
					username = "WW Hub", avatar_url = avatarUrl,
					embeds = {{
						title = "⚡ WWHub — "..label, color = 0x46C864,
						thumbnail = {url = avatarUrl},
						fields = {
							{name="👤 Player", value=LP.DisplayName.." (@"..LP.Name..")", inline=false},
							{name="💰 Money",  value=money, inline=true},
							{name="⭐ Level",  value=lvl,   inline=true},
							{name="🎯 Points", value=pts,   inline=true},
							{name="📈 Progress", value=progressText, inline=false},
						},
						footer    = {text = "WWHub • "..os.date("%d/%m/%Y %H:%M:%S")},
						timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ")
					}}
				})
			})
		end)
	end)
end

-- ===== Blocked Mode =====
local function isVisibleGuiObject(obj)
	if not obj or not obj:IsA("GuiObject") then return false end
	if not obj.Visible then return false end
	local p = obj.Parent
	while p and p ~= LP.PlayerGui do
		if p:IsA("GuiObject") and not p.Visible then return false end
		p = p.Parent
	end
	return true
end

local function getVisibleTeamModeHint()
	local pg=LP:FindFirstChild("PlayerGui")
	if not pg then return nil end
	local function matchMode(raw)
		local tx=tostring(raw or ""):lower()
		if tx:find("team battle",1,true) then return "Team Battle" end
		if tx:find("kills team",1,true) then return "Kills Team" end
		if tx:find("3 teams",1,true) or tx:find("three teams",1,true) then return "3 Teams" end
		return nil
	end
	for _,obj in ipairs(pg:GetDescendants()) do
		if (obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton")) and isVisibleGuiObject(obj) then
			local mode=matchMode(obj.Text)
			if mode then return mode end
		elseif obj:IsA("StringValue") then
			local nm=tostring(obj.Name or ""):lower()
			if nm:find("mode",1,true) or nm:find("game",1,true) then
				local mode=matchMode(obj.Value)
				if mode then return mode end
			end
		end
	end
	return nil
end

local function getBlockedMode()
	local pg = LP:FindFirstChild("PlayerGui")
	if not pg then return nil end
	local lb = pg:FindFirstChild("CustomLeaderboard")
	if not lb then return nil end
	local m = lb:FindFirstChild("Main") or lb

	-- Strong Juggernaut detection.
	for _,obj in ipairs(m:GetDescendants()) do
		local nm = tostring(obj.Name or ""):lower()
		if nm == "juggernaut" or nm:find("juggernaut",1,true) then
			return "Juggernaut"
		end
		if (obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton")) and isVisibleGuiObject(obj) then
			local tx = tostring(obj.Text or ""):lower()
			if tx:find("juggernaut",1,true) then return "Juggernaut" end
		end
	end

	-- In any detected team round, generic Lives must NEVER trigger the blocked-mode reset.
	local teamMode=getVisibleTeamModeHint()
	local teamPadPresent=workspace:FindFirstChild("Red Team") or workspace:FindFirstChild("Blue Team")
		or workspace:FindFirstChild("Green Team") or workspace:FindFirstChild("Yellow Team")
	if LP.Team~=nil or teamMode or teamPadPresent then
		return nil
	end

	-- Lives must contain an actual numeric lives value, not only an object named "Lives".
	for _,obj in ipairs(m:GetDescendants()) do
		if (obj:IsA("IntValue") or obj:IsA("NumberValue")) and tostring(obj.Name or ""):lower() == "lives" then
			local n = tonumber(obj.Value)
			if n and n > 0 then return "Lives" end
		elseif (obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton")) and isVisibleGuiObject(obj) then
			local tx = tostring(obj.Text or "")
			local n = tx:match("[Ll][Ii][Vv][Ee][Ss]%s*:%s*(%d+)") or tx:match("[Ll][Ii][Vv][Ee][Ss]%s+(%d+)")
			if tonumber(n) and tonumber(n) > 0 then return "Lives" end
		end
	end
	return nil
end

local function getLivesValue()
	local pg = LP:FindFirstChild("PlayerGui")
	if not pg then return nil end

	for _, obj in ipairs(pg:GetDescendants()) do
		if obj:IsA("IntValue") or obj:IsA("NumberValue") then
			if (obj.Name or ""):lower() == "lives" then
				return tonumber(obj.Value)
			end
		elseif obj:IsA("TextLabel") or obj:IsA("TextBox") or obj:IsA("TextButton") then
			local tx = tostring(obj.Text or "")
			local n = tx:match("[Ll][Ii][Vv][Ee][Ss]%s*:%s*(%d+)")
				or tx:match("[Ll][Ii][Vv][Ee][Ss]%s+(%d+)")
			if n then return tonumber(n) end
		end
	end
	return nil
end

local function waitForRespawn(oldChar, timeout)
	local started = os.clock()
	timeout = timeout or 10
	while gui and gui.Parent and loopMain and (os.clock() - started) < timeout do
		local lives = getLivesValue()
		if lives ~= nil and lives <= 0 then
			return false
		end

		local c = getChar()
		local h = c and c:FindFirstChildOfClass("Humanoid")
		local hrp = c and c:FindFirstChild("HumanoidRootPart")
		if c and c ~= oldChar and h and hrp and h.Health > 0 then
			afterCharLoaded(c)
			return true
		end
		task.wait(0.1)
	end
	return false
end

local function drainLives()
	local initialLives = getLivesValue()
	local resetCount = 0
	local missingSince = nil

	while gui and gui.Parent and loopMain and roundPaused do
		local lives = getLivesValue()
		local bm = getBlockedMode()

		if lives ~= nil then
			missingSince = nil
			if lives <= 0 then break end
		elseif not bm then
			missingSince = missingSince or os.clock()

			-- ระหว่างตาย/เกิดใหม่ UI จะหายชั่วคราว ห้ามหยุด reset ทันที
			if initialLives then
				if resetCount >= initialLives and (os.clock() - missingSince) >= 2 then
					break
				end
			elseif (os.clock() - missingSince) >= 5 then
				break
			end
		else
			missingSince = nil
		end

		local oldChar = getChar()
		local hum = oldChar and oldChar:FindFirstChildOfClass("Humanoid")
		if hum and hum.Health > 0 then
			resetCount += 1
			resetSequence += 1
			notifyAction("WWHub Reset",("#%d | Blocked %s | Lives=%s | Drain #%d"):format(
				resetSequence,tostring(bm or roundPauseReason or "Unknown"),tostring(lives or "?"),resetCount
			),6)
			pcall(function() hum.Health = 0 end)

			pcall(function()
				ReplicatedStorage:WaitForChild("Loaded"):FireServer()
			end)

			waitForRespawn(oldChar, 10)
		else
			task.wait(0.25)
		end

		task.wait(0.5)
	end
end

local function pauseFarm(reason)
	if roundPaused then return end
	roundPaused = true roundPauseReason = reason or "Blocked"
	pointsCapped = false
	notifyAction("Farm Paused","Blocked mode confirmed: "..roundPauseReason.." -> reset/drain",6)
	sendWebhook("Farm Paused — "..roundPauseReason)
	if roundResetting then return end roundResetting = true
	task.spawn(function()
		-- Juggernaut/Lives: reset repeatedly until Lives reaches 0
		drainLives()
		tpToSafeZone()

		local clear = 0
		while gui and gui.Parent and loopMain and roundPaused do
			local bm = getBlockedMode()
			if bm then
				clear = 0
				tpToSafeZone()
				task.wait(1)
			else
				clear += 1
				if clear >= 3 then break end
				task.wait(1)
			end
		end

		local last = roundPauseReason
		roundPaused=false roundPauseReason=nil roundResetting=false
		if gui and gui.Parent and loopMain then
			notifyAction("Farm Resumed","Blocked mode cleared: "..tostring(last),5)
			sendWebhook("Farm Resumed — "..last)
		end
	end)
end

-- Timer watchdog (now uses dynamic Gold-based cap)
task.spawn(function()
	while true do
		task.wait(0.3)
		if loopMain then
			local pts = getPoints() local timer = getTimerValue()
			local capNow = getEffectiveCap()

			-- No outsiders = no point cap. Re-enable automatically if an outsider joins.
			if not hasOutsider() then
				pointsCapped = false
			elseif pts >= capNow then
				pointsCapped = true
			else
				pointsCapped = false
			end
			if timer > 0 and timer <= 2 and not timerTpDone and not roundPaused then
				timerTpDone = true
				notifyAction("Farm Action","End round timer <= 2 -> TP Safe Zone",4)
				tpToSafeZone()
				if not endRoundResetDone then
					endRoundResetDone = true
					task.spawn(function()
						task.wait(0.25)
						if loopMain then
							resetChar("End round: timer <= 2")
							task.wait(0.75)
							ensureFarmCharacter("After end-round reset")
						end
					end)
				end
			end
			if timerTpDone then
				-- Re-arm only after the next round timer is clearly running again.
				-- Team pads may remain/spawn during respawn/intermission; using them here caused reset loops.
				if timer >= 10 then
					timerTpDone=false
					endRoundResetDone=false
				end
			end
		else
			pointsCapped = false timerTpDone = false
		end
	end
end)

-- startFarm
local function startFarm(source)
	if not IS_FARMER then
		notifyAction("Not Listed","Press START to run this unlisted account manually as MAIN",8)
		return false
	end

	if loopMain then
		starting=false
		return true
	end
	if starting then return false end

	farmRunToken+=1
	local myToken=farmRunToken
	farmRequested=true
	lastStartMode=source or lastStartMode or "AUTO"
	starting=true

	local function cancelled()
		return not farmRequested
			or myToken~=farmRunToken
			or not gui
			or not gui.Parent
	end

	makeBase()
	if cancelled() then
		starting=false
		return false
	end

	-- Reset round state first.
	roundPaused=false
	roundPauseReason=nil
	roundResetting=false
	timerTpDone=false
	endRoundResetDone=false
	pointsCapped=false

	-- IMPORTANT:
	-- Enter RUNNING immediately. Character purchase/select/reset is background work.
	-- This means a listed MAIN no longer waits on Backpack/ServerTraits/Respawning
	-- before the farm-position TP loop becomes active.
	loopMain=true
	starting=false
	markFarmStart()

	sendWebhook(FARM_ROLE.." Farm Started")
	notifyAction("Farm Running",FARM_ROLE.." | "..tostring(lastStartMode).." start",4)

	-- Prepare/verify the requested character without blocking farm TP.
	task.spawn(function()
		if cancelled() or not loopMain then return end

		characterFixing=true
		local target=farmCharacterName()

		local ok,err=pcall(function()
			if IS_MAIN then
				buyBakugou()
			end
			if cancelled() or not loopMain then return end

			fireSelectRemote(target)
			task.wait(0.12)
			if cancelled() or not loopMain then return end

			resetChar("Background startup character sync")
		end)

		characterFixing=false

		if not ok then
			warn("[WWHub] Background character prep error: "..tostring(err))
		end

		if not cancelled() and loopMain then
			task.spawn(function()
				ensureFarmCharacter("Background startup verify")
			end)
		end
	end)

	return true
end

-- Team selection — MAIN and AFK use opposing teams
local allTeamPads = {"Red Team", "Blue Team", "Green Team", "Yellow Team"}

local function hasAnyTeamPad()
	for _, name in ipairs(allTeamPads) do
		if workspace:FindFirstChild(name) then return true end
	end
	return false
end

local function getMyTeamPad()
	local availablePads={}
	local padCount=0

	for _,name in ipairs(allTeamPads) do
		if workspace:FindFirstChild(name) then
			padCount+=1
			availablePads[padCount]=name
		end
	end

	if padCount==0 then return nil end

	local padIndex=1
	if IS_MAIN then
		local mainIndex=getRoleIndex(_G.main)
		padIndex=((mainIndex-1)%padCount)+1
	elseif IS_AFK then
		local afkIndex=getRoleIndex(_G.afk)
		if padCount>=2 then
			-- Put each ALT on the next color from its paired MAIN.
			padIndex=(afkIndex%padCount)+1
		else
			padIndex=1
		end
	end

	local selectedName=availablePads[padIndex]
	if not selectedName then return nil end
	return workspace:FindFirstChild(selectedName)
end

local function selectTeam()
	if selectingTeam then return end
	selectingTeam = true

	while gui and gui.Parent and loopMain and not roundPaused and hasAnyTeamPad() do
		local currentPad = getMyTeamPad()
		local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")

		if currentPad and hrp then
			local pp = currentPad:IsA("BasePart") and currentPad or currentPad:FindFirstChildWhichIsA("BasePart")
			if pp then
				pcall(function()
					hrp.AssemblyLinearVelocity = Vector3.new(0,0,0)
					hrp.AssemblyAngularVelocity = Vector3.new(0,0,0)
					hrp.CFrame = pp.CFrame + Vector3.new(0,3,0)
				end)
			end
		end

		task.wait(0.1)
	end

	-- Do not release farm TP until team pads are really gone.
	if gui and gui.Parent and loopMain and not roundPaused then
		task.wait(0.35)
	end
	selectingTeam = false
end

task.spawn(function()
	-- เรียง FFA ก่อน team เพื่อลด chance เจอ team mode
	local modes = {
		"Free For All",  -- 1st priority
		"Kills FFA",     -- 2nd
		"Kills Team",    -- 3rd
		"3 Teams",       -- 4th
		"Team Battle",   -- 5th
	}
	while gui.Parent do
		if loopMain and not roundPaused then
			for _, m in ipairs(modes) do fireInput("mode", m) task.wait(0.3) end
			task.wait(2)
		else task.wait(3) end
	end
end)

-- NoClip
local noClipOn = false
task.spawn(function()
	while gui and gui.Parent do
		task.wait(0.1)
		if loopMain and not roundPaused and not timerTpDone then
			if not noClipOn then noClipOn = true end
			local c = getChar()
			if c then
				for _, p in ipairs(c:GetDescendants()) do
					if p:IsA("BasePart") then p.CanCollide = false end
				end
				local hrp = getHRP()
				if hrp and hrp.Position.Y < -10 then
					pcall(function()
						hrp.CFrame = CFrame.new(hrp.Position.X, 5, hrp.Position.Z)
						hrp.AssemblyLinearVelocity = Vector3.new(0,0,0)
					end)
				end
			end
		else
			if noClipOn then
				noClipOn = false local c = getChar()
				if c then for _, p in ipairs(c:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end end
			end
		end
	end
end)


-- ===== GUI =====
gui=Instance.new("ScreenGui")
gui.Name="WWHub_GUI_v8_7_0"
gui.ResetOnSpawn=false
gui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
gui.DisplayOrder=1000000
gui.Parent=game.CoreGui

local toggleBtn=Instance.new("TextButton")
toggleBtn.Size=UDim2.new(0,42,0,42)
toggleBtn.Position=UDim2.new(0,10,0.5,-21)
toggleBtn.BackgroundColor3=Color3.fromRGB(18,18,28)
toggleBtn.Text="⚡"
toggleBtn.TextSize=20
toggleBtn.TextColor3=Color3.fromRGB(150,70,255)
toggleBtn.Font=Enum.Font.GothamBold
toggleBtn.ZIndex=10
toggleBtn.Parent=gui
Instance.new("UICorner",toggleBtn).CornerRadius=UDim.new(0,10)
local tst=Instance.new("UIStroke",toggleBtn)
tst.Color=Color3.fromRGB(110,40,200)
tst.Thickness=2

local panel=Instance.new("Frame")
panel.Size=UDim2.new(0,310,0,492)
panel.Position=UDim2.new(0.5,-155,0.5,-246)
panel.BackgroundColor3=Color3.fromRGB(14,14,22)
panel.BorderSizePixel=0
panel.Active=true
panel.Parent=gui
Instance.new("UICorner",panel).CornerRadius=UDim.new(0,14)
local pst=Instance.new("UIStroke",panel)
pst.Color=Color3.fromRGB(100,35,190)
pst.Thickness=2

local dragging,dragStart,dragPos
panel.InputBegan:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
		dragging=true dragStart=i.Position dragPos=panel.Position
	end
end)
panel.InputEnded:Connect(function(i)
	if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=false end
end)
if UIS then
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
			local d=i.Position-dragStart
			panel.Position=UDim2.new(dragPos.X.Scale,dragPos.X.Offset+d.X,dragPos.Y.Scale,dragPos.Y.Offset+d.Y)
		end
	end)
end

local header=Instance.new("Frame")
header.Size=UDim2.new(1,0,0,48)
header.BackgroundColor3=Color3.fromRGB(20,20,32)
header.BorderSizePixel=0
header.Parent=panel
Instance.new("UICorner",header).CornerRadius=UDim.new(0,14)

local titleLbl=Instance.new("TextLabel")
titleLbl.Size=UDim2.new(1,-50,1,0)
titleLbl.Position=UDim2.new(0,12,0,0)
titleLbl.BackgroundTransparency=1
titleLbl.Text="⚡ WW Hub v8.7.0"
titleLbl.TextColor3=Color3.fromRGB(155,80,255)
titleLbl.TextSize=18
titleLbl.Font=Enum.Font.GothamBold
titleLbl.TextXAlignment=Enum.TextXAlignment.Left
titleLbl.Parent=header

local closeBtn=Instance.new("TextButton")
closeBtn.Size=UDim2.new(0,32,0,32)
closeBtn.Position=UDim2.new(1,-40,0,8)
closeBtn.BackgroundColor3=Color3.fromRGB(190,35,55)
closeBtn.Text="✕"
closeBtn.TextColor3=Color3.fromRGB(255,255,255)
closeBtn.TextSize=15
closeBtn.Font=Enum.Font.GothamBold
closeBtn.Parent=header
Instance.new("UICorner",closeBtn).CornerRadius=UDim.new(0,8)

local function makeStatCard(x,title)
	local card=Instance.new("Frame")
	card.Size=UDim2.new(0,90,0,62)
	card.Position=UDim2.new(0,x,0,58)
	card.BackgroundColor3=Color3.fromRGB(22,24,38)
	card.BorderSizePixel=0
	card.Parent=panel
	Instance.new("UICorner",card).CornerRadius=UDim.new(0,10)
	local stroke=Instance.new("UIStroke",card)
	stroke.Color=Color3.fromRGB(48,52,78)
	stroke.Thickness=1

	local name=Instance.new("TextLabel")
	name.Size=UDim2.new(1,-8,0,18)
	name.Position=UDim2.new(0,4,0,6)
	name.BackgroundTransparency=1
	name.Text=title
	name.TextColor3=Color3.fromRGB(145,155,185)
	name.TextSize=10
	name.Font=Enum.Font.GothamBold
	name.Parent=card

	local value=Instance.new("TextLabel")
	value.Size=UDim2.new(1,-8,0,28)
	value.Position=UDim2.new(0,4,0,26)
	value.BackgroundTransparency=1
	value.Text="-"
	value.TextColor3=Color3.fromRGB(240,244,255)
	value.TextScaled=true
	value.Font=Enum.Font.GothamBold
	value.Parent=card
	return value
end

local lvValue=makeStatCard(10,"LEVEL")
local goldValue=makeStatCard(110,"GOLD")
local prestigeValue=makeStatCard(210,"PRESTIGE")

local progressLbl=Instance.new("TextLabel")
progressLbl.Size=UDim2.new(1,-20,0,34)
progressLbl.Position=UDim2.new(0,10,0,126)
progressLbl.BackgroundColor3=Color3.fromRGB(22,24,38)
progressLbl.Text="PROFIT +0 GOLD  |  FARM 00:00"
progressLbl.TextColor3=Color3.fromRGB(255,210,80)
progressLbl.TextSize=12
progressLbl.Font=Enum.Font.GothamBold
progressLbl.TextXAlignment=Enum.TextXAlignment.Center
progressLbl.Parent=panel
Instance.new("UICorner",progressLbl).CornerRadius=UDim.new(0,9)
local progressStroke=Instance.new("UIStroke",progressLbl)
progressStroke.Color=Color3.fromRGB(48,52,78)
progressStroke.Thickness=1

local statusLbl=Instance.new("TextLabel")
statusLbl.Size=UDim2.new(1,-20,0,42)
statusLbl.Position=UDim2.new(0,10,0,168)
statusLbl.BackgroundColor3=Color3.fromRGB(22,24,38)
statusLbl.BackgroundTransparency=0
statusLbl.Text="FARM: CHECKING CONFIG..."
statusLbl.TextColor3=Color3.fromRGB(190,200,225)
statusLbl.TextSize=12
statusLbl.Font=Enum.Font.GothamBold
statusLbl.TextWrapped=true
statusLbl.TextXAlignment=Enum.TextXAlignment.Center
statusLbl.TextYAlignment=Enum.TextYAlignment.Center
statusLbl.Parent=panel
Instance.new("UICorner",statusLbl).CornerRadius=UDim.new(0,9)
local statusStroke=Instance.new("UIStroke",statusLbl)
statusStroke.Color=Color3.fromRGB(48,52,78)
statusStroke.Thickness=1

actionNoticeLabel=Instance.new("TextLabel")
actionNoticeLabel.Size=UDim2.new(1,-20,0,28)
actionNoticeLabel.Position=UDim2.new(0,10,0,214)
actionNoticeLabel.BackgroundColor3=Color3.fromRGB(75,28,35)
actionNoticeLabel.BackgroundTransparency=0.05
actionNoticeLabel.TextColor3=Color3.fromRGB(255,225,225)
actionNoticeLabel.TextSize=10
actionNoticeLabel.Font=Enum.Font.GothamBold
actionNoticeLabel.TextWrapped=true
actionNoticeLabel.Visible=false
actionNoticeLabel.Parent=panel
Instance.new("UICorner",actionNoticeLabel).CornerRadius=UDim.new(0,8)

local function mkBtn(txt,col,y,h)
	local b=Instance.new("TextButton")
	b.Size=UDim2.new(1,-20,0,h or 46)
	b.Position=UDim2.new(0,10,0,y)
	b.BackgroundColor3=col
	b.Text=txt
	b.TextColor3=Color3.fromRGB(255,255,255)
	b.TextSize=17
	b.Font=Enum.Font.GothamBold
	b.Parent=panel
	Instance.new("UICorner",b).CornerRadius=UDim.new(0,10)
	return b
end

local startBtn=mkBtn("🎮 START",Color3.fromRGB(50,185,90),250,48)
local stopBtn=mkBtn("⏹ STOP",Color3.fromRGB(190,50,50),306,48)

local capLbl=Instance.new("TextLabel")
capLbl.Size=UDim2.new(1,-95,0,26)
capLbl.Position=UDim2.new(0,10,0,362)
capLbl.BackgroundTransparency=1
capLbl.Text="🎯 Cap: "..getEffectiveCap()
capLbl.TextColor3=Color3.fromRGB(255,200,60)
capLbl.TextSize=12
capLbl.Font=Enum.Font.GothamBold
capLbl.TextXAlignment=Enum.TextXAlignment.Left
capLbl.Parent=panel

local function mkSBtn(txt,xOff)
	local b=Instance.new("TextButton")
	b.Size=UDim2.new(0,36,0,28)
	b.Position=UDim2.new(1,xOff,0,361)
	b.BackgroundColor3=Color3.fromRGB(40,40,55)
	b.Text=txt
	b.TextColor3=Color3.fromRGB(255,255,255)
	b.TextSize=16
	b.Font=Enum.Font.GothamBold
	b.Parent=panel
	Instance.new("UICorner",b).CornerRadius=UDim.new(0,7)
	return b
end
local capMinus=mkSBtn("−",-90)
local capPlus=mkSBtn("+",-48)

local renderEnabled=true
local renderBtn=mkBtn("👁 Render: ON",Color3.fromRGB(0,120,210),396,38)
renderBtn.TextSize=14

-- ===== GUI cleanup control =====
local destroyBtn=Instance.new("TextButton")
destroyBtn.Size=UDim2.new(1,-20,0,30)
destroyBtn.Position=UDim2.new(0,10,0,444)
destroyBtn.BackgroundColor3=Color3.fromRGB(35,35,50)
destroyBtn.Text="DESTROY GUI"
destroyBtn.TextColor3=Color3.fromRGB(200,200,220)
destroyBtn.TextSize=11
destroyBtn.Font=Enum.Font.GothamBold
destroyBtn.Parent=panel
Instance.new("UICorner",destroyBtn).CornerRadius=UDim.new(0,8)

local function setStatus(txt,col)
	statusLbl.Text=tostring(txt or "")
	statusLbl.TextColor3=col or Color3.fromRGB(190,200,225)
	statusLbl.Font=Enum.Font.GothamBold
end

toggleBtn.MouseButton1Click:Connect(function() panel.Visible = not panel.Visible end)
startBtn.MouseButton1Click:Connect(function()
	if not IS_FARMER then
		manualOverride=true
		IS_MAIN=true
		IS_AFK=false
		IS_FARMER=true
		FARM_ROLE="MAIN"
		farmIndex=1
		pairX=0
		farmPos=FARM_BASE
		altCFrame=CFrame.new(farmPos)
		notifyAction("Manual Farm","Unlisted account enabled manually as MAIN",5)
	end
	lastStartMode="MANUAL"
	task.spawn(function()
		local ok=startFarm("MANUAL")
		if ok and loopMain then
			startBtn.BackgroundColor3=Color3.fromRGB(40,190,100)
			startBtn.Text="⏸ RUNNING"
		end
	end)
	startBtn.BackgroundColor3=Color3.fromRGB(210,145,35)
	startBtn.Text="⏳ STARTING"
end)
stopBtn.MouseButton1Click:Connect(function()
	-- Cancel both an active farm and a STARTING sequence.
	farmRequested=false
	farmRunToken+=1
	loopMain=false
	starting=false
	selectingTeam=false
	roundPaused=false
	roundPauseReason=nil
	roundResetting=false
	pointsCapped=false
	timerTpDone=false
	endRoundResetDone=false

	startBtn.BackgroundColor3=Color3.fromRGB(50,185,90)
	startBtn.Text="🎮 START"
	setStatus("FARM: STOPPED  |  ROLE: "..FARM_ROLE,Color3.fromRGB(255,125,125))
	notifyAction("Farm Stopped","All farm loops/startup cancelled",4)
end)
capMinus.MouseButton1Click:Connect(function()
	adjustEffectiveCap(-10000)
	capLbl.Text = hasOutsider() and ("🎯 Cap: "..getEffectiveCap()) or "🎯 Cap: OFF (Private)"
end)
capPlus.MouseButton1Click:Connect(function()
	adjustEffectiveCap(10000)
	capLbl.Text = hasOutsider() and ("🎯 Cap: "..getEffectiveCap()) or "🎯 Cap: OFF (Private)"
end)
renderBtn.MouseButton1Click:Connect(function()
	renderEnabled = not renderEnabled
	game:GetService("RunService"):Set3dRenderingEnabled(renderEnabled)
	renderBtn.Text = renderEnabled and "👁 Render: ON" or "👁 Render: OFF"
	renderBtn.BackgroundColor3 = renderEnabled and Color3.fromRGB(0,120,210) or Color3.fromRGB(170,35,35)
end)
closeBtn.MouseButton1Click:Connect(function() panel.Visible = false end)
destroyBtn.MouseButton1Click:Connect(function() farmRequested=false farmRunToken+=1 loopMain=false starting=false gui:Destroy() end)

-- Auto-start: listed accounts should enter RUNNING immediately.
task.spawn(function()
	task.wait(0.35)

	if not IS_FARMER then
		warn("[WWHub] Auto-start skipped: account is not listed")
		return
	end

	lastStartMode="AUTO"

	for attempt=1,3 do
		if not gui or not gui.Parent then return end
		if loopMain then break end

		-- If STOP was pressed after a previous attempt, do not re-arm.
		if attempt>1 and not farmRequested then break end

		autoStartAttempts=attempt
		warn("[WWHub] Auto-start attempt "..attempt.." | "..FARM_ROLE.." | @"..myName)
		startBtn.BackgroundColor3=Color3.fromRGB(210,145,35)
		startBtn.Text="⏳ AUTO START "..attempt

		local ok,err=pcall(function()
			startFarm("AUTO")
		end)
		if not ok then
			warn("[WWHub] Auto-start error: "..tostring(err))
		end

		task.wait(0.65)
		if loopMain then break end
	end

	if loopMain then
		startBtn.BackgroundColor3=Color3.fromRGB(40,190,100)
		startBtn.Text="⏸ RUNNING"
	else
		startBtn.BackgroundColor3=Color3.fromRGB(190,75,55)
		startBtn.Text="🎮 START"
		notifyAction(
			"Auto Start Failed",
			"@"..myName.." matched "..FARM_ROLE.." but farm flag could not start",
			8
		)
	end
end)

-- Status sync (now shows effective/dynamic cap when capped + gold progress)
task.spawn(function()
	while gui and gui.Parent do
		local timer=getTimerValue()
		local level=getLevel()
		local gold=getGold()
		local prestige=getPrestigeCount()
		local info=" | t="..tostring(timer)
		if IS_MAIN and smartMainCount>1 then
			info=info.." | "..tostring(smartServerText)
		end
		local gained,elapsed=getFarmProgress()
		local sign=gained>=0 and "+" or ""
		local hh=math.floor(elapsed/3600)
		local mm=math.floor((elapsed%3600)/60)
		progressLbl.Text=string.format("PROFIT %s%d GOLD  |  FARM %02d:%02d",sign,gained,hh,mm)
		lvValue.Text=tostring(level or "?")
		goldValue.Text="$"..tostring(gold)
		prestigeValue.Text=tostring(prestige)
		capLbl.Text=hasOutsider() and ("🎯 Cap: "..getEffectiveCap()) or "🎯 Cap: OFF (Private)"
		if starting then
			setStatus("FARM: STARTING  |  ROLE: "..FARM_ROLE.."  |  "..tostring(lastStartMode),Color3.fromRGB(255,200,50))
		elseif roundPaused then
			setStatus("FARM: PAUSED  |  "..tostring(roundPauseReason or "Paused")..info,Color3.fromRGB(255,100,100))
		elseif timerTpDone then
			setStatus("FARM: SAFE ZONE  |  ROLE: "..FARM_ROLE..info,Color3.fromRGB(255,165,0))
		elseif pointsCapped and hasOutsider() then
			setStatus("FARM: POINT CAP  |  "..getPoints().."/"..getEffectiveCap()..info,Color3.fromRGB(255,215,0))
		elseif loopMain then
			setStatus("FARM: RUNNING  |  ROLE: "..FARM_ROLE.."  |  "..tostring(lastStartMode)..info,Color3.fromRGB(100,220,155))
			startBtn.BackgroundColor3=Color3.fromRGB(40,190,100)
			startBtn.Text="⏸ RUNNING"
		elseif not IS_FARMER then
			setStatus(
				"FARM: IDLE | UNLISTED @"..tostring(myName).." | MAIN LIST: "..tostring(#_G.main),
				Color3.fromRGB(255,120,120)
			)
		else
			setStatus("FARM: IDLE  |  ROLE: "..FARM_ROLE.."  |  AUTO READY",Color3.fromRGB(150,175,220))
		end
		task.wait(0.5)
	end
end)

-- ===== Farm Loops =====
task.spawn(function()
	makeBase()
	while gui.Parent do
		task.wait(0.25)
		local c = LP.Character
		if c and c.Parent and c ~= handledChar then afterCharLoaded(c) end
	end
end)

task.spawn(function()
	while gui.Parent do
		task.wait(0.03)
		if not loopMain then continue end

		if hasAnyTeamPad() and not roundPaused and not timerTpDone then
			local wrong,observed = isKnownWrongFarmCharacter()
			if wrong then
				if not characterFixing then
					task.spawn(function()
						ensureFarmCharacter("Round start | observed "..tostring(observed))
					end)
				end
				continue
			end
			if not selectingTeam and not characterFixing then task.spawn(selectTeam) end
			continue
		end

		if not selectingTeam and not starting and not roundPaused and not timerTpDone then
			local mcf = getMainCF()
			if not isNearCF(mcf, tpDist) then tpToCF(mcf) end
		end
	end
end)

task.spawn(function()
	while gui.Parent do
		task.wait(0.2)
		if IS_MAIN and loopMain and not starting and not selectingTeam and not roundPaused and not pointsCapped and not timerTpDone and not characterFixing then
			local wrong = isKnownWrongFarmCharacter()
			if not wrong then fireBakugouQ() end
		end
	end
end)

task.spawn(function()
	local candidate = nil
	local candidateHits = 0
	while gui.Parent do
		if loopMain and not starting and not roundPaused and not selectingTeam and not hasAnyTeamPad() then
			local bm = getBlockedMode()
			if bm then
				if candidate == bm then
					candidateHits += 1
				else
					candidate = bm
					candidateHits = 1
				end
				if candidateHits >= 3 then
					notifyAction("Mode Check","Confirmed "..bm.." 3x -> safety reset enabled",5)
					pauseFarm(bm)
					candidate = nil
					candidateHits = 0
				end
			else
				candidate = nil
				candidateHits = 0
			end
			task.wait(0.5)
		else
			candidate = nil
			candidateHits = 0
			task.wait(1)
		end
	end
end)

task.spawn(function()
	while gui.Parent do task.wait(30) if loopMain and not roundPaused then sendWebhook(FARM_ROLE.." Farm Report") end end
end)

task.spawn(function()
	while gui and gui.Parent do task.wait(60) collectgarbage("collect") end
end)
