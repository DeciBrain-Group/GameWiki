<div class="game-header">
  <div class="game-header-icon">⚔️</div>
  <div class="game-header-info">
    <h1>StarCraft II</h1>
    <div class="badge-row">
      <span class="badge badge-realtime">Real-time</span>
      <span class="badge badge-imperfect">Imperfect Info</span>
    </div>
    <p>Real-time strategy game demanding resource management, base building, and army control under fog of war — the gold standard for complex strategic AI evaluation.</p>
  </div>
</div>

## Game Overview

**StarCraft II** (SC2) is a real-time strategy (RTS) game developed by Blizzard Entertainment where players compete to gather resources, build bases, and control armies to defeat opponents. Players must make strategic decisions continuously in real-time, balancing economy, production, and combat — all while operating under the fog of war that limits map visibility.

DSGBench uses the **TextStarCraft2** environment ([GitHub](https://github.com/histmeisah/Large-Language-Models-play-StarCraftII)), which wraps StarCraft II with a text-based interface so LLM agents can receive structured observations and issue natural language commands. The underlying interaction protocol is built on top of **PySC2** — DeepMind's Python StarCraft II Learning Environment released in collaboration with Blizzard in 2017.

### Why StarCraft II for LLM Evaluation?

- **Real-time decision pressure** — actions must be issued continuously without turn-based pauses
- **Fog of war** — agents must scout, infer, and anticipate under imperfect information
- **Multi-objective management** — simultaneously balance economy, production, and combat
- **Long-horizon planning** — build orders and tech trees require planning dozens of steps ahead
- **Opponent adaptation** — strategies must shift in response to enemy composition and aggression
- **Enormous action space** — approximately 300+ discrete actions per step, each with multiple argument types
- **Strategic interpretability** — unlike AlphaStar-style RL, LLM agents can express their reasoning in natural language

---

## Part I: StarCraft II Game Mechanics

This section covers the foundational game mechanics that AI researchers and beginners need to understand in order to develop or evaluate agents in SC2.

---

### 1.1 The Three Races

StarCraft II features three playable races, each with a completely distinct set of units, buildings, and macro mechanics. No race is inherently stronger — each has been carefully designed for balance.

| Feature | Terran | Zerg | Protoss |
|---|---|---|---|
| **Unit Strength** | Medium (versatile) | Individually weak | Strong and expensive |
| **Unit Numbers** | Medium | Very high (mass production) | Low (elite) |
| **Healing** | Medivacs + SCV repair | Slow natural regeneration | Shields recharge; no HP heal |
| **Mobility** | Medium; buildings can lift off | Very high (fast units, burrow) | Medium-high (Warp-in anywhere near Pylons) |
| **Key Macro Mechanic** | MULE calldown, Orbital Command scan | Larva injection (Queen), Creep spread | Chrono Boost (Nexus), Warp Gate |
| **Supply Structure** | Supply Depot (static, can lower) | Overlord (mobile, can be killed) | Pylon (powers buildings too) |
| **Production Logic** | Dedicated buildings (Barracks, Factory, Starport) | All units from Hatchery/Lair/Hive | Gateway / Warp Gate anywhere in Psionic Matrix |
| **Playstyle** | Defensive/Adaptive; wall-offs, bio drops | Aggressive/Reactive; zerg-ling runbys, mass expansion | Positioning/Timing; death-ball, force-field choke |
| **Macro Demand** | Very High (constant MULE + inject-equivalent management) | High (Queen inject cycle every ~29s) | Moderate (Chrono Boost on Nexus) |
| **Difficulty for Beginners** | High (many simultaneous mechanics) | Medium-High (reactive, strong economy) | Medium (forgiving early, punishing late) |

#### Terran
Terran is the human faction. Their greatest strengths are adaptability and healing: SCVs can repair mechanical units and buildings mid-battle, and Medivacs heal bio units. Nearly all Terran buildings can **lift off** and fly to a new location, enabling unique strategies such as floating a Command Center to an expansion after building it in the main base. A key early defensive technique is the **wall-off**: placing a Supply Depot and Barracks at the ramp entrance to block early Zergling rushes.

**Race-specific macro mechanic:** The **Orbital Command** (an upgrade to the Command Center costing 150 minerals) provides three abilities using Energy:
- **MULE** (50 energy): A temporary unit that mines 225 minerals in ~64 seconds (~3.45x an SCV's rate), then dies.
- **Scanner Sweep** (50 energy): Reveals a 13-radius area anywhere on the map for 9 seconds, detecting cloaked/burrowed units.
- **Extra Supplies** (50 energy): Increases a Supply Depot's supply yield by 8.

Forgetting to call down MULEs every ~50 seconds is a significant macro error.

#### Zerg
Zerg is the insectoid swarm faction. Their defining characteristic is the **Larva system**: each Hatchery spawns 3 Larva over time, and the Queen can inject additional Larva (up to 19 per Hatchery). All Zerg combat units are produced from Larva, allowing burst production of multiple unit types simultaneously. This makes Zerg the fastest race at replenishing losses.

Zerg buildings must be placed on **Creep** (a biological substance spread from Hatcheries and Creep Tumors), except for the Hatchery itself. Creep gives all Zerg units a movement speed bonus when on it.

Zerg supply comes from **Overlords** — flying units that can be killed, making Zerg supply uniquely vulnerable to air units (e.g., Vikings). Overlords can morph into Overseers (mobile detectors) or Overlord Transports.

Zerg units naturally regenerate HP when not taking damage. The Hatchery evolves to a **Lair** (enabling mid-tier tech) and then to a **Hive** (unlocking late-game units).

#### Protoss
Protoss is the advanced alien civilization. Their units are individually the most powerful and expensive. All Protoss units have **Shields** — a secondary HP bar that regenerates over time when not taking damage. Losing shields reduces combat effectiveness significantly, but retreating allows recovery.

The **Pylon** provides both supply (8 per Pylon) and a **Psionic Matrix** power field. All Protoss buildings except the Nexus, Pylon, and Assimilator must be placed within a Pylon's power field and become inactive if the powering Pylon is destroyed.

The **Warp Gate** is a transformation of the Gateway that allows units to be warped directly to any location within a Psionic Matrix (including a Warp Prism's field), enabling rapid offensive reinforcement anywhere on the map.

**Race-specific macro mechanic:** **Chrono Boost** (50 energy from Nexus) accelerates any targeted building's production/research by 50% for 20 seconds, effectively completing tasks in 2/3 the normal time.

---

### 1.2 Resource System

StarCraft II has two harvestable resources: **Minerals** and **Vespene Gas**. Both are gathered by worker units and returned to the player's Town Hall structure.

#### Minerals

| Property | Value |
|---|---|
| Minerals per field (large node) | 1,800 |
| Minerals per field (small node) | 900 |
| Fields per standard base | 8 (4 large + 4 small) |
| Total minerals per base | ~10,800 |
| Per trip (standard) | 5 minerals |
| Per trip (gold/rich patch) | 7 minerals |
| Optimal workers per field | 2 (diminishing returns after 2; 4th worker useless) |
| Rate per worker per minute | ~55–60 (standard); ~80–85 (gold) |
| Max income with 16 workers on 8 patches | ~1,206 minerals/min |

Workers gather minerals by traveling to the field, spending ~1.99 seconds mining, then returning to the Town Hall. The game supports **"smart"** mineral walking where workers automatically return and repeat.

#### Vespene Gas

| Property | Value |
|---|---|
| Gas per geyser | 2,250 |
| Geysers per standard base | 2 |
| Total gas per base | 4,500 |
| Extraction structure | Refinery (Terran) / Assimilator (Protoss) / Extractor (Zerg) |
| Per trip | 4 gas |
| Time inside geyser per trip | ~1.415 seconds |
| Optimal workers per geyser | 3 (provides ~163 gas/min) |
| Max rate at 3 workers | ~169 gas/min per geyser |
| Depletion time at saturation | ~13.25 minutes |

To extract gas, a worker must first construct a gas extraction structure on top of the geyser (costing 75 minerals). Then workers can be assigned to mine from it.

**Key ratio:** A standard base contains resources in a ~2.4:1 mineral-to-gas ratio (10,800 : 4,500). Most mid-game armies require substantial gas, making timely gas saturation critical.

#### Worker Units

| Race | Worker | Build Method | Special Property |
|---|---|---|---|
| Terran | SCV | Trained from Command Center | Must stay at building site to complete construction; can repair mechanical units |
| Protoss | Probe | Trained from Nexus | Places a warp beacon and can immediately move away; building warps in autonomously |
| Zerg | Drone | Produced from Larva | Sacrificed (morphs into) the building — every structure costs a worker |

All workers cost **50 minerals** and take 17 game seconds to produce. Keeping worker production continuous in the early game (never missing a worker) is one of the highest-impact macro habits.

---

### 1.3 Supply System

Every unit trained consumes **supply** (called **Psi** for Protoss). The hard cap is **200 supply**. Players cannot train new units when at the supply cap — this state is called being **"supply blocked"** and is a critical macro error.

| Race | Supply Structure | Supply Provided | Special Notes |
|---|---|---|---|
| Terran | Supply Depot | 8 | Can lower into ground to let ground units pass |
| Terran | Command Center / Orbital / PF | 15 | Base always provides some supply |
| Protoss | Pylon | 8 | Also powers all non-Nexus buildings nearby |
| Protoss | Nexus | 15 | |
| Zerg | Overlord | 8 | Mobile, can be killed; upgrades to Overseer or Transport |
| Zerg | Hatchery / Lair / Hive | 6 | |

**Supply costs of key units (examples):**

| Unit | Race | Supply Cost |
|---|---|---|
| Marine | Terran | 1 |
| Medivac | Terran | 2 |
| Siege Tank | Terran | 3 |
| Battlecruiser | Terran | 6 |
| Zergling | Zerg | 0.5 (2 per 1 supply, produced in pairs) |
| Roach | Zerg | 2 |
| Ultralisk | Zerg | 6 |
| Zealot | Protoss | 2 |
| Colossus | Protoss | 6 |
| Carrier | Protoss | 6 |

A 200-supply army composed entirely of the cheapest units would be ~200 Marines; a full Carrier fleet might only be ~33 units.

---

### 1.4 Tech Tree Structure

Each race has a hierarchical building dependency chain. Units and upgrades become available only after the required prerequisite buildings are constructed.

#### Terran Tech Tree

```
Command Center
├── Supply Depot
├── Refinery
└── Barracks  ─────────────────────────────────────────────┐
    ├── [+Tech Lab] → Marauder, Ghost, upgrades             │
    ├── [+Reactor] → double production                      │
    ├── Engineering Bay → Missile Turret, Sensor Tower      │
    ├── Bunker                                              │
    └── Factory (requires Barracks)                         │
        ├── [+Tech Lab] → Siege Tank, Thor, upgrades        │
        ├── [+Reactor] → double Hellion/Widow Mine          │
        ├── Armory → level-2/3 upgrades                     │
        └── Starport (requires Factory)                     │
            ├── [+Tech Lab] → Banshee, Raven, Battlecruiser │
            └── [+Reactor] → double Medivac/Viking          │
```

Key unlock gates:
- **Barracks** is required before **Engineering Bay** (tier-1.5 defensive upgrades)
- **Factory** is required before **Starport** (air production)
- **Armory** unlocks level-2 and level-3 mech upgrades
- The **Tech Lab** add-on unlocks advanced units from any production building it is attached to; the **Reactor** doubles production speed for basic units

#### Zerg Tech Tree

```
Hatchery (produces all units via Larva)
├── Spawning Pool → Zerglings, Queen
│   ├── Baneling Nest → Banelings
│   ├── Roach Warren → Roaches
│   │   └── Ravager (morph from Roach)
│   └── Lair (upgrade from Hatchery, requires Spawning Pool)
│       ├── Hydralisk Den → Hydralisks
│       │   └── Lurker Den → Lurkers
│       ├── Spire → Mutalisks, Corruptors
│       │   └── Greater Spire (upgrade, requires Hive)
│       │       → Brood Lords, Devourers
│       ├── Infestation Pit → Infestors
│       └── Nydus Network → Nydus Worms (instant cross-map transport)
│           └── Hive (upgrade from Lair, requires Infestation Pit)
│               ├── Ultralisk Cavern → Ultralisks
│               └── [unlocks level-3 upgrades at Evolution Chamber & Spire]
├── Evolution Chamber → weapon/armor upgrades (levels 1-3)
├── Spine Crawler (static ground defense, requires Spawning Pool)
└── Spore Crawler (static anti-air + detection)
```

Key unlock gates:
- A **Spawning Pool** is needed before Zerg can build any unit other than Drones/Queens
- **Lair** (upgrade from Hatchery) unlocks all mid-tier structures
- **Hive** (upgrade from Lair, requires Infestation Pit) unlocks late-game tech
- All buildings except Hatchery must be placed on **Creep**

#### Protoss Tech Tree

```
Nexus (trains Probes)
├── Pylon (required before any other building; provides power field)
├── Assimilator (gas)
└── Gateway / Warp Gate (converts to Warp Gate after Cybernetics Core)
    ├── Zealot, Stalker, Sentry, Adept, High Templar, Dark Templar
    ├── Cybernetics Core (required before Stalker; unlocks Warp Gate research)
    │   ├── Twilight Council → Blink (Stalker), Charge (Zealot), upgrades
    │   ├── Templar Archives (requires Twilight Council) → High Templar, Psionic Storm
    │   ├── Dark Shrine (requires Twilight Council) → Dark Templar
    │   ├── Stargate → Phoenix, Oracle, Void Ray
    │   │   └── Fleet Beacon (requires Stargate) → Carrier, Mothership upgrades
    │   └── Robotics Facility → Immortal, Warp Prism, Observer
    │       └── Robotics Bay (requires Robotics Facility) → Colossus, Disruptor
    └── Forge → weapon/armor/shield upgrades, Photon Cannon
```

Key unlock gates:
- **Cybernetics Core** is the pivotal Protoss building — it unlocks the majority of the tech tree and the Warp Gate transformation
- **Twilight Council** is needed for late-game Gateway units (High Templar, Dark Templar)
- Protoss production buildings must be placed within a **Pylon's power field** or they become inactive

---

### 1.5 Unit Classification System

StarCraft II units are classified along several dimensions, which determine bonus damage from certain weapons and eligibility for abilities/spells.

**Primary movement type:**
- **Ground** — moves on the map surface; cannot pass over terrain cliffs without ramps
- **Air** — flies and can pass over all terrain; targeted only by anti-air weapons

**Attack range:**
- **Melee** — must be adjacent to target (e.g., Zealot, Zergling, Ultralisk)
- **Ranged** — attacks from a distance (e.g., Marine, Stalker, Hydralisk)

**Armor attributes (each unit has exactly one from each pair):**
- **Light** vs. **Armored** — affects bonus damage from specific weapons (e.g., Marauder does +10 vs. Armored)
- **Biological** — healable by Medivacs, Transfusion; all Zerg units and buildings are Biological
- **Mechanical** — repairable by SCVs; Terran vehicles and ships; affected by Interference Matrix
- **Psionic** — subset of Protoss units; affected by certain spells
- **Massive** — immune to Graviton Beam, Vortex, Force Field movement effects
- **Structure** — all buildings; always Armored; Terran buildings also Mechanical, Zerg buildings also Biological

Notable exceptions: The **SCV** is both Biological and Mechanical (can be healed by Medivacs AND repaired by other SCVs). The **Archon** is neither Biological nor Mechanical.

**Unit roles (informal classification):**
- **Worker** — resource gathering and construction (SCV, Probe, Drone)
- **Combat** — primary fighting units
  - **Standard** — backbone units (Marine, Zergling, Zealot)
  - **Support/Caster** — spell-casting units with limited auto-attacks (Ghost, Infestor, High Templar)
  - **Siege** — high-damage, often immobile or slow (Siege Tank, Lurker, Colossus)
  - **Harassment** — fast, hit-and-run specialists (Hellion, Mutalisk, Oracle)
  - **Transport** — carry other units (Medivac, Overlord Transport, Warp Prism)
  - **Detector** — reveals cloaked/burrowed units (Raven, Overseer, Observer)

---

### 1.6 Vision and Fog of War

StarCraft II uses a **dynamic fog of war** system where map areas that are not within the line-of-sight of your units or buildings are obscured:

- **Black (unexplored)** — terrain not yet visited; completely invisible
- **Explored but shrouded (dark gray)** — previously visited but no current unit vision; shows static terrain and structures as last seen; enemy unit data is removed
- **Fully lit (bright)** — currently within a friendly unit or building's vision radius; all units and structures fully visible

**Key vision rules:**
- Ground units **cannot see up cliffs** (high-ground advantage: attackers on high ground can see defenders below, but not vice versa without detection)
- Units on **high ground** gain a defensive bonus — attackers below have a 50% miss chance against units on elevated terrain when there is no vision of the high ground provided by another unit
- **Burrowed Zerg units** have reduced vision range (typically -4), except Infestor, Lurker, and Baneling

**Vision ranges (examples):**

| Unit / Structure | Vision Range |
|---|---|
| Most buildings | 9 |
| Marine | 9 |
| Observer (Protoss detector) | 11 |
| Overlord (Zerg supply) | 11 |
| Xel'Naga Tower (neutral) | 22 |
| Siege Tank (Siege Mode) | 13 (attack range 13, longer than sight of most units) |

**Detection mechanics:** Cloaked and burrowed units are invisible to standard sight. Revealing them requires a **detector** unit or structure within detection range. Detectors per race:
- **Terran**: Missile Turret (static), Raven (mobile), Scanner Sweep (Orbital Command ability, anywhere on map)
- **Protoss**: Photon Cannon (static), Observer (mobile, invisible itself)
- **Zerg**: Spore Crawler (static), Overseer (mobile)

The **Scanner Sweep** is unique: for 50 Orbital Command energy it reveals a radius-13 circle anywhere on the map for 9 seconds, detecting all hidden units. Because this costs the same energy as a MULE, each scan represents foregone mining income.

---

## Part II: Game Phase Structure

A standard StarCraft II game is divided into distinct phases. Professional players reach each phase faster than beginners, but the structure applies across all skill levels.

### Phase Overview

```
0 min ──────────── 5 min ──────────── 10 min ──────────── 20 min ──────────── 30+ min
   [Opening]           [Early Game]        [Mid Game]            [Late Game]
   Build Order         First Army          Expansion &           Max Army
   Execution           & Scout             Tech Choices          Death-balls
```

### Phase 1: Opening / Build Order Phase (0 – ~5 min)

**Core objectives:** Execute your chosen build order flawlessly; send initial scout at the correct timing; do not supply block.

In the first 3–4 minutes, competitive play is almost entirely scripted. Professional players drill their openers until execution is automatic. This phase defines which strategic options are available for the rest of the game.

Typical milestones by timestamp:
- **~1:00** — First supply structure placed (Supply Depot / Pylon / Overlord queued)
- **~1:30–2:00** — First production building started (Barracks / Gateway / Spawning Pool)
- **~2:00–2:30** — Worker sent to scout enemy base
- **~3:00–4:00** — First gas extraction structure placed
- **~4:00–5:00** — Natural expansion considered or committed to

Key scouting indicators in the opening:
- **Zerg Spawning Pool before 2:00** → likely Zergling rush (6- or 7-Pool)
- **Two Barracks before Refinery** (Terran) → early Marine pressure
- **Two Assimilators before 3:00** (Protoss) → gas-heavy tech rush (Dark Templar / Void Ray)
- **No expansion by 4:00** (any race) → prepare for aggressive timing attack

### Phase 2: Early Game (~5 – ~10 min)

**Core objectives:** Secure natural expansion; apply or defend harassment; establish first technology tier.

This phase ends when a player has built the army composition for their chosen strategy. A worker advantage at 4 minutes compounds into larger army size at 8 minutes. First attacks typically occur during this phase, including:
- **Worker harassment** (SCV pulls, Drone pull all-ins)
- **First wave bio/ling/zealot timing pushes**
- **Proxy builds** (hidden early production structures near enemy base)

### Phase 3: Mid Game (~10 – ~20 min)

**Core objectives:** Secure third base; establish second tech tier; execute first major army engagement; harass with mobile units.

This phase is characterized by:
- **Multi-pronged attacks** — attacking multiple locations simultaneously to force army splitting
- **Drop harassment** — Medivac drops, Warp Prism attacks, Overlord Transport raids
- **Tech switches** — changing unit composition in response to scouted enemy tech
- **Upgrades** — +1/+1 attack/armor upgrades provide significant army-wide power spikes
- The completion of major research milestones (Stim Pack for Terran, +1 attack for Zerg) triggers natural **Timing Attacks**

### Phase 4: Late Game (~20 min+)

**Core objectives:** Control remaining expansion locations; maintain maximum supply (200); win decisive engagements.

Characteristics:
- Both players at or near 200 supply; economy is the limiting constraint
- **Base trade risk** — attacking the enemy base while your base is attacked simultaneously
- **High-tech units** dominate (Carriers, Ultralisks, Battlecruisers, Ghosts with nukes)
- Zerg's strength: fastest production rate allows replacing 200-supply armies rapidly
- Protoss strength: expensive but cost-efficient army fights well even at numerical disadvantage
- Terran strength: positional play, defensive siege lines, drop-based attrition

---

## Part III: Key Strategic Concepts

### 3.1 Macro vs. Micro

These two concepts represent the two main axes of StarCraft II skill.

**Macro (Macromanagement)** refers to everything involving economy and production:
- Keeping workers constantly training at all Town Halls
- Building supply structures before hitting the cap
- Spending resources efficiently (never floating large amounts of unspent minerals/gas)
- Taking expansions at the right time
- Researching upgrades at appropriate timing
- Race-specific macro mechanics: MULEs (Terran), Queen larva inject (Zerg), Chrono Boost (Protoss)

A player with superior macro will have a larger, better-equipped army by default, regardless of tactical skill. At lower skill levels, macro errors account for most of the performance gap.

**Micro (Micromanagement)** refers to individual unit control during combat:
- **Focus firing** — concentrating all attacks on a single enemy unit to kill it before it can deal more damage
- **Kiting / Dancing** — retreating damaged units while healthy ones continue attacking
- **Positioning** — using terrain, choke points, and high-ground advantages
- **Ability usage** — activating unit spells (Stim Pack, Blink, Transfusion) at optimal moments
- **Flanking** — attacking from multiple directions to maximize damage output and minimize splash damage received
- **Spell landing** — placing AoE spells (Psionic Storm, Infested Terran, Fungal Growth) optimally

> "Macro is the floor; micro is the ceiling." — common SC2 coaching adage. Superior macro guarantees a resource advantage; superior micro extracts maximum value from that army.

### 3.2 Build Orders

A **build order** is a precise, tested sequence of constructions and unit productions in the early game, specified by supply count milestones. Using a practiced build order frees cognitive resources for scouting and responding to the opponent.

**Build order notation format:**

```
[supply count]  [action]
```

Example — Terran **Reaper Expand** (standard macro opener):

```
14    Supply Depot
16    Barracks
16    Refinery
@Barracks 100%: Orbital Command
20    Command Center (natural expansion)
20    Reaper (from Barracks)
22    Supply Depot
22    Factory
@Factory 100%: Starport
...
```

Example — Zerg **Hatch First** (standard macro opener):

```
17    Hatchery (natural expansion)
17    Spawning Pool
18    Extractor
@Spawning Pool 100%: Queen x2 (each Hatchery)
20    Overlord
@100 gas: Lair or Roach Warren depending on strategy
...
```

Example — Protoss **4 Gate** (aggressive rush build):

```
9     Pylon
12    Gateway
13    Assimilator
15    Pylon
16    Cybernetics Core
@Cybernetics Core 100%: Warp Gate Research (Chrono Boost)
20    Gateway x3 (total 4 Gateways)
20    Pylon
@Warp Gate 100%: Warp in 4 Zealots + 4 Stalkers simultaneously
→ Push across map
```

Common build order archetypes by intent:

| Type | Intent | Risk | Example |
|---|---|---|---|
| **Macro Opener** | Fast economy, delayed army | Vulnerable to early aggression | Hatch First (Zerg), Reaper Expand (Terran) |
| **All-in Rush** | Win or lose on one early attack | Loses if defended; no fallback | 6-Pool (Zerg), Cannon Rush (Protoss) |
| **Timing Attack** | Hit a specific power spike | Requires precise execution | 4-Gate (Protoss), 1-1-1 (Terran) |
| **Tech Rush** | Fast advanced units | Economically weak during tech | Banshee rush (Terran), DT rush (Protoss) |

### 3.3 Scouting

Scouting is the process of gathering information about the opponent's strategy by sending units into their base and across the map.

**Standard scouting sequence:**
1. **~1:30–2:00**: Send a worker (SCV/Probe/Drone) to find the enemy base on multi-spawn maps
2. **~3:00**: Worker enters enemy base; identify number of production buildings, gas, and proxy structures
3. **~5:00**: Pull worker out (or sacrifice it for more information); replace with a dedicated scouting unit (Reaper, Zling/Overlord, Oracle/Adept)
4. **Ongoing**: Maintain a unit at watch positions near enemy base to detect army movement

**Information to scout for:**
- Is the opponent expanding? (greedy economy) → you can be more aggressive
- Are there multiple production buildings? → larger army incoming
- What technology tier are they on? → indicates their future army composition
- Are there proxy structures? → imminent rush

**React, don't just scout:** Scouted information only has value if acted upon. Professional players pre-plan responses to common scout outcomes (e.g., "if I see two early Barracks, I build 2 Spine Crawlers immediately").

### 3.4 Timing Attacks

A **timing attack** is an assault timed to exploit a specific window of inherent weakness in the opponent's build. Every build order creates such windows:
- **During early expansion** — when a player has just taken their second base, their army is split and weak
- **Before key tech completes** — attacking before Stim Pack, +1 attack upgrade, or critical defensive buildings complete
- **After a large unit investment** — when the opponent has just spent all their resources on buildings and has no army yet

Examples of classic timing attacks:
- **Two-Rax pressure** (Terran): ~4:00 Marine timing while opponent is building their natural expansion
- **Roach/Ravager all-in** (Zerg): ~7:00 push before Protoss has Colossi or Immortals
- **4-Gate** (Protoss): ~6:00 Zealot/Stalker push enabled by Warp Gate completion

---

## Part IV: TextStarCraft2 Framework

### 4.1 Overview and Architecture

**TextStarCraft2** ([arXiv:2312.11865](https://arxiv.org/abs/2312.11865), NeurIPS 2024) is a text-based interface for StarCraft II that enables LLM agents to interact with the game using natural language. Built on top of `python-sc2`, it reformulates the SC2 environment as a multi-agent **Partially Observable Markov Decision Process (POMDP)**.

The framework consists of two core adapter components:

```
+------------------+     Obs-to-Text     +------------------+     Text-to-Action  +------------------+
|  StarCraft II    | ─────────────────► |   LLM Agent      | ─────────────────► |  SC2 Game Engine  |
|  Game Engine     |                    |   (GPT/LLaMA)    |                    |  (python-sc2)     |
|  (python-sc2)    | ◄───────────────── |                  | ◄───────────────── |                   |
+------------------+   Action Execution  +------------------+      Output         +------------------+
```

1. **Observation-to-Text Adapter** (`get_information` function in `Protoss_bot.py`): Transforms raw game state (unit positions, resource counts, build status) into structured natural language text that LLMs can parse and reason over.

2. **Text-to-Action Adapter** (`handle_action` functions): Interprets natural language commands from the LLM and translates them into actual game commands executed by the `python-sc2` bot framework.

**Important constraint:** TextStarCraft2 currently supports only **Protoss** as the player's race. The opponent is typically a Zerg AI, though Terran and Protoss opponents are also compatible. Micro-operations (individual unit combat control) are handled automatically by rule-based scripts, freeing the LLM to focus on **macro-level strategic decisions**.

**Interaction frequency:** The environment operates at a **2:1 ratio** — one TextStarCraft2 step represents 2 actual SC2 game frames — reducing the query rate to LLMs. Despite this, a single game can take approximately **7 hours** of wall-clock time due to LLM inference latency.

### 4.2 Observation Format

The raw observation from each game step is transformed into structured text. The information captured includes:

| Component | Description | Example Format |
|---|---|---|
| **Game Time** | Current elapsed game time | `"At 3:27 game time"` |
| **Resources** | Minerals and gas amounts | `"135 minerals, 80 vespene gas"` |
| **Supply** | Current / cap and army supply | `"supply cap at 28 with 6 supply left, army supply is 8"` |
| **Workers** | Number of active workers | `"18 workers"` |
| **Structures** | Active buildings and their states | `"1 Nexus, 2 Gateways, 1 Cybernetics Core"` |
| **Production Queue** | Units currently being trained | `"Training: Stalker x2, Probe x1"` |
| **Army** | Own units with types and counts | `"Army: 6 Stalkers, 4 Zealots"` |
| **Enemy Intel** | Visible enemy units (fog-limited) | `"Visible enemy: 3 Roaches near north ramp"` |
| **Upgrades** | Completed research | `"Warp Gate research complete"` |

**Example raw observation text (from the paper):**

```
At 00:46 game time, the current situation is as follows:
- Workers: 15 workers
- Resources: 105 minerals, 0 vespene gas
- Supply: cap at 23, 7 supply remaining; army supply is 16
- Bases: 1 Nexus
- Structures: 1 Pylon, 2 Assimilators (under construction)
- Army: no combat units yet
- Enemy: no known enemy units
```

### 4.3 Chain of Summarization (CoS) Method

Because raw step-by-step observations are verbose and LLMs have context window limits, TextStarCraft2 introduces the **Chain of Summarization** method to compress information hierarchically:

**Level 1 — Single-Frame Summarization** (file: `TextStarCraft2_2/summarize/L1_summarize.py`):
Compresses raw per-step observation into a concise situation report. Can be implemented as:
- A **rule-based approach** (deterministic text extraction/formatting)
- A **language model-based approach** (LLM generates a brief summary)

**Level 2 — Multi-Frame Summarization** (file: `TextStarCraft2_2/summarize/gpt_test/L2_summarize.py`):
Aggregates multiple L1 summaries across several game steps into a higher-level strategic assessment. Inspired by CPU caching and RL frame-skipping techniques, this addresses the mismatch between game pace and LLM processing speed.

**L2 Summary process steps:**
1. **Situation Overview** — current state of economy, army, tech, and bases
2. **Analysis** — assessment of strengths, weaknesses, and opportunities
3. **Strategic Planning** — high-level goals for the next 2–3 minutes
4. **Opponent Strategy Analysis** — inferred enemy build based on scouting
5. **Suggestion Formulation** — specific recommended actions
6. **Decision Output** — final action queue for execution

**Example L2 strategic output structure:**

```
Situation Overview:
  Economy: 18 probes, 2 bases, ~800 minerals/min, 300 gas/min
  Army: 12 Stalkers, 8 Zealots, 2 Immortals (supply: 64/72)
  Tech: Robotics Facility complete; Blink research at 40%

Analysis:
  Strong economic lead. Army is mid-strength. Blink timing ~90s away.

Strategic Plan:
  1. Expand to third base in 30 seconds
  2. Continue warp-in cycle every 28s
  3. Harass enemy natural with Stalker group

Opponent Intel:
  Zerg has Lair (mid-tier tech). Roaches and Ravagers likely.
  No Spire seen — air units not imminent.

Action Queue:
  - build_nexus(probe_id=42, location=third_base_position)
  - warp_in(unit=stalker, count=4, pylon=proxy_pylon)
  - attack(group=stalker_patrol, target=enemy_natural_ramp)
```

### 4.4 Action Space

The action space (file: `TextStarCraft2_2/utils/action_info.py`) covers strategic macro commands grouped by category. Actions are expressed in natural language and parsed by the Text-to-Action adapter.

| Category | Example Actions |
|---|---|
| **Economy** | Build Nexus (expand), Train Probe, Assign workers to gas |
| **Production Buildings** | Build Gateway, Build Robotics Facility, Build Stargate |
| **Unit Training** | Warp-in Zealot, Train Stalker, Train Immortal, Train Colossus |
| **Research** | Research Warp Gate, Research Blink, Research Charge |
| **Supply** | Build Pylon |
| **Combat** | Attack (move army to enemy position), Retreat, Hold position |
| **Scouting** | Scout enemy base (with a designated unit) |
| **Expansion** | Take third base, Take fourth base |

Actions are expressed as structured text commands. Example LLM action output:

```json
{
  "action": "build",
  "target": "Pylon",
  "worker": "probe_12",
  "location": "near_gateway_cluster"
}
```

or in plain text:
```
BUILD Pylon near existing Gateway cluster to increase supply cap.
WARP_IN 4 Stalkers at proxy Pylon near enemy natural.
```

### 4.5 Difficulty Levels and Evaluation Scenarios

TextStarCraft2 offers **10 difficulty levels** ranging from Level 1 (VeryEasy) to Level 10 (CheatInsane). The names differ from the StarCraft II client UI but the underlying built-in AI behavior is identical.

| Level | TextStarCraft2 Name | Approximate Human Equivalent |
|---|---|---|
| 1 | VeryEasy | Complete beginner |
| 2 | Easy | Novice |
| 3 | Medium | Casual player |
| 4 | MediumHard | Intermediate |
| 5 | Harder | Bronze–Silver league |
| 6 | VeryHard | Gold–Platinum league |
| 7 | Elite | Diamond league |
| 8–10 | CheatVision / CheatMoney / CheatInsane | Superhuman (cheating AI) |

**Opponent build order types tested:**
`randombuild`, `rush`, `timing`, `power`, `macro`, `air`

**Evaluation metrics defined in the framework:**

| Metric | Abbreviation | Description |
|---|---|---|
| **Win Rate** | WR | Percentage of games won (primary metric) |
| **Population Block Ratio** | PBR | Fraction of game time spent at supply cap (lower = better macro) |
| **Resource Utilization Ratio** | RUR | Ratio of unspent resources to total income (lower = better spending) |
| **Average Population Utilization** | APU | Mean supply used / supply cap over the game |

---

## Part V: LLM Performance and Challenges

### 5.1 Key Research Findings

Research on LLMs playing StarCraft II through TextStarCraft2 has produced a series of landmark results:

| Milestone | Method | Result | Source |
|---|---|---|---|
| First LLM vs. built-in AI | Chain of Summarization (CoS) with GPT-3.5-turbo-16k | Defeated Level 5 (Harder) AI | NeurIPS 2024, arXiv:2312.11865 |
| Knowledge evaluation | Expert (Grandmaster-level) Q&A assessment | GPT-4 ranked highest; LLMs judged equivalent to ~8-year veteran player | NeurIPS 2024 |
| Human vs. LLM matches | Fine-tuned LLaMA2-7B and ChatGLM2-6B | On par with Gold-league human players | NeurIPS 2024 |
| First Elite AI defeat | Hierarchical Expert Prompt (HEP) | Defeated Level 7 (Elite) for the first time | arXiv:2502.11122, Feb 2025 |
| Very Hard difficulty | Reflection of Episodes (ROE) | Defeated Level 6 (VeryHard) | arXiv:2502.13388 |
| Small model fine-tuning | SC-Phi2 (2.8B parameters, Phi-2 + ViT) | Outperformed prior approaches on build order prediction | arXiv:2409.18989 |

### 5.2 LLM Strengths in SC2

Compared to both rule-based AI and RL-based agents (like AlphaStar), LLMs show distinctive advantages:

- **Strategic knowledge**: LLMs pre-trained on game wikis, forums, and strategy guides carry substantial SC2 domain knowledge without game-specific training
- **Interpretable decisions**: LLMs can explain their reasoning in natural language ("I'm building Immortals because I scouted Roaches"), unlike opaque RL policies
- **Flexible strategy adaptation**: LLMs can pivot strategies in response to novel situations without retraining
- **Long-horizon planning**: LLMs can plan 10–15 step build order sequences and reason about opponent responses
- **Human-like communication**: LLMs can serve as assistants, coaches, or teammates for human players with explanatory capability

### 5.3 LLM Weaknesses and Challenges

| Challenge | Description | Impact |
|---|---|---|
| **No real-time micro** | LLMs cannot execute frame-by-frame unit control (micro) — TextStarCraft2 relies on rule-based scripts for this | Limits combat effectiveness |
| **Inference latency** | LLM query cycles take seconds; a single game takes ~7 hours | Precludes use in real-time competitive play |
| **Context window limitations** | Long game histories must be compressed; information loss is unavoidable | Impairs long-horizon state tracking |
| **Hallucination** | LLMs sometimes issue invalid actions or reference non-existent units | Causes wasted turns and strategy incoherence |
| **Tech tree reasoning** | Models struggle with precise dependency chains and supply arithmetic | Leads to build order errors and supply blocks |
| **No visual input** | Current TextStarCraft2 is text-only; LLMs cannot process the minimap or unit positions spatially | Limits spatial reasoning |
| **Multi-agent coordination** | In team scenarios, LLMs struggle with communication and role division | Reduces multi-agent performance |

### 5.4 LLMs vs. Prior AI Systems

| System | Approach | Peak Performance | Notes |
|---|---|---|---|
| **Rule-based AI** (built-in) | Hard-coded decision trees | ~Level 7 (Elite) | No learning; predictable |
| **AlphaStar** (DeepMind, 2019) | RL + supervised pretraining from 971K replays | Grandmaster level (top 0.2%) | Required 64+ V100 GPUs; 200 years of gameplay equivalent |
| **LLM + CoS** (2024) | Prompted GPT-3.5/4 + summarization | Level 5 (Harder) | No game-specific training |
| **LLM + HEP** (2025) | Hierarchical prompting + expert tactics | Level 7 (Elite) | First LLM to beat Elite AI |
| **Fine-tuned LLM** | LoRA fine-tune on SC2 replays | ~Gold-league human level | Efficient: single GPU training |

AlphaStar's strengths (superhuman micro, perfect supply management, optimal build order execution) are precisely the areas where LLMs are weakest. Conversely, LLMs' strengths (domain knowledge, interpretability, adaptive planning) are AlphaStar's documented weaknesses.

---

## Part VI: RL Training Considerations

This section focuses on the most valuable aspects of training RL agents (or fine-tuning LLMs) for StarCraft II.

### 6.1 Reward Signal Design

The choice of reward signal is the single most impactful design decision in SC2 RL training.

**Primary reward (sparse):**
- **Win/Loss**: +1 for win, -1 for loss, 0 for draw. Used by AlphaStar and most competitive systems. Provides clean signal but suffers from extreme sparsity — games can last 15–30+ minutes with no intermediate reward.

**Intermediate (dense) rewards — AlphaStar's approach:**

AlphaStar's policy is conditioned on a **z-statistic** — a vectorized strategy descriptor sampled from human replay data — containing:
- **Build order component**: The first 20 buildings/units constructed, encoded as a sequence
- **Cumulative statistics component**: Aggregate game statistics (total units produced by type, structures built, etc.)

Pseudo-rewards measuring:
- **Edit distance** between the sampled build order and the executed build order
- **Hamming distance** between sampled and executed cumulative statistics

These pseudo-rewards guide the agent toward human-like strategic patterns while still optimizing for wins.

**Additional reward signals used in research:**

| Reward Signal | Description | Source |
|---|---|---|
| **Scouting reward** | Cross-entropy between true and predicted opponent strategy (rewards efficient information gathering) | ROA-Star (NeurIPS 2023) |
| **Economy development** | Tracked as a strategic dimension in evaluation | HEP framework (2025) |
| **Military effectiveness** | Unit kill/loss ratios or army value destroyed | Standard RL shaping |
| **Supply efficiency** | Penalize supply blocks (being capped); reward consistent near-cap supply usage | TextStarCraft2 PBR metric |
| **Resource utilization** | Penalize floating unspent resources | TextStarCraft2 RUR metric |
| **Tech progression** | Reward completing key technology milestones | Custom shaping |

**Reward signal recommendations for LLM fine-tuning:**

The most information-dense reward signals for trajectory-based fine-tuning are:
1. **Win/Loss** (most authoritative, but sparse)
2. **Build order adherence** (direct measurement of planning quality)
3. **Worker/production efficiency** (measures macro fundamentals)
4. **Supply block frequency** (proxy for macro management quality)
5. **Expansion timing** (measures economic aggression vs. caution)

### 6.2 Trajectory Data Quality

For offline RL or supervised fine-tuning, the quality and diversity of trajectory data determines agent ceiling performance.

**AlphaStar's dataset**: 971,000 human replays, filtered and weighted by player skill rating (MMR). High-MMR replays provided cleaner demonstrations of correct macro/micro.

**What makes a high-quality SC2 trajectory:**

| Quality Dimension | High Quality | Low Quality |
|---|---|---|
| **Player skill** | Diamond+ / Master / Grandmaster replays | Bronze/Silver replays (full of macro errors) |
| **Build order consistency** | Clear, executed build order with minimal deviations | Improvised, supply-blocked, economy-neglected |
| **Scouting behavior** | Regular scouting; strategy adjustments based on info | Blind play; no adaptation to opponent |
| **Supply management** | Rarely supply blocked; supply structures built proactively | Frequent supply blocks; army production halted |
| **Resource spending** | Minerals/gas consistently spent; low floating resources | Large unspent resource pools at game end |
| **Game length coverage** | Mix of short (all-in wins), medium (timing wins), long (macro wins) | Only one game-length archetype |
| **Matchup diversity** | Coverage across TvZ, TvP, TvT, ZvP, ZvZ, PvP | Single matchup only |
| **Outcome diversity** | Mix of wins and losses (defeats show failure modes) | Wins only (no failure recovery) |

**Trajectory annotation for LLM training:**
Beyond raw replays, highest-quality data for LLM fine-tuning includes:
- **Natural language commentary** paired with game states (narrates the strategic intent behind actions)
- **Annotated build orders** with explanations of why each decision was made
- **Expert post-game analysis** identifying decision points and errors

### 6.3 Key Capability Dimensions

For evaluating and training LLM agents on SC2, six capability dimensions have emerged from the research literature as most important:

| Dimension | Description | Proxy Metric |
|---|---|---|
| **Economy Management** | Worker production, resource spending efficiency, expansion timing | Resource Utilization Ratio (RUR), worker count at benchmark timestamps |
| **Tech Tree Navigation** | Correct build order execution, technology unlock sequencing | Build order accuracy vs. target sequence |
| **Supply Management** | Avoiding supply blocks, proactive supply building | Population Block Ratio (PBR) |
| **Military Development** | Army composition quality, unit production continuity | Army supply / max supply ratio |
| **Scouting & Adaptation** | Information gathering, strategy pivots in response to opponent | Scouting frequency, strategy switch rate |
| **Strategic Planning** | Multi-step plan formulation and execution over 5–15 min horizons | Plan coherence, milestone completion rate |

Research findings on relative importance:
- At low difficulty levels (1–5), **economy management and supply management** dominate performance differences
- At mid difficulty levels (5–7), **scouting and adaptation** become the differentiating factor
- At high difficulty levels (7+), **strategic planning and tech tree navigation** are the critical bottlenecks for LLMs

---

## Core Challenges for LLM Agents (Summary)

| Challenge | Description |
|-----------|-------------|
| **Resource Management** | Gather minerals and vespene gas efficiently; balance spending across units, buildings, and upgrades |
| **Base Building** | Construct and upgrade structures in the right order to unlock technology advancements |
| **Army Control** | Train units, position armies, and execute attacks and retreats in real-time |
| **Fog of War** | Operate with limited map visibility; scout enemy positions and anticipate attacks |
| **Tech Tree Navigation** | Choose the correct research path based on opponent strategy |
| **Macro vs. Micro** | Balance high-level economic planning with low-level unit micro-management |
| **Supply Management** | Build supply structures proactively; never halt army production |
| **Strategy Adaptation** | Pivot plans based on scouted information |

---

## Observation Space

Each step, the agent receives a structured text observation containing:

| Component | Description |
|-----------|-------------|
| **Map View** | 2D grid of the map with terrain, structures, and visible units; includes unit types, positions, health, and ownership |
| **Resources** | Current minerals and vespene gas amounts; available workers and resource nodes |
| **Base State** | Active structures, ongoing production, tech upgrades, and army composition |
| **Enemy Intel** | Visible enemy units and structures (limited by fog of war) |
| **Game State** | Current game time, supply count, and alert conditions |

**Data formats:**

- Numerical: resource counts (integers), unit health (floats)
- Categorical: unit types (`stalker`, `zergling`), structure states
- Spatial: unit positions as (x, y) coordinates on the map grid

---

## Action Space

The action space covers high-level strategic commands (macro decisions):

| Category | Example Actions |
|----------|----------------|
| **Economy** | `build_nexus`, `train_probe`, `assign_workers_to_gas` |
| **Production** | `build_gateway`, `build_robotics_facility`, `build_stargate` |
| **Unit Training** | `warp_in_zealot`, `train_stalker`, `train_immortal`, `train_colossus` |
| **Research** | `research_warp_gate`, `research_blink`, `chrono_boost` |
| **Combat** | `attack(location)`, `scout(area)`, `retreat`, `hold_position` |
| **Expansion** | `expand(location)` |

**Action representation:**

- **Discrete**: Named macro actions from a predefined set (e.g., `build_gateway`, `warp_in_stalker`)
- **Positional**: Target map coordinates for movement and attack orders

---

## Evaluation Scenes

| Scene | Description | Key Skill Tested |
|-------|-------------|-----------------|
| **1v1 Standard Match** | Agent vs. AI-controlled Zerg at varying difficulty (Level 1–10) | Overall strategic capability |
| **Macro Management** | Extended games emphasizing base expansion and resource optimization | Long-horizon economic planning |
| **Timing Attack Defense** | Opponent uses rush/timing builds; agent must defend and recover | Scouting, adaptive response |
| **Late Game Composition** | Scenarios where both players reach 200 supply; agent must win the decisive engagement | Strategic army composition |

---

## Evaluation Metrics

| Metric | Abbreviation | Description |
|--------|--------|-------------|
| **Win Rate** | WR | Percentage of matches won against AI opponents at various difficulty levels |
| **Population Block Ratio** | PBR | Fraction of game time at supply cap (lower is better macro management) |
| **Resource Utilization Ratio** | RUR | Ratio of unspent resources (lower is better spending efficiency) |
| **Average Population Utilization** | APU | Mean supply used relative to supply cap across the game |
| **Build Order Accuracy** | — | Conformance of early actions to an optimal build order sequence |
| **Scouting Coverage** | — | Map exploration rate and quality of gathered enemy intelligence |

---

## Prompt Example

A typical LLM agent prompt in the StarCraft II environment (Chain of Summarization format):

```plaintext
=== SITUATION OVERVIEW ===
Game Time: 7:42
Resources: 320 minerals, 150 vespene gas
Workers: 22 probes
Supply: 44 / 54 (10 free)
Structures: 1 Nexus, 3 Gateways (1 building), 1 Cybernetics Core,
            1 Robotics Facility, 2 Pylons
Army: 8 Stalkers, 6 Zealots, 1 Immortal
Upgrades: Warp Gate (complete)
Enemy Intel: Zerg Hatchery at natural confirmed; 8 Roaches sighted
             moving toward your base from the north ramp.

=== ANALYSIS ===
Enemy is using a Roach timing push. Your Immortal provides
strong anti-Roach firepower (+50 bonus vs Armored). You have
sufficient force if positioned correctly at the ramp choke.

=== RECOMMENDED ACTION ===
1. Position Immortal and Stalkers at your natural ramp (choke point).
2. Place Zealots in front as meatshield.
3. Continue warp-in cycle with 4 more Stalkers immediately.
4. Build additional Pylon to prepare for next wave.

What will you do? Provide a specific action sequence.
```

---

## Setup and Dependencies

```bash
# Step 1: Install StarCraft II from Blizzard
# Download from: https://starcraft2.blizzard.com/

# Step 2: Place the evaluation map
# Copy to: StarCraft II/Maps/Ancient Cistern LE.SC2Map

# Step 3: Install TextStarCraft2 dependencies
pip install python-sc2
pip install openai  # or your LLM provider

# Step 4: Clone the TextStarCraft2 repository
git clone https://github.com/histmeisah/Large-Language-Models-play-StarCraftII
cd Large-Language-Models-play-StarCraftII

# Step 5: Configure your LLM API key
export OPENAI_API_KEY="your-key-here"

# Step 6: Run the agent
python run_game.py --difficulty 5 --llm gpt-3.5-turbo-16k
```

See [Quick Start](../../getting_started/quickstart.md) for full DSGBench setup instructions.

---

## Image and Diagram Resources

For visual diagrams of StarCraft II game mechanics:

- **Blizzard Press Center** ([blizzard.gamespress.com](https://blizzard.gamespress.com/StarCraft/Focus/StarCraft-II-screenshots)) — official screenshots available for press and fansite use under Blizzard's Fansite Kit license (non-commercial only)
- **Liquipedia SC2** ([liquipedia.net/starcraft2](https://liquipedia.net/starcraft2)) — interactive tech tree diagrams for all three races; Creative Commons-licensed player event photos
- **Wikimedia Commons** ([commons.wikimedia.org/wiki/Category:StarCraft_II](https://commons.wikimedia.org/wiki/Category:StarCraft_II)) — limited publicly available game-adjacent images
- **DeepMind/PySC2 GitHub** ([github.com/google-deepmind/pysc2](https://github.com/google-deepmind/pysc2)) — feature layer visualizations under Apache 2.0 license

> Note: StarCraft II screenshots and game assets are copyrighted by Blizzard Entertainment / Microsoft. They are not in the public domain and are not released under Creative Commons. All game images require either Blizzard's explicit permission or fall under a fair use argument. The Blizzard Fansite Kit permits personal non-commercial use only.

---

## Resources and References

- [TextStarCraft2 GitHub](https://github.com/histmeisah/Large-Language-Models-play-StarCraftII) — the underlying text-based SC2 environment
- [LLMs Play StarCraft II: Benchmarks and Chain of Summarization](https://arxiv.org/abs/2312.11865) — Ma et al., NeurIPS 2024
- [Hierarchical Expert Prompt for TextStarCraft II](https://arxiv.org/abs/2502.11122) — first Elite AI defeat, Feb 2025
- [LLM-PySC2: SC2 Learning Environment for LLMs](https://arxiv.org/abs/2411.05348) — NKAI / NUDT, NeurIPS 2025
- [SC-Phi2: Fine-tuned SLM for SC2 Macromanagement](https://arxiv.org/abs/2409.18989) — efficient fine-tuning approach
- [SC2LE: StarCraft II as an RL Environment](https://arxiv.org/abs/1708.04782) — DeepMind / Blizzard, 2017
- [AlphaStar: Grandmaster level via multi-agent RL](https://storage.googleapis.com/deepmind-media/research/alphastar/AlphaStar_unformatted.pdf) — Vinyals et al., Nature 2019
- [StarCraft II Unplugged: Offline RL](https://openreview.net/pdf?id=Np8Pumfoty) — offline RL benchmark and dataset
- [Liquipedia StarCraft II Encyclopedia](https://liquipedia.net/starcraft2) — comprehensive game mechanics reference
- [StarCraft II Official Site](https://starcraft2.blizzard.com/)
- [DSGBench Paper](https://arxiv.org/abs/2503.06047) — Section 3.1 for StarCraft II environment details
