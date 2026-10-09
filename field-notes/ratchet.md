# Field notes: Ratchet (Ratchet & Clank, PCSX2)

5–11 October 2026, SCUS-97199 v1.00, CRC `CE4933D0`, boot ELF SHA-256 `e0505810…`; PCSX2 v2.8.2
then stock v2.9.103 (x86_64 under Rosetta). Labels: LR = LOCUS reproduced, doc = documented,
inf = inference.

## Route and why

**Live runtime**: the whole game runs hidden in PCSX2; LOCUS reads and writes guest memory over
PINE, captures the pose with a guest hook, and writes LOCUS's world into the game's own collision
tree. No library existed; the community decompilations (Lombyte, RC1, Veradictus) were partial
leads with stubbed routines, so live observation decided. (src: investigations/ratchet.md; docs/HANDOFF.md)

## Key facts (SCUS-97199)

| Item | Fact | Label |
| --- | --- | --- |
| Object (moby) array | `0x1845E80`, stride `0x100`, hero = entry 0; +0x10 position, +0x20 state, +0x24 class pointer, +0x2C scale, +0x74 update, +0xA6 class; 308 live | LR |
| Position | hero +0x10 (`0x1845E90`), Z up; `0x13F3D0` is a collision working copy with transient values | LR |
| Mapping | native = (x + 129, 123 − z, y + 30), 1 unit per metre; crop x −80…80, y −29…90, z −142…22 | LR |
| Frame counter | `0x15F3F8` | LR |
| Camera | world matrix `0x167290`, rows forward/left/up, position +0x30; FOV not matched | LR |
| Collision | root `0x173E40` → tree `0x906800`, 798,400 bytes; Piazza tree 753,648 bytes written in place, 1/16 and 1/64 packing grid, faces tiled ≤ 8 m | LR |
| Pose | 111-bone palette (7,104 bytes) copied at producer `0x251C9C` into two owned slots in a 16,000-byte reservation at `0x01FF8000`; 58–60 poses/s | LR |
| Health | 4 at `0x1415F8` | LR |
| Animations | 23, 24, 25 wrench combo; 43 jump-slam; 16 flinch; 69 death; 7 jump | LR |
| Damage | hit ring `0x178100` (64 × 64 bytes, next `0x173E54`, target +0xA4), flag bit 0, damage +0x2C: his own flinch, death, respawn | LR |
| Wrench | class 71; held on hand bone 56; object states 10 outbound, 11 return; recreated in another slot after respawn | LR, owner confirmed |
| Others solid | frozen crates (class 500) scaled to 0.6 m cubes, stacked; "standing on" fields cleared before moves | LR |
| Pad injection | `scePad2Read` epilogue `0x124C94` (main executable) | LR |
| Sound | 989snd; channel table `0x13E5C0` (30 × 0x70); Veldin's bank on the disc; his class sounds → bank 20–81, wrench 95–99 | LR, heard by owner |
| Cold boot | power-on to Veldin by injected input, three consecutive hidden runs (59.7 s); control from level frame ~1000 | LR |

(src: docs/HANDOFF.md "Key facts"; docs/piazza/piazza-integration-plan.md; MODLOG.md; docs/ratchet/06-sound/ratchet-sound-2026-10-11.md; connectors/pcsx2/src/rac1.rs)

## Time

The Master's provisional budget was 20 active hours for Gates 0–3; the work ran across 5–7 October
in long sessions, through pose capture, controller routing, collision substitution and continuous
play, continued each time by the owner's decision. Most of the time went into engine-level work
(PINE behaviour, savestate format, guest hooks, controller routing) that the PCSX2 connector now
holds. (src: docs/reference/LOCUS_Master_v1.7.md §23)

## What generalised

- Differential search against deliberate input; then a write-watch to tell the authoritative copy
  from working copies.
- Object table as the key to isolation, weapons, proxies; freeze by allow-list of classes.
- Substitution into the game's own collision, verified by re-decoding everything written.
- Hook at the producer site with owned slots and guards (`copy_on_execute`).
- Damage through the game's own hit path; attacks from animation ids and weapon state.
- Cold boot by pad injection observed, not timed.
- 989snd knowledge transfers to other 989snd games (Jak 1).

## What did not

- Early research went through PCSX2's debugger window by hand: slow, error-prone (an accidental
  memory edit), and replaced by the developer interface.
- The model came from a separate disc tool (Wrench) as a manual prerequisite; adapters need capture
  or a format description instead.
- The first hand-made play state, bounded trial supervisors and Python tooling were development
  shortcuts, not product paths.
- Level code moves (overlays); every hook in it had to move to the main executable.
