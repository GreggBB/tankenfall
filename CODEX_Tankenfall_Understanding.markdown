# CODEX Understanding: Battlefield 6 Portal Scripting

Perspective primer for an assisting AI. Covers how Portal scripts behave, asset integration, and development guardrails relevant to Tankenfall.

## Runtime Model
- Scripts are single-file TypeScript modules using the `mod.` namespace; no external imports or modules.
- Portal invokes exported handlers (e.g. `OnGameModeStarted`, `OnPlayerDeployed`, `OnPlayerDied`); signatures are strict about async/void.
- Global scope persists for match lifetime—define state (scores, cooldown maps, config constants) at top level.
- Asynchronous waits require `await mod.Wait(seconds)`; there is no `setTimeout`. Never block without yields or the sandbox stalls.
- Objects are opaque handles; compare via `mod.GetObjId(object)` if identity is needed.

## Player State & Interaction
- Use `mod.AllPlayers()` to fetch arrays, then iterate with `mod.CountOf`/`mod.ValueInArray`. No `for...of`.
- Detect AI vs human through `mod.GetSoldierState(player, mod.SoldierStateBool.IsAISoldier)` (true = AI).
- Movement and survivability adjustments rely on soldier states:
  - Speed: `mod.SetPlayerMovementSpeedMultiplier(player, 1.3)`.
  - Health/immortality: `mod.SetSoldierState(player, mod.SoldierStateNumber.MaxHealth, value)` etc.
  - Position/rotation: `mod.Teleport(player, positionVec, facingRadians)`.
- Maintain per-player data with ObjId-keyed Maps to track cooldowns, score contribution, ability states, and UI handles.
- Messaging uses `mod.Message` + Display functions; follow helper pattern from example mods to avoid format errors.

## Teams, Score & Objectives
- Team handles via `mod.GetTeam(player)` or `mod.GetTeam(teamNumber)`.
- Update scoreboard with `mod.SetGameModeScore(team, score)`. Optionally call `mod.DisplayNotificationMessage` for feedback.
- Capture points: `mod.GetCapturePoint(id)`, `mod.EnableGameModeObjective(capturePoint, true)`, `mod.SetCapturePointOwner`, `mod.SetCapturePointCaptureProgress`.
- HQ/Spawn control: `mod.GetHQ(id)` and `mod.EnableHQ(hq, bool)` for staging phases; remove/disable spatial spawn objects by referencing ObjIds from the spatial JSON.
- Exfil or custom win checks should set `mod.SetGameModeEnded(team)` once criteria met; run countdown logic inside async loops with `mod.Wait`.

## AI, Spawners & Waves
- Spawners fetched through `mod.GetSpawner(spawnerId)`; spawn via overloads of `mod.SpawnAIFromAISpawner`.
- Apply class/team parameters using `mod.SoldierClass` enums and `mod.Team`.
- Control density: track active AI count, toggle spawners, or despawn extras using `mod.KillSoldier(aiPlayer)`.
- Behavior tuning uses soldier states and teleporting; maintain safe spawn altitudes to prevent suicide—teleport to ground, open parachutes with `mod.SetSoldierState(..., mod.SoldierStateBool.ShouldDeployParachute, true)` if available.
- Escalate waves by time-slicing spawner activation with `await mod.Wait(interval)` loops.

## Vehicles & Tanken Drops
- Vehicle spawners: `mod.GetVehicleSpawner(id)` plus `mod.ForceVehicleSpawnerSpawn`.
- For runtime drops, combine `mod.SpawnObject(mod.RuntimeSpawn_Common.FX_Impact_SupplyDrop_Brick, pos, rot, scale)` with `mod.SpawnObject` for SFX and potential supply pods.
- Track active tankens per team; if script uses manual vehicle spawning, hold onto returned handle and monitor destruction via `OnVehicleDestroyed`.
- Provide HUD/world icon updates using `mod.GetWorldIcon`, `mod.SetWorldIconText/Position`, and assign owners so the minimap reflects vehicle location.

