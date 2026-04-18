# King of the Hill — Design Specification v3

## Overview
Classic Halo 2 Crazy King-style for Halo Wars DE. A single active zone randomly rotates between 5 preset positions on the map. Players score by keeping units in the active zone. Built on top of CTF game mode infrastructure for HUD and AI behavior.

## Current Status (V3 — April 17, 2026)
- **Multi-map support** across 18 maps via IsMap-branched init
- **Random hill selection** (5 hills per map, swaps may repeat — v1 allows repeats)
- **Working prototype** on Terminal Moraine with 5 rotating hills
- Scoring works via hijacked CTF HUD (setCTFCount)
- Hill rotates every 90 seconds with sound, message, and minimap flare notifications
- Covenant doughnut (`cpgn_scn02_covdoughnut_01`) marks the active hill, moves via SetPosition
- AI pursues the active hill via SetCTFFlag on a sentinel squad
- Win condition: first team to 200 points (all members set to Won/Defeated via iterator loops)
- Team support: 1v1, 2v2, and 3v3 via FilterPlayerList on stock Team1Players (356) / Team2Players (358)
- Tested in 1v1 vs AI on Legendary on Terminal Moraine (V1)

## Multi-Map Support (V3)

| Map              | Internal name       | Notes                                  |
|------------------|---------------------|----------------------------------------|
| Terminal Moraine | terminal_moraine    | Hill 5 is a geometric mirror — may clip |
| Chasms           | chasms              |                                        |
| Tundra           | tundra              |                                        |
| Blood Gulch      | Blood_Gulch         | Stock CTF reference uses this exact case |
| Labyrinth        | labyrinth           |                                        |
| Fort Deen        | fort_deen           |                                        |
| Beacon Hill      | beacon_hill         | Memorial Basin alias? — needs verification |
| Beasley's Plateau| beasleys_plateau    |                                        |
| The Docks        | the_docks           | Flat map, all hills Y=-18.02            |
| Red River        | redriver_1          |                                        |
| Pirth Outskirts  | pirth_outskirts     |                                        |
| Release          | release             |                                        |
| Repository       | repository          |                                        |
| Frozen Valley    | frozen_valley       |                                        |
| Barrens          | baron_1_swe         |                                        |
| Glacial Ravine   | glacial_ravine      |                                        |
| Exile            | exile               |                                        |
| Crevice          | crevice             |                                        |

If `IsMap` doesn't match for a given map, the slot Vector vars stay (0,0,0) and rotation will send the active zone to map origin — that's the "oh the internal name is wrong" symptom. Update the corresponding `KotHMapName_*` String var.

### Coordinate note
All hill Y values are the **desired final Y** for both the scoring zone AND the covenant doughnut object. No runtime offset is applied. The old `KotHH1DoughnutPos`/`KotHH2DoughnutPos` vars are retained as dead code but no longer referenced.

## Random Selection + Dispatch Chain (V3)

The old ping-pong design (triggers 257 ↔ 258) is replaced with a selector + dispatch + slot pattern:

```
    Init (250) ──────────► activates 18 MapInits + Setup (256) + Selector (280)
                                  │                              │
    Each MapInit (290-307)       (fires immediately if IsMap)   (90s timer)
        │                         │                              │
        └─► CopyLocation × 5: per-map KotH*_Hill{1..5} ──► Hill{1..5}Loc
                                                                  ▼
                                                     RandomCount(1,5) → currentHill
                                                                  │
                                                                  ▼
                                                     Dispatch1 (281) ══ currentHill==1 ══► Slot1 (270) rotates ───► re-activates Selector
                                                                  │no
                                                                  ▼
                                                     Dispatch2 (282) ══ currentHill==2 ══► Slot2 (271) rotates
                                                                  │no
                                                                  ▼
                                                     Dispatch3 (283) ...
                                                                  ▼
                                                     Dispatch5 (285) ══ currentHill==5 ══► Slot5 (274) rotates
```

