# Experiment Report — TORCS Bot Tuning

---

## Run 1 — Baseline

| Field | Details |
|---|---|
| **Baseline** | `snakeoil3_v2.py` (BOB-v1.2 Balanced v2). Lap time **2:40:53**, damage **0**. Track: 3608.2 m, 16 corner zones, 10 straight zones. Corner min speeds clustered at 44–57 km/h across 13 of 15 zones. |
| **Hypothesis** | The car is not throttling or braking when turning through corners. At apex, speed drops to 44–57 km/h but neither the brake nor throttle block is actively firing — the car drifts in a control dead-zone. Root cause: throttle target = `300 − (steer × 80)`, which at typical corner steer (0.3–0.6) still resolves to 240–276 km/h — always above actual speed — so accel increments +0.03/tick. Meanwhile brake conditions (`speedX > 50` / `speedX > 70`) become false once the car slows, so both systems go silent simultaneously. |
| **AI recommendation** | Card 2 (medium risk): replace steer-penalty throttle target on line 563 with `min(target_speed, forward_clear × 2.0)`. Single-line change, original brake thresholds intact. Predicted lap time ~2:25–2:30, damage 0. |
| **Change made** | Not implemented — Card 3 (aggressive version) was chosen instead: K=2.5 throttle target + emergency brake widened `35→50 m` + approach brake widened `60→80 m`. Three lines changed in `snakeoil3_v2_experiment.py`. |
| **Predicted result** | Lap time ~2:18–2:25 · Damage 0–200 (risk at corner 7, min ahead −1.0 m). |
| **Actual lap time** | **2:46:40** — 6 seconds *slower* than baseline. |
| **Damage** | **0** — clean lap. |
| **What surprised us?** | Wider brake windows hurt the straights more than the corner gains helped. Straight 1 dropped from 123.6 → 114.3 km/h, straight 6 from 189.6 → 185.0 km/h. The approach brake at `< 80 m` was firing at the beginning of straight exits, scrubbing speed unnecessarily. The one genuine improvement: the old 292 m compound corner 7 split into two short zones with a new 83.9 m straight at 146 km/h in between — proving the `track_ahead`-based throttle correctly found an opening the old steer-penalty logic missed entirely. |
| **Keep / modify / revert** | **Modify** — revert the wider brake windows, keep the K-based throttle idea. The two changes were bundled; they need to be separated. |
| **Next experiment** | Revert emergency brake to `< 35 m` but raise speed floor to `> 80 km/h` to allow the car to carry 76+ km/h through wider corners. Keep approach brake at `< 80 m / > 70 km/h` for now. Keep K=2.5. |

---

## Run 2 — Emergency brake speed floor raised to 80 km/h

| Field | Details |
|---|---|
| **Baseline** | Run 1 result: `snakeoil3_v2_experiment.py`, lap time 2:46:40, damage 0. |
| **Hypothesis** | The emergency brake speed floor of 50 km/h is capping corner speeds at ~49.8 km/h. Raising the floor to 80 km/h will allow the car to carry 76+ km/h through the wider corners (min ahead > 30 m) without triggering emergency braking. |
| **AI recommendation** | One-line change: emergency brake trigger `forward_clear < 35 and speedX > 50` → `forward_clear < 35 and speedX > 80`. Brake formula adjusted to `(speedX − 80) / 65`. Predicted corner 1 min speed ~76–82 km/h, corners 2 and 3 partially improved. Predicted lap time ~2:33–2:37, damage 0. |
| **Change made** | Line 551 of `snakeoil3_v2_experiment.py`: distance trigger reverted `50→35 m`, speed floor raised `50→80 km/h`, brake formula updated accordingly. |
| **Predicted result** | Lap time ~2:33–2:37 · Damage 0. Corner 1 → ~76–82 km/h, corners 2 and 3 partially improved. |
| **Actual lap time** | **2:27:50** — 13 seconds faster than original baseline, 19 seconds faster than Run 1. |
| **Damage** | **0** — clean lap. |
| **What surprised us?** | The improvement was broader than predicted. 8 of 14 corners showed significant speed gains (corners 1, 6, 7, 8, 12, 13 all jumping 14–19 km/h). The corner count dropped from 16 to 14, meaning two zones merged or disappeared as the car carried more speed through them. Straight max speeds mostly held or improved slightly. Corner 2 (tightest, min ahead 14.6 m) barely moved (43.5 → 44.7 km/h) — confirmed geometry-limited, not brake-limited. |
| **Keep / modify / revert** | **Modify** — the emergency brake change is a clear keeper. The remaining bottleneck is the throttle target K=2.5, which at 14–22 m ahead yields only 36–55 km/h ceiling, leaving corners 2–5 and the last corner under-served. New hypothesis: raise K to push medium corners toward 76 km/h goal. |
| **Next experiment** | Raise K from 2.5 to 3.5 on line 567: `forward_clear * 2.5` → `forward_clear * 3.5`. Single-line change. K=3.5 at 21.6 m ahead → target 75.6 km/h, directly hitting the 76 km/h goal for corners 3, 4, 5. Predicted lap time ~2:20–2:24, damage 0. Watch corner 2 (min ahead 14.6 m) — if min-ahead drops below 10 m in telemetry, reduce K to 3.2. |