## UI & Communication
- Quick feedback: `mod.DisplayNotificationMessage`, `mod.DisplayHighlightedWorldLogMessage`, `mod.DisplayWorldLogMessage`.
- Persistent HUD: build ParseUI widget trees (see Example Mods `Exfil.ts` for JSON-driven layout `mod.CreateWidgetFromJson`); store widget references to update text, progress bars, or timers.
- World icons: enable with `mod.EnableWorldIconImage/Text`, set colors via `mod.SetWorldIconColor`, update text with `mod.SetWorldIconText`.
- Countdown sequences: Use `mod.ShowWorldLogTimer(team/player, seconds, message)` or custom widget loops.
- Optional class members (`this.root?: mod.UIWidget`) require non-null assertions (`this.root!`) even after guard clauses; TypeScript’s control flow does not track class property narrowing.

## VFX/SFX Layering
- Runtime enums (`mod.RuntimeSpawn_Common.*`, `mod.RuntimeSpawn_Abbasid.*`) provide deployable effects; spawn via `mod.SpawnObject`.
- Manage lifetime: keep handles to VFX/SFX and disable/`mod.UnspawnObject` once finished to avoid over-budget.
- Pair drop pods with `FX_Impact_SupplyDrop_Brick`, battlefield ambience with `SFX_Aircraft_Flyby`, etc., as catalogued in context materials.

## Spatial Scene Integration
- `MP_Abbasid_TEST.spatial.json` controls placed actors (HQs, Capture Points, Area Triggers, VFX positions). Reference ObjIds in script; avoid hardcoding guessed numbers.
- Remove or toggle spatial objects by `mod.SetSpatialEntityEnabled(obj, bool)` or specialized functions depending on type (e.g., `mod.EnableHQ`).
- Ensure script modifications match actual object names/IDs; maintain updated mapping doc if objects move.

## Debugging & Workflow
- Build iteratively: after each significant change run compiler to catch signature typos or missing `mod.` prefixes.
- Console logging available via `console.log` (string only) for debugging; remove or guard behind debug flags for release build.
- Validate function names against `index.d.ts`. If TypeScript error references missing method, search file to confirm accurate signature.
- Pay attention to async vs sync mismatches—Portal rejects wrong return types.
- Performance watchpoints: avoid per-frame heavy loops, throttle updates, reuse arrays/objects when possible.

## Common Failure Modes
- Mistyped function names (`DisplayCustomNotification` vs `DisplayNotificationMessage`) cause compile failures—always double-check definitions.
- Forgetting `export` on event functions -> handlers never called.
- Using `for...of` or array methods like `.forEach` on `mod.Array`—must convert or use index iteration.
- UI widget references going out of scope; retain handles globally to update/destroy.
- Not resetting state between rounds or after vehicle destruction; ensure resets in `OnPlayerDeployed`, `OnVehicleDestroyed`, `OnGameModeStarted`.
- Leaving runtime VFX running continuously => performance degradation; disable after use.
- Infinite service loops (`while (true)`) without termination guards continue running post-match; gate with `matchActive` flags and stop once `mod.SetGameModeEnded` triggers.

## Recommended Patterns
- Centralize configuration constants (scores, cooldowns, spawn timings) at head of file.
- Use helper functions for repeated tasks (score updates, iterating players, awarding Tanken progress).
- Wrap message creation and scoreboard updates into dedicated utilities to keep event handlers concise.
- Keep AI + vehicle managers in dedicated async loops invoked from `OnGameModeStarted` so logic is orchestrated predictably.

Use this understanding alongside `CODEX_Tankenfall_Knowledge.markdown` (high-level goals) and `CODEX_Tankenfall_Code_database.markdown` (API specifics) to maintain coherence across design, implementation, and debugging.
