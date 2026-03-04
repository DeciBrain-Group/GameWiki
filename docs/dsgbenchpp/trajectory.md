# Trajectory System

Trajectories are the backbone of DSGBench++. Every agent decision is captured as a structured JSON record, enabling supervised fine-tuning (SFT), reinforcement learning from agent feedback (RLAF), and behavioral analysis.

## What is a Trajectory?

A **decision trajectory** is a complete record of an agent's gameplay session, capturing:

- The game state at each decision point (observation)
- The agent's reasoning process (if chain-of-thought is enabled)
- The action taken
- The resulting game state change
- The reward/outcome signal

This creates a structured dataset that connects **state → reasoning → action → outcome** at every step of gameplay.

## Output Structure

```
output/
├── ability_score/
│   ├── model_ability_scores.csv
│   ├── game_ability_scores.csv
│   ├── calculation_steps.json
│   └── ability_detail_score/
│       ├── civ.csv
│       ├── starcraft2.csv
│       └── ...
└── [model_name]/
    └── [game_name]/
        └── [scene_name]/
            └── [opponent]/
                ├── *.log     # Full LLM input/output logs
                ├── *.json    # Structured decision trajectory
                └── *.csv     # Per-match results summary
```

## Trajectory Applications

| Application | Description | Status |
|-------------|-------------|--------|
| **Trajectory SFT** | Use high-quality trajectories to fine-tune LLM agents via supervised learning | 🔜 Planned |
| **RLAF** | Use game outcomes and fine-grained metrics as reward signals for RL training | 🔜 Planned |
| **Trajectory Visualization** | Interactive web UI for inspecting decision flows and strategy patterns | 🔜 Planned |
| **Behavioral Analysis** | Statistical analysis of agent decision patterns across games | ✅ Available |

## Roadmap

- [ ] **Trajectory Dataset Release** — Curated high-quality decision trajectories across all six games
- [ ] **Trajectory Visualization Dashboard** — Interactive web UI for exploring agent decision flows
- [ ] **SFT Pipeline** — End-to-end supervised fine-tuning workflow

See [DSGBench++ Overview](overview.md) for the full system roadmap.