---

## Run 3 — Throttle target K raised to 3.5

| Field | Details |
|---|---|
| **Baseline** | Run 2 result: `snakeoil3_v2_experiment.py`, lap time 2:27:50, damage 0. |
| **Hypothesis** | The driver is only using hard brakes instead of gentle brakes and not holding throttle when turning corners. The K=2.5 throttle target ceiling (36–55 km/h at tight apexes) is too low — the car still goes quiet in corners 2–5 and the last corner. Raising K to 3.5 gives a target of 51–81 km/h at the same apexes, directly reintroducing active throttle through the turn. |
| **AI recommendation** | Single-line change on line 567: `forward_clear * 2.5` → `forward_clear * 3.5`. K=3.5 chosen because at min-ahead of 21.6 m it yields 75.6 km/h — hitting the 76 km/h goal for corners 3, 4, 5 without overshooting tight corner 2 (14.6 m → 51 km/h, still within approach brake safety net). Predicted lap time ~2:20–2:24, damage 0. |
| **Change made** | Line 567 of `snakeoil3_v2_experiment.py`: `forward_clear * 2.5` → `forward_clear * 3.5`. |
| **Predicted result** | Lap time ~2:20–2:24 · Damage 0. Corners 3, 4, 5 → ~72–76 km/h. Corner 1 → ~85–90 km/h. Corner 2 → ~50–55 km/h (geometry-limited). |
| **Actual lap time** | **2:23:90** — 4 seconds faster than Run 2, 17 seconds faster than original baseline. |
| **Damage** | **0** — clean lap. |
| **What surprised us?** | 10 of 14 corners clustered tightly at 69.0–69.4 km/h — a new floor identical to the approach brake threshold of `speedX > 70`. The approach brake was firing at every corner with `ahead < 80 m` and scrubbing speed to just below 70 km/h, then zeroing accel. K=3.5 raised the throttle ceiling but the approach brake kept capping the result. Corner 2 improved most (44.7 → 57.4 km/h) because its tight geometry (14 m ahead) means the approach brake released sooner. The lap time gain came almost entirely from the corners already freed in Run 2 — the remaining 10 corners were untouched by K. |
| **Keep / modify / revert** | **Modify** — the approach brake speed floor of 70 km/h is the new bottleneck. Raise it to 105 km/h AND raise K to 4.2 to push medium corners (22 m ahead) into the 80–100 km/h target window. |
| **Next experiment** | Two changes together on `snakeoil3_v2_experiment.py`: (1) line 556 — approach brake floor `70 → 105 km/h`; (2) line 567 — K `3.5 → 4.2`. Target: corner min speeds 80–100 km/h, lap time ~2:12–2:16, damage 0. Watch corner 2 (14 m ahead) — if min-ahead drops below 8 m, lower approach brake floor to 95 km/h. |

---

## Run 4 — Approach brake floor raised to 105 km/h + K raised to 4.2

