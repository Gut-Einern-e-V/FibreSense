# 🧵 FiberSense

> A handheld NIR textile fiber scanner for FAB Labs.
> Fork of [reremeter](https://github.com/arminstr/reremeter) — adapted for textiles.

[![License: CERN-OHL-S v2](https://img.shields.io/badge/Hardware-CERN--OHL--S%20v2-orange)](https://ohwr.org/cern_ohl_s_v2.txt)
[![License: MIT](https://img.shields.io/badge/Firmware-MIT-blue)](LICENSE-FIRMWARE)
[![Data: CC BY-SA 4.0](https://img.shields.io/badge/Data-CC%20BY--SA%204.0-lightgrey)](https://creativecommons.org/licenses/by-sa/4.0/)

---

## What it is

FiberSense is a handheld spectral scanner that identifies textile fiber composition. Press it against a fabric, press the button, and within 2 seconds the round display shows what it is made of.

It is a textile port of the [reremeter](https://github.com/arminstr/reremeter) project by Armin Straller and Bernhard Gessler (RealRecycling), which does the same for plastics. The measurement principle, ADS1256 ADC, and LED-sequential approach are directly derived from that work. FiberSense adds: textile-specific wavelength selection, a textile reference library, a single-PCB dual-sensor design, and an educational workshop curriculum.

---

## Hardware — three parts total

```
┌─────────────────────────────────────────────────────┐
│  Waveshare RP2040-Touch-LCD-1.28   (~15 €)          │
│                                                     │
│  RP2040 dual-core · 1.28" round touch IPS 240×240   │
│  GC9A01A display · CST816S touch · QMI8658C IMU     │
│  LiPo charging · USB-C · 1.27mm GPIO headers        │
└──────────────────────┬──────────────────────────────┘
                       │ 1.27mm pin header (soldered)
┌──────────────────────▼──────────────────────────────┐
│  FiberSense Sensor PCB  (JLCPCB, ~60–120 €)         │
│                                                     │
│  ┌──────────────────┐   ┌───────────────────────┐   │
│  │ AS7265x (I2C)    │   │ ADS1256 + G8370-03     │   │
│  │ 18ch 410–940nm   │   │ 8× NIR LEDs 940–1650nm │   │
│  │ VIS/NIR          │   │ 74HC4051 mux           │   │
│  └──────────────────┘   └───────────────────────┘   │
│                                                     │
│  Both sensors populated. Firmware auto-detects      │
│  which are present and runs accordingly.            │
└─────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│  Sensor Hat  (3D printed, black PETG)               │
│                                                     │
│  Light-tight measurement chamber                    │
│  45°/0° LED–sample–detector geometry                │
│  Spring-loaded contact (~0.5N, FreeCAD flexure)     │
└─────────────────────────────────────────────────────┘
```

No WiFi module. No separate display board. No jumpers.

---

## Sensor auto-detection

On boot the firmware scans available buses and configures itself:

| AS7265x present | ADS1256 present | Mode |
|---|---|---|
| ✓ | — | **Learn** — VIS/NIR 410–940nm, 18 channels |
| — | ✓ | **Measure** — NIR 940–1650nm, 8 channels |
| ✓ | ✓ | **Combined** — both sensors, 26 channels total |
| — | — | **Error** — "No sensor found" on display |

The round display shows the detected mode for 2 seconds on boot. No configuration needed.

---

## Why two sensors on one board

**AS7265x (Learn mode):** Covers 410–940nm — the visible and near-NIR range. Useful for color
classification and coarse fiber separation (~70–80% accuracy on pure fabrics). This range sits
before the molecular fingerprint region, so it cannot distinguish white cotton from white
polyester. That limitation is the point: in workshops, participants see it fail, understand
why, and then see Measure mode solve it. The AS7265x is also what makes the device useful
for color-based sorting tasks where full NIR is not needed.

**ADS1256 + G8370-03 + LED array (Measure mode):** LED-sequential NIR reflectometry at
940–1650nm. Hits the diagnostic absorption bands for organic fibers: O–H at 1450nm (cotton,
wool), C–H at 1200 and 1650nm (polyester), N–H at 1550–1600nm (nylon, wool). ~90–95%
accuracy on pure fibers. This is the instrument that produces real data.

**Combined mode** runs both sequentially in one scan (~1.5 seconds total) and transmits
a 26-channel spectrum. Useful for research and dataset collection.

---

## Bill of Materials

### 1. Waveshare RP2040-Touch-LCD-1.28

| Where | Price |
|-------|-------|
| Eckstein / Waveshare direct / Amazon | ~13–16 € |

This is the entire compute + display unit. Nothing else needed for the "brain".

Internal pins used by the board (cannot be reassigned):
- GC9A01A display: SPI0 (GPIO10 SCK, GPIO11 MOSI, GPIO9 CS, GPIO8 DC, GPIO12 RST)
- CST816S touch: I2C1 (GPIO6 SDA, GPIO7 SCL) — touch INT on GPIO21
- QMI8658C IMU: I2C0 (GPIO4 SDA, GPIO5 SCL)
- LiPo charger: internal, no GPIO

Free GPIOs available for sensor PCB (via 1.27mm header):
GPIO0, GPIO1, GPIO2, GPIO3, GPIO13–GPIO20, GPIO22–GPIO28 — sufficient for both sensors.

### 2. FiberSense Sensor PCB

Order from JLCPCB. Gerber files and BOM in `/hardware/sensor_pcb/`.

#### Learn mode sensor (AS7265x)

| # | Part | MPN | LCSC# | Qty | ~Price |
|---|------|-----|-------|-----|--------|
| S1 | Spectral sensor triad | AS7265x (3-chip) | — | 1 | 55–65 € |

> The AS7265x is three chips in one package communicating over I2C. Available as SparkFun
> breakout (SEN-15050) or bare IC. For PCB integration use the bare IC from Mouser/Digikey.
> I2C address: 0x49. The firmware detects it on boot by reading the device ID register.

#### Measure mode sensor

**Detector — order first (2–4 weeks lead time):**

| # | Part | MPN | Source | Qty | ~Price |
|---|------|-----|--------|-----|--------|
| D1 | InGaAs photodiode | Hamamatsu **G8370-03** | Mouser / Hamamatsu EU | 1 | ~20 € |
| D2 | TIA op-amp | OPA2381AIDR SOT-23-5 | LCSC C7484 | 1 | 1.50 € |
| D3 | TIA feedback R | 10 MΩ 0603 | LCSC | 1 | 0.05 € |
| D4 | TIA stability C | 10 pF 0603 | LCSC | 1 | 0.05 € |
| D5 | 24-bit ADC | ADS1256IDBT SSOP-16 | LCSC / JLCPCB Global | 1 | 8–12 € |

> G8370-03: TO-18 metal can, φ3mm active area, 0.9–1.7µm, no cooling needed. NOT G12183.
> Order from mouser.de or email eu-sales@hamamatsu.com. ~€20 single unit.

**NIR LEDs — order from Roithner Lasertechnik, Vienna (sales@roithner-laser.com):**

| # | MPN | λ (nm) | ~Price | Diagnostic target |
|---|-----|--------|--------|-------------------|
| L1 | Vishay VSLY3940 | 940 | 0.30 € | O–H overtone baseline (also at LCSC) |
| L2 | ELD-1050-525 | 1050 | ~3 € | C–H first overtone |
| L3 | ELD-1200-525 | 1200 | ~3 € | C–H combination (PET strong) |
| L4 | ELD-1300-525 | 1300 | ~3 € | Inter-band baseline |
| L5 | ELD-1450-525 | 1450 | ~4 € | O–H stretch (cotton/wool strong, PET absent) |
| L6 | ELD-1550-525 | 1550 | ~4 € | C–H / C–O (nylon) |
| L7 | ELD-1600-525 | 1600 | ~5 € | N–H combination (wool, nylon) |
| L8 | ELD-1650-525 | 1650 | ~5 € | C–H second overtone (all synthetics) |

All ELD-series: 5mm THT package, 20mA. Budget ~€30 for all 7 NIR LEDs + shipping from Vienna.

**LED driver (all JLCPCB-assemblable):**

| # | Part | MPN | LCSC# | ~Price |
|---|------|-----|-------|--------|
| M1 | 8:1 mux | 74HC4051D SOIC-16 | C6051 | 0.20 € |
| M2 | N-MOSFET | BSS138 SOT-23 | C112114 | 0.05 € ×8 |
| M3 | Current R | 33 Ω 0603 | — | 0.02 € ×8 |

**Shared passives (all LCSC):**

| Value | Package | Qty | Purpose |
|-------|---------|-----|---------|
| 10 MΩ | 0603 | 1 | TIA feedback |
| 100 kΩ | 0603 | 2 | LiPo ADC divider |
| 10 kΩ | 0603 | 3 | I2C pull-ups, button |
| 33 Ω | 0603 | 8 | LED current limit |
| 100 nF | 0603 | 8 | Bypass |
| 10 pF | 0603 | 1 | TIA stability |
| 10 µF | 0805 | 3 | Bulk decoupling |

**JLCPCB assembly split:**
- JLCPCB assembles: ADS1256 (Global Part), OPA2381, 74HC4051, BSS138 ×8, all passives, SD holder
- Solder by hand: G8370-03 (TO-18 THT), Roithner LEDs (5mm THT), AS7265x (if bare IC: QFN — or use SparkFun breakout on header), Waveshare board (pin header)

### 3. Sensor Hat (3D printed)

```
hardware/case/
├── sensor_hat.FCStd         Light-tight measurement chamber (parametric)
│                            · LED positions: 45° angle to sample surface
│                            · Detector: normal (0°) to sample surface
│                            · Chamber depth: 8mm (parametric, adjust per LED type)
│                            · Spring flexure: ~0.5N contact force
├── body_top.FCStd           Enclosure top — round display cutout + button
├── body_bottom.FCStd        Enclosure bottom — USB-C + LiPo access
└── stl/                     Print-ready STL exports
```

Print: black PETG for sensor hat (light-tight). Any material/color for body.
Settings: 0.15mm layer for hat, 0.2mm for body. 40% infill for hat, 20% for body.

> **The sensor hat geometry matters more than anything else in this design.**
> Reproducible LED–sample distance (±0.5mm) and ambient light rejection are the
> primary sources of measurement variance. Tune the FreeCAD model before ordering PCBs.

### Total BOM cost

| Configuration | Cost |
|---|---|
| Learn mode only (AS7265x) | ~€80–100 |
| Measure mode only (NIR array) | ~€120–160 |
| Both sensors (full board) | ~€160–220 |

---

## Sensor PCB pin assignment

All connections from Waveshare board to Sensor PCB via 1.27mm header:

| GPIO | Function | Sensor |
|------|----------|--------|
| GPIO13 | I2C1 SDA (second I2C) | AS7265x |
| GPIO14 | I2C1 SCL (second I2C) | AS7265x |
| GPIO15 | ADS1256 CS | ADS1256 |
| GPIO16 | ADS1256 DRDY | ADS1256 |
| GPIO17 | SPI MOSI (SPI0 alt) | ADS1256 |
| GPIO18 | SPI MISO (SPI0 alt) | ADS1256 |
| GPIO19 | SPI SCK (SPI0 alt) | ADS1256 |
| GPIO20 | 74HC4051 select A | LED mux |
| GPIO22 | 74HC4051 select B | LED mux |
| GPIO26 | 74HC4051 select C | LED mux |
| GPIO27 | LED enable (MOSFET gate) | LED array |
| GPIO28 | Trigger button (active LOW) | — |
| GPIO29 | LiPo voltage divider ADC | — |
| 3V3 | Power | Both sensors |
| GND | Ground | Both sensors |

> Note: GPIO4/5 (I2C0) and GPIO6/7 (I2C1) are used internally by the Waveshare board
> for QMI8658C and CST816S. AS7265x uses the second I2C on GPIO13/14.
> GPIO8–12 are used by the display. GPIO9 is shared (display CS) — use GPIO13 for I2C.

---

## Firmware

```
firmware/
├── platformio.ini           RP2040, Earle Philhower Arduino core
└── src/
    ├── config.h             Pin definitions, thresholds, constants
    ├── main.cpp             Boot → detect → idle → scan loop
    ├── sensor_detect.cpp/h  I2C scan + SPI probe → returns SensorMode enum
    ├── sensor_learn.cpp/h   AS7265x driver (SparkFun library wrapper)
    ├── sensor_measure.cpp/h ADS1256 driver (PIO SPI) + LED sequencer
    ├── sensor_combined.cpp/h Run both sequentially, merge ScanResult
    ├── calibration.cpp/h    Two-point white/dark, stored on SD as JSON
    ├── classifier.cpp/h     k-NN on SD reference CSVs, cosine distance
    ├── display_round.cpp/h  GC9A01A UI: spectrum arc, result, animations
    ├── touch.cpp/h          CST816S gesture handling
    ├── imu.cpp/h            QMI8658C: orientation check, shake, lift-wake
    └── transmit.cpp/h       USB Serial JSON output
```

### Boot sequence

```
Power on
  │
  ├─ Init display → show FiberSense logo (500ms)
  ├─ Init IMU (QMI8658C)
  ├─ I2C scan GPIO13/14 → AS7265x at 0x49?  ─── yes ──► flag LEARN
  ├─ SPI probe GPIO15/16 → ADS1256 DRDY?    ─── yes ──► flag MEASURE
  │
  ├─ LEARN + MEASURE → show "Combined mode" 2s
  ├─ LEARN only      → show "Learn mode" 2s  
  ├─ MEASURE only    → show "Measure mode" 2s
  └─ neither         → show "No sensor!" + error animation
  │
  ├─ Load calibration from SD (/cal/white.json, /cal/dark.json)
  │    └─ Not found → orange ring animation + "Calibrate first" prompt
  │
  └─ IDLE — waiting for trigger
```

### IMU behaviour

The QMI8658C accelerometer is used for three things:

**Orientation lock:** If the device is not held approximately flat (more than 45° tilt),
the trigger button is disabled and the display shows a tilt indicator. This prevents scans
taken at wrong angles against the fabric from producing silently bad data.

**Motion rejection:** If acceleration exceeds 0.3g during a scan (device moved), the scan
is automatically cancelled and repeated. Shown on display as "Hold still...".

**Gestures (for workshop / child use):**
- Double-tap on the round display edge → clears last result, returns to idle
- Shake → initiates recalibration prompt
- Lift from flat surface → wakes display from sleep (display dims after 30s idle)

### Display UI (GC9A01A, 240×240 round)

The round form factor is used as a design element, not worked around:

```
         ┌───────────────────┐
        /   ╔═══════════════╗  \
       /    ║  POLYESTER     ║   \
      |     ║     96%        ║    |
      |     ╚═══════════════╝    |
      |   spectrum arc plot      |
       \   (wavelength ring)    /
        \  ● battery  wifi ○  /
         └───────────────────┘
```

**Idle:** Last material result (large text, center). Confidence as colored fill arc around
the edge of the display (green = high confidence, yellow = medium, red = low).
Spectrum displayed as a radial bar chart using the full circle — wavelengths mapped to
angular positions, bar heights = reflectance.

**Scanning:** Animated ring sweeping around the display as each LED fires. Channel
markers light up one by one as data comes in. Children find this satisfying to watch.

**Calibration:** Split-circle animation: left half = "white reference", right half = "dark".
Each half fills as calibration step completes.

**Result:** Material name large (2 lines if needed). Confidence arc. Top 3 alternatives
shown as small text below.

### Calibration

Two-point, stored on SD card.

```
R_calibrated[i] = (raw[i] - dark[i]) / (white[i] - dark[i])
```

Triggered by: Settings menu (touch) → Calibrate, or shake gesture.

1. "Place white card against sensor. Tap to scan."
   → White PTFE or 4× white copier paper, pressed flat against sensor hat
2. "Cover sensor completely. Tap to scan."
   → Black cloth or cap over sensor hat

Saved as `/cal/white.json` and `/cal/dark.json`. Temperature at calibration time stored.
If temperature changes >5°C: orange arc animation on idle display, "Recalibrate?" prompt.

### USB Serial output

Every completed scan outputs newline-terminated JSON to USB Serial (115200 baud):

```json
{
  "device": "fibersense",
  "mode": "measure",
  "timestamp_ms": 1234567,
  "channels_nm": [940, 1050, 1200, 1300, 1450, 1550, 1600, 1650],
  "spectrum_raw": [18432, 21100, 9823, 15234, 4201, 8932, 9102, 7834],
  "spectrum_calibrated": [0.94, 0.89, 0.51, 0.72, 0.22, 0.47, 0.53, 0.43],
  "material": "Cotton",
  "confidence": 0.91,
  "temperature_c": 23.4,
  "calibration_age_ms": 720000
}
```

---

## Classification

Classification runs on the RP2040 itself using a lightweight k-NN against reference CSVs
on the SD card. No server needed for basic use.

```
data/reference_spectra/
├── learn_mode/           18-channel CSVs (410–940nm)
│   ├── Cotton_100/
│   ├── Polyester_100/
│   └── ...
└── measure_mode/         8-channel CSVs (940–1650nm)
    ├── Cotton_100/
    ├── Polyester_100/
    ├── Wool_100/
    ├── Nylon_PA6/
    ├── Viscose/
    └── Cotton_PET_5050/
```

CSV format (header = wavelengths, one row per measurement):
```
940,1050,1200,1300,1450,1550,1600,1650
0.94,0.89,0.51,0.72,0.22,0.47,0.53,0.43
```

To add a new fiber type: collect 10+ scans with the device (guided by
`tools/collect_reference.py`), copy CSV to SD card, device reloads on next boot.

For more advanced training (SVM, Random Forest, cross-validation):
`tools/train_classifier/train.py` — runs on any PC, exports `classifier_weights.h`
for optional firmware embedding without SD card.

### Expected accuracy (Measure mode, starter library)

| Fiber | Accuracy | Main diagnostic λ |
|-------|----------|-------------------|
| Cotton 100% | ~93% | 1450nm (O–H high) |
| Polyester 100% | ~96% | 1200 + 1650nm (C–H high) |
| Wool 100% | ~88% | 1450 + 1600nm (N–H protein) |
| Nylon PA6/66 | ~85% | 1550 + 1600nm (N–H amide) |
| Viscose/Lyocell | ~82% | Cotton-like spectrum |
| Cotton/PET 50/50 | ~75% | Ratio 1450/1200nm |
| Acrylic (PAN) | ✗ | Key band at 2240nm, outside G8370 range |

---

## Using the OpenTextile-NIR dataset

Sormunen et al. (2026) released 71 labeled post-industrial samples.
DOI: [10.5281/zenodo.18269172](https://doi.org/10.5281/zenodo.18269172)

The SWIR data (1000–2500nm) does not overlap with our sensor range and cannot be used
directly for calibration. What we can use:

- `rgb_mean_values.csv` → bootstrap Learn mode color classifier (71 labeled RGB samples)
- `ground_truth_final.csv` → realistic accuracy expectations + understanding label error rates
  (11 of 71 samples were flagged as outliers — label errors from manufacturer data are real)

Both files are included in `data/zenodo/` for reference.

---

## Workshop formats

**"Why does it fail?" (2h)**
Scan white cotton and white polyester with Learn mode. Watch the classifier struggle.
Look at the spectrum plot — the curves are nearly identical at 410–940nm. Discuss: what
wavelengths would you need? What is the molecular difference between cotton and polyester?
Demonstrate Measure mode. This sequence is the core pedagogical value of having both sensors.

**"Build it" (4–6h)**
Participants receive a pre-assembled PCB (JLCPCB assembled SMD parts) and a bag with the
Roithner LEDs, G8370-03, and Waveshare board. They solder the THT parts, print the sensor
hat, stack everything, flash firmware, run calibration. Emphasis on sensor hat geometry.

**"Train it" (3h)**
Collect reference spectra from a sample set with known labels. Run `train.py`. Inspect the
confusion matrix. Add samples for weak classes. Re-train. Discussion: what does it mean that
some ground truth labels are wrong? How many samples do you need?

**"Hack it" (open)**
The sensor PCB has a standard 2.54mm header footprint for adding external sensors.
The firmware sensor_detect auto-discovers new I2C devices. Participants can extend the
device with additional sensors, build scan table fixtures, or connect it to other systems
via USB serial.

---

## Roadmap

**v1.0** — this repo. Measure + Learn mode, 6 fiber types, round display UI.

**v1.1** — scan table arm (FreeCAD), foot pedal trigger (GPIO28 external), batch scan mode.

**v2.0** — second detector daughterboard: Hamamatsu G9208-256W InGaAs array (1.1–2.15µm)
for acrylic (2240nm C≡N band) and improved blend quantification. Expansion connector
reserved on v1.0 sensor PCB.

---

## License & Attribution

- **Hardware**: [CERN-OHL-S v2](LICENSE-HARDWARE)
- **Firmware**: [MIT](LICENSE-FIRMWARE)
- **Data**: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

**Based on:** reremeter by Armin Straller & Bernhard Gessler (GPL-3.0)
https://github.com/arminstr/reremeter

**Reference dataset:** Sormunen et al. (2026), OpenTextile-NIR,
Data in Brief 65, 112559. DOI: 10.1016/j.dib.2026.112559
