# Civilization — Interface Specification

<div class="badge-row"><span class="badge badge-turnbased">Turn-based</span></div>

Engine: **civrealm (`FreecivLLM-v0`)** over a Freeciv web server. Source: `DSGBench/games/civ/` and `DSGBench/agent_manager/`. For game mechanics see the [Civilization game page](../../environments/civ/civ.md).

## Evaluation scenes

| Item | Setting |
|------|---------|
| Map | Three savegame maps, selected per scene: `myagent_T1_WORLDSMALL` (scene 1), `myagent_T1_FRANCE` (scene 2), `myagent_T1_BRITISHAISLES` (scene 3), loaded via `debug.load_game` (`games/civ/civ_env.py:18-20`) |
| Mode | Single `MastabaAgent` (`LLMModel`) controlling the `myagent` nation; opponents are the nations baked into the savegame (count / difficulty not specified in the repo) |
| Turn cap | 400 (`game_turn: 400`) |
| Matches per scene | 10; fixed seed 10 per match (`agent_eval.py:930`) |

## Decision cadence

Once per game turn per controllable entity, plus one advisor call per turn. `agent.act()` is called once per env step; when `info['turn']` changes, the agent spawns **one thread per unit and city** in `info['llm_info']`, each running its own persistent per-entity dialogue and `choose_action` → LLM (`mastaba_agent.py:46-58,177-199`). One extra advisor LLM call per turn produces the `Message from advisor` injected into every entity prompt. Cities are limited to one move per turn; a unit may be re-queried within the same turn on action conflict (up to `max_deconflict_depth=3`). Each decision can cost up to 10 LLM calls (see parsing below).

## Raw observation space

Structured text dict from civrealm (no pixels), consumed at `civ_env.py` / `language_agent.py:89-158`:

| Part | Content |
|------|---------|
| `obs['unit']` / `obs['city']` | Every entity with owner, `type_rule_name`, x/y, … |
| `obs['dipl']` | Diplomatic states |
| `obs['map']['status']` | Fog of war |
| `info['llm_info']` | Per-actor index (by `ctrl_type`): name, `available_actions`, `observations['minimap']` (surrounding 5×5 tiles, e.g. `tile_north_2_east_1`) and `['upper_map']` (25-tile block statistics) |
| `info['turn']` | Turn marker |

## Raw action space

Freeciv directives in LLM-readable string form (converted by `make_action_list_readable`, `baselang_agent.py:20`): unit actions such as `move North` / `move NorthEast`, `fortify`, `auto_explore`, `build`; city actions such as `produce Settlers`. At execution they are packed as `(ctrl_type, actor_id, action_name)` triples (`language_agent.py:152,176`).

## Wrapped observation space

Pure text, two layers (`mastaba_agent.py:60-199`):

| Field | Content |
|-------|---------|
| Advisor summary | Global per-turn advice on units / cities / war state |
| Per-entity prompt | Template `unit_obs_action.txt` / `city_obs_action.txt` filled with zoomed-out (`upper_map` block statistics) + zoomed-in (minimap tiles) observations, available actions, advisor message, scenario message |

## Prompt structure

Templates live in `agent_manager/prompts/civ_prompt_collections/` (loaded as `<% name %>` placeholders by `base_prompt_handler.py`). The instruction prompt (role + JSON rules + the two commands) and task prompt are loaded as user messages once per entity dialogue (`gpt_worker.py:64-70`); per unit type a different prompt set can be selected (`config.py:3-13`: Settlers / Explorers / Workers / fallback). There is no separate system message — everything is user-role dialogue history, with per-entity memory (dialogue truncated and summarized) and Pinecone manual retrieval (`manualAndHistorySearch`).

