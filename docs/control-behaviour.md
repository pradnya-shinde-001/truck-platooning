# Task 2 — Control Behaviour Specification

## 1. Control goal
Every follower *i* must keep a **safe and short gap** `d_i` to its predecessor *i−1*. The
**disturbances** that must not amplify down the platoon are:
- the leader braking or accelerating
- a cut-in vehicle
- communication loss

## 2. Spacing policy — constant time gap

```
d_des(v) = d_0 + h · v_i
```

| Parameter | Meaning | CACC (comm OK) | Degraded | ACC (radar only) |
|---|---|---|---|---|
| `d_0` | standstill distance | 5 m | 5 m | 8 m |
| `h` | time gap | **0.5 s** | 0.5 → 1.2 s (ramp) | **1.5 s** |
| gap at 80 km/h (22.2 m/s) | | ≈ 16 m | ≈ 16 → 32 m | ≈ 41 m |

Without communication, a follower only sees its predecessor's braking through the radar, which is
delayed by sensor + actuator lag (≈ 0.5–1 s). It therefore needs a much larger time gap to stay
safe.

## 3. Controller

Spacing error: `e_i = d_i − d_des(v_i)` (measured by radar; cross-checked with V2V positions)

**CACC (Cooperative Adaptive Cruise Control):**
```
a_cmd,i = k_p · e_i + k_d · (v_{i−1} − v_i) + k_ff · a_{i−1}
```
- `v_{i−1}` and `a_{i−1}` come from the predecessor's `STATE` message.
- The feed-forward term `k_ff · a_{i−1}` lets a follower brake **at the same time** as its
  predecessor instead of after it. This is what makes short gaps possible.
- Suggested starting gains: `k_p = 0.2 s⁻²`, `k_d = 0.7 s⁻¹`, `k_ff = 1.0`.
  These will be tuned in simulation.

**ACC (fallback):** the same law with `k_ff = 0`, and `v_{i−1}` estimated from the radar's relative speed.

**Saturation:** `a_cmd` is clamped to `[−6 m/s², +1.5 m/s²]`. A rate limit (jerk ≤ 2 m/s³) keeps the
trucks comfortable and protects the cargo.

### Distance guarantee
The gap is safe if a follower can stop behind a predecessor that brakes at full strength, even
allowing for its reaction delay `τ`:
```
d_i ≥ v_i · τ + v_i² / (2·b_i) − v_{i−1}² / (2·b_{i−1}) + d_safe
```
- With V2V, `τ` ≈ 0.1 s (comm latency + actuator), so 0.5 s is enough.
- Radar-only gives `τ` ≈ 1 s, so the gap is widened to 1.5 s.
- A follower whose own braking capability `b_i` is lower (heavy load) is placed **further back**
  in the platoon at join time.

### String stability
A platoon is string-stable when spacing errors **shrink** as they propagate from truck 1 to
truck N, i.e. `|e_{i}(jω)| / |e_{i−1}(jω)| ≤ 1`.
- With feed-forward (CACC) this holds for small `h`.
- Pure ACC needs `h` > 2·τ.

This is the quantitative reason we switch time gaps per mode.

## 4. Control mode state machine
See [`diagrams/04_control_mode_sm.drawio`](../diagrams/03_control_mode_sm.drawio). The first-version diagram shows the core modes; `TunnelMode` from the table below will be added in a later version.

| Mode | Entry condition | Behaviour |
|---|---|---|
| `Manual` | not in platoon / driver control | no automation |
| `CACC` | in platoon, link `Connected`, radar OK | h = 0.5 s, feed-forward on |
| `DegradedCACC` | link `Degraded` | `h` ramps up to 1.2 s at 0.1 s per second; feed-forward weighted by data freshness |
| `ACC` | link `Lost` or data age > 200 ms | radar only, h = 1.5 s, k_ff = 0 |
| `TunnelMode` | `TUNNEL_AHEAD` received | widen gap to ACC distance **before** entering the tunnel |
| `EmergencyBrake` | `EMERGENCY_BRAKE` received **or** radar TTC < 1.5 s | full braking, re-broadcast event |
| `SafeStop` | radar fault while comm lost, or actuator fault | controlled decel 2 m/s² to standstill on hard shoulder, hazard lights, notify leader/operator |

**Guard rules for robustness:**
1. **Monotonic safety:** a mode change may only *increase* the target gap immediately. Gaps are
   decreased slowly by ramping `h` down, so there is no setpoint jump that would cause a hard
   acceleration.
2. **Hysteresis:** going back up (ACC → CACC) needs `M_recover` = 2 s of stable link. This avoids
   flickering between modes on a borderline radio channel.
3. **Independent safety layer:** the radar-based time-to-collision (TTC) check runs in **every**
   mode and overrides the controller. The system stays safe even if the V2V data is wrong or forged.
4. **Fail-safe default:** any unknown or inconsistent state, e.g. a V2V position that disagrees
   with the radar gap by > 5 m, is treated as `ACC`.

## 5. Failure scenarios

| # | Failure | Detection | Reaction | Still stable? |
|---|---|---|---|---|
| F1 | Single packets lost | `seq_no` gap | use last value, extrapolate ≤ 200 ms | yes, no mode change |
| F2 | Link degraded (interference) | 3 missed heartbeats | `DegradedCACC`, gap ramps up | yes, h grows |
| F3 | Link lost (jamming, range) | 10 missed heartbeats | `ACC`, radar only, h = 1.5 s | yes, ACC string-stable at h > 2τ |
| F4 | Tunnel (predictable loss) | `TUNNEL_AHEAD` | gaps widened **before** loss | yes, no transient at all |
| F5 | Radar failure, comm OK | sensor bitmask | keep CACC using V2V gap, leader asks truck to leave at next exit | yes, with reduced redundancy |
| F6 | Radar **and** comm lost | both | `SafeStop` | safe by construction |
| F7 | Leader truck fails / driver not ready | leader heartbeat lost or `driver_ready = false` | followers → ACC, then **leader election** (see below) | yes |
| F8 | Cut-in vehicle | radar sees new object, gap ≪ expected | `SPLIT_REQ`: truck *i* opens gap behind the intruder, platoon runs as two sub-platoons, `MERGE_REQ` when clear | yes |
| F9 | Spoofed brake command | HMAC invalid | message dropped, event logged | yes |

## 6. ✨ Leader election and rotation
When the leader is lost (F7), a Bully-style election runs:
- Each follower computes a **health score** from sensor status, braking capability, and
  `driver_ready` (only trucks with a driver can lead on public roads).
- Each follower then sends `ELECTION` to higher-scored trucks.
- The highest healthy candidate broadcasts `COORDINATOR`.
- Ties are broken by lower `position_idx`, so the leader stays at the front.
- If **no** truck with a driver is available, the platoon goes to `SafeStop`.

**Planned rotation:** the lead truck burns roughly 5–10 % more fuel than the followers, which
drive in its slipstream. With `HANDOVER_REQ`, two trucks with drivers can swap the lead role every
*X* km. The old leader leaves, re-joins at the tail, and the next driven truck becomes leader.

## 7. Black-box logging
Every state transition is appended to a per-truck ring buffer:
`(timestamp, lamport_ts, old_state, new_state, trigger, gap, v)`.
This supports post-incident analysis and will help debug the concurrent implementation in
Task 3.

## 8. Related models
- [04_control_mode_sm.drawio](diagrams/04_control_mode_sm.drawio): control mode state machine
- [05_emergency_brake_activity.drawio](diagrams/05_emergency_brake_activity.drawio): emergency brake activity diagram

Each `.drawio` file has a `.png` export next to it in the same folder.
