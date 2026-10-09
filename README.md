# locus-modder

An agent skill for making **LOCUS character adapters**: it teaches a coding agent (Claude Code) to
bring a character from a game you own into LOCUS, working only through the LOCUS desktop app's
developer interface and testing every step in the app's Proving Ground.

An adapter is a JSON file: facts about where a character lives in your copy of the game and how
LOCUS should run it. It never contains code or anything taken from the game. Everything the agent
extracts while researching stays on your machine.

## Install

```sh
git clone https://github.com/locus-worlds/locus-modder ~/.claude/skills/locus-modder
```

Then, in Claude Code, the skill is available in every project (`locus-modder`).

## Use

1. Open the LOCUS desktop app, go to **Developer ▸ Workbench**, create an adapter project and
   copy its agent configuration.
2. Add the LOCUS developer interface to Claude Code with the command the Workbench shows
   (`claude mcp add locus …`), in the folder you will work in (or with `--scope user`).
3. Start Claude Code and ask, for example: *"Use the locus-modder skill to make a LOCUS adapter
   for <character> from <game>. The disc image is at <path>."*
4. Approve the agent's requests and answer its questions in the Workbench.

Start with [SKILL.md](SKILL.md): the procedure, the rules, and the tool reference. Playbooks for
PCSX2 and DuckStation, pitfalls, techniques, templates and field notes are in the folders beside it.

## Status

Built from three integrations made by hand (Mario via libsm64, Ratchet & Clank on PCSX2, Spyro the
Dragon on DuckStation). Tool names follow the LOCUS developer interface; some tools are still
being finished (loading a draft adapter into the game, generic drawing of skinned characters). The
`src:` notes cite records in the LOCUS project repository, which is not public yet.
