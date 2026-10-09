# Evidence records

Two places hold evidence: the project's evidence store (`evidence_record`, local, with artefacts)
and the adapter's `evidence` array (published with the adapter: claims and levels only, no
artefacts). (src: docs/core/locus-modder-design-2026-10-11.md §3.1, §8)

## `evidence_record` (local)

```jsonc
// argument names provisional (design §3.1: evidence_record(claim, level, artefacts))
{
  "claim": "Hero position is f32 x, y, z at hero+0x10, Z up, game units",
  "level": "LOCUS reproduced",                 // or documented | inference | synthetic
  "step": 3,
  "build": "SCUS-97124, boot ELF sha256 <…>",
  "runtime": "PCSX2 <version> <arch>, connector pcsx2 <version>",
  "artefacts": ["snapshot:<id>", "trace:<id>", "state:<id>", "scenario-run:<id>"],
  "method": "Hold right 60 frames then left 60 frames in session A; repeat with up/down in session B",
  "observed": "x +3.1 then -3.0 m; y unchanged; z rose 1.2 on jump and returned",
  "not_tested": "Values during cutscenes; after a level change",
  "disproves": null                             // or the claim this one disproves
}
```

Rules:

- One claim per record; numbers, not adjectives.
- `LOCUS reproduced` needs artefacts from this project's sessions or the player's own files.
- `documented` names the source, its licence and pinned commit; no copied text beyond a field name.
- A player's answer is evidence: cite the `ask_player` or `request_playtest` id and quote the answer
  briefly.
- A disproved claim is never deleted: add a record that disproves it.

## Adapter `evidence` (published)

Format 0.1 (src: docs/schemas/locus-adapter-0.1.schema.json); 0.2 may extend it:

Ratchet's adapter (src: adapters/rac1-ratchet/package.json):

```json
{ "claim": "Wrench animations 23, 24, 25 (combo) and 43 (jump-slam); +0x7E is not a swing signal",
  "level": "tested",
  "record": "docs/handoffs/chat-handoff-2026-10-08.md" }
```

`level` is one of `tested`, `research`, `inference`, `synthetic`. In your adapter, `record` names
the investigation sheet section (for example `notes#S9`; provisional until 0.2 says how records
are referenced); nothing game-derived. Each attack, damage
route, collision route, presentation route and sound route has at least one entry. Tiers and
hitbox sizes that are judgement stay `inference` until a scenario and the player confirm them.
