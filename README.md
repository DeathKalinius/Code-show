# Snaproll Football

**Roblox football franchise simulator | Luau server systems**

This repository showcases the server-side game logic for Snaproll Football. The server owns player progression and game rewards, keeping important state changes out of client control.

## What this script handles

- Franchise and quick-draft flows, including offer rerolls and roster validation
- Season and playoff progression through the `Sim` module
- Coins, card packs, upgrades, achievements, and daily rewards
- Player profiles, autosaving, and an ordered leaderboard
- Franchise-name filtering and server-side request checks
- Roblox marketplace receipts, game passes, and paid-item policy checks

## Roblox Studio setup

Place this Script at `ServerScriptService > Snaproll > Server`. It expects a sibling `Sim` ModuleScript and a `Snaproll` folder in `ReplicatedStorage` containing the `Shared` ModuleScript, `Api` RemoteFunction, and `Push` RemoteEvent.

This workspace currently contains the server script only; the supporting modules, remotes, client interface, and game assets are not included. DataStore persistence also requires Studio API access to be enabled when testing in Studio.

## Main server script

```lua
-- Snaproll Football: main server script. Lives in ServerScriptService > Snaproll > Server.
-- Everything that changes coins, cards or results runs here, so players can't fake them.
-- DeathKalinius, 2026

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local DataStoreService = game:GetService("DataStoreService")
local TextService = game:GetService("TextService")
local MarketplaceService = game:GetService("MarketplaceService")
local PolicyService = game:GetService("PolicyService")
local HttpService = game:GetService("HttpService")
local RunService = game:GetService("RunService")

local Snap = ReplicatedStorage:WaitForChild("Snaproll")
local Shared = require(Snap:WaitForChild("Shared"))
local Sim = require(script.Parent:WaitForChild("Sim"))
local Api = Snap:WaitForChild("Api") :: RemoteFunction
local Push = Snap:WaitForChild("Push") :: RemoteEvent

local rng = Random.new()
local function R() return rng:NextNumber() end
Sim.setRandom(R)

local SAVE_KEY_PREFIX = "u_"
local saveStore, scoreStore
local storesOk = pcall(function()
	saveStore = DataStoreService:GetDataStore("SnaprollSave_v1")
	scoreStore = DataStoreService:GetOrderedDataStore("SnaprollScore_v1")
end)
-- in Studio without API access, GetDataStore works but every read fails; check once so the game says so
if storesOk and RunService:IsStudio() then
	storesOk = pcall(function() saveStore:GetAsync("__probe") end)
end

---------------------------------------------------------------------------
-- name filter: Roblox's own filter plus a blocklist for slurs it may miss
---------------------------------------------------------------------------
local BLK = { s = {}, t = {} }
do
	local b64 = "eyJzIjogWyJhc2hvbGUiLCAiYmFzdGFyZCIsICJiZWFuZXIiLCAiYml0Y2giLCAiYmxvd2pvYiIsICJjaGluayIsICJjdW50IiwgImRpbGRvIiwgImZhZ290IiwgImZjayIsICJmdWNrIiwgImZ1ayIsICJnYXNjaGFtYmVyIiwgImhhbmRqb2IiLCAiaGl0bGVyIiwgImhvbG9jYXVzdCIsICJpbmNlc3QiLCAiamlnYWJvIiwgImppeiIsICJraWtlIiwgIm1vbGVzdCIsICJtb3RoZXJmIiwgIm5hemkiLCAibmVncm8iLCAibmlnYSIsICJuaWdlciIsICJuaWdyIiwgIm51dHNhY2siLCAib3JnYXNtIiwgInBlbmlzIiwgInBvcmNobW9ua2V5IiwgInBvcm4iLCAicHVzeSIsICJyYWdoZWFkIiwgInJhcGlzdCIsICJyZWRza2luIiwgInJldGFyZCIsICJzYW5kbmlnIiwgInNoaXQiLCAic2x1dCIsICJzcGFzdGljIiwgInRvd2VsaGVhZCIsICJ0cmFueSIsICJ0d2F0IiwgInZhZ2luYSIsICJ3ZXRiYWNrIiwgIndoaXRlcG93ZXIiLCAid2hpdGVwcmlkZSIsICJ3aG9yZSJdLCAidCI6IFsiYW5hbCIsICJhc3MiLCAiYm9vYiIsICJib29icyIsICJjaGlua3kiLCAiY29jayIsICJjb2NrcyIsICJjb29uIiwgImNvb25zIiwgImN1bSIsICJkYWdvIiwgImRpY2siLCAiZGlja3MiLCAiZHlrZSIsICJkeWtlcyIsICJmYWciLCAiZmFncyIsICJnb29rIiwgImdvb2tzIiwgImd5cCIsICJoZWlsIiwgImhvZSIsICJob2VzIiwgImhvbW8iLCAiaG9tb3MiLCAiamFwIiwgImphcHMiLCAia2trIiwgImt5cyIsICJuaWciLCAibmlncyIsICJub25jZSIsICJwYWtpIiwgInBha2lzIiwgInBlZG8iLCAicGVkb3MiLCAicHJpY2siLCAicmFwZSIsICJyYXBlZCIsICJzZW1lbiIsICJzZXgiLCAic2V4eSIsICJza2FuayIsICJzbGF2ZSIsICJzbGF2ZXMiLCAic3BpYyIsICJzcGljcyIsICJzcXVhdyIsICJ0YXJkIiwgInRhcmRzIiwgInRpdHMiLCAidGl0dHkiLCAid2FuayIsICJ3b3AiLCAid29wcyJdfQ=="
	local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
	local map = {}
	for i = 1, #chars do map[string.sub(chars, i, i)] = i - 1 end
	local bytes, bits, nbits = {}, 0, 0
	for i = 1, #b64 do
		local v = map[string.sub(b64, i, i)]
		if v then
			bits = bits * 64 + v
			nbits += 6
			if nbits >= 8 then
				nbits -= 8
				local byte = math.floor(bits / 2 ^ nbits) % 256
				table.insert(bytes, string.char(byte))
				bits = bits % 2 ^ nbits
			end
		end
	end
	local ok, decoded = pcall(function() return HttpService:JSONDecode(table.concat(bytes)) end)
	if ok and decoded then BLK = decoded end
end
local LEET = { ["0"] = "o", ["1"] = "i", ["3"] = "e", ["4"] = "a", ["5"] = "s", ["7"] = "t", ["8"] = "b", ["9"] = "g", ["@"] = "a", ["$"] = "s", ["!"] = "i", ["|"] = "i", ["+"] = "t" }
local function collapse(s: string): string
	local out, last = {}, ""
	for c in string.gmatch(s, ".") do
		if c ~= last then table.insert(out, c) end
		last = c
	end
	return table.concat(out)
end
local function blocklistOk(name: string): boolean
	local raw = string.lower(name)
	if string.find(raw, "1488") or string.find(raw, "14%s*88") then return false end
	local sq = raw:gsub(".", function(c) return LEET[c] or c end)
	local all = collapse((sq:gsub("[^a-z]", "")))
	for _, w in BLK.s do
		if string.find(all, w, 1, true) then return false end
	end
	if string.find(sq, "k%s*k%s*k") then return false end
	local run = ""
	for tok in string.gmatch(sq, "[a-z]+") do
		local c2 = tok:gsub("(.)%1%1+", "%1%1")
		if table.find(BLK.t, tok) or table.find(BLK.t, c2) or table.find(BLK.t, collapse(tok)) then return false end
		if #tok == 1 then
			run ..= tok
			if table.find(BLK.t, run) then return false end
		else
			run = ""
		end
	end
	return true
end

-- returns the cleaned name, or nil if it isn't allowed
local function filterName(player: Player, name: string): string?
	name = string.gsub(string.gsub(name or "", "^%s+", ""), "%s+$", "")
	name = string.gsub(name, "%s+", " ")
	name = string.gsub(name, "[<>&]", "")
	if #name < 2 or #name > 24 then return nil end
	if not blocklistOk(name) then return nil end
	local ok, filtered = pcall(function()
		local result = TextService:FilterStringAsync(name, player.UserId, Enum.TextFilterContext.PublicChat)
		return result:GetNonChatStringForBroadcastAsync()
	end)
	if not ok then
		-- the filter service is down: only allow names the blocklist already passed when testing in Studio
		if RunService:IsStudio() then return name end
		return nil
	end
	if filtered ~= name then return nil end
	return name
end

---------------------------------------------------------------------------
-- profiles
---------------------------------------------------------------------------
local function blank()
	return {
		v = 1, coins = 400, owned = {}, welcome = false,
		stats = { seasons = 0, titles = 0, w = 0, l = 0, bestOvr = 0, quick = 0, pulls = 0, legendPulls = 0, packs = 0, lastChamp = false },
		ach = {}, daily = { day = "", streak = 0 }, up = { luck = 0, rerolls = 0, insure = 0, scout = 0 },
		season = nil, history = {}, settings = { sound = true, speed = 2 }, receipts = {},
	}
end

local profiles: { [Player]: any } = {}
local saving: { [Player]: boolean } = {}
local drafts: { [Player]: any } = {}
local meta: { [Player]: any } = {}

local function today(offsetDays: number?): string
	return os.date("!%Y-%m-%d", os.time() - (offsetDays or 0) * 86400) :: string
end

local function merge(into, from)
	for k, v in from do
		if type(v) == "table" and type(into[k]) == "table" and #into[k] == 0 and next(into[k]) ~= nil then
			merge(into[k], v)
		else
			into[k] = v
		end
	end
	return into
end

local function loadProfile(player: Player)
	local data
	if storesOk then
		for attempt = 1, 3 do
			local ok, res = pcall(function() return saveStore:GetAsync(SAVE_KEY_PREFIX .. player.UserId) end)
			if ok then data = res; break end
			task.wait(attempt)
		end
	end
	local prof = blank()
	if type(data) == "table" then
		prof = merge(prof, data)
		prof.stats = merge(blank().stats, data.stats or {})
		prof.up = merge(blank().up, data.up or {})
		prof.settings = merge(blank().settings, data.settings or {})
	end
	return prof
end

local function saveProfile(player: Player)
	local prof = profiles[player]
	if not prof or not storesOk or saving[player] then return false end
	saving[player] = true
	local ok = pcall(function()
		saveStore:SetAsync(SAVE_KEY_PREFIX .. player.UserId, prof, { player.UserId })
	end)
	saving[player] = false
	return ok
end

---------------------------------------------------------------------------
-- leaderboard
---------------------------------------------------------------------------
local function scoreOf(prof): number
	local tro = 0
	for _ in prof.ach do tro += 1 end
	return prof.stats.titles * 500 + prof.stats.w * 5 + tro * 40 + math.floor((prof.stats.bestOvr or 0) + .5) * 2
end

local function refreshLeaderstats(player: Player)
	local prof = profiles[player]
	local ls = player:FindFirstChild("leaderstats")
	if not prof or not ls then return end
	(ls:FindFirstChild("Titles") :: IntValue).Value = prof.stats.titles;
	(ls:FindFirstChild("Score") :: IntValue).Value = scoreOf(prof)
end

local function pushScore(player: Player)
	local prof = profiles[player]
	if not prof or not storesOk then return end
	refreshLeaderstats(player)
	local m = meta[player]
	local score = scoreOf(prof)
	if m.lastScore == score then return end
	if m.scorePending then return end
	m.scorePending = true
	task.delay(math.max(0, 6 - (os.clock() - (m.lastScorePush or 0))), function()
		local s = profiles[player] and scoreOf(profiles[player]) or score
		local ok = pcall(function() scoreStore:SetAsync(tostring(player.UserId), s) end)
		if ok then m.lastScore = s end
		m.lastScorePush = os.clock()
		m.scorePending = false
	end)
end

local boardCache = { at = -1e9, rows = {} }
local nameCache: { [number]: string } = {}
local function getBoard()
	if os.clock() - boardCache.at < 60 or not storesOk then return boardCache.rows end
	boardCache.at = os.clock()
	local ok, pages = pcall(function() return scoreStore:GetSortedAsync(false, 50) end)
	if not ok then return boardCache.rows end
	local rows = {}
	for i, entry in pages:GetCurrentPage() do
		local uid = tonumber(entry.key)
		local name = uid and nameCache[uid]
		if uid and not name then
			local okName, n = pcall(function() return Players:GetNameFromUserIdAsync(uid) end)
			name = okName and n or ("Player " .. entry.key)
			nameCache[uid] = name
		end
		table.insert(rows, { rank = i, userId = uid, name = name, score = entry.value })
	end
	boardCache.rows = rows
	return rows
end

---------------------------------------------------------------------------
-- rewards, trophies, collection
---------------------------------------------------------------------------
local function award(prof, id: string, out)
	if prof.ach[id] then return end
	local a = Shared.ACH_BY[id]
	if not a then return end
	prof.ach[id] = today()
	prof.coins += a.r
	table.insert(out.awards, id)
end

local function ownedCount(prof): number
	local n = 0
	for _ in prof.owned do n += 1 end
	return n
end

local function checkCollection(prof, out)
	local n = ownedCount(prof)
	if n >= 25 then award(prof, "col25", out) end
	if n >= 100 then award(prof, "col100", out) end
	if n >= 250 then award(prof, "col250", out) end
	if n >= #Shared.PLAYERS then award(prof, "complete", out) end
	local allIcons = true
	for _, p in Shared.PLAYERS do
		if p.icon and not prof.owned[p.id] then allIcons = false; break end
	end
	if allIcons then award(prof, "pantheon", out) end
end

local function gameCoins(player: Player, base: number): number
	return (meta[player] and meta[player].vip) and base * 2 or base
end

local function openPack(prof, pk, out)
	local res = {}
	for i = 1, pk.count do
		local last = i == pk.count
		local min = (last and pk.gtd) and pk.gtd or pk.min
		local p = Shared.rollAny(pk.boost, min, prof.up.luck, R)
		local dup = prof.owned[p.id] ~= nil
		prof.owned[p.id] = (prof.owned[p.id] or 0) + 1
		local c = 0
		if dup then c = Shared.TIERS[p.t].dupe; prof.coins += c end
		prof.stats.pulls += 1
		if p.t >= 5 then prof.stats.legendPulls += 1; award(prof, "legend", out) end
		if p.t == 6 then award(prof, "iconpull", out) end
		table.insert(res, { id = p.id, dup = dup, coins = c })
	end
	prof.stats.packs += 1
	if prof.stats.packs >= 50 then award(prof, "packs50", out) end
	if prof.stats.legendPulls >= 10 then award(prof, "legend10", out) end
	checkCollection(prof, out)
	out.pack = res
end

local function finishSeason(player: Player, prof, sea, out)
	local u = Sim.user(sea)
	local won = sea.champ == "YOU"
	prof.stats.seasons += 1
	if won then
		prof.stats.titles += 1
		award(prof, "champ", out)
		if prof.stats.titles >= 3 then award(prof, "dynasty", out) end
		if prof.stats.titles >= 5 then award(prof, "fivetitles", out) end
		if Sim.seedOf(sea, "YOU") >= 6 then award(prof, "cinderella", out) end
		if prof.stats.lastChamp then award(prof, "b2b", out) end
		if u.w == Shared.WEEKS then award(prof, "perfecto", out) end
	end
	prof.stats.lastChamp = won
	if prof.stats.seasons >= 10 then award(prof, "season10", out) end
	if prof.stats.w >= 100 then award(prof, "wins100", out) end
	if sea.stats.py >= 6500 then award(prof, "qb5000", out) end
	if sea.stats.ry >= 3800 then award(prof, "rush2000", out) end
	table.insert(prof.history, 1, { name = sea.name, rec = u.w .. "-" .. u.l, ovr = math.floor(sea.startOvr + .5), res = Sim.playoffExit(sea), champ = won })
	while #prof.history > 12 do table.remove(prof.history) end
	sea.finished = true
	pushScore(player)
end

-- applies a played game's coins and trophy events to the profile
local function applyGame(player: Player, prof, sea, m, events, out)
	if m then
		local coins
		if m.playoff then
			coins = m.won and (m.playoff == 4 and 400 or 100) or 0
		else
			coins = m.won and 35 or 10
		end
		coins = gameCoins(player, coins)
		m.coins = coins
		prof.coins += coins
		sea.earned += coins
		if m.won then prof.stats.w += 1 else prof.stats.l += 1 end
	end
	for _, e in events do award(prof, e, out) end
	if sea.phase == "done" and not sea.finished then finishSeason(player, prof, sea, out) end
end

local function reviewRoster(prof, picks, quick: boolean, out)
	local r = Shared.rating(picks)
	if r.ovr > (prof.stats.bestOvr or 0) then prof.stats.bestOvr = r.ovr end
	if r.ovr >= 90 then award(prof, "ninety", out) end
	if r.ovr >= 100 then award(prof, "team100", out) end
	if r.chem >= 6 then award(prof, "chem", out) end
	local icons = 0
	for _, id in picks do if Shared.BYID[id].icon then icons += 1 end end
	if icons >= 3 then award(prof, "rushmore", out) end
	local hc = Shared.BYID[picks.HC]
	local fit = Shared.SCHEME_FIT[hc.arch]
	if fit then
		local m = 0
		for k, id in picks do if k ~= "HC" and table.find(fit, Shared.BYID[id].arch) then m += 1 end end
		if m >= 6 then award(prof, "scheme", out) end
	end
	if quick and r.ovr >= 89 then award(prof, "quick_a", out) end
	return r
end

---------------------------------------------------------------------------
-- drafts (kept on the server so rolls can't be rerun from the client)
---------------------------------------------------------------------------
local function dealOffers(prof, D)
	local s = Shared.SLOTS[D.i]
	local ex = {}
	for _, id in D.picks do ex[id] = true end
	local n = D.mode == "quick" and 1 or (3 + prof.up.scout)
	local out = {}
	for _ = 1, n do
		local p = Shared.rollPlayer(s.pos, ex, 1, 0, Shared.DRAFT_W, prof.up.luck, R)
		if p.t == 1 and R() < prof.up.insure * .12 then p = Shared.rollPlayer(s.pos, ex, 1, 1, Shared.DRAFT_W, prof.up.luck, R) end
		ex[p.id] = true
		table.insert(out, p.id)
	end
	D.offers = out
end

local function newDraft(prof, mode: string)
	local D = { mode = mode, i = 1, picks = {}, offers = {}, rerolls = (mode == "quick" and 2 or 3) + prof.up.rerolls, tag = mode ~= "quick", done = false }
	dealOffers(prof, D)
	return D
end

local function advanceDraft(player, prof, D, out)
	D.i += 1
	if D.i > #Shared.SLOTS then
		D.done = true
		D.offers = {}
		local r = reviewRoster(prof, D.picks, D.mode == "quick", out)
		D.rating = r.ovr
		if D.mode == "quick" then
			prof.stats.quick += 1
			local c = 10 + math.max(0, math.floor(r.ovr + .5) - 75) * 2
			prof.coins += c
			D.reward = c
			if prof.stats.quick >= 10 then award(prof, "quick10", out) end
		end
	else
		dealOffers(prof, D)
	end
end

---------------------------------------------------------------------------
-- state sent to the client
---------------------------------------------------------------------------
local function stateOf(player: Player)
	local prof = profiles[player]
	local m = meta[player]
	return {
		coins = prof.coins, owned = prof.owned, stats = prof.stats, ach = prof.ach, daily = prof.daily,
		dailyReady = prof.daily.day ~= today(), up = prof.up, welcome = prof.welcome, settings = prof.settings,
		history = prof.history, season = prof.season, draft = drafts[player], score = scoreOf(prof),
		vip = m.vip, storeOn = m.storeOn, saving = storesOk, studio = RunService:IsStudio(),
	}
end

---------------------------------------------------------------------------
-- API
---------------------------------------------------------------------------
local handlers = {}

handlers.init = function(player, prof, arg, out) end

handlers.settings = function(player, prof, arg, out)
	if type(arg) ~= "table" then return end
	if type(arg.sound) == "boolean" then prof.settings.sound = arg.sound end
	if type(arg.speed) == "number" and table.find({ 1, 2, 4, 8 }, arg.speed) then prof.settings.speed = arg.speed end
end

handlers.claimDaily = function(player, prof, arg, out)
	if prof.daily.day == today() then return "Already claimed today." end
	prof.daily.streak = (prof.daily.day == today(1)) and math.min(7, prof.daily.streak + 1) or 1
	prof.daily.day = today()
	local c = 100 + 25 * (prof.daily.streak - 1)
	prof.coins += c
	out.daily = { coins = c, streak = prof.daily.streak }
	if prof.daily.streak >= 7 then award(prof, "streak7", out) end
end

handlers.openWelcome = function(player, prof, arg, out)
	if prof.welcome then return "Welcome Pack already opened." end
	prof.welcome = true
	openPack(prof, Shared.WELCOME, out)
end

handlers.buyPack = function(player, prof, arg, out)
	local pk
	for _, p in Shared.PACKS do if p.k == arg then pk = p end end
	if not pk then return "Unknown pack." end
	if prof.coins < pk.price then return "Not enough coins." end
	prof.coins -= pk.price
	openPack(prof, pk, out)
end

handlers.upgrade = function(player, prof, arg, out)
	local u
	for _, x in Shared.UPG do if x.k == arg then u = x end end
	if not u then return "Unknown upgrade." end
	local l = prof.up[u.k]
	if l >= u.max then return "Already maxed." end
	local c = Shared.upgradeCost(u.k, l)
	if prof.coins < c then return "Not enough coins." end
	prof.coins -= c
	prof.up[u.k] = l + 1
	if prof.up.luck >= 10 then award(prof, "maxluck", out) end
	local all = true
	for _, x in Shared.UPG do if prof.up[x.k] < x.max then all = false end end
	if all then award(prof, "allup", out) end
end

handlers.draftStart = function(player, prof, arg, out)
	if arg ~= "season" and arg ~= "quick" then return "Unknown draft." end
	drafts[player] = newDraft(prof, arg)
end

handlers.draftPick = function(player, prof, arg, out)
	local D = drafts[player]
	if not D or D.done then return "No draft in progress." end
	if not table.find(D.offers, arg) then return "That player isn't on the board." end
	D.picks[Shared.SLOTS[D.i].k] = arg
	advanceDraft(player, prof, D, out)
end

handlers.draftReroll = function(player, prof, arg, out)
	local D = drafts[player]
	if not D or D.done then return "No draft in progress." end
	if D.rerolls <= 0 then return "No rerolls left." end
	D.rerolls -= 1
	dealOffers(prof, D)
end

handlers.draftTag = function(player, prof, arg, out)
	local D = drafts[player]
	if not D or D.done or D.mode ~= "season" then return "No season draft in progress." end
	if not D.tag then return "Franchise Tag already used." end
	local p = Shared.BYID[arg]
	local s = Shared.SLOTS[D.i]
	if not p or p.pos ~= s.pos or not prof.owned[p.id] then return "You can't tag that player here." end
	for _, id in D.picks do if id == p.id then return "Already on your roster." end end
	D.tag = false
	D.picks[s.k] = p.id
	advanceDraft(player, prof, D, out)
end

handlers.draftAutoRest = function(player, prof, arg, out)
	local D = drafts[player]
	if not D or D.done or D.mode ~= "quick" then return "No quick draft in progress." end
	while not D.done do
		D.picks[Shared.SLOTS[D.i].k] = D.offers[1]
		advanceDraft(player, prof, D, out)
	end
end

handlers.kickoff = function(player, prof, arg, out)
	local D = drafts[player]
	if not D or not D.done then return "Finish the draft first." end
	if type(arg) ~= "table" or type(arg.name) ~= "string" then return "Pick a franchise name." end
	local name = filterName(player, arg.name)
	if not name then out.nameRejected = true; return "That name isn't allowed. Pick a different franchise name." end
	local pal = tonumber(arg.pal) or 2
	if pal < 1 or pal > #Shared.PALETTES then pal = 2 end
	prof.season = Sim.createSeason(D.picks, name, math.floor(pal))
	drafts[player] = nil
end

local function requireSeason(prof)
	return prof.season
end

handlers.playWeek = function(player, prof, arg, out)
	local sea = requireSeason(prof)
	if not sea or sea.phase ~= "reg" then return "No regular-season game to play." end
	local m, ev = Sim.playWeek(sea)
	applyGame(player, prof, sea, m, ev, out)
	out.game = m
end

handlers.simReg = function(player, prof, arg, out)
	local sea = requireSeason(prof)
	if not sea or sea.phase ~= "reg" then return "The regular season is over." end
	while sea.phase == "reg" do
		local m, ev = Sim.playWeek(sea)
		applyGame(player, prof, sea, m, ev, out)
	end
end

handlers.playRound = function(player, prof, arg, out)
	local sea = requireSeason(prof)
	if not sea or sea.phase ~= "post" then return "No playoff game to play." end
	local m, ev = Sim.playRound(sea)
	applyGame(player, prof, sea, m, ev, out)
	out.game = m
	out.bye = (m == nil) and Sim.userAlive(sea)
end

handlers.simPost = function(player, prof, arg, out)
	local sea = requireSeason(prof)
	if not sea or sea.phase ~= "post" then return "No playoffs to sim." end
	while sea.phase == "post" do
		local m, ev = Sim.playRound(sea)
		applyGame(player, prof, sea, m, ev, out)
	end
end

-- plays forward until the next game the user takes part in (used by Watch to the end)
handlers.nextGame = function(player, prof, arg, out)
	local sea = requireSeason(prof)
	if not sea then return "No season." end
	if sea.phase == "reg" then return handlers.playWeek(player, prof, arg, out) end
	while sea.phase == "post" do
		if not Sim.userAlive(sea) then
			while sea.phase == "post" do
				local m, ev = Sim.playRound(sea)
				applyGame(player, prof, sea, m, ev, out)
			end
			break
		end
		local m, ev = Sim.playRound(sea)
		applyGame(player, prof, sea, m, ev, out)
		if m then out.game = m; return end
	end
end

handlers.endSeason = function(player, prof, arg, out)
	local sea = prof.season
	if not sea then return end
	if sea.phase ~= "done" then
		-- abandoning a season in progress: no playoff rewards
		prof.season = nil
		return
	end
	prof.season = nil
end

handlers.quickToSeason = function(player, prof, arg, out)
	local D = drafts[player]
	if not D or not D.done or D.mode ~= "quick" then return "Finish a quick draft first." end
	D.mode = "season"
end

handlers.board = function(player, prof, arg, out)
	out.board = getBoard()
	out.myScore = scoreOf(prof)
end

local busy: { [Player]: boolean } = {}
Api.OnServerInvoke = function(player: Player, action: string, arg: any)
	local prof = profiles[player]
	if not prof then return { ok = false, err = "Your save is still loading. Try again in a moment." } end
	local h = handlers[action]
	if type(action) ~= "string" or not h then return { ok = false, err = "Unknown action." } end
	if busy[player] then return { ok = false, err = "Slow down a little." } end
	busy[player] = true
	local out = { awards = {} }
	local ok, err = pcall(h, player, prof, arg, out)
	busy[player] = false
	if not ok then
		warn("[Snaproll] " .. action .. " failed: " .. tostring(err))
		return { ok = false, err = "Something went wrong. Try again." , state = stateOf(player) }
	end
	if type(err) == "string" then
		out.ok = false
		out.err = err
	else
		out.ok = true
	end
	if #out.awards > 0 then pushScore(player) end
	out.state = stateOf(player)
	meta[player].dirty = true
	return out
end

---------------------------------------------------------------------------
-- Robux coin bundles
---------------------------------------------------------------------------
MarketplaceService.ProcessReceipt = function(info)
	local player = Players:GetPlayerByUserId(info.PlayerId)
	local prof = player and profiles[player]
	if not prof then return Enum.ProductPurchaseDecision.NotProcessedYet end
	if table.find(prof.receipts, info.PurchaseId) then return Enum.ProductPurchaseDecision.PurchaseGranted end
	local bundle
	for _, b in Shared.COIN_PRODUCTS do
		if b.id ~= 0 and b.id == info.ProductId then bundle = b end
	end
	if not bundle then return Enum.ProductPurchaseDecision.NotProcessedYet end
	prof.coins += bundle.coins
	table.insert(prof.receipts, info.PurchaseId)
	while #prof.receipts > 100 do table.remove(prof.receipts, 1) end
	if storesOk and not saveProfile(player) then
		prof.coins -= bundle.coins
		table.remove(prof.receipts)
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end
	Push:FireClient(player, "purchased", { coins = bundle.coins, state = stateOf(player) })
	return Enum.ProductPurchaseDecision.PurchaseGranted
end

---------------------------------------------------------------------------
-- player lifecycle
---------------------------------------------------------------------------
local function onJoin(player: Player)
	local ls = Instance.new("Folder")
	ls.Name = "leaderstats"
	local titles = Instance.new("IntValue"); titles.Name = "Titles"; titles.Parent = ls
	local score = Instance.new("IntValue"); score.Name = "Score"; score.Parent = ls
	ls.Parent = player

	meta[player] = { vip = false, storeOn = true, dirty = false }
	local prof = loadProfile(player)
	if not player.Parent then return end
	profiles[player] = prof

	task.spawn(function()
		if Shared.VIP_PASS_ID ~= 0 then
			local ok, owns = pcall(function() return MarketplaceService:UserOwnsGamePassAsync(player.UserId, Shared.VIP_PASS_ID) end)
			meta[player].vip = ok and owns or false
		end
		-- paid random items: where they're restricted, coins can't be sold for Robux
		local ok, policy = pcall(function() return PolicyService:GetPolicyInfoForPlayerAsync(player) end)
		if ok and policy and policy.ArePaidRandomItemsRestricted then meta[player].storeOn = false end
		Push:FireClient(player, "state", stateOf(player))
	end)
	refreshLeaderstats(player)
	Push:FireClient(player, "state", stateOf(player))
end

local function onLeave(player: Player)
	if profiles[player] then
		saveProfile(player)
		local prof = profiles[player]
		if storesOk then pcall(function() scoreStore:SetAsync(tostring(player.UserId), scoreOf(prof)) end) end
	end
	profiles[player] = nil
	drafts[player] = nil
	meta[player] = nil
	busy[player] = nil
end

MarketplaceService.PromptGamePassPurchaseFinished:Connect(function(player, passId, purchased)
	if purchased and passId == Shared.VIP_PASS_ID and meta[player] then
		meta[player].vip = true
		Push:FireClient(player, "state", stateOf(player))
	end
end)

Players.PlayerAdded:Connect(onJoin)
Players.PlayerRemoving:Connect(onLeave)
for _, p in Players:GetPlayers() do task.spawn(onJoin, p) end

-- autosave
task.spawn(function()
	while true do
		task.wait(90)
		for player, m in meta do
			if m.dirty then
				m.dirty = false
				task.spawn(saveProfile, player)
			end
		end
	end
end)

game:BindToClose(function()
	for _, player in Players:GetPlayers() do
		task.spawn(onLeave, player)
	end
	task.wait(RunService:IsStudio() and 1 or 5)
end)

if not storesOk then
	warn("[Snaproll] DataStores are unavailable. Progress will not save. In Studio, turn on Game Settings > Security > Enable Studio Access to API Services.")
end
```