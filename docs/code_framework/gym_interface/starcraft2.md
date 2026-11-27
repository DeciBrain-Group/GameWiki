# StarCraft II — Interface Specification

<div class="badge-row"><span class="badge badge-realtime">Real-time</span></div>

Engine: **python-sc2 (BurnySC2)**, via the TextStarCraft2 wrapper built on the PySC2 protocol. Source: `DSGBench/games/starcraft2/` and `DSGBench/agent_manager/`. For game mechanics see the [StarCraft II game page](../../environments/starcraft2/starcraft2.md). StarCraft II itself is Blizzard Entertainment's free-to-play real-time strategy game (starcraft2.com); the evaluation map **Ancient Cistern LE** is a 1v1 ladder map (`LE` = Ladder Edition) from Blizzard's official map pool.

## Evaluation scenes

All six scenes use the same map (Ancient Cistern LE), the same matchup (Protoss vs Protoss, `players: Protoss_VS_Protoss`), and the same opponent (the built-in AI at **medium** difficulty, `game_difficulty: medium`). The scenes differ only in two switches: which strategy the built-in AI plays, and whether the game waits for the LLM. All six combinations are defined in `configs/eval_config_base/eval_starcraft2_agent_vs_computer_scene1-6.yaml`:

| Scene | AI strategy (`game_ai_build`) | Timing (`asynch_mode`) | Matches per model |
|-------|-------------------------------|------------------------|-------------------|
| 1 | macro | synchronous (`False`) | 5 |
| 2 | macro | asynchronous (`True`) | 5 |
| 3 | rush | synchronous (`False`) | 5 |
| 4 | rush | asynchronous (`True`) | 5 |
| 5 | randombuild | synchronous (`False`) | 5 |
| 6 | randombuild | asynchronous (`True`) | 5 |

- **AI strategy**: `macro` plays a long economic game, `rush` attacks early, `randombuild` mixes builds unpredictably.
- **Synchronous** (`asynch_mode: False`): the game loop pauses and waits until the LLM has answered before stepping forward.
- **Asynchronous** (`asynch_mode: True`): the game keeps running in real time while the LLM is thinking, so an answer that arrives late applies to a situation that has already moved on.

## Decision cadence

The agent does not call the LLM every game step. It uses two levels of summarization with a fixed decision interval; all parameters below are set in the agent constructor (`agent_manager/agents/starcraft2_agent/gpt4_agent.py:24-60`).

1. **Every game step**, the bot's structured observation (from `get_information()`) is rendered into one short text block called the **L1 summary** (`summarize/L1_summarize.py`, decorated as a `@weave.op`) and appended to a summary queue.
2. **Every 10 steps** (`action_interval=10`), if the queue is non-empty, the agent builds the **L2 prompt**. The prompt contains the last **5** L1 summaries (`last_k=5`, `chunk_window=5`), the list of actions executed since the last call (with `EMPTY ACTION` removed), and the list of failed actions together with their failure reasons (`preprocess_actions()`).
3. **The evaluated model itself acts as the commander**: the summarizer is created with the same model that is being tested (`self.L2 = L2_summary(llm_model=self.llm_model, …)`, `gpt4_agent.py:46`), so one LLM both plays the game and analyses the situation. It returns one structured text answer. Up to **10** decisions are extracted from it (`action_window=10`) and pushed into the action queue. While pushing, each slot is replaced by `EMPTY ACTION` with probability **0.5** (`action_mix_rate=0.5`), so on average half of the queued slots do nothing.
4. **Every game step**, one action ID is popped from the front of the queue (`_get_next_action()`). If the queue is empty, the step executes `EMPTY ACTION` (the last ID in the action dictionary).
5. **Grounding counters** are kept per match: every parsed decision that is not `EMPTY ACTION` increments `valid_action_num` (line 217). At match end, `success_exe_action_num = valid_action_num − len(action_failures)` (line 248), and the per-match grounding accuracy is `match_grounding_acc = success_exe_action_num / max(valid_action_num, 1)` (line 268).

## Raw observation space

At every game step, the engine gives the bot a complete snapshot of the game through the python-sc2 (BurnySC2) library. The snapshot has three layers: **player-level numbers** (resources and supply), **global game data** (time, score, fog of war), and **one object per unit**. The wrapper bot reads only a small part of it — the rest is available but unused.

### 1. Player-level numbers

The engine reports the player's resources and supply as a group of plain numbers (the `Common` class, `sc2/game_state.py:72-90`). python-sc2 also copies each number onto the bot object as a shortcut attribute, so the bot code can read `self.minerals` instead of the longer original name. The mapping:

| Engine field | Bot attribute | Meaning |
|--------------|---------------|---------|
| `player_id` | — | ID of this player |
| `minerals` | `self.minerals` | Unspent minerals |
| `vespene` | `self.vespene` | Unspent vespene gas |
| `food_cap` | `self.supply_cap` | Supply cap |
| `food_used` | `self.supply_used` | Supply currently used |
| `food_army` | `self.supply_army` | Supply taken by army units |
| `food_workers` | `self.supply_workers` | Supply taken by workers |
| — | `self.supply_left` | Free supply (cap − used) |
| `idle_worker_count` | — | Workers standing idle |
| `army_count` | — | Number of army units |
| `warp_gate_count` | — | Warp gates ready (Protoss) |
| `larva_count` | — | Larva available (Zerg) |

### 2. Global game data

The snapshot's top-level object (`GameState`, `sc2/game_state.py:198-235`) also carries:

| Field | Content |
|-------|---------|
| `game_loop` | Frame counter, 22.4 frames per second on Faster speed; the bot converts it to seconds (`self.time`) and to a `"MM:SS"` string (`self.time_formatted`) |
| `score` | Detailed score statistics — see section 3 below |
| `upgrades` | Set of technologies already completed |
| `visibility` | One value per map tile: `0` hidden, `1` explored but in fog, `2` currently visible |
| `creep` | One value per map tile: whether Zerg creep covers it |
| `effects` | Active area effects on the ground (for example a Ravager's bile shot) with their positions |
| `psionic_matrix` | Map areas powered by Pylons or Warp Prisms (where Protoss can warp in) |
| `dead_units` | IDs of units destroyed this frame |
| `alerts`, `chat`, `player_result` | Game alerts, chat messages, and the end-of-game result |
| `abilities` | Abilities of the currently selected units |

### 3. Score statistics

The engine keeps a detailed score sheet (`ScoreDetails`, `sc2/score.py:24-60`). It contains the current score; idle production time and idle worker time; the total value of units and structures built; the value of what the player killed; minerals and vespene collected and spent, with collection rates; and per-category breakdowns of supply used, value killed, and value lost (each split into army, economy, technology, and upgrade).

The Protoss bot reads only five of these — `collected_minerals`, `spent_minerals`, `collected_vespene`, `spent_vespene`, and `score` (`Protoss_bot.py:459-464`) — and passes them into the wrapped observation.

### 4. One object per unit

Every unit and structure in the game — the player's own, the visible enemies, mineral fields, and vespene geysers — is delivered as a `Unit` object (`sc2/unit.py`). The bot reaches them through ready-made collections such as `self.units` (own army and workers), `self.structures` (own buildings), `self.enemy_units` and `self.enemy_structures` (visible enemies), `self.mineral_field`, `self.vespene_geyser`, and `self.townhalls`. Each `Unit` object exposes:

| Group | What it tells you |
|-------|-------------------|
| Identity | Unit type, name, a unique tag, race, whether it is a structure, and whether it belongs to the player, an ally, or an enemy |
| Condition | Health, shields, and energy (current and maximum); construction progress; whether it is finished, flying, burrowed, powered, active, cloaked, or revealed; active buffs; whether it is a hallucination |
| Position | 2D and 3D map position, facing direction, collision radius, distance helpers to other units or points |
| Orders | The command queue currently running on the unit — a list of `UnitOrder` entries, each with the ability, the target, and the progress (`sc2/unit.py:1025-1028`) |
| Vision | Whether the unit is currently visible, only remembered from last sighting, or a sensor-tower blip; its sight and detection ranges |
| Combat statistics | Whether it can attack and against which targets (ground, air, or both); ground and air DPS and range; armor; weapon / armor / shield upgrade levels; movement speed; damage calculation against a given target |
| Resources | Remaining minerals or gas inside a mineral field or geyser; whether a worker is carrying minerals or gas |

**Fog of war**: enemy units and structures are delivered only while their tile is visible. Once they leave sight, they either vanish from the collections or remain as "snapshot" memory objects that keep the health and position from the last sighting. The possible enemy spawn points are known from the start through `enemy_start_locations`.

### 5. Helper queries on the bot

On top of the raw snapshot, python-sc2 provides ready-made queries that the DSGBench wrapper uses (`sc2/bot_ai.py`): count units or buildings still in production (`already_pending`), check research in progress (`already_pending_upgrade`, line 721), check affordability (`can_afford`, line 497) and costs (`calculate_cost`, `calculate_supply_cost`), filter own units and structures by type, get the start location and expansion locations, list the abilities a group of units can currently cast (`get_available_abilities`, line 199), and test whether a building fits at a position (`can_place`, line 614).

## Raw action space

The engine accepts commands at the level of individual units. Every command is a `UnitCommand` (`sc2/unit_command.py:14-38`) made of four parts:

| Part | What it holds |
|------|---------------|
| Ability | What to do — an `AbilityId` constant such as `NEXUSTRAIN_PROBE` (train a Probe), `PYLONBUILD` (warp in a Pylon), `MORPH_ARCHON` (merge two Templar into an Archon), `RESEARCH_WARPGATE` (research Warp Gate), or `EFFECT_CHRONOBOOSTENERGYCOST` (Chrono Boost). These are real examples the Protoss bot issues (`Protoss_bot.py:1003-1004, 1399-1405, 1763`) |
| Unit | Who does it — one issuing unit, referenced by its unique tag |
| Target | Where or on what — optional; either a map coordinate or another unit (a mineral field to gather from, an enemy to attack, a building to place) |
| Queue flag | Whether the command waits in the unit's order queue (`True`) or replaces its current order (`False`) |

A command is created by calling the unit like a function — `unit(ability, target)` (`sc2/unit.py:1452`) — collected by the bot's `do()` method, and sent to the game by `client.actions()` (`sc2/client.py:188`), which packs each command into a protobuf `ActionRawUnitCommand`. Two sibling command types exist: toggling auto-cast on or off per ability (`ActionRawToggleAutocast`, `client.py:352`) and moving the camera (`ActionRawCameraMove`, `client.py:416`).

For convenience, python-sc2 adds higher-level methods that still decompose into unit commands: `build()` (`bot_ai.py:894`), `train()` (`bot_ai.py:937`), `research()` (`bot_ai.py:1080`), `expand_now()`, and `distribute_workers()`.

The LLM never sees this space directly. The reason is scale: an action here is a triple of *ability × unit tag × target*, so a single step offers hundreds of distinct choices (the game page quotes "approximately 300+ discrete actions per step, each with multiple argument types"), and a language model cannot reliably produce unit tags or map coordinates. The macro-action wrapper below collapses all of this into 72 named decisions.

## Wrapped observation space

What the agent actually receives from `env.step()` is much smaller than the raw snapshot. The bot's `get_information()` method compresses the game into five dictionaries, `get_process_data()` adds twelve counters, and the environment appends the execution results of the previous step (`action_executed` and `action_failures`). The exact keys below are from the **Protoss** bot (`Protoss_bot.py:438-700`). A parallel **Zerg implementation** exists with the same structure but Zerg unit, building, and research keys (`bot/Zerg_bot.py`: `get_information` at line 98, `get_process_data` at line 280, `handle_action_0` through `handle_action_69`).

### The five information dictionaries

**1. Resources** (`information.resource`) — the player-level numbers from the raw snapshot, renamed:

| Key | Meaning |
|-----|---------|
| `game_time` | Game clock as `"MM:SS"` |
| `worker_supply` | Supply taken by workers |
| `mineral` / `gas` | Unspent minerals / vespene |
| `supply_left` / `supply_cap` / `supply_used` | Free supply, supply cap, supply used |
| `army_supply` | Supply taken by army units |

**2. Buildings** (`information.building`) — one count per building type, 16 keys in total:

| Key | Building |
|-----|----------|
| `nexus_count` | Nexus (main base) |
| `pylon_count` | Pylon |
| `gas_buildings_count` | Assimilator |
| `gateway_count` | Gateway |
| `forge_count` | Forge |
| `photon_cannon_count` | Photon Cannon |
| `shield_battery_count` | Shield Battery |
| `warp_gate_count` | Warp Gate |
| `cybernetics_core_count` | Cybernetics Core |
| `twilight_council_count` | Twilight Council |
| `robotics_facility_count` | Robotics Facility |
| `statgate_count` | Stargate (the key is misspelled in the code) |
| `templar_archives_count` | Templar Archives |
| `dark_shrine_count` | Dark Shrine |
| `robotics_bay_count` | Robotics Bay |
| `fleet_beacon_count` | Fleet Beacon |

**3. Units** (`information.unit`) — one count per own unit type, 19 keys in total:

| Key | Unit |
|-----|------|
| `probe_count` | Probe |
| `Zealot_count` | Zealot |
| `stalker_count` | Stalker |
| `sentry_count` | Sentry |
| `adept_count` | Adept |
| `high_templar_count` | High Templar |
| `dark_templar_count` | Dark Templar |
| `immortal_count` | Immortal |
| `colossus_count` | Colossus |
| `disruptor_count` | Disruptor |
| `archon_count` | Archon |
| `observer_count` | Observer |
| `warp_prism_count` | Warp Prism |
| `phoenix_count` | Phoenix |
| `voidray_count` | Void Ray |
| `Oracle_count` | Oracle |
| `Carrier_count` | Carrier |
| `tempest_count` | Tempest |
| `mothership_count` | Mothership |

**4. Planning** (`information.planning`) — what is already under construction or in production, split into two sub-dictionaries. The numbers come from python-sc2's `already_pending` query, so they count units and buildings whose production has started but not finished:

- `planning_structure` — pending buildings, one key per building type, 16 keys: `planning_nexus_count`, `planning_pylon_count`, `planning_gas_buildings_count`, `planning_gateway_count`, `planning_forge_count`, `planning_photon_cannon_count`, `planning_shield_battery_count`, `planning_warp_gate_count`, `planning_cybernetics_core_count`, `planning_twilight_council_count`, `planning_robotics_facility_count`, `planning_stargate_count`, `planning_templar_archives_count`, `planning_dark_shrine_count`, `planning_robotics_bay_count`, `planning_fleet_beacon_count`.
- `planning_unit` — pending units, one key per unit type, 19 keys: `planning_probe_count`, `planning_Zealot_count`, `planning_stalker_count`, `planning_sentry_count`, `planning_adept_count`, `planning_high_templar_count`, `planning_dark_templar_count`, `planning_immortal_count`, `planning_colossus_count`, `planning_disruptor_count`, `planning_archon_count`, `planning_observer_count`, `planning_warp_prism_count`, `planning_phoenix_count`, `planning_voidray_count`, `planning_Oracle_count`, `planning_Carrier_count`, `planning_tempest_count`, `planning_mothership_count`.

**5. Research** (`information.research`) — the status of every Protoss upgrade, grouped by the building that researches it. Each status is a number from python-sc2's `already_pending_upgrade`: `0` means not started, a value between `0` and `1` means in progress, `1` means completed.

| Research building | Status keys |
|-------------------|-------------|
| `cybernetics_core` | `warpgate_research_status`, `protoss_air_weapon_level_1/2/3_research_status`, `protoss_air_armor_level_1/2/3_research_status` |
| `forge` | `protoss_ground_weapon_level_1/2/3_research_status`, `protoss_ground_armor_level_1/2/3_research_status`, `protoss_shield_level_1/2/3_research_status` |
| `twilight_council` | `blink_research_status`, `charge_research_status`, `resonating_glaives_research_status` |
| `robotics_bay` | `extended_thermal_lance_research_status`, `GRAVITICDRIVE_research_status`, `OBSERVERGRAVITICBOOSTER_research_status` |

**Enemy sightings** (`information.enemy`) — the unit types and building types of the enemy seen so far, with counts. The keys look like `enemy_UnitTypeId.ZERGLING: 1`. They are produced by `get_enemy_unity()` and `get_enemy_structure()`.

### Execution results and process data

**`action_executed` and `action_failures`** — what happened to the previous step's macro actions. `action_executed` lists the actions that ran; `action_failures` lists the actions that could not run, each with the bot's reason (for example not enough minerals, or the required building is missing).

**`process_data`** — twelve counters used for metrics (`Protoss_bot.py:451-468`):

| Key | Meaning |
|-----|---------|
| `iteration` | Game step counter |
| `time` | Game time in seconds |
| `supply_cap` / `supply_used` | Supply cap and supply used |
| `minerals` / `vespene` | Unspent minerals / vespene |
| `collected_minerals` / `spent_minerals` | Minerals collected so far / spent so far |
| `collected_vespene` / `spent_vespene` | Vespene collected so far / spent so far |
| `score` | The engine's score value |
| `completed_tech` | Number of completed technologies: building types owned (counted once each) plus finished researches (`Protoss_bot.py:465`) |

### How this becomes text

The L1 summarizer turns each dictionary into `- key: value` lines and **skips every key whose value is zero**, under the section headers `Resources:`, `Buildings:`, `Units:`, `Planning:`, `Research:`, `Enemy:` (`L1_summarize.py`). This is the text the LLM actually sees — an empty category simply does not appear.

## Prompt structure

**System** prompt (`agent_manager/prompts/starcraft2_prompt.py`, class `StarCraftIIPrompt_V4`, `K=5`), verbatim except that the full action dictionary is abbreviated in the last paragraph:

??? example "System prompt (full text, race = Protoss)"
    ```text
    You are an AI trained in analyzing and summarizing StarCraft II games. You understand the nuances and strategies of the protoss race.
    Based on the summaries of multiple rounds in a game, we want you to analyze the game progression in a structured way. Your analysis should include the following aspects:
    1. Game Overview: Provide a brief overview of the current situation based on all the rounds.
    2. Current Game Stage: Determine the stage of the game based on the information of all rounds. Is it the early game, mid-game, or late game?
    3. Our Situation: Describe our current status in terms of:
        3.1 Units and Buildings: Analyze the state of our units and buildings.
        3.2 Economy: Evaluate our economic condition, including resource collection and usage.
        3.3 Technology: Describe the status of our technological research and what technologies we have unlocked so far. Analyze our technology tree, indicating the available and potential upgrades or units.
    4. Our Strategy: Infer our potential strategy based on our current situation and the information of all rounds.
    5. Enemy's Strategy: Infer the enemy's potential strategy, based on the available information.
    6. Key Information: Highlight the most important aspects from all rounds that have significantly influenced the game.

    For Protoss, keep an eye on Nexus's energy to Chrono Boost important structures.

    Based on the game situation and strategies used by both sides, provide specific suggestions for the following areas:

    1.Our Strategy: Propose adjustments to our current strategy to counter the enemy's moves and capitalize on our strengths.

    2.Units and Buildings: Offer ways to enhance our unit composition and improve our building layout, suited to the current stage of the game.

    3.Economy: Recommend better practices for resource gathering and usage, in line with our strategic needs.

    4.Technology: Suggest focused research paths to gain technological advantages, considering our current research status and technology tree.

    Lastly, consider the current situation and the suggestions provided, make 5 actionable and specific decisions from the action dictionary
    {'TRAIN UNIT': {0: 'TRAIN PROBE', …}, 'BUILD STRUCTURE': {19: 'BUILD PYLON', …}, 'RESEARCH TECHNIQUE': {34: …}, 'OTHER ACTION': {60: 'SCOUTING PROBE', …, 71: 'EMPTY ACTION'}}.
    This dictionary comprises four categories of actions: unit production, building construction, technology research, and other actions. Remember to align these decisions with the current stage of the game, and avoid proposing actions that are not currently feasible.
    ```

The middle line is the only race-dependent sentence. It is selected by race (`starcraft2_prompt.py:9-11`):

??? example "Race-specific prompt lines"
    ```text
    protoss: "For Protoss, keep an eye on Nexus's energy to Chrono Boost important structures."
    zerg:    "For Zerg, pay attention to whether there are enough larvae. If not, we should consider
              adding the INJECTLARVA command to the queue."
    ```

**User** prompt — this is the assembled L2 input described in *Decision cadence*. It has three parts, in order:

1. Five observation chunks, each introduced by `chunk0: At … game time, our current StarCraft II situation is as follows: …` (these are the last 5 L1 summaries).
2. The `executed_actions` list — the non-empty actions parsed since the last LLM call.
3. The `failed_actions` list — the macro actions that could not be executed, with reasons.

The prompt class also embeds one race-specific worked example (few-shot). For Protoss, the example input is five chunks covering game time 06:48 to 06:52 (worker supply 63, mineral 2480, gas 2972, supply 87/99, 5 bases, 9 gas buildings, 4 gateways, 7 Zealots, one enemy Zergling sighted), and the example answer covers all six analysis sections, the four suggestion areas, and a numbered decision list. The Zerg example adds one extra analysis point: `5. Larvae: Monitor the number of available larvae. If it's too low, queue the INJECTLARVA ability to increase the larva production.`

## Wrapped action space — `Discrete(72)` Protoss / `Discrete(70)` Zerg

The LLM chooses from a fixed dictionary of named macro actions. The complete Protoss table has 72 actions (`init_protoss_actions`, `games/starcraft2/utils/action_info.py:25-69`):

| ID | Action | ID | Action | ID | Action |
|----|--------|----|--------|----|--------|
| 0 | TRAIN PROBE | 24 | BUILD FORGE | 48 | RESEARCH PROTOSSGROUNDARMORSLEVEL2 |
| 1 | TRAIN ZEALOT | 25 | BUILD TWILIGHTCOUNCIL | 49 | RESEARCH PROTOSSGROUNDARMORSLEVEL3 |
| 2 | TRAIN ADEPT | 26 | BUILD ROBOTICSFACILITY | 50 | RESEARCH PROTOSSSHIELDSLEVEL1 |
| 3 | TRAIN STALKER | 27 | BUILD STARGATE | 51 | RESEARCH PROTOSSSHIELDSLEVEL2 |
| 4 | TRAIN SENTRY | 28 | BUILD TEMPLARARCHIVE | 52 | RESEARCH PROTOSSSHIELDSLEVEL3 |
| 5 | TRAIN HIGHTEMPLAR | 29 | BUILD DARKSHRINE | 53 | RESEARCH EXTENDEDTHERMALLANCE |
| 6 | TRAIN DARKTEMPLAR | 30 | BUILD ROBOTICSBAY | 54 | RESEARCH GRAVITICDRIVE |
| 7 | TRAIN VOIDRAY | 31 | BUILD FLEETBEACON | 55 | RESEARCH OBSERVERGRAVITICBOOSTER |
| 8 | TRAIN CARRIER | 32 | BUILD PHOTONCANNON | 56 | RESEARCH PSISTORMTECH |
| 9 | TRAIN TEMPEST | 33 | BUILD SHIELDBATTERY | 57 | RESEARCH VOIDRAYSPEEDUPGRADE |
| 10 | TRAIN ORACLE | 34 | RESEARCH WARPGATERESEARCH | 58 | RESEARCH PHOENIXRANGEUPGRADE |
| 11 | TRAIN PHOENIX | 35 | RESEARCH PROTOSSAIRWEAPONSLEVEL1 | 59 | RESEARCH TEMPESTGROUNDATTACKUPGRADE |
| 12 | TRAIN MOTHERSHIP | 36 | RESEARCH PROTOSSAIRWEAPONSLEVEL2 | 60 | SCOUTING PROBE |
| 13 | TRAIN OBSERVER | 37 | RESEARCH PROTOSSAIRWEAPONSLEVEL3 | 61 | SCOUTING OBSERVER |
| 14 | TRAIN IMMORTAL | 38 | RESEARCH PROTOSSAIRARMORSLEVEL1 | 62 | SCOUTING ZEALOT |
| 15 | TRAIN WARPPRISM | 39 | RESEARCH PROTOSSAIRARMORSLEVEL2 | 63 | SCOUTING PHOENIX |
| 16 | TRAIN COLOSSUS | 40 | RESEARCH PROTOSSAIRARMORSLEVEL3 | 64 | MULTI-ATTACK |
| 17 | TRAIN DISRUPTOR | 41 | RESEARCH ADEPTPIERCINGATTACK | 65 | MULTI-RETREAT |
| 18 | MORPH ARCHON | 42 | RESEARCH BLINKTECH | 66 | CHRONOBOOST NEXUS |
| 19 | BUILD PYLON | 43 | RESEARCH CHARGE | 67 | CHRONOBOOST CYBERNETICSCORE |
| 20 | BUILD ASSIMILATOR | 44 | RESEARCH PROTOSSGROUNDWEAPONSLEVEL1 | 68 | CHRONOBOOST TWILIGHTCOUNCIL |
| 21 | BUILD NEXUS | 45 | RESEARCH PROTOSSGROUNDWEAPONSLEVEL2 | 69 | CHRONOBOOST STARGATE |
| 22 | BUILD GATEWAY | 46 | RESEARCH PROTOSSGROUNDWEAPONSLEVEL3 | 70 | CHRONOBOOST FORGE |
| 23 | BUILD CYBERNETICSCORE | 47 | RESEARCH PROTOSSGROUNDARMORSLEVEL1 | 71 | EMPTY ACTION |

The complete Zerg table has 70 actions (`init_zerg_actions`, `action_info.py:71-97`):

| ID | Action | ID | Action | ID | Action |
|----|--------|----|--------|----|--------|
| 0 | TRAIN OVERLORD | 23 | BUILD LAIR | 46 | RESEARCH ZERGMISSILEWEAPONSLEVEL2 |
| 1 | TRAIN DRONE | 24 | BUILD HIVE | 47 | RESEARCH ZERGMISSILEWEAPONSLEVEL3 |
| 2 | TRAIN QUEEN | 25 | BUILD EVOLUTIONCHAMBER | 48 | RESEARCH ZERGGROUNDARMORSLEVEL1 |
| 3 | TRAIN ZERGLING | 26 | BUILD INFESTATIONPIT | 49 | RESEARCH ZERGGROUNDARMORSLEVEL2 |
| 4 | TRAIN ROACH | 27 | BUILD SPIRE | 50 | RESEARCH ZERGGROUNDARMORSLEVEL3 |
| 5 | TRAIN HYDRALISK | 28 | BUILD GREATERSPIRE | 51 | RESEARCH INFESTORENERGYUPGRADE |
| 6 | TRAIN MUTALISK | 29 | BUILD ULTRALISKCAVERN | 52 | RESEARCH NEURALPARASITE |
| 7 | TRAIN CORRUPTOR | 30 | BUILD LURKERDEN | 53 | RESEARCH ZERGFLYERWEAPONSLEVEL1 |
| 8 | TRAIN INFESTOR | 31 | BUILD SPORECRAWLER | 54 | RESEARCH ZERGFLYERWEAPONSLEVEL2 |
| 9 | TRAIN SWARMHOST | 32 | BUILD SPINECRAWLER | 55 | RESEARCH ZERGFLYERWEAPONSLEVEL3 |
| 10 | TRAIN ULTRALISK | 33 | RESEARCH ZERGLINGMOVEMENTSPEED | 56 | RESEARCH ZERGFLYERARMORSLEVEL1 |
| 11 | TRAIN VIPER | 34 | RESEARCH ZERGLINGATTACKSPEED | 57 | RESEARCH ZERGFLYERARMORSLEVEL2 |
| 12 | MORPH BANELING | 35 | RESEARCH BANELINGMOVEMENTSPEED | 58 | RESEARCH ZERGFLYERARMORSLEVEL3 |
| 13 | MORPH RAVAGER | 36 | RESEARCH ROACHMOVEMENTSPEED | 59 | RESEARCH CHITINOUSPLATING |
| 14 | MORPH LURKER | 37 | RESEARCH TUNNELINGCLAWS | 60 | RESEARCH ANABOLICSYNTHESIS |
| 15 | MORPH BROODLORD | 38 | RESEARCH OVERLORDSPEED | 61 | RESEARCH LURKERRANGE |
| 16 | MORPH OVERSEER | 39 | RESEARCH BURROW | 62 | RESEARCH DIGGINGCLAWS |
| 17 | BUILD EXTRACTOR | 40 | RESEARCH HYDRAMOVEMENTSPEED | 63 | SCOUTING ZERGLING |
| 18 | BUILD HATCHERY | 41 | RESEARCH EVOLVEGROOVEDSPINES | 64 | SCOUTING OVERLORD |
| 19 | BUILD SPAWNINGPOOL | 42 | RESEARCH ZERGMELEEWEAPONSLEVEL1 | 65 | SCOUTING OVERSEER |
| 20 | BUILD BANELINGNEST | 43 | RESEARCH ZERGMELEEWEAPONSLEVEL2 | 66 | MULTI-ATTACK |
| 21 | BUILD ROACHWARREN | 44 | RESEARCH ZERGMELEEWEAPONSLEVEL3 | 67 | MULTI-RETREAT |
| 22 | BUILD HYDRALISKDEN | 45 | RESEARCH ZERGMISSILEWEAPONSLEVEL1 | 68 | QUEEN INJECTLARVA |
| | | | | 69 | EMPTY ACTION |

Terran exists only as a placeholder: `init_terran_actions` returns untranslated dummy strings (`0: "执行Terran动作0"`, `1: "执行Terran动作1"`), so only Protoss and Zerg are playable.

### How a macro action runs

Each macro action is dispatched to a method named `handle_action_<id>` in the bot (`Protoss_bot.py:2035-2038`, via Python's `getattr`). That method picks the units and target positions automatically — for example, `BUILD PYLON` finds a worker near the main base and a powered location, without the LLM ever naming a coordinate.

If a macro action cannot run (not enough minerals, required building missing, supply blocked), the bot catches the failure and calls `record_failure(action, reason)` (`Protoss_bot.py:745-746`). This appends a string of the form `Action failed: <ACTION NAME>, Reason: <reason>` to `action_failures`. The list is deep-copied into the transaction every step (line 2051) and shown back to the LLM in the next L2 prompt, so the model can see and correct its own infeasible decisions.

## LLM output structure and parsing

The model is expected to answer in three blocks, in order: the six analysis sections (`1. Game Overview`, `2. Current Game Stage`, `3. Our Situation` with the sub-points `3.1 Units and Buildings`, `3.2 Economy`, `3.3 Technology`, then `4. Our Strategy`, `5. Enemy's Strategy`, `6. Key Information`), the suggestions (`### Suggestions:` with `1. Our Strategy`, `2. Units and Buildings`, `3. Economy`, `4. Technology`), and the decisions (`### Decisions:` with a numbered list of action names taken from the dictionary).

??? example "Output (example)"
    ```markdown
    ### Game Overview / Current Game Stage / Our Situation / Our Strategy / Enemy's Strategy / Key Information
    ### Suggestions:
    1. Our Strategy ... 2. Units and Buildings ... 3. Economy ... 4. Technology ...
    ### Decisions:
    1. **TRAIN STALKER**
    2. **BUILD PYLON**
    3. **BUILD WARPGATE**
    4. **RESEARCH PROTOSSGROUNDWEAPONSLEVEL1**
    5. **CHRONOBOOST CYBERNETICSCORE**
    ```

The parser (`games/starcraft2/utils/action_extractor.py`) turns this free text into action IDs in five steps:

1. **Slice the decisions block.** A regular expression, `(?<=Decisions)[\s\S]*`, cuts out everything after the word `Decisions`. If the word is missing, the whole call yields the empty action (`[empty_idx]` / `["EMPTY ACTION"]`).
2. **Match action names as substrings.** Each of the 72 action names is tested against the sliced text with Python's `in` operator (plain substring containment, in the exact capitalization used in the dictionary). Every hit is collected, in dictionary order.
3. **Resolve names to IDs.** Each matched name is uppercased and looked up in a flattened `{NAME: id}` table (`ActionExtractor.extract_and_search_actions`). An exact hit returns the ID directly.
4. **Fall back to similarity search.** If the name is not in the table, `action_db_manager.search_actions(action)` queries a Chroma vector store that maps action names to embeddings. The store is created when the agent starts: `init_action_db()` (`gpt4_agent.py:50`) builds a `ChromaDBManager` (`games/starcraft2/vector_database/Chroma_vdb.py`) on top of a local `SentenceTransformer` model, `sentence-transformers/all-mpnet-base-v2`, which produces 768-dimensional vectors (`games/starcraft2/embedding/embedding_model.py:43`). The closest action's ID and name are returned. If the store returns nothing, the decision contributes no action at all.
5. **Queue the result.** In the example above, `BUILD WARPGATE` is not in the dictionary and is mapped to `BUILD GATEWAY` (ID 22) by the similarity search. The final IDs enter the action queue, are mixed with `EMPTY ACTION` at rate 0.5, and one ID is popped per game step.

## Evaluation data

| Capability dimension | Status | Data source and gaps |
|----------------------|--------|----------------------|
| Strategic planning | ✅ Available | `process_data` (`completed_tech`, `score`, supply, collected / spent minerals and vespene) and per-step `information.building` / `information.research`; additionally four per-step wandb indices: **RPM** = `collected_minerals + collected_vespene`, **EER** = `(collected_minerals + spent_vespene) * 100 / max(collected_minerals + collected_vespene, 1)`, **SUR** = `supply_used / max(supply_cap, 1)`, **TRR** = `completed_tech / max(total_research_count, 1)` (`gpt4_agent.py:200-216`); goal progress and tech-tree dependencies still need annotation |
| Real-time decision | ✅ Available | Synchronous vs asynchronous scenes directly contrast blocking and non-blocking LLM calls; `action_executed` / `action_failures` and parsing results are logged per step; **per-call latency (observation delivered → command returned) is not timestamped — needs instrumentation**; critical events (enemy attacks) need annotated criteria |
| Social reasoning | ⚠️ Needs probes | 1v1 against built-in AI, no language interaction; probes can ask for the opponent strategy type (Macro / Rush / Random), ground truth from scene config |
| Team collaboration | ➖ Not applicable | Single agent vs built-in AI, no teammates or communication |
| Adaptive learning | ✅ Available | Win / loss and score of 5 matches per scene; `action_failures` fed back each round allows checking repeated failures; cross-match memory policy must be specified |

## Metrics

The per-match metrics are computed in two places. During the match, the raw counters are accumulated step by step in `HistoryTracker.extract_match_info` (`agent_eval.py:1165-1188`): `pop_utilization` is updated every step as `supply_used / (supply_cap + previous_pop_utilization)`; `pop_cap_iter`, `pop_cap_spent_minerals`, and `pop_cap_spent_gas` take a snapshot of the iteration and of total spending **at the moment supply first hits the cap**; `game_over_iter` and `game_over_time` record when the game ended. After the match, `calc_eval_ret` (`multiprocess_eval_tasks.py:116-146`) averages the per-match matrices and logs them to wandb.

| Metric | Definition in code | Meaning |
|--------|--------------------|---------|
| Win rate | `result == 1` fraction; the environment sets `result = 1` iff `transaction['result'].name == 'Victory'`, else 0 (`agent_vs_computer_starcraft_env.py:187`) | Matches won against the built-in AI |
| PBR (Population Block Ratio) | `pop_cap_iter / game_over_iter` | Fraction of game steps spent at supply cap (lower = better macro management) |
| RUR (Resource Utilization Ratio) | `(pop_cap_spent_minerals + pop_cap_spent_gas) / game_over_time` | Resource spending efficiency (see the [game page metrics](../../environments/starcraft2/starcraft2.md#evaluation-metrics)) |
| APU (Average Population Utilization) | `pop_utilization / count` | Mean supply used relative to supply cap across the game |
| TR (Technology Rate) | `process_data['completed_tech']` | Number of completed technologies = building types owned + researches finished (`Protoss_bot.py:465`) |
| grounding_acc | `success_exe_action_num / max(valid_action_num, 1)` | Share of parsed non-empty decisions that executed without failure (`gpt4_agent.py:268`) |

## Source code

| Part | File |
|------|------|
| Environment wrapper | `games/starcraft2/agent_vs_computer_starcraft_env.py` |
| Observation (`get_information`, `get_process_data`) and action execution (`handle_action_*`) | `games/starcraft2/bot/Protoss_bot.py` |
| Text summary (L1) | `games/starcraft2/summarize/L1_summarize.py` |
| Agent: cadence, queue, grounding counters, wandb logs | `agent_manager/agents/starcraft2_agent/gpt4_agent.py` |
| Prompt (system + few-shot, race-conditional) | `agent_manager/prompts/starcraft2_prompt.py` (`StarCraftIIPrompt_V4`) |
| Output parsing (regex + exact match + Chroma fallback) | `games/starcraft2/utils/action_extractor.py` |
| Action tables (Protoss / Zerg / Terran stub) | `games/starcraft2/utils/action_info.py` |
| Metric accumulation (`extract_match_info`) | `agent_eval/agent_eval.py:1165-1188` |
| Metric aggregation (`calc_eval_ret`) | `multiprocess_eval_tasks.py` |
| Evaluation scene configs | `configs/eval_config_base/eval_starcraft2_agent_vs_computer_scene1-6.yaml` |
| Raw observation (`GameState`, `Common`, `ScoreDetails`, `Unit`) | python-sc2 library: `sc2/game_state.py`, `sc2/score.py`, `sc2/unit.py`, `sc2/bot_ai.py` |
| Raw action (`UnitCommand`, `client.actions`, `ActionRaw*`) | python-sc2 library: `sc2/unit_command.py`, `sc2/client.py`, `sc2/unit.py:1452` |
