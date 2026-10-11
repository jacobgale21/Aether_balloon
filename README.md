# Balloon Flight System

Embedded flight software for a high-altitude balloon payload, designed to collect sensor and GPS measurements, transmit flight telemetry to a ground station, and preserve onboard flight data.

> **Project status:** In development. This README describes the intended architecture and requirements; individual features should not be considered implemented or flight-qualified until tested and documented.

## Project Overview

The Balloon Flight System is being developed around a **LilyGO ESP32-based mainboard**. Its primary objective is reliable, autonomous data acquisition during balloon flight, including periods when no Wi-Fi or internet connection is available.

The firmware is intended to:

- Start automatically when power is applied, without requiring field configuration or a network connection.
- Acquire altitude/environmental, motion, and GPS measurements.
- Transmit location and sensor telemetry to a ground station over LoRa.
- Record timestamped flight data to a MicroSD card for later analysis.
- Recover gracefully from intermittent sensor, GPS, radio, or storage failures.

## System Architecture

```text
                      +-----------------------+
                      |   Power / Battery     |
                      +-----------+-----------+
                                  |
                      +-----------v-----------+
                      |  LilyGO ESP32 Board   |
                      |   Flight Controller   |
                      +-----------+-----------+
                                  |
             +--------------------+--------------------+
             |                    |                    |
    +--------v--------+  +--------v--------+  +--------v--------+
    | Flight Sensors  |  |  GPS Receiver   |  | MicroSD Storage |
    | Altitude/Motion |  | Position / Time |  |  Flight Logs    |
    +-----------------+  +-----------------+  +-----------------+
                                  |
                         +--------v--------+
                         |   LoRa Radio    |
                         +--------+--------+
                                  |
                         Wireless Telemetry
                                  |
                         +--------v--------+
                         |  Ground Station |
                         +-----------------+
```

_Conceptual diagram: actual sensor interfaces, onboard peripherals, wiring, and power distribution depend on the finalized hardware configuration._

## Hardware Components

| Component                       | Purpose                                                             | Firmware responsibility                                                        |
| ------------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| LilyGO ESP32 mainboard          | Central flight controller                                           | Boot, task scheduling, peripheral initialization, error handling               |
| Altitude / environmental sensor | Measure flight conditions and estimate altitude                     | Read, validate, timestamp, and store measurements                              |
| Motion sensor(s)                | Measure acceleration, orientation, or angular velocity as supported | Configure sampling and collect motion data                                     |
| GPS receiver                    | Obtain position, time, and potentially speed/altitude               | Receive and parse navigation messages; track fix validity                      |
| LoRa radio                      | Long-range telemetry link                                           | Package and transmit updates; monitor communication health                     |
| MicroSD card                    | Local nonvolatile flight-data storage                               | Write logs, handle storage errors, and recover after resets                    |
| Power subsystem                 | Supply payload electronics                                          | Support reliable startup and power-loss-aware operation where hardware permits |

**Hardware note:** Exact board variant, sensor models, electrical interfaces, pin assignments, and supported features must be verified against the final hardware bill of materials and wiring diagram before implementation.

## Firmware Requirements

### 1. Autonomous Startup and Reliability

- Boot directly into the flight application when power is applied.
- Operate without Wi-Fi, internet access, or user interaction.
- Initialize each peripheral independently and report initialization failures.
- Avoid blocking the entire system when one sensor or subsystem fails.
- Use watchdog supervision and bounded timeouts where appropriate.
- Record reset reasons and important faults when storage is available.

### 2. Sensor Acquisition

- Initialize altitude/environmental and motion sensors using their documented interfaces.
- Collect readings at configurable sampling rates.
- Associate readings with timestamps and measurement units.
- Detect invalid, missing, or out-of-range samples.
- Maintain consistent data structures for logging and telemetry.

### 3. GPS Tracking

- Read GPS data through the appropriate hardware interface, commonly UART.
- Parse supported GPS messages, such as NMEA sentences if provided by the module.
- Track latitude, longitude, altitude, speed, time, and fix quality when available.
- Distinguish a valid position fix from stale or unavailable GPS data.
- Continue sensor logging during GPS outages.

### 4. LoRa Ground Communications

- Configure the onboard radio for the correct hardware and permitted regional radio settings.
- Transmit compact, versioned telemetry packets containing essential flight data.
- Include packet sequence numbers and error-detection mechanisms.
- Support loss of ground-station connectivity without interrupting data collection.
- Balance telemetry update rate, payload size, airtime, and power consumption.

### 5. MicroSD Data Logging

