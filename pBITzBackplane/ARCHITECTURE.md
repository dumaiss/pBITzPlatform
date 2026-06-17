# pBITzBackplane Architecture

This document describes the active `pBITzBackplane` KiCad project as checked
out in this repository. The source of truth for connectivity is the schematic
hierarchy rooted at `pBITzBackplane.kicad_sch`, with the generated KiCad
schematic netlist used to resolve cases where `kicad_nettrace.py` misses a
connection. If the schematic and `kicad_nettrace.py` disagree, the schematic
wins.

## Scope And Sources

Active schematic hierarchy:

| Sheet | File | Role |
| --- | --- | --- |
| Root | `pBITzBackplane.kicad_sch` | Board-level mounting holes and hierarchical sheet instances. |
| Power | `Power.kicad_sch` | ATX connector, standby-powered ATtiny controller, power button, LEDs, and ATX `PS_ON#` control. |
| Backplane | `Backplane.kicad_sch` | pBITz slot connectors, service IDC header, bus pull-up resistor networks, and slot rail decoupling. |

Trace commands used:

```sh
python3 Code/tools/kicad_nettrace.py /home/kitamura/Documents/pBITzPlatform/pBITzBackplane/pBITzBackplane.kicad_sch
python3 Code/tools/kicad_nettrace.py /home/kitamura/Documents/pBITzPlatform/pBITzBackplane/pBITzBackplane.kicad_sch --list-nets
python3 Code/tools/kicad_nettrace.py /home/kitamura/Documents/pBITzPlatform/pBITzBackplane/pBITzBackplane.kicad_sch --check
HOME=/tmp XDG_CONFIG_HOME=/tmp XDG_DATA_HOME=/tmp kicad-cli sch export netlist -o /tmp/pbitzbackplane-kicad.net pBITzBackplane/pBITzBackplane.kicad_sch
```

`kicad_nettrace.py` reported:

| Sheet | Pin hits | Coverage |
| --- | ---: | ---: |
| Root | 2/2 | 100% |
| Power | 75/79 | 95% |
| Backplane | 613/616 | 100% |

The tracer found 141 nets and 671 placed pins, with no output-driver conflicts.
Its `--check` output was useful for finding opens, but it also reports some
false opens where the schematic netlist resolves global power nets,
resistor-network common pins, J7 power pins, and repeated hierarchical labels.
The KiCad schematic netlist was therefore used for the final connectivity
tables.

## Board Overview

The backplane is a mostly passive expansion board. It distributes address,
data, control, SPI/service, clock, power, and ground nets across five pBITz bus
connectors. The only active logic is the power-control island around the
ATtiny85 and BSS138 MOSFET in the `Power` sheet.

Physical PCB facts from `pBITzBackplane.kicad_pcb`:

| Item | Value |
| --- | --- |
| KiCad PCB version | 10.0 format |
| Copper stack | 4 layers: `F.Cu`, `In1.Cu`, `In2.Cu`, `B.Cu` |
| Board outline | 243.84 mm x 243.84 mm square |
| Footprints | 54 placed footprints |
| Main power input | `J1`, ATX-24, footprint `Footprints:MOLEX_39281243` |
| pBITz slots | `J2` through `J6`, 98-pin pBITz bus connectors |
| Service header | `J7`, 2x10 IDC, footprint `Connector_IDC:IDC-Header_2x10_P2.54mm_Vertical` |

Slot connector footprints:

| Ref | Value | Footprint |
| --- | --- | --- |
| `J2` | `pBITzBus` | `Footprints:AMPHENOL_10018783-10202TLF` |
| `J3` | `pBITzBus` | `Footprints:AMPHENOL_10018783-10202TLF` |
| `J4` | `pBITzBus` | `Footprints:AMPHENOL_10018783-10202TLF` |
| `J5` | `pBITzBus` | `Footprints:AMPHENOL_10018783-10202TLF` |
| `J6` | `pBITzBus` | `Footprints:SAMTEC_PCIE-098-02-X-D-RA` |

## Power Architecture

`J1` is the ATX-24 input. The active design uses the ATX `+3.3V`, `+5V`,
`+5VSB`, `GND`, `PWR_OK`, and `PS_ON#` pins. The `+12V`, `-12V`, and ATX `NC`
pins are unconnected in the schematic netlist.

