# Pitfalls

Every trap met in the Mario, Ratchet and Spyro integrations, 5–11 October 2026. Each: what you
see, why, and the check that catches it before the player does. Grouped by step.

## Access and sessions (steps 2, all)

**1. A probe attaches to the player's live game.**
Symptom: the player's game freezes for a moment, or your reads come from the wrong scene.
Cause: a research client connected to the play session's debug port (Spyro: halted and resumed
the owner's CPU). Check: never bypass the app; if `session_start` refuses because a play session
runs, ask the player to close it. (src: MODLOG.md, A7 Spyro third playtest fixes)

**2. Orphaned processes.**
Symptom: a CPU core busy for hours; a port already in use. Cause: a script polling a dead
socket (two hours once). Check: `session_stop` every session; read the app's leftover report.
(src: docs/reference/LOCUS_Master_v1.7.md §30.2)

**3. Settings reset to defaults.**
Symptom: DuckStation shows its setup wizard; GDB off. Cause: a settings file without
`[Main] SettingsVersion`. Check: the player has run DuckStation once; `session_log` shows the
profile loaded. (src: investigations/spyro1-duckstation.md S1)

**4. Stalled transport after the character leaves the world.**
Symptom: PINE status requests time out. Cause: Ratchet walked off the edge of a small test floor
and fell (three times). Check: keep him inside the world (edge walls) before long runs.
(src: docs/piazza/piazza-integration-plan.md)

**5. A few failed requests right after start.**
Symptom: 6–7 PINE failures as PCSX2 starts (cause open). Check: retry during start-up; judge the
route only after the game runs. (src: investigations/spyro1-duckstation.md S10; MODLOG.md, A7 Spyro third playtest fixes)

**6. A debugger memory view edits memory.**
Symptom: one byte changed (`0x000FD87A` 00 → 67). Cause: a keystroke meant for navigation was typed
into PCSX2's editable memory view. Check: the developer interface has no such view; if you ever
work in a debugger UI, never type into a memory view, and discard any session that was edited.
(src: investigations/ratchet.md "Caller source and watchpoint cleanup")

**7. Pause does not advance; Run does not move.**
Symptom: the game stays at the same PC. Cause: in PCSX2's debugger, System ▸ Pause re-hits the
current breakpoint, and the first Run after loading a state stops at the same PC. Check: verify
progress by the frame counter, never by a button's state. (src: docs/ratchet/02-collision/ratchet-native-floor-result.md)

**7a. PCSX2 starts but its memory interface never answers.**
Symptom: `session_start` waits, then reports no answer; the hidden PCSX2 sat at its setup wizard
or "no BIOS". Cause (Jak, 9 October): PCSX2 2.x with `-datapath P` reads its settings from
`P/PCSX2`, so a profile copied to the wrong level ran on defaults (no BIOS, PINE off); PCSX2 2.6
rejects `-datapath`; a BIOS chosen while PCSX2 is still open is not saved yet. Check: the result's
diagnosis names the settings file PCSX2 loaded; PCSX2 2.7+; the player quits PCSX2 (⌘Q) after
choosing a BIOS. Fixed in `locus-dev` the same day; if it recurs, report the diagnosis to the
player rather than working around it. (src: crates/locus-dev/src/session.rs)

## Finding fields (step 3)

**8. A guessed byte with two meanings.**
Symptom: attacks register on jumps, bumps and flinches. Cause: Ratchet's `+0x7E`, read as the
wrench combo stage, was also 3 on every jump. Check: every action value is confirmed with two
different inputs and shown absent for all other actions; prefer animation ids and weapon states.
(src: MODLOG.md, Ratchet melee fix; docs/reference/LOCUS_Master_v1.7.md §17.4 lessons)

**9. Transient values within a frame.**
Symptom: a position spikes by +0.685 or dips by −0.015 for one sample, even when idle. Cause: the
game's collision code adds and subtracts an offset on a working copy (Ratchet's camera block
`+0x80`). Check: never smooth or drop samples; find the authoritative copy (the hero object's
position) and read it at a frame boundary. (src: investigations/ratchet.md "Repeated-Z diagnostic results", "Caller source")

**10. A matrix that looks like the camera but is not.**
Symptom: camera direction plausible in one state, wrong after moving. Cause: Ratchet's bone
matrices matched the camera-to-character direction. Check: a camera must change when only the
camera moves and stay when only the character moves; orthonormal; a few metres away.
(src: docs/piazza/piazza-integration-plan.md "Camera modes")

**11. Level exports do not match live memory.**
Symptom: object positions from a disc tool match nothing in RAM. Cause: the exported level
instances differ from the live object array (Ratchet). Check: trust live memory; use disc data
only for formats and assets. (src: docs/piazza/piazza-integration-plan.md "Invisible backend")

