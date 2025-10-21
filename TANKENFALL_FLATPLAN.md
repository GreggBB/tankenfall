# Tankenfall 2: Development Research Report
## Battlefield Portal × Titanfall 2 Attrition Hybrid Mod

**Research Date:** October 2025  
**Purpose:** Comprehensive technical and design guidance for developing a professional-quality Portal mod combining Battlefield's tactical vehicle combat with Titanfall 2's Attrition mode mechanics

---

## Core Mechanics: Implementation Roadmap

### 1. Hybrid Scoring System

**Attrition-Inspired Point Values:**
```
AI Grunts (Bots on Beginner difficulty): 50 points
AI Elites (Bots on Advanced difficulty): 100 points  
Enemy Pilots (Human players): 500 points
Objective Captures: 1000 points
Vehicle Destructions ("Titan Kills"): 800 points
Assist Actions: 100 points
```

**Implementation in Portal TypeScript:**
```typescript
let teamScores = [0, 0];

export function OnPlayerDied(victim: mod.Player, killer: mod.Player, 
                              deathType: mod.DeathType, weapon: mod.WeaponUnlock) {
  const isAI = mod.GetSoldierState(victim, mod.SoldierStateBool.IsAISoldier);
  const killerTeam = mod.GetTeam(killer);
  const teamIndex = mod.GetObjId(killerTeam);
  
  if (isAI) {
    // Check difficulty level for point value
    teamScores[teamIndex] += 50; // or 100 for elite AI
  } else {
    teamScores[teamIndex] += 500;
  }
  
  UpdateScoreUI(killerTeam);
  CheckVehicleEarning(killer);
}
```

**Key Design Decision:** Objectives worth 2x player kills ensures team-focused play doesn't degenerate into pure deathmatch.

### 2. Vehicle-as-Titan Earning System

**Titanfall 2 uses hybrid passive + active earning. Adapt for Portal:**

**Earning Mechanics:**
- **Passive Timer:** 180 seconds baseline = guaranteed vehicle every 3 minutes
- **Combat Acceleration:** 
  - AI kill: -2 seconds
  - Player kill: -15 seconds  
  - Objective capture: -30 seconds
  - Damage to enemy vehicles: -5 seconds per 500 damage
  - Battery delivery equivalent: Ammo resupply to vehicles reduces allied vehicle spawn by -20 seconds

**Implementation Pattern:**
```typescript
let playerVehicleTimers: Map<number, number> = new Map();
const VEHICLE_BASE_TIME = 180;
const VEHICLE_READY_THRESHOLD = 0;

async function VehicleTimerTick() {
  while (true) {
    await mod.Wait(1);
    
    const players = mod.GetPlayers();
    for (let i = 0; i < mod.GetArrayLength(players); i++) {
      const player = mod.AtArray(players, i);
      const playerId = mod.GetObjId(player);
      
      let timer = playerVehicleTimers.get(playerId) || VEHICLE_BASE_TIME;
      timer = Math.max(0, timer - 1); // Passive reduction
      
      playerVehicleTimers.set(playerId, timer);
      
      if (timer <= VEHICLE_READY_THRESHOLD) {
        NotifyVehicleReady(player);
      }
    }
  }
}

function ReduceVehicleTimer(player: mod.Player, reduction: number) {
  const playerId = mod.GetObjId(player);
  const current = playerVehicleTimers.get(playerId) || VEHICLE_BASE_TIME;
  playerVehicleTimers.set(playerId, Math.max(0, current - reduction));
}
```

**Vehicle Calldown Mechanic:**
Use Portal's call-in system OR custom InteractPoint objects:
```typescript
export function OnPlayerInteract(player: mod.Player, point: mod.InteractPoint) {
  const playerId = mod.GetObjId(player);
  const timer = playerVehicleTimers.get(playerId) || VEHICLE_BASE_TIME;
  
  if (timer <= 0) {
    // Spawn vehicle at player location + offset
    const playerPos = mod.GetSoldierState(player, mod.SoldierStateVector.GetPosition);
    SpawnVehicleAt(playerPos, player);
    playerVehicleTimers.set(playerId, VEHICLE_BASE_TIME); // Reset timer
    
    // VFX and audio
    TriggerVehicleDropEffects(playerPos);
  }
}
```

### 3. AI Wave System for Infantry Warfare

**Titanfall 2's Escalation Pattern:**
- Early: Grunts only (groups of 4)
- Mid: Grunts + Spectres
- Late: All types + Reapers

**Portal Adaptation:**
```typescript
let matchPhase = 0;
const AI_SPAWNERS = {
  GRUNT: mod.GetSpawner(1),
  ELITE: mod.GetSpawner(2)
};

export async function OnGameModeStarted() {
  StartAIWaveSystem();
}

async function StartAIWaveSystem() {
  while (true) {
    await mod.Wait(30); // 30-second waves
    
    const elapsed = GetMatchElapsedTime();
    
    if (elapsed < 300) { // 0-5 minutes: Grunts only
      SpawnAIWave(AI_SPAWNERS.GRUNT, 8, mod.GetTeam(1));
      SpawnAIWave(AI_SPAWNERS.GRUNT, 8, mod.GetTeam(2));
    } else if (elapsed < 600) { // 5-10 minutes: Mix
      SpawnAIWave(AI_SPAWNERS.GRUNT, 6, mod.GetTeam(1));
      SpawnAIWave(AI_SPAWNERS.ELITE, 2, mod.GetTeam(1));
      SpawnAIWave(AI_SPAWNERS.GRUNT, 6, mod.GetTeam(2));
      SpawnAIWave(AI_SPAWNERS.ELITE, 2, mod.GetTeam(2));
    } else { // 10+ minutes: Elite-heavy
      SpawnAIWave(AI_SPAWNERS.GRUNT, 4, mod.GetTeam(1));
      SpawnAIWave(AI_SPAWNERS.ELITE, 4, mod.GetTeam(1));
      SpawnAIWave(AI_SPAWNERS.GRUNT, 4, mod.GetTeam(2));
      SpawnAIWave(AI_SPAWNERS.ELITE, 4, mod.GetTeam(2));
    }
  }
}

function SpawnAIWave(spawner: mod.Spawner, count: number, team: mod.Team) {
  for (let i = 0; i < count; i++) {
    mod.SpawnAIFromAISpawner(spawner, team);
  }
}
```

