# Field notes: Mario (Super Mario 64, libsm64)

5–11 October 2026. The control case: a host module, not an emulator. Labels: LR = LOCUS
reproduced, doc = documented, inf = inference.

## Route and why

**libsm64** (pinned `fd118132`, library SHA-256 `c108ef6d…`) runs SM64's own Mario code as a
library built from the player's ROM (SHA-1 `9bef1128…`): no emulator, no memory search. The
fastest route when such a library exists. libsm64 is decompilation-derived, so LOCUS never
distributes it: the player supplies it, like an emulator. Profile `host_module`.
(src: docs/reference/LOCUS_Master_v1.7.md §10.3; adapters/sm64-mario/package.json)

## Key facts

| Item | Fact | Label |
| --- | --- | --- |
| State | `SM64MarioState` each tick: position, velocity, facing, action, health (8 wedges) | LR |
| World | LOCUS triangles fed to libsm64 at 100 units/m; SM64's cell grid limits it to about ±81.9 m around the origin; out-of-grid faces are dropped and counted | LR |
| Controls | DualSense read by LOCUS (gilrs), Mario's stick relative to the camera that shows him | LR |
| Camera | A LOCUS approximation of Lakitu (libsm64 has no camera code) | LR |
| Others solid | A libsm64 moving surface object (0.6 × 0.6 × 1.1 m box) at each other character's position | LR |
| Attacks | Action flags: attacking `0x00800000`, air `0x00000800`; dive and ground pound by action value; stomp only when falling onto a head, with bounce | doc (SM64 headers), LR in play |
| Damage | `sm64_mario_take_damage` with knockback; SM64's ~2 s post-hurt blink kept as his trait | LR |
| Look | libsm64's mesh each tick (`mesh_stream`), the ROM's texture atlas; decals baked over triangle colours | LR, owner confirmed |
| Sound | `sm64_audio_tick` PCM (32 kHz; no music in the pinned build) over a local socket | LR, heard by owner (occasional static, cause open) |

(src: docs/piazza/piazza-integration-plan.md; MODLOG.md, Gate 5 Mario side, Phase A A5, A5 fix, A5 playtest; docs/core/character-sound-2026-10-11.md)

## Time

The libsm64 experiments (idle, traversal, ceiling, ledge and gap, contact) ran on 5 October, each a
360-tick capture under a 30-second supervisor; Mario joined Piazza on 6–7 October. No hour count
was recorded. (src: docs/mario/libsm64-idle-result.md; docs/mario/libsm64-contact-result.md)

## What generalised

- The six fields (position, facing, action, health, frame, camera) and the attack-from-action-flags
  pattern became adapter 0.1's shape.
- Contact as a moving body inside the source's own collision (here a surface object): the pattern
  every later character used.
- Damage through the source's own API, keeping its native reactions and invulnerability.
- Drawing what the source draws (decals) is part of "looks right": compare at close range.

## What did not

- No memory search, savestates or hooks: none of the emulator technique comes from Mario.
- A module needs host-side cameras (no game camera to capture).
- libsm64-specific limits (the grid) are not general; every route has its own coordinate limits.