**12. Region and revision.**
Symptom: published addresses read garbage. Cause: another region or revision. Check: step 0
fingerprints; prefer the build public research targets. (src: investigations/spyro1-duckstation.md S0)

## Control and isolation (step 4)

**13. Pad order changes between launches.**
Symptom: the wrong pad drives the character, or two pads do. Cause: emulator binds by enumeration
order (PCSX2 swapped it). Check: hide every other device; ask the player to try each pad.
(src: MODLOG.md, Trial10 correction)

**14. Input dropped without focus.**
Symptom: the character ignores the pad when another window is in front. Cause: an input path that
requires focus (a browser gamepad API; PCSX2 before background events were allowed). Check: the
`isolation-idle` playtest with the world window in front. (src: docs/ratchet/04-controls/ratchet-focus-routing-result-2026-10-06.md)

**15. A pause button reaches the hidden game.**
Symptom: the game stops responding, or a held game starts running. Causes: a pad button bound to
the emulator's pause toggle resumed a held research game (~185 s unsupervised); in a cold boot,
in-level Start opened the game's pause menu. Check: no emulator hotkeys on pads; Start and Select
unbound on shared pads (Spyro); the ready script never presses Start in a level.
(src: MODLOG.md, Trial18; investigations/spyro1-duckstation.md S10)

**16. Frozen weapons.**
Symptom: the wrench sticks in his hand after a throw and later attacks are blocked. Cause: the
weapon's object was frozen with the level. Check: freeze by allow-list of classes; `item-throw`
scenario. (src: MODLOG.md, wrench throw follow-up)

**17. Writing past the end of the object table.**
Symptom: unrelated data corrupted. Cause: a freeze scanned a fixed 1,024 slots and modified 418
records past the array's end. Check: stop at the bound (Ratchet: after 8 empty slots) and refuse
entries that fail the guards. (src: docs/piazza/piazza-integration-plan.md "Invisible backend")

**18. Re-enabling a frozen object hangs the game.**
Cause: restoring a frozen object's update routine (Ratchet). Check: build the silenced state from
an unsilenced one instead of thawing. (src: MODLOG.md, Gate 5 Ratchet side)

**19. Freeing objects by overwriting them.**
Risk: a freed object stays in a collision chain or a list. Check: free the game's way (Spyro:
unlink from the region's collision chain, then state `0xFD`, as its own free routine does).
(src: investigations/spyro1-duckstation.md S9)

## Environment (step 5)

**20. Stale ground-triangle index.**
Symptom: within four frames of the collision swap he is moved by a large exact offset and falls
forever. Cause: each frame the game carries him by the movement of the triangle he stood on; after
the swap the stored index names another triangle (Spyro, `+0x274`). Check: after every swap,
clear cached ground references (write −1) with the CPU stopped and lift him slightly.
(src: investigations/spyro1-duckstation.md S7)

**21. Collision restored on respawn.**
Symptom: after dying he falls through the world until game over. Cause: respawn re-initialised
the level, restoring its collision and objects. Check: detect respawn and reinstall;
`hurt-and-respawn` scenario checks he lands. (src: investigations/spyro1-duckstation.md S10)

**22. Features outside the crop.**
Symptom: falls through at one place (Spyro on the north stairs). Cause: the stairs lay outside
the crop, and turned boxes were kept only when their centre was inside. Check: keep anything that
may reach into the crop; walls at the crop's edges in his world; stairs and edge scenarios.
(src: investigations/spyro1-duckstation.md S9)

**23. Capacity.**
Symptom: the world does not fit (Spyro: 548 bytes spare in the level's block). Check: record
capacity used; a window that follows him, rewritten near its edge. (src: investigations/spyro1-duckstation.md S9, S10)

**24. Coordinate limits.**
Symptom: geometry refused or behaviour odd far away. Causes: libsm64's grid of ±81.9 m (vertices a
few cm past it refused the whole world until out-of-grid faces were dropped and counted); Ratchet's
native Z must stay positive. Check: far-field scenario; declare the crop. (src: MODLOG.md, Phase A A5 fix; docs/HANDOFF.md "Known limits")

**25. Precision and winding.**
Check: snap to the format's packing grid (R&C 1/16 and 1/64), tile large faces crack-free, keep
the game's winding (Spyro: clockwise from above), and re-decode everything you wrote.
(src: docs/piazza/piazza-integration-plan.md; investigations/spyro1-duckstation.md S3)

## Presentation, scale and camera (steps 5–7)

