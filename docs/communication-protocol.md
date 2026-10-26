# Task 1 — Inter-Truck Communication Protocol

## 1. Assumptions
- The project is a **software simulation**: trucks, sensors and radio links are simulated; nothing controls a physical vehicle.
- Platoon of 1 lead truck (human driver) + N followers (default N = 3, max 8).
- Each truck has: front radar (distance + relative speed to predecessor), GNSS, speed/acceleration
  sensors, brake/throttle actuators and a V2V radio (broadcast, ~300 m range, e.g. ITS-G5 / C-V2X).
- The wireless channel is **unreliable**: messages can be lost, delayed, duplicated or reordered.
- Clocks are loosely synchronised via GNSS; logical ordering of events uses **Lamport timestamps**.
- Platoon position index: leader = 0, followers = 1 … N (in driving order).

## 2. Transport: UDP

| Layer | Choice |
|---|---|
| Application | **Our platooning protocol** (this document, §4–§9) |
| Transport | **UDP**, connectionless broadcast/unicast datagrams |
| Radio (real deployment) | IEEE 802.11p / ITS-G5 or C-V2X (assumed, not designed by us) |
| Radio (our simulation) | UDP over localhost, one socket per simulated truck |

**Why UDP and not TCP**
- Status updates are periodic (20 Hz). A lost `STATE` message is already outdated when it could be resent,
  so TCP's retransmission and in-order delivery would only add delay (head-of-line blocking).
- No connection setup is needed. Trucks can broadcast to the whole platoon, which fits the
  one-to-many nature of V2V and matches how real V2X radios work.
- UDP makes communication faults (loss, delay, duplication, reordering) visible to the
  application, so the simulation can **inject** them and test the fault handling of Task 2.

**What our protocol adds on top of UDP**, because UDP itself guarantees nothing:

| UDP does not guarantee… | Our protocol handles it with… |
|---|---|
| delivery | ACK + retransmission for all event messages; flooding for `EMERGENCY_BRAKE` (§6) |
| ordering | `seq_no` per sender; Lamport timestamps for membership events (§4, §8) |
| no duplicates | `seq_no` ≤ last seen → drop (§4) |
| freshness | `timestamp_ms`, data older than 200 ms is discarded (§4) |
| authenticity | HMAC over header + payload (§4, §9) |
| link health | heartbeat timeouts → Connected / Degraded / Lost (§7) |

## 3. Signals and data exchanged

| Category | Signal | Unit | Producer → Consumer | Why it is needed |
|---|---|---|---|---|
| Kinematics | position (lat, lon) | deg | every truck → all | ordering, gap cross-check vs radar |
| | speed `v` | m/s | every truck → successor | speed-matching term of controller |
| | acceleration `a` | m/s² | every truck → successor | feed-forward (string stability) |
| | heading | deg | every truck → all | lane / curve awareness |
| Intent | planned acceleration `a_plan` | m/s² | leader → all | all followers react at once, not one-by-one |
| | lane change intent | enum | leader → all | coordinated lateral manoeuvre |
| Health | heartbeat | — | all ↔ all | link-loss detection |
| | sensor status (radar, GNSS, brakes) | bitmask | all → leader | platoon-wide health picture |
| | driver ready (leader only) | bool | leader → all | triggers leader handover if false |
| Events | see §5 | — | event-driven | membership & safety |

## 4. Message format

All messages share a fixed 24-byte header followed by a type-specific payload.

| Field | Size | Description |
|---|---|---|
| `version` | 1 B | protocol version (= 1) |
| `msg_type` | 1 B | see §5 |
| `platoon_id` | 2 B | 0 = not in a platoon |
| `sender_id` | 4 B | unique truck ID |
| `position_idx` | 1 B | 0 = leader, 1…N followers, 0xFF = standalone |
| `flags` | 1 B | bit0 = ACK requested, bit1 = retransmission, bit2 = tunnel mode |
| `seq_no` | 4 B | per-sender counter → detects loss, duplicates, replays |
| `lamport_ts` | 4 B | logical clock for event ordering |
| `timestamp_ms` | 4 B | GNSS time → detects stale data (age > 200 ms is discarded) |
| `payload_len` | 2 B | bytes following the header |
| *payload* | var | type-specific |
| `hmac` | 8 B | truncated HMAC-SHA256 over header + payload (shared platoon key) |

Receivers **drop** a message when the HMAC is invalid, when `seq_no` ≤ last seen from that sender
(replay or duplicate), or when the data is older than 200 ms.

