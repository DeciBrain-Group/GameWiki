<div class="game-header">
  <div class="game-header-icon">🌍</div>
  <div class="game-header-info">
    <h1>Diplomacy (Welfare Diplomacy)</h1>
    <div class="badge-row">
      <span class="badge badge-turnbased">Turn-based</span>
      <span class="badge badge-perfect">Perfect Info</span>
    </div>
    <p>Multi-player negotiation and strategy game testing alliance formation, strategic deception, and multi-agent coordination across 7 competing powers.</p>
  </div>
</div>

## Game Overview

**Diplomacy** is a board game set in pre-World War I Europe, where 7 players each control a major power (England, France, Germany, Russia, Austria-Hungary, Italy, Turkey) and compete for territorial dominance. Unlike most strategy games, **Diplomacy has no dice and no randomness** — outcomes are entirely determined by coordinated unit movement and negotiation.

DSGBench uses the **welfare-diplomacy** variant, which adds a cooperative scoring dimension: agents are rewarded not just for domination but for maximizing collective welfare, creating richer mixed-motive dynamics.

### Why Diplomacy for LLM Evaluation?

- **Natural language negotiation** — the entire negotiation phase is conducted in free-form text, a natural fit for LLMs
- **Theory of mind** — agents must model other players' intentions, trust levels, and likely betrayals
- **Mixed-motive dynamics** — cooperate to survive, betray to win; navigating this tension is the core skill
- **Simultaneous orders** — all players submit orders secretly and simultaneously, requiring anticipation of others' moves
- **Long-horizon alliance management** — relationships must be maintained across many turns

---

## Core Challenges for LLM Agents

| Challenge | Description |
|-----------|-------------|
| **Negotiation Strategy** | Crafting persuasive messages while concealing true intentions |
| **Alliance Formation** | Identifying reliable partners and building mutual trust |
| **Betrayal Timing** | Knowing when to break an alliance for strategic advantage |
| **Order Coordination** | Synchronizing unit movements with allies for supported attacks |
| **Multi-party Reasoning** | Tracking relationships and commitments across all 6 opponents |
| **Bluff Detection** | Identifying when opponents are negotiating in bad faith |

---

## Game Structure

Diplomacy proceeds in alternating **Spring** and **Fall** turns, each with two phases:

```
Negotiation Phase
  └── Players exchange free-form messages (any number of rounds)

Order Phase
  └── Players simultaneously submit sealed orders for all units

Resolution Phase (automatic)
  └── All orders are adjudicated simultaneously
  └── Successful attacks capture territories; failed orders bounce

Winter Phase (Fall only)
  └── Players build or disband units based on supply center count
```

---

## Observation Space

| Component | Description |
|-----------|-------------|
| **Board State** | Territories controlled, unit positions for all powers |
| **Supply Centers** | Current supply center ownership and count per power |
| **Message History** | Full diplomatic message history (all communications visible after turn) |
| **Order History** | Past orders submitted by all players (revealed after resolution) |
| **Current Phase** | Spring/Fall, Negotiation/Order/Resolution |

---

## Action Space

=== "Negotiation Phase"

    The agent generates free-form natural language messages directed at specific players:

    ```
    To: France
    "I propose we attack Germany from both sides — I'll take Munich
    if you take Berlin. Agreed?"
    ```

=== "Order Phase"

    The agent submits structured movement orders for each unit:

    | Order Type | Example |
    |------------|---------|
    | **Move** | `A PAR - BUR` (Army in Paris moves to Burgundy) |
    | **Support** | `A MAR S A PAR - BUR` (Army in Marseille supports Paris→Burgundy) |
    | **Hold** | `A BUR H` (Army in Burgundy holds position) |
    | **Convoy** | `F ENG C A LON - BRE` (Fleet convoys Army from London to Brest) |

---

## Evaluation Scenes

| Scene | Players | Description |
|-------|---------|-------------|
| **Full 7-player** | 6 rule-based + 1 LLM | LLM agent vs. rule-based AI opponents |
| **LLM vs. LLM** | Multiple LLMs | Head-to-head multi-agent evaluation |
| **Coalition Test** | Custom | Test specific alliance/betrayal scenarios |

---

## Evaluation Metrics

| Metric | Description |
|--------|-------------|
| **Supply Center Count** | Number of supply centers controlled at game end |
| **Survival Rate** | Percentage of games where the agent survives to the end |
| **Solo Victory Rate** | Percentage of games where the agent achieves solo dominance (18+ centers) |
| **Welfare Score** | Collective welfare score (welfare-diplomacy variant) |
| **Alliance Stability** | Average duration of maintained alliances |
| **Negotiation Success Rate** | Rate of proposed coordinations that are honored by both parties |

---

## Setup & Dependencies

```bash
# Download baseline network parameters and place at:
./games/welfare_diplomacy/welfare_diplomacy_baselines/network_parameters/fppi2_params.npz
```

See [Quick Start](../../getting_started/quickstart.md) for full setup instructions.

---

## Resources

- [Welfare Diplomacy GitHub](https://github.com/mukobi/welfare-diplomacy) — the underlying environment
- [Original Diplomacy Research (FAIR)](https://ai.meta.com/research/cicero/) — Meta's Cicero agent
- [DSGBench Paper](https://arxiv.org/abs/2503.06047) — Section 3.3 for Diplomacy environment details
