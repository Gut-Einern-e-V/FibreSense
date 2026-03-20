# 🤖 FiberSense — AI Agent Prompt Set

> Six independent prompts. Each generates one complete component.
> Start with Prompt 1, then 2. The rest are optional extensions.

---

## SYSTEM PROMPT (prepend to every prompt below)

```
You are an experienced embedded-systems and Python developer.
Output complete, directly usable files. Never partial snippets unless asked.
C++: Arduino framework, RP2040 Earle Philhower core, PlatformIO.
Python: 3.10+, include requirements.txt with every script.
All constants in config.h or top of file. No magic numbers.
No blocking delays in firmware — millis()-based timing only.
All public functions: brief Doxygen /** @brief */ comment.
```

---

## PROMPT 1 — RP2040 firmware

```
Generate complete firmware for FiberSense, a handheld textile NIR scanner.

Hardware base: Waveshare RP2040-Touch-LCD-1.28
  - RP2040 dual-core, 133MHz
  - GC9A01A 1.28" round IPS display 240×240, SPI (BOARD-INTERNAL, fixed pins)
    CS=GPIO9, DC=GPIO8, SCK=GPIO10, MOSI=GPIO11, RST=GPIO12, BL=GPIO25
  - CST816S capacitive touch, I2C0 (BOARD-INTERNAL): SDA=GPIO6, SCL=GPIO7, INT=GPIO21
  - QMI8658C 6-axis IMU, I2C0 (BOARD-INTERNAL): SDA=GPIO4, SCL=GPIO5
  - LiPo charging: internal, no GPIO needed
  - USB-C: USB Serial via TinyUSB

Sensor PCB connected via 1.27mm header to free GPIOs:
  AS7265x spectral sensor, I2C1: SDA=GPIO13, SCL=GPIO14
  ADS1256 24-bit ADC, SPI via PIO:
    CS=GPIO15, DRDY=GPIO16, MOSI=GPIO17, MISO=GPIO18, SCK=GPIO19
  74HC4051 LED mux select: A=GPIO20, B=GPIO22, C=GPIO26
  LED enable MOSFET: GPIO27 (HIGH = LED on during measurement only)
  Trigger button: GPIO28 (active LOW, internal pull-up)
  LiPo ADC: GPIO29 (100k/100k divider, 4.2V → 2.1V)

════════════════════════
SENSOR AUTO-DETECTION (sensor_detect.cpp/h)
════════════════════════

On boot, probe both buses:

  bool learn_present = false;
  bool measure_present = false;

  // I2C1 scan: AS7265x responds at address 0x49
  Wire1.begin(13, 14);
  Wire1.beginTransmission(0x49);
  learn_present = (Wire1.endTransmission() == 0);

  // SPI probe: ADS1256 — assert CS, wait, check DRDY goes LOW within 100ms
  // If DRDY responds → ADS1256 present
  measure_present = ads1256_probe(GPIO15, GPIO16);

  SensorMode mode:
    LEARN     = learn_present && !measure_present
    MEASURE   = !learn_present && measure_present
    COMBINED  = learn_present && measure_present
    NONE      = neither (→ ERROR state)

Display detected mode for 2 seconds on boot, then proceed.

════════════════════════
DATA STRUCTURE
════════════════════════

  struct ScanResult {
    float channels_calibrated[26]; // max: 18 learn + 8 measure
    float channels_raw[26];
    uint16_t wavelengths_nm[26];
    uint8_t n_channels;
    SensorMode mode;
    uint32_t timestamp_ms;
    float temperature_c;           // RP2040 internal temp
    uint32_t calibration_age_ms;
    bool calibration_warning;      // true if temp drifted >5°C since cal
    bool motion_rejected;          // true if IMU detected movement during scan
    bool valid;
  };

  struct ClassifyResult {
    char material[32];
    float confidence;
    struct { char material[32]; float distance; } top3[3];
    bool low_confidence;           // true if confidence < 0.6
  };

════════════════════════
STATE MACHINE (main.cpp)
════════════════════════

  States: BOOT, DETECT, CALIBRATION_CHECK, IDLE, SCANNING, CLASSIFYING, RESULT, ERROR

  BOOT:
    Init display, show logo 500ms.
    Init IMU. Init touch.

  DETECT:
    Run sensor auto-detection (see above).
    Show mode on display 2s. Load calibration from SD.
    If no calibration: → CALIBRATION_CHECK state (prompt to calibrate).
    Else: → IDLE.

  IDLE:
    Display last ClassifyResult (or "Ready" if none).
    Spectrum arc plot of last scan.
    Battery + calibration age indicators.
    Check: IMU flat? If tilted >45°: show tilt icon, disable trigger.
    Wait for: button press (GPIO28) OR touch tap on display center.

  SCANNING:
    Check IMU orientation one more time. If tilted: cancel → IDLE.
    Start scan (sensor-specific, see below).
    Poll IMU during scan: if accel > 0.3g → cancel, show "Hold still", retry.
    WS2812B: yellow (if present — optional).
    Display: animated ring sweeping, channel markers lighting up one by one.

  CLASSIFYING:
    Apply calibration: R[i] = (raw[i] - dark[i]) / (white[i] - dark[i]).
    Clamp to 0.0–1.2.
    Run k-NN classifier against reference CSVs on SD.
    → RESULT.

  RESULT:
    Display material name (large), confidence arc, top 3 alternatives.
    Save scan to SD: /data/scans.csv (append CSV row).
    Output JSON to USB Serial.
    After 5s or tap: → IDLE.

  ERROR:
    Display error message + icon.
    If mode == NONE: "No sensor found. Check PCB connection."
    If no calibration: "Calibrate first. Hold button 3s."
    Hold button 3s: → CALIBRATION flow (blocking, guided).

════════════════════════
SENSOR — LEARN MODE (sensor_learn.cpp)
════════════════════════

  Library: SparkFun_AS7265x
  Init on Wire1 (GPIO13/14). Read all 18 calibrated channels.
  Wavelengths_nm: 410,435,460,485,510,535,560,585,610,645,680,705,730,760,810,860,900,940
  Integration time: 50ms. n_channels = 18.

════════════════════════
SENSOR — MEASURE MODE (sensor_measure.cpp)
════════════════════════

  ADS1256 driver (write from scratch using RP2040 PIO for SPI):

  PIO state machine for SPI:
    Use pio_spi from pico-extras or equivalent.
    Clock: 1MHz (ADS1256 max 1.92MHz at AVDD=3.3V).
    CPOL=0, CPHA=1 (SPI mode 1), MSB first.
    Reason for PIO: timing-deterministic, unaffected by other interrupts.

  ADS1256 init sequence:
    1. Assert CS
    2. Send RESET command (0xFE), wait 100µs
    3. Write ADCON register: gain=64 (PGA_64)
    4. Write DRATE register: 30000 SPS
    5. Send SELFCAL (0xF0), wait DRDY
    6. Deassert CS

  readMean(n_samples):
    Take n_samples readings via RDATA command.
    Each reading: assert CS, wait DRDY LOW (timeout 100ms), send RDATA (0x01),
    read 3 bytes (24-bit signed), deassert CS.
    Discard top and bottom 10%. Return mean as float.

  LED sequencing (8 channels):
    wavelengths_nm = {940, 1050, 1200, 1300, 1450, 1550, 1600, 1650}
    for i in 0..7:
      Set 74HC4051 A/B/C = binary encoding of i (GPIO20/22/26)
      Set LED enable HIGH (GPIO27)
      Wait 3ms (photodiode settle + LED warm-up)
      raw[i] = readMean(32)
      Set LED enable LOW (GPIO27)
      Wait 1ms
    n_channels = 8. Total scan time: ~80ms.

════════════════════════
SENSOR — COMBINED MODE (sensor_combined.cpp)
════════════════════════

  Run sensor_learn.scan() → 18 channels
  Run sensor_measure.scan() → 8 channels
  Merge into single ScanResult: learn channels first, then measure channels.
  Total n_channels = 26, wavelengths_nm merged and sorted.

════════════════════════
IMU (imu.cpp)
════════════════════════

  Library: QMI8658C (write minimal driver, I2C0 on GPIO4/5, address 0x6B)
  
  isFlat(): returns true if |pitch| < 45° and |roll| < 45°
    Use accel X/Y/Z, compute tilt angle: atan2(sqrt(ax²+ay²), az)
  
  getAccelMagnitude(): returns sqrt(ax²+ay²+az²) in g
    During scan: poll at 50ms intervals. If > 1.3g (0.3g above 1g gravity): motion flag.
  
  detectShake(): 3 readings > 2.0g within 500ms → returns true
    Used to trigger recalibration prompt.
  
  isLifted(): accel Z drops below 0.7g → device tilted significantly / lifted
    Used for lift-to-wake (display wakes from dim).

════════════════════════
TOUCH (touch.cpp)
════════════════════════

  CST816S minimal driver (I2C0, GPIO6/7, INT=GPIO21)
  Read gesture register: 0x01=swipe up, 0x02=down, 0x03=left, 0x04=right, 0x05=tap
  detectTap(): returns true on single tap anywhere on display
  detectDoubleTap(): returns true on double tap
  Double tap → clear result, return to IDLE.

════════════════════════
DISPLAY (display_round.cpp)
════════════════════════

  Library: Adafruit GC9A01A (or equivalent GC9A01 library for RP2040)
  240×240 round IPS. Use DMA for transfers to keep scan animation smooth.

  drawIdle(ClassifyResult& last):
    Background: black.
    Center: material name in white, large font (2 lines if needed).
    Below name: confidence percentage, smaller font, colored (green/yellow/red).
    Outer ring: filled arc 0–360° * confidence, color = confidence level.
      Green if > 0.8, yellow if 0.6–0.8, red if < 0.6.
    Radial spectrum plot:
      Map each wavelength to an angular position around the display center.
      Draw radial bar from center outward, height proportional to calibrated reflectance.
      Bar color: gradient from blue (short λ) to red (long λ).
    Bottom center: battery icon + calibration age in minutes.
    Top center: tilt warning icon if IMU says not flat.

  drawScanning(uint8_t channel_complete, uint8_t n_total):
    Sweeping arc animation around the rim.
    Each completed channel: a bright dot at its angular position snaps into place.
    Center: "Scanning..." text.
    Progress: arc from 0 to (channel_complete/n_total * 360°).

  drawResult(ClassifyResult& result):
    Brief full-screen flash of result color (green/yellow/red, 200ms).
    Then drawIdle(result).
    Top 3 alternatives: small text, bottom quarter of display.

  drawCalibration(uint8_t step):
    Split circle: left half = white ref (fills white when done),
    right half = dark ref (fills dark gray when done).
    Center: step instruction text.

  drawError(const char* message):
    Red background fill. White error text centered. Icon.

  drawBootLogo():
    "FiberSense" centered, large. Thin arc animating around rim. 500ms.
    Then fade to DETECT state.

════════════════════════
CALIBRATION (calibration.cpp)
════════════════════════

  Triggered by: hold button 3s OR shake gesture OR Settings touch menu.

  Procedure:
    Step 1: drawCalibration(1) → "Place white card. Tap when ready."
      On tap: scan() → store as white[]. Left half of circle fills.
    Step 2: drawCalibration(2) → "Cover sensor. Tap when ready."
      On tap: scan() → store as dark[]. Right half of circle fills.
    Save to SD: /cal/white.json and /cal/dark.json
      JSON: {"wavelengths_nm":[...],"values":[...],"temp_c":23.4,"timestamp_ms":...}
    Show "Calibration complete" → → IDLE.

  apply(ScanResult& raw) → fills raw.channels_calibrated[]:
    calibrated[i] = (raw[i] - dark[i]) / (white[i] - dark[i])
    Clamp to 0.0–1.2.
    If |current_temp - cal_temp| > 5.0: set raw.calibration_warning = true.

════════════════════════
CLASSIFIER (classifier.cpp)
════════════════════════

  On boot: scan /ref/ directory on SD. Load all CSVs.
  CSV format: header row = wavelengths_nm, data rows = calibrated reflectance values.
  One subdirectory per class. Multiple CSVs per class OK.

  classify(ScanResult& scan) → ClassifyResult:
    For each reference sample: compute cosine distance to scan.channels_calibrated[].
    Sort by distance ascending. Take k=5 nearest.
    Winner = class with most votes in k nearest.
    Confidence = votes_for_winner / k.
    Return top 3 classes with their distances.
    If confidence < 0.6: set low_confidence = true.

  Memory: pre-allocate static arrays. Max 20 classes × 30 samples each.
  If SD missing: classifier returns {"material":"No SD","confidence":0}.

════════════════════════
TRANSMIT (transmit.cpp)
════════════════════════

  On each completed scan+classify: output to USB Serial (115200 baud):

  {
    "device": "fibersense",
    "mode": "measure",
    "timestamp_ms": 1234567,
    "channels_nm": [940,1050,1200,1300,1450,1550,1600,1650],
    "spectrum_raw": [18432,21100,9823,15234,4201,8932,9102,7834],
    "spectrum_calibrated": [0.94,0.89,0.51,0.72,0.22,0.47,0.53,0.43],
    "material": "Cotton",
    "confidence": 0.91,
    "top3": [
      {"material":"Cotton","distance":0.04},
      {"material":"Viscose","distance":0.12},
      {"material":"Cotton_PET_5050","distance":0.21}
    ],
    "temperature_c": 23.4,
    "calibration_age_ms": 720000,
    "calibration_warning": false,
    "motion_rejected": false
  }

════════════════════════
OUTPUT FILES
════════════════════════

  1.  platformio.ini
  2.  src/config.h
  3.  src/main.cpp
  4.  src/sensor_detect.h + sensor_detect.cpp
  5.  src/sensor_learn.h + sensor_learn.cpp
  6.  src/sensor_measure.h + sensor_measure.cpp   (includes full ADS1256 PIO driver)
  7.  src/sensor_combined.h + sensor_combined.cpp
  8.  src/imu.h + imu.cpp
  9.  src/touch.h + touch.cpp
  10. src/display_round.h + display_round.cpp
  11. src/calibration.h + calibration.cpp
  12. src/classifier.h + classifier.cpp
  13. src/transmit.h + transmit.cpp
```

