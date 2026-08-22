# Reading Temperature and Humidity Using a DHT11 Sensor

A simple Data Acquisition (DAQ) project built with an Arduino Uno and a DHT11 sensor.
Reads ambient temperature (°C) and relative humidity (%) every 2 seconds and logs
the data as CSV output via USB serial to a connected computer.



---

## Overview

This project demonstrates the core principles of a DAQ pipeline:

```
Physical Environment → DHT11 Sensor → Arduino Uno → USB Serial → PC (Serial Monitor / CSV Logger)
```

The Arduino reads temperature and relative humidity every **2 seconds** using a
non-blocking `millis()` timing approach, then prints each reading as a CSV row to the
Serial Monitor at **9600 baud**. An optional Python script logs the data to a `.csv`
file for further analysis.

---

## Hardware Requirements

| Component | Quantity | Notes |
|---|---|---|
| Arduino Uno R3 | 1 | Budget clones work — check digital pins |
| DHT11 sensor (3-pin module) | 1 | 3-pin module has built-in pull-up resistor |
| Breadboard (400-tie) | 1 | For prototyping |
| Jumper wires (M–M) | 3 | Red, black, blue |
| USB-A to USB-B cable | 1 | Must be a **data** cable, not charge-only |

---

## Software Requirements

- **[Arduino IDE 2.x](https://www.arduino.cc/en/software)** — free download
- **DHT sensor library by Adafruit** — install via Arduino Library Manager  
  *(Tools → Manage Libraries → search "DHT sensor library")*
- **Adafruit Unified Sensor** — required dependency, also in Library Manager
- **Python 3** *(optional, for CSV logging)*

---

## Circuit Wiring

The 3-pin DHT11 module has a built-in pull-up resistor — **no external resistor needed**.

| DHT11 Module Pin | Arduino Uno Pin |
|---|---|
| VCC | 5V |
| GND | GND |
| S (DATA) | Digital Pin 3 |

> **Note:** Digital Pin 2 was found faulty on the board used in this project (common
> on budget clones). Pin 3 is used instead — all digital I/O pins are electrically
> equivalent for this application.

**Breadboard layout:**

```
Arduino 5V  ──── DHT11 VCC
Arduino GND ──── DHT11 GND
Arduino D3  ──── DHT11 S (DATA)
```

---

## DAQ Signal Flow

```
┌──────────────┐    1-Wire signal    ┌──────────────┐    Serial/CSV    ┌─────────────┐    USB    ┌──────────┐
│  DHT11       │ ─────────────────►  │  Arduino Uno │ ───────────────► │  USB Cable  │ ────────► │ Computer │
│  Sensor      │                     │  (ATmega328P)│                  │  9600 baud  │           │ Serial   │
│  Temp + Hum  │                     │  Processes   │                  │  data xfer  │           │ Monitor  │
└──────────────┘                     └──────────────┘                  └─────────────┘           └──────────┘
```

---

## Arduino Sketch

```cpp
// DHT11 Environmental DAQ System
// Author: Governor-Yorhokpor Isaac | PS/EPH/22/0010
// Department of Physics, UCC | ENP416 Final Assessment

#include "DHT.h"

#define DHTPIN   3       // Digital pin connected to S (DATA)
#define DHTTYPE  DHT11   // Sensor model

DHT dht(DHTPIN, DHTTYPE);

unsigned long lastReadTime = 0;
const unsigned long INTERVAL = 2000; // 2-second sampling interval

void setup() {
  Serial.begin(9600);
  dht.begin();
  Serial.println("DHT initialised - starting readings...");
  Serial.println("timestamp_ms,temperature_C,humidity_pct"); // CSV header
}

void loop() {
  unsigned long now = millis();

  if (now - lastReadTime >= INTERVAL) {
    lastReadTime = now;

    float humidity    = dht.readHumidity();
    float temperature = dht.readTemperature(); // Celsius

    if (isnan(humidity) || isnan(temperature)) {
      Serial.println("ERROR: Sensor read failed. Check wiring.");
      return;
    }

    Serial.print(now);
    Serial.print(",");
    Serial.print(temperature, 1); // 1 decimal place
    Serial.print(",");
    Serial.println(humidity, 1);

  } // ← closing brace of the interval if-block
}   // ← closing brace of loop()
```

> **Why `millis()` and not `delay()`?**  
> `delay()` freezes the entire processor for its duration — no other code runs.
> `millis()` is non-blocking: it returns the time elapsed since power-on, so the
> loop continues running freely and only triggers a reading when the interval has passed.

---

### Optional — Python Serial Logger

Logs readings to `sensor_log.csv` in real time:

```python
import serial
import time
import csv

port = "COM3"   # Change to your port (e.g. /dev/ttyUSB0 on Linux/Mac)

ser = serial.Serial(port, 9600, timeout=1)
time.sleep(2)   # Wait for Arduino reset

with open("sensor_log.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["timestamp_ms", "temperature_C", "humidity_pct"])
    print("Logging... press Ctrl+C to stop.")
    try:
        while True:
            line = ser.readline().decode("utf-8").strip()
            if line and not line.startswith("time"):
                writer.writerow(line.split(","))
                print(line)
    except KeyboardInterrupt:
        print("Logging stopped.")
```

### Optional — Plot the logged data

```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv("sensor_log.csv")
df["time_s"] = df["timestamp_ms"] / 1000

fig, ax1 = plt.subplots()
ax2 = ax1.twinx()
ax1.plot(df["time_s"], df["temperature_C"], "r-", label="Temp (°C)")
ax2.plot(df["time_s"], df["humidity_pct"],  "b-", label="Humidity (%)")
ax1.set_xlabel("Time (s)")
ax1.set_ylabel("Temperature (°C)", color="red")
ax2.set_ylabel("Humidity (%)", color="blue")
plt.title("DHT11 Environmental DAQ — Live Readings")
plt.tight_layout()
plt.savefig("sensor_plot.png", dpi=150)
plt.show()
```

---

## Sample Output

Serial Monitor output (9600 baud):

```
DHT initialised - starting readings...
timestamp_ms,temperature_C,humidity_pct
2000,29.4,72.0
4000,29.4,72.0
6000,29.5,72.0
8000,29.5,71.9
10000,29.5,71.9
12000,29.5,71.9
```

---

## Sensor Specifications

| Parameter | Value |
|---|---|
| Temperature range | 0 – 50 °C |
| Temperature accuracy | ±2 °C |
| Temperature resolution | 1 °C |
| Humidity range | 20 – 80% RH |
| Humidity accuracy | ±5% RH |
| Humidity resolution | 1% RH |
| Max sampling rate | 1 Hz (one reading per second) |
| Communication protocol | 1-Wire single-bus serial |
| Supply voltage | 3.3 – 5.5 V |

---

## Precision & Accuracy Analysis

### Precision — Standard Deviation

Using 6 repeated readings under stable conditions:

| Reading | Temp (°C) | Humidity (%) |
|---|---|---|
| 1 | 29.4 | 72.0 |
| 2 | 29.4 | 72.0 |
| 3 | 29.5 | 72.0 |
| 4 | 29.5 | 71.9 |
| 5 | 29.5 | 71.9 |
| 6 | 29.5 | 71.9 |

```
σ = √[ Σ(xᵢ − x̄)² / (n − 1) ]

Temperature:  x̄ = 29.47°C,  σ = 0.05°C   → High precision
Humidity:     x̄ = 71.95%,   σ = 0.05%    → High precision
```

### Accuracy — Comparison with Reference Instrument

| Parameter | DHT11 Mean | Reference | Absolute Error | % Error | Within Spec? |
|---|---|---|---|---|---|
| Temperature | 29.47°C | 30.0°C | 0.53°C | 1.78% |  Yes (±2°C) |
| Humidity | 71.95% | 74.0% | 2.05% | 2.77% |  Yes (±5% RH) |

**Conclusion:** The DHT11 is highly **precise** (σ ≈ 0.05) and acceptably **accurate**
(errors within manufacturer spec), making it suitable for educational DAQ and qualitative
environmental monitoring.

---

## Known Issues & Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `avrdude: not in sync` | Wrong/missing CH340 driver | Install CH340 driver from wch-ic.com |
| `Sensor read failed` | Loose DATA wire | Re-seat S pin wire firmly |
| `Sensor read failed` | External pull-up + module already has one | Remove external 10kΩ resistor |
| Temperature = 0.0 | Wrong `#define DHTPIN` | Update pin number to match actual wiring |
| No serial output | Wrong baud rate | Set Serial Monitor to 9600 |
| Port not found | Charge-only USB cable | Use a data-capable USB cable |
| Port shows `COM3` no label | Missing CH340 driver | See driver fix above |

---

## Limitations & Future Improvements

**Current limitations:**
- Temperature accuracy of ±2°C is insufficient for scientific-grade measurements
- Integer-only output (1°C, 1% RH resolution) limits data granularity
- Maximum 1 Hz sampling rate — only suitable for slowly changing environments

---

