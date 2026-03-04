<div class="game-header">
  <div class="game-header-icon">🗺️</div>
  <div class="game-header-info">
    <h1>Stratego</h1>
    <div class="badge-row">
      <span class="badge badge-turnbased">Turn-based</span>
      <span class="badge badge-imperfect">Imperfect Info</span>
    </div>
    <p>Strategic board game with hidden piece ranks, testing bluffing, positional strategy, and opponent modeling under incomplete information.</p>
  </div>
</div>

## Game Overview

**Stratego** is a two-player turn-based strategy board game played on a 10×10 grid. Each player controls 40 pieces of varying ranks (Marshal, Colonel, Spy, Scout, Miner, etc.) plus Bombs and a Flag. The critical mechanic: **piece ranks are hidden from the opponent** — you only discover a piece's rank when it attacks or is attacked. The goal is to capture the opponent's Flag.

DSGBench uses the [Open Stratego](https://github.com/JBamberger/stratego) reinforcement learning environment, adapted with a text-based interface for LLM agents.

### Why Stratego for LLM Evaluation?

- **Hidden information with sequential revelation** — piece ranks are revealed progressively through combat, requiring memory and inference
- **Positional strategy** — piece placement at game start is a deep strategic decision
- **Deception through positioning** — high-value pieces can masquerade as expendable ones
- **Risk assessment** — every attack carries uncertainty (is that piece stronger than mine?)
- **Long-term opponent modeling** — tracking which pieces have been revealed and inferring hidden piece identities

---

## Core Challenges for LLM Agents

| Challenge | Description |
|-----------|-------------|
| **Setup Strategy** | Initial piece placement — protecting the Flag with Bombs, positioning Miners to defuse Bombs |
| **Rank Inference** | Deducing opponent piece ranks from movement patterns and combat history |
| **Risk Management** | Deciding whether to attack an unknown piece based on probability estimates |
| **Bluffing** | Moving high-rank pieces aggressively to appear as weak scouts |
| **Flag Defense** | Protecting the Flag while maintaining offensive pressure |
| **Miner Management** | Identifying and deploying Miners to defuse opponent Bombs |

---

## Game Mechanics

### Piece Ranks

| Rank | Name | Count | Special Ability |
|------|------|-------|----------------|
| 10 | Marshal | 1 | Defeats all pieces except Spy |
| 9 | General | 1 | — |
| 8 | Colonel | 2 | — |
| 7 | Major | 3 | — |
| 6 | Captain | 4 | — |
| 5 | Lieutenant | 4 | — |
| 4 | Sergeant | 4 | — |
| 3 | Miner | 5 | Can defuse Bombs |
| 2 | Scout | 8 | Can move any distance in a straight line |
| 1 | Spy | 1 | Defeats Marshal if attacking first |
| B | Bomb | 6 | Immovable; defeats any piece except Miner |
| F | Flag | 1 | Immovable; capture to win |

### Combat Resolution

When two pieces meet in combat:

- Higher rank wins (loser is removed from board)
- Equal ranks: both pieces are removed
- Special cases: Miner defeats Bomb; Spy defeats Marshal (when attacking)
- Surviving piece occupies the contested square; **both ranks are revealed to both players**

---

## Observation Space

| Component | Description |
|-----------|-------------|
| **Own Pieces** | Full rank and position of all own pieces |
| **Opponent Pieces (visible)** | Positions of all opponent pieces on the board |
| **Revealed Opponent Ranks** | Ranks of opponent pieces revealed through combat |
| **Unknown Opponent Pieces** | Positions of unrevealed opponent pieces (rank hidden) |
| **Combat History** | Record of all past combats and their outcomes |
| **Immobile Pieces** | Pieces that have never moved (likely Bombs or Flag) |

---

## Action Space

Each turn, the agent selects one piece and one valid destination:

| Action Type | Description |
|-------------|-------------|
| **Move** | Move a mobile piece to an adjacent empty square |
| **Scout Move** | Move a Scout piece any number of squares in a straight line |
| **Attack** | Move a piece into an opponent-occupied square (initiates combat) |

The agent cannot move Bombs or the Flag. Each turn, exactly one piece moves.

---

## Evaluation Scenes

| Scene | Configuration | Key Skill Tested |
|-------|--------------|-----------------|
| **Standard Match** | LLM vs. rule-based opponent | Full game strategy |
| **Endgame** | Pre-configured board state | Tactical decision-making under known information |
| **Opening Setup** | Evaluate quality of initial piece placement | Setup strategy |
| **Mirror Match** | LLM vs. same LLM model | Symmetric strategy analysis |

---

## Evaluation Metrics

| Metric | Description |
|--------|-------------|
| **Win Rate** | Percentage of matches won (Flag captured) |
| **Piece Efficiency** | Ratio of opponent pieces eliminated to own pieces lost |
| **Flag Protection** | Turns survived before opponent reaches the Flag area |
| **Rank Inference Accuracy** | Accuracy of inferred opponent piece ranks vs. actual |
| **Attack Success Rate** | Percentage of attacks that win combat |
| **Decision Validity** | Rate of legal, executable moves submitted |

---

## Resources

- [Open Stratego Environment](https://github.com/JBamberger/stratego) — the underlying RL environment
- [Stratego Official Rules](https://www.hasbro.com/common/instruct/Stratego.PDF)
- [DSGBench Paper](https://arxiv.org/abs/2503.06047) — Section 3.6 for Stratego environment details
