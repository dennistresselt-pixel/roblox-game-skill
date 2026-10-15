# Retention Systems Reference

Production-ready systems that give players a reason to come back **tomorrow**, **next week**, and **next season**, plus the design rules for tuning them. Everything here is server-authoritative, unit-tested, and type-checks under `--!strict` against the Roblox API definitions.

Use this when the user asks for daily rewards, login streaks, quests, missions, a battle pass / season pass, playtime rewards, comeback bonuses, invite or referral rewards, offline earnings, or more generally "make players come back".

---

## 1. Why Retention Is the Whole Game

Roblox discovery is driven by engagement signals: how many players come back (D1/D7/D30 retention), how long they stay (session length), and how much they play in total. A game with great retention gets recommended more, which brings more players, who also come back. A game with poor retention bleeds out however much you spend on ads.

| Metric | Okay | Good | Great |
|---|---|---|---|
| D1 retention | 15% | 25-30% | 35%+ |
| D7 retention | 5% | 10-15% | 20%+ |
| D30 retention | 1% | 3-5% | 8%+ |
| Avg. session length | 8 min | 15-20 min | 30 min+ |

### The Return Stack

Players come back because they have a **reason at every time scale**. A game that only has a moment-to-moment loop is fun once. Each layer below is a separate "come back" promise. This reference implements every row.

| Time scale | Player thought | System (this file) |
|---|---|---|
| Seconds | "One more!" | Your core loop + reward juice (toasts, pulses, sounds) |
| Minutes | "My next gift is in 1:42" | Playtime gift ladder (§6.3) |
| Session end | "Tomorrow's reward is bigger" | Daily streak + calendar preview (§6.1) |
| Day | "I have 3 new quests" | Daily quests + sweep bonus (§6.2) |
| Week | "Weekly quests reset Monday" | Weekly quests (§6.2) |
| Season | "Tier 18 has the cosmetic I want" | Season pass, free + premium tracks (§6.4) |
| Lapsed | "They gave me a welcome-back bonus" | Comeback reward (§6.5) |
| Social | "My friend is online, we get +20%" | Friend bonus + invite referrals (§6.6) |
| Idle | "My tycoon earned while I was gone" | Offline earnings (§6.5, idle genres) |

---

## 2. The Engagement Line (Read This First)

The goal is a game players **want** to return to, not one they feel trapped by. Most Roblox players are children and teens. Manipulative patterns are against Roblox's rules in several cases, attract reports and moderation, and they also lose on the numbers: they lift D1 and then wreck D30 and your ratings.

| Build this | Not this |
|---|---|
| Streaks that **bank completed weeks** and earned **freezes** that protect them | Streaks that wipe to zero after one missed day |
| Freezes earned by playing | Selling "streak repair" for Robux |
| Countdown to a real reset time | Fake "only 2 left!" scarcity or fake timers that reset |
| Premium season track that is **retroactive** (buy at tier 20, claim 1-20) | Pressure to buy the pass on day one or lose the rewards |
| Time-limited **cosmetics** | Time-limited **power** that permanently weakens players who missed it |
| Free daily reroll for quests | Paid rerolls aimed at kids |
| Paid random items **with odds shown**, gated by `PolicyService` | Undisclosed loot boxes, or showing them where `ArePaidRandomItemsRestricted` is true |
| A gentle break reminder after long sessions | Penalties or loss messages when a player leaves |
| Rewards for playing **with** friends | Guilt messages ("your friends are waiting for you!") |

Policy checks every monetized retention feature must pass:

- Paid random items (eggs, crates, gacha bought with Robux or with currency bought with Robux) must show drop odds, and must be hidden or disabled for players where `PolicyService:GetPolicyInfoForPlayerAsync(player).ArePaidRandomItemsRestricted` is `true`. See `references/monetization-systems.md`.
- Trading of items that can be bought must respect `IsPaidItemTradingAllowed`.
- Notifications must be opt-in (§9) and never misleading.

---

## 3. Architecture

```
ServerStorage/Modules/
  RetentionConfig.luau        -- every tunable number (rewards, pools, season dates)
  RetentionLogic.luau         -- pure math: no Roblox APIs, fully unit-tested
ServerScriptService/
  RetentionService.luau       -- ModuleScript: clock, rewards, remotes, analytics, social
StarterPlayer/StarterPlayerScripts/
  RetentionController.client.luau  -- HUD, panels, toasts, invite + notification prompts
```

```
gameplay code ──track("Collect", 5)──▶ RetentionService ──▶ RetentionLogic (mutates Data.Retention)
                                            │        ▲
                    Retention_Sync / _Event │        │ Retention_Claim* (indices only)
                                            ▼        │
                                      RetentionController (display only)
```

Design decisions:

- **Pure logic module.** `RetentionLogic` takes `now`/`day` as arguments and never touches `os.time()`, DataStores or remotes. That makes streak, quest and season edge cases testable in milliseconds with the standalone `luau` CLI (§11), which is where retention bugs hide (time zones, gaps, double claims).
- **Server clock only.** Every time comparison uses `os.time()` on the server. The client never sends a time, a reward, or an amount, only *which* reward slot it wants to claim.
- **Deterministic quests.** Quests are seeded from `(UserId, day)`, so a player who server-hops sees the same quests everywhere and cannot re-roll by rejoining.
- **Lives in the existing profile.** State is stored as `Data.Retention` inside the profile managed by `DataManager` (see `templates/game-scaffold.md`), so it gets session locking and auto-save for free. `RetentionLogic.reconcile` fills missing fields on load, so existing players migrate automatically.
- **Extensible rewards.** Genres plug in their own reward keys (`Gems`, `Pets`, `Eggs`) with `RetentionService.setRewardHandler`.

### Data Schema (`Data.Retention`)

| Field | Type | Purpose |
|---|---|---|
| `FirstSeen` / `LastSeenDay` / `LastLogout` | number | New vs. returning vs. comeback; offline earnings |
| `Daily.Streak` / `BestStreak` / `LastClaimDay` / `Freezes` | number | Login streak |
| `Quests.Day` / `Week` / `Daily[]` / `Weekly[]` | | Active quests `{ Id, Progress, Claimed }` |
| `Quests.RerollsUsed` / `SweepClaimed` | | Daily limits |
| `Playtime.Day` / `Seconds` / `Claimed{}` | | Gift ladder (accumulates across sessions, resets daily) |
| `Season.Id` / `XP` / `ClaimedFree{}` / `ClaimedPremium{}` | | Season pass |
| `Boosts{ [id] = { Multiplier, ExpiresAt } }` | | Absolute expiry, survives server hops |
| `ReferredBy` / `ReferralsRewarded` / `Titles{}` | | Social + flex |

Claimed sets use **string keys** (`"3"`, not `3`) on purpose: sparse integer-keyed tables do not survive DataStore JSON encoding reliably (see §12).

---

## 4. RetentionConfig (`ServerStorage/Modules/RetentionConfig.luau`)

All tuning lives here. The defaults are a sensible starting point for a simulator. Re-tune them from your analytics (§10) and with the season math in §8.4.

```luau
--!strict
-- RetentionConfig — every tunable number for the retention layer lives here.
-- Tune these from analytics, not from gut feel (see workflows/retention-audit.md).

export type Reward = {
	Cash: number?,
	SeasonXP: number?,
	Items: { { Id: string, Quantity: number } }?,
	Boost: { Id: string, Multiplier: number, Duration: number }?,
	StreakFreeze: number?,
	Title: string?,
}

export type QuestDef = {
	Id: string,
	Event: string, -- matches RetentionService.track(player, Event, amount)
	Target: number,
	Text: string, -- "{n}" is replaced with Target
	Reward: Reward,
}

local RetentionConfig = {}

-- Daily reset happens at this many seconds after 00:00 UTC.
-- 0 = midnight UTC. 17 * 3600 = 10:00 PDT / 19:00 CEST (common "after school" reset).
RetentionConfig.RESET_OFFSET_SECONDS = 0

----------------------------------------------------------------------------
-- 1. Daily login streak
----------------------------------------------------------------------------
-- 7-day calendar. The cycle repeats with a growing multiplier each week.
RetentionConfig.DailyCalendar = {
	{ Cash = 100 },
	{ Cash = 150 },
	{ Cash = 200, SeasonXP = 100 },
	{ Cash = 300 },
	{ Cash = 400, Boost = { Id = "Luck", Multiplier = 2, Duration = 15 * 60 } },
	{ Cash = 500, SeasonXP = 200 },
	{ Cash = 1000, SeasonXP = 500, Items = { { Id = "MysteryChest", Quantity = 1 } } }, -- anchor day
} :: { Reward }

RetentionConfig.WEEK_MULTIPLIER_STEP = 0.25 -- week 2 = 1.25x, week 3 = 1.5x ...
RetentionConfig.WEEK_MULTIPLIER_MAX = 2.0

-- One-off milestone rewards on top of the calendar (flex items / titles).
RetentionConfig.StreakMilestones = {
	[7] = { Title = "Regular" },
	[14] = { Items = { { Id = "StreakAura", Quantity = 1 } } },
	[30] = { Title = "Dedicated", Items = { { Id = "GoldenTrail", Quantity = 1 } } },
	[100] = { Title = "Legend" },
} :: { [number]: Reward }

-- Streak protection: players EARN freezes by playing, they are never sold.
RetentionConfig.FREEZE_EVERY_N_DAYS = 7 -- +1 freeze each completed week
RetentionConfig.MAX_FREEZES = 2
-- When a gap exceeds the freezes, the streak falls back to the start of the
-- current week instead of zero ("checkpoint", never a full wipe).
RetentionConfig.SOFT_RESET_TO_WEEK_START = true

----------------------------------------------------------------------------
-- 2. Daily + weekly quests
----------------------------------------------------------------------------
RetentionConfig.DAILY_QUEST_COUNT = 3
RetentionConfig.WEEKLY_QUEST_COUNT = 4
RetentionConfig.FREE_REROLLS_PER_DAY = 1

-- Bonus for finishing ALL daily quests (the "sweep").
RetentionConfig.DailySweepReward = { Cash = 500, SeasonXP = 300 } :: Reward

RetentionConfig.DailyQuestPool = {
	{ Id = "d_collect", Event = "Collect", Target = 200, Text = "Collect {n} items", Reward = { Cash = 150, SeasonXP = 150 } },
	{ Id = "d_play", Event = "PlayMinutes", Target = 15, Text = "Play for {n} minutes", Reward = { Cash = 150, SeasonXP = 150 } },
	{ Id = "d_spend", Event = "SpendCash", Target = 1000, Text = "Spend {n} cash", Reward = { Cash = 200, SeasonXP = 150 } },
	{ Id = "d_open", Event = "OpenChest", Target = 3, Text = "Open {n} chests", Reward = { Cash = 200, SeasonXP = 150 } },
	{ Id = "d_upgrade", Event = "Upgrade", Target = 2, Text = "Buy {n} upgrades", Reward = { Cash = 150, SeasonXP = 150 } },
	{ Id = "d_friend", Event = "PlayWithFriend", Target = 5, Text = "Play {n} minutes with a friend", Reward = { Cash = 250, SeasonXP = 200 } },
} :: { QuestDef }

RetentionConfig.WeeklyQuestPool = {
	{ Id = "w_collect", Event = "Collect", Target = 5000, Text = "Collect {n} items", Reward = { Cash = 2000, SeasonXP = 1000 } },
	{ Id = "w_play", Event = "PlayMinutes", Target = 120, Text = "Play for {n} minutes", Reward = { Cash = 2000, SeasonXP = 1000 } },
	{ Id = "w_daily", Event = "DailySweep", Target = 4, Text = "Finish all daily quests on {n} days", Reward = { Cash = 3000, SeasonXP = 1500 } },
	{ Id = "w_open", Event = "OpenChest", Target = 25, Text = "Open {n} chests", Reward = { Cash = 2500, SeasonXP = 1000 } },
	{ Id = "w_rebirth", Event = "Rebirth", Target = 1, Text = "Rebirth {n} time", Reward = { Cash = 2500, SeasonXP = 1200 } },
} :: { QuestDef }

----------------------------------------------------------------------------
-- 3. Playtime gift ladder (accumulates across sessions, resets daily)
----------------------------------------------------------------------------
-- Front-loaded: the first gifts unlock fast so a new player gets 3 rewards
-- in their first 5 minutes; later gaps widen.
RetentionConfig.PlaytimeGifts = {
	{ Minutes = 1, Reward = { Cash = 50 } },
	{ Minutes = 3, Reward = { Cash = 75 } },
	{ Minutes = 5, Reward = { Cash = 100, SeasonXP = 50 } },
	{ Minutes = 10, Reward = { Cash = 150 } },
	{ Minutes = 15, Reward = { Cash = 200, SeasonXP = 100 } },
	{ Minutes = 25, Reward = { Items = { { Id = "MysteryChest", Quantity = 1 } } } },
	{ Minutes = 40, Reward = { Cash = 400, SeasonXP = 200 } },
	{ Minutes = 60, Reward = { Boost = { Id = "Cash", Multiplier = 2, Duration = 20 * 60 } } },
} :: { { Minutes: number, Reward: Reward } }

----------------------------------------------------------------------------
-- 4. Season pass (free track + optional premium track)
----------------------------------------------------------------------------
RetentionConfig.Season = {
	Id = "S1",
	Name = "Season 1: Frontier",
	StartsAt = 1793491200, -- 2026-11-01 00:00 UTC; set real dates per season
	EndsAt = 1798934400, -- 2027-01-03 00:00 UTC (~9 weeks)
	XP_PER_TIER = 1000,
	PREMIUM_GAMEPASS_ID = 0, -- 0 = premium track disabled
	-- Tier rewards. Every tier should give *something* on the free track.
	FreeTrack = {} :: { Reward },
	PremiumTrack = {} :: { Reward },
}
-- Generate a 30-tier placeholder track; replace with hand-authored rewards.
for tier = 1, 30 do
	local isAnchor = tier % 5 == 0
	RetentionConfig.Season.FreeTrack[tier] = if isAnchor
		then { Cash = 500 * tier, Items = { { Id = `S1_Cosmetic_{tier}`, Quantity = 1 } } }
		else { Cash = 100 * tier }
	RetentionConfig.Season.PremiumTrack[tier] = if isAnchor
		then { Items = { { Id = `S1_PremiumCosmetic_{tier}`, Quantity = 1 } } }
		else { Cash = 150 * tier }
end

----------------------------------------------------------------------------
-- 5. Comeback, social, offline
----------------------------------------------------------------------------
RetentionConfig.COMEBACK_AFTER_DAYS = 3
RetentionConfig.ComebackReward = {
	Cash = 1000,
	Boost = { Id = "XP", Multiplier = 2, Duration = 30 * 60 },
} :: Reward

RetentionConfig.FRIEND_BONUS_PER_FRIEND = 0.10 -- +10% per friend in the server
RetentionConfig.FRIEND_BONUS_MAX = 0.50
RetentionConfig.InviteReferralReward = { Cash = 500, SeasonXP = 250 } :: Reward -- both inviter and invitee
RetentionConfig.MAX_REFERRAL_REWARDS = 10 -- lifetime cap per inviter (blunts alt-account farming)

RetentionConfig.OFFLINE_EARNINGS_ENABLED = false -- idle / tycoon genres
RetentionConfig.OFFLINE_RATE_FRACTION = 0.5 -- earn 50% of online rate while away
RetentionConfig.OFFLINE_CAP_HOURS = 8

----------------------------------------------------------------------------
-- 6. Healthy-play guardrails
----------------------------------------------------------------------------
-- Gentle break reminder after this much continuous play (0 = disabled).
RetentionConfig.BREAK_REMINDER_MINUTES = 120

return RetentionConfig
```

