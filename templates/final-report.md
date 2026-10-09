# <Character> adapter: report for <player>

<!-- Plain words for the player. Facts only; no game data, no screenshots of the game, no paths
     with personal names. Every "works" names the scenario or the player's answer behind it. -->

## What you have

- **Adapter** `<id>` version `<version>`, hash `<adapter hash>`, for `<game>` `<serial>` (image
  SHA-1 `<…>`), on `<runtime version, arch>` with connector `<version>`.
- It lives in your Workbench project. Nothing has been uploaded. **Publishing is your decision**
  (Developer ▸ Workbench ▸ Publish); before that a LOCUS reviewer with the same game reruns the
  scenarios.

## What works, and how we know

| Area | Result | Evidence |
| --- | --- | --- |
| Reads (position, facing, action, health, frame, camera) | | scenario `spawn-idle`, traces |
| Controls and isolation | | `isolation-idle`; your answer on pad routing |
| Ground and walls | | step-5 scenarios (list pass/fail) |
| Looks and size | | your answer |
| Camera | | `camera-forward`; your answer |
| Items and companions | | |
| Attacks | | `dummy-*` scenarios; your arena playtest |
| Hurt, death, respawn | | `hurt-*` scenarios (3 runs each); your answer |
| Others solid | | |
| Sounds | | `sound-triggers`; your answer per sound |
| Start from power-on | | two cold runs: <durations> |

## Not done or not supported

<Each item: what, why, what would be needed (for example a new LOCUS building block).>

## Not tested

<Everything not run: other levels, other regions, long sessions, playing beside other characters
online, Piazza (reachable only online).>

## Things to know

<Limits a player would notice: crop edges, missing reactions in this level, approximations.>

## Time spent

| Step | Hours |
| --- | --- |
