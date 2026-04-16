# Halo Wars DE — Custom Game Modes

Custom skirmish game modes for Halo Wars: Definitive Edition, built mostly through triggerscript modding and SCN modifications.

## Game Modes

### King of the Hill (in development)
Classic Halo 2-style territory control. 5 zones rotate every 90 seconds on Terminal Moraine. Hold the active zone to score — first to 15 wins. Contested zones (both teams present) award no points.
#### Additional Thoughts
* Pull soundclips and get the OG announcer voice integrated
* Recreate holographic hill marker
* Use Keep Away mode as a template for AI play (AI already sends units to the flag, just have it send them to the hill)

### Drag the Reactor (in development)
CTF variant using the reactor core from the campaign (level 14). Hitch the reactor with your Elephant and tow it to your base to score. 
#### Additional Thoughts
* Modify other units to add hitch mode (e.g. Wraith, maybe a ghost because it'd be funny)
* Scoring plan:  Modified Keep Away mode (timer based) or CTF based (capture zone)
* Flag reset dynamics:  Core explodes?  Flag carrier explodes?  Can't figure out how to detach the core after hitching to it

## Installation

1. Copy the `ModData/` folder to your Halo Wars mod directory:
   - **Steam Deck:** `/home/deck/HaloWarsMods/[modname]/ModData/`
   - **Windows:** Create a folder and point `ModManifest.txt` to it
2. Add the mod path to `%localappdata%\Halo Wars\ModManifest.txt`
3. Launch Halo Wars DE and select Terminal Moraine in skirmish

## Project Structure

```
ModData/                          # Mirrors game file structure — copy directly to mod folder
├── scenariodescriptions.xml      # Map registration (required for ModData SCN loading)
├── data/
│   ├── triggerscripts/           # Game mode logic
│   └── tactics/                  # Unit behavior overrides
└── scenario/skirmish/design/
    └── terminal_moraine/         # Map-specific SCN modifications

docs/                             # Design documents and references
├── koth-design.md                # King of the Hill full design spec
├── drag-the-reactor-design.md    # Drag the Reactor design spec
├── dbid-reference.md             # Confirmed trigger DBIDs
└── session-notes/                # Development session summaries

tools/                            # Helper scripts
```

## Related Projects

- [Scenario Editor](https://github.com/BoldBoimler/halo-wars-scenario-editor) — Browser-based map editor for HW:DE `.scn` files
- Mod Launcher — Python script for selecting mods at runtime

## Development

All game mode logic lives in triggerscripts embedded in SCN files or overriding ERA defaults via ModData. See `docs/` for design specs and the DBID reference.

Key constraints:
- Files must use **CRLF line endings** (Unix LF crashes the game)
- XML encoding must be **us-ascii**
- All trigger IDs must be unique within a file
- NextTriggerVarID/NextTriggerID/etc. in headers must exceed all used IDs

## License

MIT
