# Presentation: how he looks in LOCUS

Steps 6–8. The world window draws generic kinds, so the adapter says which kind and where the data
comes from; it never carries a decoder.

| Kind | Sent once | Each frame | Was used for |
| --- | --- | --- | --- |
| `skeleton` | mesh, skin weights, bind pose, textures | bone matrices | Ratchet (111-bone palette) |
| `morph` | parts, frames, faces, texture atlas | per part: animation, frame, next frame, blend; part transforms | Spyro, Sparx |
| `mesh_stream` | texture atlas | triangles | Mario (libsm64) |
| `items` | item models | attached to a bone, or a free transform | the wrench |
| `particles` | sprite set | positions, sizes, colours | Spyro's fire |

(src: docs/core/locus-modder-design-2026-10-11.md §5.3; field names provisional)

## Model: two routes, in order

1. **Capture what the game decodes.** Every game unpacks its compressed models into plain vertices
   just before handing them to the graphics hardware: Spyro's renderer unpacks into the scratchpad
   before the GTE; R&C uploads to the PS2's vector unit. A `copy_on_execute` template at that point
   copies the decoded vertices (and the pose the game computed). The adapter gives the PC, the
   source register or address, and the buffer layout. Limits: only what the game draws when it
   draws it (culled or low-detail models capture poorly); no full animation set up front.
2. **Describe the format as data** (fields, bit fields, arrays, pointers, conditional encodings,
   running sums, lookup tables; bounded). Spyro's vertex stream shows what a description must
   express: per-frame 21-bit offsets, per-part cumulative vertex ranges, per-vertex choice of a
   byte index into a 128-entry delta table, a small halfword delta or an absolute halfword pair,
   with the next vertex's encoding selected by a low bit. If the description language cannot
   express a format, that is a missing generic building block: record it, tell the player.

Route 2 is proven on Spyro (his model description gives what the known decoder gives on his real
RAM); route 1 is not yet. Check a description with `format_try` on a snapshot or an extracted state
before the adapter uses it: record counts (animations, frames, vertices, faces) and the vertex
bounds should match what you know of him (his size in game units), and `preview: true` saves the
points or triangles as an OBJ in the project. Records carry integer fields and real fields:
`{"f32": addr}`, `{"f16": addr}`, `{"fixed": [e, frac_bits]}` (PS1 4.12 is 12); reals are values
only (no arithmetic, no conditions: test a float by reading its bytes as `u32`). (src: docs/core/m1-format-routes.md; docs/core/state-extract-format-try.md; docs/core/jak-followups-2026-10-09.md)

**A described skeleton model becomes a skin.** Add a `mesh` section beside the decoder, naming
which records are what (configuration only):

```json
"mesh": {
  "vertex": { "record": "vertex", "position": ["x", "y", "z"], "scale": "scale",
              "normal": ["nx", "ny", "nz"], "uv": ["u", "v"], "joints": ["j0", "j1"], "weights": ["w0", "w1"] },
  "face": { "record": "face", "indices": ["a", "b", "c"] },
  "submesh": { "record": "fragment" },
  "joint": { "record": "joint", "inverse_bind": ["m0", "…", "m15"], "order": "row_major" }
}
```

Vertices and faces belong to the latest `submesh` record before them (or name one with a
`submesh` field); face indices count from the submesh's first vertex (`"from": "all"`: across
every vertex); `scale` is a number or a field of the vertex or its submesh (merc-style per-fragment
scales); weights are normalised; normals are computed from the faces when not given; inverse binds
come from the joint records (give them when the game's joint matrices are joint-to-world, not
skinning palettes). `format_try` reports the skin it builds (joints, vertices, triangles,
submeshes, bounds) or the record and field that did not fit, and `preview: true` saves
`previews/f<N>.lskin` (the `skeleton` model file) and an OBJ. Not yet: textures (the console's
texture decoder), triangle strips and quads (emit triangles), and pointing an adapter's
presentation at a described skin (today only a local export is drawn).

Where the model lives: in RAM for the current level (Spyro's `g_Models[0]`; animations the level
loaded), or on the disc (Ratchet's model was exported once with the Wrench tool from the disc: a
manual prerequisite that the adapter route must replace). Disc tools and live memory disagree
sometimes; trust live memory for what is drawn. (src: investigations/spyro1-duckstation.md S6; docs/ratchet/01-probe/ratchet-model-preview.md)

## Pose

| Way | Cost | Exactness | Example |
| --- | --- | --- | --- |
| Poll animation state (animation, next, frame, next frame, progress) each frame | a few bytes | as exact as the decoding of frames and interpolation | Spyro: body/head/tail animation and frame at `0x80078A70–7E`, progress / 16; parts placed by translation and matrix from pivots |
| Hook the producer: copy the bone palette where the game finishes computing it | 7 KB a frame | exact | Ratchet: 111 bones, 7,104 bytes at `0x251C9C`, 58–60 poses/s |

Hook at the producer, not a consumer, and validate every record with guards (counter sequence,
class and scale of the owner). (src: investigations/spyro1-duckstation.md S8; docs/HANDOFF.md "Key facts")

## Textures

Video memory has the same layout for every game on a console, so the connector decodes it:
PS1 VRAM from a savestate (`state_extract(parts: ["vram"])`: Spyro's faces use one 4-bit page at
(960, 256) with 17 CLUT variants, cropped to 578 × 32 used texels); PS2 GS memory (connector
vocabulary, design §5.1). The adapter gives pages, CLUTs and which faces use which.
(src: investigations/spyro1-duckstation.md S6, S8)

## Items, companions, effects (step 8)

- **Held item**: find the bone whose transform carries the item: Ratchet's wrench object
  orientation agreed with hand bone 56 (position error about 7 µm after undoing the bone's
  inverse-bind translation). Held: drawn on the bone; in flight: the object's own transform while
  its state says thrown (10 outbound, 11 return). (src: docs/ratchet/05-wrench/ratchet-wrench-result-2026-10-07.md; docs/reference/LOCUS_Master_v1.7.md §17.4)
- **Companion**: an object of its own class (Sparx: class 120, own model frames and faces; follows
  his own position; colour by health). (src: investigations/spyro1-duckstation.md S10, S11)
- **Effects**: the game's particle table (Spyro's flame: class, timer, position in unsigned 16-bit
  game units / 4, size, colour), drawn as sprites. LOCUS-made approximations (a cone for fire) were
  rejected by the owner; prefer the game's own data. (src: investigations/spyro1-duckstation.md S11; MODLOG.md, A7 Spyro third playtest fixes)

## Checks

- `adapter_try(model: <the player's export folder>)` on PCSX2 draws him in the research world as
  play does (kind `skeleton`); the result's notes say "drawn: …" or why not.
- `render_preview(model, animation, frame)` (not built yet) beside `screenshot(game)` of the same frame: parts
  attached (head, tail), symmetry, colours, texture placement, decals (pitfall 29).
- Head and part turning: render with a part turned (Spyro: head turned 45°).
- Scale against the 1.8 m pole and the Ratchet silhouette; the player decides.
- Animations the level lacks: list them; the renderer must not ask for them (pitfall 31).
