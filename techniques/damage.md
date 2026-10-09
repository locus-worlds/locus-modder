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
| A **function** the game calls to hurt him (an event, a damage routine) | `call` through a `call_function` hook (PCSX2, vocabulary 0.5) | Jak 1: his process's event delivery with an attack message and record; Mario (a module): `sm64_mario_take_damage` with knockback (libsm64 API) |

Direct health writes are the last resort: no hurt animation, and the adapter must declare that
fallback. (src: investigations/spyro1-duckstation.md S9, S10; MODLOG.md, Gate 10; connectors/pcsx2/src/rac1.rs "hits on Ratchet"; docs/reference/LOCUS_Master_v1.7.md §17.4)

## When the game hurts him only through a call

Some games have no flag or ring to write: the hurt path is a function call (an event sent to his
object, a damage routine with a record of what hit him), and his health is never polled, so a
health write changes the number and nothing else. Then the adapter has LOCUS's `call_function`
hook template call that function, once per hit, with an argument record LOCUS writes. Everything
in it is data you find; the template's code is LOCUS's (`connectors` shows its layout).

1. **The function and its arguments.** Find who calls it in his own game (`code_xrefs` on its
   address; the boot executable's symbol table helps when it has one) and what each argument
   register holds: his object, the message, a pointer to a record. Write down the record's layout
   from the callers that build it (or from an outside source, facts only): which fields matter,
   which bits say a field is valid, which values are symbols or constants.
2. **The calling convention's context.** Registers the game keeps fixed while its code runs (a
   symbol table, the current object) must be set: `registers` takes constants, `load` (a word the
   hook reads at the call: the current value of a global), `field`, or `record` (the record's
   address). Give every loaded object a `not` list (0, the game's false or null) so the hook
   refuses rather than calls with nothing.
3. **The site.** A function's epilogue that the game's main thread passes every frame, outside
   his own update (so his state change waits for his next turn, as when an enemy hits him), with
   nothing live but what the epilogue restores. A pad read's caller is a good candidate; it cannot
   be the pad hook's own `pc`. `original` is the epilogue's words, `expect` the first.
4. **The record from the hit.** In `entity.apply_damage`: `{"call": {"hook": …, "record":
   {name: {at: "+0x…", type, value}}, "refused_when": […]}}`. Values: constants, `damage`,
   `direction.x|y|z` (attacker to target, his native axes) with `scale` (a knockback length),
   `source.x|y|z`, `tier` with `table` (one constant per tier: light, medium, heavy, lethal),
   `record` with `add` (a pointer into the record itself), `field:<name>`.
5. **His own rule decides.** `refused_when` lists the function's answers that mean "not hurt"
   (invulnerable, already hurt); those outcomes say so instead of claiming damage. How much a hit
   takes is often his game's own rule (one point per hit); declare `damage_table` to match it.
6. **Give it storage of its own.** Two pages no other hook (and no answer buffer) uses, found zero
   in your states and declared writable; the validator refuses overlapping storage and shared sites.
7. **Install it.** The hook is a guest patch: start research sessions with `session_start(hooks:
   true)` (every hook the adapter declares; `input: true` does the same); `adapter_try` notes a call hook missing from the game.

Check it as below, and watch his hurt state and health on the call's frame; a hit inside his
invulnerability must be refused, not counted.
(src: docs/core/damage-call-hook-2026-10-09.md)

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
