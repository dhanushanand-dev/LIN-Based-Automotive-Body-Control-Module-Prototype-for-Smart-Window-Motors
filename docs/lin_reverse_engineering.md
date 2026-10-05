# LIN Protocol Reverse Engineering

> **Note:** The source code for this project is kept in a private repository. File paths below refer to that repository. Access is [available on request](https://github.com/dhanushanand-dev).

The window-motor ECUs used here came with **no LIN description file (LDF) and no documentation**. Everything the final firmware uses was found by experiment, with the STM32 acting as the LIN master. Each stage below is a separate firmware in `firmware/reverse_engineering_stages/`.

## LIN in 60 seconds

A LIN frame is sent by the master and looks like this:

```
| Break (≥13 bits low) | Sync 0x55 | PID | Data 0..8 bytes | Checksum |
```

- **PID** = 6-bit frame ID + 2 parity bits (`P0 = ID0^ID1^ID2^ID4`, `P1 = !(ID1^ID3^ID4^ID5)`).
- The master always sends the header. The data part comes either from the master (a *command* frame) or from a slave (a *status / response* frame).
- **Classic checksum** covers the data only and is used for diagnostic frames `0x3C`/`0x3D`. **Enhanced checksum** covers the PID plus the data and is used for normal frames.
- `0x3C` is the master diagnostic request and `0x3D` is the slave diagnostic response.

## Stage 00: Relay power-up and first ID scan
File: `00_relay_power_wakeup_bruteforce.c`

- **Question:** Will the ECU answer once it has power, and on which frame ID?
- **Method:**
  1. A relay switches the ECU's 12 V supply on when the LIN button is pressed. The firmware then waits 2.5 s for the ECU to boot.
  2. Probe a guessed status frame ID (`0x22`).
  3. Send a header for every ID from `0x00` to `0x3F` and print any reply.
  4. Switch the power off again.
  It also has a UART loopback self-test: hold UP at boot.
- **Result:** The guessed ID `0x22` never answered, and stage 01's log confirms it stays silent. The next stage dropped the relay and powered the ECU directly. It added a LIN wake-up pulse and a diagnostic request, and scanned several baud rates.

## Stage 01: Multi-baud scan and ID brute force
File: `01_multi_baud_bruteforce.c`

- **Question:** What baud rate does the ECU use, and which frame IDs does it answer?
- **Method:**
  1. Run a UART loopback self-test.
  2. Try 9600, 10400, 19200 and 20000 baud. At each one, send a wake-up pulse and a diagnostic request on `0x3C`, then poll `0x3D`. Accept only replies with a valid checksum.
  3. Once a baud rate answers, send a header for every ID from `0x00` to `0x3F` and record any reply with a valid checksum.
- **Result** (real log: [`results/lin_multi_baud_discovery_results.txt`](../results/lin_multi_baud_discovery_results.txt)):
  - No reply at 9600 or 10400. **The ECU answers at 19200 baud.**
  - Diagnostic reply on `0x3D`: `02 06 F2 65 00 FE FF FF` + checksum `A0`.
  - Only **ID `0x16`** answers the ID scan: `11 60 32 FF FF FF FF FF` + `85`. This is the FL door's status frame.

## Stage 02: Focused status polling
File: `02_focused_status_polling.c`

- **Question:** What does the status frame look like at rest, and does it change?
- **Method:** Poll the status ID every 100 ms at 19200 baud and flag any change from the baseline.
- **Result:** The idle baseline status frame was captured for each ECU. The final firmware uses it for its heartbeat and health checks.

## Stage 03: Command frame discovery
File: `03_command_frame_discovery.c`

- **Question:** Which master-published frame ID moves the motor?
- **Method:** Send controlled payload patterns on a list of candidate IDs. After each one, poll the status frame 10 times and watch the motor.
- **Result:** The **command frame ID `0x15`** was identified, along with the first payloads that move the motor.

## Stage 04: Frame classification, stop and sleep
File: `04_frame_classification.c`

- **Question:** How do we stop reliably, and how does the ECU behave around sleep?
- **Method:**
  - Hold a command while repeating it every 50 ms.
  - Detect when the motor stops from changes in the status frame.
  - Classify the ECU's responses before and after sleep.
- **Result:** An all-`FF` **neutral payload**, sent as a short burst, stops the motor reliably. The LIN go-to-sleep frame (`0x3C`: `00 FF FF FF FF FF FF FF`) puts the ECU to sleep, and a break pulse wakes it again.

## Stage 05: Command payloads for all doors
File: `05_all_motor_payload_discovery.c`

- **Question:** Which byte and bit pattern means UP or DOWN for each door?
- **Method:** Send each candidate first byte quickly, and check the status frame within 50 ms to spot movement. After each try, send neutral.
- **Result:** Each door uses a different byte and bit in the 8-byte command payload:

| Door | Status ID | Diagnostic NAD | Manual UP | Manual DOWN |
|---|---|---|---|---|
| FL | `0x16` | `0x02` | `01 00 00 00 00 00 00 00` | `02 00 00 00 00 00 00 00` |
| FR | `0x17` | `0x04` | `40 00 00 00 00 00 00 00` | `20 00 00 00 00 00 00 00` |
| RL | `0x18` | `0x08` | `00 08 00 00 00 00 00 00` | `00 04 00 00 00 00 00 00` |
| RR | `0x19` | `0x10` | `00 00 03 …` *(unconfirmed)* | `00 00 01 00 00 00 00 00` |

All doors receive commands on frame ID `0x15` with an enhanced checksum.

## Stage 06: Rear-right UP investigation
File: `06_rr_cw_investigation.c`

- **Question:** What is the RR UP (clockwise) command?
- **Method:** Hold each candidate for a longer confirmation window and ignore status-only changes.
- **Result:** A candidate (`00 00 03 …`) is stored, but **physical UP movement was not confirmed**. The production firmware therefore keeps it disabled (`ENABLE_RR_UP_COMMANDS 0`), which is a deliberate safety choice.

## How door auto-detection works

The diagnostic request `7F 06 B2 00 FF 7F FF FF` on `0x3C` is a LIN **Read-By-Identifier** service (SID `0xB2`) sent to the wildcard node address `0x7F`. Each door ECU replies on `0x3D` with its own **node address (NAD)** in the first byte: FL `0x02`, FR `0x04`, RL `0x08`, RR `0x10`. The rest of the reply is the positive response (`0xF2`) and the supplier and function IDs.

The firmware matches the whole 9-byte reply against its profile table, so it knows which door is connected and which IDs and payloads to use.

## From discovery to product

| Folder | What it is |
|---|---|
| `validated_control/` | First working controller for FL only. Short press gives a manual "jerk", and holding for 5 s or more gives auto/express. |
| `generic_bcm/01` | Same control, but table-driven for all four doors with auto-detection. |
| `generic_bcm/02` | Adds wake retries and recovery after an ECU power loss. |
| `generic_bcm/03` | Adds LIN/UP/DOWN status LEDs. |
| `generic_bcm/04` | Final snapshot of the door profile table. Same logic as 03. |
| `stm32_nucleo_f446re_project/Core/Src/main.c` | **Latest (v4):** PLC hold-to-run mode with debounced inputs, a two-input interlock, and a heartbeat fail-safe. |
