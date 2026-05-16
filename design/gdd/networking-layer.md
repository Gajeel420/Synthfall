# Networking Layer

> **Status**: In Design
> **Author**: Design Team + Claude Code Game Studios
> **Last Updated**: 2026-05-16
> **Implements Pillar**: Pillar 2 — Asymmetry Is the Point

## Overview

Standard multiplayer games synchronize multiple copies of the same experience.
SYNTHFALL synchronizes two fundamentally different games running in one session: a
ground-level FPS for four Reclaimer clients and a god-view city-control interface for
the single Hive Mind client. The Networking Layer defines how these two views share
one authoritative game state — what each role can read, what each role can write, and
how all five clients observe changes consistently. It owns the session lifecycle from
lobby through post-match, the asymmetric authority model (Reclaimers own their
characters; the Hive Mind owns all environmental systems and deployed units), and the
replication schema for the Corruption Meter, hazard states, unit positions, and
objective progress. Fourteen of the thirty game systems depend on the contracts this
layer establishes.

## Player Fantasy

When the Networking Layer is working, players never think about it. For Reclaimers,
this means movement and combat responses feel instantaneous — shots land or miss based
on what actually happened, not network timing. When a blast door slams the squad, it
reads as the Hive Mind reacting to them, not a delayed packet arriving 200ms later.
For the Hive Mind, unit deployments and hazard triggers take effect where and when
they appear to — the city-control fantasy requires the environment to respond to
intent, not lag. Beneath both experiences is a shared trust in the authority model:
neither side can exploit the session, and both sides see the same ground truth. The
networking layer's player fantasy is its own disappearance.

## Detailed Design

### Core Rules

1. **Server is the single source of truth.** All game-critical state lives on the server.
   No client may modify authoritative state without a server-validated request. If the
   server rejects an action, the client's local prediction is corrected.

2. **Topology: listen-server for VS and MVP.** One of the four Reclaimer clients acts
   as host and server simultaneously. The hosted player experiences no network round-trip
   for their own actions — this is the only asymmetry in local experience between host
   and non-host Reclaimers. The authority model is designed to run identically on a
   dedicated server at Launch; no architectural change is required, only a deployment
   change.

3. **Reclaimer clients own their characters.** Each Reclaimer client owns its own
   character NetworkObject. It sends movement input and action requests to the server
   via ServerRpc. The server validates and applies all results. Clients display locally
   predicted movement, reconciled against the server position.

4. **The Hive Mind owns nothing.** The Hive Mind client does not own any NetworkObject.
   All Hive Mind actions — unit deployment, hazard triggers, Corruption pressure — are
   sent as ServerRpc calls. The server validates and executes. The Hive Mind cannot
   directly set the state of any object.

5. **State visibility is asymmetric.** Reclaimer clients receive: all Reclaimer
   positions, Synth units within detection range, active hazard states, Corruption Meter
   value, and objective states. The Hive Mind client receives: all Reclaimer positions
   (full omniscience), all unit positions, and all environmental states. Hive Mind
   omniscience is implemented as broader replication visibility — not a workaround.

6. **The Corruption Meter is a server-owned NetworkVariable.** Only the server writes
   to it. All clients receive the current value. Reclaimer actions and Hive Mind commands
   trigger ServerRpcs that request server-side modification. Neither side can manipulate
   the meter client-side.

### States and Transitions

| Session State | Entry Condition | Exit Condition | Server Behavior |
|---|---|---|---|
| **Lobby** | Session created by host | All 5 clients connected and roles assigned | Hold; wait for ready signals |
| **Connecting** | All roles assigned, host triggers start | All clients confirm spawn complete | Spawn all player NetworkObjects |
| **InMatch** | All clients spawned | Win/loss condition fires, or all clients disconnect | Run match logic; replicate all state |
| **SectorTransition** | (InMatch) Reclaimers reach sector exit | All clients load next sector | Pause match logic; stream next sector |
| **PostMatch** | Win/loss fires | Host ends session, or timer expires | Compute debrief data; broadcast to clients |

