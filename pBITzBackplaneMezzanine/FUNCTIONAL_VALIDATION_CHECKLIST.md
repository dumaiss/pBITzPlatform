# pBITzBackplaneMezzanine Functional Validation Checklist

This checklist validates the `pBITzBackplaneMezzanine` design files for
reasonable functional assurance before fabrication or bring-up. The goal is to
catch obvious gotchas that would stop the board from powering on or acting as
the intended SPI-attached USB/SD mezzanine, not to prove certification readiness
or remove all possible risk.

Out of scope:

- Mechanical fit, dimensions, connector keying, mounting, and enclosure fit.
- DRC/ERC closure.
- Power-budget/current-capacity analysis.
- Firmware/software behavior for MAX3421E, SD-card protocol, or host drivers.
- Physical-board continuity, powered voltage measurements, and bench bring-up.

Authority rule: the schematic and KiCad schematic netlist are authoritative. If
`kicad_nettrace.py` disagrees with the schematic, the schematic wins.

## 1. Source And Artifact Setup

- [ ] Confirm the active root schematic is `pBITzBackplaneMezzanine.kicad_sch`.
- [ ] Confirm the active hierarchy contains only `USBPorts.kicad_sch`, `MezzanineConnector.kicad_sch`, and `SD_Card.kicad_sch`.
- [ ] Export a fresh KiCad schematic netlist:

```sh
HOME=/tmp XDG_CONFIG_HOME=/tmp XDG_DATA_HOME=/tmp kicad-cli sch export netlist -o /tmp/pbitzbackplane-mezzanine-kicad.net pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch
```

- [ ] Run the hierarchy tracer as a secondary cross-check:

```sh
python3 Code/tools/kicad_nettrace.py pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch
python3 Code/tools/kicad_nettrace.py pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch --check
```

- [ ] Confirm the tracer reports no output-driver conflicts.
- [ ] Record tracer-reported opens that are resolved by the KiCad netlist as tracer limitations, not design failures.

## 2. Mezzanine Connector J5

Validate J5 as the board interface for rails, ground, SPI, interrupt, and SD
card-detect.

- [ ] Confirm `J5.1` is `+5V`.
- [ ] Confirm `J5.2` is `+3V3`.
- [ ] Confirm `J5.3` is `/MezzanineConnector/MOSI`.
- [ ] Confirm `J5.4` is `/MezzanineConnector/SPI_CS0`.
- [ ] Confirm `J5.5` is `/MezzanineConnector/MISO`.
- [ ] Confirm `J5.8` is `/MezzanineConnector/SPI_CS1`.
- [ ] Confirm `J5.9` is `/MezzanineConnector/SPI_CLK`.
- [ ] Confirm `J5.13` is `/MezzanineConnector/~{SPI_INT}`.
- [ ] Confirm `J5.17` is `/MezzanineConnector/~{SD_PRESENT}`.
- [ ] Confirm `J5.12` is intentionally unconnected.
- [ ] Confirm J5 ground pins 6, 7, 10, 11, 14, 15, 16, 18, 19, and 20 are `GND`.

Pass criteria: J5 maps exactly to the interface table in `ARCHITECTURE.md`.

## 3. Rail And Ground Distribution

Validate rail presence and support connections, not current capacity.

- [ ] Confirm `+5V` enters at `J5.1`.
- [ ] Confirm `+5V` powers FE1.1 `U1.18/VBUSM` and `U1.20/VDD5`.
- [ ] Confirm `+5V` feeds the four downstream USB VBUS fuses `F1` through `F4`.
- [ ] Confirm `+3V3` enters at `J5.2`.
- [ ] Confirm `+3V3` powers MAX3421E `U2.2/VL`, `U2.23/VCC`, and reset pin `U2.12/~{RES}`.
- [ ] Confirm `+3V3` powers U3/U4 `VCC` pins and SD socket `J4.4/VDD`.
- [ ] Confirm FE1.1 `U1.21/VD33F` generates `+3V3_INTERNAL`.
- [ ] Confirm `+3V3_INTERNAL` is used for FE1.1 support pull-ups and `U1.26/OVCJ`.
- [ ] Confirm all IC ground pins, SD ground pins, connector shields, and J5 ground pins connect to `GND`.
- [ ] Confirm `+5V`, `+3V3`, `+3V3_INTERNAL`, and `GND` are not shorted together in the schematic netlist.

Pass criteria: every required supply and ground pin is on the documented net,
and rail names remain distinct except through intended component paths.

## 4. SPI Signal Routing

Validate the host/backplane SPI wiring and local fanout.

