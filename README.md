# pBITz Platform

pBITz is the shared hardware platform used by the **Coffee Series** of homebrew computers. It provides the common physical and electrical foundation — backplane, power control, service interfaces, reusable KiCad assets, test hardware, and platform tools — while leaving the personality of each machine to its CPU board and machine-specific peripherals.

The goal is not to define one rigid computer architecture. Instead, pBITz provides a practical common platform on which very different machines can be built: 8-bit, 16-bit, and eventually wider systems can share the same mechanical infrastructure, power distribution, expansion-card format, and platform services without pretending that a Z80, 6309, 68000, 68030, or ARM-class processor has the same bus semantics.

This repository contains the **platform-level** hardware and support material. Machine-specific CPU, video, sound, and operating-system work normally lives in the corresponding Coffee Series repository.

> **Important:** pBITz is a custom parallel expansion bus. The current platform uses DIN 41612-style connectors; it is not PCI Express and is not electrically compatible with PC expansion buses.

## Design philosophy

pBITz deliberately keeps the backplane simple.

The backplane distributes signals, power, clocks, and service connections, but it does not try to impose a universal bus controller or chipset. The CPU board for a particular machine is responsible for turning that machine's native bus into the pBITz signal conventions and for defining how the generic control lines are interpreted.

That leads to a few guiding principles:

- **The backplane is infrastructure, not the computer.** The machine emerges from the CPU board, IO Controller, and installed peripheral cards.
- **Use conventional parallel buses where practical.** Address, data, read/write, interrupt, arbitration, and timing signals remain visible rather than being hidden behind an FPGA SoC.
- **Share mechanics and services without forcing identical CPU semantics.** An 8-bit machine may use only part of the available data/address width; a 16-bit machine can use more of it.
- **Keep platform services separate from machine personality.** Power management, the service SPI bus, removable storage, USB hosting, test interfaces, and reusable CAD assets belong here.
- **Prefer inspectable hardware.** The project is intended to remain understandable with schematics, logic probes, oscilloscopes, PLDs, and ordinary firmware tools.

## Platform overview

At a high level, a pBITz system looks like this:

```text
                        +----------------------+
                        |   Coffee Series      |
                        |     CPU board        |
                        +----------+-----------+
                                   |
          +------------------------+------------------------+
          |                    pBITz bus                    |
          | address / data / control / clocks / power      |
          +-----+------------------+------------------+-----+
                |                  |                  |
          +-----+-----+      +-----+-----+      +-----+-----+
          |  Video    |      |   Sound   |      | Other I/O |
          |   card    |      |   card    |      |   cards   |
          +-----------+      +-----------+      +-----------+
                                   |
                           pBITz backplane
                                   |
                +------------------+------------------+
                |                                     |
          ATX power / PMU                    service / SPI header
                                                      |
                                             +--------+--------+
                                             |   mezzanine     |
                                             | USB host + SD   |
                                             +-----------------+
```

The exact cards and signal meanings depend on the machine being built.

## pBITz bus

The pBITz expansion bus is intentionally broader than any one Coffee Series CPU requires. The current signal set provides room for:

- up to a **24-bit address bus**;
- up to a **16-bit data bus**;
- generic CPU control/status lines (`CTRL_*`) whose meanings are assigned by the CPU implementation;
- chip-select lines;
- multiple clock lines;
- interrupt and bus-control signals;
- a shared platform **SPI/service bus**;
- system reset and power-management signals;
- `+5 V`, `+3.3 V`, and ground distribution.

An individual machine does not need to use every signal. For example, an 8-bit CPU board can use the low data byte and the subset of address/control lines it needs, while a 16-bit machine can make use of the wider path.

The generic control-line approach is deliberate. Signals such as read, write, memory request, I/O request, bus arbitration, wait/ready, interrupt, and byte strobes differ significantly between processor families. The CPU board performs the required mapping rather than forcing every machine through a single artificial protocol.

See [`pBITz_Control_Signals.md`](pBITz_Control_Signals.md) and, more importantly, the current KiCad schematics for the concrete mapping used by a particular hardware revision.

## Backplane

[`pBITzBackplane/`](pBITzBackplane/) contains the main pBITz backplane.

It is primarily a passive bus-distribution board with an always-on power-management island. Its responsibilities include:

- distributing the pBITz address, data, control, clock, and service signals between slots;
- distributing `+5 V`, `+3.3 V`, and ground from an ATX supply;
- providing bus pull-ups where required;
- exposing the platform service interface on an IDC header;
- coordinating ATX power-up and orderly shutdown through the PMU.

The directory contains the KiCad source, fabrication outputs, architecture notes, and revision-specific validation material.

### Power management

The backplane includes an ATtiny-based **PMU** powered from the ATX standby rail. It handles the enclosure power switch and ATX `PS_ON#` / `PWR_OK` sequencing and can coordinate an orderly shutdown with the machine's IO Controller.

PMU firmware lives under [`Code/PMU/`](Code/PMU/).

## Backplane mezzanine

[`pBITzBackplaneMezzanine/`](pBITzBackplaneMezzanine/) contains the platform I/O mezzanine attached through the backplane service connector.

The current design provides:

- a **MAX3421E** SPI USB host controller;
- an **FE1.1** USB hub;
- multiple downstream USB connections;
- an **SD card** operating in SPI mode;
- the necessary buffering, level conversion, power distribution, and card-detect/interrupt paths.

