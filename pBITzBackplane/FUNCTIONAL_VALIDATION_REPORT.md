# pBITzBackplane Functional Validation Report

Date: 2026-06-17

Result: PASS for the workspace-only functional validation scope.

This report validates the `pBITzBackplane` design files for reasonable
assurance that the board should power on and behave as the intended passive
shared bus backplane with an always-on ATtiny power-control island. The goal is
to catch obvious schematic/netlist/PCB gotchas, not to certify the design or
prove hardware behavior on a fabricated board.

## Scope

In scope:

- Active schematic hierarchy.
- KiCad schematic netlist connectivity.
- `kicad_nettrace.py` secondary connectivity cross-check.
- Selected PCB pad-net agreement for functional connectors and control parts.
- Slot-to-slot bus consistency.
- Rail and ground distribution in the design files.
- Pull-up network coverage.
- J7 service-header mapping.
- ATtiny power-control connectivity.
- Intentional opens/no-connects.

Out of scope:

- Physical-board continuity checks.
- Powered rail measurements.
- Bench bring-up.
- Mechanical validation.
- DRC/ERC closure.
- Power-budget/current-capacity validation.
- Firmware/software behavior.

## Artifacts Used

| Artifact | Purpose |
| --- | --- |
| `pBITzBackplane.kicad_sch` | Root schematic and active hierarchy. |
| `Power.kicad_sch` | ATX and always-on power-control circuitry. |
| `Backplane.kicad_sch` | Slot connectors, service header, pull-ups, and rail decoupling. |
| `pBITzBackplane.kicad_pcb` | PCB pad-net cross-check for functional nets. |
| `Code/tools/kicad_nettrace.py` | Secondary hierarchy/connectivity check. |
| `/tmp/pbitzbackplane-kicad.net` | KiCad schematic netlist exported for validation. |
| `ARCHITECTURE.md` | Design intent reference. |
| `FUNCTIONAL_VALIDATION_CHECKLIST.md` | Validation checklist. |

Authority rule: the schematic and KiCad schematic netlist are authoritative.
`kicad_nettrace.py` is used only as a secondary check.

## Commands Run

```sh
HOME=/tmp XDG_CONFIG_HOME=/tmp XDG_DATA_HOME=/tmp kicad-cli sch export netlist -o /tmp/pbitzbackplane-kicad.net pBITzBackplane/pBITzBackplane.kicad_sch
python3 Code/tools/kicad_nettrace.py /home/kitamura/Documents/pBITzPlatform/pBITzBackplane/pBITzBackplane.kicad_sch
python3 Code/tools/kicad_nettrace.py /home/kitamura/Documents/pBITzPlatform/pBITzBackplane/pBITzBackplane.kicad_sch --check
```

Additional temporary Python checks parsed the KiCad schematic netlist and PCB
file to verify the functional pin/net expectations in the checklist.

## Summary Results

| Area | Result | Evidence |
| --- | --- | --- |
| Active hierarchy | PASS | Exported netlist contains only `/`, `/Power/`, and `/Backplane/`. |
| Nettrace coverage | PASS | Root 2/2, Power 75/79, Backplane 613/616 pin hits. |
| Nettrace conflicts | PASS | No driver-driver conflicts reported. |
| Schematic-netlist functional checks | PASS | 712 checks passed, 0 failed. |
| Selected PCB pad-net checks | PASS | 546 checks passed, 0 failed. |
| D/A/C pull-up rule | PASS | 66 expected D/A/C pull-up nets found, none missing; no SPI/service or slot clock nets pulled up through `RN1`-`RN4`. |
| Intentional opens | PASS | Only expected connector/resistor-network opens were found. |
| Hardware/powered checks | OUT OF SCOPE | Requires physical board or powered bench setup. |

## Checklist Results