---

## PROMPT 2 — Reference collection tool (PC Python)

```
Generate tools/collect_reference.py

Guided CLI for collecting reference spectra from a known-composition sample.

CLI:
  python collect_reference.py \
    --port /dev/ttyUSB0 \
    --material "Cotton_100" \
    --n 20 \
    --output ./data/reference_spectra/measure_mode/Cotton_100/

Behaviour:
  1. Open serial port at 115200. Wait for FiberSense JSON lines.
  2. Print: "Press trigger button on device. (N remaining)"
  3. Parse incoming JSON. Extract spectrum_calibrated + channels_nm + mode.
  4. Reject scan if: motion_rejected=true OR calibration_warning=true (print warning, ask retry).
  5. Live matplotlib plot: all accepted scans overlaid. Running mean as thick line.
  6. After n accepted scans:
     - Show per-channel mean ± std dev.
     - Flag channels where std dev > 0.05 (high variance — likely measurement problem).
     - Prompt: "Accept these N scans? [y/n/retry]"
  7. On accept: save as single CSV file.
     Filename: {material}_{timestamp}.csv
     Format: header row = wavelengths (from channels_nm), one data row per scan.
  8. Print: "Saved {n} scans to {output}. Total samples for {material}: {count}"
     Print: "Run tools/train_classifier/train.py to retrain."

Handles both learn mode (18ch) and measure mode (8ch) JSON automatically.
Requirements: pyserial, matplotlib, numpy, pandas
```

