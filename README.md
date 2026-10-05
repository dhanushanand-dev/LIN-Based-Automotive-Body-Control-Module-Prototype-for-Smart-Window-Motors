# LIN-Based Body Control Module for Automotive Window Motors

**An STM32 firmware prototype that controls real automotive power-window motor ECUs over the LIN bus. I reverse-engineered the LIN protocol on the bench because no documentation was available.**

![MCU](https://img.shields.io/badge/MCU-STM32F446RE-03234B)
![Language](https://img.shields.io/badge/language-C-blue)
![Bus](https://img.shields.io/badge/bus-LIN%202.x%20%40%2019200%20baud-orange)
![Framework](https://img.shields.io/badge/framework-STM32%20HAL-lightgrey)

---

## Working Demonstration

Physical validation of the reverse-engineered LIN control system using automotive smart window motor ECUs.

### Demo 1

[▶ Watch LIN Window Motor Demo 1](media/demo-video/lin-window-motor-demo-1.mp4)

### Demo 2

[▶ Watch LIN Window Motor Demo 2](media/demo-video/lin-window-motor-demo-2.mp4)

---

## What this project does

In a car, a **Body Control Module (BCM)** is the master on a **LIN bus**. It tells each door's window-motor ECU when to move. This project builds that BCM on an **STM32 NUCLEO-F446RE**:

1. I started with production window-motor ECUs and **no protocol documentation**.
2. I wrote a series of test firmwares to **reverse-engineer the protocol**. They find the baud rate, the frame IDs, the status frames, and the command bytes that move each motor.
3. I used those results to build a **generic BCM controller**. It wakes the bus, **auto-detects which door ECU is connected** (FL / FR / RL / RR), and drives the motor UP or DOWN from push-buttons or PLC outputs, with safety interlocks.

## Key features

- **Bit-level LIN master:** break, sync (`0x55`), protected ID with parity bits, and data. It supports both **classic and enhanced checksums**. Everything is written on top of the STM32 USART in LIN mode.
- **ECU auto-detection:** it sends a LIN diagnostic *Read-By-Identifier* request (`0x3C`/`0x3D`) and identifies which door is connected from the node address in the reply.
- **Profile table:** one `MotorProfile` per door holds the status ID, command ID, baseline status, and UP/DOWN payloads, so adding a new ECU only means adding data.
- **Hold-to-run control:** the motor runs only while the UP/DOWN input is held. When the input is released, the firmware immediately sends a *neutral* (stop) burst.
- **Safety:**
  - Inputs are debounced.
  - Pressing two inputs at once forces an **immediate stop**.
  - Reversing direction always sends neutral first.
  - Commands that haven't been verified (RR UP) are **disabled in firmware**.
- **Fault handling:**
  - A 1 s status heartbeat checks that the ECU is still answering.
  - If communication is lost while the motor is moving, it stops at once.
  - Wake-up is retried automatically up to 5 times.
  - After a power cycle the controller recovers and returns to the sleep state.
- **LIN power management:** wake-up pulse, and a go-to-sleep command on `0x3C`.
- **Status LEDs:** the LIN LED blinks while the bus is asleep and stays solid while it's awake. Separate LEDs show UP and DOWN movement.
- **Debug console:** every frame is logged to the ST-Link virtual COM port at 115200 baud.

## Hardware

| Part | Role |
|---|---|
| STM32 **NUCLEO-F446RE** (Cortex-M4, 180 MHz) | LIN master / BCM |
| LIN transceiver module (TTL ↔ LIN) | Converts USART1 TX/RX to the single-wire LIN bus |
| Automotive window-motor ECU(s) + motor | LIN slave being controlled |
| 12 V bench supply | Powers the ECU and the LIN bus |
| 3 push-buttons **or** PLC digital outputs | LIN wake/sleep, UP, DOWN |
| 3 LEDs | LIN status, UP active, DOWN active |

The pin-out and wiring are in [docs/build_and_flash.md](docs/build_and_flash.md#wiring).

## How it works

```
 Buttons / PLC ──► STM32 NUCLEO-F446RE ──USART1 (LIN mode)──► LIN transceiver ──LIN bus──► Door ECU ──► Window motor
                        │
                        └── USART2 ──► ST-Link virtual COM port (debug log, 115200 baud)
```

1. **Power on.** The firmware waits 1.5 s for the power rails, then enters **SLEEP**. The LIN LED blinks.
2. **LIN input pressed.** It sends a wake-up pulse and a diagnostic request, reads the reply, and looks the reply up in the door profile table. On a match it is **AWAKE_READY** and the LIN LED is solid.
3. **UP or DOWN held.** It sends the door's manual UP/DOWN payload on the command frame (`0x15`) every 50 ms.
4. **Input released.** It sends a neutral burst (`FF FF … FF` × 5) and the motor stops.
5. **LIN input pressed again.** It sends neutral, then the LIN *go-to-sleep* command, and returns to **SLEEP**.

The state machine and the frame-level detail are in [docs/firmware_architecture.md](docs/firmware_architecture.md).

## Reverse-engineering journey

The firmware was built in stages. Every stage is kept in `firmware/reverse_engineering_stages/` in the private source repository:

| Stage | Goal | Outcome |
|---|---|---|
| 01 Multi-baud scan | Find the bus speed and responding IDs | **19200 baud** confirmed. The diagnostic reply and status ID `0x16` were found ([log](results/lin_multi_baud_discovery_results.txt)) |
| 02 Status polling | Watch the status frame live | Baseline status frame captured |
| 03 Command discovery | Find which frame moves the motor | Command frame ID found |
| 04 Frame classification | Classify responses, stop and sleep behaviour | Neutral/stop and sleep handling understood |
| 05 All-motor payloads | Find the UP/DOWN byte for each door | Payload map for FL, FR, RL, RR |
| 06 RR investigation | Confirm the rear-right UP command | Still unconfirmed, so it is **disabled in firmware** |

The full write-up is in [docs/lin_reverse_engineering.md](docs/lin_reverse_engineering.md).

## Source code

The firmware source code is kept in a **private repository**. It is available to recruiters and interviewers on request: contact me via [GitHub @dhanushanand-dev](https://github.com/dhanushanand-dev).

This public repository is a showcase. It contains the documentation, demo videos and test results:

```
├── README.md
├── docs/
│   ├── lin_reverse_engineering.md   # How the protocol was discovered, stage by stage
│   ├── firmware_architecture.md     # State machine, LIN driver, safety logic
│   ├── build_and_flash.md           # Wiring, build, flash, serial console
│   └── source_map.md                # What every firmware file in the private repo is
├── results/
│   └── lin_multi_baud_discovery_results.txt   # Real console log from stage 01
└── media/demo-video/                # Working demonstration videos
```

The private source repository is laid out like this:

```
automotive-lin-window-bcm/
├── README.md
├── docs/
│   ├── lin_reverse_engineering.md   # How the protocol was discovered, stage by stage
│   ├── firmware_architecture.md     # State machine, LIN driver, safety logic
│   ├── build_and_flash.md           # Wiring, build (CubeIDE / CMake), flash, serial console
│   └── source_map.md                # What every firmware file is
├── firmware/
│   ├── stm32_nucleo_f446re_project/ # ★ Buildable STM32 project (latest firmware: Core/Src/main.c)
│   ├── reverse_engineering_stages/  # Stage 01–06 discovery firmwares
│   ├── validated_control/           # First working single-door (FL) controller
│   └── generic_bcm/                 # Generic multi-door BCM iterations 01–04
├── results/
│   └── lin_multi_baud_discovery_results.txt   # Real console log from stage 01
└── media/demo-video/                # Working demonstration videos
```

The firmware builds with STM32CubeIDE or with CMake and the Arm GNU toolchain. The steps are in [docs/build_and_flash.md](docs/build_and_flash.md).

## Skills demonstrated

- Embedded C on ARM Cortex-M4 (STM32 HAL, register-level USART/LIN control)
- LIN protocol: frame format, PID parity, classic/enhanced checksums, diagnostic frames, wake-up and sleep
- **Protocol reverse engineering** of undocumented automotive ECUs
- Event-driven state machines, input debouncing, and fail-safe design
- Automotive BCM concepts: hold-to-run, interlocks, heartbeat and loss-of-communication handling
- Bench bring-up and hardware debugging with a serial trace
- Build systems: STM32CubeIDE and CMake cross-compilation

## Status and limitations

- FL, FR and RL: UP and DOWN are mapped. RR: only DOWN is enabled, because the UP payload hasn't been physically confirmed (`ENABLE_RR_UP_COMMANDS = 0`).
- *Auto/express* payloads were found for FL, but they are not used in the hold-to-run (PLC) build.
- This is a bench prototype and is not intended for in-vehicle use.

## Author

**Dhanush Anand**: embedded systems / automotive electronics · [GitHub @dhanushanand-dev](https://github.com/dhanushanand-dev)
<!-- Add your LinkedIn, email, or portfolio link here -->
