# HD63C03Y VGA Terminal — Preliminary BOM

**Revision:** Rev A draft
**Board:** 100 × 100 mm, 4 layers
**Supply:** Regulated +5 V
**Display target:** 80 × 30 characters, 8 × 16 glyphs, approximately 640 × 480 at 60 Hz
**Host interfaces:** RS-232 fitted; RS-422/pBITz interface DNP
**Keyboard:** PS/2
**Terminal target:** VT100-compatible firmware with ANSI color extensions

## 1. Architecture summary

* Hitachi HD63C03Y CPU at 3 MHz
* Motorola MC6845B CRTC at 3 MHz
* 24.000 MHz master video clock
* 14.7456 MHz baud-rate reference
* 32 KiB firmware EEPROM
* 512 KiB banked SRAM
* 8 KiB programmable character EEPROM
* Two independently mapped 16 KiB CPU SRAM windows
* One independently selected 16 KiB video bank
* Four-slot shared-SRAM cycle:

  1. Character fetch
  2. Primary attribute fetch
  3. Secondary attribute fetch
  4. CPU access
* Multiple ATF22V10 PLDs for clocks, decode, arbitration, baud generation, and video attributes

## 2. CPU memory map

| Address     | Function                                       |
| ----------- | ---------------------------------------------- |
| `0000–0027` | HD63C03Y internal registers                    |
| `0028–003F` | Unused/reserved external space                 |
| `0040–013F` | HD63C03Y internal 256-byte RAM                 |
| `0140–3FFF` | SRAM window A, selected by `BANK_A`            |
| `4000–7FEF` | SRAM window B, selected by `BANK_B`            |
| `7FF0–7FFF` | External control-register block                |
| `8000–FFFF` | AT28C256 firmware EEPROM and interrupt vectors |

Although the internal register/RAM area obscures part of logical window A, no physical SRAM is lost: the same physical bank can be mapped into window B to access offsets `0000–013F`.

### External register block

| Address     | Register              | Access     |
| ----------- | --------------------- | ---------- |
| `7FF0`      | MC6845 register index | Write      |
| `7FF1`      | MC6845 register data  | Read/write |
| `7FF2`      | SRAM bank A, bits 0–4 | Write      |
| `7FF3`      | SRAM bank B, bits 0–4 | Write      |
| `7FF4`      | Video bank/control    | Write      |
| `7FF5–7FFF` | Reserved              | —          |

Suggested `VIDEO_BANK/CONTROL` format:

| Bit | Function                   |
| --: | -------------------------- |
| 0–4 | Physical 16 KiB video bank |
|   5 | Global font-bank select    |
|   6 | Video enable               |
|   7 | Reserved                   |

## 3. Video-bank layout

Each video bank is 16 KiB:

| Offset      |  Size | Function                                    |
| ----------- | ----: | ------------------------------------------- |
| `0000–0FFF` | 4 KiB | Character cells                             |
| `1000–1FFF` | 4 KiB | Primary attributes                          |
| `2000–2FFF` | 4 KiB | Secondary attributes                        |
| `3000–3FFF` | 4 KiB | Spare: line metadata, tab state, future use |

An 80 × 30 display requires 2,400 entries in each plane, leaving spare space for circular scrolling.

### Suggested attributes

Primary attribute byte:

| Bits | Function       |
| ---- | -------------- |
| 0–2  | Foreground RGB |
| 3–5  | Background RGB |
| 6    | Bright/bold    |
| 7    | Blink          |

Secondary attribute byte:

| Bit | Function            |
| --: | ------------------- |
|   0 | Underline           |
|   1 | Protected field     |
|   2 | Application-defined |
|   3 | Application-defined |
| 4–7 | Reserved            |

Reverse video and conceal can be encoded when the character is written by swapping or replacing foreground/background colors.

## 4. Character EEPROM layout

The AT28C64B contains two programmable 256-character fonts:

| Address     | Function                          |
| ----------- | --------------------------------- |
| `0000–0FFF` | Font bank 0: 256 × 16-byte glyphs |
| `1000–1FFF` | Font bank 1: alternate font       |

Address assignment:

```text
A0–A3   = MC6845 raster address RA0–RA3
A4–A11  = character code
A12      = global font-bank select
```

DEC Special Graphics characters can be placed in unused positions of the 256-character font and selected by firmware character translation.

## 5. Main integrated circuits