**AI Configuration for Grunt vs Elite:**
```typescript
export function OnPlayerDeployed(player: mod.Player) {
  if (mod.GetSoldierState(player, mod.SoldierStateBool.IsAISoldier)) {
    const spawnerType = DetermineSpawnerType(player);
    
    if (spawnerType === "GRUNT") {
      mod.SetPlayerMaxHealth(player, 75);
      mod.SetAIToHumanDamageModifier(0.7); // Reduced damage
      mod.AISetMoveSpeed(player, mod.MoveSpeed.Walk);
    } else if (spawnerType === "ELITE") {
      mod.SetPlayerMaxHealth(player, 150);
      mod.SetAIToHumanDamageModifier(1.2); // Increased damage
      mod.AISetMoveSpeed(player, mod.MoveSpeed.Sprint);
    }
  }
}
```

### 4. Class Abilities System (Titanfall-Inspired)

**Portal doesn't have native abilities, but we can simulate via:**
- **InteractPoint pickups** for one-time buffs
- **Timed cooldown tracking** for repeatable abilities
- **Modifier changes** triggered by script

**Stim Ability (Speed Boost):**
```typescript
let abilityCD: Map<number, number> = new Map();
const STIM_COOLDOWN = 20;
const STIM_DURATION = 5;

export async function OnPlayerInteract(player: mod.Player, point: mod.InteractPoint) {
  const pointId = mod.GetObjId(point);
  
  if (pointId === 10) { // Stim pickup point
    const playerId = mod.GetObjId(player);
    const lastUse = abilityCD.get(playerId) || 0;
    const now = Date.now() / 1000;
    
    if (now - lastUse >= STIM_COOLDOWN) {
      ActivateStim(player);
      abilityCD.set(playerId, now);
    } else {
      const remaining = STIM_COOLDOWN - (now - lastUse);
      mod.DisplayNotificationMessage(
        mod.Message("Cooldown: {} seconds", Math.ceil(remaining)), 
        player
      );
    }
  }
}

async function ActivateStim(player: mod.Player) {
  // Visual feedback
  mod.DisplayCustomNotificationMessage(
    mod.Message("STIM ACTIVATED"),
    mod.CustomNotificationSlots.TopRight,
    5,
    player
  );
  
  // Movement speed increase (Portal limitation: no per-player speed mod)
  // Workaround: Use temporary invulnerability or damage boost as proxy
  // OR place player in different team temporarily with different modifiers
  
  await mod.Wait(STIM_DURATION);
  
  // End effect
}
```

**Limitation Workaround:** Portal doesn't support per-player movement speed modifications via TypeScript. **Best solution:** Use Portal Builder's team-based movement modifiers and assign players to sub-teams based on ability state, OR provide alternative benefits (faster health regen, damage boost).

### 5. Enhanced Movement System

**Slide-Jump Implementation:**

Portal's movement system is less flexible than Titanfall 2, but we can approximate:

**Portal Builder Settings:**
- Movement Speed Multiplier: 125% (faster baseline)
- Slide: Enabled
- Sprint: Enabled

**Script Enhancement:**
```typescript
// Monitor for slide-jump combos and reward with temporary benefits
let playerMomentum: Map<number, number> = new Map();

export function OnPlayerAction(player: mod.Player, action: string) {
  if (action === "SLIDE_JUMP") { // Pseudo-code, Portal doesn't expose this directly
    const playerId = mod.GetObjId(player);
    const momentum = playerMomentum.get(playerId) || 0;
    
    playerMomentum.set(playerId, momentum + 1);
    
    // Reward chained movement with temporary buff
    if (momentum >= 3) {
      ApplyMomentumBuff(player);
    }
  }
}
```

**Practical Limitation:** Portal's movement system is locked to Battlefield physics. **Focus instead on:** Map design with jump pads, ziplines, and grapple points to simulate Titanfall's verticality and mobility.

### 6. Capture Point Objectives

**Implementation:**
```typescript
let captureProgress: Map<number, number> = new Map();
const CAPTURE_REQUIREMENT = 100;

export function OnPlayerEnterAreaTrigger(player: mod.Player, trigger: mod.AreaTrigger) {
  const triggerId = mod.GetObjId(trigger);
  
  // Start capture loop
  CapturePointLoop(player, trigger);
}

async function CapturePointLoop(player: mod.Player, trigger: mod.AreaTrigger) {
  const triggerId = mod.GetObjId(trigger);
  
  while (mod.PlayerInAreaTrigger(player, trigger)) {
    let progress = captureProgress.get(triggerId) || 0;
    progress += 1;
    captureProgress.set(triggerId, progress);
    
    // UI Update
    UpdateCaptureUI(trigger, progress);
    
    if (progress >= CAPTURE_REQUIREMENT) {
      CompleteCapturePoint(player, trigger);
      break;
    }
    
    await mod.Wait(0.5); // Update every 0.5 seconds
  }
}

function CompleteCapturePoint(player: mod.Player, trigger: mod.AreaTrigger) {
  const team = mod.GetTeam(player);
  
  // Award points
  AwardTeamPoints(team, 1000);
  
  // Audio + VFX
  mod.PlaySound(/* capture sound */, trigger, team);
  mod.EnableVFX(/* capture VFX object */, true);
  
  // Notification
  mod.DisplayHighlightedWorldLogMessage(
    mod.Message("Objective Captured!"),
    team
  );
  
  // Reduce vehicle timers for capturing team
  const players = mod.GetPlayersOnTeam(team);
  for (let i = 0; i < mod.GetArrayLength(players); i++) {
    ReduceVehicleTimer(mod.AtArray(players, i), 30);
  }
}
```