Each Slot trigger does the same rotation pipeline as the legacy ping-pong: SetPosition doughnut, CreateSquad+SetCTFFlag, CopyLocation zone, Revealer ×2, ShowMessage, PlaySoundFile, FlareMinimap ×2, then TriggerActivate the Selector (which restarts the 90s timer).

### Known v1 limitation: repeats allowed
The selector rolls 1..5 with no "don't pick last hill" guard. Over 5 rotations you'll see ~1 repeat on average. A no-repeat guard would add one MathInt bump trigger (if raw >= lastHill, +1) and a CopyInt for lastHill state — deferred to v2.

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
- **Scoring:** +1 point every 3 seconds while any teammate has units in the zone
- **Contested:** NOT YET IMPLEMENTED — both teams can currently score simultaneously
- **Win condition:** first team to **200 points** wins (adjustable via var 2054)
- **Team support:** 1v1, 2v2, 3v3 — scoring filters by Team1Players (356) / Team2Players (358); win/defeat state applied to every team member via IteratorPlayerList loop
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
Based on stock CTF with KotH triggers added. Trigger IDs 250-307, var IDs 2000-2278.

### Triggers

| ID       | Name                         | Type                | Purpose                                                       |
|----------|------------------------------|---------------------|---------------------------------------------------------------|
| 250      | KotH Init                    | Always-true init    | SetResources diagnostic, activates 18 map-init triggers, timer loop, Setup, Selector |
| 251      | KotH Team1 Score             | Conditional         | CanGetUnits on Team1Players (356) in zone → MathInt → CopyInt → setCTFCount |
| 252      | KotH Team2 Score             | Conditional         | CanGetUnits on Team2Players (358) in zone → MathInt → CopyInt → setCTFCount |
| 254      | KotH Timer Loop              | 3s timer loop       | Activates 251, 252, 259, 260 → self-reactivates              |
| 256      | KotH Setup                   | 5s delay            | CopyLocation Hill1Loc→ZoneCenter, →846, CopyFloat 1723, CreateObject doughnut at Hill1Loc |
| 257      | KotH Rotate to Hill 2        | DEPRECATED          | Legacy ping-pong — CommentOut="true"                          |
| 258      | KotH Rotate to Hill 1        | DEPRECATED          | Legacy ping-pong — CommentOut="true"                          |
| 259      | KotH Team1 Win Check         | Conditional         | CompareInteger Team1Score >= 200 → IteratorPlayerList(356)→winners, IteratorPlayerList(358)→losers, activate 262/263 |
| 260      | KotH Team2 Win Check         | Conditional         | CompareInteger Team2Score >= 200 → IteratorPlayerList(358)→winners, IteratorPlayerList(356)→losers, activate 262/263 |
| 262      | KotH Winners Loop            | Iterator loop       | NextPlayer(winnersIter) → SetPlayerState Won (one player per eval) |
| 263      | KotH Losers Loop             | Iterator loop       | NextPlayer(losersIter) → SetPlayerState Defeated (one player per eval) |
| 270-274  | KotH Rotate Slot {1..5}      | Non-conditional     | When activated by Dispatch: SetPosition doughnut, CreateSquad+SetCTFFlag, CopyLocation zone, Revealer×2, ShowMessage, PlaySoundFile, Flare×2, activate Selector |
| 280      | KotH Selector                | 90s timer           | RandomCount(1,5) → KotHCurrentHill → activate Dispatch 1      |
| 281-285  | KotH Dispatch {1..5}         | Conditional         | CompareInteger KotHCurrentHill == N → activate Slot N, else → activate Dispatch N+1 |
| 290-307  | KotH MapInit {Map}           | Conditional (IsMap) | CopyLocation × 5: per-map hill vars → Hill{1..5}Loc slot vars |

### Scoring Pipeline (per tick, per team)
1. **CanGetUnits** — FilterPlayerList (Team1Players=356 or Team2Players=358) + FilterLocation + FilterDistance → found units?
2. **MathInt** — score += 1
3. **CopyInt** — score → display var
4. **setCTFCount** — push to HUD widget