| Field | Details |
|---|---|
| **Baseline** | Run 3 result: `snakeoil3_v2_experiment.py`, lap time 2:23:90, damage 0. |
| **Hypothesis** | The approach brake speed floor of 70 km/h is capping 10 of 14 corners at ~69 km/h. Raising it to 105 km/h removes that ceiling. Simultaneously raising K from 3.5 to 4.2 pushes the throttle target at medium corners (22 m ahead) to ~92 km/h — squarely inside the 80–100 km/h target window. |
| **AI recommendation** | Line 556: `speedX > 70` → `speedX > 105`. Line 567: `forward_clear * 3.5` → `forward_clear * 4.2`. Predicted lap time ~2:12–2:16, damage 0. Corners 3, 4, 5 → 85–96 km/h. Corner 2 → 58–65 km/h (geometry-limited). |
| **Change made** | Line 556: approach brake floor raised `70 → 105 km/h`. Line 567: K raised `3.5 → 4.2`. |
| **Predicted result** | Lap time ~2:12–2:16 · Damage 0. Corners 3, 4, 5, 12, 13 → 85–100 km/h. Corner 1 → 80–90 km/h (governed by emergency brake). Corner 2 → 58–65 km/h (geometry-limited by 14 m ahead). |
| **Actual lap time** | **2:01:03** — 22 seconds faster than Run 3, 39 seconds faster than original baseline. |
| **Damage** | **0** — clean lap. |
| **What surprised us?** | The lap time jump of 22 seconds was the largest single-run gain of the entire experiment. Corners 3, 4, 5, 12, 13 all converged at exactly 79.6–79.9 km/h — a new cluster floor, this time caused by the emergency brake (`< 35 m, speedX > 80`) firing in those zones (min ahead 20–23 m, all below 35 m). The approach brake change freed the 69 km/h ceiling perfectly, but the emergency brake immediately became the next wall. Straight-line speeds also improved meaningfully: straight 5 rose to 163 km/h, straight 6 to 198.6 km/h, straight 9 to 162.5 km/h — the corner exit improvements compounded onto the straights. |
| **Keep / modify / revert** | **Modify** — the emergency brake speed floor of 80 km/h is the new bottleneck. Raise it to 125 km/h to free the 79–80 km/h cluster. Simultaneously raise K from 4.2 to 5.0 to push medium corners (22 m ahead) to ~110 km/h target — inside the new 90–120 km/h window. |
| **Next experiment** | Line 551+554: emergency brake floor `80 → 125 km/h`. Line 567: K `4.2 → 5.0`. Target: corner min speeds 90–120 km/h. Predicted lap time ~1:52–1:58 — first sub-2:00 run. Watch corner 2 (12.6 m ahead, currently 62.3 km/h) and corner 9 (9.1 m ahead) — tightest geometries on track, most likely to show wall proximity with raised speeds. |

---

## Run 5 — Emergency brake floor raised to 125 km/h + K raised to 5.0

| Field | Details |
|---|---|
| **Baseline** | Run 4 result: `snakeoil3_v2_experiment.py`, lap time 2:01:03, damage 0. |
| **Hypothesis** | The emergency brake floor of 80 km/h is capping corners 3, 4, 5, 12, 13 at ~79–80 km/h. Raising it to 125 km/h removes that ceiling. K=5.0 pushes the throttle target at medium corners (22 m ahead) to 110 km/h — inside the 90–120 km/h target window. |
| **AI recommendation** | Line 551: `speedX > 80` → `speedX > 125`. Line 554: `(speedX - 80)` → `(speedX - 125)`. Line 567: K `4.2 → 5.0`. Predicted lap time ~1:52–1:58, damage 0. |
| **Change made** | Emergency brake floor raised `80 → 125 km/h`, formula anchored to new floor. K raised `4.2 → 5.0`. |
| **Predicted result** | Lap time ~1:52–1:58 · Damage 0. |
| **Actual lap time** | **DNF** — crashed at dist 2471.8 m (corner 9). |
| **Damage** | **23** — 3 impact events at dist 2471–2495 m, trackPos −1.661 to −0.594 (left wall). |
| **What surprised us?** | The throttle cap of 85 km/h controlled corner 9's apex speed but did nothing about entry speed. The car arrived from straight 7 at ~162 km/h, the approach brake scrubbed to ~105 km/h, then released — leaving ~57 km/h still to scrub in only 9.1 m. A throttle cap only limits what the car chases; it cannot slow a car already travelling above the cap. The fix requires a dedicated entry brake zone starting ~120 m before corner 9, not just an apex throttle limit. Corner 16 (dist 3226–3399) also showed min ahead −1.0 m for the first time — next DNF candidate once corner 9 is fixed. |
| **Keep / modify / revert** | **Modify** — add zone-specific entry brake for corner 9 (dist 2290–2410 m): partial brake if speedX > 105. Raise corner 9 throttle cap to 95 km/h (centre of 85–105 window). Add same pattern for corner 16 (dist 3106–3226 entry, 3226–3287 corner). |
| **Next experiment** | Corner 9: early brake entry zone 2290–2410 m + throttle cap 85→95 km/h. Corner 16: same pattern. All global parameters unchanged. |

---

## Run 6 — Corner 9 entry brake + corner 9 & 16 caps at 95 km/h

