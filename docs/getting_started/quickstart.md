# Quick Start

Get DSGBench++ running in minutes. This guide walks you through installation, game setup, and running your first evaluation.

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| **OS** | Windows 11 + WSL | Required for StarCraft II compatibility |
| **Python** | 3.9.6 | Use conda for version management |
| **Docker Desktop** | Latest | Required for Civilization and Street Fighter III |
| **Git** | Any | For cloning the repository |

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/DeciBrain-Group/DSGBench
cd DSGBench
```

### 2. Create Python Environment

```bash
conda create --name dsgbench python=3.9.6
conda activate dsgbench
```

### 3. Install PromptCoder

```bash
git clone https://github.com/dhh1995/PromptCoder
cd PromptCoder && pip install -e . && cd ..
```

### 4. Install Dependencies

```bash
pip install -r pip_requirements.txt

# Fix websocket compatibility issue
pip uninstall websocket websocket-client -y
pip install websocket==0.2.1 websocket-client==1.8.0
```

---

## Game Setup

Each game requires separate setup. You only need to configure the games you plan to evaluate.

=== "StarCraft II"

    1. Download StarCraft II from [Blizzard](https://www.blizzard.com/zh-tw/)
    2. Place the map file in the correct location:
    ```
    StarCraft II/Maps/Ancient Cistern LE.SC2Map
    ```
    3. Verify the `TextStarCraft2` environment is accessible from your `games/starcraft2/` directory.

=== "Civilization"

    Runs via Docker using the FreeCiv-web server:

    ```bash
    docker pull akon123456/freeciv-web
    docker tag akon123456/freeciv-web:latest freeciv/freeciv-web:latest

    # Start the FreeCiv web service
    conda activate dsgbench && start_freeciv_web_service
    ```

=== "Street Fighter III"

    Runs via the DIAMBRA engine:

    ```bash
    docker pull diambra/engine:v2.2
    ```

    Then place the ROM file:
    ```
    ./games/streetfight3/roms/sfiii3n.zip
    ```

    !!! warning "ROM file"
        You must provide a legal copy of the Street Fighter III ROM file (`sfiii3n.zip`).

=== "Diplomacy"

    Download the pre-trained baseline network parameters:

    1. Download `fppi2_params.npz`
    2. Place it at:
    ```
    ./games/welfare_diplomacy/welfare_diplomacy_baselines/network_parameters/fppi2_params.npz
    ```

=== "Werewolf / Stratego"

    No additional setup required. These games run purely in Python.

---

## Configuration

### LLM API Keys

Edit or create a config file under `configs/llm_configs/`:

```yaml
# configs/llm_configs/gpt4o.yaml
model_name: gpt-4o
api_key: sk-your-api-key-here
api_url: https://api.openai.com
```

Supported model providers: OpenAI, Anthropic, DeepSeek, Qwen, and any OpenAI-compatible API endpoint.

### Weights & Biases (Optional)

To log evaluation results to W&B, set your API key in `tasks_config.py`:

```python
WANDB_API_KEY = "your_wandb_key"
```

### Task Configuration

Configure which games and LLMs to evaluate in `tasks_config.py`:

```python
TASKS = [
    {"game": "starcraft2", "llm": "gpt-4o", "scenes": ["1v1_basic"]},
    {"game": "werewolf",   "llm": "gpt-4o", "scenes": ["6player"]},
    # Add more tasks here
]
```

---

## Running an Evaluation

### Standard Run

```bash
# Run all configured tasks
python multiprocess_eval_tasks.py
```

### Street Fighter III (requires DIAMBRA wrapper)

```bash
diambra run --images.no-pull -r [path/to/roms] python multiprocess_eval_tasks.py
```

### Calculate Model Ability Scores

After evaluation completes, compute the aggregated ability scores:

```bash
pip install dsgbench-ability-calc==0.2.0
python calc_score.py
```

---

## Understanding the Output

Results are saved to the `output/` directory with the following structure:

```
output/
├── ability_score/
│   ├── model_ability_scores.csv      # Aggregated scores per model
│   ├── game_ability_scores.csv       # Scores broken down by game
│   ├── calculation_steps.json        # Score calculation details
│   └── ability_detail_score/
│       ├── civ.csv
│       ├── starcraft2.csv
│       └── ...
└── [model_name]/
    └── [game_name]/
        └── [scene_name]/
            └── [opponent]/
                ├── *.log             # Full LLM input/output logs
                ├── *.json            # Structured decision trajectory
                └── *.csv             # Per-match results summary
```

!!! tip "Trajectory Files"
    The `.json` trajectory files are first-class assets in DSGBench++. They capture every agent decision in a structured format and can be used directly for SFT training or RL reward computation. See [Trajectory System](../dsgbenchpp/trajectory.md) for details.

---

## Next Steps

<div class="system-overview">
  <div class="module-card">
    <h4>🎮 Explore Games</h4>
    <p>Learn about each game environment, its challenges, and observation/action spaces</p>
  </div>
  <div class="module-card">
    <h4>📊 View Leaderboard</h4>
    <p>See how different LLMs perform across all six game environments</p>
  </div>
  <div class="module-card">
    <h4>🗂️ Trajectory System</h4>
    <p>Understand how to collect and use decision trajectories for training</p>
  </div>
  <div class="module-card">
    <h4>💻 API Reference</h4>
    <p>Explore the code framework and Gym interface for extending DSGBench</p>
  </div>
</div>
