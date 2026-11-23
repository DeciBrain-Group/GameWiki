# Environment Interfaces

This section documents how each DSGBench game is wrapped so that an LLM agent can play it and the evaluator can compute metrics from the logs. For every game it records the **raw** observation and action spaces exposed by the game engine, the **wrapped** observation and action spaces exposed to the agent, the **prompt** sent to the LLM, the **output** expected from the LLM and how it is parsed, and which **evaluation data** the wrapper makes available.

## Unified wrapping pipeline

<div class="system-overview">
  <div class="module-card"><h4>1 · Native environment</h4><p>Game engine or third-party API exposes the full game state and low-level commands.<br><em>→ raw state</em></p></div>
  <div class="module-card"><h4>2 · Observation wrapper</h4><p>Organises the raw state into structured fields, then renders them as text.<br><em>→ structured observation</em></p></div>
  <div class="module-card"><h4>3 · Prompt assembly</h4><p>System prompt + recent observations + executed / failed actions.<br><em>→ prompt</em></p></div>
  <div class="module-card"><h4>4 · LLM decision</h4><p>The model returns analysis, suggestions and a decision list.<br><em>→ raw LLM output</em></p></div>
  <div class="module-card"><h4>5 · Output parsing</h4><p>Decisions are extracted and mapped to action IDs (exact match or similarity search).<br><em>→ action IDs</em></p></div>
  <div class="module-card"><h4>6 · Action execution</h4><p>The wrapper expands macro actions into low-level commands and logs results.<br><em>→ executed / failed actions</em></p></div>
</div>

## Comparison across games

Summary only; click a game for the full specification.

| Game | Raw observation | Raw action | Wrapped observation | Prompt structure | Wrapped action | LLM output | Evaluation data |
|------|-----------------|------------|---------------------|------------------|----------------|------------|-----------------|
| [StarCraft II](starcraft2.md) | Full game state via python-sc2: units and structures (type, position, health), resources and supply, upgrades, map, visible enemies under fog of war | Unit-level commands (ability + target); combinatorially large | Six dictionaries (resource, building, unit, planning, research, enemy) rendered as text summaries; plus executed / failed actions and process data | System prompt (role, 6-part analysis, 4-part suggestions, 72-action dictionary, ask for 5 decisions) + last 5 summaries + executed / failed actions | `Discrete(72)` macro actions: train, build, research, other (scout, attack / retreat, chrono boost, empty) | Markdown: analysis → suggestions → `Decisions` numbered list of 5 action names | Strategic planning, real-time decision and adaptive learning data available; social reasoning needs probes; team collaboration not applicable |
| Civilization | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Street Fighter III | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Diplomacy | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Werewolf | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| Stratego | TODO | TODO | TODO | TODO | TODO | TODO | TODO |

!!! info "Why this matters for evaluation"
    Fine-grained metrics are computed from what the wrapper logs: step timestamps, parsed actions and their execution results, resource and technology states, probe answers and ground truth. Each game page therefore ends with an **Evaluation data** table stating, per capability dimension, which data is already logged, which needs annotation, and what is missing.
