# Sound: follow the game's table, decode its samples

Step 12. The owner's decisions (11 October): his own action sounds only (no music, no level
ambience), samples extracted locally from the player's own copy, the emulator's audio never
unmuted or captured, positional, mixed by the world window. (src: docs/core/character-sound-2026-10-11.md)

## Shape of the answer

Every console game so far had the same two parts:

1. **Requests to follow**, in one of two shapes (adapter `sounds.table` or `sounds.buffer`; poll
   them, write nothing):
   - **a table** of the sounds playing, with a sound id, the object that owns the sound, a state,
     often a position and volume (Ratchet, Spyro);
   - **a command buffer** the game fills with sound requests and sends to the sound library every
     frame, usually double-buffered: records with a command, a request id, the sound's name or
     number, a position and volume, often no owner (Jak and Daxter). See
     [A command buffer](#a-command-buffer).
2. **Samples to decode**: a bank on the disc or in sound memory, in the console's ADPCM, which the
   connector decodes. The adapter describes where the bank is and its layout; when requests name
   their sounds, also the bank's name table (`samples.names`).

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

## A command buffer

PCSX2, vocabulary 0.5. (src: docs/core/sound-command-buffer-2026-10-09.md)

- **Find it**: a structure the game sends its sound library each frame. In a game with symbols
  (Jak 1's GOAL symbol table) its name says "sound" and "rpc"; otherwise `mem_search` the name of
  a sound you just caused (sounds are often requested by name) and walk back to the buffer that
  holds it. Two halves of the same size and a pointer that flips between them every frame mean
  double buffering: the half named by that pointer is being filled; read the other one.
- **Describe it** (`sounds.buffer`): `at` the structure (a `ptr` field), `halves` (the words
  holding each half's address), `filling` (the word naming the half being filled), in a half its
  record `count` and `records` (inline, or `records_pointer` when a word holds their address),
  `stride`, `max`, the record `fields`, `start` / `stop` tests on the command field, and which
  fields are the `sound` (a 16-byte `block` for a name, or a number), the `voice` (the request id:
  the same while the game repeats the request and in its stop), the `position` (scaled to the
  mapping's native units) and `volume` with `full_volume`.
- **Ownership**: when the records carry no owner, `owner.near_m` keeps requests placed within that
  many metres of him (after mapping). Sounds the game places at the listener (its "ear", often
  his position) pass that test too, so also give `only`: his own sounds by name (jump, landing
  on each surface, footsteps, attacks, hurt, death). An `owner.field` with `is` names him when the
  records do carry an owner.
- **Samples by name**: `samples.names` points at the bank file's name table (count, first entry,
  stride, name length) and `samples.bank_at` at the bank inside the file, so a request's name
  finds its sound. With `only`, the session's bank holds just those sounds.
- **Reliability**: a half lives about one frame after it is sent. `adapter_try`'s
  status (`session_status` → `trial.counters`) gives `sound_buffer_reads`,
  `sound_buffer_max_step` (game frames between two reads; 1 means none skipped),
  `sound_buffer_frames_skipped` (frames whose requests may have been missed), `sound_buffer_torn`
  (reads discarded because the halves swapped mid-read), `sounds_not_owned`, `sounds_missing` and
  `sounds_missing_names` (names he asked for that the bank lacks). A loaded machine stalls the
  reader; check the counters while his actions run, and repeat a test that skipped frames.

## Samples

- **PS1 SPU and PS2 SPU2 ADPCM** share the block format (16 bytes → 28 samples; shift, filter,
  loop-start, repeat and end flags). Decoded by the connector.
- **989snd banks** (`SBlk`): header counts, 12-byte sounds, grain programs (40-byte grains in R&C's
  and Jak 1's version; delays in 1/240 s), tones with centre note and fine tune; the sample rate follows from
  the note (OpenGOAL's reimplementation documents the format: facts only). The connector renders
  one sample per sound id by running the grain program once; random variants keep the first option.
  A bank file may start with something else (Jak 1's `.SBK` files: a table of sound names, then
  the bank further on): give `samples.bank_at`.
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
  0.5 s, none from silenced objects. For a command buffer also: `sound_buffer_max_step` 1 and
  no `sounds_missing_names` you expected to hear.
- Ask the player to play and listen: hear each; record the answer per sound. Until then the sound step
  is "built, not heard". (pitfall 45)