---

## PROMPT 3 — Classifier trainer (PC Python)

```
Generate tools/train_classifier/train.py

CLI: python train.py --input ./data/reference_spectra/measure_mode/ [--report] [--embed]

Steps:
  1. Load all CSVs from input directory.
     Directory structure: input/ClassName/sample_XXX.csv
     Each CSV: header = wavelengths_nm, rows = measurements.
     Print: loaded N samples across M classes.

  2. Outlier removal per class:
     Z-score per channel. Remove rows where any channel z-score > 3.
     Print how many samples removed per class.

  3. Train three models on full dataset:
     - k-NN (k=1, 3, 5), cosine metric
     - SVM, RBF kernel, C=10, gamma=scale
     - Random Forest, n_estimators=100
     5-fold cross-validation for each. Print accuracy + std dev.

  4. Print confusion matrix for best model (highest mean CV accuracy).

  5. Save best model as model.pkl (loaded by classify_server.py if used).

  6. --report flag: save train_report.md with:
     - Class sample counts
     - CV results table
     - Confusion matrix as markdown table
     - Per-class precision/recall

  7. --embed flag: also export classifier_weights.h for firmware embedding:
     Format usable by classifier.cpp without SD card:
     static const float REFERENCE_MEANS[N_CLASSES][N_CHANNELS] = {...};
     static const char* CLASS_NAMES[] = {...};
     #define N_REF_CLASSES N
     #define N_REF_CHANNELS N

Requirements: scikit-learn, pandas, numpy, matplotlib
```

