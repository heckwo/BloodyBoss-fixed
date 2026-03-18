# BloodyBoss-fixed
A test update for BloodyBoss fixes.

# BloodyBoss — Community Bug Fix Release

A comprehensive bug fix pass on the BloodyBoss mod addressing all known spawn, despawn, and persistence issues reported by the community. This release does not add new features — it fixes the underlying systems so the existing ones work correctly and reliably.

---

## What Was Fixed

### 🔴 Critical — Boss Becomes Invincible / Stuck In World

**Root cause:** `KillBoss()` was destroying the wrong entity. The first block was intended to destroy the map icon but was accidentally destroying `bossEntity` instead of `iconEntity`. If `GetBossEntity()` also failed, the boss entity was never destroyed at all — leaving it in the world with its BloodyBoss buff (which grants invincibility), while `bossSpawn` was set to false so the mod thought it was gone.

**Fix:** The first block now correctly targets `iconEntity`. The icon is destroyed first, then the boss entity.

---

### 🔴 Critical — Boss Stops Spawning / bossSpawn Flips to False in Config

**Root cause:** Two separate bugs combined to cause this.

**Bug A — Premature state set:** `bossSpawn = true` was being set synchronously in `CheckSpawnDespawn()` immediately after calling `Spawn()`. But `SpawnUnitWithCallback` is asynchronous — the entity doesn't exist yet when that line runs. If the spawn callback ever threw an exception or failed silently, `bossSpawn` was permanently stuck at `true` with no actual entity in the world. The spawn guard `if (!bossSpawn)` then blocked every future scheduled spawn attempt. On the next server restart, `CleanupBossStatesOnStartup` found no entity and correctly reset it to `false` — which is the "flipping to false" users were seeing. The false was the correction; the stuck true was the real problem.

**Fix:** Removed `bossSpawn = true` and `saveDatabase()` from `CheckSpawnDespawn()` entirely. `ModifyBoss()` inside the confirmed-success spawn callback is now the sole authority for setting `bossSpawn = true`.

**Bug B — Server crash from background thread:** `System.Threading.Timer` fires its callback on a .NET thread pool background thread. The timer was calling `bossAction` directly on that thread. The very first line of `bossAction` — checking `Plugin.SystemsCore == null` — calls `World.GetExistingSystemManaged()`, which is an IL2CPP ECS call. Accessing IL2CPP ECS from any thread other than the main game thread causes an `AccessViolationException` in native unmanaged memory. This crashes the entire server process below BepInEx's catch handlers, writing nothing to the BepInEx log. The server auto-restarts, the timer resets, and the cycle repeats — meaning the timer never held long enough to fire a scheduled spawn.

**Fix:** The timer callback now only queues work via `ActionScheduler.RunActionOnMainThread()`. The timer itself never touches ECS. All boss logic runs safely on the main game thread.

---

### 🔴 Critical — Wrong Players Get Loot / No Loot on Kill

**Root cause:** `vbloodKills` (the killer tracking list) was declared `static` on `BossEncounterModel`. This means every boss instance in the game shared a single list. When Boss A was killed, `SendAnnouncementMessage()` called `RemoveKillers()` which cleared the list for every boss simultaneously. If Boss B was killed shortly after, its killer list was empty and no rewards were given. Players who killed Boss A also appeared in Boss B's killer list, receiving rewards they didn't earn.

**Fix:** Removed the `static` keyword. Each `BossEncounterModel` instance now has its own independent `vbloodKills` list.

---

### 🔴 Critical — Admin Despawn Leaves Ghost Tracking Entries

**Root cause:** `DespawnBoss()` called `KillBoss()` which destroyed the entity, then tried to unregister it with `if (GetBossEntity()) { BossGameplayEventSystem.UnregisterBoss(...) }`. Since the entity was just destroyed, `GetBossEntity()` always returned false — the unregister was never reached. `BossTrackingSystem.UnregisterBoss()` was never called at all. Both tracking systems retained stale entries indefinitely.

**Fix:** The entity reference is captured before destruction. After `KillBoss()` runs, both `BossGameplayEventSystem.UnregisterBoss()` and `BossTrackingSystem.UnregisterBoss()` are called using the captured reference. `bossEntity` is also explicitly set to `Entity.Null` after despawn.

---

### 🔴 Critical — False Positives on Nearby Entities

**Root cause:** `IsBloodyBoss()` in `BossEntityExtensions` used `.Contains("bb")` to identify boss entities. Any entity whose vanilla name contains the substring "bb" anywhere would be misidentified as a BloodyBoss, causing the discovery scanner to attempt to process random NPCs every second.

**Fix:** Changed to `.EndsWith("bb")` to match only the intentional `nameHash + "bb"` suffix applied during spawn.

---

### 🔴 Critical — HP Threshold Mechanics Fire at Wrong Time

**Root cause:** `HealthMonitorSystem` — the only system that correctly tracks previous HP for threshold crossing detection — was never called anywhere in the codebase. The fallback path in `ProcessStatChangeEvent` always passed `100f` as `previousHpPercent`, meaning the threshold crossing check (`previousHp >= threshold && currentHp < threshold`) would evaluate true on the very first damage event for any threshold below 100%, causing mechanics to fire immediately on the first hit rather than at the configured percentage.

Additionally, on spawn the HealthMonitorSystem saw the boss entity before `ModifyBoss` applied stats, reading health as `0%`. When stats were applied it jumped to `100%`, triggering a false health-change event and potentially firing all sub-100% mechanics on spawn.

