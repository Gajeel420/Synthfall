# Systems Index: SYNTHFALL

> **Status**: Draft
> **Created**: 2026-05-16
> **Last Updated**: 2026-05-16
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

SYNTHFALL is a 5-player asymmetric team shooter (4 Reclaimers vs 1 Hive Mind) set
in the corrupted smart city of AXIOM on Mars. The game's mechanical scope is driven
by two radically different play experiences that share one world: an FPS survival
loop for Reclaimers and an omniscient city-control loop for the Hive Mind. The
Corruption Meter is the shared tension engine that links both sides — Reclaimer
mistakes feed it, Hive Mind pressure accelerates it, and objective completions slow
it. Every system exists to serve one of the four pillars: the city as active opponent,
asymmetry as the core design value, every match telling a story, or player identity
over loadout. The design must be completed in dependency order — networking and
session authority first, tension engine second, then the Reclaimer and Hive Mind
toolsets that operate on top of it.

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | Networking Layer | Foundation | Vertical Slice | Designed | design/gdd/networking-layer.md | — |
| 2 | Input System | Foundation | Vertical Slice | Not Started | — | — |
| 3 | Event Bus (inferred) | Foundation | Vertical Slice | Not Started | — | — |
| 4 | Match State Machine | Core | Vertical Slice | Not Started | — | F1, F3 |
| 5 | Player Session & Role System | Core | Vertical Slice | Not Started | — | F1, C1 |
| 6 | Corruption Meter | Core | Vertical Slice | Not Started | — | F3, C1 |
| 7 | Win/Loss Conditions | Core | Vertical Slice | Not Started | — | C1, C2 |
| 8 | Reclaimer Movement | Feature — Reclaimer | Vertical Slice | Not Started | — | F1, F2, C1 |
| 9 | Sensor Grid / Hive Mind Vision | Feature — Hive Mind | Vertical Slice | Not Started | — | F1, R1 |
| 10 | Hive Mind View System | Feature — Hive Mind | Vertical Slice | Not Started | — | F2, H4 |
| 11 | Sector / Level System | Feature — Level | MVP | Not Started | — | F1, C1 |
| 12 | Spawn/Respawn System (inferred) | Feature — Reclaimer | MVP | Not Started | — | F1, C1, C4 |
| 13 | Combat System | Feature — Reclaimer | MVP | Not Started | — | F1, F2, R1, C2 |
| 14 | Class Ability System | Feature — Reclaimer | MVP | Not Started | — | F2, R1, R6, C2 |
| 15 | Resource Management | Feature — Reclaimer | MVP | Not Started | — | R2, R3, C2 |
| 16 | Objective System | Feature — Reclaimer | MVP | Not Started | — | R1, C2, C3 |
| 17 | Hive Mind Unit Deployment | Feature — Hive Mind | MVP | Not Started | — | F1, C4, C2, C1 |
| 18 | Unit AI System | Feature — Hive Mind | MVP | Not Started | — | H1, R1 |
| 19 | Environmental Hazard System | Feature — Hive Mind | MVP | Not Started | — | F1, C4, C2, L1 |
| 20 | Reclaimer HUD | Presentation | MVP | Not Started | — | C2, R2, R3, R4 |
| 21 | Hive Mind UI / Overlay | Presentation | MVP | Not Started | — | H5, H1, H3, C2 |
| 22 | Post-Match Debrief System | Presentation | Launch | Not Started | — | C3, C2, C1 |
| 23 | Audio System (inferred) | Presentation | Launch | Not Started | — | F3, C2 |
| 24 | VFX System (inferred) | Presentation | Launch | Not Started | — | F3, C2, H1 |
| 25 | Matchmaking / Lobby System | Meta | Launch | Not Started | — | F1, C4 |
| 26 | Progression System | Meta | Launch | Not Started | — | C1, R3 |
| 27 | Settings System (inferred) | Meta | Launch | Not Started | — | F2, P4, P1 |
| 28 | Mission Log (inferred) | Meta | Launch | Not Started | — | C1, C3 |
| 29 | Cosmetic System | Meta | Launch | Not Started | — | M2 |
| 30 | Hive Mind Personalities | Meta | Live Ops | Not Started | — | H1, H3 |

*Systems marked (inferred) were not explicitly named in the game concept but are
required by the systems that were.*

---

## Categories

| Category | Description |
|----------|-------------|
| **Foundation** | Infrastructure systems with no dependencies — networking, input, internal messaging |
| **Core** | Session and tension engine systems — match state, role assignment, Corruption, win/loss |
| **Feature — Reclaimer** | All Reclaimer-side gameplay: movement, combat, classes, resources, objectives |
| **Feature — Hive Mind** | All Hive Mind-side gameplay: unit deployment, AI, hazards, sensor grid, god-view |
| **Feature — Level** | Spatial systems: sector/level loading, modular tile management |
| **Presentation** | Player-facing feedback: HUDs, post-match, audio, VFX |
| **Meta** | Metagame systems: matchmaking, progression, cosmetics, settings, live ops |

