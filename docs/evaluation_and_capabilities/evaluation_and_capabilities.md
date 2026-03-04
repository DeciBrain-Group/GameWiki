# Evaluation Metrics

DSGBench provides fine-grained, multi-dimensional metrics for evaluating LLM agent performance across all six game environments.

## Evaluation Dimensions

| Dimension | Description |
|-----------|-------------|
| **Strategic Planning** | Ability to form and execute long-horizon plans |
| **Real-time Decision Making** | Speed and accuracy of tactical decisions under time pressure |
| **Adaptability** | Dynamic strategy adjustment in response to environmental changes |
| **Multi-agent Interaction** | Cooperation, competition, negotiation, and social reasoning |

## Per-Game Metrics

Each game environment has a set of game-specific metrics. See individual game pages for details:

- [StarCraft II Metrics](../environments/starcraft2/starcraft2.md#evaluation-metrics)
- [Civilization Metrics](../environments/civ/civ.md#evaluation-metrics)
- [Street Fighter III Metrics](../environments/street_fighter/street_fighter.md#evaluation-metrics)
- [Diplomacy Metrics](../environments/diplomacy/diplomacy.md#evaluation-metrics)
- [Werewolf Metrics](../environments/werewolf/werewolf.md#evaluation-metrics)
- [Stratego Metrics](../environments/stratego/stratego.md#evaluation-metrics)

## Ability Score Calculation

The DSGBench ability score aggregates per-game metrics into a single number using the `dsgbench-ability-calc` package:

```bash
pip install dsgbench-ability-calc==0.2.0
python calc_score.py
```

Output files:
- `output/ability_score/model_ability_scores.csv` — overall model scores
- `output/ability_score/game_ability_scores.csv` — scores per game per model
- `output/ability_score/calculation_steps.json` — detailed calculation breakdown

See the [Leaderboard](../dsgbench/leaderboard.md) for comparative results across evaluated LLMs.
