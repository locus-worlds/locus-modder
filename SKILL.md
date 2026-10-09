---
name: locus-modder
description: Make a LOCUS adapter (a JSON file) that brings a character from a game the player owns into LOCUS, working only through the LOCUS desktop app's developer interface (locus-dev MCP) and testing every step in the Proving Ground. Use when the player asks to make or finish a LOCUS adapter, to "bring <character> into LOCUS", to add a PS1/PS2 game to LOCUS, or hands over a disc image for LOCUS.
---

# locus-modder

You turn the player's own game into a **LOCUS adapter**: JSON (format 0.2) that tells a LOCUS
connector where the character lives in the emulated console's memory and how to run him in a
shared world, plus the scenarios that prove each claim. The character keeps running his original
code inside an emulator that the LOCUS app runs hidden. You work only through the app's developer
interface; you test every step in the Proving Ground; the player judges feel, looks and sound, and
only the player publishes.

Read this file to the end, then the playbook for the runtime ([pcsx2](playbooks/pcsx2.md) or
[duckstation](playbooks/duckstation.md)), then [pitfalls.md](pitfalls.md). Open a technique file
when a step names it. [field-notes/](field-notes/) show how Mario, Ratchet and Spyro were really done.
For Jak and Daxter start with [jak1/recon-brief.md](jak1/recon-brief.md).

`src:` notes cite records in the LOCUS repository (paths from its root). They let reviewers trace
every claim; you do not need them to work.

## What runs today

Checked against `locus-dev` on 9 October 2026 (PCSX2 2.8.2 and 2.9.103, DuckStation). Tools
that are not built yet answer every call with "not available yet" and the reason; trust that
answer over this table, and tell the player at the start which steps you can close.

