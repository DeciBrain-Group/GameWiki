<div class="game-header">
  <div class="game-header-icon">🥊</div>
  <div class="game-header-info">
    <h1>Street Fighter III</h1>
    <div class="badge-row">
      <span class="badge badge-realtime">Real-time</span>
      <span class="badge badge-perfect">Perfect Info</span>
    </div>
    <p>Fast-paced 1v1 fighting game testing real-time reaction speed, tactical combo planning, and frame-level decision-making.</p>
  </div>
</div>

## Game Overview

**Street Fighter III: 3rd Strike** is a classic 2D fighting game where two players compete in real-time combat, depleting each other's health bars through attacks, combos, and defensive maneuvers. In DSGBench, the LLM agent controls a fighter against an AI opponent using the [DIAMBRA](https://diambra.ai/) engine, which provides a standardized reinforcement learning interface for arcade fighting games.

Unlike other DSGBench environments, Street Fighter III presents a fundamentally different challenge: **speed over depth**. There is no long-term planning — only immediate, frame-accurate tactical decisions.

### Why Street Fighter III for LLM Evaluation?

- **Real-time pressure** — decisions must be made rapidly, testing the agent's ability to act under tight time constraints
- **Combo execution** — sequences of inputs must be chained precisely
- **Opponent modeling** — recognizing and countering the opponent's attack patterns
- **Reaction vs. planning** — shifts the evaluation axis from strategic depth to tactical responsiveness
- **Contrast baseline** — provides a deliberate counterpoint to the turn-based, high-complexity games

---

## Core Challenges for LLM Agents

| Challenge | Description |
|-----------|-------------|
| **Reaction Time** | LLMs naturally have inference latency; acting fast enough is itself a challenge |
| **Combo Sequences** | Multi-step attack chains require precise input ordering |
| **Defensive Timing** | Blocking, parrying, and dodging require predicting opponent moves |
| **Move Prioritization** | Choosing the highest-value action from a large discrete action set |
| **Health Management** | Trading damage efficiently while avoiding knockouts |

---

## Observation Space

Each frame, the agent receives:

| Component | Description |
|-----------|-------------|
| **Player State** | Character health, position (x, y), current animation state |
| **Opponent State** | Opponent health, position, animation state |
| **Game State** | Round number, timer, round score |
| **Action History** | Recent actions taken by both players |

Both players share the same screen, making this a **perfect information** game — the agent can observe all relevant state without hidden information.

---

## Action Space

The action space is a discrete set of game controller inputs:

| Category | Actions |
|----------|---------|
| **Movement** | Left, Right, Jump, Crouch, Back |
| **Attacks** | Light/Medium/Heavy Punch, Light/Medium/Heavy Kick |
| **Special Moves** | Character-specific special move inputs |
| **Defense** | Block (hold back), Parry (toward + attack timing) |
| **No-op** | Do nothing (stand idle) |

Actions are issued at the game's frame rate. The DIAMBRA wrapper handles the low-level input simulation.

---

## Evaluation Scenes

| Scene | Description | Key Skill Tested |
|-------|-------------|-----------------|
| **Standard Match** | Best-of-3 rounds against AI opponent | Overall combat performance |
| **Survival Mode** | Fight consecutive opponents with diminishing health | Endurance and resource management |
| **Specific Character** | Agent uses a designated character against a fixed opponent | Character mastery |

---

## Evaluation Metrics

| Metric | Description |
|--------|-------------|
| **Win Rate** | Percentage of matches (best-of-3 rounds) won |
| **Round Win Rate** | Percentage of individual rounds won |
| **Health Remaining** | Average health % remaining at round end (winner's perspective) |
| **Damage Dealt** | Total damage inflicted per match |
| **Action Efficiency** | Ratio of effective actions (landed hits) to total actions |
| **Combo Rate** | Frequency of successful multi-hit combos executed |

---

## Setup & Dependencies

```bash
# Pull the DIAMBRA engine
docker pull diambra/engine:v2.2

# Place ROM file
cp sfiii3n.zip ./games/streetfight3/roms/

# Run evaluation with DIAMBRA wrapper
diambra run --images.no-pull -r ./games/streetfight3/roms python multiprocess_eval_tasks.py
```

!!! warning "ROM Requirement"
    Street Fighter III requires a legal copy of the game ROM (`sfiii3n.zip`). DIAMBRA does not distribute ROM files.

See [Quick Start](../../getting_started/quickstart.md) for full setup instructions.

---

## Resources

- [DIAMBRA Documentation](https://docs.diambra.ai/)
- [llm-colosseum](https://github.com/OpenGenerativeAI/llm-colosseum) — the underlying LLM-fighting game adapter
- [DSGBench Paper](https://arxiv.org/abs/2503.06047) — Section 3.4 for Street Fighter III environment details