This keeps common storage and USB services out of the CPU-specific logic while still making them available to machines through the platform service interface.

## Platform support hardware

### pBITz Test Card

[`pBITzTestCard/`](pBITzTestCard/) is a work-in-progress platform development and diagnostic card. The current schematic work includes concepts for:

- a console interface;
- external programming support;
- an MCU-based controller;
- USB/JTAG programming and test functions.

This project is currently development hardware rather than a finished production design.

### pBITz Terminal

[`pBITzTerminal/`](pBITzTerminal/) explores a standalone serial terminal suitable for use with pBITz and other retro systems.

The current architecture is based around an HD63C03-family CPU and MC6845-compatible CRTC, with a VGA-style text display, PS/2 keyboard support, serial communications, banked SRAM, programmable character memory, and PLD-based video/bus arbitration.

The terminal is an experimental design and should not be treated as a completed board yet.

## Platform tools and firmware

[`Code/`](Code/) contains code that belongs to the platform rather than to one specific Coffee Series machine.

Current contents include:

- [`Code/PMU/`](Code/PMU/) — ATtiny power-management firmware, including a host-testable state machine;
- [`Code/tools/kicad_nettrace.py`](Code/tools/kicad_nettrace.py) — a KiCad schematic connectivity-analysis helper used during design reviews.

A typical connectivity check is:

```sh
python3 Code/tools/kicad_nettrace.py pBITzBackplane/pBITzBackplane.kicad_sch --check
```

`kicad_nettrace.py` is a review aid, not a replacement for KiCad's own connectivity model. When the helper and KiCad disagree, the **KiCad schematic/netlist is authoritative**.

## Shared KiCad assets

[`99-KiCad-Lib/`](99-KiCad-Lib/) is the shared CAD library used across the platform and Coffee Series projects. It contains:

- custom schematic symbols;
- footprints;
- 3D models;
- pBITz connector definitions;
- reusable parts used by the various machines;
- project artwork and board logos.

Keeping these assets in the platform repository avoids maintaining slightly different copies in every machine repository.

## Datasheets

[`00-Datasheets/`](00-Datasheets/) collects component documentation used while designing the platform and Coffee Series machines. It is organized by broad component class, including:

- analog;
- CPUs and MCUs;
- connectors;
- logic;
- memory;
- oscillators;
- peripherals;
- power components.

The datasheet directory is a working engineering reference, not intended to be a complete historical semiconductor archive.

## Repository layout

| Path | Purpose | Status |
| --- | --- | --- |
| `pBITzBackplane/` | Main backplane, ATX interface, PMU hardware, service bus | Active hardware design |
| `pBITzBackplaneMezzanine/` | Shared USB/SD service mezzanine | Active hardware design |
| `Code/PMU/` | Backplane power-management firmware | Implemented / evolving |
| `Code/tools/` | Platform engineering and KiCad-analysis tools | Active |
| `pBITzTestCard/` | Platform diagnostic/programming card | Work in progress |
| `pBITzTerminal/` | Standalone serial/VGA terminal experiment | Experimental |
| `99-KiCad-Lib/` | Shared KiCad symbols, footprints, and 3D models | Shared infrastructure |
| `00-Datasheets/` | Engineering reference datasheets | Shared infrastructure |
| `pBITz_Control_Signals.md` | Cross-CPU control-signal mapping notes | Design reference |

## Relationship to the Coffee Series

pBITz exists so that the Coffee Series machines can share a physical platform without becoming variants of the same computer.

Examples of machines built or planned around the platform include architectures based on processors such as the Z80, HD6309, 68000/68030, and ARM-class CPUs. Each machine is free to define its own memory architecture, CPU timing, interrupt model, video hardware, sound hardware, firmware, and operating system while reusing the pBITz mechanical and platform infrastructure where it makes sense.

In other words:

> **pBITz provides the chassis and electrical vocabulary; the CPU and peripheral set define the computer.**

## Source of truth and revision notes

This is an actively developed hardware platform. Schematics, PCB layouts, production files, architecture notes, and validation reports do not always change at the same time.

When reviewing a design:

1. treat the current **KiCad schematic and PCB files** as the source of truth for the active hardware revision;
2. use architecture documents to explain intent and operation;
3. check the date/revision of validation reports and fabrication outputs before assuming they describe the current board;
4. regenerate production files after a hardware revision rather than assuming an older BOM/Gerber package remains current.

This is particularly important when a mechanical or connector revision changes without altering the higher-level bus concept.

## Hardware status and validation

Validation reports in this repository document engineering checks performed against particular revisions of the design files. They are useful review records, but they are not certification and do not replace physical bring-up.

Unless explicitly stated otherwise, repository-level validation may include schematic/netlist consistency and selected PCB checks, while powered measurements, signal-integrity characterization, enclosure fit, manufacturing tolerances, and real-hardware testing remain separate activities.

## License

Unless noted otherwise, the hardware and associated source material in this repository are released under the **Solderpad Hardware License v2.1**, with the option described in the license to treat the work under Apache License 2.0.

See [`Licence.md`](Licence.md) for the complete terms.

---

pBITz is an evolving homebrew-computer platform. Expect the schematics to remain understandable, the buses to remain visible, and occasionally a logic analyzer to become part of the debugging process.
