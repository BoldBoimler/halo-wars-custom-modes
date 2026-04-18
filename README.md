# Halo Wars DE — Custom Game Modes

Custom skirmish game modes for Halo Wars: Definitive Edition, built through triggerscript modding, SCN modifications, and objects_update.xml patching.

## Game Modes

### King of the Hill (V3 — multi-map)
Classic Halo 2 Crazy King-style territory control. Hold the active zone to score — first team to 200 wins. The hill randomly rotates between 5 preset positions every 90 seconds with audio/visual notifications. Supports 1v1, 2v2, and 3v3.

**What works:**
- **18 maps supported** — per-map hill coordinates selected via IsMap branching on Init
- 5 hills per map, **randomly selected** each rotation (RandomCount 1..5 + dispatch chain)
- Scoring via hijacked CTF HUD (setCTFCount)
- AI pursues the active hill via SetCTFFlag engine behavior
- Covenant doughnut ground marker moves between hills via SetPosition
- Sound, on-screen message, and minimap flare on each rotation
- Win condition triggers game-over screen

**Supported maps:**
Terminal Moraine, Chasms, Tundra, Blood Gulch, Labyrinth, Fort Deen, Beacon Hill, Beasley's Plateau, The Docks, Red River, Pirth Outskirts, Release, Repository, Frozen Valley, Barrens, Glacial Ravine, Exile, Crevice.

**Known issues:**
- Orphan sentinel squads accumulate at each rotation (engine protects CTF flag squads from Kill/Destroy)
- Stock CTF flag spawns from ERA EditorData (~60-90s) — can't be prevented, only redirected
- "Capture 3 Flags" welcome message from stock CTF
- No contested logic yet — both teams can score simultaneously
- No no-repeat guard — random selector can pick the same hill twice in a row (~1 in 5 rotations)
- Minimap flares and revealers fire for team leader only on each rotation (teams share FoW in HW, so vision is shared; flare is cosmetic)
- Terminal Moraine Hill 5 is a geometric mirror guess — may need tuning in playtest
- Beacon Hill internal name unverified (may actually be Memorial Basin's internal name)

**Built on CTF mode** — required for setCTFCount HUD and SetCTFFlag AI lure. Skirmish mode was tested and rejected (no HUD, no flag spawning).

#### Roadmap
- Contested detection (both teams present = nobody scores)
- No-repeat guard on random selector (MathInt bump pattern)
- Visual hill marker improvements (PowerGrant healing beam, obstruction ring)
- Custom welcome message and announcer voice
- Team-wide minimap flares and per-teammate revealers (cosmetic polish)
- Verify per-map IsMap internal names (especially Beacon Hill)

### Drag the Reactor (on hold)
CTF variant using the reactor core from campaign mission 14. Hitch the reactor with your Elephant and tow it to your base to score.

**Status:** Proven primitives (CreateSquad reactor, CanGetUnits zone detection, scoring loop) but blocked by CreateSquad-in-loop crashes and inability to unhitch/reset the reactor. On hold pending KotH completion.

#### Open Questions
- Can Move (DBID=66) reposition a hitched unit?
- Does SetCTFFlag work on non-flag ProtoSquads?
- Can we modify other units to add hitch mode (Wraith, Ghost)?

## Installation

1. Copy the `ModData/` folder to your Halo Wars mod directory:
   - **Steam Deck:** `/home/deck/HaloWarsMods/maps/ModData/`
   - **Windows:** Create a folder and point `ModManifest.txt` to it
2. Add the mod path to `%localappdata%\Halo Wars\ModManifest.txt`
3. Launch Halo Wars DE and select **Capture the Flag** on Terminal Moraine (KotH uses CTF mode)

## Project Structure

```
ModData/                              # Mirrors game file structure
├── data/
│   ├── triggerscripts/
│   │   └── capflagworld.triggerscript  # KotH game logic (CTF mode override)
│   ├── objects_update.xml              # Unit modifications (flag behavior)
│   └── tactics/                        # Unit behavior overrides
└── scenario/skirmish/design/
    └── terminal_moraine/               # Map-specific SCN modifications

docs/
├── koth-design.md                      # King of the Hill full design spec
├── drag-the-reactor-design.md          # Drag the Reactor design spec
├── dbid-reference.md                   # Confirmed trigger DBIDs
└── session-notes/                      # Development session summaries

tools/
└── scenario-editor/                    # Browser-based SCN editor
```

## Key Technical Discoveries

### EditorData Dual Execution
The engine loads `<EditorData>` TriggerSystem from the ERA **independently** of ModData overrides. CommentOut changes in the outer Triggers block do NOT affect EditorData copies. Stock CTF triggers are unstoppable — only redirectable via runtime variable manipulation (CopyLocation, CopyFloat).

### CTF Flag Squad Protection
`game_CTF_monitor_01` squads have deep engine protection in CTF mode:
- Destroy (DBID=38) silently fails
- Kill (DBID=37) crashes the game
- Teleport (DBID=354) moves entity but visual stays behind

### Useful Effects Discovered
| DBID | Effect | Use Case |
|------|--------|----------|
| 35 | CreateObject | Spawn scenery/decoration objects at runtime |
| 537 | SetPosition | Instant reposition of Objects (no pathfinding) |
| 99 | CopyLocation | Runtime Vector variable overwrite |
| 103 | CopyFloat | Runtime Float variable overwrite |
| 285 | Revealer | Reveal fog of war at location for a player |
| 135 | FlareMinimapNormal | Ping a player's minimap |
| 1076 | SetCTFFlag | Engine-level AI pursuit of flagged squad |
| 1083 | setCTFCount | Drive CTF HUD score widget (CTF mode only) |

## Development

KotH logic lives in `capflagworld.triggerscript`, overriding the stock CTF world script via ModData. All modifications are additive — stock triggers are preserved, KotH triggers use IDs 250+ and vars 2000+.

### Key Constraints
- Files must use **CRLF line endings** — Unix LF crashes the game
- XML encoding must be **us-ascii**
- All trigger, effect, condition, and var IDs must be unique within a file
- Header counters (NextTriggerVarID, NextTriggerID, etc.) must exceed all used IDs
- Output tags must use **short closing form** (`</o>` not `</Output>`)
- EditorData section from ERA cannot be overridden — design around it

### Testing
- Primary test platform: Steam Deck with HW:DE via Steam
- Dev machine: Mac, file transfer via scp
- Launch CTF mode on Terminal Moraine for KotH testing
- 7777 supplies diagnostic confirms Init trigger fires (remove for release)

## Credits

Developed by Tom van Ryswyk (BoldBoimler) with Claude (Anthropic).

- **GitHub:** https://github.com/BoldBoimler/halo-wars-scenario-editor
- **HW:DE Discord** — active modding community

## License

MIT