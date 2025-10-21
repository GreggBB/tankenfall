# CODEX Code Database: Tankenfall

Reference checklist for BF6 Portal scripting from `Context files and code library`, `index.d.ts`, `index-modlib.ts`, and working examples.

## Lifecycle & Game Flow
- Entry points must match SDK signatures (async vs sync):
  - `export async function OnGameModeStarted()`
  - `export async function OnPlayerJoinGame(player: mod.Player)`
  - `export async function OnPlayerLeaveGame(playerId: number)`
  - `export function OnPlayerDeployed(eventPlayer: mod.Player)`
  - `export function OnPlayerDied(eventPlayer: mod.Player, eventOtherPlayer: mod.Player, eventDeathType: mod.DeathType, eventWeaponUnlock: mod.WeaponUnlock)`
  - `export function OnPlayerEarnedKill(...)`, `OnPlayerDamaged(...)`, `OnPlayerEnterAreaTrigger(...)`, etc.
- Use `OngoingGlobal()` for heartbeat loops (runs every frame) only when absolutely required; throttle with `mod.Wait`.
- `mod.Wait(seconds)` is the only async delay; never block loops without `await`.

## Player & Team Utilities
- Retrieve collections via `mod.AllPlayers()` and iterate with `mod.CountOf(array)` + `mod.ValueInArray(array, index)`.
- Identify players/teams through ObjIds: `mod.GetObjId(player)` / `mod.GetObjId(team)`; store in Maps keyed by number for meters/cooldowns.
- Team helpers:
  - `mod.GetTeam(player: mod.Player): mod.Team`
  - `mod.GetTeam(teamNumber: number): mod.Team`
  - `mod.SetGameModeScore(team, score)` and `mod.SetGameModeScore(player, score)` to sync scoreboard.
  - `mod.DisplayHighlightedWorldLogMessage` / `mod.DisplayNotificationMessage` for announcements (use helper `MakeMessage` from examples to manage format args).
- Movement and states:
  - `mod.SetPlayerMovementSpeedMultiplier(player, multiplier)`
  - `mod.SetSoldierState(player, mod.SoldierStateBool.IsImmortal, true)` etc. (check `SoldierStateBool/Number/Vector` enums lines 11225+).
  - `mod.Teleport(player, positionVector, facingRadians)` for phase shift/vertical reposition.
  - Access states with `mod.GetSoldierState(player, mod.SoldierStateBool.IsAISoldier)` (true => AI), `mod.SoldierStateNumber.CurrentHealth`, `mod.SoldierStateVector.GetPosition`, etc.
- Ability input detection works by pairing slot activity with firing state:
  - `const slotActive = mod.IsInventorySlotActive(player, mod.InventorySlots.ClassGadget);`
  - `const isFiring = mod.GetSoldierState(player, mod.SoldierStateBool.IsFiring);`
  - Trigger ability when both true and a per-player `abilityInputHeld` flag is false, then set the flag to debounce.

## Score, Progression & Messaging
- Scoring pattern from brief: use constants for kill values and objective ticks; update arrays `[team1Score, team2Score]`.
- Personal Tanken meter: Map `playerId -> progress`, increment on events, reset on spawn, clamp to threshold (e.g. 50).
- Broadcast + UI:
  - `mod.Message("text")` plus helper for placeholders.
  - `mod.DisplayNotificationMessage(message)` global, or pass player/team for targeted.
  - `mod.DisplayHighlightedWorldLogMessage` for center-left log.
  - `mod.ShowWorldIcon` replacement is handled by `mod.EnableWorldIconImage/Text`, `mod.SetWorldIconText`, `mod.SetWorldIconPosition`, `mod.SetWorldIconOwner` after retrieving via `mod.GetWorldIcon(id)`.
  - Custom HUD uses ParseUI widget tree; mirror structure in example mods, ensure `Widget` alias = `mod.UIWidget`.

## Spawners, AI & Waves
- Acquire spawners by ID: `const spawner = mod.GetSpawner(spawnerId);`
- Spawn AI:
  - `mod.SpawnAIFromAISpawner(spawner)` or overload with `SoldierClass`, `Message`, `Team` (see lines 11946-11968).
  - Manage availability with `mod.SpawnAIWave(spawner, count)` (verify exact signature in `index.d.ts` if needed).
- Control HQs & capture points:
  - `mod.GetHQ(id)`, `mod.EnableHQ(hq, bool)`.
  - `mod.GetCapturePoint(id)`, `mod.EnableGameModeObjective(capturePoint, true)`, `mod.SetCapturePointOwner(capturePoint, crewTeam)`.
- Remove high-elevation spawn platforms (ObjIds 801-818) after `mod.Wait(60)` using `mod.DisableSpatialEntity` or `mod.HideSpatialObject` (check map JSON for actual object type).
- AI aggression: use `mod.SetAIBehaviorProfile(player, profileEnum)` if available, or repeatedly teleport to safe zones, set aggressiveness via `mod.SetAIModeChaseTarget`.

## Vehicle / “Tanken” Control
- Vehicle spawners: `const vehicleSpawner = mod.GetVehicleSpawner(id);` then `mod.ForceVehicleSpawnerSpawn(vehicleSpawner);` or `mod.SetVehicleSpawnerAutoSpawn`.
- For runtime drops use `mod.SpawnObject(mod.RuntimeSpawn_Common.VehicleDropPod, position, rotationVec, scaleVec)` followed by `mod.SpawnVehicle` if direct API exists (double-check 13240+ for accurate signature).
- Use `mod.GetVehicleDefinition(unlockId)` to pick vehicle type; pair with `mod.AssignPlayerToVehicle`.
- Add call-in VFX/SFX:
  - `mod.SpawnObject(mod.RuntimeSpawn_Common.FX_Impact_SupplyDrop_Brick, position, ZERO_VEC)` for drop impact.
  - Enable/disable via `mod.EnableVFX(vfx, boolean)` and clean with `mod.UnspawnObject`.

