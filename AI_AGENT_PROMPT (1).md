# 🤖 AI Agent Prompt — FiberSense Code Generator

> Modular prompts. Use them independently or in sequence.
> Each prompt generates one complete, functional component.
> Start with Prompt 1 (RP2040 firmware), then Prompt 2 (classification server).

---

## SYSTEM PROMPT (prepend to all prompts below)

```
You are an experienced embedded-systems and Python developer.
You write production-quality, well-commented code.
You output complete, directly usable files — never partial snippets unless asked.
For C++: use the Arduino framework on RP2040 (Earle Philhower core), PlatformIO.
For Python: target Python 3.10+, include requirements.txt with every script.
No magic numbers — all constants in config files or at the top of the file.
No blocking delays in firmware — millis()-based timing throughout.
All functions have brief Doxygen comments.
```

---

## PROMPT 1 — RP2040 firmware (measurement MCU)

```
Generate the complete RP2040 firmware for FiberSense, a LED-sequential NIR reflectometer.

The RP2040 has one job: acquire calibrated spectra and transmit them as JSON.
It does NOT classify. It does NOT run WiFi. It does not make decisions.

════════════════════════
HARDWARE
════════════════════════

Mode selected at compile time in config.h:
  MODE_LEARN    → AS7265x via I2C
  MODE_MEASURE  → ADS1256 + LED array via SPI + PIO

Shared peripherals:
  ILI9341 SPI:     CS=GPIO10, DC=GPIO11, RST=GPIO12, MOSI=GPIO13, SCK=GPIO14
  SD card SPI:     CS=GPIO5, MOSI=GPIO6, MISO=GPIO7, SCK=GPIO15
  Trigger button:  GPIO0 (active LOW, internal pull-up)
  WS2812B:         GPIO28 (1 pixel)
  ESP32-C3 UART:   TX=GPIO0, RX=GPIO1 (UART1, 115200 baud)
  USB Serial:      UART0 via USB — debug + direct JSON output

MODE_LEARN only:
  AS7265x I2C:     SDA=GPIO4, SCL=GPIO5

MODE_MEASURE only:
  ADS1256 SPI:     CS=GPIO22, DRDY=GPIO21, MOSI=GPIO19, MISO=GPIO20, SCK=GPIO18
  74HC4051 select: A=GPIO26, B=GPIO27, C=GPIO28
  LED enable:      GPIO29 (HIGH = LED on during measurement only)

════════════════════════
DATA STRUCTURE
════════════════════════

  struct ScanResult {
    float channels_calibrated[32];
    float channels_raw[32];
    uint16_t wavelengths_nm[32];
    uint8_t n_channels;
    uint32_t timestamp_ms;
    float temperature_c;
    uint32_t calibration_age_ms;
    bool calibration_warning;
    bool valid;
  };

════════════════════════
STATE MACHINE
════════════════════════

  BOOT → CALIBRATION_CHECK → IDLE → SCANNING → TRANSMITTING → IDLE
                                                     ↓ on error
                                                   ERROR

  BOOT:              Init peripherals, check SD, load calibration from SD.
  CALIBRATION_CHECK: No calibration file → ERROR. Age > 30min → warning flag.
  IDLE:              Show last result (received from server). Wait for button or UART trigger.
  SCANNING:          Acquire spectrum. Apply calibration. WS2812B: yellow blink.
  TRANSMITTING:      Send JSON via USB Serial AND ESP32-C3 UART.
                     Wait up to 3s for classification result back via UART.
                     Update display. Append raw scan to /data/scans.csv on SD.
  ERROR:             Show error message. WS2812B: red. Button retries.

════════════════════════
SENSOR — MODE_LEARN (sensor_learn.cpp)
════════════════════════

  SparkFun_AS7265x library. Read all 18 channels.
  Wavelengths: 410,435,460,485,510,535,560,585,610,645,680,705,730,760,810,860,900,940 nm.
  Integration time: 50ms. Return ScanResult with n_channels=18.

════════════════════════
SENSOR — MODE_MEASURE (sensor_measure.cpp)
════════════════════════

  Write ADS1256 driver from scratch (no external library):
    - Init: SPS=30000, gain=64, single-ended channel 0
    - readMean(n): take n readings, discard top+bottom 10%, return mean as float
    - DRDY polling with 100ms timeout (return NaN on timeout)
    - Use RP2040 PIO state machine for SPI (timing-deterministic, no WiFi jitter)

  LED sequencing per channel (8 channels):
    For i in 0..7:
      Set 74HC4051 A/B/C select bits for index i
      Set LED enable HIGH
      Wait 3ms settle
      Call readMean(32) → raw[i]
      Set LED enable LOW
      Wait 1ms
  
  Wavelengths: [940, 1050, 1200, 1300, 1450, 1550, 1600, 1650]

════════════════════════
CALIBRATION (calibration.cpp)
════════════════════════

  Two-point: white + dark reference.
  Formula: calibrated[i] = (raw[i] - dark[i]) / (white[i] - dark[i])
  Clamp to 0.0–1.2.
  Storage: /cal/white.json and /cal/dark.json on SD.
  
  Triggered by holding button 3 seconds:
    Step 1: "Place white reference. Press button." → scan → save white
    Step 2: "Cover sensor. Press button." → scan → save dark

  Temperature warning: if |current_temp - calibration_temp| > 5.0°C → set calibration_warning=true.

════════════════════════
JSON OUTPUT (transmit.cpp)
════════════════════════

  Emit newline-terminated JSON to USB Serial AND ESP32-C3 UART:

  {
    "device": "fibersense",
    "mode": "measure",
    "timestamp_ms": 1234567,
    "channels_nm": [940, 1050, 1200, 1300, 1450, 1550, 1600, 1650],
    "spectrum_raw": [18432, 21100, 9823, 15234, 4201, 8932, 9102, 7834],
    "spectrum_calibrated": [0.94, 0.89, 0.51, 0.72, 0.22, 0.47, 0.53, 0.43],
    "temperature_c": 23.4,
    "calibration_age_ms": 720000,
    "calibration_warning": false
  }

  Classification result received back via ESP32-C3 UART:
  { "material": "Cotton", "confidence": 0.91, "top3": [...] }
  → display on ILI9341.

════════════════════════
DISPLAY (display.cpp)
════════════════════════

  Adafruit_ILI9341 + Adafruit_GFX.

  showIdle(last_result):
    Material name large. Confidence bar (green >0.8, yellow 0.6–0.8, red <0.6).
    Mini line chart: x=wavelength nm, y=calibrated reflectance.
    Status bar: calibration age, temperature, SD icon, WiFi icon.

  showScanning():
    "Scanning..." + animated dot per LED channel completing.

  showError(message):
    Red background. Error text. "Hold 3s to recalibrate."

════════════════════════
LED (led.cpp)
════════════════════════

  FastLED, non-blocking, millis()-based.
  IDLE: slow blue pulse (2s). SCANNING: yellow 2Hz. OK: green 500ms → idle.
  ERROR: red 3× blink → solid red. UNCALIBRATED: orange pulse. TRANSMITTING: white 10Hz.

════════════════════════
OUTPUT FILES
════════════════════════

  1. platformio.ini
  2. src/config.h
  3. src/sensor_if.h
  4. src/main.cpp
  5. src/sensor_learn.h + sensor_learn.cpp
  6. src/sensor_measure.h + sensor_measure.cpp  (includes full ADS1256 driver)
  7. src/calibration.h + calibration.cpp
  8. src/transmit.h + transmit.cpp
  9. src/display.h + display.cpp
  10. src/led.h + led.cpp
```

