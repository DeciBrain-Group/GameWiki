# Starcraft II Environment

## Game Detailed Description

**Starcraft II** is a real-time strategy (RTS) game where players compete to gather resources, build bases, and control units to defeat opponents. The game requires players to make strategic decisions in real-time, balancing economy, army production, and combat. The game environment is dynamic and adversarial, making it a challenging scenario for LLM agents.

### Key Features:
- **Resource Management**: Players must gather minerals and vespene gas to fund their operations.
- **Base Building**: Construct and upgrade structures to enable technology advancements.
- **Army Control**: Train units and lead them in battles to destroy the opponent's base.
- **Fog of War**: Players only have partial visibility of the map, requiring exploration and anticipation.

---

## Observation Space

The **observation space** in Starcraft II consists of various state information available to the agent, such as:

1. **Map View**:
   - A 2D grid of the map, with features like terrain, structures, and units.
   - Data includes unit types, positions, health, and ownership (player or enemy).

2. **Resources**:
   - Current amounts of minerals and vespene gas.
   - Available workers and resource nodes.

3. **Game State**:
   - Information about the player's base, including active structures, upgrades, and army composition.
   - Limited visibility of enemy actions due to the fog of war.

### Data Format:
- **Dimensions**: A multi-channel 2D tensor representing the map state.
- **Types**:
  - Numerical data for resources (e.g., integers for minerals and gas).
  - Categorical data for unit types and states (e.g., `marine`, `zergling`).

---

## Action Space

The **action space** in Starcraft II is diverse, including both high-level and low-level commands:

1. **Unit Commands**:
   - **Move**: Direct a unit or group of units to a specific location.
   - **Attack**: Target an enemy unit or structure.
   - **Patrol**: Command units to patrol between points.

2. **Base Management**:
   - **Build**: Construct buildings like barracks, factories, or supply depots.
   - **Train**: Produce units (e.g., marines, tanks).
   - **Research**: Upgrade units or unlock new technologies.

3. **Macro Actions**:
   - **Expand**: Build additional bases to increase resource gathering.
   - **Defend**: Position units to defend against enemy attacks.
   - **Scout**: Explore the map for enemy positions or resource nodes.

### Action Representation:
- **Discrete**: Select from a predefined set of actions (e.g., `train_marine`, `build_barracks`).
- **Continuous**: Target specific positions on the map (e.g., `move_to(x, y)`).

---

## Eval Scenes

In Starcraft II, evaluation scenarios are designed to test different aspects of agent performance. Example evaluation scenes include:

1. **1v1 Matches**:
   - Agent competes against another AI-controlled player.
   - Tests strategic planning, resource management, and combat efficiency.

2. **Skirmish Mode**:
   - Focused on small-scale combat scenarios.
   - Evaluates unit control and micro-management capabilities.

3. **Macro Management**:
   - Emphasizes long-term planning, base expansion, and resource optimization.
   - Tests adaptability in prolonged games.

4. **Fog of War Scenarios**:
   - Agent operates under limited information.
   - Evaluates scouting, prediction, and counter-strategy formulation.

---

## Metrics

The following metrics are used to evaluate agent performance in Starcraft II:

1. **Win Rate**:
   - Percentage of matches won against different levels of AI opponents.

2. **Resource Efficiency**:
   - Measures how efficiently the agent gathers and spends resources.

3. **Action Accuracy**:
   - Evaluates how effectively the agent executes actions (e.g., precise unit control).

4. **Adaptability**:
   - Assesses how quickly the agent adjusts strategies based on enemy actions or game changes.

5. **Combat Effectiveness**:
   - Ratio of units lost to units destroyed.

6. **Exploration Efficiency**:
   - Evaluates how well the agent uncovers the map and anticipates enemy strategies.

---

## Prompt Example

Here is an example prompt for initializing an LLM agent in the Starcraft II environment:

```plaintext
You are a commander in a real-time strategy game. Your goal is to defeat the enemy by gathering resources, building structures, and controlling your army. The map is partially visible, and you must anticipate enemy actions. 

- Current resources: 500 minerals, 200 gas
- Current units: 10 marines, 5 tanks
- Enemy sighted: Small zergling group approaching your base

What will you do next? Provide a detailed action plan.
```

## Example Trajectory