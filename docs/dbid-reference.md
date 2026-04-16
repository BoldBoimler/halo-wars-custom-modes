# Trigger DBID Reference — Confirmed & Tested

All DBIDs confirmed from working scripts and in-game testing.

## Conditions

| DBID | Name               | Status  | Inputs → Outputs                                  | Notes                                      |
|------|--------------------|---------|----------------------------------------------------|---------------------------------------------|
| 2    | CanGetUnits        | ✅      | FilterPlayer, FilterPlayerList, FilterObjectType, FilterLocation, FilterDistance, FilterBoxForward/HalfX/Y/Z, FilterUnitList → FoundUnits, FoundUnitCount | Must provide ALL 10 optional inputs (null vars for unused). Proven for zone detection |
| 4    | CanGetSquads       | ❌      | Similar to CanGetUnits                              | Crashed in testing — use CanGetUnits instead |
| 7    | TriggerActiveTime  | ✅      | CompareType (Operator), CompareTime (Time in ms)    | Proven for 3-second loop timing              |
| 14   | CompareInteger     | ⚠️      | FirstInteger, CompareType, SecondInteger             | Used in CTF for score check, not directly confirmed |
| 80   | IsAlive            | —       | TestUnit/UnitList/Squad/SquadList                    | Used in CTF for flag carrier                 |
| 114  | CompareBool        | —       | FirstBool, CompareType, SecondBool                   | Used in CTF                                  |
| 131  | PlayerInState      | —       | Player, PlayerState                                  | Condition version of SetPlayerState          |
| 429  | CompareTeams       | —       | (from CTF)                                           | Team detection at game start                 |
| 974  | CanGetDesignSpheres| —       | Type → CenterPointList, RadiusList                   | Used for creep camps                         |

## Effects

| DBID | Name               | Status  | Key Inputs                                         | Notes                                        |
|------|--------------------|---------|----------------------------------------------------|----------------------------------------------|
| 31   | TriggerActivate    | ✅      | Trigger (via TriggerVar of Type="Trigger")          | Trigger chaining — proven                    |
| 33   | PlaySound          | —       | Sound, Queue                                        | Found in campaign triggers                   |
| 36   | CreateSquad        | ⚠️      | ProtoSquad, CreateLocation, Facing, SquadOwner → CreatedSquad | Works in Init, crashes in rapid loop |
| 38   | Destroy            | ❌      | DestroyUnit/Squad/etc.                               | Crashed on reactor — may not work on hitched |
| 66   | Move               | 🔲      | Squad, TargetLocation, AttackMove, QueueOrder        | Not yet tested — key for AI and magnetic pull|
| 98   | CopyInt            | —       | IntSource → IntCopy                                  | Used in CTF                                  |
| 102  | CopyBool           | —       | BoolSource → BoolCopy                                | Used in CTF                                  |
| 112  | GetUnits           | —       | Same filters as CanGetUnits                          | Effect version                               |
| 130  | SetPlayerState     | ⚠️      | Player, PlayerState (Won/Defeated/Playing)           | Used for win conditions, not directly tested |
| 179  | MathInt            | ⚠️      | FirstInt, MathOperator (Add), SecondInt → ResultInt  | Score increment — used but not visually confirmed |
| 189  | GetLocation        | ❌      | Unit, Squad, Object → OutputLocation                 | Failed silently on reactor squad             |
| 277  | SetResources       | ✅      | Player, Resources (Cost format: 0=sup,1=react,2-4=0)| Proven. Replaces ALL resources, doesn't add  |
| 392  | LaunchScript       | —       | ScriptName + External vars                           | Used for creep/crate scripts                 |
| 658  | TimerCreate        | 🔲      | Various → timer ID                                   | Used in CTF hold timer, untested by us       |
| 659  | TimerDestroy       | 🔲      | TimerID                                              | Used in CTF                                  |
| 661  | TimerIsDone        | 🔲      | TimerID                                              | Used in CTF                                  |
| 729  | PlayChat           | —       | ChatSound, ChatString, ChatSpeaker, Duration, etc.   | Campaign dialogue                            |
| 904  | ShowMessage        | 🔲      | Player, PlayerList, Sound, Message, Duration          | CTF notifications — untested by us           |
| 943  | GetNPCPlayersByName| —       | NPCPlayerName, NPCPlayerState → NPCPlayerList        | Creep player detection                       |
| 1076 | SetCTFFlag         | —       | Squad                                                | Marks squad as CTF flag with engine behavior |
| 1084 | PlaySoundFile      | 🔲      | (params unknown)                                     | Used in CTF — untested by us                 |

## Status Legend
- ✅ Proven in-game by us
- ⚠️ Used in our scripts but not independently confirmed
- ❌ Tested and failed / crashed
- 🔲 Found in existing scripts but not yet tested by us
- — Known from reading scripts, not attempted

## TriggerVar Types

Types confirmed from working scripts:

| Type         | Notes                                              |
|--------------|----------------------------------------------------|
| Player       | Value "1" = Player 1, "0" = Gaia                  |
| PlayerList   | List of players (for team operations)              |
| PlayerState  | Won, Defeated, Playing                             |
| Integer      | General purpose                                    |
| Float        | General purpose                                    |
| Bool         | true/false                                         |
| String       | Text values                                        |
| Time         | Milliseconds                                       |
| Operator     | Comparison operators                               |
| Vector       | X,Y,Z position                                     |
| VectorList   | List of positions                                  |
| Unit/UnitList| Individual units and lists                         |
| Squad/SquadList | Individual squads and lists                     |
| Object/ObjectList | Individual objects and lists                  |
| ObjectType   | Game object type tags (Building, Unit, Towable)    |
| ProtoSquad   | Squad prototype name                               |
| ProtoObject  | Object prototype name                              |
| Cost         | Format: `0=supplies,1=reactors,2=0,3=0,4=0`       |
| Trigger      | Value = target trigger ID (for TriggerActivate)    |
| Team/TeamList| Team groupings                                     |

## Critical Rules

1. **All optional inputs need TriggerVars** — even null/unused inputs must reference a TriggerVar with `IsNull="true"`
2. **CRLF line endings mandatory** — LF crashes the game
3. **XML encoding: us-ascii**
4. **Next*ID header values must exceed all used IDs**
5. **EvalLimit="0" + `<And />` = infinite loop** — use EvalLimit="1" for fire-once
6. **TriggerActivate target is stored in a TriggerVar**, not referenced directly