---

## PROMPT 2 — Classification server (runs on RPi Zero 2W or any PC)

```
Generate the complete classification server for FiberSense.

Receives JSON spectra via HTTP POST, classifies fiber, returns JSON result.
Never touches hardware. Retrained by dropping CSV files into a folder.

════════════════════════
FILES
════════════════════════

server.py
  Flask, port 5000, CORS enabled.
  POST /classify    → FiberSense JSON → ClassifyResult JSON
  POST /retrain     → reload CSVs, retrain, return accuracy report JSON
  GET  /references  → list loaded classes + sample counts
  GET  /health      → uptime, loaded classes, last scan timestamp
  Thread-safe: retraining uses a read-write lock, classify runs concurrently with old model.

classifier.py
  load_references(path):
    Walk reference_spectra/ — each subdirectory = one class.
    Each CSV: header row = wavelengths_nm, data rows = measurements.
    Return dict: class_name → np.array(n_samples, n_channels).

  train(references) → (model, label_encoder):
    Build X (n_total × n_channels) and y arrays.
    Fit sklearn NearestNeighbors(n_neighbors=5, metric='cosine').

  classify(spectrum_calibrated, channels_nm, model) → dict:
    Interpolate to standard wavelength grid if channels differ.
    Return:
    {
      "material": str,
      "confidence": float,
      "top3": [{"material": str, "distance": float}],
      "warning": str or null
    }
    If confidence < 0.6: warning = "confidence below threshold, result uncertain"

train.py  (standalone, for offline evaluation)
  CLI: python train.py --input ./reference_spectra/ --report
  Train k-NN + SVM + Random Forest. 5-fold cross-validation.
  Print confusion matrix for best model.
  Save model.pkl. With --report: save train_report.md.

requirements.txt: flask, numpy, scikit-learn, pandas, scipy

════════════════════════
REFERENCE CSV FORMAT
════════════════════════

  reference_spectra/Cotton_100/sample_001.csv:
  940,1050,1200,1300,1450,1550,1600,1650
  0.94,0.89,0.51,0.72,0.22,0.47,0.53,0.43
  0.93,0.88,0.52,0.71,0.23,0.46,0.54,0.44

  Multiple rows per file = repeated measurements of same physical sample.
  Multiple files per class = different physical samples of same fiber type.
```

---

## PROMPT 3 — ESP32-C3 WiFi bridge (dumb serial relay)

