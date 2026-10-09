# Object tables: isolate the level, keep what is his

Step 4 (and steps 8 and 11). Games keep their objects in a table: an array with a fixed stride, or
a linked list. You need it to silence the level, to find his weapons, companions and effects, and
to borrow objects as proxies.

## Find it

1. Start from the hero record (step 3). In many engines the hero is an object like the others:
   look for identical-looking records at a fixed distance before and after it (Ratchet: entry 0 of
   the moby array).
2. Stride: the distance between two records with the same layout (class pointer at the same
   offset, a position at the same offset).
3. Per-entry fields to identify: class (number or pointer), state, position, update routine
   pointer, owner, scale, collision link. Compare entries of known objects (a crate, an enemy).
4. Bound: count live entries; find what marks the end (empty slots, a count, a list terminator).
5. Lists: follow `next` pointers from a global head; check the list closes.

Decompilations name these tables; confirm live.

## Examples

| | Ratchet (R&C, PS2) | Spyro (PS1) |
| --- | --- | --- |
| Table | moby array `0x1845E80`, stride `0x100`, hero = entry 0; 308 live entries then empty slots | `g_LevelMobys` `0x80075828`, `g_DynMobys` `0x80075890`, `0x58` bytes each |
| Fields | +0x00 bounding sphere (×1024), +0x10 position, +0x20 state, +0x24 class pointer, +0x2C scale, +0x74 update routine, +0xA6 class number | +0x48 state (alive ≤ 127), +0x34 region; collision chain `0x80075778` |
| Silence | clear the update routine and park the object outside the world's crop | the game's own free: unlink from the region's collision chain, state `0xFD` |
| Kept | hero (class 0), wrench (class 71) | Sparx (first dynamic object) |
| Count | 308 frozen and parked | 174 freed, 66 unlinked from collision |

(src: docs/piazza/piazza-integration-plan.md "Invisible backend"; investigations/spyro1-duckstation.md S9; MODLOG.md, Mario proxy height)

## Silence by allow-list

- Keep by class: the hero, his weapons and held items, projectiles, companions, effects. Silence
  the rest. The adapter declares `world.silence` with `keep_classes` (provisional names).
- Prefer the game's own removal (its free routine's effect) to overwriting.
- Stop exactly at the bound and guard each entry before writing (pitfall 17).
- Thawing a frozen object hung Ratchet's game: build the silenced state from an unsilenced one.
- Objects spawned later by other paths are not covered: watch the table for 60 s of play after
  silencing and record anything new.

## Follow, never cache

Objects can be destroyed and recreated in other slots (Ratchet's wrench, 305 → 300 after respawn).
Find each followed object by class (and owner, when several share a class) with guards, rediscover
it when a guard fails (a bounded scan; Ratchet: 512 slots, retry every 200 ms), and keep a
connector-side cache only as a hint. (src: MODLOG.md, wrench respawn)

## Objects as tools

- **Proxies for other characters** (step 11): a parked object of a solid class, moved to the other
  character's body each tick. Ratchet's game: crates (class 500) rescaled through `+0x2C` (a crate
  is a 1 m cube at scale 1/24; the game's collision follows the scale) and stacked: three 0.6 m
  cubes for a 1.6 m body. (src: MODLOG.md, Mario proxy height)
- **Particles and effects** are often a separate table (Spyro's `g_Particles`, 32-byte entries).
  (src: investigations/spyro1-duckstation.md S11)