- [ ] Confirm `MOSI` from `J5.3` reaches `U3.3` and `U4.9`.
- [ ] Confirm `U3` buffers MOSI to `/USBPorts/MOSI` and that `/USBPorts/MOSI` reaches `U2.16/MOSI`.
- [ ] Confirm `U4` buffers MOSI to `/SD_Card/MOSI` and that `/SD_Card/MOSI` reaches `J4.2/CMD`.
- [ ] Confirm `SPI_CLK` from `J5.9` reaches `U3.7` and `U4.14`.
- [ ] Confirm `U3` buffers SPI clock to `/USBPorts/CLK` and that `/USBPorts/CLK` reaches `U2.13/SCLK`.
- [ ] Confirm `U4` buffers SPI clock to `/SD_Card/CLK` and that `/SD_Card/CLK` reaches `J4.5/CLK`.
- [ ] Confirm `SPI_CS0` from `J5.4` is buffered by `U4` to `/SD_Card/~{CS}` and reaches `J4.1/DAT3/CD`.
- [ ] Confirm `SPI_CS1` from `J5.8` is buffered by `U3` to `/USBPorts/~{CS}` and reaches `U2.14/~{SS}`.
- [ ] Confirm shared MISO connects `J5.5`, `U2.15/MISO`, `J4.7/DAT0`, and `R6.2`.
- [ ] Confirm `R6` pulls shared MISO to `+3V3`.
- [ ] Confirm `U2.18/INT` reaches `J5.13`.
- [ ] Confirm `J4.CD` reaches `J5.17`.

Pass criteria: every SPI and status signal reaches the intended device pin and
no chip-select or clock line is swapped between USB and SD.

## 5. USB Host Controller And Hub

Validate MAX3421E and FE1.1 support wiring.

- [ ] Confirm MAX3421E `U2.21/D+` reaches FE1.1 `U1.16/DPU` through `R4`.
- [ ] Confirm MAX3421E `U2.20/D-` reaches FE1.1 `U1.15/DMU` through `R5`.
- [ ] Confirm `R4` and `R5` are 33 ohm series resistors.
- [ ] Confirm `U2` has a 12 MHz crystal on `XI`/`XO` and 22 pF load capacitors to ground.
- [ ] Confirm `U1` has a 12 MHz crystal on `XIN`/`XOUT` and 22 pF load capacitors to ground.
- [ ] Confirm FE1.1 `BUSJ` and `XRSTJ` are pulled to `+3V3_INTERNAL`.
- [ ] Confirm FE1.1 `REXT` has a 2.7k resistor to `GND`.
- [ ] Confirm FE1.1 `OVCJ` is held inactive by `+3V3_INTERNAL`.
- [ ] Confirm FE1.1 `DRV`, `LED1`, `LED2`, `PWRJ`, and `TESTJ` are intentionally unused or no-connected.

Pass criteria: MAX3421E can electrically talk to the FE1.1 upstream port, and
the FE1.1 support pins match the intended simplified hub design.

## 6. USB Downstream Connectors

Validate downstream VBUS and USB 2.0 D+/D- polarity.

- [ ] Confirm `F2` feeds `/USBPorts/VBUS_PIN1` to `J1.1`.
- [ ] Confirm `F1` feeds `/USBPorts/VBUS_PIN2` to `J1.10`.
- [ ] Confirm `F3` feeds `/USBPorts/VBUS_PIN3` to `J2.1`.
- [ ] Confirm `F4` feeds `/USBPorts/VBUS_PIN4` to `J3.1`.
- [ ] Confirm FE1.1 `DM1` reaches USB-A port 1 `D1-` at `J1.2`.
- [ ] Confirm FE1.1 `DP1` reaches USB-A port 1 `D1+` at `J1.3`.
- [ ] Confirm FE1.1 `DM2` reaches USB-A port 2 `D2-` at `J1.11`.
- [ ] Confirm FE1.1 `DP2` reaches USB-A port 2 `D2+` at `J1.12`.
- [ ] Confirm auxiliary header `J2` carries port 3 as `VBUS`, `DM3`, `GND`, `DP3`.
- [ ] Confirm auxiliary header `J3` carries port 4 as `VBUS`, `DM4`, `GND`, `DP4`.
- [ ] Confirm USB3 SuperSpeed pins on `J1` are intentionally unconnected.
- [ ] Confirm connector shields/drains connect to `GND`.

Pass criteria: each downstream USB 2.0 port has fused VBUS, ground, and correct
D+/D- polarity.

## 7. SD-Card Interface

Validate the SD socket in SPI mode.

