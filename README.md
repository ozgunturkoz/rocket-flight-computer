# Rocket Flight Computer: Avionics Software Architecture 🚀

Flight computer software of **Astra Rocket Team** for the **TEKNOFEST Rocket Competition**
(Turkey **5th + Best Design Award** 2024 · **7th** 2025 · **9th** 2026).
I led the team's 3-person software team. This page describes the architecture and the main engineering decisions.

> 🔒 **The firmware source code is team-owned and kept private.** This repository only documents the design.
> I'm happy to walk through any part of it in a technical interview.

---

## At a glance

| | |
|---|---|
| **MCU** | STM32H7 (HAL + register-level drivers where needed) |
| **RTOS** | FreeRTOS (CMSIS-RTOS2): 8 tasks, mutexes, message queues |
| **Sensors** | 2× BMP280 + BMP390 barometers, BNO055 IMU (quaternion + linear accel), GNSS (NMEA) |
| **Recovery** | 8-state flight state machine with two-stage deployment (drogue at apogee, main at low altitude) |
| **Redundancy** | Triple barometer with fault voting, dual parallel pyro channels per event, independent watchdog |
| **Storage** | microSD black-box logging (FAT32 over SPI) + SPI NOR flash |
| **Links** | Binary telemetry to the ground station, plus the competition's hardware-in-the-loop test protocol |
| **Language** | C (firmware), C# ([ground station](https://github.com/ozgunturkoz/astra-ground-station)) |

---

## 1. Task architecture (FreeRTOS)

| Task | Priority | Period | Job |
|---|---|---|---|
| `SensorTask` | Above normal | 20 Hz | Read sensors, run the filter chain, update the state machine, feed the watchdog |
| `TelemTask` | Normal | 5 Hz | Binary telemetry to the ground station |
| `TestTelemTask` | Normal | 10 Hz | Test-device packets (SİT/SUT) |
| `LoggerTask` | Normal | 10 Hz | microSD black-box logging |
| `GPSTask` | Normal | event | NMEA sentence parsing (fed by a UART queue) |
| `USART3Task` | Normal | event | Test-device command and synthetic-data packets |
| `PyroCommandTask` | Normal | 50 Hz | Manual deployment commands (ground tests) |
| `BuzzerTask` | Low | 0.5 Hz | Audible status codes on the pad |

**Design rules**
- The recovery decision runs in the highest-priority task, so logging or telemetry can never delay a deployment.
- **Mutex ordering contract.** Locks are always taken in one fixed global order to prevent deadlocks. Mutexes are
  non-recursive by design, so functions that need a lock document that the caller already holds it. This came
  from a real bug: a re-entrant lock attempt timed out on every call and silently stopped state updates in one test mode.
- **ISR → queue → task.** Interrupt handlers only push raw bytes or lines into message queues; parsing happens in
  tasks. Queue-overflow counters are exposed for debugging.

## 2. Sensor filtering & fusion

```
 BMP280 #1 ─┐
 BMP280 #2 ─┼─ median(5) ─ Kalman ─┬─ voting / weighted fusion ─ baro altitude ─┬─ Kalman ─► altitude
 BMP390 ────┘                       │   (faulty sensor voted out)                └─ d/dt  ─► vertical velocity
 BNO055 quaternion + linear accel ─ rotate to world frame ─ Kalman ─────────────────────► vertical accel
```
- **Two-stage filtering.** The median filter removes single-sample spikes such as I2C glitches or electrical noise.
  The Kalman filter then smooths continuous noise. A Kalman filter alone would absorb a spike slowly and report a
  wrong altitude while doing so.
- **Sensor voting.** If one barometer disagrees with the other two, it is excluded and the exclusion is logged.
- **Hardware-independent filter module.** The filtering layer does no HAL or RTOS calls, only math. Recorded flight
  logs can be replayed through it on a PC, which is the only way to verify the most critical code without a real flight.

## 3. Flight state machine & recovery

```
CALIBRATION → IDLE → BOOST → COAST → APOGEE ──(1st deployment: drogue)──► DROGUE → MAIN ──(2nd deployment: main)──► LANDED
```
- Every transition needs **N consecutive confirming samples**, so one noisy reading can't fire a pyro charge.
- **Lockout timers** after liftoff and after apogee prevent premature transitions, for example during motor burn
  or when the deployment shock disturbs the sensors.
- Apogee detection combines an altitude gate with the velocity sign change. Main deployment is altitude-gated during descent.
- All pyro outputs go through a **single driver module** (one entry point for safety checks). Each event drives
  **two channels in parallel**, so one failed MOSFET or igniter wire can't prevent deployment.

## 4. Reliability

- **Independent watchdog (IWDG)** with a ~4 s timeout, fed only by the sensor task. If an I2C bus hangs or a task
  locks up, the board resets instead of falling ballistic. Implemented with direct register access.
- **Black box.** Raw and filtered data, state transitions and events are logged to microSD for post-flight analysis.
- **Battery monitoring** via ADC (register-level).

## 5. Protocols & hardware-in-the-loop testing

| Packet | Link | Direction | Format |
|---|---|---|---|
| Telemetry | UART (RF) | → ground station | binary, little-endian, checksum (team-defined) |
| SİT sensor data | UART | → test device | fixed binary, big-endian (competition spec) |
| SUT status | UART | → test device | 16-bit status word (competition spec) |
| SUT synthetic data | UART | ← test device | fixed binary, big-endian (competition spec) |

- **SİT (System Transfer Test).** The flight computer streams live sensor data to the competition's test device.
- **SUT (System Compliance Test).** The test device injects **synthetic flight data**, and the flight computer must
  run its recovery algorithm on it and report status bits. This is a hardware-in-the-loop check of the whole
  decision chain.
- Packet building and parsing is separated from UART I/O. Functions fill buffers and return lengths, so formats can be
  unit-tested on a PC and the transport (blocking, interrupt or DMA) doesn't affect the format.

---

## My role
- **2025 – present · Software Team Lead.** Architecture, task design, code reviews and simulation integration for the STM32-based avionics.
- **2024 – 2025 · Lead Software Developer.** Core data-processing algorithms, SPI/I2C/UART peripheral drivers and a redundant ESP-NOW wireless link.
- **2023 – 2024 · Software Developer.** Ground-station UI, test tools and hardware/software integration tests.

Related: [Ground station (2026)](https://github.com/ozgunturkoz/astra-ground-station) · [Ground station (earlier)](https://github.com/ozgunturkoz/astra-arayuz)

**Özgün Türköz** · [LinkedIn](https://linkedin.com/in/ozgunturkozz) · ozgunturkoz3@gmail.com
