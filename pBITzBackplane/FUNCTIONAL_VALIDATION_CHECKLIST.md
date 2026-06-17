# pBITzBackplane Functional Validation Checklist

This checklist validates the `pBITzBackplane` design files for reasonable
functional assurance before fabrication or bring-up. The goal is to catch
obvious gotchas that would stop the board from powering on or acting as the
intended shared bus backplane, not to prove certification readiness or remove
all possible risk.

Out of scope:

- Mechanical fit, dimensions, connector keying, mounting, and enclosure fit.
- DRC/ERC closure.
- Power-budget/current-capacity analysis.
- ATtiny firmware behavior and any external software behavior.
- Physical-board continuity, powered voltage measurements, and bench bring-up.

Authority rule: the schematic and KiCad schematic netlist are authoritative. If
`kicad_nettrace.py` disagrees with the schematic, the schematic wins.

## 1. Source And Artifact Setup

- [ ] Confirm the active root schematic is `pBITzBackplane.kicad_sch`.
- [ ] Confirm only `Power.kicad_sch` and `Backplane.kicad_sch` are in the active hierarchy.
- [ ] Confirm `ClockTree.kicad_sch` is not used for this validation.
- [ ] Export a fresh KiCad schematic netlist:

```sh
HOME=/tmp XDG_CONFIG_HOME=/tmp XDG_DATA_HOME=/tmp kicad-cli sch export netlist -o /tmp/pbitzbackplane-kicad.net pBITzBackplane/pBITzBackplane.kicad_sch
```

- [ ] Run the hierarchy tracer as a secondary cross-check:

```sh
python3 Code/tools/kicad_nettrace.py /home/kitamura/Documents/pBITzPlatform/pBITzBackplane/pBITzBackplane.kicad_sch
python3 Code/tools/kicad_nettrace.py /home/kitamura/Documents/pBITzPlatform/pBITzBackplane/pBITzBackplane.kicad_sch --check
```

- [ ] Confirm the tracer reports no output-driver conflicts.
- [ ] Record any tracer-reported opens that are resolved by the KiCad netlist as tracer limitations, not design failures.

## 2. Backplane Slot Connectivity

Validate `J2` through `J6` as electrically parallel pBITz slots.

- [ ] For every slot connector `J2`, `J3`, `J4`, `J5`, and `J6`, confirm pins A1-A49 and B1-B49 match the pinout in `ARCHITECTURE.md`.
- [ ] Pick one connector as the reference slot, preferably `J2`.
- [ ] Confirm each address net is commoned across all five slots: `A0` through `A23`.
- [ ] Confirm each data net is commoned across all five slots: `D0` through `D15`.
- [ ] Confirm each control net is commoned across all five slots: `~{RESET}` and `CTRL_1` through `CTRL_19`.
- [ ] Confirm each chip-select net is commoned across all five slots: `CS0`, `CS1`, `CS2`, `CS3`.
- [ ] Confirm each slot clock net is commoned across all five slots: `CLK0`, `CLK1`, `CLK2`, `CLK3`.
- [ ] Confirm each SPI/service net is commoned across all five slots: `MOSI`, `MISO`, `SLK_CLK`, `SPI_CS0`, `SPI_CS1`, `SPI_CD`, `~{SPI_INT}`.
- [ ] Confirm the power-control nets are commoned across all five slots: `~{SHUTDOWN_RQ}` and `~{PWR_OFF}`.
- [ ] Confirm every slot has the same `+5V`, `+3V3`, and `GND` pin distribution.

Pass criteria: every shared bus net appears on all expected slot pins, and no
slot has a swapped, missing, or extra connection relative to the schematic
pinout.

## 3. Bus Isolation

Validate that the backplane behaves as a shared bus without unintended shorts
between unrelated nets.

- [ ] Confirm no address net is shorted to another address net.
- [ ] Confirm no data net is shorted to another data net.
- [ ] Confirm no control net is shorted to another control net.
- [ ] Confirm no chip-select net is shorted to another chip-select net.
- [ ] Confirm no SPI/service net is shorted to another SPI/service net.
- [ ] Confirm no address, data, control, chip-select, clock, or SPI/service net is shorted to `+5V`, `+3V3`, `+5V_AON`, or `GND`, except through intentional pull-ups or component paths.
- [ ] Confirm `+5V`, `+3V3`, and `+5V_AON` are not shorted to each other.
- [ ] Confirm `+5V`, `+3V3`, and `+5V_AON` are not shorted to `GND`.

