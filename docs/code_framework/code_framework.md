# Code Framework

DSGBench is built on a modular codebase that separates game environments, agent management, evaluation logic, and configuration.

## Repository Structure

```
DSGBench++/
├── games/                  # Game environment implementations (OpenAI Gym interface)
│   ├── starcraft2/
│   ├── civ/
│   ├── streetfight3/
│   ├── welfare_diplomacy/
│   ├── werewolf/
│   └── stratego/
├── agent_manager/          # Agent, prompt, and LLM interfaces per game
├── agent_eval/             # Evaluation metrics and scoring logic
├── configs/                # Game scene and LLM configuration files
│   └── llm_configs/        # Per-LLM API configuration files
├── utils/                  # Common utilities and logging
├── assets/                 # Images and visual assets
├── multiprocess_eval_tasks.py  # Multi-process evaluation runner
├── calc_score.py           # Model ability score calculator
├── create_yaml.py          # Config generator for multiple LLMs
└── tasks_config.py         # Task and W&B configuration
```

## Key Components

| Component | Purpose |
|-----------|---------|
| `games/` | Each game environment exposes a standard OpenAI Gym interface |
| `agent_manager/` | Handles LLM API calls, prompt construction, and response parsing |
| `agent_eval/` | Computes per-game metrics from trajectory data |
| `configs/` | YAML configuration files for games, scenes, and LLM API endpoints |
| `multiprocess_eval_tasks.py` | Orchestrates parallel evaluation across multiple tasks and models |
| `calc_score.py` | Aggregates raw metrics into normalized ability scores |

## Adding a New Game

DSGBench is designed to be extensible. To add a new game environment:

1. Implement the OpenAI Gym interface in `games/your_game/`
2. Create an agent manager in `agent_manager/your_game/`
3. Define evaluation metrics in `agent_eval/your_game/`
4. Add configuration files in `configs/your_game/`

See the [Gym Interface](gym_interface/gym_interface.md) documentation for the required interface specification.

## Adding a New LLM

To evaluate a new LLM:

1. Create a config file in `configs/llm_configs/your_model.yaml`:
```yaml
model_name: your-model-name
api_key: your-api-key
api_url: https://your-api-endpoint
```

2. Add the model to your task configuration in `tasks_config.py`
3. Run `python create_yaml.py` to generate evaluation configs