## 5. Message types

| ID | Name | Channel | Payload | Sent when |
|---|---|---|---|---|
| 0x01 | `STATE` | periodic | pos, v, a, heading, a_plan | every 50 ms (20 Hz) |
| 0x02 | `HEARTBEAT` | periodic | sensor bitmask, driver_ready | every 100 ms; can be piggy-backed on `STATE` |
| 0x10 | `JOIN_REQ` | reliable | truck capabilities, current pos | a standalone truck wants to join |
| 0x11 | `JOIN_ACK` | reliable | assigned `platoon_id`, `position_idx`, gap | leader accepts |
| 0x12 | `JOIN_NACK` | reliable | reason (full, incompatible, unsafe) | leader rejects |
| 0x13 | `LEAVE` | reliable | position_idx | truck leaves (exit, destination) |
| 0x14 | `SPLIT_REQ` | reliable | split position, gap to open | cut-in detected / requested |
| 0x15 | `MERGE_REQ` | reliable | — | cut-in vehicle has left the gap |
| 0x16 | `TOPOLOGY` | reliable | ordered list of member IDs | after any membership change |
| 0x20 | `EMERGENCY_BRAKE` | reliable, high priority | decel target | leader or any truck detects a hazard |
| 0x21 | `TUNNEL_AHEAD` | reliable | distance, expected outage duration | map / V2I says comm loss is coming |
| 0x30 | `ELECTION` | reliable | candidate health score | leader lost |
| 0x31 | `COORDINATOR` | reliable | new leader ID | election won |
| 0x32 | `HANDOVER_REQ` | reliable | proposed new leader | planned rotation / driver fatigue |
| 0x7F | `ACK` | — | acked `seq_no`, msg_type | response to any message with bit0 set |

## 6. Reliability

- **Periodic messages** are best-effort broadcasts. Losing one is acceptable because the next one
  arrives 50 ms later.
- **Event messages** use ACK + retransmission:
  - The sender retransmits every 20 ms until it has an ACK from every addressed member, with at most 5 attempts.
  - `EMERGENCY_BRAKE` is additionally **flooded**: each follower re-broadcasts it once, so it still
    reaches truck 3 when the leader → truck 3 link is weak.
  - If ACKs are still missing after 5 attempts, that member is marked `Degraded` and the leader
    widens the platoon gaps.

## 7. Timing and timeouts

| Parameter | Value | Meaning |
|---|---|---|
| `T_state` | 50 ms | STATE broadcast period |
| `T_hb` | 100 ms | heartbeat period |
| `N_degraded` | 3 missed heartbeats (300 ms) | link → **Degraded** |
| `N_lost` | 10 missed heartbeats (1 s) | link → **Lost** |
| `M_recover` | 20 consecutive heartbeats (2 s) | link → **Connected** again (hysteresis) |
| `T_ack` | 20 ms | retransmission interval |
| `T_election` | 300 ms | wait for higher-ranked candidate in leader election |
| `max_age` | 200 ms | maximum age of usable kinematic data |

## 8. Event ordering with Lamport clocks

Every truck keeps a counter `L`:
- It does `L = L + 1` before sending, and stamps the message with `L`.
- On receive, it does `L = max(L, msg.lamport_ts) + 1`.

Membership changes are applied in `(lamport_ts, sender_id)` order. This resolves races such as two
trucks sending `JOIN_REQ` at the same time, or a `SPLIT_REQ` crossing a `LEAVE`.

## 9. Security

| Threat | Countermeasure |
|---|---|
| Spoofed `EMERGENCY_BRAKE` from a fake truck (phantom braking) | HMAC with a per-platoon key distributed in `JOIN_ACK` |
| Replay of an old recorded message | strictly increasing `seq_no` per sender + `max_age` |
| Eavesdropping on the platoon key in `JOIN_ACK` | *Assumption:* the key is encrypted with the joining truck's pre-installed fleet certificate (PKI is out of scope) |
| Jamming | detected as link loss → safe fallback to radar-only ACC (see control.md) |

## 10. Test scenarios

Each scenario is run in the simulation (with fault injection where needed). The check is that the
receiver detects the situation and follows a valid transition in the state machines.

