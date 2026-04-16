# Drag the Reactor — Design Specification

## Overview
CTF-style game mode where players use their Elephant to hitch and tow a Gaia-owned reactor to their base. The reactor deals ram damage when towed through enemies and has a massive health pool, making escort gameplay critical.

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
- Cannot be unhitched once hitched

### Detection Pattern (PROVEN)
**Flip the logic:** don't find the reactor then check what's near it. Instead:
- `CanGetUnits` with FilterPlayer=Gaia, FilterObjectType=Towable, FilterLocation=base position, FilterDistance=200
- This correctly detects "is a Gaia towable thing near my base"
- GetLocation on reactor squad returns null/invalid — do NOT use

### Known Issues
- Reactor currently invisible (`.vis` file not extracted from ERA)
- CreateSquad crashes when called rapidly in conditional trigger chains (works in Init)
- Destroy crashes on hitched reactor
- Elephant requires `GaiaOwned` hitch rule in tactics to hitch neutral reactor
- Duplicate reactor bug if reactor exists in both SCN and triggerscript

## Trigger Architecture

### Core Triggers
| Trigger         | Purpose                                              |
|-----------------|------------------------------------------------------|
| ReactorInit     | CreateSquad reactor at map center (651, 18, 651)     |
| ProximityCheck  | CanGetUnits: Gaia Towable near P1/P2 base            |
| TimerStart      | TimerCreate when reactor detected near base           |
| TimerCheck      | TimerIsDone after 2 minutes → score point            |
| TimerReset      | On reactor leaving zone: TimerDestroy, reset          |
| RespawnReactor  | On destruction: spawn new reactor at last position    |
| WinCheck        | CompareInteger score → SetPlayerState Won/Defeated    |

### DBIDs Used
| DBID | Name           | Status                    |
|------|----------------|---------------------------|
| 2    | CanGetUnits    | ✅ Proven for zone detect |
| 7    | TriggerActiveTime | ✅ Proven              |
| 31   | TriggerActivate| ✅ Proven                 |
| 36   | CreateSquad    | ✅ Works in Init, ❌ crashes in loop |
| 38   | Destroy        | ❌ Crashes on reactor     |
| 130  | SetPlayerState | ⚠️ Used but untested directly |
| 179  | MathInt        | ⚠️ Used but unconfirmed visually |
| 658  | TimerCreate    | 🔲 Not yet tested         |
| 659  | TimerDestroy   | 🔲 Not yet tested         |
| 661  | TimerIsDone    | 🔲 Not yet tested         |

## Required File Modifications
- `unsc_veh_elephant_01.tactics` — add GaiaOwned hitch rule
- Terminal Moraine `.scn` — remove any pre-placed reactor, embed triggers in TriggerSystem
- `scenariodescriptions.xml` — required for ModData SCN loading

## Open Questions
1. Why does CreateSquad crash in conditional trigger chains?
2. Can Move (DBID=66) reposition a hitched unit instead of Destroy+Recreate?
3. Does the reactor `.vis` file need to be in ModData for visibility?
4. Can we add towing to Warthog/Ghost via tactics files?