---

## 5. RetentionLogic (`ServerStorage/Modules/RetentionLogic.luau`)

```luau
--!strict
--[[
	RetentionLogic — pure, deterministic retention math.

	No Roblox APIs in here on purpose: every function takes `now`/`day` as
	arguments, so the whole module runs in the standalone `luau` CLI and in
	unit tests with fake clocks. RetentionService owns time, data and network.

	All functions mutate the `state` table passed in (the player's
	`Data.Retention` table) and return what happened so the caller can grant
	rewards and log analytics.
]]

local RetentionLogic = {}

export type Reward = { [string]: any }

local DAY = 86400

----------------------------------------------------------------------------
-- State
----------------------------------------------------------------------------

local function template()
	return {
		Version = 1,
		FirstSeen = 0,
		LastSeenDay = -1,
		LastLogout = 0,
		ReferredBy = 0,
		ReferralsRewarded = 0,
		Titles = {} :: { [string]: boolean },
		Daily = { Streak = 0, BestStreak = 0, LastClaimDay = -1, Freezes = 0, TotalClaims = 0 },
		Quests = {
			Day = -1,
			Week = -1,
			Daily = {} :: { any },
			Weekly = {} :: { any },
			RerollsUsed = 0,
			SweepClaimed = false,
		},
		Playtime = { Day = -1, Seconds = 0, Claimed = {} :: { [string]: boolean } },
		Season = { Id = "", XP = 0, ClaimedFree = {} :: { [string]: boolean }, ClaimedPremium = {} :: { [string]: boolean } },
		Boosts = {} :: { [string]: { Multiplier: number, ExpiresAt: number } },
	}
end

local function fill(target: { [string]: any }, defaults: { [string]: any })
	for key, value in defaults do
		if target[key] == nil then
			target[key] = value
		elseif type(value) == "table" and type(target[key]) == "table" and next(value) ~= nil then
			fill(target[key], value)
		end
	end
end

-- Returns a fresh state, or fills missing fields of an existing one in place.
function RetentionLogic.reconcile(state: { [string]: any }?): { [string]: any }
	local fresh = template() :: any
	if state == nil then
		return fresh
	end
	fill(state, fresh)
	return state
end

----------------------------------------------------------------------------
-- Time helpers (server clock only — never accept time from the client)
----------------------------------------------------------------------------

function RetentionLogic.dayIndex(now: number, offset: number): number
	return (now - offset) // DAY
end

-- Weeks roll over on Monday (day 0 of the unix epoch was a Thursday).
function RetentionLogic.weekIndex(day: number): number
	return (day + 3) // 7
end

function RetentionLogic.secondsUntilReset(now: number, offset: number): number
	return DAY - ((now - offset) % DAY)
end

----------------------------------------------------------------------------
-- Rewards
----------------------------------------------------------------------------

-- Scales the numeric currency parts of a reward. Items/boosts are not scaled.
function RetentionLogic.scaleReward(reward: Reward, multiplier: number): Reward
	local out = table.clone(reward)
	if out.Cash then
		out.Cash = math.floor(out.Cash * multiplier + 0.5)
	end
	if out.SeasonXP then
		out.SeasonXP = math.floor(out.SeasonXP * multiplier + 0.5)
	end
	return out
end

----------------------------------------------------------------------------
-- Daily login streak
----------------------------------------------------------------------------

function RetentionLogic.canClaimDaily(state: any, day: number): boolean
	return state.Daily.LastClaimDay ~= day
end

-- What the streak WOULD be if the player claimed today. Used by the UI to
-- show "Day 5" before claiming, and by claimDaily to apply it.
-- Returns (streak, freezesUsed, broken).
function RetentionLogic.previewStreak(state: any, day: number, config: any): (number, number, boolean)
	local daily = state.Daily
	if daily.LastClaimDay < 0 or daily.Streak <= 0 then
		return 1, 0, false
	end
	local gap = day - daily.LastClaimDay
	if gap <= 1 then
		return daily.Streak + 1, 0, false
	end
	local missed = gap - 1
	if missed <= daily.Freezes then
		return daily.Streak + 1, missed, false -- earned freezes cover the gap
	end
	if config.SOFT_RESET_TO_WEEK_START then
		-- Completed weeks are banked: a 10-day streak falls back to day 8,
		-- a 14-day streak to day 15. Never back to zero.
		return (daily.Streak // 7) * 7 + 1, 0, true
	end
	return 1, 0, true
end

function RetentionLogic.dailyRewardFor(streak: number, config: any): Reward
	local calendar = config.DailyCalendar
	local slot = ((streak - 1) % #calendar) + 1
	local week = (streak - 1) // #calendar
	local multiplier = math.min(1 + week * config.WEEK_MULTIPLIER_STEP, config.WEEK_MULTIPLIER_MAX)
	return RetentionLogic.scaleReward(calendar[slot], multiplier)
end

export type DailyResult = {
	Streak: number,
	FreezesUsed: number,
	Rewards: { Reward },
	StreakBroken: boolean,
}

function RetentionLogic.claimDaily(state: any, day: number, config: any): DailyResult?
	if not RetentionLogic.canClaimDaily(state, day) then
		return nil
	end
	local daily = state.Daily
	local streak, freezesUsed, broken = RetentionLogic.previewStreak(state, day, config)

	daily.Streak = streak
	daily.BestStreak = math.max(daily.BestStreak, streak)
	daily.LastClaimDay = day
	daily.Freezes -= freezesUsed
	daily.TotalClaims += 1

	local rewards = { RetentionLogic.dailyRewardFor(streak, config) }
	local milestone = config.StreakMilestones[streak]
	if milestone then
		table.insert(rewards, milestone)
	end
	if streak % config.FREEZE_EVERY_N_DAYS == 0 and daily.Freezes < config.MAX_FREEZES then
		table.insert(rewards, { StreakFreeze = 1 })
	end

	return {
		Streak = streak,
		FreezesUsed = freezesUsed,
		Rewards = rewards,
		StreakBroken = broken,
	}
end

----------------------------------------------------------------------------
-- Quests
----------------------------------------------------------------------------

-- Park–Miller PRNG: deterministic per (player, day) so quests are identical
-- on every server the player joins that day, and products stay < 2^53.
local function makeRng(seed: number): () -> number
	local s = seed % 2147483646 + 1
	return function()
		s = (s * 16807) % 2147483647
		return s
	end
end

local function pickDistinct(pool: { any }, count: number, rng: () -> number, exclude: { [string]: boolean }?): { any }
	local candidates = {}
	for _, def in pool do
		if not (exclude and exclude[def.Id]) then
			table.insert(candidates, def)
		end
	end
	local picked = {}
	for _ = 1, math.min(count, #candidates) do
		local index = (rng() % #candidates) + 1
		table.insert(picked, { Id = candidates[index].Id, Progress = 0, Claimed = false })
		table.remove(candidates, index)
	end
	return picked
end

local function findDef(pool: { any }, id: string): any
	for _, def in pool do
		if def.Id == id then
			return def
		end
	end
	return nil
end
RetentionLogic.findDef = findDef

-- Refreshes daily/weekly quest sets when the day/week has rolled over.
-- Returns true if anything was regenerated.
function RetentionLogic.refreshQuests(state: any, day: number, userId: number, config: any): boolean
	local quests = state.Quests
	local changed = false
	if quests.Day ~= day then
		quests.Day = day
		quests.Daily = pickDistinct(config.DailyQuestPool, config.DAILY_QUEST_COUNT, makeRng(userId * 31 + day))
		quests.RerollsUsed = 0
		quests.SweepClaimed = false
		changed = true
	end
	local week = RetentionLogic.weekIndex(day)
	if quests.Week ~= week then
		quests.Week = week
		quests.Weekly = pickDistinct(config.WeeklyQuestPool, config.WEEKLY_QUEST_COUNT, makeRng(userId * 17 + week * 7919))
		changed = true
	end
	return changed
end

-- Adds progress to every active quest listening for `event`.
-- Returns the list of quests that just became complete (for toasts).
function RetentionLogic.track(state: any, event: string, amount: number, config: any): { any }
	local completed = {}
	if amount <= 0 or amount ~= amount then
		return completed
	end
	for _, set in { { state.Quests.Daily, config.DailyQuestPool }, { state.Quests.Weekly, config.WeeklyQuestPool } } do
		for _, quest in set[1] do
			local def = findDef(set[2], quest.Id)
			if def and def.Event == event and quest.Progress < def.Target then
				quest.Progress = math.min(quest.Progress + amount, def.Target)
				if quest.Progress >= def.Target then
					table.insert(completed, def)
				end
			end
		end
	end
	return completed
end

-- kind: "Daily" | "Weekly". Returns rewards to grant (possibly including the
-- daily sweep bonus), or nil if the claim is invalid.
function RetentionLogic.claimQuest(state: any, kind: string, index: number, config: any): { Reward }?
	local list = if kind == "Daily" then state.Quests.Daily elseif kind == "Weekly" then state.Quests.Weekly else nil
	if not list then
		return nil
	end
	local quest = list[index]
	if not quest or quest.Claimed then
		return nil
	end
	local pool = if kind == "Daily" then config.DailyQuestPool else config.WeeklyQuestPool
	local def = findDef(pool, quest.Id)
	if not def or quest.Progress < def.Target then
		return nil
	end

	quest.Claimed = true
	local rewards = { def.Reward }

	if kind == "Daily" and not state.Quests.SweepClaimed then
		local all = true
		for _, q in state.Quests.Daily do
			all = all and q.Claimed
		end
		if all then
			state.Quests.SweepClaimed = true
			table.insert(rewards, config.DailySweepReward)
			RetentionLogic.track(state, "DailySweep", 1, config)
		end
	end
	return rewards
end

function RetentionLogic.rerollQuest(state: any, index: number, userId: number, config: any): boolean
	local quests = state.Quests
	local quest = quests.Daily[index]
	if not quest or quest.Claimed or quests.RerollsUsed >= config.FREE_REROLLS_PER_DAY then
		return false
	end
	local exclude = {}
	for _, q in quests.Daily do
		exclude[q.Id] = true
	end
	local seed = userId * 31 + quests.Day + (quests.RerollsUsed + 1) * 104729
	local replacement = pickDistinct(config.DailyQuestPool, 1, makeRng(seed), exclude)[1]
	if not replacement then
		return false
	end
	quests.Daily[index] = replacement
	quests.RerollsUsed += 1
	return true
end

----------------------------------------------------------------------------
-- Playtime gift ladder
----------------------------------------------------------------------------

function RetentionLogic.addPlaytime(state: any, day: number, seconds: number)
	local playtime = state.Playtime
	if playtime.Day ~= day then
		playtime.Day = day
		playtime.Seconds = 0
		playtime.Claimed = {}
	end
	playtime.Seconds += seconds
end

function RetentionLogic.claimGift(state: any, day: number, index: number, config: any): Reward?
	local playtime = state.Playtime
	local gift = config.PlaytimeGifts[index]
	if not gift or playtime.Day ~= day then
		return nil
	end
	local key = tostring(index)
	if playtime.Claimed[key] or playtime.Seconds < gift.Minutes * 60 then
		return nil
	end
	playtime.Claimed[key] = true
	return gift.Reward
end

-- Seconds until the next unclaimed gift unlocks (0 = one is ready, nil = all done).
function RetentionLogic.nextGiftIn(state: any, config: any): number?
	for index, gift in config.PlaytimeGifts do
		if not state.Playtime.Claimed[tostring(index)] then
			return math.max(0, gift.Minutes * 60 - state.Playtime.Seconds)
		end
	end
	return nil
end

----------------------------------------------------------------------------
-- Season pass
----------------------------------------------------------------------------

function RetentionLogic.seasonActive(season: any, now: number): boolean
	return now >= season.StartsAt and now < season.EndsAt
end

-- Moves the player onto the current season, wiping last season's progress.
function RetentionLogic.syncSeason(state: any, season: any)
	if state.Season.Id ~= season.Id then
		state.Season = { Id = season.Id, XP = 0, ClaimedFree = {}, ClaimedPremium = {} }
	end
end

function RetentionLogic.seasonTier(state: any, season: any): number
	return math.min(state.Season.XP // season.XP_PER_TIER, #season.FreeTrack)
end

-- Returns the list of tiers newly reached (for "Tier up!" celebrations).
function RetentionLogic.addSeasonXP(state: any, amount: number, season: any, now: number): { number }
	if amount <= 0 or not RetentionLogic.seasonActive(season, now) then
		return {}
	end
	RetentionLogic.syncSeason(state, season)
	local before = RetentionLogic.seasonTier(state, season)
	state.Season.XP += amount
	local after = RetentionLogic.seasonTier(state, season)
	local reached = {}
	for tier = before + 1, after do
		table.insert(reached, tier)
	end
	return reached
end

-- Premium rewards stay claimable retroactively: buying the pass at tier 20
-- unlocks tiers 1-20 immediately (good value, not a pressure tactic).
function RetentionLogic.claimSeasonTier(state: any, season: any, tier: number, premium: boolean, ownsPremium: boolean): Reward?
	RetentionLogic.syncSeason(state, season)
	if tier < 1 or tier > RetentionLogic.seasonTier(state, season) then
		return nil
	end
	if premium and not ownsPremium then
		return nil
	end
	local claimed = if premium then state.Season.ClaimedPremium else state.Season.ClaimedFree
	local key = tostring(tier)
	if claimed[key] then
		return nil
	end
	local track = if premium then season.PremiumTrack else season.FreeTrack
	local reward = track[tier]
	if not reward then
		return nil
	end
	claimed[key] = true
	return reward
end

----------------------------------------------------------------------------
-- Login bookkeeping, comeback, offline earnings
----------------------------------------------------------------------------

export type LoginInfo = { IsNew: boolean, DaysAway: number, IsComeback: boolean }

function RetentionLogic.onLogin(state: any, day: number, now: number, config: any): LoginInfo
	local isNew = state.FirstSeen == 0
	if isNew then
		state.FirstSeen = now
	end
	local daysAway = if state.LastSeenDay >= 0 then day - state.LastSeenDay else 0
	state.LastSeenDay = day
	return {
		IsNew = isNew,
		DaysAway = daysAway,
		IsComeback = not isNew and daysAway >= config.COMEBACK_AFTER_DAYS,
	}
end

function RetentionLogic.offlineEarnings(lastLogout: number, now: number, ratePerSecond: number, config: any): number
	if not config.OFFLINE_EARNINGS_ENABLED or lastLogout <= 0 or ratePerSecond <= 0 then
		return 0
	end
	local away = math.clamp(now - lastLogout, 0, config.OFFLINE_CAP_HOURS * 3600)
	return math.floor(away * ratePerSecond * config.OFFLINE_RATE_FRACTION)
end

----------------------------------------------------------------------------
-- Boosts (stored as absolute expiry times so they survive server hops)
----------------------------------------------------------------------------

function RetentionLogic.addBoost(state: any, boost: any, now: number)
	local existing = state.Boosts[boost.Id]
	if existing and existing.ExpiresAt > now and existing.Multiplier == boost.Multiplier then
		existing.ExpiresAt += boost.Duration -- same boost stacks duration
	elseif existing and existing.ExpiresAt > now and existing.Multiplier > boost.Multiplier then
		return -- keep the stronger active boost
	else
		state.Boosts[boost.Id] = { Multiplier = boost.Multiplier, ExpiresAt = now + boost.Duration }
	end
end

function RetentionLogic.boostMultiplier(state: any, id: string, now: number): number
	local boost = state.Boosts[id]
	if boost and boost.ExpiresAt > now then
		return boost.Multiplier
	end
	return 1
end

function RetentionLogic.pruneBoosts(state: any, now: number)
	for id, boost in state.Boosts do
		if boost.ExpiresAt <= now then
			state.Boosts[id] = nil
		end
	end
end

function RetentionLogic.friendBonus(friendsInServer: number, config: any): number
	return 1 + math.min(friendsInServer * config.FRIEND_BONUS_PER_FRIEND, config.FRIEND_BONUS_MAX)
end

return RetentionLogic
```