| ATX pin group | Backplane net | Use |
| --- | --- | --- |
| `+3.3V` pins 1, 2, 12, 13 | `+3V3` | Distributed to every pBITz slot on A9, A10, B9, B10; also J7 pin 2. |
| `+5V` pins 4, 6, 21, 22, 23 | `+5V` | Distributed to every pBITz slot on A1, A2, B1, B2; feeds J7 pin 1, LEDs, and resistor-network commons. |
| `+5VSB` pin 9 | `+5V_AON` | Always-on rail for the ATtiny power controller and its pull-ups/bulk caps. |
| Ground pins | `GND` | Board ground, pBITz slot grounds, J7 grounds, logic ground. |
| `PWR_OK` pin 8 | `/Power/PWR_OK` | ATtiny input on `U1.6/PB1`. |
| `PS_ON#` pin 16 | `/Power/~{PS_ON}` | Pulled down through `Q1` under ATtiny control. |

The power-control island:

| Ref | Part | Role |
| --- | --- | --- |
| `U1` | `ATtiny85-20P` | Standby-powered power supervisor/controller. |
| `Q1` | `BSS138` | Pull-down path for ATX `PS_ON#`. Source is `GND`, drain is `/Power/~{PS_ON}`, gate is driven by `U1.5/PB0`. |
| `SW1` | `SW_Push` | Power button input path to `U1.7/PB2` through `R1`. |
| `R5` | `10k` | Pull-up from ATtiny reset net to `+5V_AON`. |
| `R6` | `10k` | Pull-down from Q1 gate to `GND`. |
| `R7` | `10k` | Pull-up from `/Backplane/~{SHUTDOWN_RQ}` to `+5V_AON`. |
| `R8` | `10k` | Pull-up from `/Backplane/~{PWR_OFF}` to `+5V_AON`. |
| `D1`, `D2` | LED indicators | LED anodes on `+5V`; cathodes return through `R20` and `R19` respectively. |

ATtiny pin use:

| U1 pin | ATtiny function | Schematic net |
| ---: | --- | --- |
| 1 | `~{RESET}/PB5` | `Net-(U1-~{RESET}{slash}PB5)` |
| 2 | `XTAL1/PB3` | `/Backplane/~{SHUTDOWN_RQ}` |
| 3 | `XTAL2/PB4` | `/Backplane/~{PWR_OFF}` |
| 4 | `GND` | `GND` |
| 5 | `AREF/PB0` | `Net-(Q1-G)` |
| 6 | `PB1` | `/Power/PWR_OK` |
| 7 | `PB2` | `/Power/PWR_SW` |
| 8 | `VCC` | `+5V_AON` |

### Power Up/Down Protocol

The PMU is the always-on power-control island powered from `+5V_AON`. The IO
Controller participates in shutdown through the two active-low backplane power
control nets:

| Signal | Direction | Protocol | Electrical idle |
| --- | --- | --- | --- |
| `/Backplane/~{PWR_OFF}` | To PMU | Level signal. `HIGH` means unasserted and requests/permits `KEEP POWER ON`; `LOW` means asserted and requests `REMOVE POWER`. | Pulled high by `R8` to `+5V_AON`. |
| `/Backplane/~{SHUTDOWN_RQ}` | From PMU | Edge signal. The PMU uses the assertion edge to advise the IO Controller to clean up for shutdown. | Pulled high by `R7` to `+5V_AON`. |

Shutdown sequence:

1. During normal operation, `/Backplane/~{PWR_OFF}` remains high so the PMU
   keeps power on.
2. When the PMU wants an orderly shutdown, it signals the IO Controller on
   `/Backplane/~{SHUTDOWN_RQ}` with an edge event.
3. The IO Controller performs shutdown cleanup.
4. When cleanup is complete, the IO Controller asserts `/Backplane/~{PWR_OFF}`
   by driving it low.
5. The PMU treats low `/Backplane/~{PWR_OFF}` as the level request to remove
   power.

## pBITz Slot Bus

`J2` through `J6` share the same electrical pinout. The connectors carry a
24-bit address bus, 16-bit data bus, 20 control lines, four chip-select lines,
four clock nets, SPI/service nets, power, ground, and two power-control nets.

The current schematic names the SPI clock net `/Backplane/SLK_CLK`, while the
connector pin function is `SPI_CLK`. Treat `SLK_CLK` as the schematic net name.

