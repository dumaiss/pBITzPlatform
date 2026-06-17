# pBITzBackplaneMezzanine Architecture

This document describes the active `pBITzBackplaneMezzanine` KiCad project as
checked out in this repository. The source of truth for connectivity is the
schematic hierarchy rooted at `pBITzBackplaneMezzanine.kicad_sch`, with the
generated KiCad schematic netlist used to resolve cases where
`kicad_nettrace.py` misses a connection. If the schematic and
`kicad_nettrace.py` disagree, the schematic wins.

## Scope And Sources

Active schematic hierarchy:

| Sheet | File | Role |
| --- | --- | --- |
| Root | `pBITzBackplaneMezzanine.kicad_sch` | Board-level hierarchy wiring between mezzanine connector, USB subsystem, and SD-card subsystem. |
| USBPorts | `USBPorts.kicad_sch` | MAX3421E SPI USB host controller, FE1.1 USB hub, downstream USB connectors, fuses, crystals, and USB support parts. |
| MezzanineConnector | `MezzanineConnector.kicad_sch` | 2x10 IDC mezzanine connector and mounting holes. |
| SD_Card | `SD_Card.kicad_sch` | SD-card socket, CD74HC4050 level shifter, SD-card pull-up, and decoupling. |

Trace commands used:

```sh
HOME=/tmp XDG_CONFIG_HOME=/tmp XDG_DATA_HOME=/tmp kicad-cli sch export netlist -o /tmp/pbitzbackplane-mezzanine-kicad.net pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch
python3 Code/tools/kicad_nettrace.py pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch
python3 Code/tools/kicad_nettrace.py pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch --check
```

`kicad_nettrace.py` reported:

| Sheet | Pin hits | Coverage |
| --- | ---: | ---: |
| Root | 0/0 | 0% |
| USBPorts | 136/173 | 79% |
| MezzanineConnector | 27/28 | 96% |
| SD_Card | 31/37 | 84% |

The tracer found 97 nets and 211 placed pins, with no output-driver conflicts.
Its `--check` output is useful for finding obvious opens, but it misses several
connections that the KiCad schematic netlist resolves, including global power
connections on J5, fuses, the SD-card VDD pin, and 4050 power pins. The KiCad
schematic netlist is therefore used for the final connectivity tables.

## Board Overview

The mezzanine is an SPI-attached peripheral module. It takes power, ground, SPI,
interrupt, and card-detect/service signals from a 2x10 IDC mezzanine connector
and provides:

- A MAX3421E SPI USB host controller.
- An FE1.1 USB 2.0 hub behind the MAX3421E host interface.
- Two downstream USB-A receptacles on a stacked USB connector.
- Two auxiliary downstream USB 1x4 headers.
- An SPI-mode SD-card socket.

Physical PCB facts from `pBITzBackplaneMezzanine.kicad_pcb`:

| Item | Value |
| --- | --- |
| KiCad PCB version | 20260206 format |
| Copper stack | 4 layers: `F.Cu`, `In1.Cu`, `In2.Cu`, `B.Cu` |
| Footprints | 41 placed footprints |
| Mezzanine connector | `J5`, 2x10 IDC, footprint `Connector_IDC:IDC-Header_2x10_P2.54mm_Vertical` |
| USB-A connector | `J1`, stacked USB-A, footprint `Connector_USB:USB3_A_Molex_48406-0001_Horizontal_Stacked` |
| Auxiliary USB headers | `J2`, `J3`, 1x4 pin headers |
| SD-card connector | `J4`, footprint `Footprints:AMPHENOL_10067847-001RLF` |

Mechanical fit, board outline, connector mating height, and enclosure concerns
are outside the validation scope.

## Mezzanine Connector J5

`J5` is the module interface to the rest of the pBITz system. It carries both
rails, ground, SPI, interrupt, and SD-card-present information.

| J5 pin | Net | Role |
| ---: | --- | --- |
| 1 | `+5V` | USB hub supply and downstream USB VBUS source. |
| 2 | `+3V3` | Logic, MAX3421E, 4050 buffers, and SD-card supply. |
| 3 | `/MezzanineConnector/MOSI` | SPI MOSI from host/backplane into the mezzanine. |
| 4 | `/MezzanineConnector/SPI_CS0` | SD-card chip select input. |
| 5 | `/MezzanineConnector/MISO` | Shared SPI MISO back to host/backplane. |
| 6 | `GND` | Ground. |
| 7 | `GND` | Ground. |
| 8 | `/MezzanineConnector/SPI_CS1` | USB controller chip select input. |
| 9 | `/MezzanineConnector/SPI_CLK` | SPI clock input. |
| 10 | `GND` | Ground. |
| 11 | `GND` | Ground. |
| 12 | `unconnected-(J5-Pin_12-Pad12)` | Intentionally unused. |
| 13 | `/MezzanineConnector/~{SPI_INT}` | MAX3421E interrupt output to host/backplane. |
| 14 | `GND` | Ground. |
| 15 | `GND` | Ground. |
| 16 | `GND` | Ground. |
| 17 | `/MezzanineConnector/~{SD_PRESENT}` | SD-card-detect output to host/backplane. On the backplane service header this occupies the pin named `SPI_CD`. |
| 18 | `GND` | Ground. |
| 19 | `GND` | Ground. |
| 20 | `GND` | Ground. |

