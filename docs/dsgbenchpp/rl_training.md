# RL Training

DSGBench++ provides a reinforcement learning pipeline that leverages collected gameplay trajectories and game environment reward signals to iteratively improve agent strategic capability.

!!! info "In Development"
    The RL training pipeline is currently under active development. This page will be updated as components become available.

## Design Goals

The RL training system in DSGBench++ is designed around three requirements:

1. **Game-native rewards** — Use actual game outcomes (win/loss, score delta, resource efficiency) as the primary reward signal, avoiding the need for human annotation
2. **Trajectory reuse** — Collected evaluation trajectories serve as the starting dataset for offline RL, minimizing the need for expensive online rollouts
3. **Iterative improvement** — Trained agents are re-deployed into the evaluation loop, generating higher-quality trajectories for the next training cycle

## Planned Architecture

```
Collected Trajectories
        │
        ▼
  Trajectory Filtering
  (quality thresholding)
        │
        ▼
  Reward Annotation
  (game metrics → scalar reward)
        │
     ┌──┴──────────────────┐
     │                     │
     ▼                     ▼
  SFT Warmup          GRPO/PPO Training
  (imitation)         (policy optimization)
     │                     │
     └──────────┬──────────┘
                │
                ▼
        Improved Agent
                │
                ▼
    Re-deployment → New Trajectories
```

## Roadmap

- [ ] **RL Training Pipeline** — End-to-end GRPO/PPO training loops using game environment rewards
- [ ] **Elo Rating System** — Dynamic competitive ranking for continuous agent evaluation
- [ ] **Hierarchical Evaluation** — Progressive difficulty evaluation framework

See [Trajectory System](trajectory.md) for details on the data that feeds into RL training.
