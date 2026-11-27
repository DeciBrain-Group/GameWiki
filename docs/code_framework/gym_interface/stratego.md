# Stratego — Interface Specification

<div class="badge-row"><span class="badge badge-turnbased">Turn-based</span></div>

Engine: **custom Stratego implementation** (`StrategoMultiAgentEnv`, procedural variant). Source: `DSGBench/games/stratego/` and `DSGBench/agent_manager/`. For game mechanics see the [Stratego game page](../../environments/stratego/stratego.md).

## Evaluation scenes

| Item | Setting |
|------|---------|
| Board | 10×10, two 2×2 lakes; 40 pieces per side (`STANDARD_STRATEGO_CONFIG`, `games/stratego/game/config.py:3-23`) |
| Mode | Agent vs agent — both sides are `StrategoAgent` with `LLMModel` (`game_mode: agent_vs_agent`) |
| Opponent | Evaluated model (Red) vs `gpt-4o-mini` (Blue), both prompted with `StategoPrompt` |
| Scenes | Scene 1 (`game_init: random`): Red always uses `STANDARD_INITS[0]`, Blue's layout is drawn at random. Scene 2 (`game_init: fixed`): Blue uses `STANDARD_INITS[1]`. Otherwise identical (`configs/eval_config_base/eval_stratego_agent_vs_agent_scene1.yaml` / `scene2.yaml`, `games/stratego/game/util.py:312-322`) |
| Matches per scene | 10 |
| Cap | 2000 half-turns (`max_turns`) |

Note: configs specify `agent_prompt: StategoPrompt`, but `agent_manager/prompts/__init__.py:3` aliases it to `StategoPrompt_v2` — the JSON-output variant used below.

## Decision cadence

One LLM call per attempted move. `stratego_play` calls `agent.step()` once per env step; `step` → `llm_gen` issues one `model.query_single_turn_gen(messages)` per retry iteration (`stratego_agent.py:125`). Each env step advances exactly one player's piece; the side is chosen by `current_player` (player 1 uses `self.model`, player −1 uses `self.model_opp`, `stratego_agent.py:105-114`). Invalid output triggers a re-query (see below), so the cadence is 1 call per ply plus retries. When `current_player == -1` the returned coordinates are flipped (`[9-int(t) for t in action_cmd]`, line 192) so Blue sees itself as "Red".

## Raw observation space

34-layer 10×10 integer array `StateLayers` (`stratego_procedural_impl.py:69-96,145-163`, wrapped as a partially-observable Box observation with `valid_actions_mask`, `stratego_multiagent_env.py:421-450`):

| Layer | Content |
|-------|---------|
| 0, 1 | Own / opponent piece layers (rank encoded by the `SP` enum: Spy=1 … Marshal=10, Flag=11, Bomb=12, UNKNOWN=13) |
| 2 | Obstacle (lake) layer |
| 3, 4 | Own / opponent partial-observability view layers |
| 5 | Scalar layer |
| 6, 7 | Move-tracking layers |
| 8–32 | Captured-piece count layers |
| 32, 33 | Not-yet-moved piece layers |

Unrevealed enemy pieces are UNKNOWN (13) — this is how fog of war is implemented.

## Raw action space

Flat 1D action index over "source square × (direction + distance)": `((r_src*10 + c_src) * (10+10)) + dst_offset`, giving `action_size = 10*10*(10+10)+1 = 2001` (`stratego_procedural_impl.py:253-277`). `env.step` receives `{player: 1d_index}`. An LLM cannot emit such indices reliably and would have no idea which moves are legal — hence the text wrapper below.

## Wrapped observation space

Pure text, built per move by `get_gameState` (`stratego_agent.py:299-341`):

| Field | Content |
|-------|---------|
| `## Board State` | 10×10 grid with `c0`–`c9` / `r0`–`r9` headers. Own pieces as `R(<symbol>)`; unrevealed enemies as `B(#)`; revealed enemies as `B(<symbol>)`; lakes as `~~~~`; empty as `....` |
| `## Valid moves` | Every legal move enumerated one per line (piece, source, destinations) |
| `## History moves` | Last 5 moves, most recent last |

??? example "User prompt (worked example embedded in the system prompt, lines 167-199 — same format as runtime output)"
    ```text
    ## Board State:
      ,  c0 ,  c1 ,  c2 ,  c3 ,  c4 ,  c5 ,  c6 ,  c7 ,  c8 ,  c9
    r0, R(²), R(¹), R(6), R(¹), R(¹), R(o), R(6), R(²), R(o), R(¶)
    r1, R(o), R(o), R(4), R(s), R(4), R(3), R(4), R(5), R(9), R(o)
    r2, R(3), R(¹), R(²), R(4), R(7), R(¹), R(¹), R(¹), R(6), R(¹)
    r3, R(o), ...., R(7), R(²), R(3), R(5), R(8), R(²), R(3), R(5)
    r4, ...., R(5), ~~~~, ~~~~, ...., ...., ~~~~, ~~~~, ...., ....
    r5, ...., ...., ~~~~, ~~~~, ...., ...., ~~~~, ~~~~, B(#), ....
    r6, B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#), ...., B(#)
    r7, B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#)
    r8, B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#)
    r9, B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#), B(#)

    ## Valid moves:
    Scout 'R(¹)' at position 2,1  could move to: 3 1
    Colonel 'R(7)' at position 3,2  could move to: 3 1
    Sergeant 'R(3)' at position 3,4  could move to: 4 4
    Captain 'R(5)' at position 3,5  could move to: 4 5
    Sergeant 'R(3)' at position 3,8  could move to: 4 8
    Captain 'R(5)' at position 3,9  could move to: 4 9
    Captain 'R(5)' at position 4,1  could move to any of: 3 1, 5 1, 4 0

    ## History moves: (history of the last 5 moves,The smaller the number, the closer it is to the current.)
    1. Sergeant 'R(3)' at position '2,8' moved to: 3 8
    2. Lieutenant 'R(4)' at position '3,2' moved to: 3 1
    ```

