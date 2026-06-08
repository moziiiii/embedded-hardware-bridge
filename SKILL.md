---
name: embedded-hardware-bridge
description: |
  Translate schematic/PCB hardware designs into structured firmware inputs.
  Extract pin assignments, ADC channels, communication buses, and generate
  board_pin.h / board_init.c boilerplate from netlists, schematic screenshots,
  or manual IO descriptions. Then cross-verify firmware against hardware facts.
tags: [embedded, firmware, hardware, schematic, mcu, pinout, bsp, register]
---

# Embedded Hardware Bridge

Turn hardware designs into firmware-ready structured data, then verify
the resulting code against the original hardware.

## When to use

- You just received a schematic (PDF, screenshot, or netlist) and need to
  start firmware development.
- You need an MCU pin assignment table with ADC channels, PWM outputs,
  I2C/UART/SPI buses, and power/control signals.
- You want to generate `board_pin.h` and `board_init.c` from hardware facts.
- You need to verify that existing firmware matches the schematic.
- You are switching MCU packages or migrating a design to a new chip.
- You received an updated schematic and need to find every code change required.

## Input formats (pick one)

| Priority | Format | Accuracy | When to use |
|----------|--------|----------|-------------|
| **1st** | Netlist text file | 100% | Always ask the hardware engineer first |
| **2nd** | Schematic PDF / screenshot | ~95% | If netlist is unavailable |
| **3rd** | Manual IO description | variable | Bare-minimum: pin number + function per row |

If you have a netlist, skip directly to **Phase 2**. If you have screenshots
or a PDF, start from **Phase 1**.

---

## Phase 1: Extract hardware facts

### From netlist (preferred)

A netlist is a text file exported from the EDA tool. Feed it directly:

```
This is the netlist for my project. Extract every MCU connection:
- For each MCU pin: give me the net name, connected components, and function.
- Identify ADC channels: trace back to the voltage divider, give me resistor
  values and the voltage-divider formula.
- Identify communication buses: I2C device addresses, UART baud rates,
  SPI chip-select pins.
- Identify PWM outputs: timer channel, control target, default safe state.
- Flag every active-low signal explicitly.
- Output everything as structured tables.
```

### From schematic screenshots / PDF

If no netlist is available, provide the schematic and key context:

```
[Attach schematic PDF or screenshots]

Project:
- Product function: [describe in 1-2 sentences]
- MCU model: [exact part number]
- Power rails: [e.g. 3.3V, 5V, 12V, battery]
- Key modules: [e.g. buttons, fans, sensors, charger IC, boost converter]

Please extract:
1. MCU pin assignment table: pin #, port name, net name, function, I/O/Analog/AF.
2. ADC channels: sense network, resistor divider, max input voltage,
   conversion formula, over-range risk.
3. Communication buses: I2C/SPI/UART device list, addresses, pull-up resistors,
   chip-select states.
4. PWM/Timer outputs: control target, suggested frequency, safe default state.
5. Power/charging/on-off control: critical nets, active level, power-up default.
6. Signal polarity: list every active-low or edge-triggered signal explicitly.
7. Uncertainty items: things you need me to zoom in on or check in the datasheet.
```

---

## Phase 2: Generate firmware scaffold

Once the hardware fact table exists, generate the firmware hardware-abstraction
files. This phase is deterministic — it should produce identical output given
identical hardware facts.

### Required outputs

```
board_pin.h     — All pin definitions, macros, ADC channel mapping
board_init.c    — GPIO direction, pull-ups, analog selects, peripheral init
drv_adc.h/c     — ADC read functions with voltage-divider math
drv_pwm.h/c     — PWM set-frequency / set-duty functions
drv_i2c.h/c     — I2C init, read/write wrappers
drv_gpio.h/c    — GPIO read/write/toggle wrappers with active-level handling
```

### Coding rules (embedded-specific, non-negotiable)

