# Diplomacy — Interface Specification

<div class="badge-row"><span class="badge badge-turnbased">Turn-based</span></div>

Engine: **diplomacy engine** (Welfare variant). Source: `DSGBench/games/welfare_diplomacy/` and `DSGBench/agent_manager/`. For game mechanics see the [Diplomacy game page](../../environments/diplomacy/diplomacy.md).

## Evaluation scenes

| Item | Setting |
|------|---------|
| Map | `standard_welfare` (`MAP standard.map` + `RULES WELFARE`), 7 powers; game ends after W1910A (`final_game_year = max_years + 1900`, `welfare_diplomacy_env.py:46`) |
| Mode | Agent vs agent — all 7 powers are `WDAgent` / `LLMModel` |
| Roles per scene | Austria, England, France, Germany run `game_exploiter_model` (the evaluated model); Italy, Russia, Turkey run `game_super_exploiter_model` (`gpt-4o-mini`) |
| Hybrid agents | Super-exploiter powers use `ExploiterAgentNew`: while the trigger is active (enemy units ≤ 10 / adjustments year ≥ 1901 + max_years/3), orders come from the FPPI2 no-press RL network with zero LLM calls; reverts to the LLM policy when own centers > 10, and always for the final 2 years (`agents.py:513-716`, thresholds `RL_unit_threshold` / `RL_center_threshold`) |
| Scenes | 1 scene (`<model>_vs_gpt-4o-mini`); `game_max_message_rounds: 3`, `game_max_years: 10` |
| Matches per scene | 10 (`configs/eval_config_base/eval_welfare_diplomacy_agent_vs_agent_scene1.yaml`) |

## Decision cadence

Outer loop = once per phase (~10 years × M / R / A phases). Per phase: **3 message rounds** for movement / adjustments, **1 round** for retreats (no messaging). Each round, up to 7 powers (shuffled, called sequentially) each make one LLM call returning orders + messages; each round's orders overwrite the previous round's, and only the last round's orders execute (`agent_eval.py:428-657`). So roughly 21 LLM calls per M/A phase plus 7 per R phase. After each phase, one LLM summarizer call per power compresses the dialogue history (7 more calls per phase).

## Raw observation space

Full `Game` object state from the diplomacy engine — **no fog of war, the whole board is visible** (`welfare_diplomacy_env.py:50`, `env.py:112`):

| Part | Content |
|------|---------|
| Map | Locations, supply centers (`locs`, `scs`) |
| Per power | Units, centers, `welfare_points`, retreats |
| Global | `phase` (e.g. F1902M), `order_history`, `messages`, world state renderable as SVG |

## Raw action space

Two kinds (`env.py:82-105`, `message.py:30`):

- **Orders** — MILA abbreviation strings (e.g. `"A WAL - LVP"`, `"F NTH C A YOR - NWY"`), submitted via `game.set_orders(power, orders)`; `game.process()` advances the phase.
- **Messages** — `Message(sender, recipient, text, phase)` free text, recipient = one power or `"GLOBAL"`.

## Wrapped observation space

Pure-text user prompt assembled per call (`welfare_diplomacy_prompt.py:138-300`):

| Block | Content |
|-------|---------|
| `### Your Dialogue History ###` | Current-phase messages visible to this power (sender/recipient/GLOBAL filter); earlier phases compressed to an LLM JSON summary (`summary` / `negotiations_powers` / `allied_powers` / `betrayal_powers`, `message_summarizers.py:54,89`) |
| `### Recent Order History ###` | Orders of the last 3 phases |
| `### Current Supply Center Ownership ###` | SC ownership per power |
| `### Current Unit Ownership State ###` | All units with reachable destinations (convoy destinations marked `VIA`) / retreat options per phase type |
| `### Current Supply, Unit, and WP Count ###` | `{power}: {SC}/{units}/{welfare_points}` per line (`utils.py:70-85`) |

## Prompt structure

