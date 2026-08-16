# pBITz Backplane Architecture

This document describes the current architecture and physical signal distribution of the `pBITzBackplane` KiCad project.

The source of truth for connectivity is the schematic hierarchy rooted at [`pBITzBackplane.kicad_sch`](pBITzBackplane.kicad_sch). This document explains that design in human-readable form; if this document and the current KiCad sources disagree, the **KiCad schematic wins**.

The logical meaning of the portable `CTRL_*` and `CS[3:0]` conventions is documented separately in [`../pBITz_Control_Signals.md`](../pBITz_Control_Signals.md). This file is intentionally the place where the **physical backplane connectors and their pin mappings** are documented.

---

## 1. Schematic hierarchy

The active design is divided into three sheets:

| Sheet | File | Role |
| --- | --- | --- |
| Root | `pBITzBackplane.kicad_sch` | Top-level hierarchy and board-level interconnect. |
| Power | `Power.kicad_sch` | ATX input, standby-powered PMU, power switch, indicators, and `PS_ON#` control. |
| Backplane | `Backplane.kicad_sch` | DIN expansion connectors, cartridge connector, service header, bus pull-ups, and slot decoupling. |

The backplane is deliberately simple. Apart from the power-management island it is primarily a passive distribution board; CPU-specific decode, timing adaptation, and bus semantics live on the CPU board or its associated glue logic.

---

## 2. Physical connector architecture

The current backplane has **two different expansion connector types** serving different mechanical roles:

| Ref | Role | Connector | Notes |
| --- | --- | --- | --- |
| `J8` | pBITz expansion slot | DIN 41612, 96-position | Amphenol `86093968114T55F1LF`; right-angle backplane connector. |
| `J9` | pBITz expansion slot | DIN 41612, 96-position | Same electrical pinout as J8. |
| `J10` | pBITz expansion slot | DIN 41612, 96-position | Same electrical pinout as J8. |
| `J11` | pBITz expansion slot | DIN 41612, 96-position | Same electrical pinout as J8. |
| `J6` | Cartridge slot | PCIe x8-style, 98-contact | Samtec `PCIE-098-02-X-D-RA`; right-angle connector. **Mechanical connector only: this is not PCI Express.** |
| `J7` | Platform service header | 2x10 IDC | Exposes power and the SPI/service interface to the backplane mezzanine. |

The four ordinary expansion positions therefore use the DIN connector introduced with the current pBITz mechanical architecture, while the cartridge slot deliberately retains the compact right-angle PCIe x8 connector.

### Important: the cartridge connector is not PCI Express

`J6` uses a connector mechanically associated with PCI Express, but its electrical interface is the **pBITz parallel bus**. A PCI Express card must never be connected to it, and a pBITz cartridge must never be inserted into a PC PCIe slot.

The cartridge connector is now part of the **main backplane**, not the USB/SD mezzanine. It receives the same current pBITz bus presented to the DIN expansion slots, including the general clock lines.

---

## 3. pBITz bus carried by the backplane

The current backplane distributes the following logical groups:

| Group | Signals | Notes |
| --- | --- | --- |
| Address | `A0..A23` | 24-bit parallel address bus. |
| Data | `D0..D15` | 16-bit parallel data bus. |
| Control | `CTRL_0..CTRL_19` | Cross-CPU control vocabulary; see `pBITz_Control_Signals.md`. |
| Device select | `CS0..CS3` | Push-pull 4-bit device address `CS[3:0]`; `0000` means no device selected. |
| General clocks | `CLK0`, `CLK1` | Two current general-purpose pBITz clock lines. Passive distribution only. |
| Auxiliary control | `/PROG`, `/CART` | Platform/machine auxiliary control lines. |
| Service SPI | `SPI_CLK`, `MOSI`, `MISO`, `SPI_CS0`, `SPI_CS1`, `/SPI_INT`, `/SPI_CD` | Shared platform service interface. `SPI_CLK` is separate from the general pBITz clock lines. |
| Power management | `/PWR_OFF`, `/SHUTDOWN_RQ` | PMU/IO-controller shutdown handshake. |
| Power | `+5V`, `+3V3`, `GND` | Distributed to expansion and cartridge connectors. |

### Clock count

The **current bus has two general clock lines: `CLK0` and `CLK1`**.

