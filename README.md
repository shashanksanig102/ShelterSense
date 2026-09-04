# ResiliSense

**Predictive Edge AI for Emergency Shelter Resilience**

Resilient America Preparedness Challenge — Track A (Developers)
Sponsors/Partners: Qualcomm Technologies / Qualcomm for Good, EDGE AI FOUNDATION

## One-sentence summary

An offline, multimodal edge-AI system that detects developing environmental
and ventilation failures in emergency shelters *before* they cross
conventional single-sensor alarm thresholds.

## Core idea

Fixed-threshold monitoring ("if CO2 > 1000ppm, alarm") tells you a shelter
has *already* become unsafe. ResiliSense instead fuses CO2, particulate
matter, gas signatures, temperature/humidity, occupancy, and fan/HVAC
behavior over time, learning the combined pattern that precedes a critical
condition — so it can flag `Degrading` before a single-sensor system would
even reach `Warning`.

The system runs entirely on the Arduino UNO Q (no cloud dependency), because
shelters stand up in exactly the connectivity-degraded conditions a disaster
creates.

---

## Repository layout

```
firmware/
  resilisense.ino          Arduino sketch (UNO Q MCU side) - sensor reads,
                            fan PWM control, serial telemetry/commands

software/
  data_logger/logger.py    Records labeled experimental trials to CSV
  preprocessing/features.py Rolling-window feature engineering
  inference/infer.py       Live prediction loop + local alerting
  baseline/baseline.py     Fixed-threshold baseline + warning lead time calc
  simulate/generate_demo_data.py  Synthetic data for pipeline testing ONLY

model/
  training/train.py        Random Forest training with trial-based split

data/                       Raw + feature CSVs go here (see data/README.md)
audio/                      Pre-recorded alert announcements (see audio/README.md)
requirements.txt
```

---

## Hardware

| Component | Purpose |
|---|---|
| Arduino UNO Q | Main compute (STM32U585 MCU + Dragonwing QRB2210 Linux/AI side) |
| BME688 | Temperature, humidity, pressure, gas/VOC signature |
| Winsen MH-Z1911A (NDIR) | Authoritative CO2 measurement |
| MiCS-6814 | Broader gas signature (CO/NO2/NH3-associated channels) |
| PMS5003 | Particulate matter (PM1.0 / PM2.5 / PM10) |
| 12V DC fan/blower | Simulated shelter HVAC load for controlled experiments |
| MOSFET/fan driver | Isolates the 12V fan from the MCU's GPIO |
| CQRobot 3W 8Ω passive speaker + Class-D amp | Local audio alerts (output only — no microphone) |

**Explicitly not used:** microphone, camera/occupancy sensor, cloud server,
mains-voltage HVAC control hardware. Population is entered manually by a
shelter operator (see rationale in the proposal doc) rather than sensed
automatically.

### Hardware verification checklist (do this before wiring)

- [ ] Confirm UART peripheral mapping on the UNO Q for the PMS5003 and
      MH-Z1911A — do not assume `Serial1`/`Serial2` match a classic Uno.
      Check Arduino's official UNO Q documentation/pinout diagram.
- [ ] Confirm which analog pins are ADC-capable for the MiCS-6814's 3 outputs.
- [ ] Confirm the MiCS-6814 breakout's exact pin labeling (varies by vendor).
- [ ] Never connect the passive speaker directly to a GPIO — it needs the
      Class-D amplifier in between.
- [ ] Never connect the 12V fan directly to the UNO Q — use the MOSFET/fan
      driver with its own external 12V supply, sharing a common ground with
      the low-voltage electronics.
- [ ] Confirm BME688 I2C address (0x76 vs 0x77 depending on SDO wiring).

---

## Setup

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Flash `firmware/resilisense.ino` to the UNO Q's MCU side via the Arduino IDE
(select the UNO Q board definition once installed via Boards Manager).

---

## Usage

### 1. Generate synthetic data to sanity-check the pipeline (no hardware needed)

```bash
python software/simulate/generate_demo_data.py --out data/synthetic_trials.csv --n-trials 12
python software/preprocessing/features.py data/synthetic_trials.csv data/synthetic_features.csv
python model/training/train.py --features data/synthetic_features.csv --model-out model/demo_model.pkl
```

**Reminder: synthetic data is for exercising the code only. Do not report
metrics from it as competition evidence — real experimental trials are
required for that (see Data Collection Protocol below).**

### 2. Collect a real experimental trial

```bash
python software/data_logger/logger.py \
    --port /dev/ttyACM0 --trial-id 1 --state normal \
    --out data/raw/trial_001_normal.csv --duration 300
```

Repeat for `degrading` and `critical` trials (see Data Collection Protocol),
gradually reducing fan PWM via the `F=` serial command (or a small script
that sends it) between/during trials.

### 3. Build features and train on real trials

```bash
python software/preprocessing/features.py data/raw/all_trials.csv data/features.csv
python model/training/train.py --features data/features.csv --model-out model/resilisense_rf.pkl --test-trials 9 10
```

### 4. Run live inference

```bash
python software/inference/infer.py --port /dev/ttyACM0 --model model/resilisense_rf.pkl
```

### 5. Compare against the baseline (for your warning-lead-time claim)

```bash
python software/baseline/baseline.py data/raw/trial_003_critical.csv
```
Then combine with `infer.py`'s logged predictions for the same trial using
`warning_lead_time()` in `baseline.py` to compute the actual measured lead
time — **only report this number after measuring it on a real trial.**

---

## Data collection protocol

Run each scenario multiple times, varying population, to build a dataset
robust enough that trial-based train/test splitting is meaningful:

| Trial type | Fan setting | Purpose |
|---|---|---|
| Normal | ~100% | Baseline healthy conditions |
| Degrading | Gradually 100% → ~65% | Learn the "developing problem" signature |
| Critical | Further reduced to ~20-30% | Learn the unsafe-condition signature |
| Recovery | Restored to 100% | Confirm the system detects recovery, not just decline |

Log a plain-text lab notebook alongside your CSVs noting exact wall-clock
times you changed the fan setting during each trial — you'll need this to
correctly label multi-phase trials and to honestly compute warning lead time.

---

## Honest scope and limitations

This prototype is a **decision-support research tool**, not certified
life-safety equipment:

- MiCS-6814 raw values require calibration before they can be described as
  quantitative gas concentrations (ppm) — treat them as relative signatures
  for classification, not certified readings.
- BME688 gas output is an environmental/VOC signature, not a certified gas
  detector.
- PMS5003 is a particulate sensor, not a certified wildfire/life-safety alarm.
- MH-Z1911A is the project's dedicated CO2 measurement, but is not a
  certified evacuation alarm on its own.
- Any "warning lead time" figure quoted in the proposal must come from a
  real measured experiment on real hardware — never from synthetic data or
  an unmeasured estimate.

## Competition alignment

| Pillar | How ResiliSense addresses it |
|---|---|
| Anticipate and Mitigate Risk | Predicts deterioration before a fixed threshold is crossed |
| Enable Real-Time Response | Local inference + immediate local audio alert, no network round-trip |
| Operate in Mission-Critical Environments | Fully functional with zero internet/cellular connectivity |