- Use the **datasheet, not internet guesses**, for register names and bit
  definitions.
- Every output pin must be set to a **safe default level before setting TRIS**
  to output (prevents transient shorts).
- ADC, PWM, and communication peripherals **software-disabled before sleep**,
  re-initialized on wake.
- No `float` / `double` if the MCU lacks an FPU.
- No `malloc` / `free` on bare-metal targets.
- ISR functions: no blocking calls, no while-loop waits.
- All `volatile` shared variables accessed atomically (disable interrupts
  around reads/writes on 8-bit MCUs).

---

## Phase 3: Cross-verify firmware against hardware

After firmware scaffolding is written, verify it.

### Review checklist

```
□ Every MCU pin's TRIS direction matches the schematic (output vs input).
□ Every analog pin's ANSEL/ADC channel register matches the netlist.
□ Every PWM pin's timer, channel, and polarity registers are correct.
□ Every I2C device address and pull-up resistor value is confirmed.
□ Every active-low signal is read/written with the correct polarity.
□ ADC voltage-divider formulas match the resistor values on the schematic.
□ Cross-file data flows: sensor → global variable → business logic —
  are the units consistent at every step?
□ All unused pins are set to a safe state (input with pull-up, or output low).
□ Watchdog is fed inside every blocking delay.
□ Power-up default: all output pins start in a safe state (loads OFF).
```

### Pre-burn minimum safety gate

Before flashing any board, verify at least these three:

```
1. ADC over-range: does max input voltage exceed the MCU reference voltage?
2. GPIO default safety: all output pins default to safe state (load OFF)?
3. Power sequencing: does the power-on/off logic sequence correctly?
```

If **any** of these fail, **do not flash the board**.

---

## Phase 4: Deliver

Embedded-specific delivery steps (non-obvious for AI-generated code):

1. **Flatten the directory**: all `.c` and `.h` in one folder.
2. **Strip include paths**: `#include "drv_xxx.h"` — no subdirectory prefixes.
3. **Convert encoding**: target IDE may require ANSI/GBK instead of UTF-8.
4. **Remove UTF-8 BOM**: it can break vendor compilers (e.g. `picc.exe`).
5. **Add debug variables**: flat global `volatile` variables the IDE's watch
   window can see:
   ```c
   volatile unsigned char DBG_STATE;
   volatile unsigned int  DBG_BAT_MV;
   // Update in a periodic task for IDE watch window visibility.
   ```
6. **Write a delivery note**: power-up behavior, integration test steps,
   known risk points.

---

## Phase 5: Integration testing

Hardware bring-up order — add one module at a time:

| Round | Test | Method | Pass criteria |
|-------|------|--------|--------------|
| 1 | SysTick + GPIO | Blink LED | LED toggles at 1 Hz |
| 2 | Buttons | Short/long press | Debounce correct |
| 3 | ADC | Compare to DMM | Voltage error < 2% |
| 4 | PWM | Oscilloscope | Frequency and duty correct |
| 5 | I2C/SPI/UART | Read sensor ID / loopback | Correct ID or echo |
| 6 | Full state machine | Walk every state | Every transition correct |

---

## Supporting files

Place these in the project root:

```
CONSTITUTION.md   — Non-negotiable hardware rules (ISR limits, FPU ban, etc.)
io_table.md       — Hardware fact table (generated in Phase 1)
board_pin.h       — Pin definitions (generated in Phase 2)
board_init.c      — Peripheral init (generated in Phase 2)
```

---

## Limitations

- This skill **cannot** parse proprietary EDA binary formats (PADS `.sch`,
  Altium `.SchDoc` without `altium-monkey`, etc.). Use netlist export or
  screenshots as workarounds.
- AI-extracted pin assignments from screenshots are ~95% accurate. Always
  cross-check against the netlist if available.
- This skill does **not** replace hardware debugging (oscilloscope, logic
  analyzer, JTAG/SWD). It reduces software bugs, not hardware faults.
