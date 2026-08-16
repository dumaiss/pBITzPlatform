# pBITz Control Signal Conventions

This document defines the **logical meaning and recommended use of the pBITz `CTRL_*` signals**.

It intentionally does **not** define connector pin numbers, connector rows, routing, pull-ups, termination, or other physical implementation details. Those belong in the pBITz Backplane Architecture documentation and the KiCad design files.

The purpose of the `CTRL_*` convention is to make expansion cards as reusable as practical across machines built around different CPU and bus families. In particular, simple video, sound, storage, and I/O cards should see a familiar peripheral-facing bus even when the CPU itself does not natively provide Z80-style `/RD`, `/WR`, `/MREQ`, or `/IORQ` signals.

The guiding principle is:

> **CPU boards translate their native bus into pBITz transaction semantics; expansion cards should not need to understand the CPU that is driving the backplane.**

This is a compatibility convention rather than an attempt to force every processor into an identical bus architecture. Some CPU-specific signals have no useful cross-family equivalent and are therefore optional or reserved.

---

## 1. Design goals

The control-signal map is intended to provide four things:

1. **A stable peripheral interface.** Common cards should be able to depend on reset, read, write, wait-state, and interrupt semantics without caring which CPU family is installed.
2. **A natural mapping for simple 8-bit CPUs.** A Z80-like bus can expose most of the core pBITz controls directly.
3. **A normalization target for other buses.** Memory-mapped CPUs, 16/32-bit CPUs, and MCU/SoC external buses can use glue logic to synthesize the same peripheral-facing controls.
4. **Room for family-specific extensions.** Signals with no meaningful common semantic are not forced into false equivalence merely to fill the connector.

The CPU board, or the glue logic associated with it, is therefore responsible for generating canonical pBITz transactions when the native processor bus does not already provide them.

---

## 2. Naming and polarity

The physical bus signals remain named `CTRL_0` through `CTRL_19`.

This document assigns semantic names such as `/RD` or `/WAIT` to those signals for discussion. A leading `/` means **active low**.

Unless a machine-specific extension explicitly says otherwise, expansion cards should interpret the canonical active-low controls according to the semantics in this document rather than according to the native signal name of any particular CPU.

---

## 3. Canonical control map

| pBITz signal | Canonical role | Direction relative to CPU board | Portability | Notes |
| --- | --- | --- | --- | --- |
| `CTRL_0` | `/RESET` | System → cards/CPU | **Core** | Shared system reset. Low means reset asserted. |
| `CTRL_1` | `/RD` | CPU → card | **Core** | Peripheral read strobe. Low means the selected target must drive read data. |
| `CTRL_2` | `/WR` | CPU → card | **Core** | Peripheral write strobe. Low means the selected target should capture write data. |
| `CTRL_3` | `/MREQ` | CPU → card | Common | Memory-space transaction qualifier. May be synthesized on CPUs with only memory-mapped I/O. |
| `CTRL_4` | `/IORQ` | CPU → card | Common | I/O-space transaction qualifier. May be synthesized from an address aperture on CPUs with no native I/O space. |
| `CTRL_5` | Primary CPU cycle/status | CPU → card | Family-specific | Directly useful on buses with an instruction/special-cycle qualifier. Portable cards should not require it. |
| `CTRL_6` | Reserved family status | — | Reserved | Available for family-specific status such as refresh if a future design requires it. |
| `CTRL_7` | `/HALT` or CPU-stop status | Normally CPU → card | Family-specific | Indicates a stopped/halted CPU where the processor provides a useful status output. Direction and semantics are not portable across all families. |
| `CTRL_8` | `/BYTE_HI` | CPU → card | 16-bit extension | Recommended upper-byte lane qualifier for `D15..D8`. Unused by an 8-bit bus master. |
| `CTRL_9` | `/BYTE_LO` | CPU → card | 16-bit extension | Recommended lower-byte lane qualifier for `D7..D0`. Unused by an 8-bit bus master. |
| `CTRL_10` | `/BUSREQ` | Bus master/card → CPU | DMA extension | Requests that the CPU release the bus. Native arbitration may require glue logic. |
| `CTRL_11` | `/BUSACK` | CPU → bus master/card | DMA extension | Indicates that the CPU-side bridge has granted/released the bus. |
| `CTRL_12` | `/WAIT` | Card → CPU | **Core optional** | Low means the current transaction must be extended because the selected target is not ready. |
| `CTRL_13` | Reserved | — | Reserved | Intentionally unassigned. Suitable for a future cross-family function only when a real requirement exists. |
| `CTRL_14` | `/IRQ` | Card → CPU | **Core optional** | Normal maskable interrupt request. CPU-side glue converts this to the processor's native interrupt mechanism. |
| `CTRL_15` | `/NMI` | Card → CPU | Common optional | Highest-priority/non-maskable interrupt request where the CPU architecture supports such a concept. |
| `CTRL_16` | Secondary interrupt / family extension | Card → CPU | Optional | Reserved for a useful secondary interrupt class, such as a fast interrupt, when a machine defines one. Portable cards should not require it. |
| `CTRL_17` | Reserved | — | Reserved | No portable interrupt-chain or vectoring semantic is currently assigned. |
| `CTRL_18` | Reserved | — | Reserved | No portable interrupt-chain or vectoring semantic is currently assigned. |
| `CTRL_19` | `/CS_CART` | CPU/decode → card | Platform optional | Cartridge/boot-ROM select generated by the machine's address decode when that feature exists. |

