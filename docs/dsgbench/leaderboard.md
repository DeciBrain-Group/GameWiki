# Leaderboard

Performance of LLM-based agents across DSGBench game environments.

!!! info "Data Coming Soon"
    The full leaderboard with experimental results will be published alongside the DSGBench paper release. Check back soon or see the [paper](https://arxiv.org/abs/2503.06047) for preliminary results.

## Overall Ability Scores

Rankings are computed by the `dsgbench-ability-calc` package, which aggregates per-game scores into a single **model ability score** reflecting overall strategic decision-making capability.

| Rank | Model | Ability Score | SC2 | Civ | SF3 | Dip | Wlf | Stg |
|------|-------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| — | *Results to be published* | — | — | — | — | — | — | — |

## Score Methodology

Each game produces raw performance metrics (win rate, score, etc.) which are normalized to a 0–100 scale and weighted by game complexity. The final **Ability Score** is a weighted average across all six games.

See [Evaluation Metrics](../evaluation_and_capabilities/evaluation_and_capabilities.md) for the full scoring methodology.

## Reproduce Results

```bash
# After running evaluations:
pip install dsgbench-ability-calc==0.2.0
python calc_score.py

# Outputs saved to:
# output/ability_score/model_ability_scores.csv
# output/ability_score/game_ability_scores.csv
```