| Field | Details |
|---|---|
| **Baseline** | Run 5 result: DNF, damage 23. Corner 9 caused crash. |
| **Hypothesis** | A throttle cap alone cannot prevent a crash when entry speed is too high. Corner 9 needs a dedicated entry brake zone (dist 2290–2410) to scrub speed. Corner 16 treated the same preemptively. |
| **AI recommendation** | Entry brake zone 2290–2410: partial brake if speedX > 105. Corner 9 cap: 85→95 km/h. Corner 16 entry + cap same logic. |
| **Change made** | Added `in_corner_9_entry` and `in_corner_16_entry` brake zones. Corner caps set to 95 km/h. |
| **Predicted result** | Lap time ~1:52–1:58 · Damage 0. Corner 9 min speed 85–100 km/h. |
| **Actual lap time** | **DNF** — crashed again at dist 2494.8 m (corner 9). |
| **Damage** | **24** — 2 impact events at dist 2494–2500 m, trackPos −1.146 (off track left). |
| **What surprised us?** | The entry brake zones were redundant — the global approach brake (`< 80 m, speedX > 105`) was already doing the same job, so the extra braking disrupted the car's line approaching corner 9. The damage happened at 70.9 km/h with `trackPos −1.146` — the car was already off the track edge (±1.0). Corner 9 is a tracking/steering failure, not a speed failure: min speed was 49.8 km/h (well-controlled), but the car slid offline. Corner 16 also showed min speed 18.6 km/h — the entry brake + near-zero forward_clear caused a `min(300, 95, −1.0 × 5.0) = −5 km/h` throttle target, completely stalling the car. |
| **Keep / modify / revert** | **Modify** — remove both entry brake zones (global approach brake already covers entry). Lower caps to 92 km/h (midpoint of ≥85 <100 window). Fix `forward_clear` floor: use `max(0, forward_clear) * 5.0` to prevent negative throttle targets when sensor reads negative. |
| **Next experiment** | Remove entry brakes, caps to 92 km/h, `max(0, forward_clear)` guard. Target: damage 0, lap time ~1:52–1:58. |

---

## Run 7 — Entry brakes removed, caps lowered to 92 km/h, negative sensor guard added

| Field | Details |
|---|---|
| **Baseline** | Run 6 result: DNF, damage 24. Corner 9 off-track slide, corner 16 stalled. |
| **Hypothesis** | Removing entry brakes + `max(0, forward_clear)` guard + 92 km/h caps would fix both failures. |
| **AI recommendation** | Remove entry brake zones, lower caps 95→92 km/h, use `max(0, forward_clear) * 5.0`. |
| **Change made** | Entry brake zones removed. Caps: 95→92 km/h. Throttle term guarded. File: `snakeoil3_v2_experiment.py`. |
| **Predicted result** | Lap time ~1:52–1:58 · Damage 0. |
| **Actual lap time** | **DNF** — 4th consecutive crash at corner 9, same coordinates (dist ~2493 m, trackPos −1.140). |
| **Damage** | **27** — 4 impact events. Corner 16 also off-track (min speed 13.6 km/h, min ahead −1.0 m). |
| **What surprised us?** | Speed tuning is irrelevant to these two crashes. The car hits the wall at 51–70 km/h with `trackPos −1.14` — it is off the track edge (±1.0) at low speed. This is a pure steering failure. The steer-to-center multiplier of 0.10 produces only `+0.10` correction at `trackPos = −1.0`, far too weak to pull the car back inside a 9.1 m gap. No speed change can fix a car that is steering off the road. Corner 16 has the identical geometry and fails the same way. |
| **Keep / modify / revert** | **New file** — `snakeoil3_v2_experiment2.py`. Fix root cause: zone-scoped steer-to-center raised to `0.40` inside danger zones. Add 30 km/h throttle floor so car stays steerable when `forward_clear ≤ 0`. |
| **Next experiment** | Run 8 with `snakeoil3_v2_experiment2.py`. Once damage = 0, attack straights for sub-2:00. |

---

## Run 8 — Zone-scoped steer-to-center 0.40 + 30 km/h throttle floor (current / pending)