**Fix:** `HealthMonitorSystem.Update()` is now wired into the main timer loop. A `GetLastHealthPercent()` accessor was added to `BossTrackingSystem` so `ProcessStatChangeEvent` can supply correct previous HP values. `RegisterSpawnedBoss` now seeds `LastHealthPercent` at `100f`. The health monitor now only processes mechanics and logs warnings when HP goes **down** — increases are spawn/heal events and are ignored.

---

### 🟡 Medium — Boss Spawns with Vanilla Stats After Server Restart

**Root cause:** `CleanupBossStatesOnStartup()` used `GameData.Users.Online.FirstOrDefault()` to get a user entity for calling `ModifyBoss()`. On automated server restarts no player is connected yet, so this returned null and `ModifyBoss` was skipped entirely. The boss entity survived the restart registered for tracking but with no health multipliers, stat scaling, team assignment, or mechanics applied.

**Fix:** Changed both startup reconfiguration blocks to use `GameData.Users.All.FirstOrDefault()`. The `Users.All` collection always contains valid user entities for passing to game systems regardless of whether any player is currently online.

---

### 🟡 Medium — Boss Lifetime Confusion / Instant Despawn

**Root cause:** `Lifetime` is stored and passed in **seconds**, but the spawn announcement message divided it by 60 and showed only the result — with no unit label. A boss configured with `Lifetime = 120` would show "2" in chat and live for 2 minutes, not 2 hours. If `Lifetime` was 0 (unset or hand-edited JSON default), `SpawnUnitWithCallback` destroyed the entity on the same frame it was created, causing the "spawns and despawns instantly" report.

**Fix:** Added a `Lifetime <= 0` guard in both `Spawn()` overloads that aborts and logs a clear error before attempting to spawn. The spawn announcement now shows both units: `2 minutes (120s)`.

---

### 🟡 Medium — BossEntityFactory Lifetime Mismatch

**Root cause:** `BossEntityFactory.CreateBoss()` was using `model.Lifetime + 30` as the spawn duration with no explanation. This made boss entities live 30 seconds longer than configured, and mismatched the map icon lifetime (`Lifetime + 5`), meaning the icon would disappear before the boss entity expired.

**Fix:** Changed to `model.Lifetime` directly.

---

### 🟡 Medium — Memory Leak on Long-Running Servers

**Root cause:** `BossComponentsHelper._bossStateCache` is a dictionary keyed by entity that caches boss state structs. Nothing in the despawn, death, or unregister paths called `RemoveBossState()`. On servers running for extended periods with repeated boss spawns, this dictionary grew indefinitely and held references to destroyed entities.

**Fix:** `BossTrackingSystem.UnregisterBoss()` now calls `BossComponentsHelper.RemoveBossState()` as part of the standard cleanup path.

---

### 🟡 Medium — DamageCorrelationSystem Overhead With No Benefit

**Root cause:** `ProcessCorrelatedDamage()` logged an Info-level message implying damage was being processed, then did nothing. The correlation layer was never completed — both damage hooks still process independently. The dictionary lookups and cleanup timers ran on every single damage event server-wide with zero benefit.

**Fix:** Replaced the misleading log with a `Trace`-level message and a detailed comment explaining the incomplete state and exactly how to complete the implementation in a future pass.

---

### 🟡 Low — Admin Despawn Debug Info Was Useless

**Root cause:** `FilterKillersByDistance()` replaced `vbloodKills` with the filtered list, then logged `"X valid players out of Y total"` — but both X and Y called `GetKillers()` which returned the already-replaced list, making both numbers always identical.

**Fix:** Capture `originalCount` before replacing the list.

---

## New Commands

```
.bb schedule time
```
Shows the current server time in the exact `HH:mm` format used by `.bb schedule set`. Eliminates timezone/format confusion when scheduling bosses.

---

## Files Changed

| File | Changes |
|------|---------|
| `DB/Models/BossEncounterModel.cs` | KillBoss fix, vbloodKills instance field, DespawnBoss unregister fix, CheckSpawnDespawn async fix, Lifetime guard, DateTime.Parse → TryParse (4 locations), FilterKillersByDistance log fix |
| `Systems/BossSystem.cs` | Thread safety fix (RunActionOnMainThread), HealthMonitorSystem wired in, minute tick log |
| `Systems/BossTrackingSystem.cs` | GetLastHealthPercent() accessor, RemoveBossState() on unregister, UpdateActiveBossesOnMainThread(), RegisterSpawnedBoss seeds 100f |
| `Systems/BossGameplayEventSystem.cs` | Pulls previousHpPercent from BossTrackingSystem instead of hardcoding 100f |
| `Systems/DamageCorrelationSystem.cs` | Replaced misleading log with honest TODO comment |
| `Systems/HealthMonitorSystem.cs` | HP increase direction filter — only processes mechanics on decrease |
| `Utils/BossEntityExtensions.cs` | Contains("bb") → EndsWith("bb") |
| `Factory/BossEntityFactory.cs` | Lifetime+30 removed, dev-test path comment added |
| `Command/BossCommand.cs` | Spawn() bool return checked, catch block logs full stack trace |
| `Command/BossScheduleCommand.cs` | `.bb schedule time` command added |
| `Plugin.cs` | Users.Online → Users.All in both startup reconfiguration blocks |

---

## Building From Source

Requires .NET 6 SDK or newer.

```
dotnet build -c Release
```

Output: `bin/Release/net6.0/BloodyBoss.dll`

Replace the existing `BloodyBoss.dll` in your `BepInEx/plugins/BloodyBoss/` folder and restart the server.

---

## Credits

Original mod by the BloodyBoss development team. This fix release compiled and verified by the V Rising modding community.
