# <Game> — Interface Specification

<!-- 复制本文件到 docs/code_framework/gym_interface/<game>.md，按星际争霸 II 页面（starcraft2.md）的写法逐节填写。
     每个事实都要能在 DSGBench 仓库的 README、games/<game>/readme.md 或代码里找到；查不到写 TODO，不要猜。 -->

<div class="badge-row"><span class="badge badge-realtime">Real-time</span></div>
<!-- 回合制游戏改为 <span class="badge badge-turnbased">Turn-based</span>（类名以 docs/stylesheets/extra.css 为准） -->

Engine: **<engine / API>**. Source: `DSGBench/games/<game>/` and `DSGBench/agent_manager/`. For game mechanics see the [<Game> game page](../../environments/<game>/<game>.md).

## Evaluation scenes

| Item | Setting |
|------|---------|
| Map / setup | |
| Mode | |
| Players / roles | |
| Opponent | |
| Scenes | |
| Matches per scene | |

## Decision cadence

<!-- 多久调用一次大模型；一次调用输入多少轮观测；一次输出几个动作；同步还是实时 -->

## Raw observation space

| Part | Content |
|------|---------|
| Source | |
| Own side | |
| Others / opponents | |
| Global | |

## Raw action space

<!-- 原生环境接受什么形式的动作，规模多大，为什么大模型不能直接输出 -->

## Wrapped observation space

| Field | Content |
|-------|---------|
| | |

## Prompt structure

??? example "System prompt (excerpt)"
    ```text
    (paste from agent_manager/prompts/<game>_prompt.py)
    ```

??? example "User prompt (example)"
    ```text
    (paste one real example)
    ```

## Wrapped action space — `<type and size>`

| IDs / format | Category | Actions |
|--------------|----------|---------|
| | | |

<!-- 宏动作如何展开为底层指令；不可执行时如何反馈 -->

## LLM output structure and parsing

??? example "Output (example)"
    ```text
    (paste one real output)
    ```

Parsing: <!-- 如何从输出中提取动作（正则、JSON、相似检索），解析失败时如何处理 -->

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | | |
| Real-time decision | | |
| Social reasoning | | |
| Team collaboration | | |
| Adaptive learning | | |

<!-- Status 只用四种：✅ Available / ⚠️ Partial（或 ⚠️ Needs probes）/ ❓ To confirm / ➖ Not applicable -->

## Source code

| Part | File |
|------|------|
| Environment wrapper | |
| Observation and action execution | |
| Agent and decision cadence | |
| Prompt | |
| Output parsing | |