Pass criteria: unrelated nets are isolated. Nets with resistor-network pull-ups
may show resistance to `+5V`; they must not read as direct shorts.

## 4. Slot Power Rail Distribution

Validate rail presence and distribution, not current capacity.

- [ ] Confirm ATX `J1` pins 4, 6, 21, 22, and 23 connect to schematic net `+5V`.
- [ ] Confirm ATX `J1` pins 1, 2, 12, and 13 connect to schematic net `+3V3`.
- [ ] Confirm ATX `J1` pin 9 connects to schematic net `+5V_AON`.
- [ ] Confirm all ATX ground pins connect to schematic net `GND`.
- [ ] Confirm every slot has `+5V` on A1, A2, B1, and B2.
- [ ] Confirm every slot has `+3V3` on A9, A10, B9, and B10.
- [ ] Confirm every slot has `GND` on A4, A31, A35, A39, A43, B4, B31, B35, B39, B43, and B47.
- [ ] Confirm `J7.1` is `+5V`.
- [ ] Confirm `J7.2` is `+3V3`.
- [ ] Confirm J7 ground pins 6, 7, 10, 11, 14, 15, 16, 18, 19, and 20 connect to `GND`.

Pass criteria: rail and ground distribution matches the schematic netlist.

## 5. Pull-Up Network Function

Validate that the resistor networks implement the D/A/C pull-up architecture:
all Data, Address, and non-SPI/non-clock bus Control/select lines are pulled up
to `+5V`; SPI/service and slot clock nets are not.

- [ ] Confirm `RN1.20` is `+5V`.
- [ ] Confirm `RN1` covers `CART`, `~{RESET}`, `CTRL_1` through `CTRL_8`, `CTRL_10` through `CTRL_17`, and `PROG`.
- [ ] Confirm `RN2.20` is `+5V`.
- [ ] Confirm `RN2` covers `A4` through `A11` and `A15` through `A23`.
- [ ] Confirm `RN2.10` and `RN2.11` are intentionally unconnected.
- [ ] Confirm `RN3.16` is `+5V`.
- [ ] Confirm `RN3` covers `A0` through `A3`, `A12` through `A14`, `D4` through `D7`, and `D12` through `D15`.
- [ ] Confirm `RN4.16` is `+5V`.
- [ ] Confirm `RN4` covers `CTRL_9`, `CTRL_18`, `CTRL_19`, `CS0` through `CS3`, `D0` through `D3`, and `D8` through `D11`.
- [ ] Confirm all address nets `A0` through `A23` are pulled up through `RN1`-`RN4`.
- [ ] Confirm all data nets `D0` through `D15` are pulled up through `RN1`-`RN4`.
- [ ] Confirm bus control/select nets `~{RESET}`, `CTRL_1` through `CTRL_19`, `CS0` through `CS3`, `CART`, and `PROG` are pulled up through `RN1`-`RN4`.
- [ ] Confirm slot clock nets `CLK0` through `CLK3` are not tied into `RN1`-`RN4`.
- [ ] Confirm SPI/service nets `MOSI`, `MISO`, `SLK_CLK`, `SPI_CS0`, `SPI_CS1`, `SPI_CD`, and `~{SPI_INT}` are not tied into `RN1`-`RN4`.

Pass criteria: every D/A/C bus net has a resistor path to `+5V`, and no
SPI/service or slot clock net is pulled up through these networks.

## 6. Service Header J7

Validate J7 as a direct breakout of power, ground, and shared SPI/service nets.

- [ ] Confirm `J7.1` is `+5V`.
- [ ] Confirm `J7.2` is `+3V3`.
- [ ] Confirm `J7.3` is `/Backplane/MOSI`.
- [ ] Confirm `J7.4` is `/Backplane/SPI_CS0`.
- [ ] Confirm `J7.5` is `/Backplane/MISO`.
- [ ] Confirm `J7.8` is `/Backplane/SPI_CS1`.
- [ ] Confirm `J7.9` is `/Backplane/SLK_CLK`.
- [ ] Confirm `J7.13` is `/Backplane/~{SPI_INT}`.
- [ ] Confirm `J7.17` is `/Backplane/SPI_CD`.
- [ ] Confirm `J7.12` is intentionally unconnected.
- [ ] Confirm all J7 ground pins match the schematic netlist.
- [ ] Confirm each J7 SPI/service signal is commoned to the matching pins on `J2` through `J6`.