| Ref. | Qty | Selected part/function                 | Package | Source       |
| ---- | --: | -------------------------------------- | ------- | ------------ |
| U1   |   1 | HD63C03YCP CPU                         | PLCC-68 | BUY          |
| U2   |   1 | MC6845B CRTC                           | DIP-40  | STASH        |
| U3   |   1 | AT28C256-15PU firmware EEPROM, 32 KiB  | DIP-28  | STASH        |
| U4   |   1 | AS6C4008-55PIN SRAM, 512 KiB           | DIP-32  | STASH        |
| U5   |   1 | AT28C64B-15PU character EEPROM, 8 KiB  | DIP-28  | STASH        |
| U6   |   1 | ATF22V10C-10SU system decode PLD       | SOIC-24 | STASH—VERIFY |
| U7   |   1 | ATF22V10C-10SU SRAM arbitration PLD    | SOIC-24 | STASH—VERIFY |
| U8   |   1 | ATF22V10C-10SU master-clock/timing PLD | SOIC-24 | STASH—VERIFY |
| U9   |   1 | ATF22V10C-10SU baud-rate PLD           | SOIC-24 | STASH—VERIFY |
| U10  |   1 | ATF22V10C-10SU attribute/RGB PLD       | SOIC-24 | STASH—VERIFY |

### Intended PLD responsibilities

**U6 — SYS**

* EEPROM chip enable
* MC6845 select
* external register decode
* bank-latch write strobes
* video-control write strobe
* readback qualification

**U7 — ARB**

* shared-SRAM CPU/video ownership
* `MR` wait generation
* SRAM `/OE` and `/WE`
* SRAM/data-transceiver enable
* bank-latch output enables
* CPU/video address-mux select

**U8 — CLOCK**

* 24 MHz phase counter
* 12 MHz HD63C03 EXTAL output
* 3 MHz MC6845 character clock
* character/attribute fetch strobes
* character-boundary pipeline load

**U9 — BAUD**

* programmable division of 14.7456 MHz
* external SCI clocks for:

  * 115200 baud
  * 57600 baud
  * 38400 baud
  * 19200 baud
  * 9600 baud

**U10 — VIDEO**

* foreground/background selection
* bold/highlight
* blink
* underline
* cursor rendering
* blanking
* six RGB intensity outputs

## 6. SRAM banking and video pipeline

| Ref.    | Qty | Part/function                               | Package          | Source       |
| ------- | --: | ------------------------------------------- | ---------------- | ------------ |
| U11–U13 |   3 | SN74HC373 bank/control latches              | TSSOP-20         | STASH—VERIFY |
| U14     |   1 | SN74HC373 character staging latch           | TSSOP-20         | STASH—VERIFY |
| U15     |   1 | SN74HC373 primary-attribute staging latch   | TSSOP-20         | STASH—VERIFY |
| U16     |   1 | SN74HC373 secondary-attribute staging latch | TSSOP-20         | STASH—VERIFY |
| U17     |   1 | SN74HC373 active primary-attribute latch    | TSSOP-20         | STASH—VERIFY |
| U18     |   1 | SN74HC273 pipeline/control latch            | SOIC-20          | STASH—VERIFY |
| U19–U22 |   4 | 74AHCT157 quad 2:1 address multiplexers     | TSSOP-16/SOIC-16 | BUY          |
| U23     |   1 | 74HCT245 bidirectional SRAM bus transceiver | TSSOP-20/SOIC-20 | BUY          |
| U24     |   1 | CD74HC299 universal shift register          | SOIC-20          | STASH—VERIFY |
| U25     |   1 | SN74HC244 VGA output buffer                 | SOIC-20          | STASH—VERIFY |
| U26     |   1 | 74HC00 sync inversion/miscellaneous glue    | SOIC-14          | STASH        |

### Bank-latch arrangement

`BANK_A`, `BANK_B`, and `VIDEO_BANK` latch outputs share SRAM physical address lines `A14–A18`.

Only one latch output is enabled at a time:

```text
CPU access in window A → enable BANK_A
CPU access in window B → enable BANK_B
Video fetch            → enable VIDEO_BANK
```

The PLD must guarantee mutually exclusive, break-before-make output enables.

### Shared-SRAM timing

At 24 MHz, one character period contains eight pixel clocks.

```text
Pixel clocks 0–1: character-plane SRAM read
Pixel clocks 2–3: primary-attribute SRAM read
Pixel clocks 4–5: secondary-attribute SRAM read
Pixel clocks 6–7: CPU SRAM access
```

The CPU `MR` input aligns and stretches SRAM accesses when required. EEPROM and internal-register accesses do not require SRAM arbitration.

## 7. Clock generation

| Ref. | Qty | Part/function                                       | Package         | Source |
| ---- | --: | --------------------------------------------------- | --------------- | ------ |
| X1   |   1 | 24.000 MHz, 5 V HCMOS oscillator                    | SMD             | BUY    |
| X2   |   1 | ECS-2100AX-147.4, 14.7456 MHz oscillator            | Through-hole XO | STASH  |
| Cx   |   2 | 100 nF oscillator decoupling                        | 0603            | STASH  |
| Rx   |   2 | 10 kΩ oscillator-enable pull resistors, if required | 0603            | STASH  |