*SectorTransition is a sub-state of InMatch. The Corruption Meter persists through
sector transitions; it is not reset.*

### Interactions with Other Systems

| System | Data Flow | Interface |
|---|---|---|
| **Match State Machine** | Networking Layer provides session lifecycle events; Match State Machine maps them to match state | `OnClientConnectedCallback`, `OnClientDisconnectCallback` → Match State Machine subscribes via Event Bus |
| **Corruption Meter** | Corruption value lives as a server-owned `NetworkVariable<float>`. Reclaimer and Hive Mind actions trigger ServerRpcs that request modification. All clients read the current value via `OnValueChanged`. | `CorruptionMeter.NetworkVariable<float>` + `ModifyCorruptionServerRpc(float delta)` |
| **Player Session & Role System** | On client connect, server assigns Reclaimer or Hive Mind role. Role assignment determines replication visibility group (Reclaimer vs. Hive Mind state sets). | `OnClientConnectedCallback` → Role System assigns role; Networking Layer configures visibility per role |
| **All Feature Systems (R1–R5, H1–H5, L1)** | Feature systems send ServerRpcs for state-changing actions; receive updates via NetworkVariables and ClientRpcs. No feature system reads state from another client directly. | Per-system RPC contracts defined in each system's GDD |

## Formulas

### F1A — Server Tick Period

`T_tick = 1000 / R_tick`

**Variables:**
| Variable | Symbol | Type | Range | Description |
|---|---|---|---|---|
| Tick rate | R_tick | int | 20–128 Hz | Server ticks per second. **Designed value: 30 Hz.** |
| Tick period | T_tick | float | 7.8–50ms | Duration of one server tick |

**Output Range:** 7.8ms (128Hz) to 50ms (20Hz). Designed value: 33.3ms at 30Hz.