| Field | Details |
|---|---|
| **Baseline** | Run 7 result: DNF, damage 27. 4th consecutive crash at corners 9 and 16 — confirmed steering failure, not speed failure. |
| **Hypothesis** | Steer-to-center multiplier of 0.10 is too weak inside corners 9 and 16 (min ahead −1.0 m). At `trackPos = −1.0` the correction is only `+0.10` — not enough to fight lateral drift in a 9 m gap. Raising to `0.40` in those zones gives 4× recovery force. A 30 km/h throttle floor keeps the car steerable when `forward_clear ≤ 0` (target would otherwise be 0). |
| **AI recommendation** | `snakeoil3_v2_experiment2.py`: `in_danger_zone` gate for dist 2410–2515 and 3226–3401. `steer_center_gain = 0.40` inside, `0.10` outside. `throttle_floor = 30` inside zones, applied as `max(throttle_floor, min(...))`. No global parameters changed. |
| **Change made** | New file `snakeoil3_v2_experiment2.py`. Steer-to-center: zone-scoped `0.10 → 0.40`. Throttle: `max(30, min(target, cap9, cap16, max(0, forward_clear) * 5.0))` inside danger zones. All other corners unchanged. |
| **Predicted result** | Lap time ~1:52–1:58 · Damage 0. Car holds track through corners 9 and 16. Sub-2:00 reachable next run by addressing straight speeds (straights 1, 4, 8 underperforming). |
| **Actual lap time** | **1:55:99** — first clean lap since Run 4. Sub-2:00 target ✅ achieved. 44:54 faster than original baseline. |
| **Damage** | **0** — clean lap. |
| **What surprised us?** | The steer-to-center fix resolved both corner 9 and corner 16 completely. Corner 9 min speed 57.6 km/h (geometry-limited, on track). Corner 14/16 min speed 46.5 km/h (on track). The 30 km/h throttle floor kept the car steerable through both −1.0 m ahead zones without causing any overrun. Straight 10 gained a new 192 m zone at 167.5 km/h — previously the car was crashing before reaching it. Straight-line speeds (123.6–167.5 km/h on underperforming straights) are now the primary remaining time source: estimated 6+ seconds still available. |
| **Keep / modify / revert** | **Keep** — steer-to-center fix and throttle floor are confirmed safe and effective. Next focus: straight-line acceleration. |
| **Next experiment** | Run 9 (new file): (1) Accel step `0.03 → 0.04` for faster corner-exit throttle build. (2) Gear 3→4 shift threshold `110 → 95 km/h` to use 4th gear immediately after corner exits. Estimated gain: ~4–6 seconds. Predicted lap time ~1:48–1:52, damage 0. Watch corner 2 (min-ahead 6.8 m) — faster acceleration means slightly higher entry speed there. |

---

## v3 Run 1 — Wider sensors + track memory (snakeoil3_v3.py)

| Field | Details |
|---|---|
| **Baseline** | Run 8 result: `snakeoil3_v2_experiment2.py`, lap time 1:55:99, damage 0. |
| **Hypothesis** | Wider sensor array (±80° outer, ±30° mid-range) gives earlier wall detection. Track memory enables anticipatory braking from lap 2 onward, replacing reactive hard-brake entries with smoother pre-braking. |
| **AI recommendation** | New file `snakeoil3_v3.py`: sensor array `-80 -45 -30 -19 -12 -7 -4 -2.5 -1.7 0 1.7 2.5 4 7 12 19 30 45 80`. Wall bias correction using wide sensors. Track memory dict recording min `forward_clear` per 1 m bucket, anticipatory brake from lap 2 when memory predicts `ahead < 20 m` within 80 m. |
| **Change made** | New file `snakeoil3_v3.py`. Sensor array widened. Wall bias steer correction added. Track memory + anticipatory brake block added. All v2_experiment2 logic otherwise unchanged. |
| **Predicted result** | Lap time ~1:52–1:56 · Damage 0. Smoother corner entries from memory anticipation. Corner 9 and 16 protected by existing steer-to-center 0.40 gain. |
| **Actual lap time** | **1:57:08** — 1:09 *slower* than Run 8. |
| **Damage** | **13** — corner 9 wall contact again (dist 2491.6 m, trackPos −1.139, speed 67.0 km/h). |
| **What surprised us?** | The wall bias correction backfired in corner 9. When the car drifts left in that zone, the −80° sensor reads short and the +80° reads long — `wall_bias` becomes negative, adding a leftward steer correction that fights the 0.40 steer-to-center and wins. The v3 sensor was steering the car *into* the wall instead of away from it. Corner 16 (last corner) went off-track and the car froze at near-zero throttle on re-entry — `accel` had decayed to ~0 during the excursion and recovered only at +0.03/tick. The anticipatory brake also caused corner 9 to arrive faster than experiment2 because memory triggered gentle braking 80 m before the corner but then released, leaving the car with more speed at the 9.1 m gap than the reactive-only logic did. |
| **Keep / modify / revert** | **Modify** — three targeted fixes in new file `snakeoil3_v3_experiment.py`: (1) disable wall bias inside danger zones; (2) lower corner 9 cap 92→75 km/h; (3) add off-track recovery: force `accel=0.2, brake=0.0` when `trackPos > ±1.1`. |
| **Next experiment** | v3 Run 2 with `snakeoil3_v3_experiment.py`. Predicted: damage 0, lap time ~1:54–1:56. |

