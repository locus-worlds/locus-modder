# Attacks: unambiguous signals, anchored hitboxes

Step 9. LOCUS's world decides hits: the attacker's adapter declares when an attack is active and
where its volume is; the world tests it against each target's hurt volume, once per attack
instance, and sends an identified hit. (src: docs/reference/LOCUS_Master_v1.7.md §17.4)

## Find the signal

1. For each attack input: `input_script` the press (and combos, air versions, charged versions)
   while `mem_watch` follows the action/animation field, the state field, and every object he
   owns (weapon state).
2. Keep a signal only if it appears for that attack **and for no other action**: run walk, run,
   jump, double jump, land, hurt, idle and every other attack, and check it stays absent.
3. Prefer, in order: animation ids; the weapon object's own state; a state field with a documented
   meaning. Never a byte whose meaning you guessed.

| Character | Signals used | Trap |
| --- | --- | --- |
| Ratchet | animation 23, 24, 25 (combo), 43 (jump-slam); wrench object state 10 (outbound), 11 (return) | `+0x7E` looked like combo stage but was 3 on every jump |
| Spyro | state 11 (charge, observed); flame flag `+0x198` = 1 with frame `+0x1A0` in 10–32 | flame frame window from the decompilation, observed later |
| Mario | SM64 action flags (attacking `0x00800000`, air `0x00000800`), dive and ground-pound action values; stomp only when falling | — |

(src: adapters/rac1-ratchet/package.json; adapters/spyro1-spyro/package.json; adapters/sm64-mario/package.json; MODLOG.md, Ratchet melee fix)

## Declare it (format 0.1 shape; 0.2 keeps it)

- `trigger.when` / `unless`: conditions on fields (`in`, `above`, `below`, `flags_any`,
  `flags_none`). `unless` excludes overlaps (a swing while the wrench is thrown).
- `instance`: one hit per target per instance: `per_value` (each combo stage), `per_activation`,
  or `per_activation_of` a condition.
- `category` (`melee`, `projectile`, `explosive`, `stomp`, `beam`, `environmental`) and `tier`
  (`light` … `lethal`) calibrated in his own game: the fraction of a standard enemy's health the
  attack removes there. Reviewers check tiers against evidence; do not inflate.
- `max_reach`, `approach` (stomps: from above), `on_landed` (bounce).
- `hitbox`: a shape anchored on a **bone** (the weapon in the hand), an **object** (the weapon in
  flight) or the **body**; sized from his model (Ratchet's wrench capsule was measured from the
  wrench model in its grip pose; the hand anchor matched the object transform within 3 mm mean).
  A fallback shape for when the anchor is unavailable.

(src: adapters/rac1-ratchet/package.json; docs/reference/LOCUS_Master_v1.7.md §17.4)

## Check it

- `world_dummy(stand)` at 1 m in the arena; `dummy-<attack>`: exactly one hit per swing, right
  kind, category, tier.
- `dummy-<attack>-miss`: out of reach and facing away: no hit.
- `no-false-hits`: 20 s of movement beside the dummy: zero hits.
- Projectiles: the hit lands on the thrown object's path, once per throw including the return.
- Then the player in the arena: "does a hit register only when it visibly connects?"
- Owner feedback is the final judge of hitbox size ("more accurate" after anchoring on the wrench
  itself). (src: MODLOG.md, Phase A owner playtest of A4)