Pass criteria: J7 is a faithful service breakout and does not introduce any
slot-only or header-only variant of the SPI/service nets.

## 7. Power-Control Signal Paths

Validate electrical connectivity of the always-on power-control island without
validating ATtiny firmware behavior.

- [ ] Confirm `+5V_AON` powers `U1.8`.
- [ ] Confirm `U1.4` connects to `GND`.
- [ ] Confirm `J1.8` ATX `PWR_OK` connects to `U1.6/PB1`.
- [ ] Confirm `U1.5/PB0` connects to `Q1.1` gate.
- [ ] Confirm `Q1.2` source connects to `GND`.
- [ ] Confirm `Q1.3` drain connects to `J1.16` ATX `PS_ON#`.
- [ ] Confirm `R6` pulls the Q1 gate net to `GND`.
- [ ] Confirm `SW1` and `R1` form the schematic-defined `PWR_SW` input path to `U1.7/PB2`.
- [ ] Confirm `R5` pulls the ATtiny reset net to `+5V_AON`.
- [ ] Confirm `R7` pulls `/Backplane/~{SHUTDOWN_RQ}` to `+5V_AON`.
- [ ] Confirm `R8` pulls `/Backplane/~{PWR_OFF}` to `+5V_AON`.
- [ ] Confirm `/Backplane/~{SHUTDOWN_RQ}` appears on every slot A3 and at `U1.2/PB3`.
- [ ] Confirm `/Backplane/~{PWR_OFF}` appears on every slot B3 and at `U1.3/PB4`.

Pass criteria: the ATtiny can electrically observe `PWR_OK`, observe the power
button path, drive the MOSFET gate net, and connect to the two backplane
power-control nets. Firmware behavior is not evaluated here.

## 8. Intentional Opens And No-Connects

Validate that unconnected pins are intentional and limited to the expected list.

- [ ] Confirm ATX `J1.10` and `J1.11` `+12V` are unconnected.
- [ ] Confirm ATX `J1.14` `-12V` is unconnected.
- [ ] Confirm ATX `J1.20` `NC` is unconnected.
- [ ] Confirm `J7.12` is unconnected.
- [ ] Confirm `RN2.10` and `RN2.11` are unconnected.
- [ ] Confirm no additional connector pins or resistor-network pins are unconnected in the KiCad schematic netlist.

Pass criteria: all opens match the schematic netlist and no unexpected open
exists on a functional bus, rail, ground, service, or power-control signal.

## 9. Gotcha Review

Review the design intent at the system level after the detailed connectivity
checks pass.

- [ ] Confirm the board has exactly one active logic island: the always-on ATtiny power-control circuit.
- [ ] Confirm the main pBITz bus is passive and shared across `J2` through `J6`.
- [ ] Confirm the design does not depend on any active local address decode, buffering, or slot-select logic on the backplane.
- [ ] Confirm every net required for a card to see the bus is present on every slot: power, ground, address, data, control, chip select, clock, SPI/service, and power-control nets.
- [ ] Confirm all D/A/C bus lines are pulled up and that pull-ups are not accidentally applied to SPI/service or slot clock nets.
- [ ] Confirm the always-on power-control path can electrically connect ATX `PWR_OK`, `PS_ON#`, power button input, `~{SHUTDOWN_RQ}`, and `~{PWR_OFF}` to the ATtiny pins.
- [ ] Confirm all known unconnected pins are intentional and documented.
- [ ] Confirm any tracer limitations are resolved against the schematic netlist before accepting the result.

Pass criteria: no unresolved design-file issue remains that would obviously
prevent the backplane from distributing rails, ground, slot bus signals,
service signals, or power-control signals as intended.

## 10. Final Acceptance

- [ ] Schematic hierarchy matches the intended active design.
- [ ] KiCad schematic netlist agrees with the architecture document.
- [ ] `kicad_nettrace.py` reports no output-driver conflicts.
- [ ] Slot-to-slot continuity passes for all shared bus nets.
- [ ] Isolation checks pass for unrelated nets and rails.
- [ ] J7 maps exactly to the documented service header pinout.
- [ ] Pull-up networks cover exactly the documented nets.
- [ ] Power-control connectivity passes without requiring firmware validation.
- [ ] Intentional opens are limited to the documented list.
- [ ] Gotcha review finds no unresolved obvious functional risk.

Functional validation passes when every applicable item above is checked and
all deviations are either corrected in the design or explicitly recorded as
accepted design changes.