### 7. Exfiltration Endgame Sequence

**Triggered at score threshold or time limit:**
```typescript
const EXFIL_SCORE_TRIGGER = 10000;
let exfilActive = false;

function CheckExfiltration() {
  if (teamScores[0] >= EXFIL_SCORE_TRIGGER || teamScores[1] >= EXFIL_SCORE_TRIGGER) {
    if (!exfilActive) {
      ActivateExfiltration();
    }
  }
}

async function ActivateExfiltration() {
  exfilActive = true;
  
  // Announcement
  mod.DisplayHighlightedWorldLogMessage(
    mod.Message("EXTRACTION ZONE ACTIVE"),
    mod.AllPlayers()
  );
  
  // Spawn extraction zone VFX
  mod.EnableVFX(mod.GetSpatialObject(100), true);
  
  // Enable extraction capture point
  const exfilZone = mod.GetAreaTrigger(50);
  
  // 60-second timer for extraction
  await mod.Wait(60);
  
  // Check which team controls zone
  DetermineExfilWinner();
}

function DetermineExfilWinner() {
  const exfilZone = mod.GetAreaTrigger(50);
  const team1Count = CountPlayersInZone(exfilZone, mod.GetTeam(1));
  const team2Count = CountPlayersInZone(exfilZone, mod.GetTeam(2));
  
  if (team1Count > team2Count) {
    DeclareWinner(mod.GetTeam(1));
  } else if (team2Count > team1Count) {
    DeclareWinner(mod.GetTeam(2));
  } else {
    // Tie = higher score wins
    DeclareWinner(teamScores[0] > teamScores[1] ? mod.GetTeam(1) : mod.GetTeam(2));
  }
}
```

---

## Technical Approaches That Work

### 1. Event-Driven Architecture

**Core Pattern:**
```typescript
// Global state
let gameState = {
  phase: "SETUP",
  scores: [0, 0],
  vehicleTimers: new Map(),
  captureProgress: new Map()
};

// Event handlers modify state
export function OnPlayerDied(victim, killer, deathType, weapon) {
  UpdateScores(victim, killer);
  UpdateVehicleTimers(killer);
  CheckPhaseTransitions();
}

export function OnPlayerEnterAreaTrigger(player, trigger) {
  StartCapture(player, trigger);
}

// Separate game loop for continuous updates
async function GameLoop() {
  while (gameState.phase !== "ENDED") {
    await mod.Wait(0.1);
    UpdateUI();
    UpdateTimers();
    CheckWinConditions();
  }
}
```

### 2. ObjId System for Object Management

**Critical Pattern:**
```typescript
// ALWAYS use GetObjId for comparisons and storage
const playerId = mod.GetObjId(player);
playerData.set(playerId, { kills: 0, vehicleTimer: 180 });

// Retrieve objects by ID
const spawner = mod.GetSpawner(1);
const capturePoint = mod.GetCapturePoint(2);
const areaTrigger = mod.GetAreaTrigger(3);
```

### 3. ParseUI for Complex Interfaces

```typescript
import { ParseUI } from 'modlib';

const vehicleUI = ParseUI({
  type: "Container",
  name: "vehicle_hud",
  size: [400, 100],
  anchor: mod.UIAnchor.BottomCenter,
  children: [
    {
      type: "Container",
      name: "timer_bg",
      size: [400, 50],
      anchor: mod.UIAnchor.TopCenter,
      children: [
        {
          type: "Text",
          name: "timer_text",
          textLabel: "VEHICLE READY: 180s",
          size: [380, 40],
          anchor: mod.UIAnchor.Center
        }
      ]
    },
    {
      type: "Image",
      name: "vehicle_icon",
      imageType: mod.UIImageType.Vehicle,
      size: [80, 80],
      anchor: mod.UIAnchor.BottomLeft
    }
  ]
});
```

### 4. Performance-Optimized AI Management

**For 16 players + 48 AI (8v8 + 24 AI per side):**

```typescript
// Use static bot count (not dynamic backfill)
// Set in Portal Builder: Bot Spawn Type = Static, PvE AI

// Efficient spawn management
let activeAI: Map<number, mod.Player> = new Map();
const MAX_AI_PER_TEAM = 24;

function SpawnAIWave(spawner: mod.Spawner, count: number, team: mod.Team) {
  const currentCount = CountAIOnTeam(team);
  const toSpawn = Math.min(count, MAX_AI_PER_TEAM - currentCount);
  
  for (let i = 0; i < toSpawn; i++) {
    const ai = mod.SpawnAIFromAISpawner(spawner, team);
    activeAI.set(mod.GetObjId(ai), ai);
  }
}

// Quick cleanup on death
export function OnPlayerDied(player: mod.Player, killer, deathType, weapon) {
  if (mod.GetSoldierState(player, mod.SoldierStateBool.IsAISoldier)) {
    const spawner = GetSpawnerForAI(player);
    mod.AISetUnspawnOnDead(spawner, true);
    mod.SetUnspawnDelayInSeconds(spawner, 2);
    activeAI.delete(mod.GetObjId(player));
  }
}

// Throttled update loop
async function AIBehaviorUpdate() {
  while (true) {
    await mod.Wait(0.5); // Update every 0.5s, not every frame
    
    activeAI.forEach(ai => {
      UpdateAIBehavior(ai);
    });
  }
}
```

### 5. Audio/Visual Feedback Layering

**Every major event needs 3-layer feedback:**