---

## PROMPT 4 — Real-time visualiser (PC Python)

```
Generate tools/visualizer.py

Live spectrum viewer reading USB serial JSON from FiberSense.

CLI: python visualizer.py --port /dev/ttyUSB0 [--ref ./data/reference_spectra/measure_mode/]

Layout (matplotlib, 3 panels, live updating at ~2Hz):

  Left panel — spectrum:
    x = wavelength nm. y = calibrated reflectance 0–1.2.
    Current scan: solid colored line.
    Previous 5 scans: faded (alpha 0.5 → 0.1).
    If --ref: dashed lines for mean spectrum of each reference class.
    Vertical dotted lines at key wavelengths: 1200, 1450, 1650nm with labels.

  Center panel — classification:
    Last result material name (large).
    Horizontal bar chart: top 3 classes, bar length = 1 - distance (similarity).
    Color: green if confidence > 0.8, yellow if 0.6–0.8, red if < 0.6.
    Motion rejected / calibration warning flags shown as icons.

  Right panel — session history:
    Last 50 scans: scatter plot, x=scan number, y=confidence.
    Color by classified material (consistent color per class).
    Rolling accuracy if ground truth labels provided (--truth flag).

Keyboard: s=save PNG, c=clear history, q=quit.
Auto-reconnect on serial disconnect (retry every 2s).
Save session CSV on quit: session_YYYYMMDD_HHMMSS.csv

Requirements: pyserial, matplotlib, numpy, pandas
```

