# Sound: follow the game's table, decode its samples

Step 12. The owner's decisions (11 October): his own action sounds only (no music, no level
ambience), samples extracted locally from the player's own copy, the emulator's audio never
unmuted or captured, positional, mixed by the world window. (src: docs/core/character-sound-2026-10-11.md)

## Shape of the answer

Every console game so far had the same two parts:

1. **A table to follow**: the game's own record of sounds it asked to play, with a sound id, the
   object that owns the sound, a state, often a position and volume. Poll it once a game frame or
   faster; write nothing.
2. **Samples to decode**: a bank on the disc or in sound memory, in the console's ADPCM, which the
   connector decodes. The adapter describes where the bank is and its layout (a format description).

Publish a trigger for every new entry whose owner is him, his items or his companions, at his
mapped position, with an event id; publish a stop for looping sounds when they end or are
replaced. (src: docs/core/character-sound-2026-10-11.md "Bank and trigger contract")

## Find the library and the table

- **Strings first**: the sound library often names itself (R&C's IOP RAM: "989snd (c)2000, 2001
  Sony…"). For a PS1 game the decompilation or the code that writes SPU registers leads to the
  driver's table.
- **The table**: fixed-size slots in main RAM (prefer the main executable's data over level code).
  Start a known sound (jump) and `mem_diff` before/after: a slot gains an id, an owner pointer equal
  to his record, a state.
- **Ownership**: the owner field equals his object address (or his item's or companion's).
- **States**: learn the slot life cycle (Ratchet: 7 requested, 1 sent, 2 playing, 6 stopping, 0
  free; Spyro: flags 1 used, 2 keyed on, 0x40 freeing, 0x100 looping).

| | Ratchet (989snd, PS2) | Spyro (PS1 SPU) |
| --- | --- | --- |
| Table | `0x13E5C0`, 30 slots × 0x70: handle +0x00, state +0x04, definition +0x08, bank sound index +0x0C, volume +0x10, pitch bend +0x14, owner +0x18, position +0x20 | `g_Spu.m_ActiveSounds` at `0x80075F30`, 24 × 28 bytes: owner +0, volume +4, pitch +8, id +0xD, flags +0xE |
| Read | 7 words per slot + frame counter, every 5 ms (211 words) | 808 bytes of `g_Spu`, once a game frame, no halt |
| Owners kept | hero object; objects of class 71 (wrench) | `&g_Spyro`, `*g_Sparx` |
| Bank | the level's 989snd bank on the disc (`SBlk`; one per level, 19 on the disc) | the level's samples in SPU RAM from the play state (from `0x1010`), with the level's 20-byte definitions |
| Ids | class sound index → bank index (his 64 sounds: bank 20–81; wrench 95–99) | frame sounds in his animation frames (top byte of the frame word) + named `SoundTable` entries |

(src: docs/ratchet/06-sound/ratchet-sound-2026-10-11.md; investigations/spyro1-duckstation.md S12)

## Samples

- **PS1 SPU and PS2 SPU2 ADPCM** share the block format (16 bytes → 28 samples; shift, filter,
  loop-start, repeat and end flags). Decoded by the connector.
- **989snd banks** (`SBlk`): header counts, 12-byte sounds, grain programs (40-byte grains in R&C's
  version; delays in 1/240 s), tones with centre note and fine tune; the sample rate follows from
  the note (OpenGOAL's reimplementation documents the format: facts only). The connector renders
  one sample per sound id by running the grain program once; random variants keep the first option.
- **Sound memory in a savestate** (PS1): the bank is whatever the level loaded; each definition's
  address, length and end flags must agree (Spyro: every sample ended one block before the next).
- Choose disc or savestate by where the format is clearer; pin the bank by length and SHA-256 and
  refuse any other.

## Labels

Map ids to actions by which animation or state plays them (Spyro: frame sounds of animation 5 are
likely his jump; Ratchet: bank 47 in the wrench animations). Unknown ids stay "unknown": the whole
level's bank is shipped locally, Piazza only plays what is triggered.

## Check it

- `sound-triggers`: jump, attack, hurt in the Proving Ground; `bus_trace` shows his ids within
  0.5 s, none from silenced objects.
- `ask_player(play_and_report)`: hear each; record the answer per sound. Until then the sound step
  is "built, not heard". (pitfall 45)