```typescript
function OnVehicleEarned(player: mod.Player) {
  // 1. Audio
  mod.PlaySound(/* titan ready sound */, player, player);
  
  // 2. Visual - UI Notification
  mod.DisplayCustomNotificationMessage(
    mod.Message("VEHICLE READY"),
    mod.CustomNotificationSlots.TopCenter,
    5,
    player
  );
  
  // 3. Visual - HUD Update
  mod.SetUIWidgetText(vehicleTimerWidget, "READY");
  mod.SetUIWidgetColor(vehicleTimerWidget, mod.Color.Green);
}

function OnObjectiveCaptured(team: mod.Team, point: mod.AreaTrigger) {
  // 1. Audio - Spatial
  mod.PlaySound(/* capture complete */, point, mod.AllPlayers());
  
  // 2. Visual - VFX
  mod.EnableVFX(GetVFXForPoint(point), true);
  
  // 3. Visual - World Message
  mod.DisplayHighlightedWorldLogMessage(
    mod.Message("Objective Captured!"),
    team
  );
  
  // 4. Visual - Score Update
  UpdateScoreUI(team);
}
```

### 6. Cinematic Intro Sequence

```typescript
export async function OnGameModeStarted() {
  // Deploy cam setup (done in Godot SDK)
  // In script: Delay player spawning for intro
  
  // Display mission briefing
  mod.DisplayHighlightedWorldLogMessage(
    mod.Message("ATTRITION MODE: Eliminate enemy forces and capture objectives"),
    mod.AllPlayers()
  );
  
  await mod.Wait(3);
  
  mod.DisplayHighlightedWorldLogMessage(
    mod.Message("Earn your vehicle through combat performance"),
    mod.AllPlayers()
  );
  
  await mod.Wait(3);
  
  // Enable VFX for atmosphere
  mod.EnableVFX(mod.GetSpatialObject(1), true);
  
  // Start match
  StartGameLogic();
}
```

---

## Common Mistakes to Avoid

### 1. Spawn System Failures

**Problem:** Spawn camping, instant death on spawn, black spawn screens

**Solutions:**
- Test ALL spawn points extensively
- Implement 3-second invulnerability: 
```typescript
export function OnPlayerDeployed(player: mod.Player) {
  ApplySpawnProtection(player, 3);
}

async function ApplySpawnProtection(player: mod.Player, duration: number) {
  // Portal doesn't have native invulnerability
  // Workaround: Temporary massive health boost
  const originalHealth = mod.GetSoldierState(player, mod.SoldierStateNumber.CurrentHealth);
  mod.SetPlayerMaxHealth(player, 10000);
  
  await mod.Wait(duration);
  
  mod.SetPlayerMaxHealth(player, 100);
}
```
- Use wave spawns (groups of 2-4, every 8-12 seconds)
- Never create tiny maps with spawn line-of-sight

### 2. Unclear Objectives

**Problem:** Players don't know what to do or where to go

**Solutions:**
- Prominent UI markers for objectives
- WorldIcon objects in 3D space at objectives:
```typescript
const objectiveIcon = mod.GetWorldIcon(1);
mod.SetWorldIconVisible(objectiveIcon, true);
mod.SetWorldIconColor(objectiveIcon, 1, 0, 0); // Red for uncaptured
```
- Introductory message explaining mode
- Minimap indicators

### 3. Performance Issues with AI

**Problem:** Lag, stuttering, dropped frames with 16+ players + 48+ AI

**Solutions:**
- Cap AI at 24 per team (48 total) for 16-player matches
- Use beginner/intermediate difficulty (lower processing)
- Throttle behavior updates (0.5-1s intervals, not per-frame)
- Limit vehicle count (2-3 per team maximum)
- Set vehicle spawn delay multiplier to 2.5-3x normal
- Disable vehicle health regeneration
- Use simple AI behaviors for background units

### 4. Contradictory Rules Logic

**Problem:** Impossible conditions, blocks not linked to Mod Block

**Solutions:**
- Triple-check all logic connections
- Test edge cases (empty teams, late joins, simultaneous events)
- Use subroutines for complex logic:
```typescript
// Good: Modular logic
function CanPlayerSpawnVehicle(player: mod.Player): boolean {
  const timer = playerVehicleTimers.get(mod.GetObjId(player)) || 180;
  const inVehicle = mod.GetSoldierState(player, mod.SoldierStateBool.InVehicle);
  const teamVehicleCount = CountTeamVehicles(mod.GetTeam(player));
  
  return timer <= 0 && !inVehicle && teamVehicleCount < 3;
}

// Bad: Nested conditions without checking
if (timer <= 0) {
  // Spawn vehicle (might be in vehicle already, causing bugs)
}
```

### 5. Poor Cooldown Balance

**Problem:** Abilities too frequent (spam) or too rare (forgotten)

**Solutions:**
- Standard formula: Cooldown = 3-6x duration
- Short abilities (5s): 15-25s cooldown
- Medium abilities (8s): 40-50s cooldown
- Long abilities (10s): 60-90s cooldown
- Playtest extensively and adjust

### 6. Untested Vehicle Spawning

**Problem:** Vehicles spawn inside geometry, fall through world, clip

**Solutions:**
- Use Godot SDK to place VehicleSpawner objects precisely
- Test every spawn point in-game
- Use ForceVehicleSpawnerSpawn for scripted spawns:
```typescript
function SpawnVehicleAtLocation(spawner: mod.VehicleSpawner) {
  // Verify spawner validity
  if (spawner) {
    mod.ForceVehicleSpawnerSpawn(spawner);
    
    // VFX at spawn
    TriggerVehicleDropVFX(spawner);
  }
}
```

### 7. Overcomplicated Scoring

**Problem:** Players can't understand how points are earned

**Solutions:**
- Maximum 4-5 scoring categories
- Clear UI feedback on every point gain
- Scoreboard breakdown by category
- Avoid hidden multipliers or opaque calculations

---

