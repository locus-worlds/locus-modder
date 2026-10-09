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
6. Trees and pointer slots: some engines link objects through a slot that holds the object's
   address (a "pointer to a pointer"), and keep children and siblings (a process tree). Read one
   link by hand: if the word at the link is not a record but points at a word that is, the link
   takes two reads. Check the self-slot (a record whose slot points back at it).

Decompilations name these tables; confirm live.

## Declaring them

| Table | `memory.objects.<name>` |
| --- | --- |
| Array | `at`, `stride`, `count` (and `end` or `stop`) |
| List | `next: "+0x10"`: the word at +0x10 is the next record |
| List through slots | `next: ["+0xC", "+0x0"]`: the word at +0xC, then the word it points at (+0x0) is the next record; up to 4 reads (vocabulary 0.6) |
| Tree | `next` (the sibling) and `child: ["+0x10", "+0x0"]` (the first child), walked depth first from `at`: a record, its children, then its sibling (vocabulary 0.6) |
| A language's null | `null: ["0x…"]`: words that end a chain besides 0 (a Lisp's false symbol, a sentinel) |

Every walk reads each record once (a link back to a record already read ends that branch) and at
most `count` records; a link outside the adapter's regions ends it. Records start at the address
the chain gives; their fields are `+0x…` from there. Choose what you keep with the table's `alive`
test and `named` selections (`flags_any` / `flags_none` on a category mask, `in` on a class or
type word), and check the walk with `mem_read` on two or three records before trusting it.

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
  the rest. The adapter declares `world.silence` with `keep_classes` (provisional names); a
  tree's records are filtered by `alive` and `named` tests on their own fields (a category mask,
  a class word). Silencing a walked list or tree is DuckStation's `free` only so far; on PCSX2,
  declare the table to find and follow objects, and record silencing as not yet supported.
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
