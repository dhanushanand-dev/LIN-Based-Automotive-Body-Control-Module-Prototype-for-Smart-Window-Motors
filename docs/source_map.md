# Source Map

> **Note:** The source code for this project is kept in a private repository. File paths below refer to that repository. Access is [available on request](https://github.com/dhanushanand-dev).

Every firmware file in the private source repository is listed here. The `.c` snapshots are complete `main.c` files from different points in development. To run one, copy it over `firmware/stm32_nucleo_f446re_project/Core/Src/main.c` (see [build_and_flash.md](build_and_flash.md#running-a-reverse-engineering-stage)).

## ★ Latest firmware (buildable project)

`firmware/stm32_nucleo_f446re_project/`
- `Core/Src/main.c`: **Generic LIN BCM controller v4 with PLC hold-to-run.** This is what the project builds and flashes. It is described in [firmware_architecture.md](firmware_architecture.md).
- `Core/Src/gpio.c`, `usart.c`, `stm32f4xx_*.c`, `Core/Inc/*.h`: STM32CubeMX-generated peripheral setup.
- `Drivers/`: ST HAL and CMSIS libraries, vendored as CubeMX generates them.
- `code_files.ioc`: STM32CubeMX configuration (pins, clocks, USARTs).
- `.cproject`, `.project`: STM32CubeIDE project files.
- `CMakeLists.txt`, `CMakePresets.json`, `cmake/`: CMake build.
- `STM32F446RETX_FLASH.ld` / `STM32F446RETX_RAM.ld`: linker scripts used by CubeIDE.
- `STM32F446XX_FLASH.ld`: linker script used by CMake.
- `Core/Startup/startup_stm32f446retx.s`: startup file used by CubeIDE.
- `startup_stm32f446xx.s`: startup file used by CMake. It has the same content, but each build system expects its own path.

## Reverse-engineering stages

`firmware/reverse_engineering_stages/` (explained in [lin_reverse_engineering.md](lin_reverse_engineering.md))

| File | Stage |
|---|---|
| `00_relay_power_wakeup_bruteforce.c` | Relay-switched ECU power-up, guessed-ID probe and first ID scan `0x00`–`0x3F` |
| `01_multi_baud_bruteforce.c` | Multi-baud reachability scan and full ID search `0x00`–`0x3F` |
| `02_focused_status_polling.c` | Live status-frame polling and baseline capture |
| `03_command_frame_discovery.c` | Controlled payload patterns to find the command frame |
| `04_frame_classification.c` | Stop detection, sleep-wait and response classification |
| `05_all_motor_payload_discovery.c` | Fast byte-0 command discovery across all door motors |
| `06_rr_cw_investigation.c` | Rear-right UP (CW) payload investigation |

## Validated control

`firmware/validated_control/front_left_motor_validated_control.c`: the first fully working controller, for the front-left door only. A short press gives a manual jerk, and holding for 5 s or more gives auto/express. It uses a serial `READY` handshake.

## Generic BCM iterations

`firmware/generic_bcm/`

| File | What changed |
|---|---|
| `01_generic_bcm_basic_control.c` | Table-driven profiles for FL/FR/RL/RR, auto-detection by diagnostic reply |
| `02_generic_bcm_wake_and_recovery.c` | Wake retries, ECU-lost detection, recovery after a power cycle |
| `03_generic_bcm_led_status_control.c` | LIN / UP / DOWN status LEDs |
| `04_generic_bcm_profile_table.c` | Final snapshot of the door profile table. Its logic is the same as 03. |

The latest step after 04 is the PLC hold-to-run version in the buildable project (see above).

## Results

- `results/lin_multi_baud_discovery_results.txt`: real serial log from stage 01. It shows no reply at 9600 or 10400 baud, a reply at 19200 baud, and status ID `0x16` found.

## Cleanup notes

This repository is a cleaned copy of the original development workspace. These files were removed:

- STM32CubeMX `Backup/` folders, which were `.bak` copies of the generated sources.
- A second copy of the generic BCM 04 snapshot, which was byte-for-byte identical.
- `front_left_motor_reference_control.c`, which was byte-for-byte identical to the validated control file.

Build outputs, IDE workspace metadata, archives and logs are excluded by `.gitignore`.
