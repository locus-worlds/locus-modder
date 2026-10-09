# Ready state from power-on

Step 13. LOCUS never distributes a savestate. The adapter carries a recipe that takes the player's
own disc from power-on to a running, installed character, then saves a local state on their
machine. (src: docs/reference/LOCUS_Master_v1.7.md §30.3)

## Routes

| Route | How | Gives | Limits |
| --- | --- | --- | --- |
| New game | Scripted input from power-on | Start-of-game character, automatic | Only what the start has (Ratchet: wrench, no Clank) |
| Player's own save | Load their memory-card save, copied into the isolated profile | What they unlocked | They must have played there; the save stays on their machine |
| Unlock flags | Set ownership flags on the running copy | Selected items | Only where the level loads the item's assets and code; story side effects |

Choose the level the adapter plays in with care: it decides which of his animations exist (pitfall
31) and which objects you silence.

## Write it as observations

```jsonc
"ready_state": [                                   // field names provisional (design §5)
  { "boot": {} },
  { "until": { "field": "frame", "rises": true }, "timeout_s": 30 },
  { "press": ["start"], "when": { "field": "menu", "equals": 1 } },
  { "until": { "field": "level", "equals": 0 } },
  { "until": { "field": "level_frame", "at_least": 1000 } },
  { "install": ["world", "hooks", "silence"] },
  { "save_state": "ready" }
]
```

- Wait on fields, never on seconds alone. Useful signals: a global frame counter; a level id or
  level frame counter; menu or screen ids; the pad-read rate (about 60/s in engine, over 1,000/s
  during films in R&C); the emulator log's film lines; screenshots while you develop the script.
- Prompts ignore input for about 3 s after they appear: wait for the prompt, then press.
- Region and card prompts differ: with fresh empty cards R&C asked "memory card unformatted:
  format?" (No) then "continue anyway?" (Yes). Record each prompt and the answer.
- Films: skip with the game's own button once the film is detected.
- Never press Start in a level (pause menu).
- Control can start late (a level's opening camera): wait on a measured level frame.
- Input for the script comes from the `pad_inject` template (PCSX2: at `scePad2Read`'s epilogue in
  the main executable) or the game's pad buffer (DuckStation); restrict it to one socket and remove
  it when the ready state is saved, if it shares a reservation.

(src: MODLOG.md, Gate 9 start and Gate 9 cont.; docs/reference/LOCUS_Master_v1.7.md §30.2, §30.3)

## What R&C's cold boot looked like

Power-on, isolated profile, fresh empty cards, no savestate, controllers hidden: intro film →
title Start → New Game (Cross) → no format (Triangle) → continue (Cross) → films skipped with Start
→ Veldin running (level counter `0x15F3F8`), health 4, hero at spawn. Three consecutive hidden runs
reached the level in 59.7 s on the first attempt; an injected stick moved him about 4.5 m once the
level frame passed about 1,000. (src: docs/reference/LOCUS_Master_v1.7.md §30.3)

## Then install and save

On the cold-booted level: reserve guest memory if hooks need it, install hook templates, write the
world (step 5), silence objects (step 4), remove a boot-only pad patch, check every guard by
readback, and `state_save("ready")`. The play session starts from that local state, running.

## Closed by

Two consecutive cold runs reach the ready condition (spawn position, full health, level id), and
`adapter_test` passes from the saved state. Record each run's duration and every prompt met.
