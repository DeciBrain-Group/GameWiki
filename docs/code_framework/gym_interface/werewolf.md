# Werewolf — Interface Specification

<div class="badge-row"><span class="badge badge-turnbased">Turn-based</span></div>

Engine: **custom Python environment**. Source: `DSGBench/games/werewolf/` and `DSGBench/agent_manager/`. For game mechanics see the [Werewolf game page](../../environments/werewolf/werewolf.md).

## Evaluation scenes

| Item | Setting |
|------|---------|
| Players / roles | 8 players: 1 Seer, 1 Doctor, 4 Villagers, 2 Werewolves (`games/werewolf/config.py:25`, `runner.py:77-86`) |
| Mode | Agent vs agent — all 8 players are `WereWolfAgent` with `LLMModel` |
| Opponent | Scene 1: evaluated model on the village ("good") side vs `gpt-4o-mini` werewolves; Scene 2: `gpt-4o-mini` village vs evaluated model as werewolf — the same game, only the side of the evaluated model changes (`configs/eval_config_base/eval_werewolf_agent_vs_agent_scene1.yaml` / `scene2.yaml`) |
| Scenes | scene1 (`<model>_vs_gpt-4o-mini`), scene2 (`gpt-4o-mini_vs_<model>`) — 2 scenes per model |
| Matches per scene | 10 (`num_matches: 10`) |

## Decision cadence

Sequential: the environment returns an observation for **exactly one player at a time**, that player's agent makes **one LLM call**, then the env steps to the next action. Per round (`werewolf_env.py:38-39`):

```python
self.action_seq=['new_round','eliminate','protect','unmask','resolve_night_phase',
                 'check_for_winner','run_day_phase','vote','exile','check_for_winner','run_summaries']
```

LLM calls per round: 1 eliminate (one randomly chosen living werewolf decides for the pack, `game.py:82-83`), 1 protect (Doctor, skipped if dead), 1 investigate (Seer, skipped if dead), up to 8 debate calls (`MAX_DEBATE_TURNS = 8`, one per speaking player), 1 vote per living player, then 1 summarize per living player. A `bid` action exists in the prompt registry but is not used by the env's action sequence. The match ends when `check_for_winner` fires (werewolves outnumber the village, or both wolves are exiled).

## Raw observation space

Full game state as an in-memory object tree (`games/werewolf/model.py:71-530`):

| Part | Content |
|------|---------|
| `GameState` / `Round` | Round number, phase, per-round logs (votes, kills, debate transcripts) |
| `Player` | Name, role, personality, private observations; aliveness is implicit — a dead player is simply absent from `current_players` (no explicit alive flag) |
| `GameView` | Per-role view of the same state |

## Raw action space

A fixed state machine (`action_seq` above) driving 7 action types, each taking a player-name string or free text (`werewolf_env.py:38,146`):

- `remove` (werewolves kill), `protect` (doctor save), `investigate` (seer check), `debate` (public statement), `vote`, `summarize`, `bid`

This is not a flat action space an LLM could emit directly — targets must be valid living players, so the env enumerates legal options per call (see below).

## Wrapped observation space

`env.step()` returns one observation for the current player (`werewolf_env.py:166-172`, `model.py:146-179`):

| Field | Content |
|-------|---------|
| `player_name` | Name of the player whose turn it is |
| `game_state.name` / `role` | Who you are; werewolves additionally get `werewolf_context` (teammate identity, `model.py:309-313`) |
| `game_state.round` | Current round number |
| `game_state.remaining_players` | Living player names (shuffled order, with "You" marker) |
| `game_state.observations` | Private history grouped by round (moderator announcements, private chats) |
| `game_state.debate` | Statements so far this round, plus `debate_turns_left` |
| `game_state.personality` | Persona string (empty by default) |
| `action` | Which of the 7 actions is being requested |
| `options` | Legal target player names for this action (shuffled) |

## Prompt structure

Single **user** message rendered from a Jinja template (`agent_manager/prompts/werewolf_prompt.py:15-272`); there is no separate system prompt.

