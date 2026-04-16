# Halo Wars DE Modding — Session Summary & Lessons Learned
## March 26, 2026

---

## What We Accomplished This Session

### 1. Confirmed Triggerscript Override Works
- `skirmishworld.triggerscript` placed in `ModData/data/triggerscripts/` successfully overrides the ERA version
- The game reads and executes custom triggers from this file
- This is the foundation for ALL custom game mode work

### 2. Built and Tested Trigger Primitives
Each of these was individually proven to work in-game:

| Effect/Condition | Type | DBID | Status |
|---|---|---|---|
| SetResources | Effect | 277 | ✅ Proven (5000 supplies test) |
| CanGetUnits (radius) | Condition | 2 | ✅ Proven (zone detection) |
| TriggerActiveTime | Condition | 7 | ✅ Proven (3-second loop) |
| TriggerActivate | Effect | 31 | ✅ Proven (trigger chaining) |
| CreateSquad | Effect | 36 | ✅ Works in Init (always-true) |
| MathInt | Effect | 179 | ⚠️ Used but not visually confirmed |
| CompareInteger | Condition | 14 | ⚠️ Used but game crashed before win |
| SetPlayerState | Effect | 130 | ⚠️ Used but game crashed before win |
| Destroy | Effect | 38 | ❌ Crashes when used on reactor squad |
| GetLocation | Effect | 189 | ❌ Failed silently on reactor squad |
| Move | Effect | 66 | 🔲 Not yet tested (found in capflagworld) |
| ShowMessage | Effect | 904 | 🔲 Not yet tested (found in capflagworld) |
| PlaySoundFile | Effect | 1084 | 🔲 Not yet tested |

### 3. Drag the Core v3c — Partial Success
- Reactor spawns at map center ✅
- Scoring loop runs every 3 seconds ✅
- Zone detection (Gaia Towable near base) triggers correctly ✅
- Crashes on score (likely CreateSquad inside conditional chain) ❌
- No score display (invisible integer) — needs UI solution

### 4. Discovered Game Architecture
- Full CTF game flow mapped from capflagworld.triggerscript
- Flash UI lives in `inGameUI.era`, extracted to `.gfx` files
- Each map has its own ERA file
- ModData override likely works for `.gfx` files too (untested)

---

## Critical Lessons Learned (DO NOT FORGET)

### File Format Rules
1. **CRLF line endings are MANDATORY** — files with Unix (LF) line endings crash the game. Always verify with `file skirmishworld.triggerscript` shows "CRLF line terminators"
2. **XML encoding must be `us-ascii`** — match the original file header
3. **All optional inputs need TriggerVars** — even null/unused inputs must reference a TriggerVar with `IsNull="true"`. Omitting them may crash

### TriggerScript Structure Rules
4. **NextTriggerVarID, NextTriggerID, NextConditionID, NextEffectID in the header MUST be updated** — they must be higher than any ID used in the file
5. **EvalLimit="0" with `<And />` = fires every evaluation cycle** — this creates infinite loops. Use `EvalLimit="1"` for fire-once triggers
6. **EvalLimit="0" with a condition = ConditionalTrigger pattern** — trigger stays active, evaluates each cycle, fires when condition is met
7. **Trigger IDs, Effect IDs, Condition IDs, and TriggerVar IDs must all be unique** within the file
8. **Effect IDs can be reused across different triggers** in the original files, but safest to keep unique
9. **The `<And />` (empty) condition always evaluates to true** — used by the existing Start trigger

### Trigger Chaining Pattern
10. **Triggers activate other triggers via TriggerActivate (DBID=31)** — but the target trigger ID is stored in a TriggerVar of Type="Trigger", not referenced directly. The effect's Input references the TriggerVar ID, which contains the target Trigger ID as its value
11. **ConditionalTrigger="true" vs "false"** — ConditionalTrigger="true" means the trigger evaluates its condition when activated. ConditionalTrigger="false" with a condition means it waits (like TriggerActiveTime delay)

