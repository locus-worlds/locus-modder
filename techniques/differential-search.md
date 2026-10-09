# Finding the character: differential search with injected input

Step 3. Change one thing you control, snapshot memory before and after, keep the values whose
change matches, repeat with the opposite change, then confirm in another session. Inject the input
yourself (`input_script`): it is exact and repeatable, unlike a person at a pad.

Ratchet was found by watching floats during deliberate input and then a debugger write-watch;
Spyro by a recorder (a 16 KB window at 27 Hz and the full 2 MB every 5 s) during a guided session
with the owner, correlated with his input. Both were confirmed with a second session.
(src: docs/core/project-review-2026-10-11.md §2; investigations/spyro1-duckstation.md S2)

## Setup

1. A quiet spot: a research savestate where nothing else moves, the character idle on flat ground.
   Silence comes later (step 4); before that, prefer a calm corner of the first level.
2. Types to expect: PS2 games mostly little-endian `f32` positions in game units; PS1 games `i32`
   game units, fixed point 4,096 = 1.0, angles 4,096 or 256 per turn. Search all candidate types.
3. A free source of input: an attract demo plays recorded input (Spyro's position was first seen
   moving in the demo, before any injected input). (src: investigations/spyro1-duckstation.md S2)

## Recipes

| Field | Inject | Keep values that |
| --- | --- | --- |
| Frame counter | nothing, 1 s apart | rose by the frame rate (60 NTSC); confirm with `mem_watch` over 400 samples |
| Position (horizontal) | hold right N frames; then left N frames | rose then fell (or fell then rose), by similar amounts; three neighbours (x, y, z) |
| Position (vertical) | jump in place | rose then returned to the same value; this is the up axis and its sign |
| Facing | turn on the spot (stick in a circle) | changed monotonically and wrapped; compare with the direction of travel when walking (Spyro: 0.38 rad mean error, turning lag) |
| Action / animation | idle, walk, jump, attack, each separately | constant while idle; a distinct value per action; record value per action |
| Health | take one hit (an enemy, a fall, a hazard) | dropped by one unit and stayed |
| Input word | press one button at a time | the bit set while held (active high or low) |
| Camera | move only the camera; then only the character | a matrix (rows of unit vectors) a few metres away whose forward axis points at him; changed only with the camera |

Narrow with `mem_diff(a, b, filter)` and `mem_search(type, relation, within snapshots)`; stop when
fewer than about 20 candidates remain, then `mem_watch` them during a new input script.

## Authoritative or a copy?

Games keep copies (render, camera, collision working copies). To tell the owner apart:

- **Write and watch the game react**: write the position 2 m up; the authoritative field makes him
  fall and land under his own code (Spyro: fall animation, landed). A copy is overwritten next frame.
  Approval needed (`mem_write`). (src: investigations/spyro1-duckstation.md S1 continued)
- **Find the writer**: `code_xrefs` on the address, `breakpoint` on write (DuckStation), or static
  analysis. Ratchet's candidate at `0x13F3D0` was a collision working copy with transient +0.7
  offsets; his object record's `+0x10` (`0x1845E90`) is the position used in play.
  (src: investigations/ratchet.md "Fresh-session restoration observed"; docs/HANDOFF.md "Game memory map")
- Never smooth or drop odd samples; explain them.

## Records reached through pointers

When the character lives on a heap (objects allocated per level, processes):

1. Find the field by value as above in session A.
2. `mem_pointer_scan(target)` for chains from static addresses (main executable data) or, in a
   runtime with a symbol table, from a symbol.
3. Repeat in session B (fresh boot) and after a level change or respawn; keep the chains that still
   resolve.
4. Add a **guard** to the chain: a value that identifies the record (class pointer, type tag, a
   constant), so a stale chain is refused instead of read.

The hero is often entry 0 of the object table (Ratchet) or a fixed global (Spyro's `g_Spyro`).
Decompilations and symbol maps give struct layouts; confirm each offset live before relying on it.
(src: investigations/spyro1-duckstation.md S6)

## Confirm

- Second session, different script (other directions, other durations). Every field follows.
- Every action value from two inputs, absent for every other action (see pitfall 8).
- Record per field: address or chain with guard, type, units, axis and sign, value per action,
  label, the two confirmations' trace ids.