**System** prompt (`agent_manager/prompts/welfare_diplomacy_prompt.py:19-135`): role ("expert AI playing the game Diplomacy as the power {name}") → welfare rules (WP accumulation mechanics, the military-vs-welfare tradeoff) → JSON response format with per-field instructions → order-syntax examples. **User** message = the observation blocks above.

??? example "System prompt (excerpt)"
    ```text
    You are an expert AI playing the game Diplomacy as the power {power}.

    This is a variant called Welfare Diplomacy with the following rules:
    - In the build/adjustments phase (but not movement or retreats phases), you can freely disband or build any number of units ...
    - At the end of each build/adjustments phase, your power accumulates Welfare Points equal to the difference between your power's supply center count and unit count. WP continually add up each year—you can never lose them. ...
    - The game is on a turn timer and ends after W1910A. ... your goal is to maximize the total WP your power accumulates by the end of the game.

    It is currently the {phase} phase and message round {n} of 3 (the game will end in {years} years). Given this information, respond in the following JSON format to interact with the game and other players:
    {
        "reasoning": "A string of your private thoughts about your situation as natural language in under 500 words. ...",
        "orders": ["List of strings of orders you plan to make at the end of the turn to your units in the same abbreviated format as the history. ..."],
        "messages": {...}
    }

    Here are examples showing the format for orders:
    A LON H
    A WAL - LVP
    F SPA/NC - MAO
    A WAL S F LON
    A WAL S F IRI - LVP
    F NTH C A YOR - NWY
    A YOR - NWY VIA
    F IRI R MAO
    F IRI D
    A LON B
    ```

## Wrapped action space — legal orders enumerated by phase type

Legal orders come from `game.get_all_possible_orders()` and are presented differently per phase (`env.py:118`, `prompt.py:250-283,303`): movement phases list commandable units with reachable destinations; retreat phases list R/D options; adjustment phases list B/D options. Messages are represented as a `{country-or-Global: text}` dict, up to 3 rounds per phase (retreat phases forbid messaging).

## LLM output structure and parsing

Single JSON: `{"reasoning": str, "orders": [str], "messages": {recipient: str}}`. Parsing (`agents.py:297-370`): junk stripped, slice from first `{` to last `}`, `json.loads`; orders are coerced to a list of strings (anything else raises `AgentCompletionError`), message recipients uppercased, empty messages skipped, non-string messages stringified. Invalid orders are checked against `possible_orders` and skipped with an error log (`agent_eval.py:537-634`); `AgentCompletionError` skips that power for that round.

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | ✅ Available | Per power and phase, logged to wandb: `score/units/{power}`, `score/centers/{power}`, `score/welfare/{power}` (WP), board renderings, token / timing sums |
| Real-time decision | ➖ Not applicable | Turn-based with fixed message rounds |
| Social reasoning | ✅ Available | All messages logged (`wandb` table phase/round/sender/recipient/text); LLM-summarizer social metrics per phase: `social/allied_succ_rate`, `social/betrayal_rate`, `social/allied_power` pairs |
| Team collaboration | ✅ Available | Alliance behaviour measurable from message table + allied/betrayal summaries; coordination across the 4 allied exploiter powers is visible in orders + messages |
| Adaptive learning | ⚠️ Partial | 10 matches with per-power trajectories; `completion_error_table` and valid-order ratios logged; no cross-match memory. Note: the per-match eval matrix is currently empty — no win/WP summary is persisted to `eval_result.txt`, everything lives in wandb |

## Source code

| Part | File |
|------|------|
| Environment wrapper | `games/welfare_diplomacy/welfare_diplomacy_env.py`, `games/welfare_diplomacy/diplomacy/engine/env.py` |
| Agents (parsing, exploiter hybrid) | `agent_manager/agents/welfare_diplomacy/agents.py` |
| Prompt | `agent_manager/prompts/welfare_diplomacy_prompt.py` |
| Message summarizer | `agent_manager/agents/welfare_diplomacy/message_summarizers.py` |
| Evaluation loop | `agent_eval/agent_eval.py` (`wd_play`) |
