# Embedded Hardware Bridge

A Claude Code skill that bridges the gap between hardware design and firmware
development. Give it a schematic (netlist, PDF, or screenshot) and it produces
a hardware fact table, generates `board_pin.h` / `board_init.c` scaffolding,
and cross-verifies firmware against the original circuit.

## The problem

Embedded firmware developers spend hours on tedious, error-prone work:

- Manually transcribing pin assignments from schematics to code
- Calculating ADC voltage-divider formulas by hand
- Tracing I2C/SPI/UART connections across schematic pages
- Hunting for active-low signals that are inverted in code
- Finding register-bit typos that compile fine but brick the board

AI coding tools (Claude Code, Copilot, Cursor) are great at writing
application logic but terrible at hardware correctness — they can't read
schematics, they guess register names, and they don't know if a signal is
active-high or active-low.

This skill gives AI tools **hardware ground truth** to work from.

## How it works

```
Schematic (netlist / PDF / screenshot)
         │
         ▼
    Phase 1: Extract hardware facts
    - Pin assignment table
    - ADC channels & voltage divider formulas
    - Communication bus topology
    - Signal polarity map
         │
         ▼
    Phase 2: Generate firmware scaffold
    - board_pin.h, board_init.c
    - Driver stubs (ADC, PWM, I2C, GPIO)
         │
         ▼
    Phase 3: Cross-verify
    - Register-by-register review
    - Hardware-consistency checklist
    - Pre-burn safety gate
         │
         ▼
    Phase 4: Deliver
    - Encoding conversion, directory flattening
    - Debug watch variables
    - Integration test plan
```

## Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/YOUR_USERNAME/embedded-hardware-bridge.git \
  ~/.claude/skills/embedded-hardware-bridge
```

Or install via Claude Code:

```
/install-skill embedded-hardware-bridge
```

## Usage

### Quick start

```
I just received the schematic for a [product]. MCU is [model].
Please use the embedded-hardware-bridge skill to extract the hardware
facts and generate the firmware scaffold.
```

### With a netlist (ideal — 100% accurate)

Always ask your hardware engineer for a netlist export first. It takes them
one minute: PADS → File → Export → Netlist. Then:

```
[Paste netlist text]

Extract the full hardware fact table from this netlist.
MCU: [model]. Product: [description].
```

### With schematic screenshots / PDF (fallback)

```
[Attach schematic]

Use embedded-hardware-bridge to extract hardware facts.
MCU: SC8F096AD824NPR.
Product: mask controller with fan, battery, boost converter.
Key modules: buttons ×3, LED ×4, TH06 temp sensor, HP203N pressure sensor,
CN3302 charger, fan PWM, boost EN.
```

### Verify existing firmware

```
Compare these files against the hardware fact table:
- board_pin.h
- board_init.c
- drv_adc.c
- drv_pwm.c
- drv_i2c.c

Flag every inconsistency with file name, line number, and the correct value.
```

## What it catches

Real bugs found by this workflow:

| Bug | How it was caught |
|-----|-------------------|
| TRIS direction inverted (output set as input) | Phase 3 cross-verification |
| Altitude/temperature divided by 100 twice | Cross-file data-flow trace |
| PWM period register used TMR2 instead of PWMTL | Register final check |
| Sensor Init called but Read missing from 500ms task | Init/Read pairing check |
| Charging-state exit left stale keyEvent → false power-on | State-transition event-clear check |
| IDE encoding UTF-8 → GBK corrupted comments | Phase 4 deliverable steps |
| Watchdog reset during debug breakpoints | Pre-burn checklist |

## Supporting files

Create these in your project root (the skill will populate them):

```
CONSTITUTION.md   — Non-negotiable hardware rules for your MCU
io_table.md       — Generated hardware fact table
board_pin.h       — Generated pin definitions
board_init.c      — Generated peripheral initialization
```

## Limitations

- Cannot parse proprietary EDA binary formats (PADS `.sch`, Altium `.SchDoc`
  without `altium-monkey`, etc.). **Always ask for a netlist export first.**
- AI-extracted pin assignments from screenshots are ~95% accurate.
  Cross-check against the datasheet.
- Does not replace hardware debugging (oscilloscope, logic analyzer, JTAG).
  It reduces software bugs, not PCB mistakes.

## License

MIT — use it, modify it, ship products with it.

## Related

- [Tansuo2021/ADtoKeil](https://github.com/Tansuo2021/ADtoKeil) — Altium `.SchDoc` → Keil firmware skill suite
- [github/spec-kit](https://github.com/github/spec-kit) — Spec-Driven Development framework
- [anthropics/skills](https://github.com/anthropics/skills) — Agent Skills protocol
