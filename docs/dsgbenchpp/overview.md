# DSGBench++ Overview

**DSGBench++** is a closed-loop infrastructure for building, evaluating, and training LLM-based strategic decision-making agents. It extends DSGBench (v1) from a static benchmark into a full agent development lifecycle system.

## Core Philosophy

The rise of LLM agents demands evaluation and training frameworks that go beyond static benchmarks. DSGBench++ is built around three principles:

1. **Closed-Loop Design** — Evaluation and training are not separate stages. Trajectories from evaluation become training data; trained agents generate richer trajectories.

2. **Strategic Depth** — Strategic games (StarCraft II, Civilization, Diplomacy, etc.) provide the ideal testbed for agentic decision-making, demanding long-horizon planning, opponent modeling, and real-time adaptation.

3. **Trajectory as the Core Asset** — Decision trajectories are first-class citizens. Every interaction is captured, structured, and made available for SFT and RLAF.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DSGBench++ Closed Loop                   │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │  Environment │──▶│    Agent     │──▶│  Trajectory    │  │
│  │ Construction │   │  Evaluation  │   │  Collection    │  │
│  └──────────────┘   └──────────────┘   └───────┬────────┘  │
│         ▲                                       │           │
│         │                                       ▼           │
│  ┌──────┴───────┐                  ┌────────────────────┐   │
│  │  RL Training │◀─────────────────│  Trajectory        │   │
│  │  & Iteration │                  │  Processing &      │   │
│  └──────────────┘                  │  Visualization     │   │
│                                    └────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Four Core Modules

| Module | Description | Status |
|--------|-------------|--------|
| **Environment Construction** | Six diverse strategic game environments with Gym interface | ✅ Available |
| **Agent Evaluation** | Fine-grained multi-dimensional assessment of strategic quality | ✅ Available |
| **Trajectory Collection & Processing** | Structured JSON capture, filtering, visualization | ✅ Available |
| **RL Training** | GRPO/PPO pipelines using trajectories and game feedback | 🔜 In Progress |

## Explore DSGBench++

- [Trajectory System](trajectory.md) — How trajectories are collected, structured, and used
- [RL Training](rl_training.md) — Reinforcement learning pipeline details
- [Quick Start](../getting_started/quickstart.md) — Get running in minutes