**26. Scale judged in isolation.**
Symptom: the player says he is too small. Cause: units per metre chosen by guess (Spyro 1,000 →
720 → 560) and compared with a body box (1.1 m) instead of Ratchet's drawn height (1.41 m).
Check: compare drawn heights against the other characters and ask the player.
(src: investigations/spyro1-duckstation.md S9, S10)

**27. Renderer and connector disagree on the mapping.**
Symptom: the drawn character lags behind his hitbox. Cause: the world window was built before the
scale changed. Check: one mapping, carried with the data, not copied into two places.
(src: MODLOG.md, A7 Spyro third playtest fixes)

**28. Assets in unexpected places.**
Symptom: "model not found". Cause: the wrench model sat in the level's gadget table, not with
ordinary object classes. Check: search every section of the level's data. (src: docs/ratchet/05-wrench/ratchet-wrench-result-2026-10-07.md)

**29. Drawing what the game draws differently.**
Symptom: holes in Mario's face. Cause: decals (texture over the triangle colour) were alpha-masked.
Check: compare `render_preview` with the game's screenshot at close range. (src: MODLOG.md, Phase A A5 playtest; Piazza 9 Oct)

## Items (step 8)

**30. Objects recreated in new slots.**
Symptom: the wrench vanishes after respawn. Cause: the game recreated it in slot 300 (was 305).
Check: follow by class and owner with guards; `hurt-and-respawn` then `item-held`.
(src: MODLOG.md, wrench respawn)

## Damage (step 10)

**31. Level-dependent animations crash the renderer.**
Symptom: the game crashes on a hit (Spyro: invalid writes from his renderer, even without LOCUS).
Cause: each level loads only some of his animations; the contact hit asked for one Artisans lacks.
Check: list loaded animations; use only reactions that exist; run each hurt scenario three times.
(src: investigations/spyro1-duckstation.md S10)

**32. Game over ends the session.**
Check: `keep_at_least` on lives. (src: investigations/spyro1-duckstation.md S9)

**33. Native invincibility looks like missed hits.**
Symptom: only one hit in ~2 s lands on Mario under a combo. Cause: SM64's post-hurt blink. Check:
report invulnerability as the character's trait; do not fight it. (src: MODLOG.md, Phase A owner playtest of A4)

**34. Source ticks restart.**
Symptom: the world refuses every state after a respawn ("source timing did not advance").
Check: monotonic ticks across restarts. (src: investigations/spyro1-duckstation.md S10)

## Others solid (step 11)

**35. Moving a proxy he stands on hangs the game.**
Cause: his "standing on" references (Ratchet: four fields) still point at the proxy when it is
parked. Check: clear them before any large move; `dummy-walks-away` scenario.
(src: MODLOG.md, Gate 5 Ratchet side; docs/reference/LOCUS_Master_v1.7.md §17.4)

**36. Rescaled proxies are wide too.**
Cause: scaling one crate to 1.6 m made it 1.6 m wide. Check: stack small cubes to the height.
(src: MODLOG.md, Mario proxy height)

## Code changes (hooks)

**37. Code writes over PINE crash PCSX2.** SIGBUS. Use templates through `.pnach` or a paused
state. (src: MODLOG.md, Gate 9 start)

**38. Hooks in level code.** A hook in an overlay stops in the next level and a continuous patch
writes into foreign code. Hook the main executable. (src: MODLOG.md, Gate 9 cont.)

**39. Patch silently off.** Named `.pnach` groups are off by default in PCSX2 2.x; an unnamed
patch disables bundled patches. (src: docs/reference/LOCUS_Master_v1.7.md §30.2)

**40. Overlapping reservations.** Two patches in the same reserved heap. Record every reservation
and what is left of the game's heap. (src: docs/reference/LOCUS_Master_v1.7.md §30.2)

## Ready state (step 13)

**41. Fixed delays.** Flaky menus. Wait on observed conditions; prompts need ~3 s.
(src: MODLOG.md, Gate 9 cont.)

**42. Control starts late.** Input ignored during a level's opening camera (Ratchet until frame
~700–850). Wait on a frame threshold measured in two runs. (src: MODLOG.md, Gate 9 cont.)

## Sound (step 12)

**43. Rapid sounds collapsed.** Messages without event ids keep only the latest per topic. Every
trigger carries an id. (src: docs/core/character-sound-2026-10-11.md)

**44. Capturing the mix.** The emulator's output is the whole game (music, level). Follow the
sound table; never unmute or capture. (src: docs/core/character-sound-2026-10-11.md)

## Reporting

**45. Claims that were never run.** Ratchet's and Spyro's sound were built and unit-tested
without being heard, and recorded as "not heard" until the owner listened. Keep that discipline:
"built" is not "works". (src: docs/ratchet/06-sound/ratchet-sound-2026-10-11.md; investigations/spyro1-duckstation.md S12)