---

## 6. RetentionService (`ServerScriptService/RetentionService.luau`)

A **ModuleScript** (not a Script) because `GameManager` requires it.

```luau
--!strict
--[[
	RetentionService — server glue for the retention layer.

	Owns: the clock (os.time on the server only), reward granting, remotes,
	analytics, friend bonus, premium-pass checks. All math is in RetentionLogic.

	Wiring (GameManager):
		RetentionService.init()                       -- after RemoteHandler.init()
		DataManager.loadPlayer(player)                -- yields until loaded
		RetentionService.onPlayerLoaded(player)
		...
		RetentionService.onPlayerRemoving(player)     -- BEFORE DataManager.unloadPlayer

	Gameplay code reports progress with one call:
		RetentionService.track(player, "Collect", 5)
	and applies boosts / friend bonus to its own payouts with:
		local amount = base * RetentionService.getMultiplier(player, "Cash")
]]

local Players = game:GetService("Players")
local AnalyticsService = game:GetService("AnalyticsService")
local MarketplaceService = game:GetService("MarketplaceService")
local ServerScriptService = game:GetService("ServerScriptService")
local ServerStorage = game:GetService("ServerStorage")

local DataManager = require(ServerScriptService.DataManager)
local RemoteHandler = require(ServerScriptService.RemoteHandler)
local RetentionLogic = require(ServerStorage.Modules.RetentionLogic)
local Config = require(ServerStorage.Modules.RetentionConfig)

local RetentionService = {}

type Reward = Config.Reward
type RewardHandler = (player: Player, data: { [string]: any }, value: any, source: string) -> ()

local SYNC_REMOTE = "Retention_Sync" -- server -> client snapshot
local EVENT_REMOTE = "Retention_Event" -- server -> client celebrations/toasts
local TICK_SECONDS = 5

local loaded: { [Player]: boolean } = {}
local ownsPremiumPass: { [Player]: boolean } = {}
local friendCounts: { [Player]: number } = {}
local sessionStart: { [Player]: number } = {}
local breakReminded: { [Player]: boolean } = {}
local rewardHandlers: { [string]: RewardHandler } = {}

local function now(): number
	return os.time()
end

local function today(): number
	return RetentionLogic.dayIndex(now(), Config.RESET_OFFSET_SECONDS)
end

local function stateOf(player: Player): any
	local data = DataManager.getData(player)
	return data and data.Retention
end

----------------------------------------------------------------------------
-- Reward granting (extensible per genre: Gems, Pets, Eggs ...)
----------------------------------------------------------------------------

-- Genre templates register handlers for their own reward keys:
--   RetentionService.setRewardHandler("Gems", function(player, data, amount) ... end)
function RetentionService.setRewardHandler(key: string, handler: RewardHandler)
	rewardHandlers[key] = handler
end

rewardHandlers.Cash = function(player, data, amount, source)
	DataManager.updateData(player, "Cash", function(cash)
		return (cash or 0) + amount
	end)
	AnalyticsService:LogEconomyEvent(
		player,
		Enum.AnalyticsEconomyFlowType.Source,
		"Cash",
		amount,
		data.Cash,
		if source == "Playtime" then Enum.AnalyticsEconomyTransactionType.TimedReward.Name else Enum.AnalyticsEconomyTransactionType.Gameplay.Name,
		source
	)
end

rewardHandlers.Items = function(_player, data, items, _source)
	for _, item in items do
		table.insert(data.Inventory, { Id = item.Id, Quantity = item.Quantity })
	end
end

rewardHandlers.Boost = function(_player, data, boost, _source)
	RetentionLogic.addBoost(data.Retention, boost, now())
end

rewardHandlers.StreakFreeze = function(_player, data, amount, _source)
	local daily = data.Retention.Daily
	daily.Freezes = math.min(daily.Freezes + amount, Config.MAX_FREEZES)
end

rewardHandlers.Title = function(_player, data, title, _source)
	data.Retention.Titles[title] = true
end

rewardHandlers.SeasonXP = function(player, _data, amount, _source)
	RetentionService.addSeasonXP(player, amount)
end

local function grant(player: Player, rewards: { Reward }, source: string)
	local data = DataManager.getData(player)
	if not data then
		return
	end
	for _, reward in rewards do
		for key, value in reward :: { [string]: any } do
			local handler = rewardHandlers[key]
			if handler then
				handler(player, data, value, source)
			else
				warn(`[RetentionService] No reward handler for "{key}" (source {source})`)
			end
		end
	end
	RemoteHandler.fireClient(EVENT_REMOTE, player, "Reward", { Source = source, Rewards = rewards })
end

----------------------------------------------------------------------------
-- Snapshot for the client UI (only what the UI needs, nothing secret)
----------------------------------------------------------------------------

local function questView(list: { any }, pool: { any }): { any }
	local out = {}
	for index, quest in list do
		local def = RetentionLogic.findDef(pool, quest.Id)
		if def then
			table.insert(out, {
				Index = index,
				Text = string.gsub(def.Text, "{n}", tostring(def.Target)),
				Progress = quest.Progress,
				Target = def.Target,
				Claimed = quest.Claimed,
				Reward = def.Reward,
			})
		end
	end
	return out
end

local function snapshot(player: Player): any
	local state = stateOf(player)
	local t, d = now(), today()
	local streakIfClaimed = RetentionLogic.previewStreak(state, d, Config)
	local season = Config.Season
	return {
		ResetIn = RetentionLogic.secondsUntilReset(t, Config.RESET_OFFSET_SECONDS),
		Daily = {
			CanClaim = RetentionLogic.canClaimDaily(state, d),
			Streak = state.Daily.Streak,
			NextStreak = streakIfClaimed,
			Freezes = state.Daily.Freezes,
			BestStreak = state.Daily.BestStreak,
			Calendar = Config.DailyCalendar,
			TodayReward = RetentionLogic.dailyRewardFor(streakIfClaimed, Config),
		},
		Quests = {
			Daily = questView(state.Quests.Daily, Config.DailyQuestPool),
			Weekly = questView(state.Quests.Weekly, Config.WeeklyQuestPool),
			RerollsLeft = Config.FREE_REROLLS_PER_DAY - state.Quests.RerollsUsed,
			SweepClaimed = state.Quests.SweepClaimed,
		},
		Playtime = {
			Seconds = state.Playtime.Seconds,
			NextGiftIn = RetentionLogic.nextGiftIn(state, Config),
			Gifts = Config.PlaytimeGifts,
			Claimed = state.Playtime.Claimed,
		},
		Season = if RetentionLogic.seasonActive(season, t)
			then {
				Name = season.Name,
				EndsIn = season.EndsAt - t,
				XP = state.Season.XP,
				Tier = RetentionLogic.seasonTier(state, season),
				XPPerTier = season.XP_PER_TIER,
				TierCount = #season.FreeTrack,
				PremiumPassId = season.PREMIUM_GAMEPASS_ID,
				OwnsPremium = ownsPremiumPass[player] == true,
				ClaimedFree = state.Season.ClaimedFree,
				ClaimedPremium = state.Season.ClaimedPremium,
			}
			else nil,
		Boosts = state.Boosts,
		FriendBonus = RetentionLogic.friendBonus(friendCounts[player] or 0, Config),
	}
end

local function sync(player: Player)
	if loaded[player] then
		RemoteHandler.fireClient(SYNC_REMOTE, player, snapshot(player))
	end
end

----------------------------------------------------------------------------
-- Public API for gameplay systems
----------------------------------------------------------------------------

-- Report gameplay progress. Event names must match RetentionConfig quest pools.
function RetentionService.track(player: Player, event: string, amount: number?)
	local state = stateOf(player)
	if not loaded[player] or not state then
		return
	end
	RetentionLogic.refreshQuests(state, today(), player.UserId, Config)
	local completed = RetentionLogic.track(state, event, amount or 1, Config)
	for _, def in completed do
		RemoteHandler.fireClient(EVENT_REMOTE, player, "QuestComplete", { Text = def.Text, Target = def.Target })
		AnalyticsService:LogCustomEvent(player, "QuestCompleted", 1, { [Enum.AnalyticsCustomFieldKeys.CustomField01.Name] = def.Id })
	end
	if #completed > 0 then
		sync(player)
	end
end

function RetentionService.addSeasonXP(player: Player, amount: number)
	local state = stateOf(player)
	if not state then
		return
	end
	local boosted = math.floor(amount * RetentionLogic.boostMultiplier(state, "XP", now()))
	local reached = RetentionLogic.addSeasonXP(state, boosted, Config.Season, now())
	for _, tier in reached do
		RemoteHandler.fireClient(EVENT_REMOTE, player, "SeasonTier", { Tier = tier })
		AnalyticsService:LogProgressionCompleteEvent(player, "Season_" .. Config.Season.Id, tier)
	end
end

-- Combined multiplier for a payout kind ("Cash", "XP", "Luck" ...):
-- active boost for that kind x friends-in-server bonus.
function RetentionService.getMultiplier(player: Player, kind: string): number
	local state = stateOf(player)
	if not state then
		return 1
	end
	return RetentionLogic.boostMultiplier(state, kind, now()) * RetentionLogic.friendBonus(friendCounts[player] or 0, Config)
end

-- Idle/tycoon genres: provide the player's online income per second.
RetentionService.getIncomePerSecond = function(_player: Player): number
	return 0
end

----------------------------------------------------------------------------
-- Remote handlers (all inputs validated; the client only sends indices)
----------------------------------------------------------------------------

local function isIndex(value: any, max: number): boolean
	return type(value) == "number" and value == math.floor(value) and value >= 1 and value <= max
end

local function onClaimDaily(player: Player)
	local state = stateOf(player)
	local result = state and RetentionLogic.claimDaily(state, today(), Config)
	if not result then
		return
	end
	grant(player, result.Rewards, "DailyLogin")
	AnalyticsService:LogCustomEvent(player, "DailyStreak", result.Streak, {
		[Enum.AnalyticsCustomFieldKeys.CustomField01.Name] = if result.StreakBroken then "broken" else "kept",
	})
	sync(player)
end

local function onClaimQuest(player: Player, kind: any, index: any)
	local state = stateOf(player)
	if not state or (kind ~= "Daily" and kind ~= "Weekly") or not isIndex(index, 10) then
		return
	end
	RetentionLogic.refreshQuests(state, today(), player.UserId, Config)
	local rewards = RetentionLogic.claimQuest(state, kind, index, Config)
	if rewards then
		grant(player, rewards, kind .. "Quest")
		sync(player)
	end
end

local function onRerollQuest(player: Player, index: any)
	local state = stateOf(player)
	if state and isIndex(index, Config.DAILY_QUEST_COUNT) and RetentionLogic.rerollQuest(state, index, player.UserId, Config) then
		sync(player)
	end
end

local function onClaimGift(player: Player, index: any)
	local state = stateOf(player)
	if not state or not isIndex(index, #Config.PlaytimeGifts) then
		return
	end
	local reward = RetentionLogic.claimGift(state, today(), index, Config)
	if reward then
		grant(player, { reward }, "Playtime")
		sync(player)
	end
end

local function onClaimSeasonTier(player: Player, tier: any, premium: any)
	local state = stateOf(player)
	if not state or not isIndex(tier, #Config.Season.FreeTrack) or type(premium) ~= "boolean" then
		return
	end
	local reward = RetentionLogic.claimSeasonTier(state, Config.Season, tier, premium, ownsPremiumPass[player] == true)
	if reward then
		grant(player, { reward }, "Season")
		sync(player)
	end
end

----------------------------------------------------------------------------
-- Social: friend bonus + invite referrals
----------------------------------------------------------------------------

local function recountFriends()
	local players = Players:GetPlayers()
	for _, player in players do
		local count = 0
		for _, other in players do
			if other ~= player then
				local ok, isFriend = pcall(player.IsFriendsWith, player, other.UserId)
				if ok and isFriend then
					count += 1
				end
			end
		end
		if friendCounts[player] ~= count then
			friendCounts[player] = count
			sync(player)
		end
	end
end

-- Invites sent with ExperienceInviteOptions.LaunchData = tostring(inviterUserId)
-- arrive here. Both players are rewarded once, and only for a NEW player.
local function handleReferral(player: Player, state: any, isNew: boolean)
	if not isNew or state.ReferredBy ~= 0 then
		return
	end
	local joinData = player:GetJoinData()
	local inviterId = tonumber(joinData and joinData.LaunchData)
	if not inviterId or inviterId == player.UserId then
		return
	end
	state.ReferredBy = inviterId
	grant(player, { Config.InviteReferralReward }, "Referral")
	local inviter = Players:GetPlayerByUserId(inviterId)
	local inviterState = inviter and loaded[inviter] and stateOf(inviter)
	if inviter and inviterState and inviterState.ReferralsRewarded < Config.MAX_REFERRAL_REWARDS then
		inviterState.ReferralsRewarded += 1
		grant(inviter, { Config.InviteReferralReward }, "Referral")
		sync(inviter)
	end
	-- Inviter offline? Queue it with MemoryStore/DataStore and grant on their next join.
	AnalyticsService:LogCustomEvent(player, "ReferralJoin", 1)
end

----------------------------------------------------------------------------
-- Lifecycle
----------------------------------------------------------------------------

local function checkPremiumPass(player: Player)
	local passId = Config.Season.PREMIUM_GAMEPASS_ID
	if passId == 0 then
		return
	end
	local ok, owns = pcall(MarketplaceService.UserOwnsGamePassAsync, MarketplaceService, player.UserId, passId)
	ownsPremiumPass[player] = ok and owns
end

function RetentionService.onPlayerLoaded(player: Player)
	local data = DataManager.getData(player)
	if not data then
		return
	end
	data.Retention = RetentionLogic.reconcile(data.Retention)
	local state = data.Retention
	local t, d = now(), today()

	local login = RetentionLogic.onLogin(state, d, t, Config)
	RetentionLogic.refreshQuests(state, d, player.UserId, Config)
	RetentionLogic.syncSeason(state, Config.Season)
	RetentionLogic.pruneBoosts(state, t)
	checkPremiumPass(player)

	loaded[player] = true
	sessionStart[player] = t

	if login.IsComeback then
		grant(player, { Config.ComebackReward }, "Comeback")
		RemoteHandler.fireClient(EVENT_REMOTE, player, "WelcomeBack", { DaysAway = login.DaysAway })
	end
	local offline = RetentionLogic.offlineEarnings(state.LastLogout, t, RetentionService.getIncomePerSecond(player), Config)
	if offline > 0 then
		grant(player, { { Cash = offline } }, "Offline")
	end
	handleReferral(player, state, login.IsNew)

	AnalyticsService:LogCustomEvent(player, "SessionStart", 1, {
		[Enum.AnalyticsCustomFieldKeys.CustomField01.Name] = if login.IsNew then "new" elseif login.IsComeback then "comeback" else "returning",
	})

	sync(player)
	task.spawn(recountFriends)
end

function RetentionService.onPlayerRemoving(player: Player)
	local state = stateOf(player)
	if state then
		state.LastLogout = now()
		local minutes = (now() - (sessionStart[player] or now())) / 60
		AnalyticsService:LogCustomEvent(player, "SessionMinutes", math.floor(minutes))
	end
	loaded[player] = nil
	ownsPremiumPass[player] = nil
	friendCounts[player] = nil
	sessionStart[player] = nil
	breakReminded[player] = nil
	task.defer(recountFriends)
end

-- Runs every TICK_SECONDS: playtime ladder, PlayMinutes quests, friend
-- minutes, day rollover for players who stay online past the reset.
local playMinuteRemainder: { [Player]: number } = {}
local lastDay = -1

local function tick()
	local d = today()
	local dayRolled = d ~= lastDay
	lastDay = d
	for player in loaded do
		local state = stateOf(player)
		if state then
			local giftReadyBefore = RetentionLogic.nextGiftIn(state, Config) == 0
			RetentionLogic.addPlaytime(state, d, TICK_SECONDS)

			local acc = (playMinuteRemainder[player] or 0) + TICK_SECONDS
			if acc >= 60 then
				acc -= 60
				RetentionService.track(player, "PlayMinutes", 1)
				if (friendCounts[player] or 0) > 0 then
					RetentionService.track(player, "PlayWithFriend", 1)
				end
			end
			playMinuteRemainder[player] = acc

			if dayRolled then
				RetentionLogic.refreshQuests(state, d, player.UserId, Config)
				RemoteHandler.fireClient(EVENT_REMOTE, player, "NewDay", {})
				sync(player)
			elseif not giftReadyBefore and RetentionLogic.nextGiftIn(state, Config) == 0 then
				RemoteHandler.fireClient(EVENT_REMOTE, player, "GiftReady", {})
				sync(player)
			end

			local played = os.time() - (sessionStart[player] or os.time())
			if Config.BREAK_REMINDER_MINUTES > 0 and not breakReminded[player] and played >= Config.BREAK_REMINDER_MINUTES * 60 then
				breakReminded[player] = true
				RemoteHandler.fireClient(EVENT_REMOTE, player, "BreakReminder", { Minutes = Config.BREAK_REMINDER_MINUTES })
			end
		end
	end
	for player in playMinuteRemainder do
		if not loaded[player] then
			playMinuteRemainder[player] = nil
		end
	end
end

function RetentionService.init()
	RemoteHandler.createRemote(SYNC_REMOTE, "Event")
	RemoteHandler.createRemote(EVENT_REMOTE, "Event")
	RemoteHandler.register({ Name = "Retention_ClaimDaily", Type = "Event", Handler = onClaimDaily, Cooldown = 1 })
	RemoteHandler.register({ Name = "Retention_ClaimQuest", Type = "Event", Handler = onClaimQuest, Cooldown = 0.25 })
	RemoteHandler.register({ Name = "Retention_RerollQuest", Type = "Event", Handler = onRerollQuest, Cooldown = 1 })
	RemoteHandler.register({ Name = "Retention_ClaimGift", Type = "Event", Handler = onClaimGift, Cooldown = 0.25 })
	RemoteHandler.register({ Name = "Retention_ClaimSeasonTier", Type = "Event", Handler = onClaimSeasonTier, Cooldown = 0.1 })
	RemoteHandler.register({ Name = "Retention_RequestSync", Type = "Event", Handler = sync, Cooldown = 2 })

	MarketplaceService.PromptGamePassPurchaseFinished:Connect(function(player, passId, purchased)
		if purchased and passId == Config.Season.PREMIUM_GAMEPASS_ID then
			ownsPremiumPass[player] = true
			AnalyticsService:LogCustomEvent(player, "SeasonPassPurchased", 1)
			sync(player)
		end
	end)

	lastDay = today()
	task.spawn(function()
		while true do
			task.wait(TICK_SECONDS)
			tick()
		end
	end)
end

return RetentionService
```

