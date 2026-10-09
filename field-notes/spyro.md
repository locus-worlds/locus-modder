# Field notes: Spyro (Spyro the Dragon, DuckStation)

9–11 October 2026, SCUS-94228 (US; switched from the PAL SCES-01438 because public research
targets the US build), image SHA-1 `cf3ce6be…`; DuckStation 0.1-8946-g75ae7dead. Labels: LR =
LOCUS reproduced, doc = documented (CC0 decompilation TheMobyCollective/spyro-1 at `0d228ab7…`;
Spyro Scope, GPL-3.0, facts only), inf = inference.

## Route and why

**Live runtime** in DuckStation over its GDB server (Export Shared Memory is broken on macOS). A
new console generation and emulator, chosen to test whether Phase A's layer was generic: it was;
protocol, bus, world host and adapter format were untouched. A public decompilation gave struct
layouts; every field used was confirmed live. (src: docs/reference/LOCUS_Master_v1.7.md §10.5)

## Key facts (SCUS-94228)

| Item | Fact | Label |
| --- | --- | --- |
| `g_Spyro` | `0x80078A58`: position i32 x, y, z (z up, body centre 350–370 units above the floor) | LR |
| Facing | `0x80078B74`, low 12 bits, 4,096 per turn | LR |
| State | `0x80078AD0` (`m_State`): 0 stand, 5 jump, 11 charge, 15 glide, 14 hurt, 29 struggle | LR |
| Health | `0x80078BBC` (Sparx 3 gold → 0) | doc, then LR |
| Frame counters | `0x800758C8`, `0x80075AE8`, `0x80075E54` (60/s) | LR |
| Input | `0x800773C0` held buttons, active high; input lock `0x80078C48` | LR |
| Camera | `g_Camera` `0x80076DD0` (view matrix +0x14, position +0x28) | doc, LR |
| Collision | pointer `0x800785D4` → level block; 12-byte packed triangles, 4,096-unit grid of nested int16 tables; ground index +0x274 must be cleared after a swap | LR |
| Scale | 560 units per metre (90 % of Ratchet's drawn 1.41 m, the owner's judgement) | LR |
| Objects | `g_LevelMobys` `0x80075828`, `g_DynMobys` `0x80075890`, 0x58 bytes; freed the game's way; Sparx kept | LR |
| Model | `g_Models[0]`; 25 animations in Artisans; body/head/tail parts; vertex streams decoded from the renderer's assembly; one 4-bit VRAM page | LR |
| Damage | `m_DamageFlags` +0x2C: bit 0 hit, bit 10 last-point struggle; only hurts Artisans has loaded | LR |
| Lives | `g_SpyroLifeCount` `0x8007582C` kept ≥ 3 | LR |
| Attacks | charge = state 11; flame `+0x198` = 1, frames 10–32 of `+0x1A0` | LR |
| Effects | `g_Sparx` `0x80075898` (class 120); `g_Particles` `0x80075824` | LR |
| Sound | `g_Spu` `0x80075F08`, `m_ActiveSounds` 24 × 28 bytes; samples from SPU RAM in the play state | LR, heard by owner |

(src: investigations/spyro1-duckstation.md S2–S12; adapters/spyro1-spyro/package.json)

## Time

9 October: pin, access, reading, collision, model decoding, textures. 10 October: connector,
drawing, three owner playtests and their fixes. 11 October: sound. About three days for a new
emulator and engine, with a public decompilation. (src: investigations/spyro1-duckstation.md; MODLOG.md)

## What generalised

- GDB server as the memory route: read while running, block writes with the CPU stopped.
- Guided input session plus a recorder, then a decompilation's layout, then live confirmation.
- Savestates as the source of video and sound memory (RAM, VRAM, SPU RAM in the machine state).
- Polling animation state for pose (`morph`): cheap, no hook.
- Following the game's active-sound table: no hook, no writes.
- Freeing objects the game's own way.
- Re-installing everything after respawn; monotonic ticks.

## What did not

- The model decoder was transcribed from renderer assembly into Rust: exactly what adapters may
  no longer carry. Milestone M1 must show capture or a format description can replace it.
- The play state was made by the owner by hand; a ready-state recipe from power-on is still to do.
- The terrain block barely fitted Piazza (548 bytes spare); a window that follows him was needed.
- Sound and model were tied to the play state's level (Artisans); another level needs its own
  extraction and its own list of loaded animations.