### CanGetUnits (DBID=2) — Proven Pattern
12. **Must provide ALL 10 optional inputs** — FilterPlayer, FilterPlayerList, FilterObjectType, FilterLocation, FilterDistance, FilterBoxForward, FilterBoxHalfX, FilterBoxHalfY, FilterBoxHalfZ, FilterUnitList — plus both outputs (FoundUnits, FoundUnitCount)
13. **Null inputs use TriggerVars with `IsNull="true"`** — each unused filter needs its own null var of the correct type
14. **FilterPlayer expects a Player var** — value "1" = Player 1 (human in slot 1), "0" = Gaia
15. **FilterObjectType can filter by game ObjectType tags** — "Building", "Unit", "Towable" etc. These match `<ObjectType>` tags in objects.xml

### SetResources (DBID=277) — Proven Pattern
16. **Cost format is `0=supplies,1=reactors,2=0,3=0,4=0`** — setting resources REPLACES the current amount, doesn't add to it
17. **Setting resources resets ALL resource types** — `0=7777,1=0` will zero out your reactor/power count! Don't use as a diagnostic if the player needs reactors

### CreateSquad (DBID=36)
18. **Works in always-true Init triggers** — confirmed: reactor spawns at specified location
19. **May crash when called rapidly from conditional trigger chains** — the Destroy+CreateSquad cycle in scoring caused crashes. Needs investigation
20. **SquadOwner determines ownership** — Player 0 = Gaia (neutral, bright green on minimap)

### Reactor-Specific Findings
21. **`cpgn_scn14_reactor_01` exists as both Object (objects.xml) and Squad (squads.xml)** — can be used with CreateSquad
22. **Has `<ObjectType>Towable</ObjectType>`** — Elephant can hitch it natively
23. **Has a visual model** — `campaign\scn14\reactor_01\reactor_01.vis` (may need to be in accessible path)
24. **HP: 26500, Velocity: 12, Ram damage: 500 DPS with AOE 40 and unit throwing**
25. **CoreSlide persistent action** — designed to be dragged
26. **Only 2 objects in the entire game have the Towable tag**
27. **Ownership never transfers when Elephant hitches reactor** — stays Gaia-owned
28. **Cannot be unhitched** — campaign never needed unhitch logic, so it likely doesn't work
29. **Duplicate reactor bug** — if reactor exists in SCN AND triggerscript spawns one, you get two. Remove from SCN if using CreateSquad

### Elephant/Hitch Mechanics
30. **Hitch action is defined in tactics file** — just 3 lines: Name, ActionType=Hitch, WorkRange=1
31. **TargetRule: Relation=Ally, TargetType=Towable** — Elephant hitches allied Towable objects
32. **Gaia-owned reactor requires special hitch rules** — needs `GaiaOwned` or similar rule to allow hitching neutral objects
33. **Elephant sometimes spins when trying to hitch** — known game bug, not our fault

### CTF Game Mode Architecture (from capflagworld.triggerscript)
34. **SetCTFFlag (DBID=1076)** — marks a squad as "the flag" with engine-level behavior (UI icons, carrier tracking, AI awareness)
35. **Flag proto is `game_CTF_monitor_01`** — a squad wrapping `game_CTF_flag_01` object
36. **No physical flag object** — just an icon over the carrier's head
37. **Flag carrier death destroys the flag** — then respawn logic triggers
38. **2-minute hold timer** — TimerCreate (658), TimerDestroy (659), TimerIsDone (661)
39. **Score increment: MathInt (DBID=179)** — `FirstInt + SecondInt → ResultInt` with Add operator
40. **Team detection at game start** — Get Teams trigger, CompareTeams (DBID=429), PlayerListAdd
41. **Win: SetPlayerState Won/Defeated** — loops through player lists to set each player's state
42. **Score display is Flash UI** — `hud_skirm_objectivescreen.gfx`, cannot be modified via triggerscripts alone

### Detection Logic
43. **"Find reactor, check what's near it" DOESN'T WORK** — GetLocation on reactor squad returns null/invalid, so CanGetUnits checks position 0,0,0
44. **"Check what's near the base" DOES WORK** — flip the logic: FilterPlayer=Gaia, FilterObjectType=Towable, FilterLocation=base position. This correctly detects reactor near base
45. **Must filter by ObjectType=Towable** — filtering just Gaia player catches rebel creep units too, causing false triggers and crashes
46. **Scoring radius of 200 game units works** — 80 was too small to reliably detect

---

## Terminal Moraine Map Data

