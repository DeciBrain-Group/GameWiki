<div class="game-header">
  <div class="game-header-icon">🏛️</div>
  <div class="game-header-info">
    <h1>Civilization (FreeCiv)</h1>
    <div class="badge-row">
      <span class="badge badge-turnbased">Turn-based</span>
      <span class="badge badge-imperfect">Imperfect Info</span>
    </div>
    <p>4X strategy game testing long-horizon planning, resource allocation, technology tree navigation, and diplomatic decision-making across hundreds of turns.</p>
  </div>
</div>

## Game Overview

**Civilization** (implemented via [FreeCiv](https://www.freeciv.org/), an open-source Civilization clone) is a turn-based **4X strategy game** — **eXplore, eXpand, eXploit, eXterminate**. Players guide a civilization from 4000 BC through the modern era, managing cities, researching technologies, conducting diplomacy, and competing with AI-controlled civilizations for global dominance.

DSGBench uses the **CivRealm** framework ([ICLR 2024](https://arxiv.org/abs/2401.10568), [GitHub](https://github.com/bigai-ai/civrealm)), which wraps FreeCiv-web with structured text-based interfaces so LLM agents can receive game-state observations and issue strategic commands each turn.

### Why Civilization for LLM Evaluation?

Civilization uniquely tests a combination of capabilities that few other environments offer:

- **Ultra-long horizon planning** — games span 200–500 turns, requiring consistent multi-step strategy across an entire session
- **Exponentially expanding state space** — state space grows from ~10¹⁵ to ~10⁶⁵⁰ as the game progresses; action space from ~10⁴ to ~10¹⁶⁶
- **Multi-objective optimization** — simultaneously balance economy, military, science, culture, and diplomacy
- **Partial observability** — fog of war hides enemy positions, military strength, and resources
- **Delayed feedback** — strategic decisions may not show consequences for dozens of turns
- **Natural language diplomacy** — players communicate via chat and formal diplomatic actions using natural language
- **Dynamic action space** — available units, buildings, and technologies change as civilization advances

---

## The 4X Framework

The term "4X" stands for four core strategic dimensions, each essential to mastering Civilization:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      The 4X Framework                               │
│                                                                     │
│  eXplore      Send units to reveal the map, find resources,         │
│  (Turn 1–50)  meet other civilizations, locate strategic sites      │
│                                                                     │
│  eXpand       Found new cities, claim territory, push borders       │
│  (Turn 10–150)outward before opponents do                           │
│                                                                     │
│  eXploit      Work tiles, build improvements, establish trade       │
│  (Ongoing)    routes, generate food/production/gold/science         │
│                                                                     │
│  eXterminate  Wage war, capture cities, eliminate threats,          │
│  (Optional)   defend against attack — or avoid conflict entirely    │
└─────────────────────────────────────────────────────────────────────┘
```

| Dimension | Core Mechanics | Key Units/Buildings |
|-----------|----------------|---------------------|
| **eXplore** | Fog of war, scouting, meeting civilizations, finding resources | Explorer, Trireme, Caravel |
| **eXpand** | City founding, border growth, settler production | Settlers, Workers, Engineers |
| **eXploit** | Tile improvements, trade routes, wonder construction, science/culture generation | Caravan, Freight, all city buildings |
| **eXterminate** | Combat, city capture, diplomacy as warfare tool | All military units, Diplomat, Spy |

---

## Turn Structure

Each game turn represents a specific historical time period. Early turns cover centuries; later turns cover single years:

```
Ancient Era      4000 BC – 1000 BC    ~1 turn = 50–100 years
Medieval Era     1000 BC – 1000 AD    ~1 turn = 25–50 years
Industrial Era   1000 AD – 1800 AD    ~1 turn = 10–20 years
Modern Era       1800 AD – 1900 AD    ~1 turn = 5 years
Information Age  1900 AD – present    ~1 turn = 1–2 years
```

**Within each turn, a player must:**

1. Move military and civilian units
2. Manage city production queues (what to build next)
3. Set research direction (which technology to investigate)
4. Conduct diplomacy with other civilizations
5. Adjust economic sliders (tax/science/luxury rates)
6. End turn — all cities grow/produce, opponents take their turns

---

## Core Game Systems

### City System

Cities are the fundamental building block of a civilization. Every resource, unit, and building originates from a city.

```
City Tile Layout (3×3 radius, 21 workable tiles):

        [ ] [ ] [ ] [ ] [ ]
        [ ] [T] [T] [T] [ ]
        [ ] [T] [C] [T] [ ]
        [ ] [T] [T] [T] [ ]
        [ ] [ ] [ ] [ ] [ ]

[C] = City center tile (always worked)
[T] = Workable tile (citizens work these to generate resources)
[ ] = Border territory (claimed but not worked)
```

**City resource outputs per turn:**

| Resource | Symbol | Generated By | Used For |
|----------|--------|--------------|----------|
| **Food** | Food | Grassland, rivers, farms, fisheries | Population growth |
| **Production** | Shields | Hills, mines, forests, factories | Building units and structures |
| **Gold** | Gold | Trade routes, commerce tiles, markets | Upkeep, buying units |
| **Science** | Bulbs | Libraries, universities, scientists | Technology research |
| **Culture** | Culture | Temples, cathedrals, great wonders | Border expansion, policies |
| **Luxuries** | Luxuries | Luxury resources, markets | Citizen happiness |

**City Growth Mechanics:**

- Population grows when the granary fills (food surplus accumulates each turn)
- Each new citizen requires one additional food unit per turn for upkeep
- Larger cities produce more of everything but require more food and can create unhappiness
- Corruption reduces gold output — cities farther from the capital suffer more corruption

**City Founding Strategy — Key Considerations:**

```
Good city site checklist:
  ✓ Adjacent to river (bonus trade; needed for some buildings)
  ✓ Multiple food resources (grassland, game, fish) for growth
  ✓ Hills or mountains nearby (shields for production)
  ✓ Trade resources or luxury goods in radius
  ✓ Defensible terrain (hills raise defense modifier)
  ✓ Coastal access (enables naval units, trade)
  ✗ Avoid: isolated tundra/desert with no resources
  ✗ Avoid: too close to existing cities (tile overlap wastes potential)
```

---

### Technology Tree

The technology tree spans five broad eras. Each technology costs a fixed number of **research points (bulbs)** that accumulates from science output. Cost formula: **(r + 2) × √(r + 2) × 10** bulbs, where **r** = number of prerequisite technologies.

#### Era Overview

```
ANCIENT ERA (4000 BC – 500 BC)
Alphabet ──► Writing ──► Philosophy ──► Democracy
    └──► Map Making ──► Navigation
    └──► Bronze Working ──► Iron Working ──► Warrior Code
    └──► Pottery ──► Monarchy
    └──► Ceremonial Burial ──► Mysticism ──► Religion
Agriculture ──► Irrigation ──► Monarchy

MEDIEVAL ERA (500 BC – 1500 AD)
Mathematics ──► Engineering ──► Construction ──► Bridge Building
Feudalism ──► Chivalry ──► Leadership
Chemistry ──► Gunpowder ──► Explosives
University ──► Astronomy ──► Navigation
Music Theory ──► Printing Press

INDUSTRIAL ERA (1500 AD – 1800 AD)
Steam Engine ──► Railroad ──► Industrialization
Physics ──► Electricity ──► Magnetism
Theory of Gravity ──► Nuclear Fission
Medicine ──► Sanitation

MODERN ERA (1800 AD – 1950 AD)
Combustion ──► Automobile ──► Advanced Flight
Electronics ──► Radio ──► Computers
Atomic Theory ──► Nuclear Power ──► Space Flight
Plastics ──► Mass Production

INFORMATION AGE (1950 AD – present)
Miniaturization ──► Computers ──► Robotics
Genetic Engineering ──► Future Tech
Space Flight ──► Rocketry ──► Apollo Program
```

#### Key Technology Stats (FreeCiv Classic Ruleset)

| Technology | Prerequisites | Key Unlock |
|-----------|---------------|------------|
| **Alphabet** | (none) | Enables diplomatic actions |
| **Bronze Working** | (none) | Phalanx unit, enables mining |
| **Iron Working** | Bronze Working | Legion unit, improved combat |
| **Pottery** | (none) | Granary building |
| **Wheel** | (none) | Chariot unit |
| **Writing** | Alphabet | Library, enables embassies |
| **Monarchy** | Ceremonial Burial, Code of Laws | Government switch from Despotism |
| **Republic** | Code of Laws, Literacy | Republican government |
| **Democracy** | Republic, Invention | Democratic government |
| **Communism** | Philosophy, Industrialization | Communist government |
| **Map Making** | Alphabet | Trireme ship, ocean exploration |
| **Navigation** | Astronomy, Map Making | Caravel, ocean exploration |
| **Gunpowder** | Chemistry, Invention | Musketeers unit |
| **Steam Engine** | Physics, Invention | Railroad (with Railroad tech) |
| **Railroad** | Steam Engine | Movement bonus; rail trade |
| **Industrialization** | Railroad, Banking | Factory building |
| **Electricity** | Magnetism | Power Plant improvements |
| **Nuclear Fission** | Atomic Theory, Mass Production | Manhattan Project, nuclear weapons |
| **Space Flight** | Computers, Rocketry | Apollo Program, spacecraft |
| **Apollo Program** | Space Flight | Enables spaceship victory race |

The full game features **87 technology types** (in the CivRealm/FreeCiv ruleset used by DSGBench).

---

### Resource System

Resources appear on map tiles and provide bonuses to adjacent cities that work those tiles.

```
Resource Categories:

STRATEGIC RESOURCES          LUXURY RESOURCES
(required for units)         (happiness bonus)
─────────────────────        ─────────────────
Iron     → Legion            Gold     → +1 luxury per city
Horses   → Cavalry/Chariot   Gems     → +1 luxury per city
Coal     → Industrialization  Spice    → +1 luxury per city
Oil      → Modern units      Silk     → +1 luxury per city
Uranium  → Nuclear           Wine     → +1 luxury per city

FOOD/PRODUCTION RESOURCES
(tile yield bonus)
─────────────────────
Game     → +3 food (forests)
Fish     → +2 food (ocean)
Wheat    → +2 food (plains)
Buffalo  → +1 food +1 shield (grassland)
Pheasant → +1 food +1 shield (forest)
Coal     → +2 shields (hills)
Gold     → +1 trade (mountains)
```

**Resource Access:**

- Strategic resources must be within a city's working radius AND connected by road to the capital
- Luxury resources improve happiness: each unique luxury type reduces unhappiness by 1 across all cities
- If a resource is cut off (road destroyed, city captured), associated units may be disbanded

---

### Government System

FreeCiv features **6 government types**, each with different trade-offs for military, economic efficiency, and diplomatic flexibility:

| Government | Available From | Free Unit Support | Trade Bonus | Senate? | Notes |
|-----------|---------------|-------------------|-------------|---------|-------|
| **Despotism** | Start | 1 per city | None | No | Starting government; tile output penalty (-1 to any tile producing 3+) |
| **Monarchy** | Monarchy tech | 3 per city | None | No | Balanced; reduced corruption vs. Despotism |
| **Communism** | Communism tech | 3 per city | None | No | Flat corruption rate; strong military upkeep |
| **Republic** | Republic tech | 1 per city | +1 trade/tile | Yes | High production efficiency; senate limits war |
| **Democracy** | Democracy tech | 1 per city | +1 trade/tile | Yes | Maximum trade; senate strongly limits aggression |
| **Fundamentalism** | Theology tech | 2 per city | None | No | Reduced unit unhappiness; moderate trade |

**Senate Mechanic:** Under Republic and Democracy, a Senate can **block unprovoked declarations of war**. This is a significant constraint on aggressive strategies with these governments. Switching to Communism before a planned war is a common tactic.

**Switching Governments:** Changing government causes a period of **Anarchy** (typically 1–3 turns), during which no science is produced and cities may riot. The Statue of Liberty wonder eliminates anarchy transitions.

---

### Diplomatic System

Diplomacy is one of FreeCiv's most complex systems, with five distinct relationship states:

```
Diplomatic State Machine:

  [WAR] ──────────────────────────────────────────────►[CEASE-FIRE]
    ▲   (agree cease-fire)                               │
    │                                                    │ (16 turns pass)
    │ (declare war / cease-fire expires)                 ▼
    └─────────────────────────────────────────────[ARMISTICE]
                                                         │
                                                         │ (16 turns pass)
                                                         ▼
                                                      [PEACE]
                                                         │
                                                         │ (negotiate alliance)
                                                         ▼
                                                    [ALLIANCE]
```

| State | Duration | What is Permitted |
|-------|----------|-------------------|
| **War** | Until treaty | Attack units, capture cities, incite revolt |
| **Cease-Fire** | 16 turns (→ War) | No attacks; temporary halt |
| **Armistice** | 16 turns (→ Peace) | Units relocating out of territory |
| **Peace** | Until broken | Shared borders; no attacks; zones of control |
| **Alliance** | Until broken | Share tiles; no zones of control; coordinate war |

**Diplomatic Actions Available:**

- Establishing embassies (via Diplomat unit)
- Trading gold, technology, maps, territory
- Proposing cease-fire, peace, or alliance
- Declaring war (may trigger Senate veto under Republic/Democracy)
- Sending threats or making demands
- Natural language chat (full text communication in multiplayer/LLM mode)

**Espionage Actions (via Spy unit):**

- Investigate cities (reveal population, buildings, production)
- Steal technologies
- Sabotage city buildings or production
- Incite city revolt (causes city to defect to your civilization)
- Plant nukes

---

### Victory Conditions

FreeCiv supports three primary victory conditions:

| Victory Type | Requirement | Typical Game Length |
|-------------|-------------|---------------------|
| **Space Race (Peaceful)** | Be first civilization to complete and launch a spaceship to Alpha Centauri | Long (full tech tree needed) |
| **Domination (Military)** | Conquer and hold all cities of all other civilizations (or eliminate all rivals) | Variable |
| **Score (Default)** | Highest civilization score at game end (turn limit) | Fixed turn limit |

**Score Victory** is the most common scenario for LLM evaluation, as it allows measuring performance at any point in the game. Score is computed from population, buildings, technologies, wonders, territory, and other factors.

**Space Race Victory** requires completing the Apollo Program wonder, then building spacecraft components:
- Up to 16 structural components (reduce travel time)
- 3 module types: habitation (10,000 colonists), life support, solar panels
- All component pairs must be built before launch

---

## FreeCiv Implementation

FreeCiv is a **free and open-source** implementation of the Civilization series, released under the **GNU GPL-2.0-or-later** license. It most closely resembles **Civilization II** in rules and graphics.

### FreeCiv vs. Commercial Civilization Series

| Aspect | FreeCiv | Civ II | Civ V | Civ VI |
|--------|---------|--------|-------|--------|
| **License** | GPL open-source | Commercial | Commercial | Commercial |
| **Default Ruleset** | Civ II-style | — | — | — |
| **Technology Count** | 87 | ~88 | 81 | 67 |
| **Unit Count** | 52 | ~60 | 62 | 72 |
| **Building Count** | 68 | ~55 | 57 | 69 |
| **Government Types** | 6 | 6 | 5 (policies) | 10 (policies) |
| **Diplomatic States** | 5 | 5 | 5 | 6 |
| **Map Flexibility** | High (random + real-world) | Medium | Medium | Medium |
| **AI Research Access** | Full source code | Limited | SDK only | SDK only |
| **Web Playable** | Yes (FreeCiv-web) | No | No | No |
| **Customizability** | Very high (rulesets) | Low | Modding | Modding |

### FreeCiv-Specific Features

**Corruption System:** Cities lose trade resources proportional to distance from the capital. Government type controls severity — Communism uses flat corruption; Democracy/Republic minimize it.

**Partisans:** When a city with Democracy or Communism is conquered and the defender has Gunpowder, **partisan units** spawn automatically at no cost to the defender. Quantity depends on city size.

**Civil War:** If a civilization's capital is captured, there is a chance of **civil war** — the civilization splits into two. Probability increases under despotic/monarchic governments.

**Ruleset Configurability:** FreeCiv supports multiple built-in rulesets:
- `classic` — Default (Civ II emulation)
- `civ2civ3` — Civ II rules with Civ III balance improvements
- `multiplayer` — Optimized for competitive human play
- `sandbox` — Experimental/testing

### FreeCiv Terrain Types

| Terrain | Food | Shields | Trade | Movement Cost | Defense Bonus | Notes |
|---------|------|---------|-------|---------------|---------------|-------|
| **Grassland** | 2 | 0 | 0 | 1 | 0% | Best for food/population |
| **Plains** | 1 | 1 | 0 | 1 | 0% | Balanced output |
| **Hills** | 1 | 2 | 0 | 2 | +100% | Good shields; can be mined |
| **Mountains** | 0 | 1 | 0 | 3 | +200% | Hard to move; very defensive |
| **Forest** | 1 | 2 | 0 | 2 | +50% | Can be cleared for plains |
| **Jungle** | 1 | 0 | 0 | 3 | +50% | Requires clearing |
| **Desert** | 0 | 1 | 1 | 1 | 0% | Oil wells possible with Construction |
| **Tundra** | 1 | 0 | 0 | 1 | 0% | Poor output |
| **Ocean** | 1 | 0 | 2 | — | — | Fish resources add food |
| **River** (modifier) | +0 | +0 | +1 | — | — | Bonus trade; required by some buildings |

### Unit Statistics Reference

FreeCiv's **52 unit types** span civilian, military, and special roles:

```
Unit Stats Format: [Attack / Defense / HP / Firepower / Movement / Cost]

EARLY UNITS (Ancient Era):
  Warriors      [1/1/10/1/1/10]   Basic defense; no tech required
  Phalanx       [1/2/10/1/1/20]   Better defense; Bronze Working
  Archers       [3/2/10/1/1/30]   Ranged attack; Warrior Code
  Legion        [3/2/10/1/1/40]   Strong early offensive; Iron Working
  Chariot       [3/1/10/1/2/30]   Fast; Wheel + Horses required
  Horsemen      [2/1/10/1/2/30]   Mobile; Horseback Riding

MEDIEVAL UNITS:
  Knights       [5/2/10/1/2/70]   Heavy cavalry; Chivalry + Horses
  Pikemen       [1/2/10/1/1/30]   Anti-cavalry; Feudalism
  Musketeers    [3/3/20/1/1/60]   Gunpowder upgrade

INDUSTRIAL UNITS:
  Riflemen      [5/5/20/1/1/80]   Industrialization era standard
  Cavalry       [8/3/20/1/3/90]   Mobile strike; Gunpowder + Horses
  Cannon        [8/5/20/1/1/90]   City siege; Metallurgy
  Artillery     [12/2/20/1/1/100] Advanced siege

MODERN UNITS:
  Infantry      [6/6/30/1/1/90]   Mass production era standard
  Armor         [16/8/30/1/3/100] Tank; Automobile tech
  Helicopter    [10/3/20/2/6/100] Air mobile; Combustion
  Stealth Ftr   [8/4/20/2/9/110]  Air superiority; Stealth

NAVAL UNITS:
  Trireme       [1/1/10/1/3/40]   Ancient coastal naval
  Frigate       [4/2/20/1/5/80]   Pre-industrial warship
  Ironclad      [4/4/30/1/4/90]   Steam-powered warship
  Battleship    [12/12/40/2/5/160] Modern capital ship
  Submarine     [12/2/30/2/3/130] Stealthy; anti-ship specialist

CIVILIAN/SUPPORT:
  Settlers      [0/1/10/1/1/40]   Found cities; improve terrain
  Workers       [0/1/10/1/1/30]   Terrain improvements
  Engineers     [0/2/10/1/2/40]   Double-speed improvements
  Caravan       [0/1/10/1/1/50]   Trade routes; wonder production
  Diplomat      [0/0/10/1/2/30]   Diplomacy; espionage
  Spy           [0/0/10/1/3/30]   Advanced espionage
```

### Key Buildings Reference

FreeCiv has **68 building types** in the CivRealm ruleset. Key buildings by category:

**Food & Growth:**

| Building | Tech Required | Effect | Cost/Maint |
|----------|--------------|--------|------------|
| Granary | Pottery | Food storage carries over on growth | 60/1 |
| Supermarket | Refrigeration | +25% food from irrigated tiles | 80/3 |
| Aqueduct | Construction | Allows city to grow beyond size 8 | 80/2 |
| Sewer System | Sanitation | Allows city to grow beyond size 12 | 120/2 |

**Production:**

| Building | Tech Required | Effect | Cost/Maint |
|----------|--------------|--------|------------|
| Barracks | (none) | New military units start as veterans | 40/1 |
| Factory | Industrialization | +50% city shields | 200/4 |
| Manufacturing Plant | Robotics | +50% city shields (stacks with Factory) | 320/6 |
| Power Plant | Electricity | +50% shields (stacks with Factory) | 160/4 |
| Hydro Plant | Electronics | +50% shields, no pollution | 240/4 |
| Nuclear Plant | Nuclear Power | +50% shields, no pollution | 160/2 |

**Science:**

| Building | Tech Required | Effect | Cost/Maint |
|----------|--------------|--------|------------|
| Library | Writing | +50% science output | 80/1 |
| University | University | +50% science output | 160/3 |
| Research Lab | Computers | +50% science output | 160/2 |

**Culture & Happiness:**

| Building | Tech Required | Effect | Cost/Maint |
|----------|--------------|--------|------------|
| Temple | Mysticism | 1 content → happy citizen | 40/1 |
| Colosseum | Construction | 3 unhappy → content | 100/4 |
| Cathedral | Religion | 4 unhappy → content | 120/3 |

**Trade & Economy:**

| Building | Tech Required | Effect | Cost/Maint |
|----------|--------------|--------|------------|
| Marketplace | Currency | +50% luxury; +25% tax revenue | 80/1 |
| Bank | Banking | +50% luxury; +50% tax revenue | 120/2 |
| Stock Exchange | Economics | +50% luxury; +100% tax revenue | 200/4 |
| Port Facility | Seafaring | +50% production for naval units | 80/3 |

### Wonders of the World

Wonders are unique buildings — only one civilization can build each wonder. They provide powerful empire-wide effects:

| Wonder | Tech Required | Effect |
|--------|--------------|--------|
| **Pyramids** | Masonry | Every city acts as if it has a Granary |
| **Great Wall** | Masonry | All enemy units stopped at borders (vs. barbarians); +2 defense |
| **Colossus** | Bronze Working | +1 trade on every trade tile in city |
| **Hanging Gardens** | Pottery | 1 extra content citizen in every city |
| **Oracle** | Mysticism | Temple effects doubled in all cities |
| **Lighthouse** | Map Making | All sea units move +1; Trireme safe everywhere |
| **Sun Tzu's Art of War** | Feudalism | All cities get free Barracks; veterans get promoted faster |
| **Great Library** | Literacy | Get every tech known by 2+ other civs |
| **Marco Polo's Embassy** | Trade | Free embassy with every civ |
| **Magellan's Expedition** | Navigation | All ships move +2 |
| **Copernicus' Observatory** | Astronomy | +50% science in city |
| **Isaac Newton's College** | Theory of Gravity | +100% science in every city |
| **King Richard's Crusade** | Chivalry | +1 shield on every tile worked |
| **Leonardo's Workshop** | Invention | Upgrade 1 unit per turn for free |
| **Shakespeare's Theatre** | Free Artistry | All citizens in city happy |
| **J.S. Bach's Cathedral** | Music Theory | 2 unhappy → content in all cities |
| **Darwin's Voyage** | Railroad | Get 2 free technology advances |
| **Eiffel Tower** | Steam Engine | All AI civs like you better |
| **Statue of Liberty** | Democracy | No anarchy when changing government |
| **Hoover Dam** | Electronics | Hydro Plant in every city (requires river) |
| **Manhattan Project** | Nuclear Fission | All civs can now build nuclear units |
| **Apollo Program** | Space Flight | Space race begins; all civs can build spacecraft |
| **SETI Program** | Computers | +100% science in cities with Research Lab |
| **Cure for Cancer** | Genetic Engineering | +1 happy citizen in every city |
| **United Nations** | Communism | Senate blocks all declarations of war |

---

## CivRealm Framework

CivRealm is an interactive environment published at **ICLR 2024** ([paper](https://arxiv.org/abs/2401.10568), [GitHub](https://github.com/bigai-ai/civrealm)) that wraps FreeCiv-web as a benchmark for decision-making agents.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CivRealm Architecture                        │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  FreeCiv-web │◄──►│    Proxy     │◄──►│     Agent        │  │
│  │   Server     │    │  (CivRealm)  │    │  (LLM or RL)     │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│                            │                                    │
│              ┌─────────────┼─────────────────┐                 │
│              ▼             ▼                 ▼                  │
│       Tensor API     Language API      Mini-Game API            │
│       (RL agents)   (LLM agents)    (simplified tasks)          │
└─────────────────────────────────────────────────────────────────┘
```

The proxy extracts game state from FreeCiv's graphics and network protocol, converts it to structured observations, and submits agent actions back to the server.

### Observation Space (LLM Interface)

For LLM agents, observations are provided as structured text derived from six information sources:

```
Turn N Observation Structure:
┌─────────────────────────────────────────────────────────────┐
│ MAP TILES (5×5 local window around each unit)               │
│   tile(0,0): terrain=grassland, food=2, shields=0, ...      │
│   tile(0,1): terrain=plains, unit=enemy_warrior, ...        │
│   ...                                                       │
├─────────────────────────────────────────────────────────────┤
│ CITY STATUS (per city owned)                                │
│   city_id: 3  name: Rome  pop: 4  food: 12/20              │
│   production: Barracks (8/40)  buildings: [Granary]         │
│   workers: [plains, plains, grassland, forest]              │
├─────────────────────────────────────────────────────────────┤
│ CIVILIZATION STATS                                          │
│   gold: 127  gold/turn: +8  science/turn: 14               │
│   era: Ancient  tech: Bronze_Working (researching)         │
│   government: Monarchy  happiness: 3/5                     │
├─────────────────────────────────────────────────────────────┤
│ MILITARY UNITS (per unit owned)                             │
│   unit_id: 12  type: Warriors  hp: 10/10  moves: 1/1       │
│   location: (14, 22)  veteran: False                        │
├─────────────────────────────────────────────────────────────┤
│ DIPLOMATIC STATE                                            │
│   Persia: WAR  Egypt: PEACE  Greece: CEASE-FIRE (6 turns)  │
├─────────────────────────────────────────────────────────────┤
│ AVAILABLE ACTIONS                                           │
│   [move_unit, build_city, research_tech, end_turn, ...]     │
└─────────────────────────────────────────────────────────────┘
```

**Feature layers for map observation** (full tensor API):

| Layer | Dimensions | Description |
|-------|-----------|-------------|
| `terrain` | M×N | Terrain type per tile |
| `unit_type` | M×N | Unit type present (if any) |
| `unit_owner` | M×N | Owner of unit (player index) |
| `city_id` | M×N | City ID on tile (if city) |
| `city_owner` | M×N | Owner of city |
| `road` | M×N | Road/railroad present |
| `irrigation` | M×N | Irrigation present |
| `mine` | M×N | Mine present |
| `pollution` | M×N | Pollution level |
| `visibility` | M×N | Fog of war state |

### Action Space

The LLM agent issues commands from the following action categories:

| Category | Actions | Example |
|----------|---------|---------|
| **Unit Movement** | Move unit in 8 directions | `move_unit(unit_id=12, dir="north")` |
| **Combat** | Attack enemy unit or city | `attack(unit_id=12, target_id=45)` |
| **City Production** | Queue unit or building | `build_unit(city_id=3, unit="warrior")` |
| **City Building** | Construct improvement | `build_building(city_id=3, building="granary")` |
| **Research** | Set technology goal | `research(tech="iron_working")` |
| **City Founding** | Settler founds city | `found_city(settler_id=7)` |
| **Terrain Work** | Workers/Engineers improve tile | `build_road(worker_id=9)`, `irrigate(worker_id=9)` |
| **Diplomacy** | Interact with other civs | `declare_war(civ="persia")`, `propose_peace(civ="egypt")` |
| **Trade** | Establish trade route | `establish_trade_route(caravan_id=15, city_id=22)` |
| **Government** | Initiate revolution | `change_government(gov="republic")` |
| **Turn End** | End current turn | `end_turn()` |

The action space is **dynamic** — available actions change as the game progresses and depends on which units are alive, which buildings are built, and which technologies have been researched.

---

## DSGBench Evaluation Setup

### Evaluation Scenes

| Scene | Description | Starting Conditions | Key Skill Tested |
|-------|-------------|---------------------|-----------------|
| **Early Expansion** | Maximize territory and city count by Turn 50 | Ancient Era start, 1 city | Exploration, settler timing, city placement |
| **Technology Race** | Reach a target technology tier before AI opponent | Defined era start | Long-horizon planning, research prioritization |
| **Military Conflict** | Win a war against an AI-controlled opponent | Near-equal forces | Tactical unit management, production timing |
| **Diplomatic Challenge** | Maintain peace or secure alliance while gaining points | Multi-civ scenario | Diplomatic reasoning, NL communication |
| **Full Game** | Complete a full game to any victory condition | Standard start | All-around strategic capability |

### Evaluation Metrics

| Metric | Description | Range |
|--------|-------------|-------|
| **Civilization Score** | In-game score at game end (population + territory + tech) | 0 → unbounded |
| **Victory Rate** | Percentage of games won (any victory condition) | 0–100% |
| **Turn Efficiency** | Turns needed to achieve defined milestones | Lower is better |
| **Tech Era Reached** | Highest technology era achieved by Turn N | Ancient → Information |
| **City Count at Turn 50** | Number of cities founded by midgame | Higher is better |
| **Decision Validity Rate** | % of actions that are syntactically and contextually legal | 0–100% |
| **Gold Sustainability** | Whether agent avoids bankruptcy during game | Binary / rate |

---

## Core Challenges for LLM Agents

### 1. Ultra-Long Horizon Planning

Full Civilization games span **200–500 turns**. Decisions made in Turn 10 (e.g., which technology to research first) may not show their consequences until Turn 80. LLMs with fixed context windows cannot easily maintain coherent strategy across an entire game.

```
Planning Horizon Challenge:
  Turn 1:  Choose to research Bronze Working (for Phalanx defense)
                               ↓
  Turn 15: Build Phalanx in first city
                               ↓
  Turn 30: Repel enemy attack (thanks to Turn 1 choice)
                               ↓
  Turn 45: Switch to Iron Working (enabled by Bronze Working)
                               ↓
  Turn 60: Build Legion for offensive campaign

Without explicit long-term planning, an LLM may make myopic
decisions that undermine later strategic options.
```

### 2. Exponentially Expanding State and Action Space

| Game Stage | State Space | Action Space |
|-----------|-------------|--------------|
| Turn 1 (1 city, 2 units) | ~10¹⁵ | ~10⁴ |
| Turn 50 (4 cities, 20 units) | ~10⁴⁰ | ~10³⁰ |
| Turn 200 (15 cities, 80 units) | ~10¹⁵⁰ | ~10¹⁰⁰ |
| Late game (30+ cities) | ~10⁶⁵⁰ | ~10¹⁶⁶ |

Source: CivRealm (ICLR 2024)

### 3. Multi-Objective Trade-offs

Every turn requires balancing at least 5 competing priorities simultaneously:

```
Resource Allocation Decision Tree (per turn):

Should this city build a Barracks or a Library?
  ├─ Barracks → better military production for 10+ turns
  │    └─ But delays science by ~15 turns...
  └─ Library → +50% science now and every turn after
       └─ But vulnerable to attack while building...

Should I research Monarchy or Iron Working next?
  ├─ Monarchy → better economy, less corruption
  │    └─ But no new offensive units...
  └─ Iron Working → Legion unit for aggressive expansion
       └─ But remaining under Despotism longer...
```

### 4. Fog of War and Partial Observability

The LLM agent only sees tiles that have been explored by its own units. Enemy civilizations' positions, military strength, and diplomatic intentions are all hidden until scouted. Decisions must be made under uncertainty:

- How many cities does the enemy have?
- Are they building an army or focusing on science?
- Is the unexplored territory worth settling?

### 5. Diplomatic Natural Language Interaction

FreeCiv allows **natural language chat** between civilizations. In LLM evaluation scenarios with multiple LLM agents or human opponents, the agent must:
- Interpret natural language threats, offers, and proposals
- Formulate persuasive diplomatic messages
- Detect deception or broken promises
- Balance short-term treaty benefits against long-term trust

### 6. Technology Tree Navigation

With 87 technologies and complex prerequisite chains, the LLM must decide a research path that supports its chosen strategy. There is no single optimal path — it depends on:
- Map resources available (Horses? Iron? Coal?)
- Chosen victory condition (military vs. space race)
- Opponent strategies and threat level
- Starting position and terrain

---

## Strategic Concepts for Beginners

### Early Game Priority Order

A commonly effective early game sequence:

```
Turn 1–5:   Scout with starting unit; identify nearby resources
Turn 3–8:   Build Settlers in starting city
Turn 5–12:  Found second city on best available tile
Turn 8–15:  Start building Workers for terrain improvements
Turn 10–20: Research Pottery (for Granary) or Bronze Working (for Phalanx)
Turn 15–25: Connect cities with roads (enables resource trade)
Turn 20–30: Build 2–3 military units for defense
Turn 25–40: Found 3rd and 4th cities; claim key resource tiles
Turn 40–60: Switch to internal development (Libraries, Markets)
```

### City Placement Strategies

**Wide Expansion (ICS — Infinite City Sprawl)**

- Plant cities close together (3–4 tiles apart)
- Maximizes total population and unit support
- Each city may only grow to size 4–6 due to tile overlap
- Strong early military production
- Risk: high maintenance costs; corruption if far from capital

**Tall Development (OCP — Optimal City Placement)**

- Cities spaced 6+ tiles apart; no tile overlap
- Each city can grow large (size 10–15)
- Fewer cities but each is highly productive
- Stronger science and culture output
- Risk: slower territorial expansion; opponents may claim land

**Hybrid Approach (Recommended for LLM agents)**

```
Phase 1 (Turn 1–30):  Wide expansion to claim territory
Phase 2 (Turn 30–80): Consolidate; build infrastructure
Phase 3 (Turn 80+):   Tall development; maximize output per city
```

### Technology Research Paths

| Strategy Focus | Recommended Early Techs | Reasoning |
|---------------|------------------------|-----------|
| **Military Aggression** | Bronze Working → Iron Working → Warrior Code → Chivalry | Legion/Knights for early warfare |
| **Economic Growth** | Pottery → Monarchy → Trade → Currency | Granary + Marketplace for fast growth |
| **Science Race** | Alphabet → Writing → Literature → Republic | Library → fast science curve |
| **Cultural Victory** | Ceremonial Burial → Mysticism → Philosophy | Temples + Philosophy for culture |
| **Defensive Turtle** | Masonry → Bronze Working → City Walls | Defense while building internally |

### Culture vs. Military vs. Science Balance

```
The Strategic Triangle:

        Science
           /\
          /  \
         /    \
        /      \
       /        \
  Culture ────── Military

- Going pure Science: fast tech, but vulnerable to attack
- Going pure Military: conquest, but fall behind in tech
- Going pure Culture: happy, but need military protection
- Balanced approach: research to unlock military techs,
  maintain small defensive force, build libraries early
```

### Diplomatic Strategy

Key diplomatic principles for LLM agents:

1. **Meet everyone early** — You can only negotiate with civilizations you've encountered
2. **Build embassies** — Reveals their gold, tech, and government type; enables trading
3. **Respect cease-fires** — Breaking treaties costs reputation and may trigger Senate revolt
4. **Use peace strategically** — Peace with distant civs frees military for nearby threats
5. **Trade technologies** — Trading a known tech for an unknown one is always profitable
6. **Beware alliances** — Alliance means you must join your ally's wars

---

## RL Training Considerations

### Reward Signal Design

For RL training in the Civilization environment, reward signal design is challenging due to sparse feedback and long game horizons:

| Reward Type | Source | Frequency | Notes |
|------------|--------|-----------|-------|
| **Terminal Reward** | Win/loss/score at game end | Once | Extremely sparse over 200–500 turns |
| **Score Delta** | Change in civilization score per turn | Every turn | Denser; but noisy and non-monotonic |
| **Milestone Reward** | Tech advancement, city founding | Event-triggered | Shaped reward; better learning signal |
| **Efficiency Reward** | Gold/science efficiency per turn | Every turn | Encourages resource optimization |
| **Action Validity Reward** | Penalize invalid/illegal actions | Every step | Encourages interface mastery |

**Recommended Reward Architecture for DSGBench++:**

```
R_total = α·R_terminal + β·R_score_delta + γ·R_milestone + δ·R_validity

Suggested weights:
  α = 1.0   (win/loss signal, highest weight)
  β = 0.3   (per-turn score progress)
  γ = 0.5   (tech/city milestones)
  δ = 0.1   (action legality)
```

### High-Quality Trajectory Characteristics

For SFT or offline RL, a high-quality Civilization trajectory should exhibit:

1. **Consistent research chain** — Technologies follow a logical prerequisite path without random switching
2. **City production coherence** — Production queue aligns with strategic goals (building Library during science race, not Colosseum)
3. **Proper worker management** — Workers actively building roads/farms rather than sitting idle
4. **Proactive scouting** — Settlers sent with military escort; explorers covering unexplored territory
5. **Diplomatic consistency** — Maintain treaty states; avoid reputation loss from broken promises
6. **Non-bankruptcy management** — Gold balance never hits negative; upkeep costs managed
7. **Action validity** — All issued commands are legal given current game state

### Key Capability Dimensions

| Dimension | Measurement | Baseline (Random) | Strong LLM |
|-----------|-------------|-------------------|------------|
| **Strategic Planning** | Tech era reached by Turn 100 | Medieval | Industrial |
| **Resource Management** | Gold/turn efficiency | Deficit/neutral | Consistent surplus |
| **Military Timing** | Win rate in combat scenarios | ~5% | 30–50% |
| **Diplomatic Reasoning** | Treaty compliance rate | ~40% | 80–90% |
| **City Development** | Avg city pop at Turn 80 | 2–3 | 5–8 |
| **Decision Validity** | % legal actions issued | 50–60% | 85–95% |

### Research Findings (CivRealm ICLR 2024)

Key findings from the CivRealm paper:

- **RL agents** show reasonable performance in isolated mini-games (battle, development, diplomacy) but **fail to make substantial progress in the full game**
- **LLM agents (BaseLang)** struggle with global coordination — each unit has its own LLM with local context, leading to **lack of global perspective** and sluggish expansion
- **Mastaba (hierarchical LLM)** outperforms BaseLang by introducing an "advisor" LLM that maintains empire-wide state and sends strategy guidance to unit-level LLMs
- **Context window limits** are a fundamental bottleneck — as city/unit counts grow, fitting all relevant state into LLM context becomes impossible
- The full Civilization game **remains largely unsolved** for both RL and LLM agents as of 2024

---

## Setup and Dependencies

### Quick Setup

```bash
# 1. Pull and start the FreeCiv web service
docker pull akon123456/freeciv-web
docker tag akon123456/freeciv-web:latest freeciv/freeciv-web:latest

# 2. Activate DSGBench environment and start FreeCiv
conda activate dsgbench
start_freeciv_web_service

# 3. Install CivRealm
pip install civrealm

# 4. Run a test game
python -c "
import civrealm
env = civrealm.make('freeciv-v0')
obs, info = env.reset()
done = False
while not done:
    action = env.action_space.sample()
    obs, reward, done, truncated, info = env.step(action)
env.close()
"
```

See [Quick Start](../../getting_started/quickstart.md) for full setup instructions.

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `FREECIV_HOST` | FreeCiv server hostname | `localhost` |
| `FREECIV_PORT` | FreeCiv server port | `6000` |
| `CIVREALM_MAX_TURNS` | Maximum turns per episode | `500` |
| `CIVREALM_RULESET` | Ruleset to use | `classic` |

---

## Resources and References

### Papers

- **CivRealm (ICLR 2024)** — [arxiv.org/abs/2401.10568](https://arxiv.org/abs/2401.10568) — Core paper on the FreeCiv LLM/RL evaluation framework
- **DSGBench** — [arxiv.org/abs/2503.06047](https://arxiv.org/abs/2503.06047) — DSGBench benchmark paper including Civilization environment
- **Digital Player (2025)** — [arxiv.org/html/2502.20807v1](https://arxiv.org/html/2502.20807v1) — LLM-based human-like agent evaluation in games
- **Vox Deorum** — Hybrid LLM architecture for 4X/Grand Strategy (Civilization V)

### Code and Tools

- [FreeCiv Official Site](https://www.freeciv.org/) — Game downloads and documentation
- [FreeCiv GitHub](https://github.com/freeciv/freeciv) — Source code (GPL)
- [CivRealm GitHub](https://github.com/bigai-ai/civrealm) — LLM/RL evaluation framework
- [FreeCiv-web](https://www.freecivweb.com/) — Browser-playable version
- [FreeCiv Wiki — Technology](https://freeciv.fandom.com/wiki/Technology) — Full technology reference
- [FreeCiv Wiki — Units Catalogue](https://freeciv.fandom.com/wiki/Units_Catalogue) — Full unit stats
- [FreeCiv Wiki — Buildings](https://freeciv.fandom.com/wiki/Buildings) — Full buildings reference
- [FreeCiv Wiki — Wonders](https://freeciv.fandom.com/wiki/Wonders) — Full wonders reference
- [FreeCiv Wiki — Diplomacy](https://freeciv.fandom.com/wiki/Diplomacy) — Diplomatic system details

### Image / Media Resources

FreeCiv is licensed under the **GNU GPL-2.0-or-later**, which means screenshots and derived materials can generally be used under the same license. The FreeCiv project provides:
- In-game tileset graphics (GPL licensed)
- [FreeCiv Art project](https://freeciv.fandom.com/wiki/The_Art_of_Freeciv) — community artwork
- FreeCiv-web screenshots (accessible via browser at freecivweb.com)