---

## v3 Run 2 — Wall bias guarded + corner 9 cap 75 km/h + off-track recovery (current / pending)

| Field | Details |
|---|---|
| **Baseline** | v3 Run 1: `snakeoil3_v3.py`, lap time 1:57:08, damage 13. Wall bias caused corner 9 crash; off-track freeze at last corner. |
| **Hypothesis** | Wall bias inverts correction direction inside corners 9 and 16 — disabling it there restores the 0.40 steer-to-center dominance. Corner 9 cap of 92 km/h too high combined with anticipatory braking — lowering to 75 km/h tightens speed control. Off-track `accel` decay prevented full throttle re-entry — a 0.2 forced throttle on off-track fixes this. |
| **AI recommendation** | Line 586: add `and not in_danger_zone` to wall bias condition. Line 615: corner 9 cap `92→75` km/h. Line 623: add off-track recovery block `if abs(trackPos) > 1.1: accel=0.2, brake=0.0`. |
| **Change made** | New file `snakeoil3_v3_experiment.py`. Wall bias gated by `not in_danger_zone`. Corner 9 cap: `92→75` km/h. Off-track recovery block added before throttle control. |
| **Predicted result** | Lap time ~1:54–1:56 · Damage 0. Corner 9 stays on track. Last corner re-joins with full throttle. |
| **Actual lap time** | CHECK — run not yet executed. |
| **Damage** | **1** — single graze at dist 2493.0 m, speed 62.8 km/h, trackPos −0.796. Car stayed on track. |
| **What surprised us?** | Damage reduced from 13 to 1 — wall bias gate and 75 km/h corner 9 cap are working. The car brushed the wall at trackPos −0.796 (just inside the edge) rather than going fully off at −1.14. Off-track recovery also worked — last corner min speed 58.2 km/h with the car staying on track. However the lap time of 1:57:55 is still 1:56 slower than Run 8 (experiment2). The v3 features (wider sensors + track memory anticipatory brake) are adding net overhead — the anticipatory brake fires before corners already handled smoothly by the reactive system, causing unnecessary speed scrubbing. Straight-line speeds dropped vs Run 8: straight 6 went from 202.1 → 197.3 km/h, straight 5 from 163.7 → 159.4 km/h. The speed-threshold gearbox is still the bottleneck — no downshifts means the car bogs on corner exits. |
| **Keep / modify / revert** | **Modify** — add RPM-based gearbox (upshift at 8500 RPM, downshift at 3000 RPM, 5-tick hysteresis) as the next isolated change. Corner 9 steer_center_gain also a candidate for 0.40→0.50 to eliminate the last 1-point damage. |
| **Next experiment** | `snakeoil3_v3_rpm_test.py`: replace speed-threshold gearbox with RPM-based system. Upshift at 8500 RPM, downshift at 3000 RPM, 5-tick hysteresis to prevent gear-hunting. Estimated gain: 3–5 seconds on straights 1, 4, 8. Predicted lap time ~1:52–1:55, damage 0–1. |

---

## v3 RPM Test — RPM-based gearbox (current / pending)