Clock tree:

```text
24.000 MHz
 ├── direct → pixel shift clock
 ├── ÷2     → 12 MHz HD63C03 EXTAL
 └── ÷8     → 3 MHz MC6845 character clock

14.7456 MHz
 └── programmable divider → HD63C03 P22/SCLK
```

## 8. RS-232 interface

| Ref.  | Qty | Part/function                             | Package                  | Source       |
| ----- | --: | ----------------------------------------- | ------------------------ | ------------ |
| U27   |   1 | MAX233-compatible dual RS-232 transceiver | SOIC-20                  | BUY          |
| J1    |   1 | DE-9 serial connector                     | Right-angle through-hole | BUY/TBD      |
| R1–R6 |   6 | 0 Ω DTE/DCE routing links                 | 0603                     | BUY/STASH    |
| C1    |   1 | 100 nF decoupling                         | 0603                     | STASH        |
| D1    |   1 | RS-232 ESD protection array               | SMD                      | OPTIONAL/DNP |

One transmitter/receiver pair carries TX/RX. The second pair may carry RTS/CTS.

The connector routing should permit DTE or DCE population through 0 Ω links.

## 9. PS/2 keyboard interface

| Ref.   | Qty | Part/function                 | Package                  | Source       |
| ------ | --: | ----------------------------- | ------------------------ | ------------ |
| J2     |   1 | Mini-DIN-6 PS/2 receptacle    | Right-angle through-hole | BUY          |
| R7–R8  |   2 | 4.7 kΩ CLK/DATA pull-ups      | 0603                     | STASH        |
| R9–R10 |   2 | 100–220 Ω series resistors    | 0603                     | BUY/TBD      |
| D2     |   1 | Low-capacitance ESD array     | SMD                      | OPTIONAL/DNP |
| F1     |   1 | Keyboard +5 V resettable fuse | 1206                     | STASH/TBD    |

The HD63C03 drives CLK and DATA low by changing the relevant port pin to an output-low state and releases the line by returning it to input mode.

## 10. VGA output

| Ref.    | Qty | Part/function                      | Package                  | Source       |
| ------- | --: | ---------------------------------- | ------------------------ | ------------ |
| J3      |   1 | DE-15 VGA receptacle               | Right-angle through-hole | BUY          |
| R11–R16 |   6 | 1.0 kΩ RGB normal/bright resistors | 0603                     | STASH        |
| R17–R18 |   2 | 33 Ω HSYNC/VSYNC series resistors  | 0603                     | STASH        |
| D3      |   1 | VGA ESD protection array           | SMD                      | OPTIONAL/DNP |

Each color channel uses two approximately equal resistor branches:

```text
normal-color output ── 1 kΩ ──┐
bright-color output ── 1 kΩ ──┼── VGA color pin
                              └── 75 Ω termination in monitor
```

Approximate levels:

```text
Normal:  one 1 kΩ branch active  → approximately 0.35 V
Bright:  two 1 kΩ branches active → approximately 0.65 V
```

U25 supplies six RGB outputs and buffered HSYNC/VSYNC.

## 11. Reset, configuration, and indicators

| Ref.    | Qty | Part/function                                 | Package          | Source    |
| ------- | --: | --------------------------------------------- | ---------------- | --------- |
| U28     |   1 | 5 V active-low reset supervisor, ≥20 ms delay | SOT-23           | BUY       |
| SW1     |   1 | Momentary reset switch                        | SMD/TH           | BUY       |
| SW2     |   1 | 4-position configuration DIP switch           | SMD/TH           | STASH/BUY |
| LED1    |   1 | Power LED                                     | 0603/1206        | STASH     |
| LED2    |   1 | Status/RX LED                                 | 0603/1206        | STASH     |
| LED3    |   1 | Status/TX LED                                 | 0603/1206        | STASH     |
| R19–R21 |   3 | 1 kΩ LED resistors                            | 0603             | STASH     |
| BZ1     |   1 | 5 V piezoelectric terminal bell               | Through-hole/SMD | OPTIONAL  |
| Q1      |   1 | Small transistor/MOSFET for bell              | SOT-23           | OPTIONAL  |
| R22     |   1 | Bell gate/base resistor                       | 0603             | OPTIONAL  |

The internal 256-byte RAM is available immediately after reset, allowing firmware to establish a stack and initialize the undefined external bank latches before using external SRAM.

## 12. Power and decoupling