### Core peripheral subset

A simple reusable 8-bit peripheral should ideally need only:

- its normal address and data lines;
- one of the standard pBITz card/chip-select signals;
- `CTRL_0` (`/RESET`);
- `CTRL_1` (`/RD`);
- `CTRL_2` (`/WR`);
- optionally `CTRL_12` (`/WAIT`); and
- optionally `CTRL_14` (`/IRQ`).

This is the preferred compatibility target for video, sound, and other register-oriented cards.

A card that is fully qualified by its chip-select generally does **not** need to depend on `/MREQ` or `/IORQ`. This is desirable because it allows the CPU board to choose whether a peripheral appears in native I/O space or in a memory-mapped compatibility aperture.

---

## 4. Transaction semantics

### 4.1 `/RD` and `/WR` are peripheral strobes, not raw CPU pins

`CTRL_1` and `CTRL_2` are deliberately defined as separate active-low transaction strobes even though many processors expose only a single `R/W` signal.

The CPU-side glue must generate clean pBITz `/RD` and `/WR` strobes qualified by the processor's valid-bus-cycle timing.

This is important for card reuse: a peripheral designed around `/RD` and `/WR` should not need alternate logic merely because the host CPU happens to use `R/W`, `/AS`, `E`, or some other bus-cycle convention.

### 4.2 `/MREQ` and `/IORQ` describe address spaces

Some CPUs provide distinct memory and I/O cycles; others are entirely memory mapped.

For a CPU with native memory and I/O strobes, `CTRL_3` and `CTRL_4` can map naturally to those cycles.

For a memory-mapped CPU, the CPU board may reserve an address region as a **pBITz I/O aperture** and generate:

- `CTRL_4` (`/IORQ`) for accesses inside that aperture; and
- `CTRL_3` (`/MREQ`) for ordinary memory-space accesses.

This lets a card that genuinely requires an I/O-cycle qualifier remain reusable on a memory-mapped processor.

Simple cards should still prefer the standard decoded card-select signals where possible and avoid depending on the distinction.

### 4.3 `/WAIT` always means "the target is not ready"

`CTRL_12` has one portable semantic:

> **Low means extend the current bus transaction.**

The CPU board must translate that request into the processor's native cycle-extension mechanism.