```
Generate minimal firmware for ESP32-C3 as a serial-to-WiFi bridge.
No application logic. No classification. Forwards JSON between RP2040 and server.

Hardware:
  UART from RP2040: RX=GPIO6, TX=GPIO7, 115200 baud
  WiFi: station mode via WiFiManager
  Status LED: GPIO5

Behaviour:
  On receiving complete JSON line (\n terminated) from RP2040:
    POST to http://{SERVER_IP}:5000/classify, timeout 2000ms.
    On success: forward response JSON to RP2040 UART + \n
    On failure: send {"error":"server_unreachable"} + \n

  SERVER_IP stored in NVS. Set via serial command: CONFIG_SERVER=192.168.1.100
  WiFiManager on first boot → captive portal "FiberSense-WiFi".
  Send WiFi status to RP2040 every 5s:
    {"wifi_status":"connected","rssi":-62,"server_ip":"192.168.1.100"}

Output: platformio.ini + src/main.cpp (~150 lines).
```

---

## PROMPT 4 — Reference collection tool (PC Python)

```
Generate tools/collect_reference.py

Guided CLI tool for collecting reference spectra from a known-composition sample.

CLI:
  python collect_reference.py --port /dev/ttyUSB0 --material "Cotton_100" --n 20
                              --output ./reference_spectra/Cotton_100/

Behaviour:
  1. Open serial port, wait for FiberSense JSON.
  2. Prompt: "Press trigger button on device (N remaining)."
  3. Receive JSON, extract spectrum_calibrated + channels_nm.
  4. Live matplotlib plot updating with each scan.
  5. After n scans: show mean spectrum + per-channel std deviation.
  6. Flag outliers (any channel >2σ from mean) → prompt to retake or accept.
  7. Save accepted scans as CSVs in --output directory.
  8. Print: "Added N scans to {material}. Run python server/train.py to update classifier."

Requirements: pyserial, matplotlib, numpy, pandas
```

---

## PROMPT 5 — Real-time spectrum visualiser (PC Python)

```
Generate tools/visualizer.py

Live spectrum viewer reading USB serial JSON from FiberSense.

CLI: python visualizer.py --port /dev/ttyUSB0 [--reference ./reference_spectra/]

Layout (matplotlib, live):
  Left: spectrum plot.
    x = wavelength nm. y = calibrated reflectance 0–1.
    Current scan: solid line. Previous 5: faded (decreasing alpha).
    If --reference: overlay dashed lines for each reference class mean.

  Right: last classification result received from device.
    Material name. Confidence bar. Top 3 as horizontal bars.

  Bottom: rolling 50-scan history of confidence values.

Keys: s=save PNG, c=clear history, q=quit.
Auto-reconnect on serial disconnect.
Save session log CSV on exit.

Requirements: pyserial, matplotlib, numpy, pandas
```

---

## PROMPT 6 — ADS1256 standalone driver (testable without FiberSense)

```
Generate a standalone Arduino library for the ADS1256 on RP2040.
lib/ADS1256/ADS1256.h and lib/ADS1256/ADS1256.cpp

Clean, reusable driver. No FiberSense dependencies.

API:
  ADS1256(uint8_t cs_pin, uint8_t drdy_pin, SPIClass& spi)
  bool begin(uint8_t data_rate, uint8_t gain)
  int32_t readChannel(uint8_t ch)
  int32_t readDifferential(uint8_t pos, uint8_t neg)
  float toVoltage(int32_t raw, float vref=2.5)
  bool selfCalibrate()
  bool waitDRDY(uint32_t timeout_ms=100)

Constants: ADS1256_30SPS ... ADS1256_30000SPS, ADS1256_GAIN_1 ... ADS1256_GAIN_64

Implementation:
  Full register map (STATUS, MUX, ADCON, DRATE, IO).
  Correct SPI command sequence per datasheet (WREG, RREG, SYNC, WAKEUP, RDATA).
  CS timing: tCSS=100ns before first SCLK, tCSH=100ns after last.
  DRDY must be LOW before RDATA.
  SPI mode 1, MSB first, max 1.92MHz.

Test sketch: examples/ads1256_test/ads1256_test.ino
  Init 1000 SPS, gain 1. Print ch0 voltage at 10Hz. Print all 8 channels at 1Hz.
```

---

## CALIBRATION CONTEXT (add to any prompt touching calibration)

```
Two-point reflectance calibration for LED-sequential NIR:

White reference: PTFE diffuse reflector or 4 layers white copier paper, pressed against sensor hat.
Dark reference: sensor hat blocked completely.

For each channel i:
  R_calibrated[i] = (raw[i] - dark[i]) / (white[i] - dark[i])

Result: 0.0 = no reflectance, 1.0 = white reference level.
Values slightly above 1.0 are valid (some fabrics reflect more than copier paper at NIR).
Clamp to 0.0–1.2, not 0.0–1.0.

InGaAs dark current approximately doubles every 8°C.
If device temperature changes >5°C since calibration: warn user, recommend recalibration.
Temperature via RP2040 internal temperature sensor (accurate to ±2°C, sufficient for this purpose).

Recalibration takes ~10 seconds. Encourage users to recalibrate at start of each session.
```

---

*FiberSense AI Agent Prompt Set — CC BY-SA 4.0*
*Use prompts independently or in sequence. Each generates one complete working component.*
