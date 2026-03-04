<div class="game-header">
  <div class="game-header-icon">⚔️</div>
  <div class="game-header-info">
    <h1>StarCraft II</h1>
    <div class="badge-row">
      <span class="badge badge-realtime">Real-time</span>
      <span class="badge badge-imperfect">Imperfect Info</span>
    </div>
    <p>Real-time strategy game demanding resource management, base building, and army control under fog of war — the gold standard for complex strategic AI evaluation.</p>
  </div>
</div>

## Game Overview

**StarCraft II** is a real-time strategy (RTS) game where players compete to gather resources, build bases, and control armies to defeat opponents. Players must make strategic decisions continuously in real-time, balancing economy, production, and combat — all while operating under the fog of war that limits map visibility.

DSGBench uses the **TextStarCraft2** environment, which wraps StarCraft II with a text-based interface so LLM agents can receive structured observations and issue natural language commands.

### Why StarCraft II for LLM Evaluation?

- **Real-time decision pressure** — actions must be issued continuously without turn-based pauses
- **Fog of war** — agents must scout, infer, and anticipate under imperfect information
- **Multi-objective management** — simultaneously balance economy, production, and combat
- **Long-horizon planning** — build orders and tech trees require planning dozens of steps ahead
- **Opponent adaptation** — strategies must shift in response to enemy composition and aggression

---

## Core Challenges for LLM Agents

| Challenge | Description |
|-----------|-------------|
| **Resource Management** | Gather minerals and vespene gas efficiently; balance spending across units, buildings, and upgrades |
| **Base Building** | Construct and upgrade structures in the right order to unlock technology advancements |
| **Army Control** | Train units, position armies, and execute attacks and retreats in real-time |
| **Fog of War** | Operate with limited map visibility; scout enemy positions and anticipate attacks |
| **Tech Tree Navigation** | Choose the correct research path based on opponent strategy |
| **Macro vs. Micro** | Balance high-level economic planning with low-level unit micro-management |

---

## Observation Space

Each step, the agent receives a structured text observation containing:

| Component | Description |
|-----------|-------------|
| **Map View** | 2D grid of the map with terrain, structures, and visible units; includes unit types, positions, health, and ownership |
| **Resources** | Current minerals and vespene gas amounts; available workers and resource nodes |
| **Base State** | Active structures, ongoing production, tech upgrades, and army composition |
| **Enemy Intel** | Visible enemy units and structures (limited by fog of war) |
| **Game State** | Current game time, supply count, and alert conditions |

**Data formats:**

- Numerical: resource counts (integers), unit health (floats)
- Categorical: unit types (`marine`, `zergling`), structure states
- Spatial: unit positions as (x, y) coordinates on the map grid

---

## Action Space

The action space covers both high-level strategic commands and low-level unit control:

| Category | Example Actions |
|----------|----------------|
| **Unit Commands** | `move(unit_id, x, y)`, `attack(unit_id, target)`, `patrol(unit_id, waypoints)` |
| **Base Management** | `build(worker_id, structure)`, `train(structure, unit_type)`, `research(structure, upgrade)` |
| **Macro Actions** | `expand(worker_id, location)`, `rally(units, point)`, `scout(unit_id, area)` |

**Action representation:**

- **Discrete**: Select from a predefined set of named actions (e.g., `train_marine`, `build_barracks`)
- **Positional**: Target specific map coordinates (e.g., `move_to(32, 47)`)

---

## Evaluation Scenes

| Scene | Description | Key Skill Tested |
|-------|-------------|-----------------|
| **1v1 Standard Match** | Agent vs. AI-controlled opponent at varying difficulty levels | Overall strategic capability |
| **Skirmish Mode** | Small-scale focused combat scenarios with pre-set unit compositions | Unit micro-management and tactical decisions |
| **Macro Management** | Extended games emphasizing base expansion and resource optimization | Long-horizon economic planning |
| **Fog of War** | Scenarios with limited map visibility and hidden enemy positions | Scouting, inference, and counter-strategy |

---

## Evaluation Metrics

| Metric | Description |
|--------|-------------|
| **Win Rate** | Percentage of matches won against AI opponents at various difficulty levels |
| **Resource Efficiency** | Rate of resource gathering and spending relative to optimal benchmarks |
| **Action Accuracy** | Effectiveness of executed actions (unit command precision, build order correctness) |
| **Adaptability** | Speed and quality of strategy adjustment in response to enemy actions |
| **Combat Effectiveness** | Ratio of enemy units destroyed to own units lost (kill/death ratio) |
| **Exploration Efficiency** | Map coverage rate and quality of enemy intelligence gathered |

---

## Prompt Example

A typical LLM agent prompt in the StarCraft II environment:

```plaintext
You are a commander in a real-time strategy game. Your goal is to defeat
the enemy by gathering resources, building structures, and controlling
your army. The map is partially visible, and you must anticipate enemy
actions.

Current state:
- Resources: 500 minerals, 200 vespene gas
- Army: 10 Marines, 5 Siege Tanks
- Alert: Small Zergling group sighted approaching your main base

What will you do next? Provide a specific action plan.
```

---

## Setup & Dependencies

```bash
# Download StarCraft II from Blizzard and install
# Place the evaluation map at:
StarCraft II/Maps/Ancient Cistern LE.SC2Map
```

See [Quick Start](../../getting_started/quickstart.md) for full setup instructions.

---

## Resources

- [TextStarCraft2 GitHub](https://github.com/histmeisah/Large-Language-Models-play-StarCraftII) — the underlying text-based SC2 environment
- [StarCraft II Official Site](https://starcraft2.blizzard.com/)
- [DSGBench Paper](https://arxiv.org/abs/2503.06047) — Section 3.1 for StarCraft II environment details
