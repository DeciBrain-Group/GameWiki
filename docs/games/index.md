# Games Overview

DSGBench evaluates LLM-based agents across **six high-dynamics strategic games**, spanning real-time and turn-based formats, and perfect and imperfect information scenarios.

## Environment Comparison

| Game | Mode | Info | Strategic | Multi-agent |
|------|------|------|-----------|-------------|
| [StarCraft II](../environments/starcraft2/starcraft2.md) | Real-time | Imperfect | ✅ | ❌ |
| [Civilization](../environments/civ/civ.md) | Turn-based | Imperfect | ✅ | ❌ |
| [Street Fighter III](../environments/street_fighter/street_fighter.md) | Real-time | Perfect | ❌ | ❌ |
| [Diplomacy](../environments/diplomacy/diplomacy.md) | Turn-based | Perfect | ✅ | ✅ |
| [Werewolf](../environments/werewolf/werewolf.md) | Turn-based | Imperfect | ✅ | ✅ |
| [Stratego](../environments/stratego/stratego.md) | Turn-based | Imperfect | ✅ | ❌ |

## Key Capability Dimensions Tested

<div class="system-overview">
  <div class="module-card">
    <h4>🧠 Strategic Planning</h4>
    <p>Long-horizon goal decomposition, technology trees, resource allocation (Civilization, StarCraft II)</p>
  </div>
  <div class="module-card">
    <h4>⚡ Real-time Decision</h4>
    <p>Rapid action selection under time pressure (StarCraft II, Street Fighter III)</p>
  </div>
  <div class="module-card">
    <h4>🤝 Multi-agent Interaction</h4>
    <p>Negotiation, alliance, deception, and theory of mind (Diplomacy, Werewolf)</p>
  </div>
  <div class="module-card">
    <h4>🎭 Imperfect Information</h4>
    <p>Reasoning and acting under uncertainty, opponent modeling (Stratego, Werewolf, StarCraft II)</p>
  </div>
</div>

Select a game from the navigation to see its detailed environment specification, observation/action space, and evaluation metrics.