Older pBITz connector symbols and documentation contained additional `CLK2`/`CLK3` names. They are not active clocks in the current backplane. On the instantiated cartridge connector `J6`, the legacy A5/A6 positions associated with those old symbol names are explicitly unconnected; `CLK0` and `CLK1` are carried on J6 B5 and B6.

`SPI_CLK` is the clock of the separate service SPI interface and is not counted as a general pBITz CPU/peripheral clock.

The backplane does not synthesize or condition `CLK0` or `CLK1`; it only distributes them. Their source and meaning are defined by the machine/CPU implementation.

---

## 4. DIN 41612 expansion-slot pinout

`J8` through `J11` are electrically identical 96-position DIN 41612 pBITz slots. The connector uses rows A, B, and C with positions 1 through 32.

| Pos. | Row A | Row B | Row C |
| ---: | --- | --- | --- |
| 1 | `+5V` | `+5V` | `/PWR_OFF` |
| 2 | `+5V` | `+5V` | `/SHUTDOWN_RQ` |
| 3 | `+3V3` | `+3V3` | `GND` |
| 4 | `+3V3` | `+3V3` | `GND` |
| 5 | `CLK0` | `CLK1` | `SPI_CLK` |
| 6 | `/PROG` | `/CART` | `/SPI_CD` |
| 7 | `CTRL_0` (`/RESET`) | `CTRL_1` | `CTRL_2` |
| 8 | `CTRL_3` | `CTRL_4` | `CTRL_5` |
| 9 | `CTRL_6` | `CTRL_7` | `CTRL_8` |
| 10 | `CTRL_9` | `CTRL_10` | `CTRL_11` |
| 11 | `CTRL_12` | `CTRL_13` | `CTRL_14` |
| 12 | `CTRL_15` | `CTRL_16` | `CTRL_17` |
| 13 | `CTRL_18` | `CTRL_19` | `CS0` |
| 14 | `CS1` | `CS2` | `CS3` |
| 15 | `D0` | `D1` | `D2` |
| 16 | `D3` | `D4` | `D5` |
| 17 | `D6` | `D7` | `D8` |
| 18 | `D9` | `D10` | `D11` |
| 19 | `D12` | `D13` | `D14` |
| 20 | `D15` | `GND` | `GND` |
| 21 | `A0` | `A1` | `A2` |
| 22 | `A3` | `A4` | `A5` |
| 23 | `A6` | `A7` | `A8` |
| 24 | `A9` | `A10` | `A11` |
| 25 | `A12` | `A13` | `A14` |
| 26 | `A15` | `A16` | `A17` |
| 27 | `A18` | `A19` | `A20` |
| 28 | `A21` | `A22` | `A23` |
| 29 | `GND` | `GND` | `GND` |
| 30 | `/SPI_INT` | `MISO` | `MOSI` |
| 31 | `SPI_CS0` | `SPI_CS1` | `GND` |
| 32 | `GND` | `GND` | `GND` |

The connector shield/mechanical contacts `S1` and `S2` are also tied to `GND`.

The DIN layout deliberately groups related signals while inserting ground positions through the connector, particularly around the address and service portions of the bus.

---

## 5. Cartridge slot J6

`J6` is the dedicated cartridge connector. It uses the right-angle Samtec `PCIE-098-02-X-D-RA`, a 98-contact connector with A and B sides numbered 1 through 49.

Although the mechanical format is PCIe x8, **the entire electrical assignment is custom pBITz**.

### 5.1 Cartridge bus coverage

The cartridge connector exposes the full current pBITz interface needed by a cartridge to behave like a directly attached bus device:

- all `A0..A23` address lines;
- all `D0..D15` data lines;
- all `CTRL_0..CTRL_19` control lines;
- all four `CS[3:0]` device-select lines;
- both current general clocks, `CLK0` and `CLK1`;
- `/PROG` and `/CART`;
- the complete SPI/service interface;
- `/PWR_OFF` and `/SHUTDOWN_RQ`;
- `+5V`, `+3V3`, and ground.

The cartridge slot is therefore not a reduced sideband interface. Electrically it is another endpoint on the pBITz bus, with a different mechanical connector chosen for cartridge use.

### 5.2 Cartridge physical pinout