**Rationale:** 30Hz matches the Left 4 Dead server model (the design reference for
SYNTHFALL's pacing). Synth units are large targets — hit detection is not sub-pixel
sensitive. Keeps host CPU and upstream bandwidth affordable on a listen-server.

**Example:** R_tick = 30 → T_tick = 1000 / 30 = **33.3ms per tick**

---

### F1B — Maximum Positional Error Per Tick

`E_pos = V_entity × T_tick / 1000`

**Variables:**
| Variable | Symbol | Type | Range | Description |
|---|---|---|---|---|
| Entity speed | V_entity | float | 0–12 m/s | Design maximum sprint speed |
| Tick period | T_tick | float | 33.3ms | Fixed at 30Hz |
| Positional error | E_pos | float | 0–0.40m | Maximum distance entity travels between ticks |

**Output Range:** 0.0m (stationary) to 0.40m (12 m/s sprint). Unbounded above if
V_entity exceeds the designed maximum — sprint cap must be enforced by the movement
system.

**Example (Stalker unit, 2.5 m/s):** E_pos = 2.5 × 33.3 / 1000 = **0.083m (8.3cm)**
— 14% of Stalker hitbox radius (0.6m). Acceptable without interpolation correction.

**Example (Reclaimer sprint, 6 m/s):** E_pos = 6.0 × 33.3 / 1000 = **0.20m (20cm)**
— exceeds acceptable threshold for FPS hit detection. Reclaimer movement uses
client-side prediction + server reconciliation rather than raw tick snapping.

---

### F2A — Host Upstream Bandwidth

`BW_host = (N_r × S_r + N_u × S_u + S_c + S_o) × R_tick × N_remote / 1024`

**Variables:**
| Variable | Symbol | Type | Range | Description |
|---|---|---|---|---|
| Reclaimer clients | N_r | int | 4 | Fixed for SYNTHFALL |
| Bytes per Reclaimer/tick | S_r | int | 36–60 | Position + anim + health (delta-compressed) |
| Active Synth units | N_u | int | 0–**30** | Hard cap — network budget constraint |
| Bytes per unit/tick | S_u | int | 20–28 | Position + state (high delta compression) |
| Corruption Meter bytes | S_c | int | 8 | Float + dirty flag |
| Overhead per tick | S_o | int | 40–80 | NGO header + RPC overhead |
| Tick rate | R_tick | int | 30 Hz | Fixed |
| Remote clients | N_remote | int | 4 | Non-host clients receiving from server |
| Host upstream | BW_host | float | — | KB/s consumed from host upload |

**Bandwidth Ceiling:** 200 KB/s. Minimum host upload requirement: **2 Mbps**
(provides 25% headroom above ceiling).

**Example (mid-match, 15 units):** (4×48 + 15×24 + 8 + 60) × 30 × 4 / 1024
= 620 × 120 / 1024 = **72.7 KB/s** — 36% of ceiling. ✅

**Example (maximum stress, 30 units):** (4×60 + 30×28 + 8 + 80) × 30 × 4 / 1024
= 1168 × 120 / 1024 = **136.9 KB/s** — 68% of ceiling. ✅

**Design constraint:** The 30-unit cap is validated here. Exceeding it breaks the
bandwidth budget. Unit cap is both a gameplay design decision and a network budget
constraint — they resolve to the same number.

---

### F3A — Effective Latency (Reclaimer)

`L_effective = RTT / 2 + T_tick`

**Variables:**
| Variable | Symbol | Type | Range | Description |
|---|---|---|---|---|
| Round-trip time | RTT | float | 0–500ms | Network round-trip latency |
| Tick period | T_tick | float | 33.3ms | Server processing floor |
| Effective latency | L_effective | float | 33.3–283ms | Min time between action and confirmed server result |

**Output Range:** 33.3ms minimum (zero network). Client-side prediction masks this
for movement; server reconciliation (correction) arrives after L_effective.

---

### Latency Tolerance Tiers

| Tier | RTT | L_effective | Reclaimer Impact | Hive Mind Impact |
|---|---|---|---|---|
| **Good** | ≤80ms | ≤73ms | Corrections imperceptible; combat feels local | Deployment and hazard triggers feel instant |
| **Acceptable** | 81–150ms | 74–108ms | Occasional single-frame corrections; 95%+ shot registration correct | 1–2 tick apparent delay on deployments |
| **Degraded** | 151–250ms | 118–158ms | Visible rubber-banding; combat noticeably impaired | Hazard triggers miss tactical windows |
| **Unplayable** | >250ms | >158ms | Reliable play impossible | Session should warn and offer exit |

**Matchmaking constraint:** Sessions must reject connections with RTT > 200ms at
join time. The Unplayable tier exists to handle mid-session degradation only.

---

### F3B — Hive Mind Deployment Window Miss Rate

`P_miss = clamp((RTT − 80) / (250 − 80), 0.0, 1.0)`

**Variables:**
| Variable | Symbol | Type | Range | Description |
|---|---|---|---|---|
| Round-trip time | RTT | float | 0–500ms | Hive Mind client latency |
| P_miss | float | 0.0–1.0 | Fraction of time-sensitive deployments that miss their tactical window |

**Output Range:** 0.0 at RTT ≤ 80ms (Good tier). 1.0 at RTT ≥ 250ms (Unplayable).
Used to anchor latency tier thresholds, not as a real-time runtime value.

**Example (RTT = 120ms):** P_miss = (120 − 80) / 170 = **0.24** — 24% of
time-sensitive placements miss their window. Edge of Acceptable.

---

### F4 — Per-Client RTT Under Host Load

`RTT_actual = RTT_baseline + RTT_queue + T_tick`

Where `RTT_queue = max(0, (BW_host − BW_ceiling) / BW_ceiling) × T_congestion_penalty`

**Variables:**
| Variable | Symbol | Type | Range | Description |
|---|---|---|---|---|
| Baseline RTT | RTT_baseline | float | 10–200ms | RTT measured at session start |
| Queue latency | RTT_queue | float | 0–200ms | Added latency when host upload is congested |
| Congestion penalty | T_congestion_penalty | float | 50–200ms | Tuning knob — host OS buffering behavior; default: 100ms |
| Actual RTT | RTT_actual | float | 43–433ms | Total effective round-trip time for non-host client |

**Example (good host, 25ms baseline, no congestion):** RTT_actual = 25 + 0 + 33.3
= **58.3ms** — Good tier. ✅

**Example (stressed host, 40ms baseline, 30 units active → 280 KB/s):**
RTT_queue = max(0, (280 − 200) / 200) × 100 = 40ms
RTT_actual = 40 + 40 + 33.3 = **113.3ms** — Acceptable tier. Session remains
playable; Hive Mind will notice deployment delay.

## Edge Cases

- **If fewer than 5 players are connected when the host triggers match start**: The
  server rejects the start request. The lobby state remains active. Minimum session
  size is 5 — no bot fill for Reclaimers or Hive Mind in MVP.

- **If the Hive Mind client disconnects mid-match**: Pause the match. Display a
  "Hive Mind connection lost" notification to all clients. Wait up to 60 seconds for
  reconnect. If the Hive Mind does not reconnect within 60 seconds, award victory to
  the Reclaimers and transition to PostMatch. Do not replace the Hive Mind with AI
  in MVP.

- **If a Reclaimer client disconnects mid-match**: The disconnected client's character
  is removed from the world immediately. The remaining Reclaimers continue. The match
  is not paused. If all 4 Reclaimers disconnect, the server ends the session without
  awarding a Hive Mind victory — all clients receive no result.

- **If the host Reclaimer disconnects (listen-server host leaves)**: The session is
  terminated for all clients. No host migration in MVP. All clients receive a "session
  ended" notice and return to lobby. Host migration is a Launch-tier concern.

- **If a ServerRpc arrives from a client who does not own the relevant object**: The
  server discards the request silently and logs a warning. Reclaimer action RPCs are
  valid only from the owning client. Hive Mind RPCs cover only deployment and hazard
  requests — never character actions.

- **If a ServerRpc requests a Corruption Meter delta that would exceed 100% or go
  below 0%**: The server clamps the Corruption value to [0.0, 100.0] before applying.
  A clamped application still counts as a valid action — no rollback or rejection.

- **If two Hive Mind actions arrive at the server in the same tick that would both
  push Corruption to 100%**: Apply both in arrival order. The first that crosses 100%
  triggers the win condition. The second is applied to the clamped value and has no
  additional effect. The win condition fires exactly once per match.

- **If host upstream bandwidth exceeds the 200 KB/s ceiling**: The server
  automatically throttles low-priority replication in order: first, Hive Mind god-view
  decoration updates (unit trail effects, environmental detail); then, non-visible
  Synth unit position frequency for Reclaimer clients. Reclaimer position data and
  Corruption Meter sync are never throttled.

- **If a client's RTT exceeds the Degraded tier (250ms) during a session**: The
  server sends a high-latency warning to that client's HUD. The client is not force-
  disconnected. If RTT exceeds 350ms for more than 10 consecutive seconds, the server
  sends a disconnect recommendation. Forced disconnect is not implemented in MVP.

- **If the sector transition state is entered but one client fails to load the next
  sector within 30 seconds**: The server terminates the session. Partial sector loads
  are not recoverable in MVP — level streaming must succeed for all clients or none.

## Dependencies

### Upstream Dependencies

None. The Networking Layer is the foundation layer — it has no design-level
dependencies on other game systems.

### Downstream Dependents

All systems that replicate state or validate authority depend on the contracts this
layer defines. Listed in design order:

| System | Dependency Type | What It Needs |
|---|---|---|
| Match State Machine | Hard | Session lifecycle events (connect, disconnect, start, end) |
| Player Session & Role System | Hard | Client connection callbacks; authority assignment mechanism |
| Corruption Meter | Hard | Server-owned NetworkVariable and ServerRpc pattern |
| Win/Loss Conditions | Hard | Server-authoritative win trigger broadcast to all clients |
| Sector / Level System | Hard | Synchronized level load across all clients |
| Spawn/Respawn System | Hard | Server-side NetworkObject spawn and despawn |
| Reclaimer Movement | Hard | Client-owned NetworkObject; position replication; prediction/reconciliation |
| Combat System | Hard | ServerRpc validation for hit registration |
| Class Ability System | Hard | ServerRpc for ability activation; authority checks per ability |
| Resource Management | Hard | Server-authoritative resource state; sync to owning client |
| Objective System | Hard | Server-side objective state; ClientRpc broadcast on completion |
| Hive Mind Unit Deployment | Hard | ServerRpc for unit spawn requests; server-delegated unit authority |
| Environmental Hazard System | Hard | ServerRpc for hazard trigger requests |
| Sensor Grid / Hive Mind Vision | Hard | Role-based replication visibility groups (Hive Mind receives omniscient state set) |

All 14 downstream dependents are hard dependencies — none can function without the
authority model this layer defines.

Each downstream system's GDD must list "Networking Layer" under its own Dependencies
section, referencing the specific authority contract it relies on.

## Tuning Knobs

| Knob | Default | Safe Range | Effect if Too High | Effect if Too Low |
|---|---|---|---|---|
| `server_tick_rate` | 30 Hz | 20–60 Hz | Host CPU overhead increases; upload bandwidth doubles at 60Hz | Below 20Hz: hit registration fails for FPS combat; unit tracking becomes jerky |
| `max_active_units` | 30 | 10–50 | Exceeds bandwidth ceiling; host upload saturates; all clients experience latency spikes | Below 15: Hive Mind swarm fantasy feels underpowered; tactical variety collapses |
| `host_bandwidth_ceiling` | 200 KB/s | 100–400 KB/s | No quality effect — only adjusts the threshold shown to hosts in matchmaking | Too low: good hosts are incorrectly warned; too high: bad hosts degrade all clients |
| `hive_mind_reconnect_window` | 60 s | 15–120 s | Long pauses break Reclaimer momentum | Below 15s: brief interruptions end matches prematurely |
| `latency_warning_threshold` | 250 ms | 150–350 ms | Warning appears after player is already suffering | Below 150ms: false warnings on Acceptable-tier connections |
| `latency_kick_recommendation_threshold` | 350 ms | 300–500 ms | Recommendation arrives too late | Below 300ms: recommendation triggers during transient spikes |
| `latency_sustained_window` | 10 s | 5–30 s | Player endures degraded play longer before recommendation | Below 5s: transient spikes trigger recommendation; loses credibility |
| `congestion_throttle_penalty` | 100 ms | 50–200 ms | Congestion overestimated; throttle triggers too aggressively | Under-estimates bufferbloat; throttle triggers too late |

**Coupled knobs:** `server_tick_rate` and `max_active_units` are coupled — raising
either increases host upstream bandwidth. Never raise both simultaneously without
re-running Formula 2A to verify the ceiling is not exceeded.

## Visual/Audio Requirements

*Not applicable — the Networking Layer is infrastructure. Players experience what it
enables, not the system itself. See Reclaimer HUD GDD and Hive Mind UI GDD for the
UI surfaces that expose network-dependent state (latency indicator, connection quality
badge).*

## UI Requirements

*Not applicable — see above. The latency warning HUD element referenced in
AC-NET-15 and AC-NET-16 is owned by the Reclaimer HUD GDD and Hive Mind UI GDD,
not this layer. This layer provides the trigger events; the UI systems consume them.*

## Acceptance Criteria

All criteria are in GIVEN / WHEN / THEN format and are independently verifiable by a
QA tester. Test files must exist at `tests/unit/networking/` and
`tests/integration/networking/` before any story referencing this GDD can be marked Done.

---

### Core Rules (Section C)

**AC-NET-01 — Server Authority: Client Prediction Correction**
GIVEN a Reclaimer client has applied local movement prediction placing the character at position P1,
WHEN the server-authoritative position is P2 and P2 differs from P1 by more than the reconciliation threshold,
THEN the client corrects its displayed position to P2 within one server tick (33.3ms) and the local prediction state is discarded.

**AC-NET-02 — Server Authority: Rejected Action Rolls Back Client**
GIVEN a Reclaimer client locally predicts an action (e.g., a damage hit),
WHEN the server rejects that action due to validation failure,
THEN the client's local state for that action is rolled back, and no lasting effect from the rejected action persists on the client display for more than one tick.

**AC-NET-03 — Topology: Host Experiences No Network Round-Trip**
GIVEN a 5-player session where Player 1 is the listen-server host,
WHEN Player 1 performs a movement input,
THEN Player 1's character position update is applied within the same server tick, while at least one non-host Reclaimer client reflects the same position update after their measured RTT plus one tick period.

**AC-NET-04 — Reclaimer Ownership: Direct Variable Write Rejected**
GIVEN a Reclaimer client owns its character NetworkObject,
WHEN the client attempts to directly set a server-owned NetworkVariable (bypassing ServerRpc),
THEN the modification is rejected by the server, a warning is logged, and the client's local state is overwritten with the authoritative server state on the next tick.

**AC-NET-05 — Hive Mind Owns No NetworkObject**
GIVEN a client has been assigned the Hive Mind role,
WHEN the session is in InMatch state,
THEN querying NetworkManager for all NetworkObjects owned by the Hive Mind client ID returns an empty collection, and the Hive Mind client has no `IsOwner == true` references to any NetworkObject in the scene.

**AC-NET-06 — Hive Mind Actions Execute via ServerRpc Only**
GIVEN a Hive Mind client sends a unit deployment request,
WHEN the request arrives at the server,
THEN the server spawns the unit under server authority, the Hive Mind client receives no ownership of the spawned object, and any attempt to directly set state on an environmental NetworkObject from the Hive Mind client is silently discarded and logged.

**AC-NET-07 — State Visibility: Reclaimers Do Not Receive Out-of-Range Units**
*(Post-MVP flag: full automated verification of replication visibility groups requires NGO test hooks. In MVP, verify via server-side delivery logs.)*
GIVEN a Reclaimer client is in InMatch with 10 deployed Synth units, 4 outside detection range,
WHEN the server sends its replication tick to the Reclaimer client,
THEN the Reclaimer client does not receive position updates for the 4 out-of-range units, AND the Hive Mind client receives all 10 unit positions in the same tick.

**AC-NET-08 — Corruption Meter: Server-Only Writes**
GIVEN the Corruption Meter is a `NetworkVariable<float>` with server write authority,
WHEN a Reclaimer or Hive Mind client attempts to directly set the NetworkVariable's value (not via `ModifyCorruptionServerRpc`),
THEN the write is rejected by NGO's ownership check, the value on all clients remains unchanged, and no exception propagates to the game loop.

**AC-NET-09 — Corruption Meter: All Clients Receive Updated Value**
GIVEN the server modifies the Corruption Meter via `ModifyCorruptionServerRpc`,
WHEN the `OnValueChanged` callback fires,
THEN all 5 connected clients each reflect the updated value within one server tick (33.3ms) of the modification being applied, with no client showing a stale value for more than one tick period.

---

### Formulas (Section D)

**AC-NET-10 — Tick Rate: Server Maintains 30 Hz Under Max Load**
GIVEN the server is running a fully connected 5-player InMatch session with 30 active Synth units,
WHEN tick rate is measured over a 10-second window,
THEN the measured tick rate is between 28 Hz and 32 Hz (±6.7% tolerance), and no individual tick period exceeds 50ms.

**AC-NET-11 — Positional Error: Prediction Active at Sprint Speed**
GIVEN a Reclaimer character is moving at maximum sprint speed (6 m/s),
WHEN a server tick fires and a position update is sent,
THEN the client-side prediction system is active, AND the maximum observed positional discrepancy between the client's predicted position and the server-confirmed position (before reconciliation) does not exceed 0.25m.

**AC-NET-12 — Active Unit Cap: Server Rejects Spawn Above 30**
GIVEN 30 Synth units are currently active in the session,
WHEN the Hive Mind client sends a `DeployUnitServerRpc` request,
THEN the server rejects the request without spawning a unit, logs the rejection with reason "unit cap exceeded", and the Hive Mind client receives a failure response. Unit count remains at 30.

**AC-NET-13 — Bandwidth: Host Upload Does Not Exceed Ceiling at Max Load**
*(Post-MVP flag: full hardware measurement deferred to pre-Launch performance testing. In MVP, verify via logged byte counts per tick against the 136.9 KB/s formula output at 30 units.)*
GIVEN a 5-player InMatch session with 30 active Synth units running for 60 continuous seconds,
WHEN host upstream bandwidth is measured at the network interface,
THEN it does not exceed 200 KB/s on a sustained basis (individual tick bursts may spike to 220 KB/s for no more than 3 consecutive ticks).

**AC-NET-14 — Latency Tiers: Matchmaking Blocks Join Above 200ms RTT**
*(Post-MVP flag: requires controllable latency injection for automated testing. Manual geographic test is advisory in MVP.)*
GIVEN a client attempting to join has a measured RTT of 210ms to the host,
WHEN the client sends a connection request,
THEN the server rejects the connection before Connecting state, the client receives a latency-cause rejection message, and the session player count is unchanged.

**AC-NET-15 — Latency Tiers: HUD Warning Fires at 250ms RTT**
GIVEN a client in InMatch has RTT rise above 250ms,
WHEN the server measures the threshold crossing,
THEN the server sends a high-latency notification to that specific client within one tick, the HUD warning becomes visible on that client's screen, and other clients do not receive the warning.

**AC-NET-16 — Latency Tiers: Disconnect Recommendation After 10 Seconds at 350ms**
GIVEN a client has sustained RTT above 350ms for 10 consecutive seconds,
WHEN the window elapses,
THEN the server sends a disconnect recommendation to that client exactly once, the client displays the recommendation, and the server does not force-disconnect the client. If RTT drops below 350ms before 10 seconds, the timer resets and no recommendation is sent.

---

### Session State Transitions

**AC-NET-17 — Lobby Rejects Start With Fewer Than 5 Players**
GIVEN a session is in Lobby state with only 4 clients connected,
WHEN the host triggers match start,
THEN the server rejects the start request, session state remains Lobby, and all connected clients receive a "waiting for players" notification.

**AC-NET-18 — Lobby to Connecting to InMatch Completes**
GIVEN exactly 5 clients are connected with all roles assigned in Lobby state,
WHEN the host triggers match start,
THEN the server transitions Lobby → Connecting → InMatch, spawning all 5 player NetworkObjects and confirming spawns from all clients, completing within 15 seconds on a LAN connection.

**AC-NET-19 — Sector Transition Preserves Corruption Meter**
GIVEN Reclaimers reach the sector exit trigger in InMatch,
WHEN the server enters SectorTransition state,
THEN match logic is paused, no Corruption changes occur, and the Corruption Meter value at transition entry is preserved exactly in the next sector (not reset, not modified).

**AC-NET-20 — Sector Transition Times Out If Client Fails to Load**
GIVEN a session is in SectorTransition and one client has not confirmed load within 30 seconds,
WHEN the timeout elapses,
THEN the server terminates the session for all clients, all clients receive "session ended", and no client is left in a partial-load state.

**AC-NET-21 — InMatch to PostMatch via Win Condition**
GIVEN a win condition fires in InMatch,
WHEN the server processes the win event,
THEN the session transitions to PostMatch, debrief data is broadcast to all clients, and the session does not accept new ServerRpc actions from any client after PostMatch entry.

---

### Edge Cases (Section E)

**AC-NET-22 — Hive Mind Disconnect: 60-Second Reconnect Window**
GIVEN the Hive Mind client disconnects mid-match,
WHEN the server detects the disconnection,
THEN: match pauses immediately; all clients receive "Hive Mind connection lost" within one tick; server waits up to 60 seconds for reconnect; if reconnected in time, match resumes; if not, server awards Reclaimer victory and transitions to PostMatch.

**AC-NET-23 — Host Disconnect: Session Terminates for All Clients**
GIVEN the listen-server host disconnects mid-match,
WHEN the server detects host connection loss,
THEN all clients receive "session ended", all clients return to lobby/main menu state, and no client receives a Hive Mind victory award as a result of the termination.

**AC-NET-24 — Reclaimer Disconnect: Character Removed, Match Continues**
GIVEN 4 Reclaimers are connected and 1 disconnects,
WHEN the server detects the disconnection,
THEN the character NetworkObject is despawned within one tick, the remaining 3 Reclaimers continue without interruption, and the match is not paused. If the final Reclaimer disconnects, the server ends the session without awarding Hive Mind victory.

**AC-NET-25 — Unauthorized ServerRpc: Silently Discarded and Logged**
GIVEN a Reclaimer client sends a ServerRpc for an action on a NetworkObject it does not own,
WHEN the server receives the RPC,
THEN the server discards the request without state change, logs a warning with the requesting clientId and target object reference, and the client receives no error callback that could expose ownership information.

**AC-NET-26 — Corruption Clamping at Boundaries**
GIVEN the Corruption Meter is at 98.0 and the server processes `ModifyCorruptionServerRpc(5.0)`,
WHEN the delta is applied,
THEN the stored value is exactly 100.0 (not 103.0), the action counts as valid (no rollback), and the win condition fires. Separately: GIVEN Corruption is at 2.0 WHEN `ModifyCorruptionServerRpc(-5.0)` is processed THEN the stored value is 0.0, not -3.0.

**AC-NET-27 — Simultaneous Win Triggers: Exactly One Victory**
GIVEN two Hive Mind RPCs arrive in the same tick where both would push Corruption to 100%,
WHEN both are processed in arrival order,
THEN the win condition fires exactly once (from the first RPC), the second RPC is applied to the clamped value with no additional effect, and the session transitions to PostMatch exactly once.

**AC-NET-28 — Bandwidth Throttling Priority: Reclaimer Positions Never Throttled**
*(Post-MVP flag: full verification requires controlled bandwidth saturation. In MVP, verify throttle priority configuration via code review and logged replication decisions.)*
GIVEN host upstream bandwidth exceeds 200 KB/s,
WHEN the server applies automatic throttling,
THEN Reclaimer position updates and Corruption Meter sync are delivered at full 30 Hz, while Hive Mind decorative state updates are reduced first, followed by non-visible unit position frequency for Reclaimers.

## Open Questions

- **NGO 2.X API verification**: The authority model and replication visibility group
  implementation depend on NGO 2.X APIs that are post-cutoff. Specific method
  signatures must be verified against live Unity docs before architecture begins.
  Owner: Lead Programmer. Target: Before first networking story enters Sprint.

- **Relay vs. LAN for VS**: Does the Vertical Slice require remote testers, or will
  LAN playtesting suffice? If remote, evaluate Unity Relay cost and availability.
  Owner: Producer. Target: VS planning session.

- **Host migration at Launch**: No host migration in MVP. For Launch, host migration
  requires full in-flight match state serialization. Is this within Launch scope?
  Owner: Technical Director. Target: Architecture review for Launch tier.

- **Hive Mind latency ceiling**: Should the Hive Mind role be allowed to connect at
  a higher RTT than Reclaimers (since their gameplay is less twitch-sensitive)?
  Owner: Game Designer + Network Programmer. Target: Before matchmaking GDD is authored.

- **Congestion throttle penalty validation**: `T_congestion_penalty = 100ms` is an
  estimate. Validate against real hardware after the networking prototype is built.
