# CODEX Knowledge: Tankenfall

## Prompt Snapshot
- Build `Tankenfall – Titanfall Prequel`, a Battlefield 6 Portal TypeScript mode blending BF Conquest with Titanfall 2 Attrition on `MP_Abbasid_Test`.
- Support 8v8 Pilots (players + aggressive AI backfill) who feel faster, tougher, and deadlier than vanilla BF; apply a `setPlayerMovementSpeedMultiplier` of `1.3`.
- Preserve Battlefield class silhouettes (Assault, Recon, Engineer, Support) while bolting on Titanfall-style perks/abilities and universal AT launchers; allow any two primary weapon picks.
- Deliver the “Tanken” (tank drop) fantasy: players accumulate personal contribution score, hit a threshold (~50 team score contribution), then call a powerful tank; meter resets after the tank is destroyed.
- Blend PvE and PvP: capture/hold objectives, hunt AI grunts/spec-ops/pilots, destroy vehicles, and drive escalation into an exfil endgame.

## Deliverable Requirements
- One compiling TypeScript script with strict `mod.` namespace usage, matching Portal event signatures.
- One supporting spatial scene `MP_Abbasid_TEST.spatial.json` that places HQ spawns, capture points, interactables, AI spawners, and cinematic intro assets (explosions, dogfights, etc.).
- Titanfall-flavoured pacing: cinematic 20s spawn freeze with HUD countdown, explosive ambience, then fast, vertical combat once unfrozen.
- Score sources per brief: +5 Pilots (player or AI), +2 Spec Ops, +1 Grunts, +10 destroyed Tanken, objective interactions granting sustained team score ticks.
- Deliver class kits & perks aligned to their roles (stim movement, cloak, deployable shield, smart pistol analogues, etc.) following research recommendations.
- Ensure AI Pilots spawn once at HQ, then transition to ground-level spawn network; remove aerial spawn platforms (object IDs 801-818) after 60s.
- Provide HUD feedback: score, Tanken readiness meter, location indicator when a Tanken is active, exfil timer, notifications for kills/objective changes.
- Sustain attrition loop: AI wave escalation, player ability cooldowns, vehicle drops, and eventual exfiltration win condition (first team to hold evac for defined window after threshold score).

## Research Highlights from Brief
- **Hybrid Scoring System:** Attribute point tiers (Grunt 50, Spec Ops 100, Pilots 500, Captures 1000, Vehicle kill 800, Assist 100) but reconcile with core +1/+2/+5/+10 scoreboard to avoid over-rewarding; use helper to map kill context to team score & personal Tanken meter.
- **Vehicle-as-Titan:** Simulate Titanfall burn cards—per-player accumulation tracked in Map keyed by ObjId; spawn tank via `mod.SpawnVehicle` at drop pads with VFX/SFX, mark on HUD, clamp to one active per player/team, enforce cooldown.
- **AI Waves:** Use spawner ObjIds for Grunts and Spec Ops; manage spawn cadence via timers, escalate difficulty, and ensure performance by limiting simultaneous AI counts (deactivate extras with `mod.DisableSpawner` or `mod.KillSoldier` when over budget).
- **Class Abilities:** Implement with interact points + timers: Stim (speed burst), Cloak (set visibility), Phase Shift (teleport + invuln window), Deployable shield, Smart Pistol lock-on; rely on `mod.GetSoldierState` and `mod.SetSoldierState`.
- **Enhanced Movement:** While per-player multipliers limited, combine base multiplier tweak, periodic `mod.Teleport` nudges, grapples via `mod.SpawnGrappleLine` analogs, and zipline objects to approximate Titanfall agility.
- **Objective Network:** Three capture points tied to conquest logic; maintain `mod.EnableGameModeObjective`, `mod.SetCapturePointOwner`, periodic score ticks, contested state messages, and AI priority targeting.
- **Exfil Endgame:** Trigger once team score target reached; spawn evac ship interact point, start countdown HUD, force defending team to recapture or contest to reset timer.
- **Technical Practices:** Follow event-driven architecture, pattern after example mods (BombSquad, Exfil); manage ObjIds intentionally; throttle async loops with `await mod.Wait`.
- **Common Pitfalls:** UI anchor mistakes, forgetting to check `index.d.ts`, failing to guard AI from suicide, mismatching async signatures, leaving unused VFX running, unbalanced cooldowns, contradictory rule conditions.
- **Audio/VFX Layering:** Use layered explosions, flyovers, and color-coded world icons; emphasize drop pods, titan call-in, exfil ship arrival; keep performance manageable by enabling/disabling loops.
- **Claude Insights:** Trigger abilities by monitoring empty Class Gadget slot + `mod.SoldierStateBool.IsFiring`; tank drops rely on pre-placed vehicle spawners (no runtime reposition); available VO events include `ObjectiveCaptured`, `VehicleTankSpawn`, while capture SFX range from `SFX_UI_Gamemode_Shared_CaptureObjectives_OnCapturedByFriendly_OneShot2D` to lead-change stingers; ignore any guidance referencing `setTimeout()`—Portal requires `await mod.Wait`.

## Constraints & Priorities
- Must remain Battlefield 6 Portal compliant: single `.ts` script, no imports, no wallrunning, limited per-player modifiers, rely on provided assets.
- Prioritize scoring, Tanken mechanic, AI escalation, basic HUD; treat exfil, advanced VFX/audio, and bespoke movement as second-tier if time constrained.
- Keep runtime performant: max ~16 players + 48 AI; use pooling, disable unused spawners, limit active VFX/SFX.
- Debug focus areas: spawn logic transitions after 60s, AI pilot suicide prevention, ensuring every class ability is selectable and functional, verifying UI asset names against definitions.
- Portal does not expose browser timers (`setTimeout`); every delay must use `await mod.Wait`, even when the Claude notes suggest otherwise.

## To Validate During Implementation
- Event handlers match SDK signatures (`OnPlayerJoinGame`, `OnGameModeStarted`, etc.).
- Use helper `MakeMessage` pattern from examples to avoid message signature errors.
- Score updates synchronise `mod.SetGameModeScore` and per-player tanken meters without overflows/reset bugs.
- Vehicle spawn uses valid asset unlock IDs and does not conflict with other spawners.
- Ensure `mod.Wait` usage in loops prevents blocking the runtime and respects async limitations.

## Old Build Observations
- Prior versions inflated Attrition rewards (50/100/500/1000) and tied Tanken unlocks to a timer; align the final build with +1/+2/+5/+10 scoring and contribution-based tanken meters.
- Legacy scripts ran endless `while(true)` loops without a termination flag and skipped calling `mod.SetGameModeEnded`; ensure match flow transitions cleanly into exfil victory and stops background tasks.
- Old HUD code assumed widget creation always succeeds; sanity-check widget handles before updates to avoid runtime null refs.

> Implementation reference lives in `CODEX_Tankenfall_Code_database.markdown`, while BF6 scripting fundamentals live in `Context files and code library/CODEX_Tankenfall_Understanding.markdown`; use the trio together to keep goals, tooling, and platform behaviour aligned.
