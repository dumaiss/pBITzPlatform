# pBITzBackplaneMezzanine Functional Validation Report

Date: 2026-06-17

Result: PASS for the workspace-only functional validation scope, with one
nonblocking advisory.

The current design files provide reasonable assurance, within the stated
workspace-only scope, that the mezzanine should receive power and support the
intended SPI-attached USB and SD-card functions. The stacked USB-A connector
polarity was explicitly checked against the connector pin functions and passes
in the current schematic and PCB.

## Scope

In scope:

- Active schematic hierarchy.
- KiCad schematic netlist connectivity.
- `kicad_nettrace.py` secondary connectivity cross-check.
- Selected PCB pad-net agreement for functional connectors and active parts.
- J5 mezzanine connector mapping.
- Rail and ground distribution in the design files.
- SPI fanout and status signal routing.
- USB host-controller/hub connectivity.
- SD-card SPI-mode connectivity.
- Intentional opens/no-connects.

Out of scope:

- Physical-board continuity checks.
- Powered rail measurements.
- Bench bring-up.
- Mechanical validation.
- DRC/ERC closure.
- Power-budget/current-capacity validation.
- Firmware/software behavior.

Authority rule: the schematic and KiCad schematic netlist are authoritative.
`kicad_nettrace.py` is used only as a secondary check.

## Artifacts Used

| Artifact | Purpose |
| --- | --- |
| `pBITzBackplaneMezzanine.kicad_sch` | Root schematic and active hierarchy. |
| `USBPorts.kicad_sch` | USB host controller, hub, USB connectors, fuses, crystals, support parts. |
| `MezzanineConnector.kicad_sch` | J5 mezzanine connector and mounting-hole pads. |
| `SD_Card.kicad_sch` | SD-card socket and SD signal buffering. |
| `pBITzBackplaneMezzanine.kicad_pcb` | PCB pad-net cross-check for functional nets. |
| `Code/tools/kicad_nettrace.py` | Secondary hierarchy/connectivity check. |
| `/tmp/pbitzbackplane-mezzanine-kicad.net` | KiCad schematic netlist exported for validation. |
| `00-Datasheets/Peripheral/max3421e.pdf` | Local reference for MAX3421E pin behavior. |
| `00-Datasheets/Peripheral/fe1.1s应用电路.pdf` | Local FE1.1 reference circuit used for D+/D- convention. |
| `ARCHITECTURE.md` | Design intent reference. |
| `FUNCTIONAL_VALIDATION_CHECKLIST.md` | Validation checklist. |

## Commands Run

```sh
HOME=/tmp XDG_CONFIG_HOME=/tmp XDG_DATA_HOME=/tmp kicad-cli sch export netlist -o /tmp/pbitzbackplane-mezzanine-kicad.net pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch
python3 Code/tools/kicad_nettrace.py pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch
python3 Code/tools/kicad_nettrace.py pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch --check
```

Additional temporary Python checks parsed the KiCad schematic netlist and PCB
file to verify the functional pin/net expectations in the checklist.

During this pass, `Code/tools/kicad_nettrace.py` was patched so `_resolve()`
checks a relative path as supplied before joining it to the root schematic
directory. This fixes the tool when called as:

```sh
python3 Code/tools/kicad_nettrace.py pBITzBackplaneMezzanine/pBITzBackplaneMezzanine.kicad_sch
```

## Summary Results

| Area | Result | Evidence |
| --- | --- | --- |
| Active hierarchy | PASS | Exported netlist contains only `/`, `/USBPorts/`, `/SD_Card/`, and `/MezzanineConnector/`. |
| Nettrace coverage | PASS | Root 0/0, USBPorts 136/173, MezzanineConnector 27/28, SD_Card 31/37 pin hits. |
| Nettrace conflicts | PASS | No driver-driver conflicts reported. |
| Schematic/PCB functional checks | PASS | 494 checks run: 494 passed, 0 failed. |
| J5 interface | PASS | J5 maps to `+5V`, `+3V3`, SPI, interrupt, SD-present, and ground as documented. |
| Rails and ground | PASS | `+5V`, `+3V3`, `+3V3_INTERNAL`, and `GND` are distributed to expected pins. |
| SPI fanout | PASS | MOSI, clock, CS0, CS1, MISO, USB interrupt, and SD-present paths match the architecture. |
| USB upstream path | PASS | MAX3421E D+/D- connect through 33 ohm resistors to FE1.1 upstream DPU/DMU. |
| USB downstream J1 polarity | PASS | FE1.1 `DM1/DM2` nets land on USB-A `D-` pins, and `DP1/DP2` nets land on USB-A `D+` pins. |
| SD-card interface | PASS | SPI-mode SD socket wiring and card-detect path match the architecture. |
| Intentional opens | PASS with advisory | Expected no-connects are documented; review `U2.22/VBCOMP` before fabrication. |
| Hardware/powered checks | OUT OF SCOPE | Requires physical board or powered bench setup. |

## Key Gotcha Check

### 1. USB-A D+/D- Polarity Passes On J1

The FE1.1 hub nets use the conventional names `DMx` for D- and `DPx` for D+.
The local FE1.1 reference circuit follows that convention. In the current
mezzanine design, both stacked USB-A ports on `J1` match that convention:

| Port | Connector pin | Connector function | Current net | Expected net |
| --- | ---: | --- | --- | --- |
| 1 | `J1.2` | `D1-` | `/USBPorts/DM1` | `/USBPorts/DM1` |
| 1 | `J1.3` | `D1+` | `/USBPorts/DP1` | `/USBPorts/DP1` |
| 2 | `J1.11` | `D2-` | `/USBPorts/DM2` | `/USBPorts/DM2` |
| 2 | `J1.12` | `D2+` | `/USBPorts/DP2` | `/USBPorts/DP2` |

The PCB has the same assignments as the schematic.

## Advisory Finding

### 2. Review MAX3421E `VBCOMP` No-Connect

`U2.22/VBCOMP` is intentionally marked no-connect in the schematic netlist. The
local MAX3421E datasheet describes `VBCOMP` as the VBUS comparator input and
recommends bypassing it to ground with a 1.0 uF ceramic capacitor. If this
design never uses the MAX3421E VBUS comparator, the no-connect may be an
intentional simplification; however, it should be reviewed before fabrication.

This was not counted as a blocking validation failure because the workspace
design files mark it as intentional and the main MAX3421E USB data path does
not depend on it in the schematic connectivity.

## Checklist Results

| Checklist section | Result | Notes |
| --- | --- | --- |
| 1. Source And Artifact Setup | PASS | KiCad netlist export succeeded; active hierarchy matched expected sheets; nettrace found no driver-driver conflicts. |
| 2. Mezzanine Connector J5 | PASS | J5 pinout matches `ARCHITECTURE.md`. |
| 3. Rail And Ground Distribution | PASS | Rails and grounds route to expected functional pins in schematic and PCB. |
| 4. SPI Signal Routing | PASS | USB and SD chip-selects, MOSI, clock, MISO, interrupt, and SD-present paths are correct. |
| 5. USB Host Controller And Hub | PASS | MAX3421E-to-FE1.1 upstream path and FE1.1 support pins match the intended simplified hub design. |
| 6. USB Downstream Connectors | PASS | J1 stacked USB-A D+/D- polarity passes on ports 1 and 2. J2/J3 auxiliary headers match the documented board nets. |
| 7. SD-Card Interface | PASS | SD socket is wired for SPI mode with card detect and expected no-connects. |
| 8. Level-Shifter Unused Gates | PASS | Unused 4050 inputs are tied to `GND`; paired outputs are no-connected. |
| 9. Intentional Opens And No-Connects | PASS with advisory | Expected no-connects only; `VBCOMP` should be reviewed. |
| 10. PCB Pad-Net Agreement | PASS | Schematic and PCB agree for key functional parts, including the J1 polarity-sensitive USB nets. |
| 11. Gotcha Review | PASS with advisory | No blocking design-file issue remains; review `VBCOMP` no-connect before fabrication. |
| 12. Final Acceptance | PASS with advisory | Current workspace-only validation passes with the `VBCOMP` review note. |

## Expected Opens

The KiCad schematic netlist contains these expected functional opens:

| Ref pins | Notes |
| --- | --- |
| `J5.12` | Unused mezzanine connector pin. |
| `J1.5`, `J1.6`, `J1.8`, `J1.9`, `J1.14`, `J1.15`, `J1.17`, `J1.18` | USB3 SuperSpeed pins unused. |
| `J4.8`, `J4.9`, `J4.WP` | SD `DAT1`, `DAT2`, and write-protect unused. |
| `U1.12`, `U1.13`, `U1.22`, `U1.23`, `U1.24`, `U1.25`, `U1.27`, `U1.28` | FE1.1 unused pins. |
| `U2.1`, `U2.4`-`U2.11`, `U2.17`, `U2.22`, `U2.26`-`U2.32` | MAX3421E unused GPIO/auxiliary pins. |
| `U3.10`, `U3.12`, `U3.15`, `U4.2`, `U4.4`, `U4.6` | Unused 4050 outputs. |

## Nettrace Notes

`kicad_nettrace.py --check` reported isolated pins that the KiCad schematic
netlist resolves correctly, including J5 power pins, U3/U4 power pins, fuse
input pins, and SD-card `VDD`. These are treated as tracer limitations because
the schematic netlist is the authority and the PCB pad-net cross-check agrees
with it.

The useful nettrace result is that it found no output-driver conflicts.

## Functional Assessment

Within the workspace-only scope, the design files provide reasonable assurance
for these areas:

- J5 receives and distributes the expected `+5V`, `+3V3`, `GND`, SPI, interrupt,
  and SD-present nets.
- MAX3421E SPI signals are buffered and routed to the expected pins.
- MAX3421E USB D+/D- connect correctly to the FE1.1 upstream port through 33
  ohm series resistors.
- FE1.1 power/support wiring is present.
- Downstream USB VBUS rails are individually fused.
- SD-card SPI-mode wiring, power, ground, and card-detect routing are present.
- Unused 4050 gates are not left with floating inputs.
- Schematic and PCB pad nets agree for the key functional components.

No unresolved design-file issue was found that would obviously prevent the
mezzanine from powering on or acting as the intended SPI-attached USB/SD
module, assuming the out-of-scope DRC/ERC, power budget, manufacturing, and
physical bring-up assumptions remain valid.
