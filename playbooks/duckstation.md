# DuckStation playbook (PS1)

What LOCUS learnt running Spyro the Dragon (SCUS-94228) in DuckStation on macOS, 9–11 October 2026.
The app's `connector-duckstation` does the plumbing. Facts with a `src:` are LOCUS reproduced
unless marked otherwise.

## Version

- Stock DuckStation **0.1-8946-g75ae7dead**, universal (x86_64 + arm64). Record what `runtimes`
  reports; another build may differ in every item below. (src: investigations/spyro1-duckstation.md S0)

## Profile (what `session_start` builds)

- DuckStation reads `$HOME/Library/Application Support/DuckStation/settings.ini`; the session runs
  with `HOME=<session>/home`, holding a **copy of the player's settings**. A settings file without
  `[Main] SettingsVersion` is reset to defaults (setup wizard, settings lost), so the player must
  have run DuckStation once. A relative BIOS search directory is made absolute.
- Forced keys: no save state on exit, no power-off confirmation, not paused at start, on focus loss
  or on controller disconnection, GDB server on a loopback port, muted, memory cards `None`,
  achievements and update check off, file logging on, no on-screen messages, `[InputSources] SDL`
  on only when a pad is routed; Start and Select unbound on a shared pad.
  (src: connectors/duckstation/src/runtime.rs; investigations/spyro1-duckstation.md S1, S7, S10)
- Launch: `-batch -nogui -statefile <state>` (exit when the game stops; the state names its disc);
  for a fresh boot `-batch -nogui -fastboot -- <cue>`. The disc serial is read from the image's own
  `SYSTEM.CNF` before launch. macOS's saved window state for DuckStation is updated by sessions
  (outside the profile; harmless).

## Memory: the GDB server

- **Export Shared Memory does not work on this macOS build** (empty object name). The GDB server
  is the route. (src: investigations/spyro1-duckstation.md S1)
- On connect the CPU stops (`S02`); `c` continues and the game runs **while answering reads**;
  `0x03` halts (`S00`); `D` detaches. Stray stop replies arrive and must be survived.
- Cost: a 4 KB `m` read about 0.1 ms; a full 2 MB sweep 0.05 s; in one second 131 of 512 4-KB
  blocks changed. Reading `g_Spu`'s 808 bytes once per game frame did not disturb the game.
- **Writes (`M`) work while running** for data (Spyro's position written; his code fell and landed).
  **Block writes are made with the CPU stopped** (collision tables, freeing objects); a terrain
  rewrite stopped his game for 14–82 ms. (src: investigations/spyro1-duckstation.md S1 continued, S10)
- **Code writes may not reach DuckStation's recompiler** (untested): there are no PS1 code hooks
  yet; the connector polls. `breakpoint` (GDB) halts the CPU: research only, never in play.
- Only one client: the app's. A research probe once connected to the owner's running session
  (halted and resumed his CPU, then failed); sessions now refuse to run beside play.
  (src: MODLOG.md, A7 Spyro third playtest fixes)

## Hidden play

- This build has **no null renderer** (Automatic, Metal, Vulkan, Software). The app hides the
  process through `NSRunningApplication.hide` for the whole session (DuckStation shows itself again
  while loading; `open -j` did not help; the System Events route needs a permission rebuilt apps
  lose). Hidden it ran at 59.5–59.8 game frames per second. (src: investigations/spyro1-duckstation.md S9, S10)

## Controllers

- Research sessions run with SDL input off and inject input. In play, a pad can be shared between
  two characters: the other character's connector holds the game's own input lock while it has the
  pad (Spyro: `m_noGamepadUpdateFrames` at `0x80078C48`). Look for such a lock in step 4.
  (src: MODLOG.md, A7 Spyro step 1 wired)
- The game's input word is an easy first find: Spyro's `0x800773C0` low 16 bits are the held
  buttons, active high (Cross `0x0040`, Square `0x0080`, Circle `0x0020`, Triangle `0x0010`,
  Start `0x0800`, Up `0x1000`, Right `0x2000`, Down `0x4000`, Left `0x8000`); its high half was
  `0xFFFF` in play and `0x0000` while the attract demo played recorded input. Writing a button into
  the game's pad buffer (Spyro: `g_Pad` `0x80077378`) works as research input.
  (src: investigations/spyro1-duckstation.md S2, S10)

## Savestates

- File: header `DUCC`, version `0x51`; a zstd screenshot (at `0x140`) and a **zstd machine state**.
  In the machine state: main RAM from `0x1A62`; **VRAM** (1024 × 512 × 16-bit) 990 bytes after RAM
  ends; sections written with markers (u32 length + name); **SPU RAM** is the 512 KB just before
  the `"MDEC"` marker that follows the `"SPU"` marker. `state_extract` gives `ram`, `vram`, `spu`.
  (src: investigations/spyro1-duckstation.md S6, S8, S12)
- Offsets inside the machine state are build-specific (they follow DuckStation's own `DoState`
  order). If a new build is used, check that RAM in the state matches a live snapshot of the same
  scene (Spyro: 456 of 512 pages identical) before trusting VRAM or SPU offsets.

## PS1 facts that shape adapters

- RAM is 2 MB at `0x80000000` (KSEG0 addresses); the scratchpad is separate. Fixed point is
  usually 4,096 = 1.0; angles are often 4,096 or 256 per turn (Spyro used both). Positions are
  integers in game units. (src: investigations/spyro1-duckstation.md S2, S8)
- The GTE takes vertices as (VX, VY, VZ); renderers often swizzle axes before it (Spyro:
  (−y, −z, x)). A renderer that unpacks into the scratchpad just before the GTE is the capture
  point for route 1. (src: investigations/spyro1-duckstation.md S6; docs/core/locus-modder-design-2026-10-11.md §5.2)
- VRAM textures: 4-bit or 8-bit pages with CLUTs; PS1 modulation is texel × vertex colour ÷ 128,
  colour 0 transparent. The connector decodes VRAM and CLUT layouts.
- **SPU**: 24 voices, PS1 ADPCM (16-byte blocks of 28 samples; shift 13–15 acts as 9; filters 0–4;
  loop-start, repeat and end flags). Pitch 4,096 = 44,100 Hz. A game's sound driver usually keeps
  its own table of active sounds in main RAM: follow that, not the SPU registers.
  (src: investigations/spyro1-duckstation.md S12)
- The frame counter rises 60 per second in NTSC games (50 in PAL ones: inference).

## Respawn and level reload

- A respawn can re-initialise the level: frame counters restart, level objects come back, the
  level's own collision returns. Detect it (counter went backwards, the first level object alive
  again) and reinstall. Source ticks sent to the world must stay monotonic across restarts (the
  world refused Spyro's states after a respawn until ticks carried an offset).
  (src: investigations/spyro1-duckstation.md S10)