| Pin index | A-side net | B-side net |
| ---: | --- | --- |
| 1 | `+5V` | `+5V` |
| 2 | `+5V` | `+5V` |
| 3 | `/Backplane/~{SHUTDOWN_RQ}` | `/Backplane/~{PWR_OFF}` |
| 4 | `GND` | `GND` |
| 5 | `/Backplane/CLK2` | `/Backplane/CLK0` |
| 6 | `/Backplane/CLK3` | `/Backplane/CLK1` |
| 7 | `/Backplane/SLK_CLK` | `/Backplane/SPI_CD` |
| 8 | `/Backplane/PROG` | `/Backplane/CART` |
| 9 | `+3V3` | `+3V3` |
| 10 | `+3V3` | `+3V3` |
| 11 | `/Backplane/CTRL_10` | `/Backplane/~{RESET}` |
| 12 | `/Backplane/CTRL_11` | `/Backplane/CTRL_1` |
| 13 | `/Backplane/CTRL_12` | `/Backplane/CTRL_2` |
| 14 | `/Backplane/CTRL_13` | `/Backplane/CTRL_3` |
| 15 | `/Backplane/CTRL_14` | `/Backplane/CTRL_4` |
| 16 | `/Backplane/CTRL_15` | `/Backplane/CTRL_5` |
| 17 | `/Backplane/CTRL_16` | `/Backplane/CTRL_6` |
| 18 | `/Backplane/CTRL_17` | `/Backplane/CTRL_7` |
| 19 | `/Backplane/CTRL_18` | `/Backplane/CTRL_8` |
| 20 | `/Backplane/CTRL_19` | `/Backplane/CTRL_9` |
| 21 | `/Backplane/CS2` | `/Backplane/CS0` |
| 22 | `/Backplane/CS3` | `/Backplane/CS1` |
| 23 | `/Backplane/D8` | `/Backplane/D0` |
| 24 | `/Backplane/D9` | `/Backplane/D1` |
| 25 | `/Backplane/D10` | `/Backplane/D2` |
| 26 | `/Backplane/D11` | `/Backplane/D3` |
| 27 | `/Backplane/D12` | `/Backplane/D4` |
| 28 | `/Backplane/D13` | `/Backplane/D5` |
| 29 | `/Backplane/D14` | `/Backplane/D6` |
| 30 | `/Backplane/D15` | `/Backplane/D7` |
| 31 | `GND` | `GND` |
| 32 | `/Backplane/A12` | `/Backplane/A0` |
| 33 | `/Backplane/A13` | `/Backplane/A1` |
| 34 | `/Backplane/A14` | `/Backplane/A2` |
| 35 | `GND` | `GND` |
| 36 | `/Backplane/A15` | `/Backplane/A3` |
| 37 | `/Backplane/A16` | `/Backplane/A4` |
| 38 | `/Backplane/A17` | `/Backplane/A5` |
| 39 | `GND` | `GND` |
| 40 | `/Backplane/A18` | `/Backplane/A6` |
| 41 | `/Backplane/A19` | `/Backplane/A7` |
| 42 | `/Backplane/A20` | `/Backplane/A8` |
| 43 | `GND` | `GND` |
| 44 | `/Backplane/A21` | `/Backplane/A9` |
| 45 | `/Backplane/A22` | `/Backplane/A10` |
| 46 | `/Backplane/A23` | `/Backplane/A11` |
| 47 | `/Backplane/~{SPI_INT}` | `GND` |
| 48 | `/Backplane/MISO` | `/Backplane/MOSI` |
| 49 | `/Backplane/SPI_CS0` | `/Backplane/SPI_CS1` |

Bus group summary:

| Group | Nets | Notes |
| --- | --- | --- |
| Address | `A0` through `A23` | Commoned across all five slots. |
| Data | `D0` through `D15` | Commoned across all five slots. |
| Control | `~{RESET}`, `CTRL_1` through `CTRL_19` | `CTRL_0` appears as the connector pin function on B11, with the schematic net named `~{RESET}`. |
| Chip select | `CS0` through `CS3` | Commoned across all five slots; generated elsewhere. |
| Clock | `CLK0` through `CLK3` | Passive slot-to-slot distribution in this active hierarchy. |
| Service SPI | `MOSI`, `MISO`, `SLK_CLK`, `SPI_CS0`, `SPI_CS1`, `SPI_CD`, `~{SPI_INT}` | Shared on all slots and exposed on J7. |
| Power control | `~{SHUTDOWN_RQ}`, `~{PWR_OFF}` | Common across slots and tied to the always-on ATtiny controller through pull-ups. |

