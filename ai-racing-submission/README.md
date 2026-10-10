# IBM AI Racing League — Submission

## What is in this folder

| File | Description |
|---|---|
| `snakeoil3_v2.py` | **Submission script.** Self-contained Python 3 client. Run this against a live TORCS server to drive the car. |
| `autostart.sh` | Bash helper that sends xte keystrokes to dismiss the TORCS GUI and start the race automatically. Required on Linux when TORCS is launched headlessly. |
| `practice.xml` | TORCS race configuration. Track: `michigan` (oval), mode: Practice, 20 laps, up to 8 `scr_server` drivers. |
| `lap_data.csv` | Raw telemetry recorded during our best clean run (Run 8). One row per simulation step: `distFromStart, speedX, trackPos, angle, steer, accel, brake, gear, track_ahead, damage`. |
| `lap_summary.csv` | One row per experiment run — lap time, damage, key change, outcome, next action. Human-readable race log. |
| `lap_summary.xlsx` | Same data as `lap_summary.csv`, formatted as an Excel workbook with a "Lap Summary" sheet. |
| `experiment-report.md` | Full written experiment log — every hypothesis, change made, result, and next step. |

---

## Python dependencies

`snakeoil3_v2.py` imports only Python standard-library modules. No `pip install` is required.

| Import | Module | Notes |
|---|---|---|
| `socket` | stdlib | UDP communication with TORCS server |
| `sys` | stdlib | Exit codes, stderr |
| `getopt` | stdlib | Command-line option parsing |
| `os` | stdlib | `os.system()` calls for TORCS relaunch |
| `time` | stdlib | `time.sleep()` during server reconnect |
| `csv` | stdlib | Telemetry output to `lap_data.csv` |

**Confirmed working: Python 3.11.8**

---

## System requirements

The script itself is pure Python and runs on any OS. The TORCS simulator it connects to is Linux-only in this configuration.

| Component | Required | Notes |
|---|---|---|
| Python 3 | ✅ Yes | 3.11.8 confirmed; 3.8+ expected to work |
| TORCS / vtorcs-RL-color | ✅ Yes | The simulator the script connects to over UDP. Must be running before the script is launched. Installation described in `vtorcs-RL-color/` (not included in this submission folder). |
| `scr_server` plugin | ✅ Yes | TORCS must be patched with the SCR server extension for UDP client connections to work. Included in the vtorcs-RL-color build. |
| `xautomation` (`xte`) | Required for `autostart.sh` | Linux package. Used by `autostart.sh` to dismiss the TORCS GUI. Not needed if the race is started manually through the GUI. Install: `sudo apt-get install xautomation` |
| Ubuntu 14.04 / 16.04 LTS | Recommended | vtorcs-RL-color build tested on these versions per original project README. |
| OpenAI Gym / numpy | ❌ Not needed | Required by `gym_torcs.py` (RL wrapper, not used by this submission). `snakeoil3_v2.py` has no such dependency. |

---

## How to run

### 1 — Start TORCS

```bash
torcs -nofuel -nodamage -nolaptime &
```

Wait for TORCS to reach the blue server-ready screen, then (if not using GUI):

```bash
sh autostart.sh
```

### 2 — Run the client

```bash
python snakeoil3_v2.py
```

#### Common options

```
-p, --port <port>     TORCS server UDP port  [default: 3001]
-H, --host <host>     TORCS server hostname  [default: localhost]
-m, --steps <#>       Maximum simulation steps  [default: 100000]
-e, --episodes <#>    Maximum learning episodes  [default: 1]
-d, --debug           Print full telemetry to stderr each step
-h, --help            Show all options
```

> **Do not hardcode the port.** Pass it with `-p` so the script works in multi-car race configurations.

### 3 — Telemetry

While running, the script writes `lap_data.csv` to the working directory (overwrites on each run). Columns:

```
distFromStart, speedX, trackPos, angle, steer, accel, brake, gear, track_ahead, damage
```

---

## Track

- **Name:** `michigan`
- **Category:** oval
- **Length:** 3608.2 m
- **Mode:** Practice (as configured in `practice.xml`)

---

## Unavailable kit components

The following components are referenced by the broader `gym_torcs` project but are **not included** in this submission folder because they are not required to run `snakeoil3_v2.py`:

| Component | Reason not included |
|---|---|
| `vtorcs-RL-color/` | Full TORCS simulator source tree. Must be built and installed separately on Linux. |
| `gym_torcs.py` | OpenAI-Gym RL wrapper. Not used by the submission script. Requires `numpy` and `gym`. |
| `example_experiment.py` / `sample_agent.py` | RL agent examples. Not part of this submission. |
| `snakeoil3_gym.py` | Vision-mode client. Not used. |
| `torcs_jm_par.py` / `run-cars.py` | Multi-car race runner. Not used by the submission script. |

---

## Acknowledgements

- Original `snakeoil` Python client: Chris X Edwards (http://xed.ch/project/snakeoil/index.html)
- vtorcs-RL-color: based on https://github.com/giuse/vtorcs
- gym-torcs wrapper: developed during spring internship 2016 at Preferred Networks
- SCR competition protocol: Daniele Loiacono et al. (https://arxiv.org/pdf/1304.1672.pdf)
