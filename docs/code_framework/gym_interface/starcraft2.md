# StarCraft II — Interface Specification

<div class="badge-row"><span class="badge badge-realtime">Real-time</span></div>

Engine: **python-sc2 (BurnySC2)**. Source: `DSGBench/games/starcraft2/` and `DSGBench/agent_manager/`. For game mechanics see the [StarCraft II game page](../../environments/starcraft2/starcraft2.md).

## Evaluation scenes

| Item | Setting |
|------|---------|
| Map | Ancient Cistern LE |
| Mode | Agent vs built-in AI (Medium) |
| Races | Protoss vs Protoss |
| Opponent strategy | Macro / Rush / Random, each in synchronous and asynchronous (real-time) mode — 6 scenes |
| Matches per scene | 10 |

## Decision cadence

The environment renders the structured observation as one L1 text summary per game step and appends it to a queue. The agent calls the LLM every **10 steps** (`action_interval=10`) with the **last 5 summaries** (`last_k=5`). Parsed actions are mixed with empty actions at a rate of **0.5** and queued; one action is executed per step. In asynchronous scenes the game keeps running while the LLM is thinking.

## Raw observation space

| Part | Content |
|------|---------|
| Source | `GameState` and bot attributes provided by python-sc2 at every game step |
| Own side | Unit and structure lists (type, position, health / shield, status); minerals, vespene; supply cap and used; upgrades |
| Enemy | Units and structures visible under fog of war |
| Global | Game time, map information, score (collected / spent statistics) |

## Raw action space

Commands to a single unit or a group (ability + target unit or position): move, attack, build, train, research, cast. Units and coordinates must be specified, which an LLM cannot produce reliably — hence the macro-action wrapper below.

## Wrapped observation space

Returned by `env.step()` as `state`:

| Field | Content |
|-------|---------|
| `information.resource` | Game time, worker supply, minerals, gas, supply left / cap / used, army supply |
| `information.building` | Count of each structure type (Nexus, Pylon, gas buildings, Warp Gate, Cybernetics Core, …) |
| `information.unit` | Count of each unit type (Probe, Zealot, Stalker, …) |
| `information.planning` | Structures and units being planned |
| `information.research` | Research status under each tech structure |
| `information.enemy` | Scouted enemy unit and structure types and counts |
| `action_executed` / `action_failures` | Actions executed in the last step; failed actions with reasons |
| `process_data` | `iteration`, `time`, supply, collected / spent minerals and vespene, `score`, number of completed technologies |

## Prompt structure

??? example "System prompt (excerpt)"
    ```text
    You are an AI trained in analyzing and summarizing StarCraft II games. You understand the nuances
    and strategies of the protoss race. Based on the summaries of multiple rounds in a game, we want you
    to analyze the game progression in a structured way. Your analysis should include:
    1. Game Overview  2. Current Game Stage  3. Our Situation (Units and Buildings / Economy / Technology)
    4. Our Strategy  5. Enemy's Strategy  6. Key Information
    ... provide specific suggestions for: 1. Our Strategy  2. Units and Buildings  3. Economy  4. Technology
    Lastly, ... make 5 actionable and specific decisions from the action dictionary
    {'TRAIN UNIT': {0: 'TRAIN PROBE', ...}, 'BUILD STRUCTURE': {19: 'BUILD PYLON', ...},
     'RESEARCH TECHNIQUE': {34: ...}, 'OTHER ACTION': {60: 'SCOUTING PROBE', ..., 71: 'EMPTY ACTION'}}
    ... avoid proposing actions that are not currently feasible.
    ```

??? example "User prompt (example)"
    ```text
    chunk{0}: At 10:07 game time, our current StarCraft II situation is as follows:

    Resources:
    - Game time: 10:07
    - Worker supply: 15
    - Mineral: 5390
    - Gas: 328
    - Supply left: 8 / Supply cap: 31 / Supply used: 23 / Army supply: 6

    Buildings:
    - Nexus count: 1 / Pylon count: 2 / Gas buildings count: 2 / Warp gate count: 4 ...

    Units:
    - Probe count: 15 / Zealot count: 2 / Stalker count: 1

    Planning: ...    Research: Warpgate research status: 1
    chunk{1} ... chunk{4}
    (followed by executed_actions and failed_actions)
    ```

## Wrapped action space — `Discrete(72)`

| IDs | Category | Actions |
|-----|----------|---------|
| 0–18 | Train unit | TRAIN PROBE, TRAIN ZEALOT, TRAIN STALKER, … MORPH ARCHON |
| 19–33 | Build structure | BUILD PYLON, BUILD GATEWAY, BUILD NEXUS, … BUILD SHIELDBATTERY |
| 34–59 | Research | RESEARCH WARPGATERESEARCH, RESEARCH BLINKTECH, weapon / armor / shield upgrades, … |
| 60–71 | Other | Scouting (Probe / Observer / Zealot / Phoenix), MULTI-ATTACK, MULTI-RETREAT, Chrono Boost, EMPTY ACTION |

Each macro action is handled by `handle_action_<id>` in the bot, which selects units and positions automatically. Infeasible actions are logged in `action_failures` and fed back in the next prompt.

## LLM output structure and parsing

??? example "Output (example)"
    ```markdown
    ### Game Overview / Current Game Stage / Our Situation / Our Strategy / Enemy's Strategy / Key Information
    ... (situation analysis)
    ### Suggestions:
    1. Our Strategy ... 2. Units and Buildings ... 3. Economy ... 4. Technology ...
    ### Decisions:
    1. **TRAIN STALKER**
    2. **BUILD PYLON**
    3. **BUILD WARPGATE**
    4. **RESEARCH PROTOSSGROUNDWEAPONSLEVEL1**
    5. **CHRONOBOOST CYBERNETICSCORE**
    ```

Parsing: a regular expression takes the text after `Decisions` → exact match against the 72 action names → unmatched decisions are mapped by similarity search in a vector store (Chroma + `all-mpnet-base-v2`) → if still unmatched, `EMPTY ACTION`. In the example, `BUILD WARPGATE` is not in the dictionary and is mapped by similarity search.

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | ✅ Available | `process_data` (completed technologies, score, supply, collected / spent resources) and per-step `information.building` / `research`; goal progress and tech-tree dependencies need annotation |
| Real-time decision | ⚠️ Partial | `action_executed` / `action_failures` and parsing results are logged; **per-call timestamps (observation delivered → LLM output returned) need to be confirmed**; critical events (e.g. enemy attacks) need annotated criteria |
| Social reasoning | ⚠️ Needs probes | 1v1 against built-in AI, no language interaction; probes can ask for the opponent strategy type (Macro / Rush), ground truth from scene config |
| Team collaboration | ➖ Not applicable | Single agent vs built-in AI, no teammates or communication |
| Adaptive learning | ✅ Available | Win / loss and score of 10 matches per scene; `action_failures` fed back each round allows checking repeated failures; cross-match memory policy must be specified |

## Source code

| Part | File |
|------|------|
| Environment wrapper | `games/starcraft2/agent_vs_computer_starcraft_env.py` |
| Observation and action execution | `games/starcraft2/bot/Protoss_bot.py` (`get_information`, `handle_action_*`, `get_process_data`) |
| Text summary | `games/starcraft2/summarize/L1_summarize.py` |
| Agent and decision cadence | `agent_manager/agents/starcraft2_agent/gpt4_agent.py` |
| Prompt | `agent_manager/prompts/starcraft2_prompt.py` |
| Output parsing | `games/starcraft2/utils/action_extractor.py` |
