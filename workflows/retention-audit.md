# Retention Audit Workflow

A guided workflow for finding out why players don't come back, and fixing it. It scores the game's "reasons to return" at every time scale, ranks the gaps by impact, and implements fixes from `references/retention-systems.md`.

Trigger phrases: "players don't come back", "low retention", "D1 is bad", "make it addictive", "keep players playing", "increase playtime", "engagement".

> **Framing:** when a user asks to make a game "addictive", build *compelling* loops: clear goals, fair rewards, social pull, visible progress. Do not build manipulative ones. Most Roblox players are children. Dark patterns break Roblox rules in several cases and lose on D30 anyway. Section 2 of `references/retention-systems.md` lists where the line is. Apply it without lecturing the user.

---

## Step 1: Get the Numbers (If They Exist)

Ask for, or read from Creator Hub → Analytics if the user can share them:

- D1 / D7 / D30 retention
- Average session length and sessions per user per day
- Onboarding funnel (if instrumented)
- Where in the map/progression players are when they leave (if logged)

If the game is unreleased or has no data, skip to Step 2 and add instrumentation as part of the fix (Step 5).

Read the numbers this way:

| Symptom | Most likely cause | Look at first |
|---|---|---|
| D1 < 15% | First session confusing or slow to reward | FTUE, first-minute reward, onboarding funnel |
| D1 okay, D7 < 5% | Nothing new after day 1 | Daily streak, daily quests, content depth |
| D7 okay, D30 < 2% | No long-term goal | Season pass, collection book, prestige, events |
| Short sessions (< 8 min) | Core loop runs out or no "one more" hook | Playtime gifts, quest pacing, upgrade curve |
| High churn at a specific level/stage | Difficulty spike or grind wall | Progression curve (`references/game-design-roblox.md` §6) |

---

## Step 2: Scan the Project

### MCP Full Mode

Use `grep_scripts` / `get_file_tree` to search for evidence of each lever:

- `os.time`, `DateTime`, `LastLogin`, `Streak`, `Daily`: daily reward systems
- `Quest`, `Mission`, `Challenge`, `Task`: quest systems
- `Season`, `Pass`, `Tier`, `BattlePass`: long-term tracks
- `IsFriendsWith`, `PromptGameInvite`, `GetJoinData`, `LaunchData`: social hooks
- `ExperienceNotificationService`: notifications
- `AnalyticsService`, `LogOnboardingFunnelStepEvent`, `LogCustomEvent`: instrumentation
- `PolicyService`, `ArePaidRandomItemsRestricted`: compliance of paid random items

Also check every hit for the **sharp edges** in §12 of the reference, especially client-supplied times, claims stored outside the session-locked profile, and sparse integer keys in saved data.

### MCP Standard Mode

Use `run_code` to list scripts and search their `Source` for the same keywords:

```luau
local hits = {}
for _, inst in game:GetDescendants() do
	if inst:IsA("LuaSourceContainer") then
		local ok, source = pcall(function() return (inst :: any).Source end)
		if ok and source and (source:find("Streak") or source:find("Quest") or source:find("Season")) then
			table.insert(hits, inst:GetFullName())
		end
	end
end
print(table.concat(hits, "\n"))
```

### Offline Mode

Ask the user to describe: what a player does in their first 5 minutes; what is different when they come back tomorrow; what goal they are working toward in a month; whether friends get anything for playing together.

---

## Step 3: Score the Return Stack

Score each lever 0-3: **0** missing, **1** exists but weak or buggy, **2** solid, **3** excellent and tuned from data.

