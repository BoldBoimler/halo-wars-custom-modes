# King of the Hill — Design Specification

## Overview
Classic Halo 2-style King of the Hill for Halo Wars DE skirmish mode. A single active zone rotates between 5 positions on the map. Players score by keeping units in the active zone. Contested zones (both teams present) award no points.

## Rules
- **5 zones** on Terminal Moraine, rotating in fixed order
- **90-second rotation** interval per zone
- **Scoring:** +1 point every 3 seconds while one team exclusively holds the zone
- **Contested:** if both teams have units in the zone, nobody scores
- **Win condition:** first team to **15 points** wins
- **Eligible units:** infantry, vehicles, aircraft only — **buildings do not count**
- **Team support:** 1v1, 2v2, 3v3 — score is per-team, not per-player

## Zone Positions (Terminal Moraine)

| Zone | Description       | X      | Y   | Z      |
|------|-------------------|--------|-----|--------|
| 0    | Map center        | 651.0  | 18  | 651.0  |
| 1    | West bridge       | 274.0  | 18  | 640.0  |
| 2    | East side         | 1028.0 | 18  | 640.0  |
| 3    | North midfield    | 651.0  | 25  | 400.0  |
| 4    | South midfield    | 651.0  | 25  | 900.0  |

Detection radius: **100 game units** per zone.

Layout forms a cross/plus pattern — center is neutral, zone 3 favors P1 (north), zone 4 favors P2 (south), zones 1 and 2 are flanks.

## Trigger Architecture

### Triggers (19 total for 1v1, more for team support)

| Trigger           | Type              | Purpose                                              |
|-------------------|-------------------|------------------------------------------------------|
| KothInit          | Always-true init  | Set Zone0Active=true, all scores=0, activate loops   |
| ScoringTick       | 3-second loop     | Activates all zone-check triggers each cycle         |
| Zone0_P1 – Zone4_P1 | Conditional    | Check zone active + P1 units present → score P1     |
| Zone0_P2 – Zone4_P2 | Conditional    | Check zone active + P2 units present → score P2     |
| Rotation0 – Rotation4 | Timed chain  | After 90s, flip zone booleans, notify players        |
| P1WinCheck        | Conditional       | CompareInteger P1Score >= 15 → SetPlayerState Won    |
| P2WinCheck        | Conditional       | CompareInteger P2Score >= 15 → SetPlayerState Won    |

### Contested Detection Pattern
Each zone has TWO check triggers (one per player/team). Both run CanGetUnits against the same zone position:
1. Zone0_P1 sets `Team1Present = true` if P1 units found
2. Zone0_P2 sets `Team2Present = true` if P2 units found
3. A scoring trigger then checks: `Team1Present AND NOT Team2Present → P1 scores` (and vice versa)
4. Both booleans reset to false at the start of each tick

### DBIDs Used (all proven)

| DBID | Name             | Purpose in KotH                      |
|------|------------------|--------------------------------------|
| 2    | CanGetUnits      | Detect units in zone radius          |
| 7    | TriggerActiveTime| 3s scoring loop, 90s rotation timer  |
| 14   | CompareInteger   | Win condition check (score >= 15)    |
| 31   | TriggerActivate  | Chain triggers together              |
| 102  | CopyBool         | Reset presence booleans              |
| 114  | CompareBool      | Check if zone is active              |
| 130  | SetPlayerState   | Set Won/Defeated                     |
| 179  | MathInt          | Increment score                      |
| 904  | ShowMessage      | Zone rotation announcements          |

### TriggerVar Types Needed

| Type        | Variables                                           |
|-------------|-----------------------------------------------------|
| Bool        | Zone0Active–Zone4Active, Team1Present, Team2Present  |
| Integer     | P1Score, P2Score, WinScore (=15), ScoreIncrement (=1)|
| Player      | Player1, Player2                                     |
| Vector      | Zone0Pos–Zone4Pos                                    |
| Float       | ZoneRadius (=100.0)                                  |
| ObjectType  | UnitFilter (to exclude buildings)                    |
| Trigger     | Refs for TriggerActivate targets                     |
| Time        | TickInterval (=3000), RotationInterval (=90000)      |
| Operator    | OpGreaterOrEqual, OpEqual                            |

## SCN Integration
All triggers embed in the SCN's own `<TriggerSystem>` block (not skirmishworld override — confirmed this doesn't work in maps mod context).

Terminal Moraine SCN TriggerSystem starts at line 416. New IDs must start above:
- NextTriggerVarID > 5521
- NextTriggerID > 299
- NextConditionID > 300
- NextEffectID > 475

## Future Enhancements

### Phase 2: Contested Logic + Teams
- PlayerList-based detection for 2v2 and 3v3
- CompareTeams (DBID=429) for team assignment at game start

### Phase 3a: Flash HUD
- Modify `hud_skirm_objectivescreen.gfx` to show KotH scores
- Depends on confirming .gfx ModData override works
- Community collaboration candidate (Flash/ActionScript expertise needed)

### Phase 3b: Zone Visual Marker
- Spawn a visible object (healing zone, teleporter pad, or custom) at active zone
- Destroy previous marker on rotation
- Goal: recreate Halo 2's holographic hill boundary
- Risk: Destroy effect may crash (crashed on hitched reactor, untested on free objects)

### Phase 3c: Announcer Voice
- PlaySoundFile (DBID=1084) on zone rotation events
- Requires custom audio files in ModData
- Audio format and path structure need investigation

### Phase 4: AI Participation
- Move (DBID=66) to send AI units toward active zone on rotation
- Based on CTF AI flag-retrieval pattern from capflagworld.triggerscript