| Checklist section | Result | Notes |
| --- | --- | --- |
| 1. Source And Artifact Setup | PASS | KiCad netlist export succeeded; active hierarchy matched expected sheets; nettrace found no output-driver conflicts. |
| 2. Backplane Slot Connectivity | PASS | `J2` through `J6` all match the expected 98-pin pBITz slot map. Address, data, control, chip-select, clock, service, rail, ground, and power-control nets are commoned as intended. |
| 3. Bus Isolation | PASS | The schematic netlist shows unrelated bus, service, control, rail, and ground nets remain separate, except for intentional component paths such as resistor-network pull-ups. |
| 4. Slot Power Rail Distribution | PASS | ATX `+5V`, `+3V3`, `+5V_AON`, and `GND` map to the expected slot, J7, and control-circuit pins in the design files. |
| 5. Pull-Up Network Function | PASS | `RN1`, `RN2`, `RN3`, and `RN4` implement the D/A/C pull-up rule: all address, data, and non-SPI/non-clock bus control/select nets are pulled up to `+5V`; SPI/service and slot clock nets are not pulled up through the resistor networks. |
| 6. Service Header J7 | PASS | J7 maps to `+5V`, `+3V3`, `GND`, and the shared SPI/service nets exactly as documented; `J7.12` is intentionally unconnected. |
| 7. Power-Control Signal Paths | PASS | `U1`, `Q1`, `J1`, `SW1`, `R1`, `R5`, `R6`, `R7`, and `R8` connect to the intended power-control nets. Firmware behavior is not evaluated. |
| 8. Intentional Opens And No-Connects | PASS | Expected opens only: ATX `+12V`, `-12V`, `NC`; `J7.12`; `RN2.10`; `RN2.11`. |
| 9. Gotcha Review | PASS | No unresolved design-file issue was found that obviously prevents rail distribution, shared bus distribution, service-header breakout, pull-up behavior, or power-control connectivity. |
| 10. Final Acceptance | PASS | All workspace-only validation items pass. |

## Expected Opens

The KiCad schematic netlist contains these expected connector/resistor-network
opens:

| Ref.pin | Net |
| --- | --- |
| `J1.10` | `unconnected-(J1-+12V-Pad10)` |
| `J1.11` | `unconnected-(J1-+12V-Pad10)` |
| `J1.14` | `unconnected-(J1--12V-Pad14)` |
| `J1.20` | `unconnected-(J1-NC-Pad20)` |
| `J7.12` | `unconnected-(J7-Pin_12-Pad12)` |
| `RN2.10` | `unconnected-(RN2-Pad10)` |
| `RN2.11` | `unconnected-(RN2-Pad11)` |

No pBITz slot connector pins were unconnected.

## Nettrace Notes

`kicad_nettrace.py --check` reported isolated pins on several nets that the
KiCad schematic netlist resolves correctly, including global power/common pins
such as `J7.1`, `J7.2`, resistor-network common pins, and `+5V_AON` pull-up
nodes. These are treated as tracer limitations because the schematic netlist is
the authority and the PCB pad-net cross-check agrees with it.

The useful nettrace result is that it found no output-driver conflicts.

## Functional Assessment

Within the workspace-only scope, the design files provide reasonable assurance
that:

- The ATX connector distributes the intended `+5V`, `+3V3`, `+5V_AON`, and
  `GND` nets.
- Every pBITz slot sees the same address, data, control, chip-select, clock,
  SPI/service, power-control, rail, and ground nets.
- J7 is a direct breakout of the intended service SPI nets plus rails and
  ground.
- The resistor networks pull up the intended bus/control nets and do not
  accidentally pull up the SPI/service or slot clock nets.
- The ATtiny power-control island is electrically connected to `PWR_OK`,
  `PS_ON#`, the power button path, `~{SHUTDOWN_RQ}`, and `~{PWR_OFF}`.
- The only connector/resistor-network opens are expected and documented.

The D/A/C pull-up validation found all 66 expected resistor-network pull-up
nets: `A0`-`A23`, `D0`-`D15`, `~{RESET}`, `CTRL_1`-`CTRL_19`, `CS0`-`CS3`,
`CART`, and `PROG`. No SPI/service nets or slot clock nets were found on the
bus pull-up networks.

No obvious design-file gotcha was found that would prevent the board from
powering on or acting as the intended passive backplane, assuming the out-of-
scope DRC/ERC, power budget, manufacturing, and physical bring-up assumptions
remain valid.