| Pos. | Side A | Side B |
| ---: | --- | --- |
| 1 | `+5V` | `+5V` |
| 2 | `+5V` | `+5V` |
| 3 | `/SHUTDOWN_RQ` | `/PWR_OFF` |
| 4 | `GND` | `GND` |
| 5 | **NC** (legacy `CLK2` position) | `CLK0` |
| 6 | **NC** (legacy `CLK3` position) | `CLK1` |
| 7 | `SPI_CLK` | `/SPI_CD` |
| 8 | `/PROG` | `/CART` |
| 9 | `+3V3` | `+3V3` |
| 10 | `+3V3` | `+3V3` |
| 11 | `CTRL_10` | `CTRL_0` (`/RESET`) |
| 12 | `CTRL_11` | `CTRL_1` |
| 13 | `CTRL_12` | `CTRL_2` |
| 14 | `CTRL_13` | `CTRL_3` |
| 15 | `CTRL_14` | `CTRL_4` |
| 16 | `CTRL_15` | `CTRL_5` |
| 17 | `CTRL_16` | `CTRL_6` |
| 18 | `CTRL_17` | `CTRL_7` |
| 19 | `CTRL_18` | `CTRL_8` |
| 20 | `CTRL_19` | `CTRL_9` |
| 21 | `CS2` | `CS0` |
| 22 | `CS3` | `CS1` |
| 23 | `D8` | `D0` |
| 24 | `D9` | `D1` |
| 25 | `D10` | `D2` |
| 26 | `D11` | `D3` |
| 27 | `D12` | `D4` |
| 28 | `D13` | `D5` |
| 29 | `D14` | `D6` |
| 30 | `D15` | `D7` |
| 31 | `GND` | `GND` |
| 32 | `A12` | `A0` |
| 33 | `A13` | `A1` |
| 34 | `A14` | `A2` |
| 35 | `GND` | `GND` |
| 36 | `A15` | `A3` |
| 37 | `A16` | `A4` |
| 38 | `A17` | `A5` |
| 39 | `GND` | `GND` |
| 40 | `A18` | `A6` |
| 41 | `A19` | `A7` |
| 42 | `A20` | `A8` |
| 43 | `GND` | `GND` |
| 44 | `A21` | `A9` |
| 45 | `A22` | `A10` |
| 46 | `A23` | `A11` |
| 47 | `/SPI_INT` | `GND` |
| 48 | `MISO` | `MOSI` |
| 49 | `SPI_CS0` | `SPI_CS1` |

The two explicit NC entries at A5 and A6 are intentional in the current design. They are remnants of the older connector symbol's larger clock allocation and must not be interpreted as active `CLK2`/`CLK3` signals.

---

## 6. Device selection and control semantics

The backplane transports the four `CS0..CS3` lines without decoding them. Collectively they form the push-pull 4-bit pBITz device address `CS[3:0]`:

- `0000` means **no pBITz device selected**;
- `0001` through `1111` select device addresses 1 through 15.

The CPU-side I/O decoder/MMU determines that a valid peripheral access is occurring and drives the device address. The selected card normally compares that value with a configured device number and then uses normal address lines for register or sub-device decode.

The meanings and CPU-family mappings of `CTRL_0..CTRL_19` are intentionally not duplicated here. See [`../pBITz_Control_Signals.md`](../pBITz_Control_Signals.md).

---

## 7. Backplane pull-ups and idle state

The backplane defines the idle state of the ordinary parallel bus with pull-up resistor networks.

The following shared bus groups are pulled high by the backplane:

- `A0..A23`;
- `D0..D15`;
- the parallel `CTRL_*` controls, including reset;
- `CS0..CS3`;
- the non-clock auxiliary parallel controls such as `/PROG` and `/CART`.

The current design uses 4.7 kΩ bussed resistor networks (`4614X-101-472LF`) for these parallel-bus pull-ups.

The general clock lines and SPI/service bus are **not** part of this pull-up network:

- `CLK0` and `CLK1` are not pulled up by the general bus networks;
- `SPI_CLK`, `MOSI`, `MISO`, `SPI_CS0`, `SPI_CS1`, `/SPI_INT`, and `/SPI_CD` are not treated as ordinary pulled-up parallel bus lines.

The power-management handshake lines have their own electrical rules and are described separately below.

This distinction is important for card design: cards can assume a defined high idle state on the normal address/data/control bus, but must not assume the same biasing on clocks or the SPI/service interface.

---

## 8. Platform service header J7

