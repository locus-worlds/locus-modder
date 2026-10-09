# Collision routes: putting his own code on LOCUS ground

Step 5. His movement code must collide with LOCUS's world, not with his level. Three routes, in
order of preference (design §5.2; Master §27.4.3):

| Route | How | Fidelity | Status |
| --- | --- | --- | --- |
| **Substitution** | Write LOCUS's world into the game's own collision format, in place, from a format description; generic builders (grids, trees, quantised vertices) | Highest: his own collision code decides | Proven twice in Rust (R&C tree, Spyro grid); from a description unproven until M1 |
| **Answer hook** | LOCUS's `answer_query` template at the routine that fills the game's collision cache near a query: while LOCUS answers, it writes LOCUS's triangles (records in the cache's own layout) instead of the level's | His own collision code decides, on LOCUS's triangles; cost: a box test per buffered triangle per fill, inside the emulated CPU | **Runs on PCSX2** (cache form, vocabulary 0.5); first proven on Jak 1 (record: docs/core/jak-collision-answer-2026-10-09.md) |
| **Proxy** | Build the world from the game's own solid objects | Coarse; small worlds only | Used only for other characters (step 11) |

(src: docs/core/locus-modder-design-2026-10-11.md §5.2; docs/reference/LOCUS_Master_v1.7.md §27.4.3)

## Substitution, step by step

1. **Find the root**: the pointer the game follows to the level's collision (Ratchet: `0x173E40` →
   tree at `0x906800`; Spyro: `0x800785D4` → header). Static analysis or a decompilation names it;
   confirm by changing nothing and decoding.
2. **Describe the format** as data: header, counts, grids or trees, triangle packing, flags.
3. **`format_try` on the original**: the description must decode the level's own collision
   completely (Ratchet: 798,400 bytes, 5,783 cells; Spyro Artisans: 7,209 triangles, every cell)
   before anything is written. Work from a state: `state_extract` gives a snapshot; read the root
   pointer with `mem_read(snapshot)`, then `format_try(path, snapshot, at: <root>, tree_bytes:
   <allocation>)` for a tree, or `params` for a decoder. Pass his position as `near`: the nearest
   triangle should be under his feet, and the bounds should cover the level he walks. Errors give
   the address (decoder) or the offset from the root (tree) where reading stopped. `preview: true`
   saves the triangles as an OBJ in the project.
4. **Capacity**: the space the level allocated is the limit (Ratchet's Piazza tree 753,648 of
   798,400 bytes; Spyro's block 129,336 bytes, 548 left). Too big: crop, simplify, or a window that
   follows him (Spyro: 64 × 72 m, rewritten when he is within 8 m of its edge).
5. **Precision**: snap vertices to the packing grid (R&C 1/16 and 1/64 units; Spyro 16-unit base
   vertices, 9-bit and 8-bit offsets), tile large faces with edge subdivision so there are no
   cracks (R&C ≤ 8 m), drop faces that degenerate after snapping and count them.
6. **Winding and surfaces**: keep the game's winding (Spyro: ground clockwise from above, faces
   reversed after the axis change); copy surface flags the game needs (or none).
7. **Mapping**: `native_from_locus` as a proper rotation (handedness kept), offset so his start is
   the spawn, scale per metre. Crop to the format's limits (R&C: native Z positive; libsm64:
   ±81.9 m). Add walls along the crop's edge in his world only.
8. **Write** with the CPU stopped or the VM paused; read back and re-decode everything; then let
   the game run.
9. **After the swap**: clear cached ground references (Spyro: `m_CollisionTriangleIndex` +0x274 =
   −1) and lift him a little so his own code lands him (pitfall 20).
10. **After respawn or level reload**: detect (frame counter restart, objects alive again) and
    reinstall world and silence.

(src: docs/piazza/piazza-integration-plan.md "World collision for Ratchet"; investigations/spyro1-duckstation.md S3, S7, S9, S10; docs/HANDOFF.md "Game memory map")

## The world LOCUS gives you

LOCUS worlds (Piazza, the Proving Ground) are built from collider primitives: boxes (some
rotated) and cylinders. Building the game's format from primitives was smaller than from a fine
render mesh (Spyro: 13,622 triangles from the mesh did not fit; 3,528 from primitives did): boxes
as five faces (no bottom), cylinders as 6- or 8-sided prisms, props under 30 cm skipped, rotated
boxes kept whenever any part may reach the crop. (src: investigations/spyro1-duckstation.md S3, S9)

## Answer hook (when substitution does not fit)

For engines whose level collision is precomputed in a form you cannot rebuild in place, look for
the place where the game **gathers** collision candidates near a query into a cache it then
probes (a "collide cache" fill). If every query (his movement, ground probes, the camera) fills
that cache through one routine before probing it, LOCUS's `answer_query` template (cache form)
can answer there: at the routine's entry, while LOCUS answers, the hook appends the records of a
LOCUS-owned buffer whose bounds meet the query's box, makes the game's own bookkeeping writes and
returns to the caller instead of running the routine. The connector keeps the buffer filled with
the world's triangles near him as he moves. (src: docs/core/jak-collision-answer-2026-10-09.md)

What to find (static reading first: `code_disassemble`, `code_xrefs`; a decompilation names the
types):

1. **The fill routine** whose job is the level's own triangles (not the moving objects' or the
   water's), and every path into it. Prefer its **entry** ("instead of"): a hook after its import
   loop does not run when the level has no fragments in the box, which is exactly where LOCUS's
   world may have ground and the level none.
2. **The cache's layout**: the record count, the first record, the record size, the game's
   limit, the query's box (integer bounds the routine already computed are the cheapest test),
   an ignore mask and each record's surface word, and what the routine writes after its records
   (its bookkeeping: a group or "prim" entry naming the records). Write a decoder description of
   the cache and check it with `format_try` against the live cache.
3. **The surface values** the game uses for ground and walls: read them where he stands (his own
   "ground surface" fields) and in the cache.
4. **The routine's first two words**: replayed when LOCUS does not answer; they must be loads,
   stores or arithmetic (no branch or jump). The patch is written only while those words are there
   (a routine loaded later is patched once it is loaded).
5. **Memory for the buffer** (two halves): free memory you can prove stays free. Give `admit`
   words that say so (e.g. a heap's pointers): the hook checks them before reading the buffer and
   LOCUS before writing it.

`world.answer` then says how LOCUS's triangles become records: the vertex and surface offsets,
the ground and wall values (by the face's normal), the radius kept around him, the refresh
distance and the tile size. Start the session with `session_start(hooks: true)` (every
hook the adapter declares, the pad hook included; `input: true` does the same and needs it); `adapter_try` keeps the buffer and reports the hook's counters
(calls, answered, full, refused). Check that his own ground fields read LOCUS's surface values
once he stands on LOCUS's world.

Limits to record: the buffer's capacity (triangles near him beyond it are left out, nearest kept),
the game's own cache limit, slopes and ceilings (check them, do not assume), and whatever the game
asks its level for **outside** the cache (other queries the route does not answer).

## Closing scenarios

Drop (five spots, feet within 2 cm of the floor; Spyro's drops landed within 0.2 cm), flat walk,
walls 0.25–2 m, blocks, stairs 0.15/0.25/0.4 m, ramps 10–60°, gaps, ceilings, far field (inside and
past the crop), respawn-ground. Examples in [../templates/scenarios.jsonc](../templates/scenarios.jsonc).