## Audio/Visual Polish Techniques

### 1. Layered Soundscapes

**Ambient + Action + UI:**
```typescript
// Place SFX objects in Godot for ambient sounds
// Trigger action sounds on events
export function OnPlayerDied(victim, killer, deathType, weapon) {
  // Kill confirmation sound
  mod.PlaySound(/* kill sound */, killer, killer);
}

// UI sounds for feedback
function UpdateScoreUI(team: mod.Team) {
  mod.PlaySound(/* score increment sound */, null, team);
}
```

### 2. Dynamic Music System

**Phase-Based Intensity:**
```typescript
let currentMusicPhase = "CALM";

function UpdateMusic() {
  const elapsed = GetMatchElapsedTime();
  const scoreDiff = Math.abs(teamScores[0] - teamScores[1]);
  
  if (elapsed > 780 || exfilActive) { // Final 2 minutes or exfil
    if (currentMusicPhase !== "INTENSE") {
      // Trigger intense music (if Portal supports, otherwise use VFX as cue)
      currentMusicPhase = "INTENSE";
    }
  } else if (scoreDiff > 3000) {
    if (currentMusicPhase !== "WINNING") {
      currentMusicPhase = "WINNING";
    }
  }
}
```

### 3. VFX for Key Moments

**Vehicle Drop Effect:**
```typescript
async function TriggerVehicleDropVFX(location: mod.Vector) {
  // Incoming VFX
  const incomingVFX = mod.GetSpatialObject(10);
  mod.EnableVFX(incomingVFX, true);
  
  await mod.Wait(2);
  
  // Impact VFX
  const impactVFX = mod.GetSpatialObject(11);
  mod.EnableVFX(impactVFX, true);
  
  // Audio
  mod.PlaySound(/* titan drop impact */, location, mod.AllPlayers());
  
  await mod.Wait(3);
  
  // Cleanup
  mod.EnableVFX(incomingVFX, false);
  mod.EnableVFX(impactVFX, false);
}
```

### 4. Color Coding for Teams

**Consistent Visual Language:**
```typescript
const TEAM_COLORS = {
  TEAM1: { r: 0, g: 0.5, b: 1 }, // Blue
  TEAM2: { r: 1, g: 0, b: 0 }    // Red
};

function UpdateCaptureUI(trigger: mod.AreaTrigger, team: mod.Team) {
  const icon = GetWorldIconForTrigger(trigger);
  const color = team === mod.GetTeam(1) ? TEAM_COLORS.TEAM1 : TEAM_COLORS.TEAM2;
  
  mod.SetWorldIconColor(icon, color.r, color.g, color.b);
}
```

### 5. Progress Indicators

**Visual Progress Bars:**
```typescript
function UpdateVehicleTimerUI(player: mod.Player) {
  const timer = playerVehicleTimers.get(mod.GetObjId(player)) || 180;
  const percentage = ((180 - timer) / 180) * 100;
  
  // Update UI container width or fill
  mod.SetUIWidgetSize(timerBarWidget, [percentage * 4, 50], player);
  
  // Color shift: Red → Yellow → Green
  if (percentage < 33) {
    mod.SetUIWidgetColor(timerBarWidget, mod.Color.Red, player);
  } else if (percentage < 66) {
    mod.SetUIWidgetColor(timerBarWidget, mod.Color.Yellow, player);
  } else {
    mod.SetUIWidgetColor(timerBarWidget, mod.Color.Green, player);
  }
}
```

### 6. Screen Effects for Drama

**Exfiltration Warning:**
```typescript
function ActivateExfiltration() {
  // Flash screen borders for all players
  // (Portal has limited screen effects, use UI containers as workaround)
  
  CreateScreenFlash(mod.AllPlayers());
}

async function CreateScreenFlash(players: mod.Array<mod.Player>) {
  const flashUI = ParseUI({
    type: "Container",
    name: "screen_flash",
    size: [1920, 1080],
    anchor: mod.UIAnchor.Center,
    backgroundColor: [1, 0, 0, 0.3] // Red overlay
  });
  
  await mod.Wait(0.2);
  
  mod.SetUIVisible(flashUI, false, mod.AllPlayers());
}
```

---

## Balance and Pacing Recommendations

### 1. Match Duration: 15-20 Minutes

**Phase Structure:**
- **0-5 min:** Opening (AI grunts only, objective captures, vehicle earning)
- **5-13 min:** Mid-game (mixed AI, vehicle combat, momentum swings)
- **13-15 min:** Endgame (exfiltration active, score acceleration)
- **15-18 min:** Overtime if needed (sudden death, limited respawns)

### 2. Score Targets

**Win Conditions:**
- **Score Limit:** 15,000 points (balanced for 15-min matches)
- **Time Limit:** 15 minutes (highest score wins)
- **Exfiltration:** Successful extraction = instant win

**Scoring Pacing:**
- Early leads easy to establish (encourages aggression)
- Mid-game harder to maintain (tactical depth required)
- Late game can be overcome with exfiltration

### 3. Team Composition for 8v8

**Recommended Setup:**
- 8 human players per team
- 16-24 AI per team (scales based on performance)
- 2-3 vehicles per team maximum concurrent

**Class Distribution:**
No hard restrictions, but bonus points for:
- Support roles (ammo, repairs): +10% score multiplier
- Objective players (time on point): +5% capture speed
- Vehicle hunters (anti-armor): Bonus points for vehicle damage

### 4. Vehicle Balance

**"Titan" Vehicles as Force Multipliers:**
- **Health:** 3-5 player focus fire to destroy
- **Firepower:** Can kill infantry in 1-2 hits, but infantry has counterplay
- **Mobility:** Slower than Titanfall titans, but intimidating presence
- **Lifespan:** Average 90-120 seconds per vehicle

