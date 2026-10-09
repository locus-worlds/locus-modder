# Collision routes: putting his own code on LOCUS ground

Step 5. His movement code must collide with LOCUS's world, not with his level. Three routes, in
order of preference (design §5.2; Master §27.4.3):

| Route | How | Fidelity | Status |
| --- | --- | --- | --- |
| **Substitution** | Write LOCUS's world into the game's own collision format, in place, from a format description; generic builders (grids, trees, quantised vertices) | Highest: his own collision code decides | Proven twice in Rust (R&C tree, Spyro grid); from a description unproven until M1 |
| **Answer hook** | A hook template where the game asks "what is below / around this point?"; a LOCUS routine answers from LOCUS's world in guest memory | Lower: LOCUS's routine answers; cost per query inside the emulated CPU | Unproven; expected for Jak 1 |
| **Proxy** | Build the world from the game's own solid objects | Coarse; small worlds only | Used only for other characters (step 11) |

(src: docs/core/locus-modder-design-2026-10-11.md §5.2; docs/reference/LOCUS_Master_v1.7.md §27.4.3)

## Substitution, step by step

1. **Find the root**: the pointer the game follows to the level's collision (Ratchet: `0x173E40` →
   tree at `0x906800`; Spyro: `0x800785D4` → header). Static analysis or a decompilation names it;
   confirm by changing nothing and decoding.
2. **Describe the format** as data: header, counts, grids or trees, triangle packing, flags.
3. **`format_try` on the original**: the description must decode the level's own collision
   completely (Ratchet: 798,400 bytes, 5,783 cells; Spyro Artisans: 7,209 triangles, every cell)
   before anything is written.
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

For engines whose level collision is precomputed in a form you cannot rebuild in place (Jak 1 is
expected to be one), find where the game gathers collision candidates for the character (a cache
fill, a "find floor" query) and give the template: the PC, where the question is (point, box,
ray), where the answer goes and its layout. LOCUS's routine answers from its own world data. The
template is not built yet: record what you find; do not improvise code. (src: docs/core/locus-modder-design-2026-10-11.md §5.2, §9)

## Closing scenarios

Drop (five spots, feet within 2 cm of the floor; Spyro's drops landed within 0.2 cm), flat walk,
walls 0.25–2 m, blocks, stairs 0.15/0.25/0.4 m, ramps 10–60°, gaps, ceilings, far field (inside and
past the crop), respawn-ground. Examples in [../templates/scenarios.jsonc](../templates/scenarios.jsonc).