### Player Start Positions
| Slot | Position | Notes |
|---|---|---|
| P1-A | 501, 37, 275 | Top of map |
| P1-B | 764, 37, 275 | Top of map (2v2 second base) |
| P2-A | 783, 34, 1004 | Bottom of map |
| P2-B | 524, 34, 1016 | Bottom of map (2v2 second base) |
| P5 | 421, 35, 133 | Unknown/spectator? |

### Key Landmarks
| Location | Approx Position | Notes |
|---|---|---|
| Map center | 651, 18, 651 | Reactor spawn point |
| Bridge 1 | 274, ?, 640 | Western bridge |
| Bridge area test zone | 450, 18, 645 | Proven CanGetUnits detection here |

### Map Bounds
- Approximately 1300 x 1300 game units
- Top-vs-bottom layout (P1 top, P2 bottom)

---

## Confirmed DBIDs Reference

### Conditions
| Name | DBID | Inputs | Notes |
|---|---|---|---|
| CanGetUnits | 2 | FilterPlayer, FilterPlayerList, FilterObjectType, FilterLocation, FilterDistance, FilterBoxForward/HalfX/Y/Z, FilterUnitList → FoundUnits, FoundUnitCount | Proven working |
| CanGetSquads | 4 | Similar to CanGetUnits but for squads | Crashed in our test — use CanGetUnits instead |
| TriggerActiveTime | 7 | CompareType (Operator), CompareTime (Time in ms) | Proven for loop timing |
| CompareInteger | 14 | FirstInteger, CompareType, SecondInteger | Used in CTF for score check |
| IsAlive | 80 | TestUnit/UnitList/Squad/SquadList | Used in CTF for flag carrier |
| CompareBool | 114 | FirstBool, CompareType, SecondBool | Used in CTF |
| SetPlayerState check | 131 | Player, PlayerState | PlayerInState condition |
| CanGetDesignSpheres | 974 | Type → CenterPointList, RadiusList | Used for creep camps |

### Effects
| Name | DBID | Inputs | Notes |
|---|---|---|---|
| TriggerActivate | 31 | Trigger (via TriggerVar) | Proven — trigger chaining |
| PlaySound | 33 | Sound, Queue | Found in campaign triggers |
| CreateSquad | 36 | ProtoSquad, CreateLocation, Facing, SquadOwner, etc. → CreatedSquad | Works in Init, crashes in rapid loop |
| Destroy | 38 | DestroyUnit/UnitList/Squad/SquadList/Object/ObjectList | Crashed on reactor — may not work on hitched units |
| Move | 66 | Squad, SquadList, TargetUnit, TargetSquad, TargetLocation, AttackMove, QueueOrder | Not yet tested — key for magnetic pull |
| CopyInt | 98 | IntSource → IntCopy | Used in CTF |
| CopyBool | 102 | BoolSource → BoolCopy | Used in CTF |
| GetUnits | 112 | Same filters as CanGetUnits | Effect version (vs condition) |
| SetPlayerState | 130 | Player, PlayerState (Won/Defeated/Playing) | Used for win conditions |
| MathInt | 179 | FirstInt, MathOperator (Add), SecondInt → ResultInt | Score increment |
| GetLocation | 189 | Unit, Squad, Object → OutputLocation | Failed on reactor squad |
| SetResources | 277 | Player, Resources (Cost format) | Proven working |
| LaunchScript | 392 | ScriptName + External vars | Used for creep/crate scripts |
| TimerCreate | 658 | Various → timer ID | Used in CTF hold timer |
| TimerDestroy | 659 | TimerID | Used in CTF |
| PlayChat | 729 | ChatSound, QueueSound, ChatString, ChatSpeaker, Duration, TalkingHead | Campaign dialogue |
| ShowMessage | 904 | Player, PlayerList, Sound, QueueSound, Message, duration | CTF notifications |
| GetNPCPlayersByName | 943 | NPCPlayerName, NPCPlayerState → NPCPlayerList | Creep player detection |
| SetCTFFlag | 1076 | Squad | Marks squad as CTF flag |
| PlaySoundFile | 1084 | (params unknown) | Used in CTF |

---

