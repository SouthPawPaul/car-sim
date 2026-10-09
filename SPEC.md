# Ring Road: Phantom Traffic Jam Simulator — Spec v2

## 1. Introduction

This app simulates cars driving around a circular, single-lane track. The goal is to reproduce the phantom traffic jam experiment of Sugiyama et al. (2008). In that experiment, cars on a ring road with no bottleneck formed stop-and-go waves on their own.

Our version tests a narrower claim: **one erratic driver is enough to turn smooth traffic into a stop-and-go jam.**

Jams like this need imperfect drivers. If every car responded instantly and perfectly, a disturbance would fade out. So each car follows a car-following model with a reaction delay (see Section 5). A small slowdown is then amplified by each car behind it, and the jam travels backwards around the track.

## 2. Units, scale and constants

The simulation runs in SI units (metres, seconds). The interface shows speeds in mph and distances in car lengths.

| Constant | Value | Notes |
| --- | --- | --- |
| `CAR_LENGTH_M` | 4.5 m | Typical family car |
| `CAR_WIDTH_M` | 2.25 m | Keeps the 20 × 40 px ratio |
| `PX_PER_M` | 40 / 4.5 ≈ 8.889 | 40 px = one car length |
| `MPH_TO_MS` | 0.44704 | |
| `REFERENCE_SPEED` | 30 mph (13.41 m/s) | |
| `REFERENCE_LAP_TIME` | 20 s | One lap at 30 mph |
| `TRACK_LENGTH_M` | 13.4112 × 20 ≈ 268.2 m | Derived from the two above |
| `SIM_DT` | 1/60 s | Fixed physics timestep |
| `MAX_CARS` | Derived, see 6.3 | 39 with default settings |

## 3. The track

- The track is a circle. Its centre-line circumference is `TRACK_LENGTH_M` (≈ 268.2 m).
- Centre-line radius = 268.2 / 2π ≈ 42.69 m ≈ **379.5 px**.
- Track width = car width + 20% of car width on each side = 20 + 4 + 4 = **28 px**.
- Overall outer diameter ≈ 787 px. Design for an 800 × 800 px canvas, and scale the whole canvas uniformly to fit the viewport. Scaling is visual only; simulation distances never change.
- One lane, no overtaking. Cars travel anticlockwise.
- Colours: neutral grey track (e.g. `#5a5a5a`) on a lighter background (e.g. `#e8e8e8`), with a dark theme variant.

## 4. The car

### 4.1 Appearance

- A rectangle of 20 px wide by 40 px long, with corner radius 4 px.
- Solid fill in a random colour, chosen in HSL with saturation 60–80% and lightness 45–60%. This keeps every car visible against the track.
- Drawn centred on the track centre-line and rotated so its long axis follows the direction of travel.
- The **selected car** gets a 2 px white outline with a dark shadow.
- **Erratic cars** (speed variability > 0) show a small "!" badge, or a pulsing outline, so they stand out from the random colours.

### 4.2 Per-car parameters

Every car has its own parameters. A new car takes the current defaults, which can be edited in the interface (Section 7).

| Parameter | Unit | Default | Range | Precision |
| --- | --- | --- | --- | --- |
| Desired speed | mph | 30.0 | 5.0–70.0 | 1 dp |
| Speed variability | mph | 0.0 | 0.0–30.0 | 1 dp |
| Minimum distance (standstill gap) | car lengths | 0.5 | 0.1–10.0 | 1 dp |
| Time gap | s | 1.2 | 0.5–3.0 | 1 dp |
| Reaction time | s | 0.8 | 0.0–2.0 | 1 dp |

**Minimum distance** is the bumper-to-bumper gap a car keeps when stopped. A moving car also keeps the time gap, so its distance grows with speed.

> **Changed from v1:** v1 set the minimum distance default to 4 car lengths (18 m). With a fixed 18 m gap plus a speed-based gap, only about 6 cars can cruise at 30 mph on this track. That is too sparse for a jam to form. The default is now 0.5 car lengths plus a 1.2 s time gap, which allows about 11 cars at 30 mph and up to 39 cars in total.

## 5. Driver behaviour

### 5.1 Car-following model

Each car uses the **Intelligent Driver Model (IDM)**. Every timestep its acceleration is:

```
a = a_max · [ 1 − (v / v0)^4 − (s* / s)^2 ]

s* = s0 + v·T + (v · Δv) / (2 · √(a_max · b))
```

| Symbol | Meaning | Value |
| --- | --- | --- |
| `v` | Car's current speed | m/s |
| `v0` | Car's current desired speed (see 5.3) | m/s |
| `s` | Gap to the car in front (rear bumper to front bumper) | m |
| `Δv` | `v − v_leader` (positive when closing in) | m/s |
| `s0` | Minimum distance × `CAR_LENGTH_M` | m |
| `T` | Time gap | s |
| `a_max` | Maximum comfortable acceleration | 2.0 m/s² |
| `b` | Comfortable braking | 3.0 m/s² |
| `b_max` | Emergency braking limit | 8.0 m/s² |

Acceleration is clamped to the range `[−b_max, a_max]`. Speed never goes below 0 (no reversing).

These values match a typical family car. Everyday acceleration is around 2 m/s², firm-but-comfortable braking is around 3 m/s², and an emergency stop is around 8 m/s².

### 5.2 Reaction time

Each car reacts to what its leader was doing `reaction time` seconds ago, rather than right now.