| # | Lever | Weight | What a 3 looks like |
|---|---|---|---|
| 1 | First-minute reward + clear first goal | ×3 | Reward within 60 s, objective arrow/text, first quest done in 5 min |
| 2 | Core loop "one more" pull | ×3 | Upgrades always within reach, visible next goal, satisfying juice |
| 3 | Daily login streak | ×2 | Visible calendar, scaling rewards, freezes, never resets to zero |
| 4 | Daily quests | ×2 | 3+ varied quests, sweep bonus, same on every server |
| 5 | Playtime gifts | ×1 | Front-loaded ladder with HUD countdown, not farmable |
| 6 | Weekly goals | ×1 | Weekly quests linked to dailies |
| 7 | Long-term goal (season / collection / prestige) | ×2 | Season pass with something on every free tier, a collection book |
| 8 | Social pull | ×2 | Friend bonus, invite rewards, co-op goals, trading |
| 9 | Comeback hook | ×1 | Welcome-back bundle for lapsed players |
| 10 | Notifications | ×1 | Opt-in at a meaningful moment, high-quality triggers |
| 11 | Live content cadence | ×2 | Updates every 1-2 weeks, seasonal events (`references/game-design-roblox.md` §8) |
| 12 | Instrumentation | ×1 | Onboarding funnel + retention events + economy events |

**Score = Σ(score × weight) out of 63.**

| Score | Grade | Meaning |
|---|---|---|
| 54-63 | A | Retention engine is complete, so tune from data |
| 42-53 | B | Solid. Fill the 1-2 weakest levers |
| 28-41 | C | Players have little reason to return after day 1 |
| < 28 | D | Retention is accidental. Ship the full retention layer first |

**Engagement line check (pass/fail, independent of score):** paid streak repair, fake timers/scarcity, guilt or loss messaging on leave, undisclosed odds on paid random items, or a missing `PolicyService` gate are each an automatic **fail**. Fix these before anything else.

---

## Step 4: Prioritized Plan

Present the scorecard, then a plan ordered by `weight × (3 − score)`, highest first. Ties go to the earlier time scale (fix D1 before D30). Example:

```
Retention Audit: Pet Hatchers — Grade C (35/63), Engagement line: PASS

Top fixes:
1. First-minute reward (0 → 3, ×3): add 1-min playtime gift + starter quest  [retention-systems §6.3]
2. Daily quests (0 → 3, ×2): add RetentionService, simulator quest pool        [§6, §8.2]
3. Social pull (1 → 3, ×2): friend bonus on coin payouts + invite button       [§6.6]
4. Long-term goal (1 → 2, ×2): Season 1 with 30 tiers, XP math from §8.4
5. Instrumentation (0 → 2, ×1): onboarding funnel, 5 steps                    [§10]

Expected impact: D1 +5-10 pts from (1); D7 roughly 2x from (2)+(3).
```

Ask the user which fixes to implement (default: all, in order).

---

## Step 5: Implement

For each approved fix:

1. If the retention layer is missing, install all four modules from `references/retention-systems.md` and wire `GameManager` (§8.1). The scaffold already supports it.
2. Replace the default quest pools with **genre-specific** quests that use the game's real verbs, and add `RetentionService.track(...)` at those call sites (§8.2).
3. Register reward handlers for the game's currencies and items (`setRewardHandler`).
4. Apply `RetentionService.getMultiplier(player, kind)` to the core payout so boosts and the friend bonus feel real.
5. Recompute season XP per tier with §8.4 and set the season dates.
6. Add the onboarding funnel steps (§10) along the real first-session path.

Follow the MCP mode conventions from `workflows/new-game.md` Step 6 (create scripts via MCP in Full/Standard mode, copy-paste blocks with placement paths in Offline mode).

---

## Step 6: Verify

- Run the `RetentionLogic` spec (§11). It must report 0 failures.
- Walk through the manual play-test checklist in §11 with a fresh profile and a returning profile.
- Spam every `Retention_*` remote from the command bar with bad arguments (strings, negatives, huge indices, NaN). Nothing must be granted. See `workflows/security-audit.md`.
- After launch, re-run Step 1 after 7 days and compare against the targets set in Step 4.

---

## Step 7: Summary

Close with:

- Scorecard before → after (projected).
- Files added/changed.
- What to watch in analytics over the next 7 days (D1, the onboarding funnel's biggest drop, quest completion rates per quest).
- Next lever to pull once the data is in (usually: content cadence and the first seasonal event).