| Ref.    | Qty | Part/function                      | Package   | Source  |
| ------- | --: | ---------------------------------- | --------- | ------- |
| J4      |   1 | Regulated +5 V input connector     | TBD       | BUY     |
| F2      |   1 | 0.75–1.0 A resettable fuse         | SMD       | BUY/TBD |
| D4      |   1 | Reverse-polarity protection device | SMD       | BUY/TBD |
| C2–C31  | ~30 | 100 nF ceramic decoupling          | 0603      | STASH   |
| C32–C35 |   4 | 4.7–10 µF local bulk capacitors    | 0603/1206 | STASH   |
| C36     |   1 | 100 µF input bulk capacitor        | Radial TH | STASH   |
| TP1     |   1 | +5 V test point                    | —         | PCB     |
| TP2     |   1 | Ground test point                  | —         | PCB     |

Use one 100 nF capacitor per IC, placed at the power pins. Additional bulk capacitors should be placed near the PLD cluster, memory/video cluster, and external connectors.

## 13. Sockets and mechanical items

| Qty | Item                           | Source    |
| --: | ------------------------------ | --------- |
|   1 | PLCC-68 socket for HD63C03YCP  | BUY       |
|   1 | DIP-40 socket for MC6845B      | STASH/BUY |
|   1 | DIP-32 socket for AS6C4008     | BUY       |
|   2 | DIP-28 sockets for EEPROMs     | STASH/BUY |
|   4 | M3 mounting holes and hardware | BUY       |
|   1 | 100 × 100 mm four-layer PCB    | FAB       |
|   1 | Enclosure                      | TBD       |

SOIC ATF22V10 devices must be programmed before assembly. Provide accessible test pads for all PLD pins and retain an optional DIP-24 development footprint or adapter strategy if frequent PLD iterations are expected.

## 14. Future RS-422/pBITz interface — DNP

Do not finalize these parts until the pBITz external peripheral electrical standard is frozen.

| Ref.    | Qty | Placeholder function                     | Status  |
| ------- | --: | ---------------------------------------- | ------- |
| U29     |   1 | RS-422 line driver, e.g. AM26C31 class   | DNP     |
| U30     |   1 | RS-422 line receiver, e.g. AM26C32 class | DNP     |
| J5      |   1 | pBITz external peripheral connector      | DNP/TBD |
| R23–R24 |   2 | 120 Ω termination                        | DNP     |
| R25–R28 |   4 | Receiver bias resistors                  | DNP/TBD |
| D5      |   1 | Differential-line TVS array              | DNP     |

## 15. Optional service/debug connections

| Ref.    | Qty | Function                                                     |
| ------- | --: | ------------------------------------------------------------ |
| J6      |   1 | TTL-level serial/debug header                                |
| J7      |   1 | Logic-analyzer header: E, R/W, MR, phase bits, SRAM controls |
| J8      |   1 | Video test header: pixel, DE, HSYNC, VSYNC, glyph load       |
| TP3–TP8 |   6 | Major clock and arbitration test points                      |

## 16. Immediate purchase list

### Required

* 1 × HD63C03YCP, verified PLCC-68
* 1 × PLCC-68 socket
* 1 × 24.000 MHz 5 V HCMOS oscillator
* 4 × 74AHCT157 address multiplexers
* 1 × SMD 74HCT245 bus transceiver
* 1 × MAX233-compatible SMD RS-232 transceiver
* 1 × DE-15 VGA connector
* 1 × Mini-DIN-6 PS/2 connector
* 1 × serial connector
* 1 × reset supervisor
* 1 × appropriately rated resettable fuse
* remaining sockets and mechanical hardware

### Verify in stash before ordering

* 5 × ATF22V10C-10SU
* 7 × SN74HC373 SMD
* 1 × SN74HC273 SMD
* 1 × CD74HC299 SMD
* 1 × SN74HC244 SMD
* 1 × 74HC00 SMD
* 1 × 14.7456 MHz oscillator
* AT28C256
* AT28C64B
* AS6C4008-55PIN
* MC6845B

## 17. Open engineering items

1. Verify the exact MC6845B markings and timing grade.
2. Simulate or bench-test the 83.3 ns SRAM fetch slots with the chosen `157` family.
3. Confirm `MR` phase alignment against the HD63C03Y external bus timing.
4. Determine VGA horizontal/vertical totals for the 24 MHz pixel clock.
5. Freeze the DE-9 DTE/DCE arrangement.
6. Confirm SOIC PLD programming and rework strategy.
7. Establish worst-case +5 V current before selecting the input fuse.
8. Freeze the attribute-byte definitions before writing the video PLD equations.
9. Decide whether the second font EEPROM bank is:

   * alternate terminal font,
   * CP437,
   * Japanese/graphics experiment,
   * or a diagnostic font.