- [ ] Confirm `J4.1/DAT3/CD` is `/SD_Card/~{CS}`.
- [ ] Confirm `J4.2/CMD` is `/SD_Card/MOSI`.
- [ ] Confirm `J4.5/CLK` is `/SD_Card/CLK`.
- [ ] Confirm `J4.7/DAT0` is `/MezzanineConnector/MISO`.
- [ ] Confirm `J4.4/VDD` is `+3V3`.
- [ ] Confirm `J4.3`, `J4.6`, `J4.G1`, and `J4.G2` are `GND`.
- [ ] Confirm `J4.CD` is `/MezzanineConnector/~{SD_PRESENT}`.
- [ ] Confirm `J4.8/DAT1`, `J4.9/DAT2`, and `J4.WP` are intentionally unconnected.
- [ ] Confirm SD input signals are driven from U4 3.3 V buffer outputs.

Pass criteria: the socket supports SPI-mode SD access and card detect without
unexpected extra SD-bus dependencies.

## 8. Level-Shifter Unused Gates

Validate unused 4050 channels do not float.

- [ ] Confirm U3 unused inputs are tied to `GND`.
- [ ] Confirm U3 unused outputs are intentionally unconnected.
- [ ] Confirm U4 unused inputs are tied to `GND`.
- [ ] Confirm U4 unused outputs are intentionally unconnected.

Pass criteria: no unused CMOS input is floating.

## 9. Intentional Opens And No-Connects

Validate that unconnected pins are intentional and limited to expected functions.

- [ ] Confirm `J5.12` is unconnected.
- [ ] Confirm USB3 SuperSpeed pins on `J1` are unconnected.
- [ ] Confirm SD `DAT1`, `DAT2`, and `WP` are unconnected.
- [ ] Confirm unused FE1.1 pins are no-connected as documented in `ARCHITECTURE.md`.
- [ ] Confirm unused MAX3421E GPIO/auxiliary pins are no-connected as documented in `ARCHITECTURE.md`.
- [ ] Confirm unused 4050 outputs are no-connected.
- [ ] Review MAX3421E `U2.22/VBCOMP`; if intentionally unused, record that decision, because the local MAX3421E datasheet recommends a bypass capacitor to ground.

Pass criteria: no functional connector, power, ground, USB 2.0, SPI, interrupt,
or SD-card pin is unexpectedly open.

## 10. PCB Pad-Net Agreement

Validate the PCB file agrees with the schematic netlist for functional nets.

- [ ] Confirm J5 schematic nets match J5 PCB pad nets.
- [ ] Confirm J1, J2, and J3 schematic nets match PCB pad nets.
- [ ] Confirm J4 schematic nets match PCB pad nets.
- [ ] Confirm U1, U2, U3, and U4 schematic nets match PCB pad nets.
- [ ] Confirm F1-F4 and R1-R6 schematic nets match PCB pad nets.
- [ ] Confirm crystal and load-capacitor schematic nets match PCB pad nets.

Pass criteria: no key functional component has schematic/PCB net assignment
drift.

## 11. Gotcha Review

Review the design intent at the system level after detailed connectivity checks.

- [ ] Confirm the module has no local voltage regulator for the main `+5V` or `+3V3` rails; those rails must arrive through J5.
- [ ] Confirm MAX3421E reset is held high and any reset behavior must be managed by power cycling or SPI chip reset.
- [ ] Confirm the shared MISO topology depends on inactive SPI devices tri-stating their outputs.
- [ ] Confirm the USB hub's downstream USB-A D+/D- polarity matches connector pin functions, not only net labels.
- [ ] Confirm auxiliary USB header pin order is documented for the intended harness or mating board.
- [ ] Confirm the SD-card detect polarity expected by the host/backplane matches the socket's `CD` behavior.
- [ ] Confirm all tracer limitations are resolved against the KiCad schematic netlist before accepting the result.

Pass criteria: no unresolved design-file issue remains that would obviously
prevent the mezzanine from receiving power, communicating over SPI, operating
the USB hub/controller path, or operating the SD-card path.

## 12. Final Acceptance

- [ ] Schematic hierarchy matches the intended active design.
- [ ] KiCad schematic netlist agrees with the architecture document.
- [ ] `kicad_nettrace.py` reports no output-driver conflicts.
- [ ] J5 interface mapping passes.
- [ ] Rail and ground distribution passes.
- [ ] SPI fanout and status routing passes.
- [ ] USB host-controller and hub support wiring passes.
- [ ] USB downstream VBUS and D+/D- polarity passes.
- [ ] SD-card interface passes.
- [ ] Intentional opens are limited to the documented list.
- [ ] PCB pad-net agreement passes for key functional parts.
- [ ] Gotcha review finds no unresolved obvious functional risk.

Functional validation passes when every applicable item above is checked and
all deviations are either corrected in the design or explicitly recorded as
accepted design changes.
