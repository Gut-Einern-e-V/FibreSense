# 🧵 FiberSense — Open Hardware Textile Fiber Scanner

> A reproducible, sub-€250 NIR reflectometer for textile fiber identification.
> Built for FAB Labs, textile recyclers, and researchers who need real data.

[![License: CERN-OHL-S v2](https://img.shields.io/badge/Hardware-CERN--OHL--S%20v2-orange)](https://ohwr.org/cern_ohl_s_v2.txt)
[![License: MIT](https://img.shields.io/badge/Firmware-MIT-blue)](LICENSE-FIRMWARE)
[![Data: CC BY-SA 4.0](https://img.shields.io/badge/Data-CC%20BY--SA%204.0-lightgrey)](https://creativecommons.org/licenses/by-sa/4.0/)

---

## What this is — and what it is not

FiberSense is a **LED-sequential NIR reflectometer**: it shines 8 narrow-band infrared LEDs
(940–1650 nm) onto a textile sample one at a time, measures reflected intensity with a single
InGaAs photodiode, and classifies the fiber composition from the resulting 8-point spectrum.

It is not a hyperspectral camera. It is not a grating spectrometer. The reference paper
(Sormunen et al. 2026, see below) uses a €50,000 Specim SWIR 3 camera covering 1000–2500 nm
at 288 channels. FiberSense covers 940–1650 nm at 8 channels and costs under €250.

That gap is real and worth being honest about:

| Capability | Specim SWIR 3 (paper) | FiberSense |
|---|---|---|
| Spectral range | 1000–2500 nm | 940–1650 nm |
| Channels | 288 | 8 |
| Spatial resolution | Hyperspectral image | Single point |
| Cotton vs PET | ✓ | ✓ |
| Cotton/PET blend % | ✓ accurate | ~±15% estimate |
| Acrylic (PAN) | ✓ (2240nm band) | ✗ outside range |
| Silk | ✓ | marginal |
| Cost | ~€50,000 | ~€200 |

FiberSense is a serious instrument for the fibers it covers. It is not a toy version of the paper.

---

## The physics in two paragraphs

NIR spectroscopy works because organic molecules absorb infrared light at wavelengths determined
by their chemical bonds. Cellulose (cotton, lyocell) has a strong O–H stretch at 1450 nm.
Polyester (PET) has strong C–H bands at 1200 nm and 1650 nm. Wool and nylon have N–H bands
near 1500–1600 nm. These bands are present, consistent, and distinguishable even with only 8
measurement points — if those 8 points are placed at the right wavelengths.

The trick is that you do not need a continuous spectrum. You need a spectrum *at the right
wavelengths*. This is why LED-sequential systems work for classification even though they look
nothing like a lab spectrometer. The 8 wavelengths in FiberSense were chosen to hit the most
diagnostic absorption features for the 6 fiber types it targets.

---

## Two learning formats, one instrument

FiberSense has two operating modes that use different sensor hardware but share everything else —
the enclosure, the firmware interface, the REST API, and the data format.

**Learn mode (AS7265x breakout, ~€100 total)**
Uses a SparkFun AS7265x breakout board (VIS/NIR, 410–940 nm, 18 channels). This range sits
*before* the molecular fingerprint region. It measures color and surface reflectance, not fiber
composition. It can separate obviously different fibers (~70% accuracy on pure fabrics) but
fails on white cotton vs white polyester. This is the point: participants experience the
limitation directly, understand *why* the wavelength range matters, and can then see the
difference when the NIR board is swapped in. Good for 2–3 hour workshops.

**Measure mode (InGaAs + LED array, ~€200–250 total)**
The actual instrument. Uses a Hamamatsu G8370-03 InGaAs photodiode with 8 NIR LEDs from
Roithner Lasertechnik (940–1650 nm). ~90–95% accuracy on pure fibers, ~75% on common blends.
This is what you build when you want real data.

Both modes connect to the same classification server (see Architecture below).

---

## Architecture

The most important design decision: **the device does not classify. It measures.**

```
┌─────────────────────────────┐
│  FiberSense hardware        │     USB Serial / WiFi (JSON)
│                             │ ──────────────────────────────►  Classification server
│  RP2040 (measurement MCU)   │                                   (RPi Zero 2W, or any PC)
│  + sensor (A or B)          │ ◄──────────────────────────────   result: material + confidence
│  + ESP32-C3 (WiFi only)     │
│  + ILI9341 display          │
│  + WS2812B + button         │
└─────────────────────────────┘
```

The RP2040 handles timing-critical ADC reads and sensor sequencing without WiFi interference.
The ESP32-C3 handles WiFi/BLE as a dumb serial bridge — no logic, no firmware complexity.
The classifier runs on a server that is updated independently of the hardware.

**Why this matters:** k-NN classification on an SD card inside a microcontroller cannot be
updated without reflashing. A classifier running on a Python server can be retrained in minutes
when new reference spectra are added. The device becomes a permanent calibrated sensor;
the intelligence is software.

---

## Hardware

### Shared (both modes)

| # | Part | MPN | LCSC# | ~Price | Notes |
|---|------|-----|-------|--------|-------|
| 1 | MCU | RP2040 (WROOM or bare chip) | C2040 | 1–4 € | Timing-deterministic via PIO |
| 2 | WiFi module | ESP32-C3-MINI-1 | C2934560 | 3–4 € | Serial bridge only, no logic |
| 3 | Display | ILI9341 2.4" TFT SPI | — (module) | 4–6 € | External module, pin header |
| 4 | SD holder | Micro-SD SPI SMD | C91145 | 0.50 € | LCSC |
| 5 | Status LED | WS2812B 5050 SMD | C2761795 | 0.15 € | LCSC |
| 6 | Trigger button | 6×6 mm SMD tact | C318884 | 0.05 € | LCSC |
| 7 | USB-C connector | 16-pin SMD | C165948 | 0.30 € | LCSC |
| 8 | LiPo charger | TP4056 + DW01A | C16581 | 0.80 € | LCSC |
| 9 | LDO 3.3V | AMS1117-3.3 SOT-223 | C6186 | 0.10 € | LCSC |
| 10 | LiPo battery | 3.7V 1000mAh JST-PH | — (external) | 4–6 € | |

### Learn mode sensor (additional, ~€60)

| # | Part | MPN | Source | ~Price |
|---|------|-----|--------|--------|
| S1 | Spectral sensor | SparkFun AS7265x Triad breakout | SparkFun / Mouser | 55–65 € |

Connects via I2C (SDA/SCL). No SMD soldering. Plug in, flash, run.

### Measure mode sensor (additional, ~€80–120)

#### Detector — order first, 2–4 weeks lead time

| # | Part | MPN | Source | ~Price | Critical notes |
|---|------|-----|--------|--------|----------------|
| D1 | InGaAs photodiode | Hamamatsu **G8370-03** | Mouser / Hamamatsu EU direct | ~20 € | TO-18, φ3mm, 0.9–1.7µm. NOT G12183 (needs cryo). |
| D2 | TIA op-amp | OPA2381AIDR SOT-23-5 | LCSC C7484 | 1.50 € | Low-noise, rail-to-rail |
| D3 | TIA feedback R | 10 MΩ SMD 0603 | LCSC | 0.05 € | |
| D4 | TIA stability C | 10 pF SMD 0603 | LCSC | 0.05 € | |
| D5 | 24-bit ADC | ADS1256IDBT SSOP-16 | LCSC / JLCPCB Global | 8–12 € | SPI, 30kSPS, 8-ch |

> **Hamamatsu G8370-03:** Order via mouser.de (search "G8370-03") or email eu-sales@hamamatsu.com.
> Single-unit price ~€20. The G8370 series needs no cooling — dark current is low enough at
> room temperature for textile reflectance measurements with 16-sample averaging.

#### NIR LEDs — order from Roithner Lasertechnik, Vienna

Email: sales@roithner-laser.com — fast response, ships EU, accepts bank transfer or PayPal.
All ELD-series: 5mm THT package, ~20mA forward current, ±20nm bandwidth.

| # | MPN | λ (nm) | ~Price | Why this wavelength |
|---|-----|--------|--------|---------------------|
| L1 | Vishay VSLY3940 | 940 | 0.30 € | O–H overtone baseline (LCSC ok) |
| L2 | Roithner ELD-1050-525 | 1050 | ~3 € | C–H first overtone |
| L3 | Roithner ELD-1200-525 | 1200 | ~3 € | C–H combination (PET strong) |
| L4 | Roithner ELD-1300-525 | 1300 | ~3 € | Inter-band baseline |
| L5 | Roithner ELD-1450-525 | 1450 | ~4 € | O–H stretch (cotton/wool strong, PET absent) |
| L6 | Roithner ELD-1550-525 | 1550 | ~4 € | C–H / C–O combination (nylon) |
| L7 | Roithner ELD-1600-525 | 1600 | ~5 € | N–H combination (wool, nylon) |
| L8 | Roithner ELD-1650-525 | 1650 | ~5 € | C–H second overtone (all synthetics) |

Budget ~€30 for all 7 NIR LEDs + shipping from Vienna.

#### LED driver (all JLCPCB-assemblable)

| # | Part | MPN | LCSC# | ~Price |
|---|------|-----|-------|--------|
| M1 | 8:1 mux | 74HC4051D SOIC-16 | C6051 | 0.20 € |
| M2 | N-MOSFET | BSS138 SOT-23 | C112114 | 0.05 € ×8 |
| M3 | LED current R | 33 Ω SMD 0603 | — | 0.02 € ×8 |

### JLCPCB assembly split

JLCPCB assembles: RP2040, ESP32-C3, ADS1256 (Global Part), OPA2381, 74HC4051, BSS138 ×8,
TP4056, AMS1117, WS2812B, SD holder, all passives.

Solder by hand after board arrives: G8370-03 (TO-18), Roithner LEDs (5mm THT),
ILI9341 (pin header), LiPo (JST-PH).

### Total BOM cost

| Configuration | Parts cost | Notes |
|---|---|---|
| Learn mode | ~€80–100 | SparkFun breakout + shared board |
| Measure mode | ~€180–250 | InGaAs + LEDs + shared board |
| Both modes (shared PCB) | ~€220–280 | Swap sensor daughter-board |

---

## The enclosure — this is the hard part

For NIR reflectometry, the mechanical setup determines measurement quality more than the
electronics. Three things must be controlled:

**1. LED–sample–detector geometry**
The 8 LEDs and the photodiode must be positioned at fixed, repeatable angles relative to the
sample. FiberSense uses a 45°/0° geometry (LEDs at 45°, detector normal to surface) — the
standard for diffuse reflectance measurement. This is enforced by the sensor hat, which is a
light-tight 3D-printed cone that physically positions everything.

**2. Contact pressure**
The sensor hat must press against the fabric with enough force to flatten the surface but not
compress the weave. The enclosure uses a spring-loaded contact (3D-printed flexure, ~0.5N
preload). This is parametric in FreeCAD — adjustable by print parameters.

**3. Ambient light rejection**
The measurement chamber must be light-tight. Black PETG for the sensor hat. All internal
surfaces matte black. The measurement integration time (~5ms per LED) is short enough that
ambient light leakage is the dominant noise source if the seal is poor.

```
hardware/case/
├── fibersense_body.FCStd        Main enclosure (parametric, FreeCAD 0.21+)
├── sensor_hat_learn.FCStd       Learn mode hat (AS7265x contact geometry)
├── sensor_hat_measure.FCStd     Measure mode hat (LED array + detector geometry)
│                                ↳ parametric: LED angle, chamber depth, spring force
├── scan_table_arm.FCStd         Optional: fixed-position arm for table scanning
├── foot_pedal_housing.FCStd     Optional: 3D-printed foot pedal for headless mode
└── stl/                         Print-ready STL exports
```

Print settings: PETG preferred (dimensional stability). Sensor hat: 0.15mm layer, 40% infill,
black filament only. Body: 0.2mm layer, 20% infill, any color.

---

## Firmware

The RP2040 firmware is intentionally minimal. It does three things: acquire, calibrate, transmit.

```
firmware/
├── platformio.ini               RP2040 (Earle Philhower Arduino core) + ESP32-C3 stub
└── src/
    ├── config.h                 Pin defs, MODE_LEARN / MODE_MEASURE, feature flags
    ├── main.cpp                 State machine: IDLE → SCAN → TRANSMIT → IDLE
    ├── sensor_if.h              Common interface: ScanResult scan() + calibrate()
    ├── sensor_learn.cpp/h       AS7265x via I2C (SparkFun library)
    ├── sensor_measure.cpp/h     LED sequencer (74HC4051 + BSS138) + ADS1256 via SPI
    │                            ↳ RP2040 PIO for timing-deterministic SPI
    ├── calibration.cpp/h        Two-point white/dark, stored as JSON on SD
    ├── transmit.cpp/h           USB Serial JSON + relay to ESP32-C3 via UART
    ├── display.cpp/h            ILI9341: spectrum plot, status, last result from server
    └── led.cpp/h                WS2812B non-blocking status animations
```

The RP2040 does **not** run a classifier. It outputs:
```json
{
  "device": "fibersense-measure",
  "timestamp_ms": 1234567,
  "channels_nm": [940, 1050, 1200, 1300, 1450, 1550, 1600, 1650],
  "spectrum_raw": [18432, 21100, 9823, 15234, 4201, 8932, 9102, 7834],
  "spectrum_calibrated": [0.94, 0.89, 0.51, 0.72, 0.22, 0.47, 0.53, 0.43],
  "temperature_c": 23.4,
  "calibration_age_min": 12
}
```

The ESP32-C3 forwards this JSON over WiFi to the classification server and relays the response
back to the RP2040 for display. It has no application logic.

### Calibration

Two-point reflectance calibration. Stored on SD, survives reboots.

```
R_calibrated[i] = (raw[i] - dark[i]) / (white[i] - dark[i])
```

White reference: white PTFE card or 4 layers of white copier paper, pressed against sensor hat.
Dark reference: sensor hat covered completely.

Temperature warning: InGaAs dark current approximately doubles every 8°C. If the RP2040
internal temperature sensor reads >5°C change since last calibration, the display shows a
recalibration prompt. Re-calibration takes ~10 seconds.

---

## Classification server

Runs on a Raspberry Pi Zero 2W (or any laptop). Never flashed, always updateable.

```
server/
├── server.py                    Flask REST API: POST /classify → JSON result
├── classifier.py                k-NN + cosine distance, loads reference CSVs at startup
├── train.py                     Retrains model from reference library, no restart needed
├── requirements.txt             flask, numpy, scikit-learn, pandas
└── reference_spectra/
    ├── Cotton_100/              One CSV per sample (multiple rows = multiple measurements)
    ├── Polyester_100/
    ├── Wool_100/
    ├── Nylon_PA6/
    ├── Viscose/
    ├── Cotton_PET_5050/
    └── README.md               How to add new fiber types
```

The server exposes one endpoint:

```
POST /classify
Body: { spectrum_calibrated: [0.94, 0.89, 0.51, ...], channels_nm: [...] }

Response:
{
  "material": "Cotton",
  "confidence": 0.91,
  "top3": [
    {"material": "Cotton", "distance": 0.04},
    {"material": "Viscose", "distance": 0.11},
    {"material": "Cotton_PET_5050", "distance": 0.19}
  ],
  "warning": null
}
```

Adding a new fiber type: collect 10+ scans of a known sample using
`python tools/collect_reference.py`, put the CSV in `reference_spectra/NewFiber/`,
run `python train.py`. No hardware interaction required.

---

## Reference data

### Using the OpenTextile-NIR dataset (Zenodo)

Sormunen et al. (2026) published 71 labeled post-industrial textile samples with both NIR
hyperspectral data and RGB values. DOI: [10.5281/zenodo.18269172](https://doi.org/10.5281/zenodo.18269172).

The SWIR hyperspectral data (1000–2500 nm) is not directly usable for FiberSense calibration —
the wavelength ranges do not overlap. However:

- The **RGB mean values** (`rgb_mean_values.csv`, 71 samples, labeled) are directly usable
  to bootstrap the learn-mode color classifier
- The **fiber composition ground truth** (`ground_truth_final.csv`) provides a validated
  reference for what percentage accuracies are realistic to expect
- The **outlier list** (11 samples flagged) is useful: it shows that even professional
  measurements on labeled garments have ~15% label errors — set accuracy expectations accordingly

```
data/
├── zenodo/
│   ├── ground_truth_final.csv       71 samples, fiber % per type
│   ├── rgb_mean_values.csv          RGB values for learn-mode bootstrap
│   └── swir_mean_spectra.csv        Full NIR spectra (reference only, not for FiberSense)
└── reference_spectra/
    ├── learn_mode/                  18-channel CSVs (410–940 nm, collected with FiberSense)
    └── measure_mode/                8-channel CSVs (940–1650 nm, collected with FiberSense)
```

### Starter reference library

The repo includes a small starter library collected on a Measure mode prototype. It is a
starting point, not a validated scientific dataset. Accuracy will improve as more samples
are added.

**Contribute:** If you collect reference spectra on known-composition samples (manufacturer
label preferred, certified label ideal), please contribute via pull request to
`data/reference_spectra/`. Include the ground truth source in a README alongside the CSV.

---

## Expected accuracy (Measure mode, starter library)

| Fiber | Accuracy | Main diagnostic | Known failure mode |
|-------|----------|----------------|-------------------|
| Cotton 100% | ~93% | 1450nm O–H high | Confused with lyocell |
| Polyester 100% | ~96% | 1200nm C–H high | — |
| Wool 100% | ~88% | 1450nm + 1600nm | Varies with washing history |
| Nylon PA6/66 | ~85% | 1550nm + 1600nm | PA6 and PA66 similar |
| Viscose/Lyocell | ~82% | Cotton-like, slight shift | Often confused with cotton |
| Cotton/PET 50/50 | ~75% | Ratio 1450/1200nm | Blend % estimate ±15% |
| Acrylic (PAN) | not supported | Key band at 2240nm | Outside G8370 range |
| Silk | marginal | Protein bands weak at 1650nm cutoff | — |

---

## Workshop formats

FiberSense is designed for three distinct workshop formats:

**"Why doesn't this work?" (2h, learn mode)**
Participants scan white cotton and white polyester with learn mode. The classifier struggles.
They look at the spectrum plot and see that the reflectance curves look nearly identical at
410–940nm. Discussion: what wavelengths would you need? What's the molecular difference?
Then the measure-mode board is demonstrated. This is the most educational sequence.

**"Build the instrument" (4–6h, measure mode PCB)**
Participants solder the Roithner LEDs and Hamamatsu photodiode onto a pre-assembled PCB
(JLCPCB assembled the SMD parts). Flash firmware, run calibration, take first scan.
Emphasis on the optical-mechanical setup: why does the sensor hat geometry matter?

**"Train the classifier" (3h, any mode)**
Participants collect reference spectra from a sample set with known composition.
Run `train.py`, inspect the confusion matrix, add more samples for weak classes,
re-train. Discussion: what does accuracy mean when the ground truth labels may be wrong?

---

## Roadmap

**v1.0 (this repo)**
Measure mode hardware validated. 6 fiber types. Classification server on RPi Zero 2W.

**v1.1**
Scan table arm. Foot pedal trigger. Batch scan mode (CSV export for multiple samples).

**v2.0 — extended range**
Second detector channel: Hamamatsu G9208-256W (InGaAs array, 1.1–2.15µm, SPI-controlled).
Adds acrylic (2240nm C≡N band becomes accessible), improves blend quantification.
PCB connector reserved on v1.0 board for this expansion.

---

## Licenses

- **Hardware** (KiCad schematics, PCB, FreeCAD enclosure): [CERN-OHL-S v2](LICENSE-HARDWARE)
- **Firmware**: [MIT License](LICENSE-FIRMWARE)
- **Documentation & reference data**: [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

---

## References

1. Sormunen T. et al. (2026). *OpenTextile-NIR: Near-infrared hyperspectral imaging and
   photography dataset for optical identification of textiles.* Data in Brief 65, 112559.
   DOI: 10.1016/j.dib.2026.112559
2. Zenodo dataset: doi.org/10.5281/zenodo.18269172
3. Hamamatsu G8370 series datasheet — InGaAs PIN photodiode, 0.9–1.7µm, TO-18
4. Texas Instruments ADS1256 datasheet — 24-bit Delta-Sigma ADC
5. Roithner Lasertechnik ELD-series — roithner-laser.com
6. RP2040 datasheet — PIO state machines for timing-deterministic SPI

---

*FiberSense — open hardware for textile fiber identification.*
*Contributions welcome. See CONTRIBUTING.md.*