---

## Priority Tiers

| Tier | Definition | Goal |
|------|------------|------|
| **Vertical Slice** | Minimum to test: "Is 1v4 asymmetric tension interesting?" | 4 Reclaimers move through a space. Corruption builds. Hive Mind watches through cameras. Match can end. |
| **MVP** | Full 1-sector run: 4 classes, Hive Mind toolset (3 units + 5 hazards), LAN/local | Answers: "Is a 30-minute asymmetric match with a living city fun for strangers?" |
| **Launch** | 3 sectors, full progression, cosmetics, Steam matchmaking, post-match debrief | Public release on Steam/Epic |
| **Live Ops** | Post-launch seasonal content | Hive Mind personalities, new sectors, faction events |

---

## Dependency Map

### Foundation Layer (no dependencies)

1. **Networking Layer** — Authority model and session lifecycle; everything syncs through it
2. **Input System** — KB/M + gamepad bindings; Reclaimer control and Hive Mind cursor inputs
3. **Event Bus** — Internal message bus; decouples 21+ MVP systems from direct coupling

### Core Layer (depends on Foundation)

1. **Match State Machine** — depends on: Networking Layer, Event Bus
2. **Player Session & Role System** — depends on: Networking Layer, Match State Machine
3. **Corruption Meter** — depends on: Event Bus, Match State Machine
4. **Win/Loss Conditions** — depends on: Match State Machine, Corruption Meter

### Feature Layer (depends on Foundation + Core)

**Reclaimer systems:**
1. **Sector / Level System** — depends on: Networking Layer, Match State Machine
2. **Spawn/Respawn System** — depends on: Networking Layer, Match State Machine, Player Session & Role System
3. **Reclaimer Movement** — depends on: Networking Layer, Input System, Match State Machine
4. **Combat System** — depends on: Networking Layer, Input System, Reclaimer Movement, Corruption Meter
5. **Class Ability System** — depends on: Input System, Reclaimer Movement, Spawn/Respawn System, Corruption Meter
6. **Resource Management** — depends on: Combat System, Class Ability System, Corruption Meter
7. **Objective System** — depends on: Reclaimer Movement, Corruption Meter, Win/Loss Conditions

**Hive Mind systems:**
1. **Hive Mind Unit Deployment** — depends on: Networking Layer, Player Session & Role System, Corruption Meter, Match State Machine
2. **Unit AI System** — depends on: Hive Mind Unit Deployment, Reclaimer Movement
3. **Environmental Hazard System** — depends on: Networking Layer, Player Session & Role System, Corruption Meter, Sector / Level System
4. **Sensor Grid / Hive Mind Vision** — depends on: Networking Layer, Reclaimer Movement *(unit positions stubbed in VS)*
5. **Hive Mind View System** — depends on: Input System, Sensor Grid / Hive Mind Vision *(H1/H3 stubbed in VS)*

### Presentation Layer (depends on Feature)

1. **Reclaimer HUD** — depends on: Corruption Meter, Combat System, Class Ability System, Resource Management
2. **Hive Mind UI / Overlay** — depends on: Hive Mind View System, Hive Mind Unit Deployment, Environmental Hazard System, Corruption Meter
3. **Post-Match Debrief System** — depends on: Win/Loss Conditions, Corruption Meter, Match State Machine
4. **Audio System** — depends on: Event Bus, Corruption Meter
5. **VFX System** — depends on: Event Bus, Corruption Meter, Hive Mind Unit Deployment

### Meta Layer (depends on Presentation + Feature)

1. **Matchmaking / Lobby System** — depends on: Networking Layer, Player Session & Role System
2. **Progression System** — depends on: Match State Machine, Class Ability System
3. **Settings System** — depends on: Input System, Audio System, Reclaimer HUD
4. **Mission Log** — depends on: Match State Machine, Win/Loss Conditions
5. **Cosmetic System** — depends on: Progression System
6. **Hive Mind Personalities** — depends on: Hive Mind Unit Deployment, Environmental Hazard System

---

## Recommended Design Order