??? example "Instruction prompt (excerpt, `civ_prompts/instruction_prompt.txt`)"
    ```text
    You are an AI that helps users play the FreeCiv game with operations.
    Give the user advices to perform actions.

    ## Rules
    - Users will give you the actor name with their observation and availible actions.
    - Make a plan based on the following available commands.

    You should only respond in JSON format as described below:
    {
        "thoughts": {
            "thought": "<your current thought>",
            "reasoning": "<self reflect on why you made this decision>",
            "plan": "- short bulleted\n- list that conveys\n- long-term plan"
        },
        "command": {"name": "<command_name>", "input": {"<name>": "<value>"}},
    }

    Here are the possible command_name for you to use:
      manualAndHistorySearch:
        use: search the Freeciv official manual and your important history interaction message for relevant information. Use sparingly.
        input: "look_up": "<search terms>"
        output: list of search results
      finalDecision:
        use: make a decision for the user to perform an action in the game.
        input: "action": "<final decision>"
        output: nothing
    ```

??? example "User prompt (filled example, from `games/civ/readme.md:102-110`)"
    ```text
    You are controlling unit: Explorer 120.
    The zoomed-out observation is {'current_block': ['4 tiles unexplored', '10 Ocean', '8 Grassland', ...], 'block_north_1': ['25 tiles unexplored'], ...}.
    The zoomed-in observation is {'current_tile': ['1 Grassland', '2 Settlers', '2 Workers', '1 Explorer', 'unit owners are: myself player_0'], 'tile_north_1': ['1 Grassland', '1 Resources'], ...}.
    The available actions are ['fortify', 'move North', 'move NorthEast'].
    You should choose one of these actions according to the above observations.
    Message from advisor: Explorer should focus on exploring the map to find a suitable location for our first city. ...
    ```

## Wrapped action space — two commands, one valid action list

Only two commands exist: `manualAndHistorySearch` (look up the Freeciv manual + interaction history in a Pinecone vector index) and `finalDecision` whose `input.action` **must be one of the `available_actions` strings** of the current entity, else the worker appends an insist prompt and retries (`gpt_worker.py:108-131`). Anti-stuck heuristics detect repeated lookups / repeated same actions and force variety (`gpt_worker.py:79-95,124-129`).

## LLM output structure and parsing

Forced JSON: `{"thoughts": {"thought", "review", "reasoning", "plan"}, "command": {"name", "input"}}` (the `review` field appears in the unit-specific prompt sets). Parsing (`gpt_worker.py:157-167`): take the first `{` to the last `}`, append missing closing braces, `json.loads`. Failure handling: not-JSON → append `insist_json`; unknown command → `insist_available_commands`; action not in the available list → `insist_avail_action`. Up to 10 attempts per decision (`base_worker.py:114-166`), then a **random available action** is chosen (counted as invalid, lowering `grounding_acc`); llama3.1 gets an extra reformat-into-JSON post-processing call.

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | ✅ Available | Per-turn derived indices from `env.evaluate_game()` (`agent_eval.py:946-1026`): **EGI** (Economics Growth Index) = economics + production + gold; **UI** (Urbanization Index); **MPI** (Military Power Index); plus CER (combat efficiency), TPI (tech progress), CVP (wonders) — all logged per turn; match-end result comes from `env.get_game_results()` |
| Real-time decision | ➖ Not applicable | Turn-based |
| Social reasoning | ⚠️ Needs probes | Diplomatic state exists in observations but no dialogue / negotiation interface is exercised |
| Team collaboration | ➖ Not applicable | Single nation controlled by one agent |
| Adaptive learning | ⚠️ Partial | Per-turn wandb metrics (population, gold, cities, techs, …) + `grounding_acc = 1 − invalid_action_num/gen_action_num`; persistent per-entity dialogue memory within a match, no cross-match memory |

## Source code

| Part | File |
|------|------|
| Environment wrapper | `games/civ/civ_env.py` |
| Agent (threads, advisor) | `agent_manager/agents/civ_agent/mastaba_agent.py` |
| Turn / action handling | `agent_manager/agents/civ_agent/language_agent.py` |
| Workers (commands, parsing, retry) | `agent_manager/agents/civ_agent/workers/gpt_worker.py`, `base_worker.py` |
| Prompt templates | `agent_manager/prompts/civ_prompt_collections/` |
| Evaluation loop | `agent_eval/agent_eval.py` (`civ_play`) |