## TriggerVar Types Reference
Types confirmed from working scripts:
- Player, PlayerList, PlayerState
- Integer, Float, Bool, String, Time, Operator
- Vector, VectorList
- Unit, UnitList, Squad, SquadList, Object, ObjectList
- ObjectType, ProtoSquad, ProtoObject
- Cost (format: `0=supplies,1=reactors,2=0,3=0,4=0`)
- Trigger (value = target trigger ID)
- Iterator, ListPosition
- EntityFilterSet, Power
- Team, TeamList

---

## Game File Structure

### ERA Files (Game Directory)
| ERA | Contents |
|---|---|
| root.era | Base game data, Flash UI (postgame) |
| release.era | Core game data (objects.xml, squads.xml, tactics, triggerscripts) |
| inGameUI.era | In-game HUD Flash files (hud_menu, minimap, objectives, widgets, skirm_objective) |
| pregameUI.era | Pregame menus |
| loadingUI.era | Loading screens |
| locale.era | Localization strings |
| scenarioshared.era | Shared scenario assets (374MB — largest) |
| terminal_moraine.era | Terminal Moraine map assets (77MB) |
| [mapname].era | Per-map assets |
| PHXscn01-15.era | Campaign mission assets |

### Flash UI Structure (from inGameUI.era)
```
art/ui/flash/
  hud/
    hud_skirm_objective/    ← CTF/game mode score display
    hud_widgets/            ← General HUD overlay
    hud_menu/               ← Build menu (UNSC/Covenant variants)
    hud_minimap/            ← Minimap + icons
    hud_reticle/            ← Selection reticle
    hud_callout/            ← Callout popups
    hud_hint/               ← Hint system
    hud_cpn_objective/      ← Campaign objectives
    pause/                  ← Pause screen
  controls/                 ← Reusable UI components
  shared/
    textures/               ← Unit icons, leader portraits, etc.
```

### ModData Override Path
Files in `ModData/` override ERA contents at the same relative path:
- ✅ Confirmed: triggerscripts, SCN files, objects.xml, tactics, scenariodescriptions.xml
- 🔲 Untested: `.gfx` Flash files, sound files, visual files

---

## Next Session Plan: King of the Hill

### Design
- 3-5 zone positions on Terminal Moraine (Tom to pick coordinates)
- Zone rotates every 90 seconds
- Score ticks +1 each check cycle (3s) while you hold the zone
- First to target score wins
- Visual marker: TBD (ShowMessage as fallback, healing zone if spawnable)

### All DBIDs confirmed available:
- CanGetUnits (2) — "any player units in zone?"
- TriggerActiveTime (7) — 90-second rotation timer
- MathInt (179) — score increment
- CompareInteger (14) — win check
- SetPlayerState (130) — victory/defeat
- ShowMessage (904) — zone change notification
- TriggerActivate (31) — trigger chain

### Tom's Homework Before Next Session
1. Remove reactor from Terminal Moraine SCN
2. Pick 3-5 zone coordinates using scenario editor
3. Install JPEXS Flash Decompiler, open `hud_skirm_objectivescreen.gfx`
4. Test `.gfx` ModData override (copy any .gfx to ModData path, see if game loads it)
5. `find extracted -name "*.swf" -o -name "*.gfx"` on other ERAs for more UI files

---

## Long-Term Roadmap

### Phase 1: King of the Hill (next session)
Working prototype on Terminal Moraine. PvP + accidental AI participation.

### Phase 2: Drag the Core (revised)
- Solve CreateSquad-in-loop crash (maybe use Move DBID=66 to reposition instead)
- Solve score display (ShowMessage or custom .gfx)
- Add towing to Warthog/Ghost via tactics files
- Magnetic pull using Move effect

### Phase 3: Full Mod Fork
- Fork LeaderOverhaulMod
- Add custom Forerunner leader
- Fix Flood AI
- Package game modes
- Custom Flash UI for score display
- Per-map configurations
- UI modding tools (JPEXS pipeline)

---

## Open Questions
1. Does `.gfx` override work from ModData? (Would unlock UI modding)
2. Why does CreateSquad crash in conditional trigger chains but work in Init?
3. Why does GetLocation fail on reactor squad? (Squad var might be stale after hitch)
4. Can Move (DBID=66) reposition a hitched unit?
5. Does SetCTFFlag work on non-flag ProtoSquads?
6. What is the second Towable object in the game?
7. Can we modify skirmishAI.triggerscript to teach AI custom behaviors?
