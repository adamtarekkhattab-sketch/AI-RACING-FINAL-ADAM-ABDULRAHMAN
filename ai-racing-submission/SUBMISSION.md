# SUBMISSION.md — IBM AI Racing League

## Team / Submission identification

| Field | Value |
|---|---|
| Submission script | `snakeoil3_v2.py` |
| Track | michigan (oval, 3608.2 m) |
| Mode | Practice |
| Python version | 3.11.8 |

---

## Best recorded result

| Metric | Value |
|---|---|
| **Lap time** | **1:55:99** |
| **Damage** | **0** |
| Experiment run | Run 8 (see `experiment-report.md`) |
| Script used for that run | `snakeoil3_v2_experiment2.py` (logic subsequently merged into `snakeoil3_v2.py`) |
| vs. original baseline | −44:54 faster than the 2:40:53 baseline |

---

## What the submission script does

`snakeoil3_v2.py` is a rule-based TORCS client. It connects over UDP to a running TORCS instance using the SCR server protocol and drives the car using sensor readings only (no vision, no learned model).

**Core control logic (`drive_example` function):**

### Steering

- `steer = angle × 15/PI` — aligns heading to track direction.
- `steer -= trackPos × gain` — steers back toward track centre.
- `gain = 0.40` inside two danger zones (corners 9 and 16, dist 2410–2515 m and 3226–3401 m).
- `gain = 0.10` everywhere else.

The higher gain in danger zones was the fix that ended a 4-run crash sequence at those corners. The root cause was confirmed as a steering failure (car sliding off the 9 m road at low speed), not a speed failure.

### Braking

Two reactive brake layers, both based on `track[9]` (centre-ahead distance sensor):

| Layer | Condition | Response |
|---|---|---|
| Emergency | `forward_clear < 35 m` AND `speedX > 125 km/h` | Full brake up to 1.0; zero throttle |
| Approach | `forward_clear < 80 m` AND `speedX > 105 km/h` | Partial brake up to 0.6; zero throttle |

### Throttle

When not braking, throttle target = `min(300, corner_9_cap, corner_16_cap, max(0, forward_clear) × 5.0)` km/h.

- K=5.0 means a straight with 200 m of clearance targets 300 km/h; a 22 m corner targets 110 km/h.
- `max(0, forward_clear)` prevents negative sensor readings from producing a negative target.
- Inside danger zones: `throttle_floor = 30 km/h` so the car stays steerable when the sensor reads ≤ 0.
- Corner 9 and 16 throttle caps: 92 km/h, preventing overrun at the tightest apexes.
- `accel` steps up by +0.03 per tick when below target; steps down by −0.02 when above.
- If `speedX < 10`, adds `1 / (speedX + 0.1)` for launch-from-rest.

### Traction control

If rear wheel spin exceeds front by > 3 rad/s, `accel -= 0.2`.

### Gearbox (speed-threshold)

| Gear | Threshold |
|---|---|
| 1 | default |
| 2 | speedX > 60 km/h |
| 3 | speedX > 80 km/h |
| 4 | speedX > 110 km/h |
| 5 | speedX > 140 km/h |
| 6 | speedX > 170 km/h |

No downshift logic. Gear is re-evaluated from scratch each step.

### Telemetry

Every step, one CSV row is appended to `lap_data.csv`: `distFromStart, speedX, trackPos, angle, steer, accel, brake, gear, track_ahead, damage`.

---

## Experiment log summary

Full notes are in `experiment-report.md`. Summary of every run:

| Run | Script | Lap Time | Damage | Key change |
|---|---|---|---|---|
| Baseline | snakeoil3_v2.py | 2:40:53 | 0 | — original steer-penalty throttle |
| Run 1 | snakeoil3_v2_experiment.py | 2:46:40 | 0 | K=2.5 throttle; wider brake windows |
| Run 2 | snakeoil3_v2_experiment.py | 2:27:50 | 0 | Emergency brake floor 50→80 km/h |
| Run 3 | snakeoil3_v2_experiment.py | 2:23:90 | 0 | K 2.5→3.5 |
| Run 4 | snakeoil3_v2_experiment.py | 2:01:03 | 0 | Approach brake floor 70→105 km/h; K 3.5→4.2 |
| Run 5 | snakeoil3_v2_experiment.py | DNF | 23 | Emergency brake floor 80→125 km/h; K 4.2→5.0 |
| Run 6 | snakeoil3_v2_experiment.py | DNF | 24 | Corner 9 & 16 entry brake zones; caps 95 km/h |
| Run 7 | snakeoil3_v2_experiment.py | DNF | 27 | Entry zones removed; caps 92 km/h; sensor floor guard |
| **Run 8** | snakeoil3_v2_experiment2.py | **1:55:99** | **0** | **Steer-to-center 0.10→0.40 in danger zones; 30 km/h throttle floor** |
| v3 Run 1 | snakeoil3_v3.py | 1:57:08 | 13 | Wider sensors; wall bias correction; track memory brake |
| v3 Run 2 | snakeoil3_v3_experiment.py | 1:57:55 | 1 | Wall bias gated; corner 9 cap 92→75 km/h; off-track recovery |
| v3 RPM Test | snakeoil3_v3_rpm_test.py (deleted) | 2:03:00 | 26 | RPM gearbox (broken — gear state not persisted) |
| v3 Exp 3 | snakeoil3_v3_experiment3.py | 1:58:00 | 0 | Corner 9 early brake zone (dist 2300–2410 m) |
| v3 Exp 4 | snakeoil3_v3_experiment4.py | not recorded | not recorded | Corner 12 early brake zone (dist 2520–2647 m) |
| v4 Last Test | snakeoil3_v4_last_test.py | not recorded | not recorded | Degenerate apex zone fix; adaptive angle gain; approach throttle 0.15 |

---

## Key findings from tuning

1. **Steer-to-center was the crash root cause.** A global gain of 0.10 produces only +0.10 correction at trackPos = −1.0 — not enough in a 9 m road gap. Raising to 0.40 inside the two danger zones stopped 4 consecutive crashes immediately (Run 8).

2. **Throttle cap cannot prevent a speed crash.** A cap only limits what the car chases; it cannot slow a car already travelling above the cap on entry. Entry speed must be handled by a brake zone, not a throttle limit.

3. **Sensor-based throttle (K×forward_clear) outperforms steer-penalty throttle.** The steer-penalty formula (`300 − steer×80`) produced a dead-zone where neither brake nor throttle was active mid-corner. K=5.0 keeps the car actively targeting a speed at all times.

4. **Wider brake windows hurt more than they help.** Widening the approach distance from 50→80 m caused the brake to fire at the beginning of straight exits, scrubbing speed unnecessarily (Run 1 was slower than baseline).

5. **max(0, forward_clear) guard is essential.** When the sensor reads a negative value (car's nose partially past a wall), an unguarded target formula can produce a negative throttle target, stalling the car (discovered in Run 6).

---

## Remaining known limitations

- No downshift logic. After braking into a corner the car stays in whichever gear it was in, with reduced torque on exit.
- RPM-based gearbox tested but not working (gear state resets each step — requires persistent module-level variable).
- Straight-line speeds on straights 1, 4, and 8 underperform (123–167 km/h). Estimated 6+ seconds still available.
- v3 track memory anticipatory braking adds net overhead vs reactive-only logic — fires before corners already handled smoothly, scrubbing straight-line speed.
