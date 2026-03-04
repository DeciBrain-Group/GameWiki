<div class="game-header">
  <div class="game-header-icon">🏛️</div>
  <div class="game-header-info">
    <h1>Civilization (FreeCiv)</h1>
    <div class="badge-row">
      <span class="badge badge-turnbased">Turn-based</span>
      <span class="badge badge-imperfect">Imperfect Info</span>
    </div>
    <p>4X strategy game testing long-horizon planning, resource allocation, and diplomatic decision-making across hundreds of turns.</p>
  </div>
</div>

## Game Overview

**Civilization** (implemented via [FreeCiv](https://www.freeciv.org/), an open-source Civilization clone) is a turn-based 4X strategy game — **eXplore, eXpand, eXploit, eXterminate**. Players build and grow a civilization from ancient times, managing cities, researching technologies, conducting diplomacy, and competing with AI-controlled civilizations for global dominance.

In DSGBench, the agent controls a civilization from a text-based interface, receiving structured game state observations and issuing high-level strategic commands each turn.

### Why Civilization for LLM Evaluation?

Civilization uniquely tests:

- **Ultra-long horizon planning** — games can span 200–500 turns, requiring consistent strategy
- **Multi-objective optimization** — balance economy, military, science, culture, and diplomacy simultaneously
- **Partial observability** — the fog of war hides enemy civilization positions and military strength
- **Delayed feedback** — strategic decisions may not show consequences for dozens of turns
- **Adaptive strategy** — pivot between victory conditions (domination, science, culture) as the game evolves

---

## Core Challenges for LLM Agents

| Challenge | Description |
|-----------|-------------|
| **Technology Tree Navigation** | Choosing which technologies to research given limited science output |
| **City Placement & Management** | Founding cities in optimal locations and managing their production queues |
| **Military Timing** | Knowing when to build armies, when to attack, and when to sue for peace |
| **Diplomatic Negotiation** | Managing relationships with multiple AI civilizations simultaneously |
| **Resource Allocation** | Balancing food, production, gold, science, and culture across cities |
| **Victory Path Selection** | Committing to and executing a specific victory condition strategy |

---

## Observation Space

Each turn, the agent receives a structured text observation containing:

| Component | Description |
|-----------|-------------|
| **Map State** | Partial map revealing explored terrain, resources, and units within sight range |
| **City Status** | Population, production queue, buildings, worked tiles for each city |
| **Civilization Stats** | Gold, science per turn, culture, current era, technology progress |
| **Military Units** | Unit types, positions, health, and movement points |
| **Diplomatic State** | Current relationships (alliance/peace/war) with other civilizations |
| **Available Actions** | List of valid actions this turn |

---

## Action Space

The agent issues high-level strategic commands each turn:

| Category | Example Actions |
|----------|----------------|
| **City Production** | `build_unit(city, "warrior")`, `build_building(city, "granary")` |
| **Technology** | `research(tech_name)` |
| **Unit Movement** | `move_unit(unit_id, direction)`, `attack(unit_id, target)` |
| **Diplomacy** | `declare_war(civ)`, `propose_peace(civ)`, `trade_offer(civ, terms)` |
| **City Founding** | `found_city(settler_id)` |
| **Turn End** | `end_turn()` |

---

## Evaluation Scenes

| Scene | Description | Key Skill Tested |
|-------|-------------|-----------------|
| **Early Game Expansion** | Start from Ancient Era, maximize territory by Turn 50 | Exploration, city founding strategy |
| **Technology Race** | Reach a target technology tier before opponent | Long-horizon planning, production prioritization |
| **Military Conflict** | Win a war against an AI-controlled opponent | Tactical unit management, production timing |
| **Full Game** | Complete a full game to any victory condition | All-around strategic capability |

---

## Evaluation Metrics

| Metric | Description |
|--------|-------------|
| **Score** | Final in-game civilization score at game end |
| **Victory Rate** | Percentage of games won (any victory condition) |
| **Turn Efficiency** | Number of turns to achieve specific milestones |
| **Tech Progress** | Technology era reached by a fixed turn count |
| **City Development** | Average city population and building count at mid-game |
| **Decision Validity** | Rate of syntactically valid and contextually legal actions |

---

## Setup & Dependencies

```bash
# Pull and start the FreeCiv web service
docker pull akon123456/freeciv-web
docker tag akon123456/freeciv-web:latest freeciv/freeciv-web:latest
conda activate dsgbench && start_freeciv_web_service
```

See [Quick Start](../../getting_started/quickstart.md) for full setup instructions.

---

## Resources

- [FreeCiv Official Site](https://www.freeciv.org/)
- [CivRealm GitHub](https://github.com/bigai-nlco/langsuite) — the underlying environment adapter
- [DSGBench Paper](https://arxiv.org/abs/2503.06047) — Section 3.2 for Civilization environment details