| Step | Today | Missing for the rest |
| --- | --- | --- |
| 0 Pin | **runs** (`game_identify`, `adapter_validate`) | — |
| 1 Recon | **runs** (web research outside the app) | — |
| 2 Access | **runs**: boot, read, snapshot, savestates, `screenshot(game)`; on PCSX2, menus by `input_script` once the adapter has a `pad_inject` hook (`session_start(input: true)`) | Until the hook exists: the player's own savestate (`session_start(state: <path>)`), or the player drives a **visible** session with their pad |
| 3 Find | **runs**: snapshot, diff, search, watch, pointer scan, disassembly, cross-references; input by `input_script` (PCSX2, with the hook) or `input_press` (where the game does not rewrite its pad record) | `breakpoint` |
| 4 Control, isolate | **runs**: `adapter_try` joins the draft to the Proving Ground | `input_route` (routing a physical pad): the player plays in the session instead |
| 5 Environment | **partly**: `adapter_try` writes the world as his collision on DuckStation; on PCSX2 only at launch (play), so a research session keeps the game's own ground | `format_try`, `state_extract` |
| 6 Presentation | **signals only**: find skeleton/mesh data | `render_preview`, `state_extract`; the world window draws `mesh_stream` only, so a skinned character (`skeleton`) is not drawn yet |
| 7–12 | **runs**: `adapter_try`, `world_spawn`, `world_scenario`, `adapter_test`, the dummy | Feel and looks need the player in a playtest; a character the world cannot draw is judged by scenarios and the player's view of the game window |
| 13 Ready state | **runs** on PCSX2 with the pad hook (`input_script` from power-on) | — |
| 14 Adapter | **runs**: `adapter_write`, `adapter_validate`, `adapter_test`, evidence | Publishing (the player's click) |

A step whose closing check needs a missing tool is **blocked on <tool>** in the sheet, never done.

## Rules

1. **The app is the only door to the game.** Never launch an emulator, open a PINE or GDB socket,
   attach a debugger, read the player's files or run scripts against the game yourself. Web
   research is fine; downloading or running tools is not. If a tool you need is missing, stop and
   tell the player what is missing. (src: docs/core/locus-modder-design-2026-10-11.md §1, §3.2)
2. **Isolated sessions only.** The app's sessions copy the player's settings, memory cards and
   saves and never write them. Never work beside the player's play session: if `session_start`
   refuses because one is running, ask the player to close it. A research probe once attached to
   the owner's live game and halted it. (src: docs/reference/LOCUS_Master_v1.7.md §30.4)
3. **Nothing game-derived leaves the machine or enters any repository**: memory dumps, snapshots,
   savestates, models, textures, sounds, game screenshots, disassembly, decompiler output. They
   stay in the project folder the app manages. Do not save tool output that holds game data
   (memory bytes, disassembly, images) into files of your own; ask tools for candidates and
   summaries rather than large dumps. The adapter holds facts: addresses, offsets, layouts,
   numbers, names. (src: docs/core/locus-modder-design-2026-10-11.md §1 principle 4)
4. **Facts only from outside sources.** Decompilations, symbol maps, wikis, cheat lists and
   extraction tools are read for facts (addresses, struct layouts, field meanings, format layouts).
   Never copy their code or text into the adapter, a format description or your notes. A GPL
   licence does not change this; a decompilation is derived from the game whatever its licence.
   Record each source's URL, licence and pinned commit. (src: docs/core/locus-modder-design-2026-10-11.md §4 step 1)
5. **Adapters are data.** No machine code, scripts or expressions. Code inside the console comes
   only from the connector's hook templates (copy, log, press, answer) whose blanks you fill. If
   the connector's vocabulary lacks something, validation says so: record the gap and tell the
   player; a new building block is a LOCUS release. (src: docs/reference/LOCUS_Master_v1.7.md §27.4.2)
6. **Bounded writes only** (`write`, `or_bits`, `and_bits`, `ring_record`, `keep_at_least`) to
   declared RAM regions. Never route around `mem_write`'s refusal to write code while PCSX2 runs:
   doing that crashed PCSX2. (src: docs/reference/LOCUS_Master_v1.7.md §30.2)
7. **Never report an unperformed test as passed.** A step closes only on a passed Proving Ground
   scenario (the app's result, tied to the adapter hash) or the player's answer in this
   conversation, recorded with `evidence_record(player: …)`. Static reading, unit conversions and synthetic events are not
   tests of the character. Never substitute a LOCUS-made controller or composed frames for the
   game's own behaviour. (src: AGENTS.md)
8. **Ask the player, in this conversation,** for what only a human can judge: how the controls and
   camera feel, how he looks and how big he is, how he sounds. Do not settle these from your own
   screenshots. Record each answer verbatim with `evidence_record(player: "<their words>")`;
   never answer for them. See [Working with the player](#working-with-the-player).
9. **Publishing is the player's click** in Workbench ▸ Publish. Never upload, publish, push or
   share anything, and never ask to.
10. **Approvals** (session start, memory writes) are given once per project in the app. If the
    player declines one, stop that line of work and ask what they prefer.
11. **No personal information** in notes or the adapter: identify the disc by hashes, not by the
    player's file path or user name.

## Evidence labels

| Label (notes) | Meaning | Adapter `evidence.level` |
| --- | --- | --- |
| LOCUS reproduced | Observed by you through the app in this project's sessions, or read from the player's own files; cites the snapshot, trace, state or scenario id | `tested` |
| documented | Stated by an outside source at a pinned revision | `research` |
| inference | Your reading, not yet observed | `inference` |
| synthetic | Produced by a LOCUS stand-in (the dummy, an injected test event), not by the game | `synthetic` for the stimulus; the game's response to it can be `tested` |

Levels follow the 0.1 schema (src: docs/schemas/locus-adapter-0.1.schema.json); 0.2 may rename
them. Every `evidence_record` gives: the claim, the label, build and runtime versions, the
artefact ids, and what was **not** tested. Format: [templates/evidence-record.md](templates/evidence-record.md).

## Before step 0

1. List the MCP tools of the developer interface and compare them with the [tool
   reference](#tool-reference). Names may differ slightly; write the mapping you use into the notes
   (`notes_append("setup", …)`). No developer interface: ask the player to open the LOCUS app's Developer ▸
   Workbench, create (or open) the adapter project, and add the configuration its **Connect your
   agent** card shows to Claude Code with `claude mcp add` (use `--scope user` when Claude Code
   runs without a project folder, since a local server is registered for one folder only), then
   start a new session: MCP servers load when a session starts. The player may allow the `locus-dev` tools in Claude Code's permissions so they
   are not asked for every call; the app's own approvals still apply.
   **Approvals**: a call that needs one (session start, memory writes) fails with the request id
   (`apr-N`). Only the player can grant it, with **Allow** in Workbench ▸ the project: ask them in
   the conversation to press it, and call again once they say they have. A yes in the
   conversation does not grant it.
2. `system_info`, `runtimes`, `connectors`: record OS, CPU architecture, emulator version and
   architecture, connector version and its **vocabulary** (field types, object tables, hook
   templates, write operations, savestate parts, console decoders, presentation kinds). The adapter
   may use only that vocabulary.
3. Start the investigation sheet from [templates/investigation-sheet.md](templates/investigation-sheet.md).
4. Agree a time budget with the player and record time per step. The first adapter for an engine
   costs days (Ratchet's 20-hour budget ran over three days); later games on the same engine are
   faster. At the budget, report continue / change route / stop. Ask when they want to be called
   to play or listen. (src: docs/reference/LOCUS_Master_v1.7.md §23)
5. Where files go: the adapter and its format descriptions are written **only** with
   `adapter_write` (it writes `adapter/package.json` and `adapter/formats/<name>.json` in the
   project and validates). Before writing, call `adapter_schema`: it returns a complete adapter
   that validates (Spyro), the section names, the JSON Schema per `section`, and each connector's
   vocabulary. Copy the shape, not the values. They hold facts only.
6. Read [Working with the player](#working-with-the-player) before the first session.

**Resuming** a project in a new conversation: read the investigation sheet, the step list and the
last evidence records before any call; do not trust memory of earlier sessions.

## Working with the player

Everything you ask the player happens in this conversation; the app shows only approvals and
your activity. (src: first Jak sessions, 9 October 2026: two recordings were lost because the
instructions and the recording started together.)

- **Before anything opens or closes**, say what will happen: "I will start a visible PCSX2
  window now". Ask them to tell you if they cannot find it.
- **Recordings the player plays into** (`mem_watch`, snapshots around an action): first say
  exactly what to do and for how long, then **wait for their "ready"** and end your turn. Start
  the recording only after they answer, and say "recording now, N seconds" in the same reply.
  One request per recording; never chain two without telling them.
- **Each answer is evidence**: `evidence_record(claim, level, player: "<their words>")`, and in
  the sheet "player (chat)". Ask one question at a time, in plain words, with the choices when
  there are some.
- **Long waits** (opening films, loading): say how long and what they will see; use the time for
  research that needs no session. Ask early whether they have a save past the opening: the
  session copies their memory cards, so "Load game" can skip a long film.
- **Before moving on** between sessions or steps, say what you closed, what is blocked and why.

## The procedure

Each step: **Do**, **Record**, **Closed by**; [Going back](#going-back) says where to return when
a later step disproves an earlier one. The Workbench shows the steps as the project's checklist; mark a step done only when its
closing check is in the app's record.

### 0 Pin the build

- **Do**: `game_identify(path)` on the image the player names. If they own several regions, prefer
  the build public research targets (Spyro was switched from PAL to US for this; for Jak, OpenGOAL
  targets NTSC-U SCUS-97124) and confirm the choice with the player.
- **Record**: serial (SYSTEM.CNF), region, version, image SHA-1 and SHA-256, boot executable name,
  size, CRC, SHA-256 and XXH64 (`game_identify` returns all of them); emulator version and
  architecture. Decompilation projects may tell pressings apart by a boot ELF hash (OpenGOAL: XXH64
  in decimal); match it and record which pressing this is.
- A file name's "Rev 1" and SYSTEM.CNF's `VER` are different labels (Jak's Rev 1 image says
  `VER = 1.00`). Pin by the hashes; record both labels as they are.
- **Closed by**: fingerprints in `source.builds[]` and `adapter_validate` passing on the identity.
- Every address differs between regions and revisions; a serial alone does not pin a revision.
  (src: investigations/spyro1-duckstation.md S0; investigations/ratchet.md target sheet)

### 1 Recon: choose routes

- **Do**: search for decompilations, symbol maps, format documentation, extraction tools, modding
  wikis, cheat and patch codes (a cheat code is an address), and LOCUS field notes for the same
  engine. For each: URL, licence, pinned commit, build it targets, what it gives.
- **Record**: the sources table and a **route table**: for each of steps 3–13, a first route and a
  fallback (for example step 5: substitution from a format description, else the answer hook).
- **Closed by**: the route table in the notes and the player's yes to the plan.

### 2 Access

- **Do**: `session_start(runtime, game, {visible: false})` with no state (a fresh boot). It
  returns only when the emulator's memory interface answers; otherwise it stops the emulator and
  says why (the settings file it loaded, its last log lines). If it fails for want of a BIOS, the
  player sets one in their emulator and quits it completely (⌘Q) so it saves; you never handle
  BIOS files. PCSX2 must be 2.7 or newer (LOCUS tests 2.9.103; 2.8.2 works): an older one is
  refused, and the player updates it. A session refuses to start beside another copy of the same
  emulator: ask the player to quit it (`allow_other_emulator: true` only if they say it must stay).
  Drive the menus with `input_script`, waiting on what you observe (`screenshot(game)`,
  `session_log`, later a frame counter) rather than fixed delays; prompts can ignore input for
  about 3 s. Reach the first controllable moment, `state_save("research-start")`. Measure hidden game frames per second
  (`session_status`) and the cost of a 4 KB `mem_read` and a full-RAM `mem_snapshot`.
- **Record**: the boot path (each input and the screen it answered), region prompts (memory card),
  time to control, hidden fps, read costs.
- **Closed by**: `session_status` running hidden near the game's frame rate, and the state loading
  in a second session to the same scene.
- This script is for research; the product version is step 13.

### 3 Find the character

Technique: [techniques/differential-search.md](techniques/differential-search.md).

- **Do**: find the six fields every character needs: **frame counter, position, facing,
  action/animation, health, camera**, and a guarded route to his record (static address or pointer
  chain from a static root or symbol, `mem_pointer_scan`). Inject input with `input_script`, take
  `mem_snapshot`s around it, narrow with `mem_diff` and `mem_search`, confirm with `mem_watch`.
- **Record**: per field: address or chain, type, units, axes and signs, value per action,
  evidence, two independent confirmations.
- **Closed by**: in a **second session** (fresh boot or the state) with a different input script, a
  `mem_watch` trace in which every field follows the injected input. After step 5, the `spawn-idle`
  scenario reads them through the adapter.
- A guessed byte can mean two things: confirm every action value with at least two different
  inputs and check that no other action produces it (Ratchet's "combo stage" was also 3 on every
  jump). (src: MODLOG.md, Ratchet melee fix 8 Oct)

### 4 Control and isolate

Technique: [techniques/object-tables.md](techniques/object-tables.md).

- **Do**: route the player's chosen pad (`input_route(device)`); every other device is hidden from
  the emulator. Find the game's input word or pad buffer and any input lock it has (needed to share
  a pad). Find the object table (array with stride, or linked list), its bound and per-entry class,
  state, position and update fields. List the classes of the hero, his weapons and held items,
  projectiles, companions and effects. Silence everything else by an **allow-list by class**,
  preferring the game's own way of freeing objects over overwriting them. Declare `input` and
  `world.silence`; `adapter_try`.
- **Record**: table layout, bound, guards, classes kept, counts silenced.
- **Closed by**: in the game level, 30 s of `mem_watch` on object states plus `screenshot(game)`
  showing nothing else acting, then (after step 5) the `isolation-idle` scenario; and the player's
  the player's report: "only pad A moves him; pads B and C do nothing".
- Keep weapons alive: a frozen wrench could not be thrown or swung. (src: MODLOG.md, wrench throw follow-up)

### 5 Environment

Technique: [techniques/collision.md](techniques/collision.md).

- **Do**: choose the route in this order: (a) **substitution**: write LOCUS's world into the game's
  own collision format from a format description; (b) the **answer** hook: LOCUS answers the
  game's collision questions; (c) a proxy built from the game's own solid objects (narrow worlds
  only). For (a): `format_try` your description on the level's original collision first; it must
  decode every element exactly before anything is written. Respect the allocation (capacity),
  precision (packing grid), winding and coordinate limits (crop). After a swap, clear any cached
  ground reference and lift him slightly so his own code lands him. Detect respawns and level
  reloads and reinstall. Set `mapping` (a proper rotation, handedness kept; offset so his start is
  the spawn; a provisional scale, confirmed at step 6). Until step 6 the world window may draw him
  only as his declared body.
- **Record**: format description, capacity used, crop, mapping, scale, drop results.
- **Closed by**: the step-5 scenarios (drop, flat walk, walls, blocks, stairs 0.15/0.25/0.4 m,
  ramps, gaps, ceilings, far field, respawn-ground) passed on the current adapter hash. Limits that
  remain (for example a crop) are declared in the adapter.

### 6 Presentation

Technique: [techniques/presentation.md](techniques/presentation.md).

- **Do**: choose the kind (`skeleton`, `morph`, `mesh_stream`). Route 1: capture what the game's
  renderer decodes (`copy_on_execute` where vertices or bone matrices go to the hardware). Route 2:
  a format description of the model. Pose from polled animation fields or a captured bone palette.
  Textures from video memory (`state_extract(state, "vram")`). List which animations this level
  has loaded. Compare `render_preview(model, animation, frame)` with `screenshot(game)` of the same
  frame.
- **Record**: kind, hook sites, layouts, animations loaded in this level, texture pages.
- **Closed by**: preview and game screenshot agree, then the player's answer: "Does he look right?
  Is his size right beside the 1.8 m pole and the Ratchet silhouette?" Scale was judged by the
  owner against the other characters each time (Spyro: 90 % of Ratchet's drawn 1.41 m).
- A scale change moves every distance: rerun the step-5, 9 and 11 scenarios.

### 7 Camera

- **Do**: find the game camera (a matrix or position and target a few metres from him, forward
  axis toward him, orthonormal; it must change when only the camera moves) and declare
  `presentation.camera.kind = "captured"`; otherwise declare a LOCUS camera. His stick is relative
  to the camera his game thinks it has, so a captured camera must be the one shown.
- **Closed by**: the `camera-forward` scenario, and a playtest with the question "do the
  stick directions feel right?". Record the field of view if found (Ratchet's is not matched).

### 8 Items, companions, effects

- **Do**: find each by class in the object table. Held items: the hand bone whose transform equals
  the item's (Ratchet: bone 56, 7 µm). In flight: the object's own transform and state values.
  Companions: their object (Spyro's Sparx). Effects: the game's particle table. Follow by class and
  owner with guards on every read; never cache a slot.
- **Closed by**: `item-held` and, where it applies, `item-throw` scenarios; `hurt-and-respawn` still
  shows the items afterwards; the player confirms they look right.
- Objects are recreated in new slots after respawn (the wrench moved from slot 305 to 300).
  (src: MODLOG.md, wrench respawn)

### 9 Attacks

Technique: [techniques/attacks.md](techniques/attacks.md).

- **Do**: for each attack input, `input_script` the press while `mem_watch` follows animation id,
  state and weapon object state. Keep only signals that appear for that attack and for no other
  action (walk, jump, land, hurt, idle). Declare trigger (`when`, `unless`, one instance per
  activation or value), category, tier (calibrated in his own game), `max_reach`, and a hitbox
  anchored on a bone, object or the body, sized from his model.
- **Closed by**: per attack, `dummy-<attack>` (exactly one hit per swing at 1 m) and
  `dummy-<attack>-miss` (none out of reach or facing away); `no-false-hits` (20 s of walking,
  jumping and landing beside the dummy, zero hits); then a playtest in the arena: "does a
  hit register only when it visibly connects?"

### 10 Hurt, death, respawn

Technique: [techniques/damage.md](techniques/damage.md).

- **Do**: find how his game applies damage (a flag word it checks each frame, a hit or event ring,
  a function) and drive it with the bounded writes. Use only reactions whose animations this level
  has loaded; for the last hit use one that leads to the game's own death. Keep lives topped up
  (`keep_at_least`) so game over cannot end the session. After respawn reinstall world and
  silence, rediscover items, and expect frame counters to restart.
- **Closed by**: `hurt-light`, `hurt-heavy` and `hurt-and-respawn` with the dummy attacking, each
  three times in a row: health drops by the damage table, his own hurt animation plays, he dies,
  respawns on LOCUS ground, and the world sees `dead` then `respawned`. The player confirms the
  reactions look right.

### 11 Others solid

- **Do**: make other characters solid inside his game: rescale and move parked objects of his game
  (Ratchet: crates through the object's scale field, stacked to the other body's height), or add
  moving boxes to his collision (Spyro). Size from each other character's declared body. Clear his
  "standing on" references before moving a proxy he stands on; restore scales at stop.
- **Closed by**: `stand-on-dummy`, `bump-dummy`, `dummy-walks-away` scenarios.

### 12 Sound

Technique: [techniques/sound.md](techniques/sound.md).

- **Do**: identify the sound library (strings in memory). Find the game's channel or voice table
  (slots with sound id, owner object, state, position); follow it by polling, writing nothing.
  Publish only slots owned by him, his items and companions. Samples come from the disc or from
  sound memory in a savestate (`state_extract(state, "spu")`), described as data and decoded by the
  connector. Label sound ids by the action that plays them.
- **Closed by**: `sound-triggers` (a jump gives his jump id within 0.5 s in `bus_trace`), then
  the player's report: "Jump, attack, let the dummy hit you: do you hear his jump, swing
  and hurt, and nothing from the level?" Record the answer per sound. Never unmute or capture the
  emulator's audio.

### 13 Ready state

Technique: [techniques/ready-state.md](techniques/ready-state.md).

- **Do**: write `ready_state` as observed steps: boot, wait until a field rises or equals, press
  when a field says the screen is ready, install (world, hooks, silence), save a local state.
  Route: new game (default), the player's own save copied into the profile, or unlock flags where
  the level loads the item. Find menu, film and level fields if you lack them.
- **Closed by**: **two consecutive cold runs** (`session_stop`, then `session_start` with no state)
  reach the ready condition (spawn position, full health, the level id) and `adapter_test` passes
  from that state. Record each run's time.

### 14 Adapter and playtest

- **Do**: `adapter_validate` (explain or fix every warning), `adapter_test` (every scenario, from a
  cold ready state), a playtest in the Proving Ground, with another installed character if
  there is one. Players reach Piazza only online; say so if the player asks to try it there.
  Fill [templates/final-report.md](templates/final-report.md) and give it to the player.
- **Closed by**: all scenarios passed on the final adapter hash and the player's playtest answer.
  Tell the player that publishing is in Workbench ▸ Publish and is their decision.

## Going back

When a later result disproves an earlier one: mark the later step blocked, record the disproof as
an evidence record ("claim X disproved by Y"), fix the earlier step, then rerun every scenario that
reads the changed fields. Keep failed routes in the notes.

| Later observation | Suspect | Back to |
| --- | --- | --- |
| Fields read nonsense after respawn or a level change | Heap object moved; chain unguarded | 3 |
| Attacks register on jumps or bumps | Ambiguous signal | 9 (and 3's action field) |
| Weapon stuck, no swing or throw | Weapon class frozen | 4 |
| Item missing after respawn | Slot cached | 8 |
| Carried off or teleported right after the collision swap | Cached ground reference | 5 |
| Falls through after respawn | Game restored its own collision | 5 |
| Falls through in one place | Crop, culling or capacity | 5 |
| Game hangs when a proxy moves | "Standing on" reference | 11 |
| Crash on a hit | Reaction animation not loaded in this level | 10, 6 |
| Stick directions wrong | Camera | 7 |
| Too small or large | Scale | 5, then 9 and 11 |
| A hook stops working in another level | Hook in level code (overlay) | playbook, hook placement |
| Cold boot sometimes stalls | Timed waits | 13 |

**When a session crashes or hangs**: read `session_log(tail)` and `session_status`, record the
last call and write before it as an evidence record, and restart from the last good state. Do not
repeat the same write to see if it happens again without a reason to expect a different result;
a crash your write caused is a finding about that write.

## Limits and unsupported items

If a route and its fallback fail within the budget, declare the item unsupported in the adapter
with the reason, and tell the player. Publish needs every step done or declared unsupported.
Honest limits are better than unverified claims.

## Tool reference

Names follow design §3.1 (src: docs/core/locus-modder-design-2026-10-11.md §3.1); `locus-dev`
may rename some. Use the mapping you recorded before step 0.

| Step | Main tools | Closing check |
| --- | --- | --- |
| Setup | `system_info`, `runtimes`, `connectors`, `notes_append` | — |
| 0 Pin | `game_identify`, `disc_list`, `runtimes` | `adapter_validate` |
| 1 Recon | web search (outside the app), `disc_list`, `disc_read`, `notes_append` | player's yes |
| 2 Access | `session_start`, `input_script`, `screenshot(game)`, `session_log`, `session_status`, `state_save`, `state_load` | second session loads the state |
| 3 Find | `mem_snapshot`, `mem_diff`, `mem_search`, `mem_watch`, `mem_pointer_scan`, `mem_read`, `input_script`, `code_disassemble`, `code_xrefs`, `breakpoint` (DuckStation only) | `mem_watch` in a second session |
| 4 Control, isolate | `input_route`, `input_press`, `mem_read`, `mem_write`, `adapter_try` | `world_scenario`, player's report |
| 5 Environment | `format_try`, `disc_read`, `state_extract`, `adapter_try`, `world_start(proving_ground)`, `world_spawn` | `adapter_test` |
| 6 Presentation | `state_extract`, `code_disassemble`, `render_preview`, `screenshot` | player's yes |
| 7 Camera | `mem_search`, `mem_watch`, `adapter_try` | scenario, playtest in chat |
| 8 Items | `mem_read`, `mem_watch`, `adapter_try` | scenarios, player's report |
| 9 Attacks | `input_script`, `mem_watch`, `world_dummy(stand)`, `bus_tail`, `bus_trace` | `adapter_test`, playtest in chat |
| 10 Hurt | `world_dummy(attack_*)`, `mem_watch`, `bus_trace`, `adapter_try` | `adapter_test`, player's report |
| 11 Others solid | `world_dummy(walk_path)`, `adapter_try`, `bus_trace` | `adapter_test` |
| 12 Sound | `mem_watch`, `disc_read`, `state_extract(spu)`, `format_try`, `bus_trace` | scenario, player's report |
| 13 Ready state | `session_stop`, `session_start`, `input_script`, `state_save`, `adapter_try` | two cold runs, `adapter_test` |
| 14 Adapter | `adapter_schema`, `adapter_write`, `adapter_validate`, `adapter_test`, `evidence_record` (playtest in chat) | scenarios + player |
| Any | `evidence_record`, `notes_append`, `world_reset`, `session_stop` | — |

## Files

| Path | Use |
| --- | --- |
| [playbooks/pcsx2.md](playbooks/pcsx2.md), [playbooks/duckstation.md](playbooks/duckstation.md) | Emulator behaviour, savestates, hooks, sound hardware |
| [pitfalls.md](pitfalls.md) | Every trap met so far: symptom, cause, check |
| [techniques/](techniques/) | How to do steps 3–13 |
| [templates/](templates/) | Investigation sheet, evidence record, scenarios, final report (the adapter shape comes from `adapter_schema`) |
| [field-notes/](field-notes/) | Mario, Ratchet, Spyro as they were done |
| [jak1/recon-brief.md](jak1/recon-brief.md) | Starting brief for Jak and Daxter (inference, to verify) |

## Provisional

Written against the design of 11 October 2026; checked against `locus-dev` on 9 October 2026.
[What runs today](#what-runs-today) lists the tools still missing. Still provisional:

- **Adapter 0.2 field names**: `adapter_schema` and `adapter_validate` are authoritative; where a
  technique file's example differs, follow them and note the difference.
- **Hook templates** (`copy_on_execute` and `pad_inject` exist on PCSX2; `answer_query` not yet) and the
  connector vocabularies: `connectors` and `adapter_schema` list what each connector accepts.
- **Proving Ground** zone and spawn names, dummy behaviours: `world_start` returns them.
- **Format routes** (capture, format descriptions): `format_try` is not built yet.
