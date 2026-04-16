# Drag the Reactor — Design Specification

## Overview
CTF-style game mode where players use their Elephant to hitch and tow a Gaia-owned reactor to their base. The reactor deals ram damage when towed through enemies and has a massive health pool, making escort gameplay critical.

## Current Status
On hold pending KotH completion. Core primitives proven but blocked by CreateSquad-in-loop crashes and inability to unhitch/reset the reactor.

Several discoveries from the KotH session (April 14) are directly applicable:
- **SetPosition (DBID=537)** could reposition the reactor instantly instead of Destroy+Recreate
- **CreateObject (DBID=35)** could spawn visual markers at the reactor or scoring zones
- **CopyLocation (DBID=99)** enables runtime position tracking without GetLocation
- **EditorData dual execution** means any CTF-based approach will have stock flag interference
- **Kill (DBID=37) crashes on engine-protected squads** — likely applies to reactor too

## Rules
- **One reactor** spawns at map center (Gaia-owned, neutral)
- Players hitch the reactor with their Elephant and tow it toward their base
- **Scoring:** proximity detection — reactor near your base for **2 minutes** scores a point
- Reactor stays Gaia-owned even when hitched (ownership never transfers)
- Ram damage (500 DPS, AOE 40) when towed through enemies
- On reactor destruction: timer resets, new reactor spawns at last known position

## Key Technical Findings

### Reactor Object
- Proto: `cpgn_scn14_reactor_01` (exists as both Object and Squad)
- ObjectType: `Towable` (one of only 2 objects with this tag)
- HP: 26,500 | Velocity: 12
- Visual: `campaign/scn14/reactor_01/reactor_01.vis` (may need accessible path)
- Has `CoreSlide` persistent action (designed to be dragged)
- Cannot be unhitched once hitched (campaign never needed unhitch logic)

### Detection Pattern (PROVEN)
**Flip the logic:** don't find the reactor then check what's near it. Instead:
- `CanGetUnits` with FilterPlayer=Gaia, FilterObjectType=Towable, FilterLocation=base position, FilterDistance=200
- This correctly detects "is a Gaia towable thing near my base"
- GetLocation on reactor squad returns null/invalid — do NOT use
- Must filter by ObjectType=Towable — filtering just Gaia player catches rebel creep units too

### Squad/Object Manipulation (from KotH findings)
| Approach | Result on CTF Flag | Expected on Reactor |
|----------|-------------------|---------------------|
| Destroy (DBID=38) | Silently fails | ❌ Crashed (March session) |
| Kill (DBID=37) | Crashes | Likely crashes |
| SetPosition (DBID=537) | Not tested on squads | Worth testing — could reset reactor position |
| Teleport (DBID=354) | Entity moves, visual stays | Worth testing |
| Move (DBID=66) | Works with Velocity > 0 | 🔲 Not tested on hitched units |
| CreateSquad (DBID=36) | Works in single-fire triggers | ✅ Works in Init, ❌ crashes in rapid loop |

### Known Issues
- Reactor currently invisible (`.vis` file not extracted from ERA)
- CreateSquad crashes when called rapidly in conditional trigger chains (works in Init)
- Destroy crashes on hitched reactor
- Elephant requires `GaiaOwned` hitch rule in tactics to hitch neutral reactor
- Duplicate reactor bug if reactor exists in both SCN and triggerscript
- Elephant sometimes spins when trying to hitch — known game bug

## Trigger Architecture

### Architecture Decision: Game Mode
Two options, both with trade-offs:

**CTF Mode (capflagworld.triggerscript)**
- Pros: setCTFCount for HUD, SetCTFFlag for AI lure, proven scoring pipeline
- Cons: stock flag spawn chain from ERA EditorData can't be stopped, "Capture 3 Flags" message
- This is what KotH uses successfully

