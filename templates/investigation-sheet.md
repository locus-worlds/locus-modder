# <Character> on <runtime>: investigation sheet

<!-- Kept by the Workbench as the project's notes; append with notes_append(step, text).
     Evidence labels: LOCUS reproduced | documented | inference | synthetic (SKILL.md).
     Every row names its artefact ids (snapshot, trace, state, scenario run). No game data here:
     no dumps, disassembly, decompiler text, screenshots; facts only. No personal paths. -->

Started <date>. Budget agreed with the player: <hours>, per step below. Tool mapping (design name →
actual): <table or "as in SKILL.md">.

## Setup

| Item | Value |
| --- | --- |
| OS, CPU architecture | |
| Runtime, version, architecture | |
| Connector, version | |
| Connector vocabulary (field types, hook templates, writes, savestate parts, decoders, presentation kinds) | |
| Controllers seen | |

## S0 Pin

| Item | Value | Label |
| --- | --- | --- |
| Serial (SYSTEM.CNF) | | |
| Region, version | | |
| Image SHA-1 / SHA-256 | | |
| Boot executable, size, CRC, SHA-256 | | |
| Why this build | | |

## S1 Recon

| Source | URL | Licence | Pinned commit | Build it targets | Gives |
| --- | --- | --- | --- | --- | --- |

Route table:

| Step | First route | Fallback | Why |
| --- | --- | --- | --- |
| 3 Find | | | |
| 5 Environment | | | |
| 6 Presentation | | | |
| 10 Damage | | | |
| 12 Sound | | | |
| 13 Ready state | | | |

Player's approval of the plan: <ask_player id, answer>.

## S2 Access

Boot path (input → screen answered), prompts, time to control, hidden fps, `mem_read` 4 KB cost,
full snapshot cost, research state id.

## S3 The character

| Field | Address or chain (+ guard) | Type, units, axes | Values per action | Label | Confirmations (trace ids) |
| --- | --- | --- | --- | --- | --- |
| Frame counter | | | | | |
| Position | | | | | |
| Facing | | | | | |
| Action / animation | | | | | |
| Health | | | | | |
| Camera | | | | | |
| Input word / lock | | | | | |

## S4 Control and isolation

Pad routing (player's answer). Object table: base, stride or list, bound, guards, fields. Classes
kept and why. Counts silenced. Objects that appeared later.

## S5 Environment

Route and why. Root, format description, `format_try` on the original (counts), capacity used,
precision, crop, mapping (matrix, offset, scale), cached references cleared, respawn handling.
Scenario results (name, adapter hash, pass/fail, numbers).

## S6 Presentation

Kind, model source, hook sites or description, pose source, animations loaded in this level,
texture pages, preview-vs-game comparison, player's answer on looks and scale.

## S7 Camera

Field, layout, checks (orthonormal, distance, moves only with camera), FOV, scenario and player.

## S8 Items, companions, effects

Per item: class, owner rule, bone, flight states, guards, scenarios, player.

## S9 Attacks

| Attack | Input | Signal | Absent for (actions checked) | Category, tier (calibration) | Hitbox, anchor | Scenarios |
| --- | --- | --- | --- | --- | --- | --- |

## S10 Hurt, death, respawn

Damage route, writes, reactions used and the animations they need, lives kept, respawn handling,
scenario runs (three each), player's answer.

## S11 Others solid

Proxy kind, sizes, standing-on fields, scenarios.

## S12 Sound

Library, table layout, owners, bank source and pin, ids and labels, scenario, player's answers per
sound.

## S13 Ready state

Route, script, prompts, cold runs (duration each), install steps, scenario from the saved state.

## S14 Adapter

Final adapter hash, `adapter_validate` warnings and their explanation, `adapter_test` results,
playtest answer, items declared unsupported.

## Time

| Step | Hours | Notes |
| --- | --- | --- |

## Failed routes and disproved claims

| Date | Claim or route | Disproved or abandoned by | What replaced it |
| --- | --- | --- | --- |