## Power Architecture

The board does not generate the main `+5V` or `+3V3` rails. They enter through
`J5`.

| Rail | Source | Loads |
| --- | --- | --- |
| `+5V` | `J5.1` | FE1.1 `VBUSM`/`VDD5`, USB downstream VBUS fuses, bulk/decoupling capacitors. |
| `+3V3` | `J5.2` | MAX3421E `VL`, `VCC`, and `~{RES}` pull-high; U3/U4 CD74HC4050 supplies; SD-card `VDD`; SD MISO pull-up; local decoupling. |
| `+3V3_INTERNAL` | FE1.1 `U1.21/VD33F` | FE1.1 support net for `OVCJ`, `BUSJ`/`XRSTJ` pull-ups, and local decoupling. |
| `GND` | J5 ground pins | Logic ground, shields, mounting-hole pads, hub/controller ground, SD ground. |

Downstream USB VBUS is protected per port by 500 mA polyfuses:

| Fuse | Input | Output |
| --- | --- | --- |
| `F2` | `+5V` | `/USBPorts/VBUS_PIN1` to stacked USB-A port 1. |
| `F1` | `+5V` | `/USBPorts/VBUS_PIN2` to stacked USB-A port 2. |
| `F3` | `+5V` | `/USBPorts/VBUS_PIN3` to auxiliary header `J2`. |
| `F4` | `+5V` | `/USBPorts/VBUS_PIN4` to auxiliary header `J3`. |

## SPI Fanout And Level Buffering

The mezzanine uses two CD74HC4050 non-inverting buffers to present 3.3 V logic
to local devices on host-driven SPI signals.

| Signal | Host/backplane net | Buffer | Local target net | Target |
| --- | --- | --- | --- | --- |
| USB MOSI | `/MezzanineConnector/MOSI` | `U3.3 -> U3.2` | `/USBPorts/MOSI` | `U2.16/MOSI` |
| USB chip select | `/MezzanineConnector/SPI_CS1` | `U3.5 -> U3.4` | `/USBPorts/~{CS}` | `U2.14/~{SS}` |
| USB SPI clock | `/MezzanineConnector/SPI_CLK` | `U3.7 -> U3.6` | `/USBPorts/CLK` | `U2.13/SCLK` |
| SD MOSI | `/MezzanineConnector/MOSI` | `U4.9 -> U4.10` | `/SD_Card/MOSI` | `J4.2/CMD` |
| SD chip select | `/MezzanineConnector/SPI_CS0` | `U4.11 -> U4.12` | `/SD_Card/~{CS}` | `J4.1/DAT3/CD` |
| SD SPI clock | `/MezzanineConnector/SPI_CLK` | `U4.14 -> U4.15` | `/SD_Card/CLK` | `J4.5/CLK` |

Shared return and status paths:

| Signal | Net | Members |
| --- | --- | --- |
| MISO | `/MezzanineConnector/MISO` | `J5.5`, `U2.15/MISO`, `J4.7/DAT0`, `R6.2`. |
| MISO pull-up | `+3V3` through `R6` | `R6.1` is `+3V3`, `R6.2` is MISO. |
| USB interrupt | `/MezzanineConnector/~{SPI_INT}` | `U2.18/INT`, `J5.13`. |
| SD present | `/MezzanineConnector/~{SD_PRESENT}` | `J4.CD`, `J5.17`. |

Unused 4050 inputs are tied to `GND`, and their paired outputs are intentionally
left unconnected. This keeps unused buffer inputs from floating.

## USB Subsystem

`U2` is a MAX3421E USB peripheral/host controller with SPI interface. In this
design it is the SPI-attached USB host controller for the module.

Key MAX3421E connections:

| U2 pin | Function | Net |
| ---: | --- | --- |
| 2 | `VL` | `+3V3` |
| 3, 19, 33 | Ground/exposed pad | `GND` |
| 12 | `~{RES}` | `+3V3` |
| 13 | `SCLK` | `/USBPorts/CLK` |
| 14 | `~{SS}` | `/USBPorts/~{CS}` |
| 15 | `MISO` | `/MezzanineConnector/MISO` |
| 16 | `MOSI` | `/USBPorts/MOSI` |
| 18 | `INT` | `/MezzanineConnector/~{SPI_INT}` |
| 20 | `D-` | `Net-(U2-D-)` through `R5` to `/USBPorts/D-` |
| 21 | `D+` | `Net-(U2-D+)` through `R4` to `/USBPorts/D+` |
| 23 | `VCC` | `+3V3` |
| 24, 25 | `XI`, `XO` | 12 MHz crystal `Y2` with 22 pF load capacitors. |