- Store timestamped sensor and GPS data locally.
- Use a documented log format, such as CSV or a compact binary record format.
- Buffer and periodically flush data while managing the tradeoff between performance and power-loss exposure.
- Handle missing cards, write failures, and unexpectedly full storage.
- Resume logging after recoverable failures or resets without overwriting prior flight records.
- Validate logs during ground testing and post-flight analysis.

## Proposed Software Stack

| Layer                | Candidate technology                                              | Role                                               |
| -------------------- | ----------------------------------------------------------------- | -------------------------------------------------- |
| Programming language | C++ with C-compatible drivers where appropriate                   | Embedded application and peripheral control        |
| Firmware framework   | ESP-IDF or Arduino-ESP32                                          | ESP32 startup, drivers, and runtime services       |
| Scheduling           | FreeRTOS (provided by ESP-IDF)                                    | Periodic acquisition, logging, and telemetry tasks |
| Sensor interfaces    | I2C, SPI, UART, as required                                       | Communication with attached hardware               |
| GPS parsing          | Lightweight NMEA parser or module-specific binary protocol parser | Decode navigation data                             |
| Radio driver         | Driver compatible with the exact LilyGO LoRa transceiver          | Configure and operate LoRa                         |
| Storage              | ESP-IDF or framework-supported SD/FAT filesystem                  | Write and manage flight logs                       |
| Build and debugging  | ESP-IDF tooling or PlatformIO                                     | Compile, flash, monitor, and test firmware         |

**These are technology candidates, not a claim that dependencies or drivers have already been selected.** Choose libraries only after confirming board and component compatibility.

## Proposed Repository Structure

```text
Balloon-Flight-System/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── hardware.md
│   └── telemetry-protocol.md
├── firmware/
│   ├── src/
│   │   ├── main.cpp
│   │   ├── sensors/
│   │   ├── gps/
│   │   ├── telemetry/
│   │   ├── storage/
│   │   └── system/
│   ├── include/
│   └── test/
├── ground-station/
└── tools/
```

_Suggested future organization; folders and files may not yet exist._

## Flight Data Flow

1. **Boot:** The ESP32 starts, loads its configuration, and initializes connected devices.
2. **Acquire:** Sensor and GPS interfaces collect measurements independently.
3. **Validate:** The firmware timestamps readings and marks invalid or missing data.
4. **Log:** Records are queued for MicroSD storage so data collection does not depend on radio connectivity.
5. **Transmit:** A compact subset of the latest measurements is sent over LoRa.
6. **Recover:** The application handles temporary faults, preserves diagnostics, and continues operating where safe.

## Development and Testing Plan

- **Phase 1 — Board bring-up:** Verify automatic boot, power cycling, serial diagnostics, and watchdog behavior.
- **Phase 2 — Sensor integration:** Verify sensor initialization, calibration, units, and sustained sampling.
- **Phase 3 — GPS integration:** Test valid fixes, loss of signal, malformed messages, and stale data.
- **Phase 4 — LoRa telemetry:** Verify packet decoding, reception range, dropped packets, and transmission intervals.
- **Phase 5 — MicroSD logging:** Test file integrity, card removal, storage exhaustion, resets, and unexpected power loss.
- **Phase 6 — End-to-end validation:** Run long-duration, battery-powered tests under representative environmental conditions before flight.

## Build and Deployment

The final build and flashing procedure will be documented once the **exact LilyGO board variant**, firmware framework, and pin configuration are confirmed.

The intended field workflow is simple: flash and verify the firmware during development, connect power at launch, and allow the payload to boot and operate autonomously. A computer or Wi-Fi connection should not be required during flight.

## Operational Considerations

- Verify battery performance and component temperature limits for the expected flight environment.
- Confirm GPS receiver operating limits are suitable for the planned altitude and dynamics.
- Configure LoRa frequency, output power, and duty cycle according to applicable regulations.
- Ensure the antenna, power supply, and SD card remain reliable under vibration and cold conditions.
- Treat LoRa telemetry as a live convenience, not the sole copy of flight data.
- Conduct a full preflight test and document known limitations before launch.

## Project Status

| Subsystem                  | Status                                               |
| -------------------------- | ---------------------------------------------------- |
| Hardware selection         | Assumed finalized; configuration details to document |
| Autonomous startup         | Planned                                              |
| Sensor acquisition         | Planned                                              |
| GPS tracking               | Planned                                              |
| LoRa telemetry             | Planned                                              |
| MicroSD logging            | Planned                                              |
| Ground-station integration | Planned                                              |
| Flight validation          | Not yet documented                                   |