`J7` is a 2x10 IDC header (`SBH11-PBPC-D10-ST-BK`) used to connect the backplane service interface to the USB/SD mezzanine.

| J7 pin | Signal |
| ---: | --- |
| 1 | `+5V` |
| 2 | `+3V3` |
| 3 | `MOSI` |
| 4 | `SPI_CS0` |
| 5 | `MISO` |
| 6 | `GND` |
| 7 | `GND` |
| 8 | `SPI_CS1` |
| 9 | `SPI_CLK` |
| 10 | `GND` |
| 11 | `GND` |
| 12 | NC |
| 13 | `/SPI_INT` |
| 14 | `GND` |
| 15 | `GND` |
| 16 | `GND` |
| 17 | `/SPI_CD` |
| 18 | `GND` |
| 19 | `GND` |
| 20 | `GND` |

The service SPI bus is a platform facility and is separate from CPU-family-specific parallel bus timing. The mezzanine currently uses this interface for shared USB-host and SD-card services.

---

## 9. Power architecture

### 9.1 ATX input

`J1` is the 24-pin ATX power connector. The backplane uses:

- `+3.3V` to create the distributed `+3V3` rail;
- `+5V` to create the distributed `+5V` rail;
- `+5VSB` as the always-on `+5V_AON` rail for the PMU;
- `GND`;
- `PWR_OK`; and
- `PS_ON#`.

The pBITz bus does not distribute the ATX `+12V` or `-12V` rails in the current design.

### 9.2 PMU

The backplane contains a small always-on power-management island built around:

- `U1`, an `ATtiny85-20P` powered from `+5V_AON`;
- `Q1`, a `BSS138` used to pull ATX `PS_ON#` low;
- the enclosure power-switch input;
- `PWR_OK` sensing; and
- the `/PWR_OFF` / `/SHUTDOWN_RQ` handshake with the machine's IO Controller.

This is the only significant active logic on the backplane itself.

### 9.3 Shutdown handshake

The intended active-low power-control protocol is:

| Signal | Role | Electrical behavior |
| --- | --- | --- |
| `/PWR_OFF` | Request to the PMU to remove main power | Pulled up to `+5V_AON` on the backplane. Requesters assert by driving low and otherwise release the line. |
| `/SHUTDOWN_RQ` | PMU request for orderly machine shutdown | PMU asserts low/release; the IO Controller provides the operating-side pull-up. |

Typical orderly shutdown sequence:

1. `/PWR_OFF` is high during normal operation.
2. The PMU asserts `/SHUTDOWN_RQ` to ask the IO Controller to perform shutdown cleanup.
3. The IO Controller completes pending work and prepares the machine for power removal.
4. The IO Controller asserts `/PWR_OFF` low.
5. The PMU removes main ATX power through `PS_ON#` control.

The exact PMU state machine is implemented in [`../Code/PMU/`](../Code/PMU/).

---

## 10. Architectural boundaries

The backplane deliberately does **not** define:

- the CPU's memory map;
- where the pBITz peripheral aperture lives;
- which physical CPU signal directly corresponds to every `CTRL_*` line;
- clock frequencies carried on `CLK0` or `CLK1`;
- interrupt encoding inside a particular CPU family;
- wait-state translation for a particular CPU; or
- which device number is assigned to a particular peripheral card.

Those decisions belong to the CPU board and machine architecture. The backplane provides the common electrical vocabulary and mechanical infrastructure on which those machines are built.

---

## 11. Source-of-truth and revision policy

For the current hardware revision:

1. the KiCad schematic hierarchy rooted at `pBITzBackplane.kicad_sch` is authoritative for connectivity;
2. this architecture document describes the intent and physical pin mapping of that schematic;
3. [`../pBITz_Control_Signals.md`](../pBITz_Control_Signals.md) is authoritative for the portable logical meanings of `CTRL_*` and `CS[3:0]`;
4. PCB, BOM, Gerber, validation, and production files should always be checked against the revision from which they were generated.

Useful repo-relative review commands include:

```sh
python3 Code/tools/kicad_nettrace.py pBITzBackplane/pBITzBackplane.kicad_sch --check
python3 Code/tools/kicad_nettrace.py pBITzBackplane/pBITzBackplane.kicad_sch --list-nets
```

When helper tooling and KiCad disagree, the current KiCad schematic/netlist is the source of truth.
