!!! warning "GameWiki Under Construction"
    Welcome to DSGBench GameWiki! This wiki is currently under active development. We are working on comprehensive game guides, strategies, and mechanics documentation. The complete wiki content will be available soon. Thank you for your interest and patience!

# Welcome to DSGBench
DSGBench is a novel strategic game benchmark designed to evaluate the performance of LLM-based agents in strategic planning, real-time decision making, adaptability, and multi-agent interactions. The benchmark encompasses six high-dynamics, complex strategy games, including StarCraft II, Civilization, Street Fighter III, and others. It provides fine-grained metrics for comprehensive evaluation, along with detailed decision trajectory analysis and trajectory datasets.


## What is DSGBench?
DSGBench focuses on testing the ability of LLM-based agents in the following aspects:

- **Decision-making**: Making optimal choices in real-time or turn-based game scenarios.
- **Strategic Planning**: Designing long-term strategies to achieve game objectives.
- **Adaptability**: Adjusting strategies dynamically in response to environmental changes.
- **Multi-agent Interaction**: Collaborating or competing with other agents in multi-agent games.

By supporting games like *Starcraft II*, *Diplomacy*, *Werewolf*, and more, DSGBench provides a comprehensive benchmark for evaluating LLM performance in decision-making and interaction-heavy tasks.


## Key Features
1. **Diverse Game Environments**:  
   DSGBench includes a variety of game environments, each offering unique challenges to LLM agents:
   - *Real-time strategy*: Starcraft II
   - *Turn-based strategy*: Civilization
   - *Social reasoning*: Werewolf
   - *Negotiation and diplomacy*: Diplomacy
   - *Strategic planning*: Stratego

2. **Standardized Evaluation Metrics**:  
   A set of well-defined metrics to evaluate agents across decision-making accuracy, adaptability, and strategic interaction.

3. **Extensible Framework**:  
   Built on a modular codebase, DSGBench can be extended to support new games or integrate with other AI frameworks like OpenAI Gym.

4. **Open Source**:  
   DSGBench is fully open-source, enabling researchers to contribute, customize, and extend its functionality.

---


## Documentation Overview

This documentation is divided into the following sections:

- [**Environments**](environments/environments.md):  
  Detailed descriptions of the supported games, including rules, challenges, and required agent capabilities.

- [**Evaluation and Capabilities**](evaluation_and_capabilities/evaluation_and_capabilities.md):  
  Overview of the evaluation metrics and the specific skills needed to succeed in DSGBench environments.

- [**Code Framework**](code_framework/overview.md):  
  Technical guide for setting up the environment, interacting with LLM agents, and running experiments.

- [**Experimental Setup**](experimental/experimental_setup/experimental_setup.md):  
  Guide to setting up and running experiments, including dataset preparation and result analysis.

---

## Quick Start

To get started with DSGBench, follow these steps:

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/your-repository/dsgbench.git
   cd dsgbench
   ```

2. **Install Dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

# Contributing

We welcome contributions to DSGBench! If you'd like to report an issue, suggest a feature, or submit a pull request, please visit our GitHub repository.

# License

DSGBench is open-sourced under the  [MIT License](https://opensource.org/licenses/MIT) - see the LICENSE file for details.

# Contact

For any questions or feedback, please contact us at [wenjietang2022@163.com](mailto:wenjietang2022@163.com).



# Citation

If you find DSGBench useful for your research, please consider citing our paper:

```bibtex
@misc{tang2025dsgbenchdiversestrategicgame,
      title={DSGBench: A Diverse Strategic Game Benchmark for Evaluating LLM-based Agents in Complex Decision-Making Environments}, 
      author={Wenjie Tang and Yuan Zhou and Erqiang Xu and Keyan Cheng and Minne Li and Liquan Xiao},
      year={2025},
      eprint={2503.06047},
      archivePrefix={arXiv},
      primaryClass={cs.AI},
      url={https://arxiv.org/abs/2503.06047}
}
```