| Field | Details |
|---|---|
| **Baseline** | v3 Run 2: `snakeoil3_v3_experiment.py`, lap time 1:57:55, damage 1. |
| **Hypothesis** | The speed-threshold gearbox has no downshift logic — after braking into a corner the car stays in a high gear with low torque, bogging on exit. Using RPM to trigger both upshifts (8500 RPM) and downshifts (3000 RPM) keeps the engine in its torque band at all times. 5-tick hysteresis prevents gear-hunting oscillation at thresholds. |
| **AI recommendation** | Replace the 6-line speed ladder with an RPM-based system. Module-level globals `_drive_tick` and `_last_gear_change` track hysteresis. `if rpm > 8500 and gear < 6: shift up`. `if rpm < 3000 and gear > 1: shift down`. Only shift if ≥5 ticks since last change. |
| **Change made** | New file `snakeoil3_v3_rpm_test.py`. Lines 523–531: added `_drive_tick`, `_last_gear_change`, `GEAR_HYSTERESIS`, `RPM_UPSHIFT`, `RPM_DOWNSHIFT` globals. Lines 556: `global` statement extended. Lines 661–676: speed-ladder replaced with RPM-based shift logic. |
| **Predicted result** | Lap time ~1:52–1:55 · Damage 0–1. Straights 1, 4, 8 gain 10–20 km/h from better corner-exit gear selection. Corner 9 residual 1-point graze may persist — separate fix planned after this run. |
| **Actual lap time** | **2:03:00** — 7 seconds *slower* than baseline Run 8. |
| **Damage** | **26** — corner 2 wall crash (dist 2496.9 m, trackPos −1.159) and corner 9. |
| **What surprised us?** | The RPM gear carry-over was broken. `DriverAction.d['gear']` initialises to 1 every step — the `R['gear']` read as "current gear" was always 1, causing the car to oscillate between gear 1 and 2 for the entire lap. Straight 1 dropped to 94.2 km/h (was 123+ km/h), confirming the car never left low gear. Corner 2 also crashed (new — min ahead −1.0 m for the first time), caused by the car entering the corner in gear 1 with no torque to steer out. The RPM system needs gear state stored in a persistent module-level variable, not read from `R['gear']`. File deleted. |
| **Keep / modify / revert** | **Revert** — `snakeoil3_v3_rpm_test.py` deleted. Return to `snakeoil3_v3_experiment.py` as base. Fix corner 9 entry speed with a zone-specific early brake zone (dist 2300–2410 m, trigger at 90 km/h). |
| **Next experiment** | `snakeoil3_v3_experiment3.py`: add corner 9 early brake zone (dist 2300–2410 m, `speedX > 90 → brake = min(0.45, (speedX−90)/80)`). Predicted: damage 0, lap time ~1:55–1:57. |

---

## v3 Experiment 3 — Corner 9 early brake zone (current / pending)

| Field | Details |
|---|---|
| **Baseline** | v3 Run 2: `snakeoil3_v3_experiment.py`, lap time 1:57:55, damage 1 (corner 9 graze at trackPos −0.796). |
| **Hypothesis** | The global approach brake (`< 80 m, speedX > 105`) fires too late for corner 9's 9.1 m gap — the car enters the tight section at ~75–80 km/h. Adding a 110 m approach zone (dist 2300–2410) with a softer 90 km/h trigger gives 30 m more braking distance, scrubbing speed to ~70 km/h before entry. The 75 km/h throttle cap then holds the apex. |
| **AI recommendation** | Add zone-specific early brake block: `if 2300 < dist < 2410 and speedX > 90: brake = min(0.45, (speedX−90)/80), accel = 0.0`. Placed after anticipatory memory brake, before reactive brakes. No global parameters changed. |
| **Change made** | New file `snakeoil3_v3_experiment3.py`. One new `if` block added at lines 606–609: corner 9 approach brake zone 2300–2410 m, floor 90 km/h, max brake 0.45. |
| **Predicted result** | Lap time ~1:55–1:57 · Damage 0. Car arrives at corner 9 apex at ~70 km/h. All other corners unchanged. |
| **Actual lap time** | **1:58:00** — clean lap, 0 damage. Corner 9 min ahead improved from −1.0 to 8.7 m — car is staying on track through the tight section. |
| **Damage** | **0** — clean lap. |
| **What surprised us?** | Corner 9 min ahead recovered from −1.0 m to 8.7 m — the early brake zone is working exactly as intended, scrubbing enough speed before the 9.1 m gap that the steer-to-center holds the car on track. Corner 14 (last corner) min speed 58.1 km/h with min ahead −1.0 m — still tight but staying on track. Lap time 1:58:00 is 2 seconds slower than Run 8 (1:55:99) but cleaner than all v3 runs. The v3 anticipatory memory brake is still adding overhead — straight 6 at 197.3 km/h vs 202.1 km/h in Run 8. Corner 12 (dist 2647–2779, min ahead 20.3 m) is the next braking target — earlier braking into it would sharpen the entry and potentially recover 1–2 seconds. |
| **Keep / modify / revert** | **Modify** — add early brake zone for corner 12 (dist 2520–2647 m), same pattern as corner 9 fix. User requested 1.5 seconds earlier braking ≈ 42 m additional braking distance at ~100 km/h approach speed. |
| **Next experiment** | `snakeoil3_v3_experiment4.py`: add corner 12 early brake zone (dist 2520–2647 m, `speedX > 90 → brake = min(0.45, (speedX−90)/80)`). Predicted: lap time ~1:56–1:58, damage 0. |

---

## v3 Experiment 4 — Corner 12 early brake zone (current / pending)