This distinction matters on buses such as the 68000 family. `/DTACK` is an active-low **completion/acknowledge** signal, while pBITz `/WAIT` is an active-low **hold/not-ready** signal. They therefore must not simply be treated as the same wire. A 68k CPU-side bridge should withhold `/DTACK` while pBITz `/WAIT` is asserted and complete the native transaction only when the pBITz target is ready.

### 4.4 Interrupt lines carry semantic requests

`CTRL_14` and `CTRL_15` describe interrupt classes rather than specific CPU pins.

- `CTRL_14` is the normal maskable interrupt request.
- `CTRL_15` is the highest-priority/non-maskable request when the host architecture supports one.

A CPU with encoded interrupt-priority inputs should use glue logic to convert these requests into its native interrupt representation. The pBITz lines should **not** be wired arbitrarily to individual encoded priority bits merely because the pin count happens to match.

Interrupt acknowledge cycles, vector presentation, and priority daisy chains are **not currently part of the portable pBITz control convention**. Cards that require CPU-family-specific vectoring are therefore family-aware unless a later pBITz extension standardizes a common mechanism.

---

## 5. CPU-family mapping guidance

The mappings below describe how a CPU board should normally adapt its native signals to the pBITz semantics. They are guidance for CPU-board glue design, not additional signals exposed by the backplane.

### 5.1 Z80-family buses

A Z80-style bus is close to the canonical pBITz model and requires little translation.

| pBITz signal | Z80-family mapping |
| --- | --- |
| `CTRL_0` | `/RESET` |
| `CTRL_1` | `/RD` |
| `CTRL_2` | `/WR` |
| `CTRL_3` | `/MREQ` |
| `CTRL_4` | `/IORQ` |
| `CTRL_5` | `/M1` |
| `CTRL_6` | Normally unused; `/RFSH` is a possible family-specific use if ever required |
| `CTRL_7` | `/HALT` |
| `CTRL_8` | Unused on an 8-bit implementation |
| `CTRL_9` | Unused on an 8-bit implementation |
| `CTRL_10` | `/BUSREQ` |
| `CTRL_11` | `/BUSACK` |
| `CTRL_12` | `/WAIT` |
| `CTRL_13` | Reserved |
| `CTRL_14` | `/INT` |
| `CTRL_15` | `/NMI` |
| `CTRL_16` | Unused / family extension |
| `CTRL_17` | Reserved |
| `CTRL_18` | Reserved |
| `CTRL_19` | `/CS_CART`, generated by address decode if supported |

`/M1` is useful as a native Z80 status signal, but portable expansion cards should not assume that another CPU family can reproduce its exact semantics.

The pBITz convention does **not** currently require the Z80 `IEI`/`IEO` daisy chain to be carried by `CTRL_17` and `CTRL_18`. A physical interrupt-priority chain is topology-sensitive and should not be implied by two ordinary shared control signals.

### 5.2 6809 / 6309-family buses

The 6809/6309 family is memory mapped and uses `R/W` rather than separate read and write strobes. CPU-side glue should therefore normalize the bus before presenting it to pBITz.

Recommended behaviour:

- Generate `CTRL_1` (`/RD`) and `CTRL_2` (`/WR`) from `R/W`, qualified by the valid portion of the processor bus cycle.
- Generate `CTRL_3` (`/MREQ`) from the processor's valid-memory-cycle timing.
- If I/O-space compatibility is desired, reserve an address aperture and generate `CTRL_4` (`/IORQ`) when that aperture is accessed.
- Do not force an approximate instruction-cycle signal onto `CTRL_5` merely to imitate `/M1`; leave it unused unless a machine has a genuine use for family-specific cycle status.
- Treat `CTRL_7` cautiously. The native `HALT` function on this family is primarily a CPU-control input, not a portable equivalent of the Z80 `/HALT` status output.
- Translate `CTRL_10`/`CTRL_11` into the family's bus-release mechanism. A typical implementation can use HALT/bus-control logic for the request and BA/BS-derived state to determine when the bus is actually available.
- Translate `CTRL_12` to the processor's memory-ready or cycle-stretching mechanism.
- Map `CTRL_14` to `/IRQ` and `CTRL_15` to `/NMI`.
- `CTRL_16` is a natural candidate for `/FIRQ` on a machine that chooses to use the optional secondary-interrupt convention, but a portable card must not assume it is present.
- `CTRL_8` and `CTRL_9` remain unused on the native 8-bit bus.