### 6.1 Daily Streak: Behaviour Summary

- One claim per server day (`RESET_OFFSET_SECONDS` after 00:00 UTC).
- Missed days are covered by **freezes** first: +1 freeze every 7-day milestone, max 2.
- With no freezes left, the streak falls back to the **start of the current week**. Completed weeks are banked: 10 falls back to 8, 14 falls back to 15. It never drops to zero.
- Rewards follow a 7-day calendar that scales +25% per completed week (capped at 2x), and day 7 is the anchor. Milestones at 7/14/30/100 add titles and cosmetics.

### 6.2 Quests: Behaviour Summary

- 3 daily + 4 weekly quests, drawn without repeats from pools and seeded per player per day/week.
- Gameplay reports events with `RetentionService.track(player, event, amount)`, and progress is clamped at the target.
- Finishing all dailies grants the **sweep** bonus *and* advances the weekly "finish dailies on N days" quest. This links the daily loop to the weekly one.
- 1 free reroll per day for a quest that has not been claimed yet.

### 6.3 Playtime Gifts: Behaviour Summary

- Unlock at cumulative minutes played **today** (1, 3, 5, 10, 15, 25, 40, 60). Playtime accrues across sessions, so leaving and rejoining does not reset the ladder.
- Front-loaded: a brand-new player earns 3 gifts in the first 5 minutes.
- The HUD shows a live countdown. That countdown is the "one more minute" hook.

