# The game's own damage path

Step 10. When the world says he was hit, his own game must apply it: his flinch, his health, his
death and respawn. The world decides how much (native, normalised or custom policy); the adapter
says how his game takes it. (src: docs/reference/LOCUS_Master_v1.7.md §17.4)

## Find how the game applies damage

Let something in his own level hurt him while you watch (`mem_watch` on health, state, anything
that changes on the hurt frame). Then find what the game did, in this order of preference:

| Kind | What you write | Example |
| --- | --- | --- |
| A **flag word** his update checks each frame | `or_bits` | Spyro: `m_DamageFlags` (+0x2C): bit 0 = a hit (Sparx lost, state 14, 90 invulnerable frames); bit 10 on his last point = the struggle from which his own death and respawn follow |
| A **hit or event ring** the game fills when something hits | `ring_record` aimed at him | Ratchet: ring `0x178100` (64 records of 64 bytes, next index at `0x173E54`, pending record in the target object at +0xA4); his hit handler accepts a record aimed at his object with flag bit 0, takes whole damage from +0x2C, plays his flinch (animation 16) or at zero health his death (69) and respawn |
| A **function** | a `log_call`-like template, not available as a write; modules only | Mario: `sm64_mario_take_damage` with knockback (libsm64 API) |

Direct health writes are the last resort: no hurt animation, and the adapter must declare that
fallback. (src: investigations/spyro1-duckstation.md S9, S10; MODLOG.md, Gate 10; connectors/pcsx2/src/rac1.rs "hits on Ratchet"; docs/reference/LOCUS_Master_v1.7.md §17.4)

## Only reactions the level can play

Each level may load only some of his animations. Spyro's Artisans lacks his contact-bounce and
usual death animations; asking for the contact hurt crashed his renderer (also without LOCUS). So:

1. List the animations loaded in the ready-state level (step 6).
2. Map each candidate damage write to the reaction animation it asks for.
3. Use only reactions present; for the last hit, one that still leads to his own death.
4. Test each in the level the adapter plays in, three times.

(src: investigations/spyro1-duckstation.md S10; docs/reference/LOCUS_Master_v1.7.md §17.4 v1.7 lessons)

## Health, lives, respawn

- Health field and unit (Ratchet 4 at `0x1415F8`; Spyro's Sparx 3 → 0, then one more hit; Mario 8
  wedges). Declare `health.max` and a `damage_table` per tier in his units.
- Keep lives at least 3 (`keep_at_least`) so game over cannot end the session.
- Respawn is native: after it, reinstall world and silence (pitfall 21), rediscover items
  (pitfall 30), keep source ticks monotonic (pitfall 34). Where his death drops him far below the
  level, the world window shows him where he died.
- Report native invulnerability (blinking) as `invulnerable`, do not fight it (pitfall 33).

## Writes count once

A write is "applied" only when health actually dropped; retry a bounded number of times
(Spyro: up to 4 rewrites). Record applied, refused and failed outcomes in the trace.
(src: investigations/spyro1-duckstation.md S9)

## Check it

`world_dummy(attack_light_once)`, `world_dummy(attack_heavy_every_1s)` (behaviour names provisional); scenarios `hurt-light`,
`hurt-heavy`, `hurt-and-respawn`, each run three times: health drops by the table, his hurt
animation id appears, phases `dead` then `respawned`, he lands on LOCUS ground, items return. The
dummy's hit is `synthetic`; his reaction is `tested`. Then ask the player whether the reactions
look right.