The bus pull-up architecture is intentional: all Data, Address, and non-SPI
bus Control/select lines are pulled up. The slot clock nets and the SPI/service
nets are not pulled up by the bus resistor networks. In this context, D/A/C
pull-ups cover `A0`-`A23`, `D0`-`D15`, `~{RESET}`, `CTRL_1`-`CTRL_19`,
`CS0`-`CS3`, `CART`, and `PROG`.

## Service Header J7

`J7` exposes the SPI/service bus on a 2x10 IDC header. The schematic netlist
maps it as follows:

| J7 pin | Net |
| ---: | --- |
| 1 | `+5V` |
| 2 | `+3V3` |
| 3 | `/Backplane/MOSI` |
| 4 | `/Backplane/SPI_CS0` |
| 5 | `/Backplane/MISO` |
| 6 | `GND` |
| 7 | `GND` |
| 8 | `/Backplane/SPI_CS1` |
| 9 | `/Backplane/SLK_CLK` |
| 10 | `GND` |
| 11 | `GND` |
| 12 | `unconnected-(J7-Pin_12-Pad12)` |
| 13 | `/Backplane/~{SPI_INT}` |
| 14 | `GND` |
| 15 | `GND` |
| 16 | `GND` |
| 17 | `/Backplane/SPI_CD` |
| 18 | `GND` |
| 19 | `GND` |
| 20 | `GND` |

## Bus Pull-Up Networks

The backplane uses four 4.7k Bourns resistor-network packages with their common
pins tied to `+5V`. These implement the D/A/C pull-up rule: every data net,
every address net, and every non-SPI/non-clock bus control/select net is pulled
up through `RN1`-`RN4`.

| Ref | Part | Common pin | Covered nets |
| --- | --- | --- | --- |
| `RN1` | `4820P-2-472LF` | pin 20 to `+5V` | `CART`, `~{RESET}`, `CTRL_1` through `CTRL_8`, `CTRL_10` through `CTRL_17`, `PROG` |
| `RN2` | `4820P-2-472LF` | pin 20 to `+5V` | `A4` through `A11`, `A15` through `A23`; pads 10 and 11 unconnected |
| `RN3` | `4816P-T02-472LF` | pin 16 to `+5V` | `A0` through `A3`, `A12` through `A14`, `D4` through `D7`, `D12` through `D15` |
| `RN4` | `4816P-T02-472LF` | pin 16 to `+5V` | `CTRL_9`, `CTRL_18`, `CTRL_19`, `CS0` through `CS3`, `D0` through `D3`, `D8` through `D11` |

Notably, slot clock nets `CLK0`-`CLK3` and SPI/service nets `MOSI`, `MISO`,
`SLK_CLK`, `SPI_CS0`, `SPI_CS1`, `SPI_CD`, and `~{SPI_INT}` are not on these
pull-up networks.

## Decoupling And Bulk Capacitance

The schematic places both bulk and local decoupling around the rails:

| Rail | Capacitors |
| --- | --- |
| `+5V_AON` | `C1`, `C6`, `C7` |
| `+5V` | `C2`, `C3`, `C12`, `C13`, `C14`, `C15`, `C17` |
| `+3V3` | `C4`, `C5`, `C8`, `C9`, `C10`, `C11`, `C16` |

The `100n` capacitors sit near slot/logic rail locations, while the `47u` and
`100u` capacitors provide bulk storage on the power rails.

## Architectural Conclusions

The board is a shared passive bus backplane with a small always-on power
controller. It does not decode addresses, buffer the CPU bus, or generate the
slot chip selects locally in the active schematic hierarchy. Slot cards see the
same address, data, control, select, clock, service SPI, rail, and ground nets
on `J2` through `J6`.

Power sequencing is centralized in the ATtiny island: `+5VSB` powers `U1`, `U1`
observes the ATX `PWR_OK` and the local power button path, can pull ATX
`PS_ON#` low through `Q1`, and also connects to the backplane-wide
`~{SHUTDOWN_RQ}` and `~{PWR_OFF}` lines. Those two lines are pulled up to
`+5V_AON`, making them available even before the main `+5V` and `+3V3` rails are
enabled.

The service header is a direct breakout of the shared SPI/service nets plus
`+5V`, `+3V3`, and ground. The pBITz connector pinout is therefore the main
system contract: CPU, memory, I/O, and peripheral cards are expected to agree on
the shared bus semantics documented in the pinout table above.
