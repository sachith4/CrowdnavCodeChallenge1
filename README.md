# Code Challenge #1: Fix a misconfigured CrowdNav

## The challenge

Your CrowdNav starts with a badly configured router: **50% of the smart cars are sent on exploration routes**, and cars **reroute only every 600 ticks**. Write a UPISAS strategy that detects this from the monitored data and fixes it at runtime.

## What you need

| Tool | Check it works |
| --- | --- |
| Docker Desktop | `docker compose version` |
| Python 3.9 or newer | `python3 --version` |

## Part 1: Set up CrowdNav

**1. Get CrowdNav.**

```bash
git clone https://github.com/JeremyGDiamond/Crowdnav-FAS2024
cd Crowdnav-FAS2024
```

**2. Set the bad configuration.** Open `crowdnav/knobs.json` and change these two values. Leave everything else as it is.

```json
"explorationPercentage": 0.5,
"reRouteEveryTicks": 600,
```

**3. Build and start the three containers** (Kafka, CrowdNav, HTTP server). The first build takes several minutes because it installs the SUMO simulator.

```bash
docker compose up -d --build
docker ps
```

`docker ps` should list `kafka`, `crowdnav` and `http-server`.

**4. Fix the API's Kafka library.** The API installs the newest `aiokafka`, which can't talk to the old Kafka in this setup: every change you send fails with "Internal Server Error". Install the last version that works:

```bash
docker exec http-server pip install "aiokafka==0.12.0"
docker restart http-server
```

This survives `docker restart`, but **not** `docker compose down` or a rebuild. After either of those, run these two commands again.

**5. Check that CrowdNav is running.** Wait about a minute, then:

```bash
curl -s localhost:8080/monitor/
```

You should get JSON with `car_stats` and `configs`, where `configs` shows `"exploration_percentage":0.5` and `"re_route_every_ticks":600`.

**6. Prepare Kafka for your commands.** CrowdNav only listens for commands on a Kafka topic that exists when it starts, and the topic is created by the first command anyone sends. Send one command with the current (bad) values, then restart CrowdNav:

```bash
curl -s -X PUT localhost:8080/execute/ -H 'content-type: application/json' -d '{"route_random_sigma":0.2,"exploration_percentage":0.5,"max_speed_and_length_factor":1,"average_edge_duration_factor":1,"freshness_update_factor":10,"freshness_cut_off_value":90,"re_route_every_ticks":600,"total_car_counter":750,"edge_average_influence":140}'

docker restart crowdnav
```

The reply must contain `"status":"success"`. You only need this step once after `docker compose up`.

> **Note:** To avoid repeating this between tests, use `docker compose stop` and `docker compose start` instead of `down` and `up`. Those keep the containers, so the topic and the library fix survive. The same goes for `docker restart crowdnav`, which is all you need between test runs anyway. Only use `down` when you really want a clean slate.

## Part 2: Set up UPISAS

**1. Get UPISAS and install it.**

```bash
git clone https://github.com/S2-group/UPISAS
cd UPISAS
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
# Experiment Runner is a git submodule
cd experiment-runner
git submodule update --init --recursive
pip install -r requirements.txt
cd ..
```

**2. Write a CrowdNav exemplar.** UPISAS has no exemplar for CrowdNav, so create `UPISAS/exemplars/crowdnav.py`. It points UPISAS at the CrowdNav API on `http://localhost:8080` and waits until the simulation publishes data. `UPISAS/exemplars/swim.py` is an example to look at.

**3. Write a run script.** Create `run_challenge.py` in the repository root. It starts your CrowdNav exemplar, then runs your strategy's MAPE-K loop (monitor, analyze, plan, execute) every few seconds until the simulation reaches step 10,000. (10,000 is an arbitrary step value so that the loop can stop.)

## Part 3: Write your strategy

Create `UPISAS/strategies/my_strategy.py`. In `analyze()`, use the monitored data to detect the bad configuration; in `plan()`, decide the new knob values for `execute()` to send. Send all nine knobs with every change, and leave `total_car_counter` as it is.

## Part 4: Run and check it worked

**1. Start a fresh simulation, then your strategy.** From the UPISAS folder, with the virtual environment active:

```bash
docker restart crowdnav
python run_challenge.py
```

**2. Check that the values changed.** While your strategy runs, in a second terminal:

```bash
curl -s localhost:8080/monitor/
```

Under `configs`, `exploration_percentage` and `re_route_every_ticks` should no longer show `0.5` and `600`, but the values your strategy sent. CrowdNav applies changes every 10 simulation steps, so give it a few seconds after your strategy executes.

## What to submit

The three files you created:

1. `UPISAS/exemplars/crowdnav.py`
2. `UPISAS/strategies/my_strategy.py`
3. `run_challenge.py`

## Evaluation

The challenge is pass/fail. We run your three files on a CrowdNav set up as in Part 1, with the bad configuration in `knobs.json`. You pass when all of these hold:

1. `run_challenge.py` runs without errors.
2. Your strategy reads the configuration through UPISAS and changes `exploration_percentage` and `re_route_every_ticks` back from `0.5` and `600` while CrowdNav is running.
3. After your strategy has executed, `/monitor/` shows the new values.
4. Your strategy uses UPISAS only: no changes to `knobs.json` or CrowdNav's code, and `total_car_counter` stays the same.

Reverting the two values is enough to pass; you don't need to optimise anything else.