### Win Pipeline (once per match, per winning team)
1. **CompareInteger** — teamScore >= 200 (in trigger 259 or 260)
2. **IteratorPlayerList** × 2 — initialize winners iterator on winning team's list, losers iterator on opposing team's list
3. **TriggerActivate** — activate 262 (Winners Loop) and 263 (Losers Loop)
4. **NextPlayer + SetPlayerState** — each loop evaluates until iterator exhausted, setting Won/Defeated on every team member

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

| ID        | Type        | Name                    | Value                              |
|-----------|-------------|-------------------------|------------------------------------|
| 2001      | Vector      | KotHZoneCenter          | (runtime — active hill pos)        |
| 2002      | Float       | KotHZoneRadius          | 40                                 |
| 2003      | Player      | KotHPlayer1             | 1                                  |
| 2004      | Player      | KotHPlayer2             | 2                                  |
| 2032      | Vector      | KotHHill2Loc            | (runtime — populated by MapInit)   |
| 2033      | Vector      | KotHH1DoughnutPos       | 651,0,651 (unused as of V3)        |
| 2034      | Vector      | KotHH2DoughnutPos       | 891,0,640 (unused as of V3)        |
| 2037      | Time        | KotHRotateDelay         | 90000                              |
| 2038      | Squad       | KotHNewFlag             | (runtime)                          |
| 2039      | ProtoObject | KotHDoughnutProto       | cpgn_scn02_covdoughnut_01          |
| 2041      | Object      | KotHDoughnutObj         | (runtime — DO NOT overwrite)       |
| 2049      | Vector      | KotHHill1Loc            | (runtime — populated by MapInit)   |
| 2054      | Integer     | KotHWinScore            | 200                                |
| 2058      | Iterator    | KotHWinnersIter         | (runtime — winning team)           |
| 2059      | Iterator    | KotHLosersIter          | (runtime — losing team)            |
| 2060      | Player      | KotHWinnerOut           | (runtime — NextPlayer output)      |
| 2061      | Player      | KotHLoserOut            | (runtime — NextPlayer output)      |
| 2062      | Trigger     | KotHWinnersLoopRef      | 262                                |
| 2063      | Trigger     | KotHLosersLoopRef       | 263                                |
| 2100-2189 | Vector      | KotH{Map}_Hill{1..5}    | Baked-in hill coords (18 maps × 5) |
| 2200-2217 | String      | KotHMapName_{Map}       | Internal map names for IsMap       |
| 2220-2222 | Vector      | KotHHill{3,4,5}Loc      | (runtime — populated by MapInit)   |
| 2230      | Integer     | KotHCurrentHill         | (runtime — RandomCount result)     |
| 2232-2233 | Integer     | KotHRandMin/Max         | 1 / 5                              |
| 2240-2244 | Integer     | KotHIdx{1..5}           | 1-5 constants for CompareInteger   |
| 2250      | Trigger     | KotHSelectorRef         | 280                                |
| 2251-2255 | Trigger     | KotHSlot{1..5}Ref       | 270-274                            |
| 2256-2260 | Trigger     | KotHDispatch{1..5}Ref   | 281-285                            |
| 2261-2278 | Trigger     | KotHMapInit{Map}        | 290-307                            |

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

### Team Support (2v2, 3v3) — IMPLEMENTED
- Scoring: CanGetUnits uses FilterPlayerList with stock vars 356 (Team1Players) and 358 (Team2Players), populated by stock group 4 triggers
- Win condition: IteratorPlayerList (DBID=120) walks each team's PlayerList and SetPlayerState applies Won/Defeated to every member via NextPlayer (DBID=118) condition loops
- FilterPlayer is set to a stock null Player var (2 or 139) so only FilterPlayerList is evaluated
- Limitation: Revealer Owner and FlareMinimapNormal FlaringPlayer still target team leader only (2003/2004). Teams share FoW in HW so the revealer is functionally team-wide; the flare is cosmetic. A future polish pass could iterate these too.

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