The MAX3421E USB pair connects to the FE1.1 upstream port:

| Path | Net |
| --- | --- |
| `U2.21/D+` -> `R4.2` -> `R4.1` -> `U1.16/DPU` | `Net-(U2-D+)` then `/USBPorts/D+` |
| `U2.20/D-` -> `R5.2` -> `R5.1` -> `U1.15/DMU` | `Net-(U2-D-)` then `/USBPorts/D-` |

`U1` is the FE1.1 USB 2.0 hub. It is supplied from `+5V`, generates
`+3V3_INTERNAL` on `VD33F`, and uses a 12 MHz crystal `Y1`. `R1` and `R2`
pull `BUSJ` and `XRSTJ` to `+3V3_INTERNAL`; `R3` is the 2.7k `REXT` resistor
to ground. `OVCJ` is tied to `+3V3_INTERNAL`, which holds the overcurrent input
inactive in this design.

Downstream ports:

| Port | FE1.1 nets | Connector |
| --- | --- | --- |
| 1 | `/USBPorts/DM1`, `/USBPorts/DP1`, `/USBPorts/VBUS_PIN1` | `J1` stacked USB-A port 1 |
| 2 | `/USBPorts/DM2`, `/USBPorts/DP2`, `/USBPorts/VBUS_PIN2` | `J1` stacked USB-A port 2 |
| 3 | `/USBPorts/DM3`, `/USBPorts/DP3`, `/USBPorts/VBUS_PIN3` | `J2` 1x4 auxiliary header |
| 4 | `/USBPorts/DM4`, `/USBPorts/DP4`, `/USBPorts/VBUS_PIN4` | `J3` 1x4 auxiliary header |

The current schematic and PCB assign the stacked USB-A connector `J1` USB 2.0
pins as follows:

| J1 pin | USB-A function | Current net |
| ---: | --- | --- |
| 2 | `D1-` | `/USBPorts/DM1` |
| 3 | `D1+` | `/USBPorts/DP1` |
| 11 | `D2-` | `/USBPorts/DM2` |
| 12 | `D2+` | `/USBPorts/DP2` |

This matches the expected polarity: FE1.1 `DMx` nets go to USB `D-` pins, and
`DPx` nets go to USB `D+` pins.

The USB3 SuperSpeed pins on `J1` are intentionally unconnected. This board uses
USB 2.0 signaling from the FE1.1 hub.

## SD-Card Subsystem

`J4` is wired for SPI-mode SD-card operation.

| J4 pin | SD function | Net |
| --- | --- | --- |
| 1 | `DAT3/CD` used as SPI CS | `/SD_Card/~{CS}` |
| 2 | `CMD` used as SPI MOSI | `/SD_Card/MOSI` |
| 3 | `VSS` | `GND` |
| 4 | `VDD` | `+3V3` |
| 5 | `CLK` | `/SD_Card/CLK` |
| 6 | `VSS` | `GND` |
| 7 | `DAT0` used as SPI MISO | `/MezzanineConnector/MISO` |
| 8 | `DAT1` | Intentionally unconnected. |
| 9 | `DAT2` | Intentionally unconnected. |
| `CD` | Card detect | `/MezzanineConnector/~{SD_PRESENT}` |
| `WP` | Write protect | Intentionally unconnected. |
| `G1`, `G2` | Shield/ground | `GND` |

`U4` buffers host-driven SD inputs down to the local 3.3 V domain. The SD MISO
line is not buffered; it is shared directly with the mezzanine MISO net and has
a 10k pull-up to `+3V3` through `R6`.

## Intentional Opens And No-Connects

Important intentional no-connects in the schematic netlist:

| Ref pins | Reason |
| --- | --- |
| `J5.12` | Unused mezzanine connector pin. |
| `J1.5`, `J1.6`, `J1.8`, `J1.9`, `J1.14`, `J1.15`, `J1.17`, `J1.18` | USB3 SuperSpeed pins unused on a USB 2.0 hub design. |
| `J4.8`, `J4.9`, `J4.WP` | SD `DAT1`, `DAT2`, and write-protect unused in SPI-mode design. |
| `U1.12`, `U1.13`, `U1.22`, `U1.23`, `U1.24`, `U1.25`, `U1.27`, `U1.28` | FE1.1 pins not used by this design. |
| `U2` GPIN/GPOUT pins, `U2.17/GPX`, `U2.22/VBCOMP` | MAX3421E auxiliary GPIO/status features not used by this design. |
| `U3.10`, `U3.12`, `U3.15`, `U4.2`, `U4.4`, `U4.6` | Outputs of unused 4050 buffer sections. |

Review note: the local MAX3421E datasheet identifies `VBCOMP` as the VBUS
comparator input and recommends bypassing it to ground. The current schematic
marks `U2.22/VBCOMP` no-connect. This was not counted as a blocking validation
failure because the comparator may be intentionally unused, but it should be
reviewed before fabrication.