## Prompt structure

**System** prompt (`StategoPrompt_v2`, `agent_manager/prompts/stratego_prompt.py:101-210`): complete Stratego rules → special-piece abilities (Bomb / Spy / Scout) → piece-notation table → move instructions → worked board example → output JSON example. **User** message = the text board above.

??? example "System prompt (excerpt)"
    ```text
    You are an expert board game player and strategist, specialized in the classic board game Stratego. Please win the game as fast as possible.
    ...
    # Here are your instructions:
    1. Please analyze the current game state ,infer the intention of your opponent's operation, and give out your macro strategy, and select one of the valid moves available.  Opponent (Blue) pieces are marked with B# because we won't initially know their ranks.
    2. Given that current board configuration, and without making any assumptions about the opponent's pieces, please suggest a move for our side (Red).

    ## Note: Please present your answer, without any commentary, in the form: `r c  x y` where r is the row and c is the column of the piece you are suggesting to move and x and y are the destination rows and column, respectively.
    ## Remember the objective is the Blue flag and that it will probably be located in Blue's rear rows .
    ## IMPORTANT: Please win the game as fast as possible
    ```

??? example "Output example embedded in prompt"
    ```json
    {
      "reasoning": "string", // Explain your macro strategy and infer the intention of your opponent's operation and adjust your strategy .
      "move": "string" // the move that you choose without any commentary. Choose from Valid moves
    }
    ```

## Wrapped action space — `"r c x y"` string chosen from Valid moves

The model answers with a move string from the enumerated `## Valid moves` list; the instruction emphasises that `r c` must be a "position" from Valid moves and `x y` a "move to" from the same entry (`stratego_agent.py:285-294,338`). No numbering — selection is by copying the coordinate string.

## LLM output structure and parsing

`{"reasoning": "...", "move": "r c x y"}`. Parsing (`stratego_agent.py:347-374`): markdown JSON block extraction via marko, falling back to `yaml.safe_load`; the move string must be exactly 7 chars (`r`, ` `, `c`, ` `, `x`, ` `, `y`).

Validation and retry (lines 148-186): `(r,c)` must be a key of the valid-move dict and `[x,y]` a destination of that key. On failure: `record_llm_grounding_errors`, sleep 1s, retry; after more than 3 failures sleep 10s; after **≥ 10** failed generations fall back to `random_choice(valid_move)` (the warning text says "more than 50 times" — a stale log message; the actual threshold is 10). Note a wrong-length response increments the error counter but not the retry counter, so it loops without the cap. Special case: llama3.1:70b gets an extra post-processing LLM call that reformats the raw response into the JSON schema (lines 126-141). A valid `(r, c, x, y)` list is converted to the 1D index and passed to `env.step`; for player −1 the coordinates are flipped first.

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | ✅ Available | Per-match record: win/loss (`player`, `opp_player`), `match_turns`, live-piece state — `match_live_pieces_rate` (live/40), `match_live_pieces_score` (sum of ranks, max 148), `match_critical_live_pieces_rate` (Flag/Marshal/General/Colonel alive, /4) — plus the three `opp_` counterparts (`agent_eval.py:860-928`, `stratego_agent.py:205-271`) |
| Real-time decision | ➖ Not applicable | Turn-based; one call per ply, no timing pressure |
| Social reasoning | ⚠️ Limited | No dialogue; opponent-model inference is visible in the required `reasoning` field but no structured probe exists |
| Team collaboration | ➖ Not applicable | 1v1, no teammates |
| Adaptive learning | ⚠️ Partial | 10 matches per scene with per-side trajectories (`save_trajectory`); no cross-match memory. `grounding_acc = 1 − grounding_errors/generate_times` logged per match for both sides |

## Source code

| Part | File |
|------|------|
| Environment wrapper | `games/stratego/stratego_multiagent_env.py` |
| Board / action encoding | `games/stratego/game/stratego_procedural_impl.py`, `games/stratego/game/config.py` |
| Agent (text rendering, parsing, retry) | `agent_manager/agents/stratego_agent/stratego_agent.py` |
| Prompt | `agent_manager/prompts/stratego_prompt.py` (`StategoPrompt_v2`) |
| Evaluation loop | `agent_eval/agent_eval.py` (`stratego_play`) |
