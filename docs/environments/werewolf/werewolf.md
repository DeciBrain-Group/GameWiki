<div class="game-header">
  <div class="game-header-icon">🐺</div>
  <div class="game-header-info">
    <h1>Werewolf (Social Deduction)</h1>
    <div class="badge-row">
      <span class="badge badge-turnbased">Turn-based</span>
      <span class="badge badge-imperfect">Imperfect Info</span>
    </div>
    <p>Social deduction game testing deception, persuasion, and probabilistic reasoning about hidden roles in a multi-agent setting.</p>
  </div>
</div>

## Game Overview

**Werewolf** (also known as Mafia) is a social deduction party game where players are secretly assigned roles — **Werewolves** or **Villagers** — and must use deduction, persuasion, and deception to achieve their faction's objective. The game alternates between a **Day phase** (public discussion and voting) and a **Night phase** (werewolves act secretly).

DSGBench uses the [werewolf_arena](https://github.com/pjin257/werewolf_arena) framework, where LLM agents replace human players. All communication is conducted entirely in natural language, making this one of the most direct tests of an LLM's social reasoning ability.

### Why Werewolf for LLM Evaluation?

- **Pure language game** — the entire game is conducted through natural language; no mechanics beyond voting
- **Theory of mind** — agents must model other players' roles based on behavioral cues
- **Deception and trust** — werewolves must convincingly lie; villagers must detect lies
- **Coalition dynamics** — forming trust within a faction while eliminating the other
- **Uncertainty under imperfect information** — most players never know who the werewolves are with certainty

---

## Core Challenges for LLM Agents

| Challenge | Description |
|-----------|-------------|
| **Role Concealment** | Werewolves must behave plausibly as villagers during discussion |
| **Suspicion Analysis** | Villagers must analyze speech patterns to identify werewolves |
| **Vote Manipulation** | Directing group voting toward desired targets |
| **Consistency** | Maintaining a coherent in-character narrative across many turns |
| **Information Synthesis** | Integrating clues from multiple players' statements |
| **Bluff Detection** | Identifying inconsistencies in other players' stories |

---

## Roles

| Role | Faction | Ability | Win Condition |
|------|---------|---------|---------------|
| **Werewolf** | Werewolves | Kill one villager per night | Outnumber villagers |
| **Villager** | Villagers | Vote to eliminate players | Eliminate all werewolves |
| **Seer** | Villagers | Each night, learn one player's true role | Eliminate all werewolves |
| **Doctor** | Villagers | Each night, protect one player from elimination | Eliminate all werewolves |

---

## Game Flow

```
Night Phase
  ├── Werewolves: secretly vote on a target to eliminate
  ├── Seer: secretly learns the role of one chosen player
  └── Doctor: secretly protects one player

Day Phase
  ├── The night's result is announced (who was eliminated, if anyone)
  ├── All living players discuss and share suspicions (free-form text)
  └── All living players vote to eliminate one player

Repeat until:
  ├── All werewolves eliminated → Villagers win
  └── Werewolves ≥ Villagers → Werewolves win
```

---

## Observation Space

| Component | Description |
|-----------|-------------|
| **Own Role** | The agent's secret role assignment |
| **Player List** | Names and status (alive/eliminated) of all players |
| **Discussion History** | Full transcript of all public Day phase discussion |
| **Elimination History** | Record of who was eliminated each day/night |
| **Vote History** | Who voted for whom in each public vote |
| **Night Result** | Public announcement of the night's outcome |

!!! note "Asymmetric Information"
    Werewolves can see each other's identities (private channel). Villagers and special roles only have access to public information.

---

## Action Space

| Phase | Action Type | Example |
|-------|-------------|---------|
| **Night (Werewolf)** | Target selection | `kill: Player_3` |
| **Night (Seer)** | Investigation | `investigate: Player_5` |
| **Night (Doctor)** | Protection | `protect: Player_2` |
| **Day (All)** | Free-form discussion | `"I noticed Player_4 was very quiet yesterday..."` |
| **Day (All)** | Vote | `vote: Player_4` |

---

## Evaluation Scenes

| Scene | Configuration | Key Skill Tested |
|-------|--------------|-----------------|
| **Standard Game** | 6 players (2 werewolf, 1 seer, 1 doctor, 2 villager) | All-around social reasoning |
| **Werewolf Role** | LLM plays a werewolf, others are rule-based | Deception and concealment |
| **Villager Role** | LLM plays a villager/seer, others are rule-based | Detection and deduction |
| **All-LLM** | All players are LLMs (different models) | Inter-model social dynamics |

---

## Evaluation Metrics

| Metric | Description |
|--------|-------------|
| **Win Rate** | Percentage of games won (varies by assigned role) |
| **Survival Rate** | Percentage of games where the agent survives to the end |
| **Vote Accuracy** | For villagers: accuracy of votes against werewolves |
| **Detection Rate** | How often the agent correctly identifies werewolves |
| **Concealment Rate** | For werewolves: how long before being correctly identified |
| **Persuasion Success** | Rate of successfully redirecting group votes |

---

## Resources

- [Werewolf Arena GitHub](https://github.com/pjin257/werewolf_arena) — the underlying multi-agent framework
- [DSGBench Paper](https://arxiv.org/abs/2503.06047) — Section 3.5 for Werewolf environment details