### 5.3 65xx / 65816-family buses

65xx-family processors are also memory mapped and expose `R/W` rather than separate peripheral strobes.

Recommended behaviour:

- Generate pBITz `/RD` and `/WR` from `R/W` and the valid phase of `PHI2`.
- On a 65816, use `VDA`/`VPA` as appropriate to ensure that pBITz transactions are generated only for valid external address cycles.
- Generate `CTRL_4` (`/IORQ`) from an address-decoded I/O aperture when compatibility with I/O-space peripherals is useful; otherwise peripherals may simply remain memory mapped and use the standard card-select lines.
- `VPA` may be useful internally to CPU glue, but it is not an exact replacement for Z80 `/M1` and should not be treated as a portable `CTRL_5` semantic.
- Translate pBITz `/WAIT` into the native `RDY` mechanism.
- Map `CTRL_14` and `CTRL_15` to the native maskable and non-maskable interrupt inputs where available.
- The 65816 has an 8-bit external data bus despite its 16-bit CPU architecture, so `CTRL_8` and `CTRL_9` are normally unused.
- Bus mastering on parts with a bus-enable facility should still be mediated by CPU-side glue so `CTRL_10` means **request ownership** and `CTRL_11` means **ownership has actually been granted**. CPU-specific lock/state signals must be respected before the grant is issued.

The 65816 `ABORT` input has no current portable pBITz control assignment. It should not be silently mapped onto a reserved control line without first defining a platform-wide bus-fault semantic.

### 5.4 68000 / 68010-family buses

The 68000 family differs more substantially from an 8-bit peripheral bus, but its asynchronous bus can be translated cleanly by CPU-side glue.

Recommended behaviour:

- Derive `CTRL_1` (`/RD`) and `CTRL_2` (`/WR`) from `/AS`, `R/W`, and the active byte-lane strobes. Do **not** expose the raw `R/W` line as though it were pBITz `/WR`.
- Generate `CTRL_3` (`/MREQ`) for ordinary pBITz memory transactions.
- Since the 68000 has memory-mapped I/O, generate `CTRL_4` (`/IORQ`) from a reserved pBITz I/O aperture if I/O-space card compatibility is desired.
- For a native 16-bit pBITz transfer, map `CTRL_8` to the upper-byte lane and `CTRL_9` to the lower-byte lane. On a 68000/68010 these correspond naturally to `/UDS` and `/LDS` respectively.
- For reusable 8-bit cards on `D7..D0`, the CPU-side bridge/address decode should hide the 68000's byte-lane and odd/even-address details from the card whenever practical.
- Map the arbitration request toward `/BR`; generate the pBITz `/BUSACK` semantic only when the native grant/acknowledge sequence has actually made the bus available.
- Do **not** connect pBITz `/WAIT` directly to `/DTACK`. The CPU-side state machine should delay `/DTACK` while the selected pBITz target asserts `/WAIT`.
- Convert pBITz interrupt requests into the native encoded `/IPL[2:0]` levels. A normal `/IRQ` may be assigned a machine-defined interrupt level; pBITz `/NMI` can be represented by the architecture's highest-priority interrupt level when appropriate.
- Native interrupt-acknowledge cycles and autovector/vector-device behaviour remain CPU-family glue concerns unless a future pBITz extension standardizes them.

### 5.5 68020 / 68030-family buses

Later 68k processors should generally be treated as requiring a **bus bridge**, not a collection of one-to-one signal substitutions.

Their wider buses and transfer-size/acknowledge conventions should be translated into the canonical pBITz transaction model:

- separate `/RD` and `/WR` peripheral strobes;
- the pBITz memory/I/O-space convention;
- appropriate `CTRL_8`/`CTRL_9` byte-lane qualification for the 16-bit pBITz data path;
- `/WAIT` translated into the CPU's native acknowledge/termination mechanism;
- bus arbitration translated into pBITz `/BUSREQ` and `/BUSACK`; and
- interrupt requests encoded into the CPU's native priority inputs.

A 32-bit CPU board may use only a 16-bit portion of the pBITz data bus for expansion cycles or may implement lane steering in glue logic. That is a CPU-board implementation detail; expansion cards should continue to see the same pBITz semantics.

### 5.6 Other external-memory buses and MCU/SoC buses

A processor does not need to resemble any of the classic buses above to participate in pBITz.

An MCU or SoC with an asynchronous external-memory controller can normally provide the canonical interface by translating its native chip-select, read, write, wait/ready, and interrupt mechanisms in glue logic.

The same rules apply:

- expose clean peripheral-facing `/RD` and `/WR` strobes;
- synthesize `/IORQ` from an address aperture when useful;
- preserve the pBITz `/WAIT` meaning rather than copying a native acknowledge signal with opposite polarity/semantics; and
- hide native interrupt encoding and bus-width details from ordinary expansion cards.

---

## 6. Card compatibility profiles

The following informal profiles help card designers decide how portable a design is likely to be.

### Profile A — Simple peripheral

Uses only decoded card select, `CTRL_0`, `CTRL_1`, and `CTRL_2`, plus address/data lines.

Optional use of `CTRL_12` and `CTRL_14` is still considered highly portable.

**Typical examples:** sound generators, register-based video devices, UART-like peripherals, simple storage interfaces.

This is the preferred profile for maximum Coffee-series reuse.

### Profile B — Address-space-aware peripheral

Adds `CTRL_3` and/or `CTRL_4` because the device needs to distinguish memory and I/O transactions.

This remains portable, but memory-mapped CPU boards may need to synthesize the distinction.

### Profile C — 16-bit peripheral

Adds `CTRL_8` and `CTRL_9` for byte-lane qualification.

The card should still use canonical `/RD` and `/WR` semantics rather than depending on a native 68k-style bus cycle.

### Profile D — Bus master / DMA device

Adds `CTRL_10`, `CTRL_11`, and usually `CTRL_12`.

The CPU board is responsible for translating the request/grant handshake into its processor's native arbitration protocol.

### Profile E — CPU-family-aware device

Depends on `CTRL_5`, `CTRL_6`, `CTRL_7`, `CTRL_16`, or future assignments of the reserved controls.

Such a card may still be useful, but it should be documented as requiring a particular CPU-family capability rather than presented as a generally portable pBITz peripheral.

---

## 7. Reserved controls and future extensions

A reserved `CTRL_*` line should remain reserved until a requirement exists on more than one design or there is a strong compatibility reason to standardize it.

In particular:

- `CTRL_6` is **not automatically `/RFSH`** simply because a Z80 can provide that signal.
- `CTRL_13` is intentionally available for a future cross-family function such as bus fault/abort handling, but no such semantic is currently standardized.
- `CTRL_17` and `CTRL_18` are **not automatically Z80 `IEI`/`IEO`**. Interrupt priority chains have physical topology requirements that are distinct from assigning two shared connector signals.

When a new assignment is proposed, the preferred question is not "what unused CPU pin can go here?" but:

> **What peripheral-facing function would make a useful card portable across more than one CPU family?**

That keeps the pBITz control block focused on compatibility rather than on reproducing the pinout of whichever CPU board happened to be designed first.

---

## 8. Source of truth

This document is the source of truth for the **logical convention** associated with `CTRL_0` through `CTRL_19`.

It is **not** the source of truth for their physical connector locations. Physical signal placement, connector orientation, power pins, ground pins, and electrical implementation belong to the current pBITz backplane schematic and Backplane Architecture document.