| Field | Details |
|---|---|
| **Baseline** | v3 Experiment 3: `snakeoil3_v3_experiment3.py`, lap time 1:58:00, damage 0. |
| **Hypothesis** | Corner 12 (dist 2647–2779 m, 131 m long, min ahead 20.3 m) currently relies on the reactive approach brake firing ~80 m before entry (~dist 2568 m). Adding a zone-specific early brake starting at dist 2520 m gives an additional ~42 m (≈1.5 s at 100 km/h) of braking distance for a smoother, better-positioned corner entry. |
| **AI recommendation** | Add `if 2520 < dist < 2647 and speedX > 90: brake = min(0.45, (speedX−90)/80), accel = 0.0`. Same pattern and parameters as corner 9 early brake. Zone starts at 2520 m (just after corner 9 exits at 2515 m) to avoid interference. |
| **Change made** | New file `snakeoil3_v3_experiment4.py`. One new `if` block added after the corner 9 early brake: corner 12 approach zone 2520–2647 m, floor 90 km/h, max brake 0.45. |
| **Predicted result** | Lap time ~1:56–1:58 · Damage 0. Smoother corner 12 entry. Corner 9 behaviour unchanged. |
| **Actual lap time** | CHECK — run not yet executed. |
| **Damage** | CHECK — run not yet executed. |
| **What surprised us?** | CHECK — pending run. |
| **Keep / modify / revert** | CHECK — pending run. If damage = 0 and lap time improves: Keep. If car brakes too hard between corners 9 and 12 (short 132 m gap): raise floor to 95 km/h or shorten zone start to 2560. If lap time regresses: zone too aggressive, raise floor to 100 km/h. |
| **Next experiment** | CHECK — pending Experiment 4. If clean and faster: apply same early brake pattern to corner 14 (last corner, dist 3226–3309, min ahead −1.0 m) and look at fixing the RPM gearbox with proper persistent `_current_gear` state. |

---

## v4 Last Test — Three-card fix: degenerate zones + adaptive angle gain + approach throttle (snakeoil3_v4_last_test.py)

| Field | Details |
|---|---|
| **Baseline** | `snakeoil3_v3_map.py` (TRACK_MAP system) — last measured clean run: 1:59:00, 0 damage. New baseline telemetry from `snakeoil3_v2.py`: 2:40:53, 0 damage, 16 corner zones identified. |
| **Hypothesis** | Three structural bugs in v3_map prevent it from reaching its potential: (1) C8/C10/C11 apex zones are 1–2 m wide after trimming — `in_apex` almost never fires so speed_cap and throttle_floor are unused. (2) Angle gain is fixed at 15/PI regardless of corner tightness — tight corners (C9 at 8.7 m, C14 at 8.6 m) need stronger heading correction. (3) The approach zone zeros accel entirely when the brake fires — this creates a coast dead-zone between the straight exit and the apex where the car has neither brake nor throttle. |
| **AI recommendation** | Three changes, all isolated: (Card 1) Raise C8/C10/C11 `end` to `start + 20` minimum. (Card 2) Add `angle_gain` field to TRACK_MAP: 20/PI for min_ahead < 10, 17/PI for min_ahead < 20, 15/PI for ≥ 20. Use inside apex zone only. (Card 3) Replace `accel = 0.0` in Layer 2 approach brake with `accel = 0.15` — brake still fires, engine stays loaded. |
| **Change made** | New file `snakeoil3_v4_last_test.py`. C8 end: 2352 → 2371. C10 end: 2563 → 2582. C11 end: 2609 → 2627. `angle_gain` field added to all TRACK_MAP entries. Drive function reads `zone['angle_gain']` inside apex, falls back to 15/PI outside. Layer 2 approach brake: `accel = 0.0` → `accel = 0.15`. |
| **Predicted result** | Lap time ~1:54–1:57 · Damage 0. C8/C10/C11 now have active speed_cap and throttle_floor. Tighter heading hold through C9/C14. Smoother corner turn-in with sustained engine torque. No risk change at danger zones (C9/C14 parameters unchanged). |
| **Actual lap time** | CHECK — run not yet executed. |
| **Damage** | CHECK — run not yet executed. |
| **What surprised us?** | CHECK — pending run. |
| **Keep / modify / revert** | CHECK — pending run. If approach throttle 0.15 causes corner overshoot (min-ahead shrinks): lower to 0.10 or restrict to non-danger zones only. If degenerate zone fix causes C10/C11 brake interference with C9 exit (dist 2509 → C10 brake_start 2522): shorten C10 end back to 2570. |
| **Next experiment** | CHECK — pending v4 run. If clean and faster: add RPM-based gearbox with persistent `_current_gear` module variable (upshift 8500 RPM, downshift 3000 RPM, 5-tick hysteresis). |