??? example "User prompt — template structure (excerpt)"
    ```text
    You are playing a digital version of the social deduction game Werewolf (also known as Mafia).

    GAME RULES:
    - Player Roles: {{num_players}} players - 2 Werewolves, 1 Seer, 1 Doctor, {{num_villagers}} Villagers.
    - Rounds consist of two phases:
        - Night Phase: Werewolves remove a player. Seer identifies a player's role. Doctor saves a player. If no one is removed, the Doctor saved the Werewolf's target.
        - Day Phase: Players debate and vote to remove one player.
    - Winning Conditions: Villagers win by voting out both Werewolves. Werewolves win when they outnumber the Villagers.

    GAME STATE:
    - It is currently Round {{round}}. {% if round == 0 %}The game has just begun.{% endif %}
    - You are {{name}} the {{role}}. {{werewolf_context}}
    - Remaining players: {{remaining_players}}

    OBSERVATIONS:
    {{observations}}

    ... (per-action block: DEBATE_SO_FAR_THIS_ROUND for debate, role-conditional INSTRUCTIONS)

    - If you are a Werewolf, your objective is to avoid suspicion and deflect attention...  (deception strategy)
    - If you are a Seer / Doctor, your objective is to ...  (information / protection strategy)
    ```

The final section is the output schema, also per action. `ACTION_PROMPTS_AND_SCHEMAS` (line 264) maps each action to its template and schema; rendering is `jinja2.Template(...).render(worldstate)` with the `game_state` dict plus a comma-joined `options` string (`were_wolf_agent.py:51-52,143-144`).

??? example "Output example embedded in prompt — vote"
    ```json
    {
      "reasoning": "string", // Explain your reasoning about who you should vote out. Avoid using violent or harmful language.
      "vote": "string" // Name of the player. Choose from: {{options}}
    }
    ```

## Wrapped action space — `Choose from: <names>`

The action space presented to the model is not a numbered list but a natural-language candidate string inside the output schema's `Choose from:` comment (`model.py:188-189`): e.g. `"Choose from: Derek, Scott, Jacob, Isaac"`. Different actions get different candidate sets (vote targets / kill targets / investigation targets). The `bid` action instead enumerates the meanings of values 0–4. No tool definitions, no action IDs — the answer must be copied verbatim from the candidate list.

## LLM output structure and parsing

Uniform two-field JSON: `{"reasoning": "...", <action key>: "..."}` with the action key being one of `say` (debate), `vote`, `remove`, `protect`, `investigate`, `summary`, `bid`.

- Legacy path (`games/werewolf/lm.py:40-102`): the schema is passed as `response_schema` to the API, the reply is extracted with `utils.parse_json`, and the result must be in the legal `options`; failure retries up to 3 times.
- Eval agent path (`were_wolf_agent.py:68-124`): up to **30 retries** — temperature 0.5 for constrained actions (vote / remove / investigate / protect / bid), 1.0 for free-text ones; after 30 failures it falls back to `random.choice(allowed_values)`. A llama3.1-specific repair prompt for the summarize format exists at lines 75-92.

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | ⚠️ Partial | Per-match result dict has only `player` (0 = Werewolves won, 1 = village won) and `match_turns` (`agent_eval.py:176-218`); goal decomposition must be derived from debate / vote logs |
| Real-time decision | ➖ Not applicable | Turn-based, sequential calls; no timing pressure |
| Social reasoning | ✅ Available | Debate transcripts and votes saved per round (`werewolf_env.py:105-134`); per-round wandb metrics **IRP** (identity inference accuracy = votes on werewolves / good votes), **VSS** (valid successful exiles), key-role survival **KSR** = (seer + doctor alive)/2 — logged per round to wandb, not persisted per match |
| Team collaboration | ✅ Available | Werewolf pair shares private `werewolf_context`; coordination (or its absence) is visible in night-kill and debate logs |
| Adaptive learning | ⚠️ Needs policy | 10 matches per scene are logged with per-agent trajectories and rewards (`{item_id, conversation, rewards}`, `were_wolf_agent.py:127-141`), but there is no cross-match memory |

## Source code

| Part | File |
|------|------|
| Environment wrapper | `games/werewolf/werewolf_env.py` (action sequence, per-phase stepping) |
| Game state / views | `games/werewolf/model.py`, `games/werewolf/game.py` |
| Agent and retry logic | `agent_manager/agents/were_wolf/were_wolf_agent.py` |
| Prompt templates | `agent_manager/prompts/werewolf_prompt.py` |
| Legacy LLM interface | `games/werewolf/lm.py` |
| Evaluation loop | `agent_eval/agent_eval.py` (`were_wolf_play`) |