### 6.4 Season Pass: Behaviour Summary

- Season XP comes from daily rewards, quests, gifts and anything else that grants `SeasonXP`, and is multiplied by an active `XP` boost.
- Free track: every tier rewards something. Premium track (optional GamePass): **retroactive**, so buying mid-season unlocks every tier already reached.
- Changing `Season.Id` starts a new season and wipes the previous season's progress. Claimed cosmetics stay in the inventory.

### 6.5 Comeback and Offline

- `COMEBACK_AFTER_DAYS` (default 3) away → welcome-back bundle + 30 min 2x XP. Lapsed players are your cheapest re-acquisition.
- Offline earnings (idle/tycoon only): set `OFFLINE_EARNINGS_ENABLED = true` and override `RetentionService.getIncomePerSecond`. Earnings are 50% of the online rate, capped at 8 hours, so playing always beats waiting.

### 6.6 Social

- **Friend bonus:** +10% per friend in the server (cap +50%), applied through `getMultiplier`. "Play with friends" is the strongest retention driver on Roblox, so reward it directly.
- **Invites:** the client sends `ExperienceInviteOptions.LaunchData = tostring(UserId)`. New players who join through it reward both players once. Rewards are capped per inviter to blunt alt-account farming.

---

## 7. RetentionController (`StarterPlayer/StarterPlayerScripts/RetentionController.client.luau`)

Builds a complete, functional UI in code: HUD buttons with badges and pulses, daily calendar, quest list with claim/reroll, season track with free/premium claims, toasts, invite button, and the notification opt-in prompt. Replace the visuals with your own art later and keep the remote calls.