- Store a rolling history of each car's position and speed for at least 2 s.
- When calculating `s` and `Δv`, use the leader's state from `t − reaction_time`. Interpolate between stored samples.
- The car's own `v` is its current speed. Drivers know their own speed; they are late noticing everyone else's.

Reaction time is the main source of instability. At 0 s, disturbances should mostly die out. At 0.8 s and above, they should grow.

### 5.3 Speed variability (the erratic driver)

A car with speed variability above 0 keeps changing its desired speed `v0`:

1. Every 3–8 s (random, uniform), pick a new target: `desired speed ± random(0, speed variability)`. Clamp it to at least 0.
2. Move `v0` towards the new target over about 2 s, rather than jumping.
3. The car's real speed then follows through the IDM, so it accelerates and brakes within the limits in 5.1.

A car with variability 0 keeps `v0` fixed at its desired speed.

### 5.4 No collisions

IDM with a reaction delay can occasionally predict a gap that's too small. As a safety net:

- If the **actual** (non-delayed) gap falls below `s0`, apply `b_max` immediately.
- After each step, if two cars overlap, move the rear car back to exactly `s0` behind its leader and set its speed to the leader's speed.
- Count these interventions in a debug counter. With sensible parameters, it should stay at or near 0.

## 6. Simulation lifecycle

### 6.1 Loop

- Physics runs at a fixed `SIM_DT` of 1/60 s, using an accumulator so behaviour doesn't depend on frame rate. Rendering uses `requestAnimationFrame`.
- Each car's state is: position along the centre-line (m, wrapped to `[0, TRACK_LENGTH_M)`), speed (m/s), current `v0`, and history buffer.
- Each step, calculate every car's acceleration first, then update every car. Updating in place would let earlier cars affect later ones within the same step.
- Integration: semi-implicit Euler (`v += a·dt`, then `x += v·dt`).

### 6.2 Initial state

The app starts with 0 cars, and the simulation is running.

### 6.3 Adding a car

- `MAX_CARS = floor(TRACK_LENGTH_M / (CAR_LENGTH_M + default s0))`. With defaults this is floor(268.2 / 6.75) = 39. Recalculate it whenever the default minimum distance changes.
- A new car goes in the middle of the **largest gap** on the track.
- The car is only added if that gap is at least `CAR_LENGTH_M + 2 · s0`, so it has `s0` clearance on both sides.
- Its starting speed is the average speed of the cars on either side of the gap, or the desired speed if the track is empty.

### 6.4 Removing a car

- Removes the **selected car** if there is one, otherwise the most recently added car.
- If the removed car was selected, clear the selection.

### 6.5 Reset

Removes all cars, clears the selection, resets the clock to 0, and resets default parameters to the values in 4.2.

## 7. Interface

### 7.1 Controls

| Control | Behaviour |
| --- | --- |
| **Start / Pause** | Toggles the simulation. Label shows the action available. |
| **Reset** | See 6.5. |
| **Add** | Adds a car (6.3). Disabled when the count is `MAX_CARS` or no gap is big enough. |
| **Remove** | Removes a car (6.4). Disabled when there are no cars. |
| **Car count** | Label, e.g. "Cars: 12 / 39". |

### 7.2 Car parameters

- **Defaults panel.** Edits the parameters in 4.2 for new cars. Includes an **"Apply to all cars"** button.
- **Selected car panel.** Clicking a car selects it and shows its parameters for editing. Clicking empty space deselects. Includes a **"Make erratic"** button, which sets that car's speed variability to 10.0 mph.
- Number inputs enforce the ranges and 1 dp precision in 4.2. Invalid input reverts to the last valid value.

### 7.3 Seeing the jam

- **Colour by speed** (toggle, off by default). When on, car fill ranges from green at desired speed to red at standstill. The erratic badge stays visible.
- **Live readouts:** simulation time, average speed (mph), slowest car speed (mph), and number of cars below 5 mph.
- **Time–space diagram** (nice-to-have). A rolling 60 s plot with position around the track on the y-axis and time on the x-axis, one dot per car per sample, coloured by speed. A jam appears as a red band sloping against the direction of travel.

## 8. Acceptance criteria

1. One car at 30 mph with 0 variability completes a lap in 20.0 s (± 0.1 s).
2. Cars never visibly overlap, at any parameter setting within the allowed ranges.
3. With 20 cars, all at variability 0, the speeds settle and stay steady with no stop-and-go waves.
4. With the same 20 cars, setting one car to 10.0 mph variability causes a stop-and-go wave within 2 minutes. At least one car drops below 5 mph, and the slow region moves backwards around the track.
5. Add and Remove are enabled and disabled according to 7.1.
6. Behaviour is the same at 30 fps and 144 fps.

Criterion 4 may need tuning. If no jam forms, increase the reaction time or the number of cars first.

## 9. Open questions

- Should the app start with some cars already on the track (e.g. 20, evenly spaced) so the experiment can begin immediately?
- Should reaction time be shown in the interface, or kept as a fixed hidden value?
- Does this need to work on phones? The design size of 800 × 800 px will scale down, but cars would be quite small.

## References

- Sugiyama, Y. et al. (2008). *Traffic jams without bottlenecks — experimental evidence for the physical mechanism of the formation of a jam.* New Journal of Physics, 10, 033001.
- Treiber, M., Hennecke, A. & Helbing, D. (2000). *Congested traffic states in empirical observations and microscopic simulations.* Physical Review E, 62, 1805. (Intelligent Driver Model)