**Anti-Vehicle Balance:**
- Dedicated anti-armor class deals 2x vehicle damage
- Flanking hits deal 1.5x damage (encourage positioning)
- Team focus fire rewarded with assist points

### 5. Respawn System

**Wave Spawns:**
```typescript
let respawnWave: Map<number, mod.Player[]> = new Map();

export function OnPlayerDied(victim: mod.Player, killer, deathType, weapon) {
  if (!mod.GetSoldierState(victim, mod.SoldierStateBool.IsAISoldier)) {
    const teamId = mod.GetObjId(mod.GetTeam(victim));
    const wave = respawnWave.get(teamId) || [];
    wave.push(victim);
    respawnWave.set(teamId, wave);
  }
}

async function RespawnWaveSystem() {
  while (true) {
    await mod.Wait(10); // 10-second wave intervals
    
    respawnWave.forEach((wave, teamId) => {
      wave.forEach(player => {
        mod.SpawnPlayerFromSpawnPoint(player, GetTeamSpawn(teamId));
      });
      wave.length = 0; // Clear wave
    });
  }
}
```

**Timing:**
- Standard: 10-second waves
- Endgame: 15-second waves (higher death stakes)
- Overtime: 20-second waves OR no respawns

### 6. AI Density Management

**Dynamic Scaling:**
```typescript
function CalculateAICount(): number {
  const humanCount = CountHumanPlayers();
  const targetTotal = 64; // 64 total combatants
  
  return Math.max(0, targetTotal - humanCount);
}

async function AIScalingSystem() {
  while (true) {
    await mod.Wait(30);
    
    const targetAI = CalculateAICount();
    const currentAI = CountActiveAI();
    
    if (currentAI < targetAI) {
      SpawnAdditionalAI(targetAI - currentAI);
    }
  }
}
```

### 7. Comeback Mechanics

**Rubber-Banding (Subtle):**
```typescript
function CalculateScoreMultiplier(team: mod.Team): number {
  const teamIndex = mod.GetObjId(team);
  const enemyIndex = teamIndex === 0 ? 1 : 0;
  const scoreDiff = teamScores[enemyIndex] - teamScores[teamIndex];
  
  if (scoreDiff > 3000) {
    return 1.15; // 15% bonus when behind
  } else if (scoreDiff > 1500) {
    return 1.08; // 8% bonus when slightly behind
  }
  
  return 1.0;
}

function AwardPoints(team: mod.Team, basePoints: number) {
  const multiplier = CalculateScoreMultiplier(team);
  const finalPoints = Math.floor(basePoints * multiplier);
  
  const teamIndex = mod.GetObjId(team);
  teamScores[teamIndex] += finalPoints;
}
```