| # | Scenario | Expected behaviour |
|---|---|---|
| T1 | Normal `STATE` + heartbeat reception | link stays `Connected`, followers in CACC |
| T2 | Single delayed `STATE` (> 200 ms old) | message discarded as stale, no mode change |
| T3 | Duplicate / out-of-order message | dropped via `seq_no` |
| T4 | Packet loss: 3 heartbeats missed | link → `Degraded`, gap widens |
| T5 | Packet loss: 10 heartbeats missed | link → `Lost`, radar-only ACC |
| T6 | Recovery after loss | `Reconnecting`, back to `Connected` only after 2 s of stable heartbeats |
| T7 | `EMERGENCY_BRAKE` with 30 % packet loss | still reaches all followers via retransmission + flooding |
| T8 | Spoofed `EMERGENCY_BRAKE` (wrong HMAC) | dropped and logged, no braking |
| T9 | Replayed old message | dropped via `seq_no` / `max_age` |
| T10 | Truck joins at the tail | `JOIN_REQ` → `JOIN_ACK` → `TOPOLOGY` update to all members |
| T11 | Two trucks join at the same time | ordered by `(lamport_ts, sender_id)`, both get distinct positions |
| T12 | Truck leaves the platoon | `LEAVE` → `TOPOLOGY` update, gap closes behind |
| T13 | Cut-in vehicle | `SPLIT_REQ` → gap opened → `MERGE_REQ` when clear |
| T14 | `TUNNEL_AHEAD` received | gaps widen **before** communication is lost |
| T15 | Leader heartbeat lost | followers → ACC, election, `COORDINATOR` broadcast |

## 11. Related models
- [`diagrams/01_membership_sm.drawio`](../diagrams/01_membership_sm.drawio): platoon membership state machine
- [`diagrams/02_comm_link_sm.drawio`](../diagrams/02_comm_link_sm.drawio): communication link health state machine
- [`diagrams/04_leader_election_sm.drawio`](../diagrams/04_leader_election_sm.drawio): leader election state machine
- [`diagrams/06_join_sequence.drawio`](../diagrams/06_join_sequence.drawio): join handshake sequence diagram

Each `.drawio` file has a `.png` export next to it in the same folder.

## 12. Creative concept: Platoon Guardian — a self-protecting truck convoy

The platoon protects itself: every truck checks whether the information it receives can be
trusted, warns the others immediately when danger appears, and the convoy reorganises itself
when a member fails. The concept has three building blocks.

### 12.1 Heartbeat and message freshness monitoring
Every status message carries a `timestamp_ms` and a `seq_no` (§4). Each following truck uses
them to detect messages that are:
- **missing** (a gap in `seq_no`, or no heartbeat within the timeout)
- **delayed** (older than `max_age` = 200 ms)
- **duplicated** (`seq_no` already seen)
- **out of order** (`seq_no` lower than the last one received)

Missed heartbeats drive the link-health state machine `Connected → Degraded → Lost → Reconnecting` (§7).

**Why it helps:** it shows distributed-system failure detection, and a truck never blindly
trusts stale information when deciding how close to drive.

### 12.2 Emergency braking alerts
When the leader (or any truck) detects a hazard, it brakes and broadcasts `EMERGENCY_BRAKE` (§5).
- The alert is high priority and is acknowledged and retransmitted until every member has ACKed it (§6).
- Each follower re-broadcasts it once (flooding), so it still reaches the tail truck even if the
  direct link from the leader is weak.
- Alerts with an invalid HMAC or a replayed `seq_no` are rejected, so a fake truck cannot trigger
  phantom braking (§9).

**Why it helps:** all followers brake at the same time instead of one after another, which is
what makes short gaps safe. It also shows reliable delivery built on top of unreliable UDP.

### 12.3 Self-healing platoon
The platoon repairs itself when something goes wrong, without a central server:
- **Leader failure:** if the leader's heartbeat is lost, followers fall back to radar-only ACC
  and elect a new leader (`ELECTION` → `COORDINATOR`, Bully-style, §5).
- **Member failure or departure:** the leader removes the truck and broadcasts a new `TOPOLOGY`.
  The truck behind it closes the gap to its new predecessor.
- **Cut-in vehicle:** the platoon splits (`SPLIT_REQ`) and merges again (`MERGE_REQ`) when the gap is clear.
- **Link recovery:** after a communication loss, a truck rejoins full cooperative driving only
  after 2 s of stable heartbeats (hysteresis), so it does not flicker between modes.

**Why it helps:** it demonstrates fault tolerance and leader election, which are core
distributed-systems concepts. The convoy keeps operating safely when individual trucks or
links fail.