**Skirmish Mode (skirmishworld.triggerscript or SCN TriggerSystem)**
- Pros: no competing stock triggers
- Cons: no HUD (setCTFCount doesn't work), no AI lure mechanism, CreateSquad failed in testing
- Rejected for KotH, may work differently for reactor mode

### Core Triggers
| Trigger         | Purpose                                              |
|-----------------|------------------------------------------------------|
| ReactorInit     | CreateSquad reactor at map center (651, 18, 651)     |
| ProximityCheck  | CanGetUnits: Gaia Towable near P1/P2 base            |
| TimerStart      | TimerCreate when reactor detected near base           |
| TimerCheck      | TimerIsDone after 2 minutes → score point            |
| TimerReset      | On reactor leaving zone: TimerDestroy, reset          |
| RespawnReactor  | On destruction: spawn new reactor at center           |
| WinCheck        | CompareInteger score → SetPlayerState Won/Defeated    |

### DBIDs Used
| DBID | Name           | Status                    |
|------|----------------|---------------------------|
| 2    | CanGetUnits    | ✅ Proven for zone detect |
| 7    | TriggerActiveTime | ✅ Proven              |
| 14   | CompareInteger | ✅ Proven (KotH win check) |
| 31   | TriggerActivate| ✅ Proven                 |
| 35   | CreateObject   | ✅ Proven (KotH doughnut) — visual markers |
| 36   | CreateSquad    | ✅ Works in Init, ❌ crashes in rapid loop |
| 37   | Kill           | ❌ Crashes on engine-protected squads |
| 38   | Destroy        | ❌ Crashes on reactor, silently fails on CTF flag |
| 66   | Move           | ⚠️ Works with Velocity > 0, untested on hitched |
| 98   | CopyInt        | ✅ Proven (KotH display pipeline) |
| 99   | CopyLocation   | ✅ Proven — runtime Vector overwrite |
| 103  | CopyFloat      | ✅ Proven — runtime Float overwrite |
| 130  | SetPlayerState | ✅ Proven (KotH win condition) |
| 179  | MathInt        | ✅ Proven (KotH scoring) |
| 277  | SetResources   | ✅ Proven (diagnostic) |
| 285  | Revealer       | ✅ Proven — reveal reactor location |
| 354  | Teleport       | ⚠️ Partial — entity moves, visual may not follow |
| 537  | SetPosition    | ✅ Proven on Objects — test on squads for reactor reset |
| 658  | TimerCreate    | 🔲 Not yet tested (used in stock CTF) |
| 659  | TimerDestroy   | 🔲 Not yet tested (used in stock CTF) |
| 661  | TimerIsDone    | 🔲 Not yet tested (used in stock CTF) |
| 904  | ShowMessage    | ✅ Proven — notifications |
| 1083 | setCTFCount    | ✅ Proven — CTF mode HUD only |
| 1084 | PlaySoundFile  | ✅ Proven — audio notifications |

## Required File Modifications
- `unsc_veh_elephant_01.tactics` — add GaiaOwned hitch rule
- Terminal Moraine `.scn` — remove any pre-placed reactor
- `capflagworld.triggerscript` or SCN TriggerSystem — game logic
- `objects_update.xml` — reactor modifications if needed
- `scenariodescriptions.xml` — required for ModData SCN loading

## Revised Approach (Post-KotH Learnings)

### Reactor Reset Without Destroy
Since Kill/Destroy crash on squads, try **SetPosition (DBID=537)** to teleport the reactor back to center instead of destroying and respawning. If SetPosition works on squad-owned objects, this solves the reset problem entirely. If not, try **Teleport (DBID=354)** which partially worked on the CTF flag (entity moved even if visual lagged).

### Scoring Display
Use setCTFCount in CTF mode (proven in KotH). Accept stock flag interference — use CopyLocation to redirect stock flag to a harmless corner of the map.

### Visual Markers
Use CreateObject (DBID=35) to spawn visual markers at base scoring zones. Covenant doughnut or other scenery to show "bring reactor here."

## Open Questions
1. Does SetPosition (DBID=537) work on Squad-type entities? (Would solve reactor reset)
2. Does Teleport preserve visual on non-CTF-flag squads? (CTF flag had visual issues)
3. Can Move (DBID=66) reposition a hitched unit?
4. Can we add towing to Warthog/Ghost via tactics files?
5. Does the reactor `.vis` file need to be in ModData for visibility?
6. Can TimerCreate/TimerIsDone handle the 2-minute hold mechanic?
7. What is the second Towable object in the game?