## World Interaction & Objectives
- Interact points: `mod.GetInteractPoint(id)`, gate abilities by comparing `mod.GetObjId(point)`.
- Area triggers: `mod.GetAreaTrigger(id)` plus event hooks `OnPlayerEnterAreaTrigger`/`OnPlayerExitAreaTrigger`.
- Capture logic support: `mod.SetCapturePointCaptureProgress(capturePoint, value)`, `mod.SetCapturePointEnabled(capturePoint, bool)`.
- Exfil sequence: spawn interact/HQ, show world icon, start timers through `mod.ShowTimerView(team/player, seconds, message)`.

## VFX / SFX Library Notes
- `RuntimeSpawn_Common` enum (line ~5019) includes reusable assets:
  - `FX_Airstrike`, `FX_ArtilleryStrike_Explosion_GS`, `FX_Granite_Strike_Smoke_Marker_Yellow`, `FX_Smoke_Marker_Custom`, `FX_Impact_SupplyDrop_Brick`.
  - `SFX_Gadgets_C4_Activate_OneShot3D`, `SFX_Aircraft_Flyby`.
- Use `mod.SpawnObject` with `mod.RuntimeSpawn_Abbasid` for map-specific set dressing (check enum around 5060+).
- Audio: `mod.SpawnObject(mod.RuntimeSpawn_Common.SFX_*...)` returns `mod.SFX`; enable via `mod.EnableSFX`.
- Remember to `mod.UnspawnObject` once effect finishes to prevent leaks.
- Capture/intro loops: `SFX_UI_Gamemode_Shared_CaptureObjectives_CaptureStartedByFriendly_OneShot2D`, `...CaptureStartedByEnemy_OneShot2D`, `...OnCapturedByFriendly_OneShot2D`, `...LeadChange_Positive_OneShot2D`, intro countdown cues (`...Intro_Countdown_OneShot2D`, `...Intro_FinalImpact_OneShot2D`).
- VO support requires spawning a VO object then invoking `mod.PlayVO(voHandle, mod.VoiceOverEvents2D.ObjectiveCaptured)`; enums like `VehicleTankSpawn`, `ObjectiveNeutralised`, `ProgressLateWinning` are available. Ignore references to `mod.PlayVOEvent`/`mod.VOEvents`—those do not exist.

## Utility Helpers from `index-modlib.ts`
- Array utilities: `ConvertArray`, `FilteredArray`, `IndexOfFirstTrue`, `IsTrueForAll/Any`, `SortedArray`.
- Logical helpers: `And`, `AndFn`, `IfThenElse`, `WaitUntil`.
- ObjId shortcuts: `getPlayerId(player)`, `getTeamId(team)`.
- Condition state pattern (`ConditionState`) for detecting rising-edge events, useful for toggling UI or gating cooldown re-entry.
- Fire-and-forget delays: wrap `await mod.Wait(seconds)` inside a helper (e.g. `schedule(delay, action)`) instead of `setTimeout`.

## Example Snippets Worth Reusing
- `MakeMessage(...)` implementation from BombSquad/Exfil for safe message creation.
- Countdown HUD: review `Example Mods/Exfil/Exfil.ts` for `mod.CreateWidgetFromJson` and timer management.
- AI wave loops: BombSquad demonstrates sequential spawner iteration with debug logging (`mod.SpawnAIFromAISpawner(mod.GetSpawner(id), mod.SoldierClass.Assault, team)`).
- World icon orchestration: Exfil uses `mod.EnableWorldIconImage/Text`, `mod.SetWorldIconPosition` to move markers as helicopters relocate.
- Vehicle call-in cinematics: search Exfil for drop pod VFX pairing with `mod.Wait` to stage events.

## Lessons from Old Builds
- Attrition scores were set to 50/100/500/1000 in V5.0; adjust to the brief’s +1/+2/+5/+10 scheme so scoreboard pacing matches expectations.
- Previous Tanken meter treated `VEHICLE_BASE_TIME` as a countdown; rework to accumulate from personal contribution (kills, captures) and reset cleanly on spawn/vehicle destruction.
- Infinite `while (true)` game loops lacked a stop condition; gate loops with `matchActive`/`gameOver` flags and exit once `mod.SetGameModeEnded` fires.
- Exfil logic spawned helicopters but never called `mod.SetGameModeEnded`; remember to finish matches after objectives resolve.
- UI helpers assumed widgets always exist; guard against `null` returns from `mod.CreateWidgetFromJson` and cache handles once created.
- AI respawn queues grew without cap when spawners were disabled; track active counts and clear `pendingAISpawn` entries when players leave to avoid runaway loops.
- Claude drafts referenced `setTimeout()`; Portal scripts cannot use browser timers—use `await mod.Wait(seconds)` exclusively for delays.
- Tank spawner positioning cannot be changed at runtime (`mod.SetVehicleSpawnerPosition` doesn’t exist); rely on pre-placed spawners and cycle them.

## Verification Tips
- Cross-check every `mod.` call against `index.d.ts`; typos won’t compile.
- Validate ObjId constants against `MP_Abbasid_TEST.spatial.json` before use.
- Avoid direct object comparison; always compare IDs or store references.
- Guard async logic with try/catch-like patterns? Portal lacks try/catch; rely on conditional early returns.
- Keep default values centralized to simplify balancing (scores, cooldowns, AI counts).

> Companions: `CODEX_Tankenfall_Knowledge.markdown` (design goals) and `Context files and code library/CODEX_Tankenfall_Understanding.markdown` (Portal fundamentals).
