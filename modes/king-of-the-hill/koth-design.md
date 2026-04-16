# King of the Hill — Design Specification v2

## Overview
Classic Halo 2-style King of the Hill for Halo Wars DE. A single active zone rotates between positions on the map. Players score by keeping units in the active zone. Built on top of CTF game mode infrastructure for HUD and AI behavior.

## Current Status (V1 — April 14, 2026)
- **Working prototype** on Terminal Moraine with 2 rotating hills
- Scoring works via hijacked CTF HUD (setCTFCount)
- Hill rotates every 90 seconds with sound, message, and minimap flare notifications
- Covenant doughnut (`cpgn_scn02_covdoughnut_01`) marks the active hill, moves via SetPosition
- AI pursues the active hill via SetCTFFlag on a sentinel squad
- Win condition: first to 120 points
- Tested in 1v1 vs AI on Legendary

## Architecture Decision: CTF Mode
KotH runs inside `capflagworld.triggerscript` using CTF game mode. This is required because:
- **setCTFCount (DBID=1083)** only works in CTF mode — provides the score HUD
- **SetCTFFlag (DBID=1076)** makes AI path to the hill — engine-level pursuit behavior
- **Skirmish mode was tested and rejected** — no HUD, no flag spawning, no AI lure

### Trade-offs of CTF Mode
- Stock flag spawn chain runs from ERA EditorData and **cannot be stopped** (see Engine Discoveries)
- "Capture 3 Flags" welcome message displays at game start
- Orphan sentinel squads accumulate at each rotation (can't kill/destroy CTF flag squads)
- Stock flag spawns ~60-90s into the game at a location we control via CopyLocation

## Rules
- **2 zones** currently (expandable to 5), rotating in fixed order
- **90-second rotation** interval per zone
- **Scoring:** +1 point every 3 seconds while a player has units in the zone
- **Contested:** NOT YET IMPLEMENTED — both teams can currently score simultaneously
- **Win condition:** first to **120 points** wins (adjustable via var 2054)
- **Team support:** 1v1 working, 2v2/3v3 requires swapping FilterPlayer to FilterPlayerList
- **AI behavior:** AI paths to active hill via SetCTFFlag, sends waves on Legendary

## Zone Positions (Terminal Moraine)

### Current (V1)
| Zone | Description  | X     | Y  | Z     | Doughnut Y |
|------|-------------|-------|----|----- -|------------|
| 0    | Map center  | 651.0 | 18 | 651.0 | 0          |
| 1    | East bridge | 891.0 | 18 | 640.0 | 0          |

### Planned (V2)
| Zone | Description       | X      | Y   | Z      |
|------|-------------------|--------|-----|--------|
| 0    | Map center        | 651.0  | 18  | 651.0  |
| 1    | West bridge       | 274.0  | 18  | 640.0  |
| 2    | East side         | 1028.0 | 18  | 640.0  |
| 3    | North midfield    | 651.0  | 25  | 400.0  |
| 4    | South midfield    | 651.0  | 25  | 900.0  |

Detection radius: **40 game units** (diameter ~80). Layout forms a cross/plus pattern.

## Current Trigger Architecture

### File: ModData/data/triggerscripts/capflagworld.triggerscript
Based on stock CTF with KotH triggers added (IDs 250-260, vars 2000-2054).

### Triggers

| ID  | Name                    | Type             | Purpose                                                       |
|-----|-------------------------|------------------|---------------------------------------------------------------|
| 250 | KotH Init               | Always-true init | SetResources diagnostic, activates scoring loop + setup + rotation |
| 251 | KotH P1 Score           | Conditional      | CanGetUnits P1 in zone → MathInt → CopyInt → setCTFCount     |
| 252 | KotH P2 Score           | Conditional      | CanGetUnits P2 in zone → MathInt → CopyInt → setCTFCount     |
| 254 | KotH Timer Loop         | 3s timer loop    | Activates 251, 252, 259, 260 → self-reactivates              |
| 256 | KotH Setup              | 5s delay         | CopyLocation var 846, CopyFloat var 1723, CreateObject doughnut |
| 257 | KotH Rotate to Hill 2   | 90s delay        | SetPosition doughnut, CreateSquad flag, SetCTFFlag, CopyLocation scoring zone, revealers, notifications → activates 258 |
| 258 | KotH Rotate to Hill 1   | 90s delay        | SetPosition doughnut, CreateSquad flag, SetCTFFlag, CopyLocation scoring zone, revealers, notifications → activates 257 |
| 259 | KotH P1 Win Check       | Conditional      | CompareInteger P1Score >= 120 → SetPlayerState Won/Defeated   |
| 260 | KotH P2 Win Check       | Conditional      | CompareInteger P2Score >= 120 → SetPlayerState Won/Defeated   |

### Scoring Pipeline (per tick, per player)
1. **CanGetUnits** — FilterPlayer + FilterLocation + FilterDistance → found units?
2. **MathInt** — score += 1
3. **CopyInt** — score → display var
4. **setCTFCount** — push to HUD widget

### Rotation Pipeline (per swap)
1. **SetPosition** — move doughnut to new hill
2. **CreateSquad** — spawn new sentinel at new hill
3. **SetCTFFlag** — mark new sentinel as CTF flag (AI lure)
4. **CopyLocation** — move scoring zone (var 2001) to new hill
5. **Revealer** × 2 — reveal new hill for both players
6. **ShowMessage** — display notification text
7. **PlaySoundFile** — play `vog_gamemode_flagcapture`
8. **FlareMinimapNormal** × 2 — ping minimap for both players
9. **TriggerActivate** — activate the return-rotation trigger

### Key TriggerVars

| ID   | Type        | Name              | Value                        |
|------|-------------|-------------------|------------------------------|
| 2001 | Vector      | KotHZoneCenter    | (runtime — active hill pos)  |
| 2002 | Float       | KotHZoneRadius    | 40                           |
| 2003 | Player      | KotHPlayer1       | 1                            |
| 2004 | Player      | KotHPlayer2       | 2                            |
| 2032 | Vector      | KotHHill2Loc      | 891,18,640                   |
| 2033 | Vector      | KotHH1DoughnutPos | 651,0,651                    |
| 2034 | Vector      | KotHH2DoughnutPos | 891,0,640                    |
| 2037 | Time        | KotHRotateDelay   | 90000                        |
| 2038 | Squad       | KotHNewFlag       | (runtime)                    |
| 2039 | ProtoObject | KotHDoughnutProto | cpgn_scn02_covdoughnut_01    |
| 2041 | Object      | KotHDoughnutObj   | (runtime — DO NOT overwrite) |
| 2049 | Vector      | KotHHill1Loc      | 651,18,651                   |
| 2054 | Integer     | KotHWinScore      | 120                          |

### Stock Vars Reused
| ID  | Type     | Name              | Purpose in KotH              |
|-----|----------|-------------------|------------------------------|
| 750 | Integer  | Team1Score        | P1 score counter             |
| 751 | Integer  | Team1CapDisplay   | P1 display var for HUD       |
| 752 | Integer  | Team2Score        | P2 score counter             |
| 753 | Integer  | Team2CapDisplay   | P2 display var for HUD       |
| 755 | Operator | GTE               | GreaterThanOrEqualTo         |
| 846 | Vector   | CenterOfPlayers   | Overwritten for flag spawn   |
| 885 | ProtoSquad| CTF monitor      | game_CTF_monitor_01          |
| 888 | Player   | Gaia              | Player 0 (flag owner)        |
| 893 | Squad    | MonitorFlag       | Stock flag squad ref         |
| 1723| Float    | FlagPlaceRadius   | Overwritten to 1.0           |

---

## Engine Discoveries (Critical)

### EditorData Executes Independently
- The `<EditorData>` section in triggerscripts contains a DUPLICATE `<TriggerSystem>` with its own Triggers block
- The engine loads and executes EditorData triggers from the ERA **independently** of ModData overrides
- `CommentOut="true"` on triggers in the outer block does NOT affect EditorData copies
- Stripping EditorData from the ModData file has no effect — ERA version loads regardless
- **Impact:** stock CTF flag spawn chain cannot be blocked, only redirected

### CTF Flag Squad Protection
The sentinel squad spawned by CTF has deep engine protection:
- **Destroy (DBID=38)** — silently fails
- **Kill (DBID=37)** — crashes the game
- **Teleport (DBID=354)** — entity teleports but visual model stays behind
- **Move (DBID=66)** — requires Velocity > 0, but stock random movement chain interferes
- **Workaround:** spawn additional sentinels with CreateSquad + reassign via SetCTFFlag

### ModData Override Scope
- Triggerscript outer Triggers block: **overridden** by ModData
- Triggerscript EditorData: **NOT overridden** — ERA version always loads
- objects_update.xml: **works** as expected
- SCN files: **works** as expected
- Skirmish triggerscripts: **works** (confirmed with skirmishworld.triggerscript)
- .gfx Flash files: **untested**

---

## DBIDs Reference (Proven This Session)

### Working Effects
| DBID | Name              | Notes                                          |
|------|-------------------|------------------------------------------------|
| 35   | CreateObject      | Spawns Objects (not Squads). Works for scenery  |
| 36   | CreateSquad       | Works in init and single-fire triggers          |
| 99   | CopyLocation      | Runtime Vector var overwrite — key for rotation |
| 103  | CopyFloat         | Runtime Float var overwrite                     |
| 130  | SetPlayerState    | Won/Defeated — game ends correctly              |
| 135  | FlareMinimapNormal| Minimap ping for specified player               |
| 285  | Revealer          | Reveals fog of war at location                  |
| 354  | Teleport          | Works on squads (partially — visual issues)     |
| 456  | PowerGrant        | Found in deathmatch/waveattack scripts. Untested|
| 537  | SetPosition       | Instant reposition for Objects. Key discovery   |
| 904  | ShowMessage       | On-screen text notification                     |
| 1076 | SetCTFFlag        | AI lure — engine pursues flagged squad          |
| 1083 | setCTFCount       | CTF HUD score display (CTF mode only)           |
| 1084 | PlaySoundFile     | Audio notification                              |

### Failed Effects on CTF Flag
| DBID | Name    | Result                              |
|------|---------|-------------------------------------|
| 37   | Kill    | Crashes the game                    |
| 38   | Destroy | Silently fails                      |
| 66   | Move    | Requires Velocity, stock chain interferes |
| 354  | Teleport| Entity moves, visual stays          |

---

## Known Issues

### Functional
- Orphan sentinels accumulate at each rotation (cosmetic — AI follows active flag)
- Stock ERA spawns its own flag at ~60-90s (can't be prevented, location is controlled)
- Both teams can score simultaneously (no contested logic yet)
- Score HUD says "Team 1" / "Team 2" from stock CTF LocStringIDs
- "Capture 3 Flags" welcome message from stock CTF
- 7777 supplies diagnostic still active in Init

### Cosmetic
- Old sentinels linger at previous hill positions
- Doughnut may clip terrain at some positions (Y coordinate needs per-hill tuning)

---

## Phase 2: Contested Logic + Teams

### Contested Detection Pattern
Each scoring tick needs to detect BOTH teams:
1. CanGetUnits for Team 1 → set Team1Present bool
2. CanGetUnits for Team 2 → set Team2Present bool  
3. Score trigger: `Team1Present AND NOT Team2Present → P1 scores` (and vice versa)
4. Both booleans reset to false at start of each tick

### Team Support (2v2, 3v3)
- Swap FilterPlayer (single player) to FilterPlayerList in CanGetUnits
- Stock vars 356 (Team1Players) and 358 (Team2Players) are populated by group 4 triggers
- Win condition needs to iterate player lists for SetPlayerState on all team members
- Stock iterator pattern exists in capflagworld group 10

---

## Phase 3: Polish

### 3a: Visual Hill Marker
- Doughnut works but could be enhanced with additional visual effects
- PowerGrant (DBID=456) could fire healing beam as timed visual beacon — needs research
- CreateObject can spawn additional scenery at active hill
- SetPosition confirmed working for show/hide pattern (Y=-200 to hide)

### 3b: Audio
- PlaySoundFile works with stock sound names (e.g., `vog_gamemode_flagcapture`)
- Custom audio would need files in ModData and format research
- Multiple sounds could layer for dramatic effect

### 3c: HUD
- setCTFCount only works in CTF mode
- Custom Flash HUD would require .gfx ModData override (untested)
- Community collaboration candidate for Flash/ActionScript work

### 3d: Welcome Message
- Override stock "Capture 3 Flags" text
- May require locale file modification or LocStringID discovery
- Could use ShowMessage at game start with custom LocStringID

---

## Phase 4: Multi-Hill Expansion

### 5-Hill Rotation
Current ping-pong architecture (257 ↔ 258) needs to become a chain:
- Option A: 5 rotation triggers in sequence, last one loops to first
- Option B: Integer hill counter + CompareInteger guards on a single set of scoring triggers
- Option B is cleaner but requires runtime Vector var selection (may not be possible)
- Option A is verbose but uses proven patterns

### Per-Hill Tuning
Each hill position needs:
- Game coordinates (X, Y, Z)
- Doughnut Y offset (terrain-dependent)
- Possibly different detection radii

---

## Phase 5: Multi-Map Support
- Zone positions are hardcoded per map
- scenariodescriptions.xml registers maps as selectable in lobby
- Same triggerscript could work on multiple maps with different coordinates
- IsMap condition (DBID=1087) can branch to different position sets per map

---

## File Inventory

### Active Mod Files
| File | Path | Purpose |
|------|------|---------|
| capflagworld.triggerscript | ModData/data/triggerscripts/ | KotH game logic |
| objects_update.xml | ModData/data/ | CTF flag modifications |

### Backups to Preserve  
| File | Description |
|------|-------------|
| capflagworld_10_25_04_PM.triggerscript | Working scoring prototype (base for all builds) |
| KotH_working_prototype.capflagworld.triggerscript | Labeled working version |

### Files to Clean Up
| File | Action |
|------|--------|
| skirmishworld.triggerscript | Delete or rename to 1skirmishworld — KotH version doesn't work |
| 1skirmishworld.triggerscript | Keep — prevents stock override from March session |