| Order | System | Priority | Layer | Effort |
|-------|--------|----------|-------|--------|
| 1 | Networking Layer | Vertical Slice | Foundation | L |
| 2 | Input System | Vertical Slice | Foundation | S |
| 3 | Event Bus | Vertical Slice | Foundation | S |
| 4 | Match State Machine | Vertical Slice | Core | M |
| 5 | Player Session & Role System | Vertical Slice | Core | M |
| 6 | Corruption Meter | Vertical Slice | Core | M |
| 7 | Win/Loss Conditions | Vertical Slice | Core | S |
| 8 | Reclaimer Movement | Vertical Slice | Feature | M |
| 9 | Sensor Grid / Hive Mind Vision | Vertical Slice | Feature | M |
| 10 | Hive Mind View System | Vertical Slice | Feature | M |
| 11 | Sector / Level System | MVP | Feature | M |
| 12 | Spawn/Respawn System | MVP | Feature | S |
| 13 | Combat System | MVP | Feature | L |
| 14 | Class Ability System | MVP | Feature | L |
| 15 | Resource Management | MVP | Feature | M |
| 16 | Objective System | MVP | Feature | M |
| 17 | Hive Mind Unit Deployment | MVP | Feature | M |
| 18 | Unit AI System | MVP | Feature | L |
| 19 | Environmental Hazard System | MVP | Feature | M |
| 20 | Reclaimer HUD | MVP | Presentation | M |
| 21 | Hive Mind UI / Overlay | MVP | Presentation | M |
| 22 | Post-Match Debrief System | Launch | Presentation | M |
| 23 | Audio System | Launch | Presentation | M |
| 24 | VFX System | Launch | Presentation | M |
| 25 | Matchmaking / Lobby System | Launch | Meta | M |
| 26 | Progression System | Launch | Meta | L |
| 27 | Settings System | Launch | Meta | S |
| 28 | Mission Log | Launch | Meta | S |
| 29 | Cosmetic System | Launch | Meta | M |
| 30 | Hive Mind Personalities | Live Ops | Meta | M |

*Effort: S = 1 session, M = 2–3 sessions, L = 4+ sessions.*
*Systems at the same dependency layer with no mutual dependencies can be designed in parallel.*

---

## Circular Dependencies

None detected. The dependency graph flows in one direction:
Foundation → Core → Feature → Presentation → Meta.

**Note**: Sensor Grid / Hive Mind Vision has a full-game dependency on Unit AI System
(unit positions), but this is stubbed in the Vertical Slice — only Reclaimer positions
are tracked in VS. The circular risk is avoided by staggered implementation scope.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| **Networking Layer** | Technical | NGO 2.X is post-cutoff (2026) — API details unverified. Asymmetric authority model (Hive Mind owns environment, Reclaimers own characters) is a novel pattern not covered by standard NGO tutorials. Wrong decisions here cascade to all 14 dependent systems. | Design ADR before GDD. Prototype authority model in VS. Pin NGO 2.X from day one — do NOT use NGO 1.X (deprecated in Unity 6.3). Evaluate relay vs. P2P vs. dedicated server topology early. |
| **Corruption Meter** | Design | The balance between tense and frustrating is narrow. Too fast: Hive Mind wins every match. Too slow: matches feel toothless. The meter is also the cross-system interface — 9 systems read or write to it. If the formula is wrong, it's a systemic fix. | Define formula with explicit tuning knobs. Set separate passive rate, player-action rate, and Hive Mind acceleration rate as independent dials. Playtest baseline values early. |
| **Unit AI System** | Technical | Three distinct AI behavior types (Crawler swarm, Stalker heat-tracking, Drone electronics disruption) in a dynamic AXIOM environment with destructible/modifiable geometry. Pathfinding must be network-aware. Unity 6.3 navigation system changes are post-cutoff. | Design each unit as a separate state machine. Prototype Crawler swarm behavior first — it's the simplest. Verify Unity NavMesh 2.0 API against engine reference docs before coding. |
| **Class Ability System** | Scope | 4 classes × unique ability sets × asymmetric balance across both Reclaimer and Hive Mind play styles = high design complexity. Each ability interacts with Corruption Meter, Resource Management, and Spawn/Respawn. | Design all 4 classes in a single GDD session. Define abilities as data-driven configs from the start. Prototype Breacher first (simplest kit). Defer passive ability trees to Launch. |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 30 |
| Design docs started | 1 |
| Design docs reviewed | 0 |
| Design docs approved | 0 |
| Vertical Slice systems designed | 1 / 10 |
| MVP systems designed | 0 / 21 |
| Launch systems designed | 0 / 29 |

---

## Next Steps

- [ ] Run `/design-system networking-layer` — first system in the design order
- [ ] Run `/design-review design/gdd/networking-layer.md` in a **fresh session** after it's complete
- [ ] Run `/design-system input-system` and `/design-system event-bus` (can be done in parallel — no mutual dependency)
- [ ] Run `/gate-check systems-design` when all VS GDDs are authored (triggers CD-SYSTEMS + TD-SYSTEM-BOUNDARY gates)
- [ ] Run `/gate-check pre-production` when all MVP GDDs are authored and reviewed
