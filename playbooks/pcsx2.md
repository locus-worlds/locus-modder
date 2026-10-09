# PCSX2 playbook (PS2)

What LOCUS learnt running Ratchet & Clank (SCUS-97199) in PCSX2 on macOS, 5–11 October 2026. The
app's `connector-pcsx2` does the plumbing; this tells you what it does, what it cannot do, and what
to expect. Facts with a `src:` are LOCUS reproduced unless marked otherwise.

## Versions

- Investigation began on v2.8.2; play is pinned to stock **v2.9.103** (commit 6f18caf2), an x86_64
  build under Rosetta. **arm64 builds are untested**: on a new Mac, record the architecture
  `runtimes` reports and treat the first session as a test of the runtime itself.
  (src: docs/reference/LOCUS_Master_v1.7.md §30.2; docs/ratchet/04-controls/ratchet-focus-routing-result-2026-10-06.md)
- Record the BIOS the player's profile uses (name and region) in S0; never copy or hash it into
  the project.

## Profile (what `session_start` builds)

- A copy of the player's profile under the run folder, started with `-datapath <run>` and
  `-statefile <state>` when resuming. Keys forced once each: `PINESlot` (one slot per session),
  `StartPaused = false`, `TogglePause = Keyboard/P`, memory-card folders inside the run (fresh,
  empty cards, which avoids the player's saves and their dialogs), and for hidden sessions GS
  `Renderer = 11` (Null) and `HideMainWindowWhenRunning = true`, audio muted. Pad bindings are
  rewritten to `SDL-0`. (src: connectors/pcsx2/src/runtime.rs)
- PINE is off by default in a fresh profile (`EnablePINE`); default slot 28011.
  (src: investigations/ratchet.md "Current LOCUS profile correction")

## Memory: PINE

- Opcodes: Read32 `0x02`, Write32 `0x06`, SaveState `0x09`, LoadState `0x0A`, Status `0x0F`
  (0 running, 1 paused). Framing: a little-endian u32 total length (including itself), then opcode
  and arguments; replies carry a u32 length and a result byte (`0x00` OK, `0xFF` fail).
  (src: investigations/ratchet.md "Verified PINE wire contract"; docs/reference/LOCUS_Master_v1.7.md §30.2)
- **PINE cannot resume a paused VM.** Play states must be saved running; a state that loads paused
  stays paused.
- **One client at a time.** A second client connects but times out. Never leave a probe attached.
- **Batches are not atomic.** Reads run one after another on PCSX2's IPC thread; a batch is not a
  frame snapshot, and host time is not game time. Use the game's frame counter.
- Reads go through EE guest mappings (guest addresses, not host pointers).
- Cost observed: the pose stream about 108,000 words/s and the sound poller about 42,000 words/s
  ran together. 6–7 failed requests right after PCSX2 starts were seen twice (cause open): retry at
  start, do not treat them as a broken route. (src: docs/ratchet/06-sound/ratchet-sound-2026-10-11.md; investigations/spyro1-duckstation.md S10)
- **Never write code over PINE while the VM runs**: it crashed PCSX2 (SIGBUS on a protected host
  page, the recompiler's code cache). Write only data over PINE. Code changes go through a `.pnach`
  patch or a paused state; `mem_write` enforces this. (src: MODLOG.md, Gate 9 start)

## Debugger and static analysis

- `breakpoint` is DuckStation-only in the developer interface. On PCSX2 find writers statically
  (`code_disassemble`, `code_xrefs` from a local Ghidra project with a PS2/R5900 processor module)
  and confirm by behaviour (`mem_watch`). The Ratchet investigation used PCSX2's debugger window by
  hand; its traps are in [pitfalls](../pitfalls.md) (an editable memory view; Pause re-hitting the
  current breakpoint). (src: docs/ratchet/03-pose-capture/ratchet-re-tooling-result-2026-10-06.md; docs/ratchet/02-collision/ratchet-native-floor-result.md)
- R5900 specifics that matter when reading code or filling a hook template: 128-bit GPRs (save and
  restore them whole), branch delay slots, VU0 macro mode and VU1 microprograms.

## Hidden play and screenshots

- Null renderer, main window hidden when running, muted, minimised. Game logic, VU work and the
  capture hook still run; 60 poses/s were streamed with the Null renderer.
  (src: docs/piazza/piazza-integration-plan.md "Invisible backend")
- The Null renderer draws nothing, so `screenshot(game)` probably needs a session started
  `visible: true` (inference; check what the app does). Use a visible session for screenshots and
  a hidden one for timing.
- Hide through `NSRunningApplication.hide`; the System Events route needs an automation permission
  that rebuilt apps lose. macOS blocks synthetic keystrokes; PCSX2's screenshot hotkey can be sent
  to the process (`CGEvent.postToPid`). (src: investigations/spyro1-duckstation.md S10; docs/reference/LOCUS_Master_v1.7.md §30.2)

## Controllers

- PCSX2 binds pads by SDL enumeration order, and **the order changed between launches** (one trial
  had the PS5 pad driving the PS2 game). So every device except the assigned one is hidden with
  `SDL_GAMECONTROLLER_IGNORE_DEVICES` (vendor/product, e.g. `0x054C/0x0CE6` for a DualSense) and
  the assigned pad is `SDL-0`. `SDL_JOYSTICK_ALLOW_BACKGROUND_EVENTS=1` lets the hidden game read
  it without focus; PCSX2 2.9.103 has no pad focus gate. (src: MODLOG.md, Trial10 correction; docs/ratchet/04-controls/ratchet-focus-routing-result-2026-10-06.md)
- Check routing with the player every new session layout (step 4).

## Savestates

- `.p2s` files are zip archives using a compression `unzip` cannot read; `state_extract` handles
  them. Parts: `eeMemory.bin` (32 MB EE RAM), `iopMemory.bin` (IOP RAM: 989snd banks and strings),
  `SPU2.bin` (sound memory). (src: docs/reference/LOCUS_Master_v1.7.md §30.2; docs/ratchet/06-sound/ratchet-sound-2026-10-11.md)
- A state captures everything: objects, collision, hooks you installed. A "play state" is built
  from a ready state by installing, then saving. Never distribute a state; step 13 rebuilds it on
  each machine.

## Code layout: overlays

- Code above roughly `0x200000` is **level code, replaced when a level loads** (an overlay). A hook
  there stops working in the next level, and a continuously applied patch then writes into foreign
  code. Hook the main executable, for example an SDK routine. Ratchet's first pad hook at
  `0x217278` failed in-level for this reason. (src: MODLOG.md, Gate 9 cont.; docs/reference/LOCUS_Master_v1.7.md §30.2)
- Data in the main executable (below the overlay) is stable across levels: prefer tables there
  (Ratchet's sound channel table `0x13E5C0`).

## Patches (`.pnach`) and hook templates

- The connector applies hook templates as a `.pnach` or into a paused state, then verifies by
  readback.
- **Named `.pnach` patch groups are off by default** in PCSX2 2.x; they are enabled per game in
  `gamesettings/<serial>_<crc>.ini` (`[Patches] Enable = <name>`). An unnamed patch disables
  PCSX2's bundled patches for the game. If a hook does nothing, check this first.
- **Heap reservations.** Hook code and its buffers need guest memory nobody else uses. Ratchet's
  capture hook took 16,000 bytes at `0x01FF8000–0x01FFBE80` by moving the SDK heap break (kernel
  heap end `0x01FFC000`, main-thread stack above it), leaving 384 bytes of SDK heap. Another patch
  must not overlap a reservation; a boot-only patch can live in it and be removed when the play
  state is built. A reservation that starves the game's allocator is a risk: record what is left.
  (src: docs/ratchet/03-pose-capture/ratchet-guest-hook-result-2026-10-06.md; docs/reference/LOCUS_Master_v1.7.md §30.2)
- **`copy_on_execute`** (Ratchet's pose): placed at the producer site (`0x251C9C`, where the game
  finishes the 111-bone palette), copies 7,104 palette bytes and metadata into one of two owned
  slots with a ready token; the reader validates guards and releases the slot. For play it skips a
  frame instead of halting when a slot is busy. Reached 58–60 poses/s, every copy fresh.
  (src: docs/HANDOFF.md "Key facts"; docs/piazza/piazza-integration-plan.md "Continuous play mode")
- **`pad_inject`** (cold boot): at `scePad2Read`'s epilogue (`0x124C94` in R&C, main executable),
  `j hook` with `move $8, $18` in the delay slot gives the caller's raw pad buffer: buttons in
  bytes 0–1, active low; left stick bytes 4–5. Inject on one socket only.
  (src: MODLOG.md, Gate 9 cont.; docs/reference/LOCUS_Master_v1.7.md §30.2)
- **`log_call`**: not yet used in a shipped integration; polling a table was enough for sound.

## Observe, don't time

- Prompts ignore input for about 3 s after they appear. The pad-read rate tells you where the game
  is: about 60 reads/s on in-engine screens, over 1,000/s during films; PCSX2's log names film
  playback. In-level Start opens the pause menu (it made early cold boots look stuck).
- Control may begin after a level's opening camera: Ratchet answered the stick only from level
  frame ~700–850 (ready point 1000). (src: MODLOG.md, Gate 9 cont.; docs/reference/LOCUS_Master_v1.7.md §30.3)

## Engine and coordinate notes (Insomniac, R&C; likely different elsewhere)

- Z up; camera world matrix rows forward/left/up, position at +0x30. Mapping used:
  native = (x + 129, 123 − z, y + 30), scale 1 native unit per metre. (src: docs/HANDOFF.md "Game memory map")
- Frame counter `0x15F3F8` (increments after the display wait). (src: docs/ratchet/02-collision/ratchet-native-room-trial.md)

## Sound hardware and libraries

- The PS2 SPU2 uses the **same ADPCM block format as the PS1 SPU** (16-byte blocks, 28 samples,
  loop and end flags); the connector decodes it.
- **989snd** (Sony's library, used by R&C and by Jak 1) runs on the IOP. Signs: the IOP banner
  string "989snd"; an EE-side command sender; per-level banks (`SBlk` header, sounds, grains,
  sample data) on the disc and loaded into IOP RAM and SPU2 RAM. Grains tick at 240 Hz.
  OpenGOAL's reimplementation documents the format (facts only).
  Details: [techniques/sound.md](../techniques/sound.md). (src: docs/ratchet/06-sound/ratchet-sound-2026-10-11.md)

## Hygiene

- Every research run is a separate isolated session on its own PINE slot, every controller hidden
  unless the step needs one. The app stops sessions and reports leftovers; read the report. An
  orphaned script once polled a dead socket for two hours. (src: docs/reference/LOCUS_Master_v1.7.md §30.2)
- After Ratchet walked off the edge of an early test floor and fell, PINE status requests stalled.
  Keep him inside the world (step 5's edge walls) before long runs. (src: docs/piazza/piazza-integration-plan.md)