```luau
--!strict
--[[
	RetentionController.client.luau — HUD + panels for the retention layer.

	Display-only: every claim is a request; the server decides. Builds its UI
	in code so it works with zero Studio setup — swap in your own art later
	and keep the remote calls.

	HUD (left edge):  [Daily !]  [Quests 2]  [Gift 1:42]  [Season T4 ▓▓░]  [Invite]
]]

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local SocialService = game:GetService("SocialService")
local MarketplaceService = game:GetService("MarketplaceService")
local ExperienceNotificationService = game:GetService("ExperienceNotificationService")

local player = Players.LocalPlayer
local Remotes = ReplicatedStorage:WaitForChild("Remotes")

local function remote(name: string): RemoteEvent
	return Remotes:WaitForChild(name) :: RemoteEvent
end

local SyncRemote = remote("Retention_Sync")
local EventRemote = remote("Retention_Event")

local snap: any = nil
local snapClock = 0 -- os.clock() when snap arrived, for local countdowns
local askedForNotifications = false

----------------------------------------------------------------------------
-- Tiny UI helpers
----------------------------------------------------------------------------

local COLORS = {
	Panel = Color3.fromRGB(24, 26, 36),
	Card = Color3.fromRGB(38, 41, 56),
	Accent = Color3.fromRGB(255, 196, 0),
	Good = Color3.fromRGB(70, 200, 110),
	Muted = Color3.fromRGB(150, 155, 175),
	Text = Color3.fromRGB(245, 245, 250),
}

local function make(className: string, props: { [string]: any }, children: { any }?): any
	local inst = Instance.new(className)
	for key, value in props do
		(inst :: any)[key] = value
	end
	if children then
		for _, child in children do
			child.Parent = inst
		end
	end
	return inst
end

local function corner(radius: number?): any
	return make("UICorner", { CornerRadius = UDim.new(0, radius or 10) })
end

local function label(text: string, size: number, props: { [string]: any }?): TextLabel
	local l = make("TextLabel", {
		BackgroundTransparency = 1,
		Text = text,
		TextSize = size,
		Font = Enum.Font.GothamBold,
		TextColor3 = COLORS.Text,
		Size = UDim2.new(1, 0, 0, size + 6),
		TextXAlignment = Enum.TextXAlignment.Left,
		TextWrapped = true,
	})
	if props then
		for key, value in props do
			(l :: any)[key] = value
		end
	end
	return l
end

local function button(text: string, color: Color3, onClick: () -> ()): TextButton
	local b = make("TextButton", {
		Text = text,
		TextSize = 16,
		Font = Enum.Font.GothamBold,
		TextColor3 = Color3.new(0, 0, 0),
		BackgroundColor3 = color,
		AutoButtonColor = true,
		Size = UDim2.new(0, 120, 0, 34),
	}, { corner(8) })
	b.Activated:Connect(onClick)
	return b
end

local function clock(seconds: number): string
	seconds = math.max(0, math.floor(seconds))
	local h, m, s = seconds // 3600, (seconds % 3600) // 60, seconds % 60
	if h > 0 then
		return string.format("%d:%02d:%02d", h, m, s)
	end
	return string.format("%d:%02d", m, s)
end

local function rewardText(reward: any): string
	local parts = {}
	if reward.Cash then table.insert(parts, `{reward.Cash} Cash`) end
	if reward.SeasonXP then table.insert(parts, `{reward.SeasonXP} XP`) end
	if reward.Boost then table.insert(parts, `{reward.Boost.Multiplier}x {reward.Boost.Id} ({reward.Boost.Duration // 60}m)`) end
	if reward.StreakFreeze then table.insert(parts, "Streak Freeze") end
	if reward.Title then table.insert(parts, `Title: {reward.Title}`) end
	for _, item in reward.Items or {} do
		table.insert(parts, `{item.Quantity}x {item.Id}`)
	end
	return table.concat(parts, " + ")
end

----------------------------------------------------------------------------
-- Root + HUD
----------------------------------------------------------------------------

local gui = make("ScreenGui", {
	Name = "RetentionGui",
	ResetOnSpawn = false,
	IgnoreGuiInset = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	Parent = player:WaitForChild("PlayerGui"),
})

local hud = make("Frame", {
	BackgroundTransparency = 1,
	Position = UDim2.new(0, 12, 0.3, 0),
	Size = UDim2.new(0, 150, 0, 260),
	Parent = gui,
}, { make("UIListLayout", { Padding = UDim.new(0, 8) }) })

local function hudButton(text: string, order: number): (TextButton, TextLabel)
	local b = make("TextButton", {
		Text = text,
		TextSize = 16,
		Font = Enum.Font.GothamBold,
		TextColor3 = COLORS.Text,
		BackgroundColor3 = COLORS.Panel,
		Size = UDim2.new(1, 0, 0, 42),
		LayoutOrder = order,
		Parent = hud,
	}, { corner(10) })
	local badge = make("TextLabel", {
		Visible = false,
		Text = "!",
		TextSize = 14,
		Font = Enum.Font.GothamBlack,
		BackgroundColor3 = Color3.fromRGB(235, 64, 52),
		TextColor3 = COLORS.Text,
		AnchorPoint = Vector2.new(0.5, 0.5),
		Position = UDim2.new(1, -4, 0, 4),
		Size = UDim2.new(0, 22, 0, 22),
		Parent = b,
	}, { corner(11) })
	return b, badge
end

local dailyBtn, dailyBadge = hudButton("Daily", 1)
local questBtn, questBadge = hudButton("Quests", 2)
local giftBtn, giftBadge = hudButton("Gift", 3)
local seasonBtn, seasonBadge = hudButton("Season", 4)
local inviteBtn = hudButton("Invite +500", 5)

-- Pulse a HUD button to pull the eye to something claimable.
local pulses: { [Instance]: Tween } = {}
local function setPulse(target: GuiObject, on: boolean)
	if on and not pulses[target] then
		local tween = TweenService:Create(target, TweenInfo.new(0.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {
			BackgroundColor3 = Color3.fromRGB(70, 60, 20),
		})
		pulses[target] = tween
		tween:Play()
	elseif not on and pulses[target] then
		pulses[target]:Cancel()
		pulses[target] = nil
		target.BackgroundColor3 = COLORS.Panel
	end
end

----------------------------------------------------------------------------
-- Panel shell (one open at a time)
----------------------------------------------------------------------------

local panel = make("Frame", {
	Visible = false,
	AnchorPoint = Vector2.new(0.5, 0.5),
	Position = UDim2.new(0.5, 0, 0.5, 0),
	Size = UDim2.new(0, 460, 0, 420),
	BackgroundColor3 = COLORS.Panel,
	Parent = gui,
}, {
	corner(16),
	make("UIPadding", {
		PaddingTop = UDim.new(0, 14),
		PaddingBottom = UDim.new(0, 14),
		PaddingLeft = UDim.new(0, 14),
		PaddingRight = UDim.new(0, 14),
	}),
	make("UISizeConstraint", { MaxSize = Vector2.new(460, 420) }),
	make("UIAspectRatioConstraint", { AspectRatio = 460 / 420 }),
})
local panelTitle = label("", 22, { Parent = panel, Size = UDim2.new(1, -40, 0, 28) })
local closeBtn = make("TextButton", {
	Text = "X",
	TextSize = 18,
	Font = Enum.Font.GothamBlack,
	TextColor3 = COLORS.Text,
	BackgroundTransparency = 1,
	Position = UDim2.new(1, -28, 0, 0),
	Size = UDim2.new(0, 28, 0, 28),
	Parent = panel,
})
local body = make("ScrollingFrame", {
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	Position = UDim2.new(0, 0, 0, 36),
	Size = UDim2.new(1, 0, 1, -36),
	AutomaticCanvasSize = Enum.AutomaticSize.Y,
	CanvasSize = UDim2.new(),
	ScrollBarThickness = 4,
	Parent = panel,
})

local openPanel: string? = nil
local renderers: { [string]: () -> () } = {}

local function clearBody()
	for _, child in body:GetChildren() do
		child:Destroy()
	end
	make("UIListLayout", { Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder, Parent = body })
end

local function show(name: string?)
	openPanel = name
	panel.Visible = name ~= nil
	if name then
		panelTitle.Text = name
		clearBody()
		renderers[name]()
	end
end
closeBtn.Activated:Connect(function()
	show(nil)
end)

local function card(order: number, height: number): Frame
	return make("Frame", {
		BackgroundColor3 = COLORS.Card,
		Size = UDim2.new(1, -6, 0, height),
		LayoutOrder = order,
		Parent = body,
	}, { corner(10), make("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingTop = UDim.new(0, 6) }) })
end

local function progressBar(parent: Instance, fraction: number, y: number)
	local track = make("Frame", {
		BackgroundColor3 = Color3.fromRGB(60, 64, 84),
		Position = UDim2.new(0, 0, 0, y),
		Size = UDim2.new(1, -140, 0, 10),
		Parent = parent,
	}, { corner(5) })
	make("Frame", {
		BackgroundColor3 = if fraction >= 1 then COLORS.Good else COLORS.Accent,
		Size = UDim2.new(math.clamp(fraction, 0, 1), 0, 1, 0),
		Parent = track,
	}, { corner(5) })
end

----------------------------------------------------------------------------
-- Daily panel
----------------------------------------------------------------------------

local function maybeAskForNotifications()
	-- Best moment to ask: right after a reward, with a concrete promise.
	if askedForNotifications then
		return
	end
	askedForNotifications = true
	task.delay(1.5, function()
		local ok, canPrompt = pcall(ExperienceNotificationService.CanPromptOptInAsync, ExperienceNotificationService)
		if ok and canPrompt then
			ExperienceNotificationService:PromptOptIn()
		end
	end)
end

renderers.Daily = function()
	local d = snap.Daily
	local streakDay = if d.CanClaim then d.NextStreak else d.Streak
	label(`Streak: {d.Streak} days   Best: {d.BestStreak}   Freezes: {d.Freezes}`, 16, { Parent = body, LayoutOrder = 0 })

	local row = make("Frame", { BackgroundTransparency = 1, Size = UDim2.new(1, 0, 0, 86), LayoutOrder = 1, Parent = body }, {
		make("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 6) }),
	})
	local slotToday = ((streakDay - 1) % #d.Calendar) + 1
	for slot, reward in d.Calendar do
		local isToday = slot == slotToday
		local done = slot < slotToday or (slot == slotToday and not d.CanClaim)
		make("Frame", {
			BackgroundColor3 = if isToday and d.CanClaim then COLORS.Accent elseif done then COLORS.Good else COLORS.Card,
			Size = UDim2.new(1 / #d.Calendar, -6, 1, 0),
			Parent = row,
		}, {
			corner(8),
			label(`Day {slot}`, 12, { TextXAlignment = Enum.TextXAlignment.Center, TextColor3 = Color3.new(0, 0, 0) }),
			label(if done then "OK" else tostring(reward.Cash or "?"), 14, {
				Position = UDim2.new(0, 0, 0, 30),
				TextXAlignment = Enum.TextXAlignment.Center,
			}),
		})
	end

	label(`Today: {rewardText(d.TodayReward)}`, 15, { Parent = body, LayoutOrder = 2, TextColor3 = COLORS.Accent })
	if d.CanClaim then
		local b = button(`Claim Day {d.NextStreak}`, COLORS.Accent, function()
			remote("Retention_ClaimDaily"):FireServer()
			maybeAskForNotifications()
		end)
		b.LayoutOrder = 3
		b.Size = UDim2.new(1, -6, 0, 44)
		b.Parent = body
	else
		label("Come back tomorrow to keep your streak!", 15, { Parent = body, LayoutOrder = 3, TextColor3 = COLORS.Muted })
	end
	local resetIn = snap.ResetIn - (os.clock() - snapClock)
	label(`Next reward in {clock(resetIn)}`, 14, { Parent = body, LayoutOrder = 4, TextColor3 = COLORS.Muted, Name = "Countdown" })
	label("Freezes protect your streak if you miss a day. You earn one every 7 days.", 12, {
		Parent = body,
		LayoutOrder = 5,
		TextColor3 = COLORS.Muted,
	})
end

----------------------------------------------------------------------------
-- Quests panel
----------------------------------------------------------------------------

local function questRows(list: { any }, kind: string, startOrder: number, canReroll: boolean)
	for i, q in list do
		local c = card(startOrder + i, 62)
		label(q.Text, 15, { Parent = c, Size = UDim2.new(1, -130, 0, 20) })
		label(`{q.Progress}/{q.Target}  -  {rewardText(q.Reward)}`, 12, {
			Parent = c,
			Position = UDim2.new(0, 0, 0, 22),
			Size = UDim2.new(1, -130, 0, 16),
			TextColor3 = COLORS.Muted,
		})
		progressBar(c, q.Progress / q.Target, 42)
		local action: TextButton? = nil
		if q.Claimed then
			action = button("Done", COLORS.Muted, function() end)
		elseif q.Progress >= q.Target then
			action = button("Claim", COLORS.Good, function()
				remote("Retention_ClaimQuest"):FireServer(kind, q.Index)
			end)
		elseif canReroll then
			local reroll = button("Reroll", COLORS.Card, function()
				remote("Retention_RerollQuest"):FireServer(q.Index)
			end)
			reroll.TextColor3 = COLORS.Text
			action = reroll
		end
		if action then
			action.Position = UDim2.new(1, -126, 0, 8)
			action.Parent = c
		end
	end
end

renderers.Quests = function()
	local q = snap.Quests
	local sweep = if q.SweepClaimed then "Daily bonus claimed!" else "Finish all 3 for a bonus chest"
	label(`Daily - resets in {clock(snap.ResetIn - (os.clock() - snapClock))} - {sweep}`, 14, {
		Parent = body,
		LayoutOrder = 0,
		TextColor3 = COLORS.Accent,
	})
	questRows(q.Daily, "Daily", 0, q.RerollsLeft > 0)
	label("Weekly", 16, { Parent = body, LayoutOrder = 50 })
	questRows(q.Weekly, "Weekly", 50, false)
end

----------------------------------------------------------------------------
-- Season panel
----------------------------------------------------------------------------

renderers.Season = function()
	local s = snap.Season
	if not s then
		label("No active season. Check back soon!", 16, { Parent = body })
		return
	end
	label(`{s.Name} - ends in {clock(s.EndsIn - (os.clock() - snapClock))}`, 15, { Parent = body, LayoutOrder = 0 })
	local c = card(1, 40)
	local into = s.XP % s.XPPerTier
	label(`Tier {s.Tier}/{s.TierCount}   {into}/{s.XPPerTier} XP`, 14, { Parent = c })
	progressBar(c, if s.Tier >= s.TierCount then 1 else into / s.XPPerTier, 22)

	for tier = 1, math.min(s.Tier + 3, s.TierCount) do
		local row = card(10 + tier, 40)
		local reached = tier <= s.Tier
		label(`Tier {tier}`, 14, { Parent = row, TextColor3 = if reached then COLORS.Text else COLORS.Muted })
		local function claimButton(premium: boolean, x: number)
			local claimed = (if premium then s.ClaimedPremium else s.ClaimedFree)[tostring(tier)]
			local text = if claimed then "Got" elseif premium then "Premium" else "Free"
			local locked = not reached or claimed or (premium and not s.OwnsPremium)
			local b = button(text, if locked then COLORS.Muted else COLORS.Good, function()
				if premium and not s.OwnsPremium and s.PremiumPassId ~= 0 then
					MarketplaceService:PromptGamePassPurchase(player, s.PremiumPassId)
				elseif not locked then
					remote("Retention_ClaimSeasonTier"):FireServer(tier, premium)
				end
			end)
			b.Size = UDim2.new(0, 90, 0, 28)
			b.Position = UDim2.new(1, x, 0, 0)
			b.Parent = row
		end
		claimButton(false, -200)
		if s.PremiumPassId ~= 0 then
			claimButton(true, -104)
		end
	end
end

----------------------------------------------------------------------------
-- Toasts (celebrate every win — this is where "juice" lives)
----------------------------------------------------------------------------

local toastStack = make("Frame", {
	BackgroundTransparency = 1,
	AnchorPoint = Vector2.new(0.5, 0),
	Position = UDim2.new(0.5, 0, 0, 60),
	Size = UDim2.new(0, 360, 0, 300),
	Parent = gui,
}, { make("UIListLayout", { Padding = UDim.new(0, 6), HorizontalAlignment = Enum.HorizontalAlignment.Center }) })

local function toast(text: string, color: Color3?)
	local t = make("TextLabel", {
		Text = text,
		TextSize = 18,
		Font = Enum.Font.GothamBlack,
		TextColor3 = Color3.new(0, 0, 0),
		BackgroundColor3 = color or COLORS.Accent,
		Size = UDim2.new(0, 0, 0, 40),
		AutomaticSize = Enum.AutomaticSize.X,
		Parent = toastStack,
	}, { corner(10), make("UIPadding", { PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14) }) })
	local scale = make("UIScale", { Scale = 0.6, Parent = t })
	TweenService:Create(scale, TweenInfo.new(0.25, Enum.EasingStyle.Back), { Scale = 1 }):Play()
	task.delay(3, function()
		local fade = TweenService:Create(t, TweenInfo.new(0.4), { BackgroundTransparency = 1, TextTransparency = 1 })
		fade:Play()
		fade.Completed:Wait()
		t:Destroy()
	end)
end

EventRemote.OnClientEvent:Connect(function(kind: string, payload: any)
	if kind == "Reward" then
		for _, reward in payload.Rewards do
			toast(`+ {rewardText(reward)}`)
		end
	elseif kind == "QuestComplete" then
		toast(`Quest complete: {(string.gsub(payload.Text, "{n}", tostring(payload.Target)))}`, COLORS.Good)
	elseif kind == "SeasonTier" then
		toast(`Season tier {payload.Tier} unlocked!`, COLORS.Good)
	elseif kind == "WelcomeBack" then
		toast(`Welcome back! You were away {payload.DaysAway} days - bonus waiting`, COLORS.Good)
	elseif kind == "GiftReady" then
		toast("A free gift is ready!")
	elseif kind == "NewDay" then
		toast("New day! Fresh quests and a daily reward are ready")
	elseif kind == "BreakReminder" then
		toast(`You've played {payload.Minutes} minutes - nice! Maybe stretch and grab some water`, COLORS.Muted)
	end
end)

----------------------------------------------------------------------------
-- HUD state + buttons
----------------------------------------------------------------------------

local function claimableQuests(): number
	local n = 0
	for _, list in { snap.Quests.Daily, snap.Quests.Weekly } do
		for _, q in list do
			if not q.Claimed and q.Progress >= q.Target then
				n += 1
			end
		end
	end
	return n
end

local function claimableTiers(): number
	local s = snap.Season
	if not s then
		return 0
	end
	local n = 0
	for tier = 1, s.Tier do
		if not s.ClaimedFree[tostring(tier)] then n += 1 end
		if s.OwnsPremium and not s.ClaimedPremium[tostring(tier)] then n += 1 end
	end
	return n
end

local function refreshHud()
	if not snap then
		return
	end
	dailyBadge.Visible = snap.Daily.CanClaim
	setPulse(dailyBtn, snap.Daily.CanClaim)
	dailyBtn.Text = if snap.Daily.Streak > 0 then `Daily  ({snap.Daily.Streak}d)` else "Daily"

	local nq = claimableQuests()
	questBadge.Visible = nq > 0
	questBadge.Text = tostring(nq)

	local nt = claimableTiers()
	seasonBadge.Visible = nt > 0
	seasonBadge.Text = tostring(nt)
	seasonBtn.Text = if snap.Season then `Season  T{snap.Season.Tier}` else "Season"
end

local function updateGiftButton()
	if not snap then
		return
	end
	local nextIn = snap.Playtime.NextGiftIn
	if nextIn == nil then
		giftBtn.Text = "Gifts done!"
		giftBadge.Visible = false
		setPulse(giftBtn, false)
		return
	end
	local remaining = nextIn - (os.clock() - snapClock)
	local ready = remaining <= 0
	giftBtn.Text = if ready then "Gift ready!" else `Gift in {clock(remaining)}`
	giftBadge.Visible = ready
	setPulse(giftBtn, ready)
end

giftBtn.Activated:Connect(function()
	if not snap then
		return
	end
	local elapsed = snap.Playtime.Seconds + (os.clock() - snapClock)
	for index, gift in snap.Playtime.Gifts do
		if not snap.Playtime.Claimed[tostring(index)] and elapsed >= gift.Minutes * 60 then
			remote("Retention_ClaimGift"):FireServer(index)
		end
	end
end)

dailyBtn.Activated:Connect(function()
	show(if openPanel == "Daily" then nil else "Daily")
end)
questBtn.Activated:Connect(function()
	show(if openPanel == "Quests" then nil else "Quests")
end)
seasonBtn.Activated:Connect(function()
	show(if openPanel == "Season" then nil else "Season")
end)

inviteBtn.Activated:Connect(function()
	local ok, canInvite = pcall(SocialService.CanSendGameInviteAsync, SocialService, player)
	if ok and canInvite then
		local options = Instance.new("ExperienceInviteOptions")
		options.PromptMessage = "Play with me! We both get a bonus."
		options.LaunchData = tostring(player.UserId) -- read by RetentionService referral handler
		SocialService:PromptGameInvite(player, options)
	end
end)

SyncRemote.OnClientEvent:Connect(function(data: any)
	local first = snap == nil
	snap = data
	snapClock = os.clock()
	refreshHud()
	updateGiftButton()
	if openPanel then
		show(openPanel) -- re-render with fresh data
	elseif first and data.Daily.CanClaim then
		show("Daily") -- the first thing a returning player sees is their streak
	end
end)

-- Countdown ticker (cheap: text-only updates once per second)
task.spawn(function()
	while true do
		task.wait(1)
		updateGiftButton()
		if openPanel == "Daily" and snap then
			local countdown = body:FindFirstChild("Countdown") :: TextLabel?
			if countdown then
				countdown.Text = `Next reward in {clock(snap.ResetIn - (os.clock() - snapClock))}`
			end
		end
	end
end)

remote("Retention_RequestSync"):FireServer()
```

---

## 8. Integration and Design Playbook

### 8.1 Wiring (already done in `templates/game-scaffold.md`)

`GameManager` loads `RetentionService` if it exists in `ServerScriptService`:

```luau
RemoteHandler.init()
RetentionService.init()                       -- registers Retention_* remotes

-- PlayerAdded
DataManager.loadPlayer(player)                -- yields
RetentionService.onPlayerLoaded(player)

-- PlayerRemoving (ORDER MATTERS: retention writes LastLogout before the save)
RetentionService.onPlayerRemoving(player)
DataManager.unloadPlayer(player)
```

### 8.2 Hooking Gameplay: `track()` Events per Genre

Add one line where the action happens on the **server**, and make sure the quest pool uses the same event names.

| Genre | Where | Call |
|---|---|---|
| Simulator | Collection handler | `track(player, "Collect", amount)` |
| Simulator | Egg hatch | `track(player, "OpenChest", 1)` |
| Simulator | Rebirth | `track(player, "Rebirth", 1)` |
| Tycoon | Button pad purchase | `track(player, "Upgrade", 1)`; `track(player, "SpendCash", price)` |
| Obby | Checkpoint reached | `track(player, "Stage", 1)` |
| RPG | Enemy killed / quest turned in | `track(player, "Defeat", 1)`; `track(player, "NpcQuest", 1)` |
| Horror | Room survived / round won | `track(player, "Survive", 1)` |
| Battle Royale | Match end | `track(player, "Match", 1)`; `track(player, "Elimination", kills)` |

`PlayMinutes` and `PlayWithFriend` are tracked automatically by the service tick.

Apply boosts and the friend bonus to your own payouts:

```luau
local payout = math.floor(baseAmount * RetentionService.getMultiplier(player, "Cash"))
```

Plug in a genre currency or item type:

```luau
RetentionService.setRewardHandler("Gems", function(player, _data, amount, _source)
	DataManager.updateData(player, "Gems", function(gems)
		return (gems or 0) + amount
	end)
end)
-- RetentionConfig: { Gems = 25 } now works in any reward table
```

### 8.3 The First Five Minutes and the Last Five Seconds

Retention is decided at two moments:

1. **First session (D1).** Within 60 seconds the player should have *earned* something (the 1-minute gift), within 5 minutes they should have finished a quest and seen the season bar move. Pick one daily quest from a "starter" pool for brand-new players if your pool has hard quests.
2. **Session end.** Before a player leaves, they should know what is waiting tomorrow. The daily panel shows tomorrow's slot and the countdown. Surface it again whenever the player claims their last gift of the day.

### 8.4 Season Math (Tune Before Launch)

```
Total XP needed    = XP_PER_TIER × tiers
Engaged XP per day ≈ login + daily quests + sweep + gifts + (weekly quests ÷ 7)
Target              = engaged players finish the free track by playing ~60% of season days

XP_PER_TIER ≈ (Engaged XP per day × season length in days × 0.6) ÷ tiers
```

With the default config, an engaged day gives about 115 (login, averaged over the week) + 450 (quests) + 300 (sweep) + 350 (gifts) + about 650 (weeklies ÷ 7) ≈ 1,850 XP. For a 63-day, 30-tier season that suggests `XP_PER_TIER ≈ 2,300`. The shipped `1000` is deliberately generous for testing. Recompute this for every season.

### 8.5 Reward Juice (Cheap, Huge Impact)

- Every claim: a toast with a scale "pop" (included), a sound, and particles near the HUD button.
- Claimable things **pulse** and show a red badge count (included). A visible "you have 2 things to claim" is the strongest pull back into your systems.
- Big moments (day 7, tier up, sweep) deserve a full-screen moment. See `references/animation-vfx.md`.

---

## 9. Experience Notifications (Bring Players Back When the Game Is Closed)

Roblox Experience Notifications reach opted-in players outside the game. They are the most powerful re-engagement tool Roblox offers, and also the one most easily abused.

1. **Opt-in, at the right moment.** The controller calls `ExperienceNotificationService:PromptOptIn()` right after the first daily claim. At that point the promise ("we'll tell you when tomorrow's reward is ready") is concrete. Always check `CanPromptOptInAsync()` first.
2. **Create message templates** in Creator Hub (Engagement → Notifications). Templates support parameters like `{streakDay}`.
3. **Send** through the Open Cloud User Notifications API from your backend or a scheduled job. Check the current Open Cloud docs for whether the endpoint can be called from game servers with `HttpService`. Good triggers: "Your day {n} reward is ready", "New season starts today", "Your friend {name} joined". Bad triggers: anything that implies loss or guilt.
4. Roblox throttles notifications per user and ranks them by engagement. Low-quality spam lowers your delivery rate, so send fewer, better ones.

---

## 10. Analytics: Measure Every Lever

The service logs these through `AnalyticsService`. View them in Creator Hub → Analytics.

| Event | Type | Use |
|---|---|---|
| `SessionStart` (new / returning / comeback) | Custom | Return-mix health |
| `SessionMinutes` | Custom | Session length distribution |
| `DailyStreak` (kept / broken) | Custom | How often freezes save streaks |
| `QuestCompleted` (quest id) | Custom | Find quests nobody finishes (too hard) or everyone finishes (too easy) |
| `ReferralJoin`, `SeasonPassPurchased` | Custom | Social + monetization |
| Season tier reached | Progression | Season pacing vs. §8.4 target |
| Every currency grant | Economy (Source) | Inflation check: retention rewards vs. gameplay income |

Add an **onboarding funnel** to see where new players drop off in their first session. This drives D1 more than anything else:

```luau
local AnalyticsService = game:GetService("AnalyticsService")
-- Call once per step, server-side, in order. Steps must be sequential integers.
AnalyticsService:LogOnboardingFunnelStepEvent(player, 1, "Spawned")
AnalyticsService:LogOnboardingFunnelStepEvent(player, 2, "FirstCollect")
AnalyticsService:LogOnboardingFunnelStepEvent(player, 3, "FirstGiftClaimed")
AnalyticsService:LogOnboardingFunnelStepEvent(player, 4, "FirstQuestCompleted")
AnalyticsService:LogOnboardingFunnelStepEvent(player, 5, "FirstUpgradeBought")
```

How to read it: find the step with the biggest drop and fix that first. A 40% loss between "Spawned" and "FirstCollect" means the game does not tell players what to do. No retention reward fixes that.

---

## 11. Testing

`RetentionLogic` has no Roblox dependencies, so its spec runs anywhere:

- **CLI:** put `RetentionLogic.luau`, `RetentionConfig.luau` and the spec in one folder and run `luau RetentionLogic.spec.luau`. The `luau` binary is on the luau-lang GitHub releases page.
- **Studio:** paste the spec into a Script, replace the two `require("./...")` lines with `require(game.ServerStorage.Modules.RetentionLogic)` etc., and run it.
- **TestEZ / Jest-Lua:** wrap each `test(...)` in an `it(...)` (see `references/testing-patterns.md`).

```luau
-- Plain-Luau spec for RetentionLogic. Runs with the `luau` CLI or the Studio
-- command bar (swap the requires for instance paths).
local Logic = require("./RetentionLogic")
local Config = require("./RetentionConfig")

local passed, failed = 0, 0
local function test(name: string, fn: () -> ())
	local ok, err = pcall(fn)
	if ok then
		passed += 1
	else
		failed += 1
		print(`FAIL {name}: {err}`)
	end
end
local function eq(actual: any, expected: any, label: string?)
	if actual ~= expected then
		error(`{label or "value"}: expected {expected}, got {actual}`, 2)
	end
end

local DAY = 86400
local T0 = 1793491200 + 5 * DAY -- inside Season 1
local function day(n: number): number
	return Logic.dayIndex(T0 + n * DAY, Config.RESET_OFFSET_SECONDS)
end

test("reconcile fills missing fields without clobbering", function()
	local s = Logic.reconcile({ Daily = { Streak = 4 } })
	eq(s.Daily.Streak, 4, "streak kept")
	eq(s.Daily.Freezes, 0, "freezes filled")
	eq(type(s.Quests.Daily), "table", "quests filled")
end)

test("daily claim once per day, streak grows", function()
	local s = Logic.reconcile(nil)
	eq(Logic.claimDaily(s, day(0), Config).Streak, 1)
	eq(Logic.claimDaily(s, day(0), Config), nil, "double claim")
	eq(Logic.claimDaily(s, day(1), Config).Streak, 2)
	eq(Logic.claimDaily(s, day(2), Config).Streak, 3)
end)

test("day 7 grants anchor reward and a freeze", function()
	local s = Logic.reconcile(nil)
	local r
	for d = 0, 6 do
		r = Logic.claimDaily(s, day(d), Config)
	end
	eq(r.Streak, 7)
	eq(r.Rewards[1].Cash, 1000, "anchor cash")
	eq(r.Rewards[2].Title, "Regular", "milestone")
	eq(r.Rewards[3].StreakFreeze, 1, "freeze")
end)

test("week 2 rewards are scaled", function()
	eq(Logic.dailyRewardFor(8, Config).Cash, 125) -- 100 * 1.25
	eq(Logic.dailyRewardFor(64, Config).Cash, 200) -- capped at 2x
end)

test("freezes cover missed days", function()
	local s = Logic.reconcile(nil)
	for d = 0, 4 do
		Logic.claimDaily(s, day(d), Config)
	end
	s.Daily.Freezes = 1
	local r = Logic.claimDaily(s, day(6), Config) -- missed day 5
	eq(r.Streak, 6)
	eq(r.FreezesUsed, 1)
	eq(r.StreakBroken, false)
	eq(s.Daily.Freezes, 0)
end)

test("long gap falls back to week checkpoint, not zero", function()
	local s = Logic.reconcile(nil)
	for d = 0, 9 do
		Logic.claimDaily(s, day(d), Config)
	end
	eq(s.Daily.Streak, 10)
	local r = Logic.claimDaily(s, day(20), Config)
	eq(r.Streak, 8, "back to start of week 2")
	eq(r.StreakBroken, true)
	eq(s.Daily.BestStreak, 10)
end)

test("quests are deterministic per player/day and distinct", function()
	local a, b = Logic.reconcile(nil), Logic.reconcile(nil)
	Logic.refreshQuests(a, day(0), 12345, Config)
	Logic.refreshQuests(b, day(0), 12345, Config)
	eq(#a.Quests.Daily, Config.DAILY_QUEST_COUNT)
	eq(#a.Quests.Weekly, Config.WEEKLY_QUEST_COUNT)
	local seen = {}
	for i, q in a.Quests.Daily do
		eq(q.Id, b.Quests.Daily[i].Id, "same on every server")
		eq(seen[q.Id], nil, "distinct")
		seen[q.Id] = true
	end
	eq(Logic.refreshQuests(a, day(0), 12345, Config), false, "no regen same day")
	eq(Logic.refreshQuests(a, day(1), 12345, Config), true, "regen next day")
end)

test("track -> claim -> sweep bonus -> weekly DailySweep progress", function()
	local s = Logic.reconcile(nil)
	Logic.refreshQuests(s, day(0), 1, Config)
	-- complete all daily quests
	for _, q in s.Quests.Daily do
		local def = Logic.findDef(Config.DailyQuestPool, q.Id)
		Logic.track(s, def.Event, def.Target * 2, Config)
		eq(q.Progress, def.Target, "progress clamps at target")
	end
	eq(Logic.claimQuest(s, "Daily", 1, Config) ~= nil, true)
	eq(Logic.claimQuest(s, "Daily", 1, Config), nil, "no double claim")
	eq(#Logic.claimQuest(s, "Daily", 2, Config), 1)
	local last = Logic.claimQuest(s, "Daily", 3, Config)
	eq(#last, 2, "sweep bonus attached")
	eq(s.Quests.SweepClaimed, true)
	for _, q in s.Quests.Weekly do
		if q.Id == "w_daily" then
			eq(q.Progress, 1, "weekly sweep quest advanced")
		end
	end
end)

test("cannot claim incomplete quest or bad index/kind", function()
	local s = Logic.reconcile(nil)
	Logic.refreshQuests(s, day(0), 1, Config)
	eq(Logic.claimQuest(s, "Daily", 1, Config), nil)
	eq(Logic.claimQuest(s, "Daily", 99, Config), nil)
	eq(Logic.claimQuest(s, "Hacked", 1, Config), nil)
	eq(#Logic.track(s, "Collect", -50, Config), 0, "negative ignored")
	eq(#Logic.track(s, "Collect", 0 / 0, Config), 0, "NaN ignored")
end)

test("reroll replaces with a new quest, limited per day", function()
	local s = Logic.reconcile(nil)
	Logic.refreshQuests(s, day(0), 99, Config)
	local before = s.Quests.Daily[2].Id
	eq(Logic.rerollQuest(s, 2, 99, Config), true)
	eq(s.Quests.Daily[2].Id ~= before, true, "changed")
	eq(Logic.rerollQuest(s, 1, 99, Config), false, "out of rerolls")
end)

test("playtime gifts unlock by time and reset daily", function()
	local s = Logic.reconcile(nil)
	Logic.addPlaytime(s, day(0), 59)
	eq(Logic.claimGift(s, day(0), 1, Config), nil, "too early")
	eq(Logic.nextGiftIn(s, Config), 1)
	Logic.addPlaytime(s, day(0), 1)
	eq(Logic.claimGift(s, day(0), 1, Config).Cash, 50)
	eq(Logic.claimGift(s, day(0), 1, Config), nil, "no double")
	Logic.addPlaytime(s, day(1), 10)
	eq(s.Playtime.Seconds, 10, "reset on new day")
	eq(Logic.claimGift(s, day(0), 2, Config), nil, "stale day rejected")
end)

test("season XP -> tiers -> free/premium claims", function()
	local s = Logic.reconcile(nil)
	local season = Config.Season
	local now = T0
	local reached = Logic.addSeasonXP(s, 2500, season, now)
	eq(#reached, 2)
	eq(Logic.claimSeasonTier(s, season, 3, false, false), nil, "tier 3 not reached")
	eq(Logic.claimSeasonTier(s, season, 2, false, false).Cash, 200)
	eq(Logic.claimSeasonTier(s, season, 2, true, false), nil, "premium needs pass")
	eq(Logic.claimSeasonTier(s, season, 2, true, true).Cash, 300, "retroactive premium")
	eq(#Logic.addSeasonXP(s, 100, season, season.EndsAt + 1), 0, "no XP after season end")
	eq(Logic.seasonTier(s, season), 2)
	s.Season.XP = 10 ^ 9
	eq(Logic.seasonTier(s, season), #season.FreeTrack, "tier capped")
end)

test("new season wipes old progress", function()
	local s = Logic.reconcile(nil)
	Logic.addSeasonXP(s, 5000, Config.Season, T0)
	local s2 = table.clone(Config.Season)
	s2.Id = "S2"
	Logic.syncSeason(s, s2)
	eq(s.Season.XP, 0)
end)

test("login bookkeeping and comeback", function()
	local s = Logic.reconcile(nil)
	local info = Logic.onLogin(s, day(0), T0, Config)
	eq(info.IsNew, true)
	eq(Logic.onLogin(s, day(1), T0 + DAY, Config).IsComeback, false)
	local back = Logic.onLogin(s, day(5), T0 + 5 * DAY, Config)
	eq(back.DaysAway, 4)
	eq(back.IsComeback, true)
end)

test("boosts stack duration, keep stronger, expire", function()
	local s = Logic.reconcile(nil)
	Logic.addBoost(s, { Id = "Luck", Multiplier = 2, Duration = 60 }, 0)
	Logic.addBoost(s, { Id = "Luck", Multiplier = 2, Duration = 60 }, 10)
	eq(s.Boosts.Luck.ExpiresAt, 120)
	Logic.addBoost(s, { Id = "Luck", Multiplier = 1.5, Duration = 600 }, 20)
	eq(Logic.boostMultiplier(s, "Luck", 100), 2, "stronger kept")
	eq(Logic.boostMultiplier(s, "Luck", 121), 1, "expired")
	Logic.pruneBoosts(s, 121)
	eq(s.Boosts.Luck, nil)
end)

test("friend bonus capped, offline earnings capped", function()
	eq(Logic.friendBonus(2, Config), 1.2)
	eq(Logic.friendBonus(20, Config), 1.5)
	local cfg = table.clone(Config)
	cfg.OFFLINE_EARNINGS_ENABLED = true
	eq(Logic.offlineEarnings(1000, 1000 + 100 * 3600, 1, cfg), 8 * 3600 * 0.5)
	eq(Logic.offlineEarnings(0, 5000, 1, cfg), 0, "first session")
end)

test("reset countdown and Monday week rollover", function()
	eq(Logic.secondsUntilReset(0, 0), 86400)
	eq(Logic.secondsUntilReset(86400 - 5, 0), 5)
	-- 1970-01-04 was a Sunday (day 3), 1970-01-05 a Monday (day 4)
	eq(Logic.weekIndex(3) ~= Logic.weekIndex(4), true)
	eq(Logic.weekIndex(4), Logic.weekIndex(10))
end)

print(`{passed} passed, {failed} failed`)
if failed > 0 then
	error("spec failed")
end
```

Manual play-test checklist (Studio, 2-player local server):

- [ ] Fresh profile: daily panel auto-opens on join, claim gives Day 1, button turns into a countdown.
- [ ] 1-minute gift unlocks, pulses, and claims once. Spamming the remote does not double-grant.
- [ ] Complete a quest → toast → claim → season bar moves.
- [ ] Set `Data.Retention.Daily.LastClaimDay -= 1` in the command bar, rejoin → can claim Day 2.
- [ ] Set it `-= 5` with 0 freezes → streak falls back to the week checkpoint, never 0.
- [ ] Two friend accounts in one server → `FriendBonus` 1.1 in the snapshot.
- [ ] Leave and rejoin within the same day → gift ladder progress kept, quests identical.

---

## 12. Sharp Edges

| Severity | Issue | Fix (already in this code) |
|---|---|---|
| CRITICAL | Trusting client time or reward amounts | Server `os.time()` only. Remotes accept slot indices, validated as integers in range. |
| CRITICAL | Double-claim across servers | Claims are stored in the session-locked profile (ProfileStore/DataManager). Never in a separate DataStore key. |
| HIGH | Sparse integer keys in saved tables (`Claimed[5] = true`) | Use string keys (`Claimed["5"]`). Arrays must be dense. |
| HIGH | Players online across the daily reset never see new quests | The service tick detects day rollover, refreshes quests and fires `NewDay`. |
| HIGH | `onPlayerRemoving` after `unloadPlayer` loses `LastLogout` | Retention cleanup runs **before** the save (see §8.1). |
| MEDIUM | Removing a quest Id from the pool mid-day | `findDef` returns nil, so the quest is hidden and unclaimable, and it is replaced at the next reset. |
| MEDIUM | Gift ladder farmed by rejoining | The ladder is per **day**, not per session. |
| MEDIUM | Referral farming with alt accounts | Rewards only for brand-new profiles, never self-referral, capped per inviter. |
| LOW | Reset at 00:00 UTC lands mid-afternoon for some regions | Set `RESET_OFFSET_SECONDS` to your audience's quiet hours. Changing it later shifts the day index, so set it before launch. |

---

## 13. Retention Checklist

- [ ] Daily streak with freezes and week checkpoints. The calendar is visible before claiming.
- [ ] At least 3 daily quests + weekly quests, linked by the sweep → weekly quest.
- [ ] Playtime gifts front-loaded (first gift ≤ 1 minute).
- [ ] Season pass with something on every free tier. Premium is retroactive and optional.
- [ ] Comeback reward for players away for 3+ days.
- [ ] Friend-in-server bonus + invite button with referral reward.
- [ ] Notification opt-in after the first daily claim.
- [ ] Onboarding funnel + retention events logged. D1/D7 targets written down.
- [ ] Season XP math computed for this season (§8.4).
- [ ] The engagement line respected (§2): no paid streak repair, odds disclosed, `PolicyService` checked.
