# Street Fighter III — Interface Specification

<div class="badge-row"><span class="badge badge-realtime">Real-time</span></div>

Engine: **DIAMBRA Arena** (MAME wrapper). Source: `DSGBench/games/streetfight3/` and `DSGBench/agent_manager/`. For game mechanics see the [Street Fighter III game page](../../environments/street_fighter/street_fighter.md).

## Evaluation scenes

| Item | Setting |
|------|---------|
| Game / ROM | Street Fighter III 3rd Strike, `sfiii3n` via DIAMBRA (`streetfight3_env.py:50,68-74`) |
| Characters | Ken vs Ken (`characters = ["Ken", "Ken"]`, super arts [3,3], outfits [1,3], `streetfight3_env.py:17-19`) |
| Mode | Agent vs agent — both sides are `StreetFight3Agent` LLM "robots"; there is no built-in AI difficulty |
| Scenes | Scene 1 (`asynch_mode: True`, asynchronous); Scene 2 (`asynch_mode: False`, synchronous) — otherwise identical (`configs/eval_config_base/eval_streetfight3_agent_vs_agent_scene1.yaml` / `scene2.yaml`) |
| Matches per scene | 10 |
| Opponent | Evaluated model (P1) vs `gpt-4o-mini` (P2) |

## Decision cadence

**Synchronous (scene 2):** every env frame calls `plan_act()` for both players inline — observe → plan → act; a `plan()` re-calls the LLM whenever the button queue is empty, so the frame loop blocks on LLM latency.

**Asynchronous (scene 1):** after reset, two daemon threads (`PlanAndActPlayer1/2`) run plan/act continuously in the background; each frame the env consumes and deletes the pending actions, so a frame whose action key is missing falls back to `0` (no-op) while the LLM is still thinking (`streetfight3_agent.py:78-174`, `agent_eval.py:226-257`).

In both modes `act()` pops exactly one button ID per frame from the queue; an empty queue returns `0`.

## Raw observation space

Raw DIAMBRA observation dict, passed through by reset/step (`streetfight3_env.py:76-83`; step additionally injects `_time_step`):

| Part | Content |
|------|---------|
| `obs['frame']` | RGB frame image, 384×224×3 — **not shown to the LLM** |
| `obs['P1']` / `obs['P2']` | Numerical state per player: `health`, `super_bar`, `wins`, … |
| `obs['timer']` | Round timer |
| `obs['_time_step']` | Frame counter added by the wrapper |

## Raw action space

Two discrete per-player action spaces of button IDs 0–17 (`config.py:5-30`): 0 = no-op, 1–8 = eight-direction movement, 9–14 = low/medium/high punch & kick, 15–17 = button combos. Special moves and combos are multi-frame key sequences (`COMBOS`, `SPECIAL_MOVES`).

## Wrapped observation space

The LLM never sees pixels. Positions are extracted from the frame by pixel-color detection (`observer.py:7-42`: scan rows 100–200 for Ken's red/green outfit colors, `KEN_RED = [248, 0, 0]`), cached for the last 10 frames, facing direction tracked by comparing x coordinates. This feeds `context_prompt()` (`robot.py:223-316`), a natural-language situation summary:

| Field | Content |
|-------|---------|
| Position | "You are very far / close to the opponent…", opponent on the left/right (normalized relative distance) |
| Health | "Your current health is {x}, and ennemy current health is {y}." |
| Super bar | Unlock hints at ≥30 (`Megafireball, Super attack 2`) and ≥120 or 0 (`Super attack 3, Super attack 4`) |
| Last actions | "Your last action was {x}. The opponent's last action was {y}." |
| Score | "Your current score is {reward}. You are winning / losing…" + advice lines |

## Prompt structure

**System** prompt = fixed template with `{character}`, `{context_prompt}`, `{move_list}` filled in; **user** message is the fixed string `"Your next moves are:"` (`robot.py:392-398`; the class's `user_prompt` template is not used — the string is hardcoded). o1-mini special case: the system prompt is moved into the user role (`robot.py:429-435`).

??? example "System prompt (template, verbatim)"
    ```text
    You are the best and most aggressive Street Fighter III 3rd strike player in the world.
    Your character is {character}. Your goal is to beat the other opponent. You respond with a bullet point list of moves.
    {context_prompt}
    The moves you can use are:
    {move_list}
    ----
    Reply with a bullet point list of moves. The format should be: `- <name of the move>` separated by a new line.
    Example if the opponent is close:
    - Move closer
    - Medium Punch

    Example if the opponent is far:
    - Fireball
    - Move closer
    ```

`{move_list}` lists the 21 macro moves by name (see below).

## Wrapped action space — 21 named macro moves

Semantic macro-action set `META_INSTRUCTIONS` (`config.py:114-140`): Move Closer, Move Away, Fireball (Hadouken), Megapunch (Shoryuken), Hurricane (Tatsumaki), Megafireball, Super attack 2/3/4, Jump Closer/Away, plus the individual punches and kicks. Each maps to a per-direction key sequence, e.g. Fireball → right `[7, 6, 5, 10]` (Down, Down+Right, Right, Medium Punch) / left `[7, 8, 1, 10]`. `plan()` looks up the sequence for the current facing direction and appends one no-op frame after every button (`NB_FRAME_WAIT = 1`).

## LLM output structure and parsing

Plain-text bullet list, one move per line. Parsing (`robot.py:335-372`):

??? example "Output (example, from the prompt itself)"
    ```text
    - Move closer
    - Medium Punch
    ```

- Regex `r"- ([\w ]+)"` extracts the move names; each is lowercased and checked against `META_INSTRUCTIONS_WITH_LOWER` (which also accepts aliases like `lower`/`med`/`high`).
- Invalid names are discarded; `grounding_errors` increments only when a single response contains more than one invalid move. The loop re-queries the LLM until at least one valid move comes back.
- Valid moves are expanded into the per-frame button queue; `act()` pops one button per frame, empty queue → 0.

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | ⚠️ Partial | Per-match record: `player` / `opp_player` (win), `match_turns` (frames), `match_time_use` (elapsed game timer); hit-derived per-frame metrics logged to wandb — **AHR** (attack hit rate), **SMHR** (super-move hit rate), **HCR** (health-consumption rate) — but not persisted per match (`agent_eval.py:220-371`) |
| Real-time decision | ✅ Available | Scene 1 vs scene 2 directly contrast async (actions fall back to 0 while the LLM thinks) vs sync (frame loop blocks); no-op frames caused by LLM latency are countable from frame logs |
| Social reasoning | ➖ Not applicable | No language; 1v1 fighting |
| Team collaboration | ➖ Not applicable | No teammates |
| Adaptive learning | ⚠️ Partial | 10 matches per scene with per-side trajectories; per-frame `grounding_acc = 1 − grounding_errors/generate_times` for both robots; no cross-match memory |

## Source code

| Part | File |
|------|------|
| Environment wrapper | `games/streetfight3/streetfight3_env.py` |
| Agent / robot (prompt, parsing, plan/act) | `agent_manager/agents/streetfight3_agent/agent/robot.py` |
| Pixel observer | `agent_manager/agents/streetfight3_agent/agent/observer.py` |
| Move tables | `agent_manager/agents/streetfight3_agent/agent/config.py` |
| Prompt template | `agent_manager/prompts/streetfight3_prompt.py` |
| Evaluation loop | `agent_eval/agent_eval.py` (`streetfight3_play`) |