**Exfiltration as Equalizer:**
- Losing team gets exfiltration advantage (spawns closer to zone)
- Winning team must actively defend (can't just farm AI)

---

## Specific Code Patterns from Community

### 1. Player Tracking System

**From BF Portal Community:**
```typescript
type PlayerData = {
  kills: number;
  deaths: number;
  vehicleTimer: number;
  abilityCD: number;
  currentStreak: number;
};

let playerStats: Map<number, PlayerData> = new Map();

export function OnPlayerJoinGame(player: mod.Player) {
  const playerId = mod.GetObjId(player);
  playerStats.set(playerId, {
    kills: 0,
    deaths: 0,
    vehicleTimer: 180,
    abilityCD: 0,
    currentStreak: 0
  });
}

export function OnPlayerLeaveGame(playerId: number) {
  playerStats.delete(playerId);
}

export function OnPlayerDied(victim: mod.Player, killer: mod.Player, 
                              deathType: mod.DeathType, weapon: mod.WeaponUnlock) {
  const victimId = mod.GetObjId(victim);
  const killerId = mod.GetObjId(killer);
  
  const victimData = playerStats.get(victimId);
  const killerData = playerStats.get(killerId);
  
  if (victimData) {
    victimData.deaths++;
    victimData.currentStreak = 0;
  }
  
  if (killerData) {
    killerData.kills++;
    killerData.currentStreak++;
    
    // Streak bonuses
    if (killerData.currentStreak === 5) {
      AwardStreakBonus(killer, 500);
    }
  }
}
```

### 2. Conditional State Pattern (modlib)

**For One-Time Triggers:**
```typescript
import { getPlayerCondition } from 'modlib';

let vehicleReadyCondition = getPlayerCondition(player, 0);

// In update loop
function UpdateVehicleTimer(player: mod.Player) {
  const timer = playerVehicleTimers.get(mod.GetObjId(player)) || 180;
  
  // Triggers ONCE when timer hits 0 (not every frame)
  if (vehicleReadyCondition.update(timer <= 0)) {
    NotifyVehicleReady(player);
  }
}
```

### 3. Economy System (from "Besieged")

```typescript
let playerCurrency: Map<number, number> = new Map();

export function OnPlayerDied(victim: mod.Player, killer: mod.Player, 
                              deathType: mod.DeathType, weapon: mod.WeaponUnlock) {
  if (!mod.GetSoldierState(killer, mod.SoldierStateBool.IsAISoldier)) {
    const killerId = mod.GetObjId(killer);
    const current = playerCurrency.get(killerId) || 0;
    
    const isAI = mod.GetSoldierState(victim, mod.SoldierStateBool.IsAISoldier);
    const reward = isAI ? 10 : 50;
    
    playerCurrency.set(killerId, current + reward);
    UpdateCurrencyUI(killer);
  }
}

export function OnPlayerInteract(player: mod.Player, point: mod.InteractPoint) {
  const pointId = mod.GetObjId(point);
  const playerId = mod.GetObjId(player);
  const currency = playerCurrency.get(playerId) || 0;
  
  if (pointId === 20) { // Vehicle purchase point
    const cost = 200;
    if (currency >= cost) {
      SpawnVehicleForPlayer(player);
      playerCurrency.set(playerId, currency - cost);
    }
  }
}
```

### 4. Area Control System

**From Community "Conquest" Implementations:**
```typescript
let zoneControl: Map<number, { team: mod.Team, strength: number }> = new Map();

export function OnPlayerEnterAreaTrigger(player: mod.Player, trigger: mod.AreaTrigger) {
  const triggerId = mod.GetObjId(trigger);
  const playerTeam = mod.GetTeam(player);
  
  const current = zoneControl.get(triggerId) || { team: null, strength: 0 };
  
  if (current.team === playerTeam) {
    current.strength += 1;
  } else {
    current.strength -= 1;
    if (current.strength <= 0) {
      current.team = playerTeam;
      current.strength = 1;
      OnZoneCaptured(playerTeam, trigger);
    }
  }
  
  zoneControl.set(triggerId, current);
}

export function OnPlayerExitAreaTrigger(player: mod.Player, trigger: mod.AreaTrigger) {
  const triggerId = mod.GetObjId(trigger);
  const current = zoneControl.get(triggerId);
  
  if (current) {
    current.strength = Math.max(0, current.strength - 1);
  }
}
```

### 5. Debug Logging Pattern

```typescript
const DEBUG_MODE = false;

function DebugLog(message: string, data?: any) {
  if (DEBUG_MODE) {
    console.log(`[${Date.now()}] ${message}`, data);
  }
}

export function OnPlayerDied(victim: mod.Player, killer: mod.Player, 
                              deathType: mod.DeathType, weapon: mod.WeaponUnlock) {
  DebugLog("Player died", {
    victim: mod.GetObjId(victim),
    killer: mod.GetObjId(killer),
    deathType: deathType
  });
  
  // Actual logic...
}
```

---

## Limitations and Workarounds

### 1. No Per-Player Movement Speed Modification

**Limitation:** `mod.SetPlayerMovementSpeed()` doesn't exist; only team-wide modifiers in Portal Builder

**Workarounds:**
- **Team Swapping:** Temporarily assign player to different team with different modifiers (complex, not recommended)
- **Alternative Buffs:** Instead of speed, provide damage boost, faster health regen, or temporary shield
- **Map Design:** Use jump pads, ziplines, vehicles for mobility bursts
- **Acceptance:** Focus on other aspects of Titanfall feel (verticality, AI combat, vehicle earning)

### 2. Limited Custom Audio/VFX Assets

**Limitation:** Can't upload custom sounds or effects, limited to game assets

**Workarounds:**
- **Creative Reuse:** Repurpose existing sounds (titan mode sounds, vehicle impacts)
- **Layering:** Combine multiple sounds for unique effects
- **Timing:** Vary timing of same sound to create variety
- **VFX Placement:** Strategic placement of existing VFX for dramatic moments

### 3. No Wallrunning in Battlefield

**Limitation:** Core Titanfall mechanic impossible to replicate

**Workarounds:**
- **Vertical Map Design:** Multi-level maps with clear paths between levels
- **Grapple Points:** (if Portal supports interaction points as grapple targets)
- **Jump Pads:** Place trampolines/boosters for vertical mobility
- **Ziplines:** Pre-placed movement paths
- **Accept Different Feel:** Focus on "spirit" of Titanfall (fast pace, AI combat, power fantasy) rather than exact mechanics

### 4. Vehicle Customization Limited

**Limitation:** Can't modify individual vehicle weapons, only global settings

**Workarounds:**
- **Restrictions:** Use Portal Builder restrictions to limit available vehicles
- **Damage Modifiers:** Adjust all vehicle damage via global modifiers
- **Health Scaling:** Make vehicles "tankier" to feel like Titans via health multipliers
- **Spawn Control:** Limit which vehicles appear on map

### 5. AI Behavior Constraints

**Limitation:** AI doesn't have perfect pathfinding, can get stuck

**Workarounds:**
- **Simple Objectives:** Give AI straightforward goals (move to point A, defend area B)
- **Open Maps:** Avoid complex geometry where AI can get stuck
- **Respawn on Stuck:** Check for stationary AI and force respawn:
```typescript
async function AntiStuckSystem() {
  let lastPositions: Map<number, mod.Vector> = new Map();
  
  while (true) {
    await mod.Wait(10);
    
    activeAI.forEach(ai => {
      const currentPos = mod.GetSoldierState(ai, mod.SoldierStateVector.GetPosition);
      const lastPos = lastPositions.get(mod.GetObjId(ai));
      
      if (lastPos && VectorDistance(currentPos, lastPos) < 5) {
        // AI hasn't moved 5 units in 10 seconds = stuck
        RespawnAI(ai);
      }
      
      lastPositions.set(mod.GetObjId(ai), currentPos);
    });
  }
}
```

### 6. Single Script File Limitation

**Limitation:** Portal only accepts one TypeScript file upload

**Workarounds:**
- **Namespace Pattern:**
```typescript
namespace PlayerManager {
  export function handleJoin(player: mod.Player) { }
  export function handleDeath(player: mod.Player) { }
}

namespace VehicleManager {
  export function spawnVehicle(player: mod.Player) { }
  export function trackVehicleTimer() { }
}

namespace UIManager {
  export function createHUD() { }
  export function updateScore() { }
}
```
- **Build Step:** Develop in multiple files locally, concatenate before upload
- **Modular Functions:** Use subroutines extensively for organization

### 7. HUD Customization Restrictions

**Limitation:** Binary HUD element toggles (on/off), not granular control

**Workarounds:**
- **Custom UI Overlays:** Create custom UI elements that supplement default HUD
- **Minimap Emphasis:** Since can't disable selectively, design around it
- **World Icons:** Use 3D icons in world space for objectives instead of relying on HUD

---

## Development Checklist

### Phase 1: Foundation (Week 1-2)
- [x] Research complete
- [ ] Set up Portal SDK (download Godot 4.4.1 tool)
- [ ] Create base experience in Portal Builder
  - [ ] Select maps (medium-sized, 8v8 appropriate)
  - [ ] Configure teams (8v8, PvE AI enabled)
  - [ ] Set modifiers (movement speed 125%, slide enabled)
- [ ] Place basic objects in Godot
  - [ ] AI_Spawner objects (ObjId 1, 2 for grunt/elite)
  - [ ] VehicleSpawner objects (ObjId 10-15, one per vehicle type)
  - [ ] AreaTrigger objects (ObjId 20-25 for capture points)
  - [ ] VFX objects (ObjId 100+ for effects)
  - [ ] DeployCam for intro sequence

### Phase 2: Core Systems (Week 3-4)
- [ ] Implement scoring system
  - [ ] AI kills (50/100 pts)
  - [ ] Player kills (500 pts)
  - [ ] Objective captures (1000 pts)
  - [ ] Score tracking and UI
- [ ] Implement vehicle earning
  - [ ] Passive timer (180s)
  - [ ] Combat acceleration
  - [ ] Ready notification
  - [ ] Spawn mechanic
- [ ] Implement AI wave system
  - [ ] Phase-based escalation
  - [ ] Grunt/elite differentiation
  - [ ] Wave spawning (30s intervals)
- [ ] Implement capture points
  - [ ] Area trigger detection
  - [ ] Progress tracking
  - [ ] Capture complete rewards

### Phase 3: Gameplay Features (Week 5-6)
- [ ] Implement ability system
  - [ ] 2-3 abilities with cooldowns
  - [ ] InteractPoint pickups
  - [ ] UI indicators
- [ ] Implement exfiltration
  - [ ] Trigger at score threshold
  - [ ] Zone activation
  - [ ] Winner determination
- [ ] Implement UI/UX
  - [ ] Vehicle timer HUD
  - [ ] Score display
  - [ ] Objective markers
  - [ ] Notifications
- [ ] Implement audio/VFX
  - [ ] Vehicle drop effects
  - [ ] Capture point audio
  - [ ] Kill confirmations
  - [ ] Phase music

### Phase 4: Polish (Week 7-8)
- [ ] Balance tuning
  - [ ] Playtest 10+ sessions
  - [ ] Adjust point values
  - [ ] Tune cooldowns
  - [ ] Balance vehicle power
- [ ] Performance optimization
  - [ ] AI count adjustment
  - [ ] Script throttling
  - [ ] Vehicle limits
- [ ] Edge case fixes
  - [ ] Spawn protection
  - [ ] Stuck AI respawn
  - [ ] Late join handling
- [ ] Final polish
  - [ ] UI alignment
  - [ ] Audio mixing
  - [ ] VFX timing
  - [ ] Intro sequence

### Phase 5: Testing & Iteration (Week 9+)
- [ ] Community playtest
- [ ] Feedback integration
- [ ] Balance patches
- [ ] Documentation (rules, changelog)

---

## Final Recommendations

### What to Prioritize

**High Priority (Must-Have):**
1. Scoring system (objectives + kills)
2. Vehicle earning mechanics (timer + acceleration)
3. AI wave spawning (escalation)
4. Basic UI (score, vehicle timer, objectives)
5. Spawn protection
6. Capture point system

**Medium Priority (Should-Have):**
7. Exfiltration endgame
8. Audio feedback (vehicle ready, captures)
9. VFX for vehicle drops
10. Ability system (2-3 abilities)
11. Team-based HUD elements
12. Cinematic intro

**Low Priority (Nice-to-Have):**
13. Advanced movement mechanics
14. Complex AI behaviors
15. Economy system
16. Advanced VFX
17. Dynamic music
18. Extensive customization

### Success Metrics

**Technical:**
- Stable 60fps with 16 players + 48 AI
- No game-breaking bugs
- Clean spawn system (no instant deaths)

**Design:**
- 75%+ playtest participants report "fun"
- Average match length 15-20 minutes
- Multiple viable strategies observed
- Score differentials 20-40% (clear winner, not blowout)

**Community:**
- Clear, accurate server description
- Active player engagement
- Positive feedback on BFPortal.gg
- Return player rate 60%+

### Development Philosophy

**Iterate, Don't Perfect:**
- Launch with minimum viable experience
- Gather feedback early
- Adjust based on data, not assumptions
- Version control everything

**Focus on Feel:**
- "Does this feel like Titanfall Attrition?" > "Is this exactly like Titanfall?"
- Fast pace, AI combat, power fantasy most important
- Accept Battlefield limitations gracefully

**Community First:**
- Playtest extensively
- Respond to feedback
- Share development process
- Credit inspirations

---

## Conclusion

**Tankenfall 2 is achievable** within Battlefield Portal's capabilities, but requires **creative adaptation** rather than direct translation. Focus on the **core feel** of Titanfall 2's Attrition mode:

✅ **Hybrid PvP/PvE scoring** (achievable via AI bots + objective points)  
✅ **Power fantasy earning** (vehicle calldowns via timer + combat)  
✅ **Escalating intensity** (AI wave progression + exfiltration endgame)  
✅ **Fast-paced combat** (Portal Builder modifiers + map design)  
✅ **Multiple scoring paths** (objectives, AI farming, player hunting)  

⚠️ **Accept limitations:**  
- No wallrunning (compensate with verticality and speed)
- No per-player movement mods (use alternative buffs)
- Limited assets (creative reuse)

**The community craves innovation.** A well-executed hybrid tactical-fast mode with Attrition scoring, vehicle earning, and exfiltration endgame will stand out. Start with **core mechanics**, test relentlessly, iterate based on fun, and polish for professional feel.

**Your mod can succeed** by delivering what players want: **accessible, high-paced combat with skill expression, clear objectives, and dramatic moments.**

Good luck, Pilot. Standby for Tankenfall.