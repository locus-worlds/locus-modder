# Jak and Daxter: The Precursor Legacy: recon brief

**Everything here is inference to verify.** Nothing about Jak 1 has been observed by LOCUS. The
table comes from the design's acid-test section and the Master brief; the leads below it come from
general knowledge of the OpenGOAL project and are not in any LOCUS record. Treat each as a
question for step 1, label it `inference` in the sheet, and replace it with what you observe.
(src: docs/core/locus-modder-design-2026-10-11.md §9; docs/reference/LOCUS_Master_v1.7.md §36.5)

## Setting

The player's second MacBook: the LOCUS desktop app, PCSX2, their PS2 BIOS, their own Jak 1 image.
PCSX2 work so far used an x86_64 build under Rosetta; **an arm64 build is untested**, so record
what `runtimes` reports and expect step 2 to test the runtime as much as the game.
(src: docs/reference/LOCUS_Master_v1.7.md §30.2)

## What the design expects

| Step | Expectation | Stress on the kit |
| --- | --- | --- |
| 0 Pin | NTSC-U **SCUS-97124**, the build OpenGOAL (open-goal/jak-project, ISC) targets | Pin by boot ELF hash: a serial can cover more than one pressing (verify whether OpenGOAL distinguishes releases) |
| 1 Recon | OpenGOAL documents the engine thoroughly: GOAL language and runtime, symbol table, processes, art groups, joints, collision, 989snd | Its decompiled game source is derived from the game: facts only, never copied, never committed |
| 3 Find Jak | GOAL objects live on process heaps, so addresses move; the runtime's symbol table names globals; the player process is reached from a symbol | Needs pointer chains from a symbol in the field vocabulary, and `mem_pointer_scan` |
| 5 Environment | Background collision precomputed per level in compact fragment formats; rebuilding it in place may not fit | Likely the first **answer hook** (a template on the collision-cache fill) rather than substitution |
| 6 Presentation | Merc skinned meshes with joints; joint matrices computed per frame | Capture of what the game uploads to the vector unit, or a merc format description; `skeleton` pose from polled joint matrices or `copy_on_execute` |
| 8 Daxter | A separate object riding Jak's shoulder | `items` on a bone (the Sparx and wrench pattern) |
| 9 Attacks | Punch, spin kick, dive, uppercut as states of the player process | Signals from state fields; anchored hitboxes |
| 12 Sound | 989snd, as in R&C | The Ratchet sound work should transfer with new table addresses |
| 13 Ready state | New game → Geyser Rock, a tutorial level with few objects | `pad_inject` from power-on |

## Leads to check in OpenGOAL (not in any LOCUS record)

Pin an OpenGOAL commit first and record it. Then look for, and confirm live:

- The **player process** reached from a global symbol (OpenGOAL names it `*target*`, of type
  `target`), its control/collision shape holding position and orientation, its state, and a
  "fact" record holding health.
- The **symbol table** layout in the original executable (how a symbol's value is addressed), so
  the adapter can express "pointer from symbol" as a chain from a fixed address.
- **Units**: GOAL code measures distance in "meters" that are a fixed multiple of the float unit
  (OpenGOAL's `meters` macro); find the factor before setting `mapping.scale`.
- The **camera** globals (a camera process and a math-camera with the view matrix).
- The **collision cache** routine that gathers background triangles near Jak each frame: the place
  an answer template would hook. Look also at how moving platforms add triangles: that is how
  other characters could become solid.
- **Merc** and **joint** data: where joint matrices end up each frame (for a polled `skeleton`)
  and where merc data is uploaded to VU1 (for capture).
- **Daxter**: his own process and which of Jak's joints he follows.
- **989snd**: the EE-side "play sound" call and any table of active sounds the game keeps (Jak's
  sound code may differ from R&C's in where it records playing sounds; the library and bank format
  should be the same family).
- **Levels**: Geyser Rock's level name and what is loaded with it; level code vs main executable
  (hooks must be in code that stays).

## Plan for the first sessions

1. Steps 0–2 as SKILL.md: pin, recon with the leads above, a hidden cold boot to Geyser Rock with
   injected input (record every prompt, including memory-card ones with fresh empty cards).
2. Step 3 by differential search first (positions are likely f32), then express what you found as
   a chain from a symbol and confirm it survives a level change and a respawn.
3. Step 5: write the collision format description and `format_try` it on Geyser Rock's own data,
   even if substitution then proves not to fit: it tells you what the answer hook must answer.
   If the connector has no answer template yet, step 5 is **blocked on a LOCUS release**: record
   exactly what the template needs (PC, question layout, answer layout) and tell the player.
   Continue with steps that do not need LOCUS ground (6 preview, 9 signals, 12 sound tables)
   inside Geyser Rock, and label them accordingly.
4. Budget: the PCSX2 engine work is reusable from Ratchet; the GOAL engine is new. Expect the first
   adapter to take days; record hours per step.

## Questions for the player at the start

- Which disc image (region) do they own, and is it their own copy?
- Which pad will drive Jak, and which other devices are attached?
- How much time do they want to give this first run, and when do they want to be asked to play?