---

## PROMPT 5 — ADS1256 PIO driver (standalone, testable)

```
Generate a standalone ADS1256 driver for RP2040 using PIO state machines.

Files:
  lib/ADS1256_PIO/ADS1256_PIO.h
  lib/ADS1256_PIO/ADS1256_PIO.cpp
  lib/ADS1256_PIO/ads1256_spi.pio    (PIO assembly for SPI)

No external library dependencies. No FiberSense dependencies.

PIO SPI:
  Implement SPI mode 1 (CPOL=0, CPHA=1) using RP2040 PIO.
  Clock: configurable, default 1MHz.
  MSB first. 8-bit words.
  Use sm_config_set_out_pins, sm_config_set_in_pins, set_sideset.
  TX FIFO threshold 1, RX FIFO threshold 1, autopull/autopush enabled.

API:
  ADS1256_PIO(uint8_t pin_mosi, uint8_t pin_miso, uint8_t pin_sck,
              uint8_t pin_cs, uint8_t pin_drdy)
  bool begin(ADS1256_SPS sps, ADS1256_GAIN gain)
  int32_t readChannel(uint8_t ch)         // 0–7 single-ended
  int32_t readDifferential(uint pos, uint neg)
  float toVoltage(int32_t raw, float vref=2.5f)
  bool selfCalibrate()
  bool waitDRDY(uint32_t timeout_ms=200)
  bool probe()    // returns true if chip responds (for sensor auto-detection)

  Enums:
    ADS1256_SPS: SPS_30, SPS_100, SPS_500, SPS_1000, SPS_3750, SPS_7500, SPS_15000, SPS_30000
    ADS1256_GAIN: GAIN_1, GAIN_2, GAIN_4, GAIN_8, GAIN_16, GAIN_32, GAIN_64

Implementation details:
  RESET command (0xFE): pulse RESET pin or send via SPI.
  WREG: 0x50 | reg_addr, then n_regs-1, then data bytes.
  RREG: 0x10 | reg_addr, then n_regs-1, then read.
  RDATA: 0x01 — wait DRDY low before sending, read 3 bytes.
  SELFCAL: 0xF0 — send command, wait DRDY (up to 1s).
  CS timing: hold CS low for entire transaction including inter-byte gaps.

Test sketch: examples/ads1256_pio_test/ads1256_pio_test.ino
  Init 1000 SPS, gain 1. Print channel 0 voltage at 10Hz. Print probe() result on boot.
```

---

## CALIBRATION CONTEXT (append to any prompt involving calibration)

```
FiberSense two-point reflectance calibration:

White reference: PTFE diffuse reflector or 4 layers white copier paper.
  Press flat against sensor hat. Must be flat and fully covering the measurement aperture.
Dark reference: sensor hat fully blocked. Black cloth cap or covered with hand.

Formula per channel:
  R_calibrated[i] = (raw[i] - dark[i]) / (white[i] - dark[i])

Valid range: 0.0 (no reflectance) to ~1.2 (slightly more reflective than white ref).
Clamp to 0.0–1.2, not 0.0–1.0. Values above 1.0 are physically valid.

Temperature dependency: InGaAs dark current doubles approximately every 8°C.
Monitor RP2040 internal temperature sensor (temperatureRead()).
If current temperature deviates >5°C from calibration temperature:
  Set calibration_warning=true in ScanResult.
  Show orange arc animation on idle display.
  Do not block scanning — warn only.

Recommended practice: recalibrate at start of each session.
Calibration takes ~10 seconds total.
```

---

*FiberSense Agent Prompt Set — CC BY-SA 4.0*
*Fork of reremeter (GPL-3.0) by Armin Straller & Bernhard Gessler.*
