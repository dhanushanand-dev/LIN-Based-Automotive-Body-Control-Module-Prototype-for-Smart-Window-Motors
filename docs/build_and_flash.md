# Build, Flash and Run

> **Note:** The source code for this project is kept in a private repository. File paths below refer to that repository. Access is [available on request](https://github.com/dhanushanand-dev).

## Wiring

```
NUCLEO-F446RE                 LIN transceiver module            Window-motor ECU
 PA9  (D8, USART1_TX) ──────► RX / TXD
 PA10 (D2, USART1_RX) ◄────── TX / RXD
 GND  ─────────────────────── GND ───────────────────────────── GND  (ECU pin 4)
                              SLP / EN ── HIGH (3.3 V)
                              VBAT ◄──── +12 V ──────────────── +12 V (ECU pin 3)
                              LIN  ──────────────────────────── LIN   (ECU pin 5)

 PC7 (D9)  ── LIN button  ── GND      (inputs use internal pull-ups; pressed = LOW)
 PB6 (D10) ── UP button   ── GND
 PA7 (D11) ── DOWN button ── GND
 PA6 (D12) ── LED (LIN status)  ── resistor ── GND
 PB9 (D14) ── LED (UP active)   ── resistor ── GND
 PB8 (D15) ── LED (DOWN active) ── resistor ── GND
```

- The ECU pin numbers (3 = +12 V, 4 = GND, 5 = LIN) are from the bench setup recorded in the stage-01 log. Check them against your own ECU's connector.
- **All grounds must be common:** the STM32, the transceiver and the ECU supply.
- For **PLC control**, wire the PLC outputs so that they pull the input pins LOW, for example with an opto-isolator or a relay contact to GND. Never apply 24 V directly to the STM32 pins.

## Option A: STM32CubeIDE

1. Install [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html).
2. Choose *File → Import → General → Existing Projects into Workspace*, then select `firmware/stm32_nucleo_f446re_project`.
3. Click **Build** (hammer icon). The output is `Debug/code_files.elf`.
4. Connect the Nucleo over USB and click **Run** (or **Debug**). The ST-Link programs the board.

> The project's internal name, `code_files`, is kept on purpose. Renaming it would break the CubeMX and IDE references.

## Option B: CMake + Arm GNU toolchain

Requirements: `cmake` ≥ 3.22, `ninja`, and `arm-none-eabi-gcc` on your PATH.

```bash
cd firmware/stm32_nucleo_f446re_project
cmake --preset Debug
cmake --build build/Debug
```

To flash, use **STM32CubeProgrammer**:

```bash
STM32_Programmer_CLI -c port=SWD -w build/Debug/code_files.elf -v -rst
```

You can also drag a `.bin` file onto the `NOD_F446RE` USB drive that appears when the Nucleo is plugged in.

## Serial console

Open the ST-Link virtual COM port (Device Manager → *Ports*) at **115200 baud, 8N1**. You can use PuTTY, Tera Term, or the CubeIDE terminal.

A normal session looks like this:

```
=== GENERIC LIN BCM CONTROLLER v4 - PLC HOLD-TO-RUN ===
[BOOT] LIN initialized at 19200 baud.
[STATE_SLEEP] Press LIN_BUTTON once to wake and auto-detect connected ECU.
[ECU AUTO-DETECT] Sending diagnostic request ID 0x3C...
[OK] Auto-detected connected ECU side: FL
[STATE] ECU_AWAKE_READY
[PLC RUN START] Command: FL_MANUAL_UP
[PLC INPUT RELEASED] Sending NEUTRAL and stopping motor.
```

## Running a reverse-engineering stage

The files in `firmware/reverse_engineering_stages/`, `validated_control/` and `generic_bcm/` are complete `main.c` snapshots. To run one:

1. Back up `Core/Src/main.c` in the STM32 project.
2. Copy the stage file over `Core/Src/main.c`.
3. Build, flash, and follow the instructions it prints on the serial console. Several stages wait for you to type `READY`.

## Troubleshooting

| Symptom | Check |
|---|---|
| `Diagnostic confirmation failed` | Is the ECU getting 12 V? Is the transceiver's SLP/EN pin HIGH? Is GND common? Are TX and RX swapped? |
| `NAD/profile is not in table` | The ECU answered but isn't one of the four known doors. Add its reply to `MOTOR_PROFILES[]`. |
| `[PLC COMMAND BLOCKED]` | That direction is disabled in firmware. RR UP is unconfirmed. |
| `HEARTBEAT WARNING` | Loose LIN wire, or ECU power was lost. |
