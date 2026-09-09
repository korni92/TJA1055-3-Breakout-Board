# TJA1055/3 Breakout Board — Rev 1.1

An open-hardware breakout for **low-speed fault-tolerant CAN (ISO 11898-3)**, built around
the NXP **TJA1055T/3** transceiver with an optional on-board Microchip **MCP2515** CAN
controller. Runs directly from **3.3 V hosts** — Raspberry Pi, ESP32 — with no level
shifters.

> ⚠️ **Use at your own risk.** This is a bench and diagnostic tool, not a qualified ECU.
> Only the 12 V rail is reverse-polarity protected; there is no fuse and no bus transient
> suppression. Working on a live vehicle bus can disturb or damage control units.

---

## Quick start

If you just want CAN and don't care about the transceiver's power-management features:

1. Fit both **SWITCH** jumpers (J10, J11). Without them STB and EN are pulled low
   internally and the transceiver sits in sleep — the board looks dead.
2. Leave both **TERM** jumpers (J6, J7) open when connecting to a vehicle. The bus is
   already terminated.
3. Set **J2 and J4 both to the right** for the on-board MCP2515, or both to the left to
   drive TXD/RXD from your own controller via J1 pins 1 and 2.
4. Feed GND, 12 V, 5 V and 3.3 V into J1.

---

## Contents

- [Features](#features)
- [What changed in Rev 1.1](#what-changed-in-rev-11)
- [Block diagram](#block-diagram)
- [Connectors](#connectors)
- [Jumpers](#jumpers)
- [Operating modes](#operating-modes)
- [Bus termination](#bus-termination)
- [Power supply](#power-supply)
- [Local wake-up](#local-wake-up)
- [Raspberry Pi](#raspberry-pi)
- [ESP32](#esp32)
- [Bit timing at 8 MHz](#bit-timing-at-8-mhz)
- [Bill of materials](#bill-of-materials)
- [Assembly](#assembly)
- [Known limitations](#known-limitations)
- [Design notes](#design-notes)
- [License](#license)

---

## Features

- **TJA1055T/3** fault-tolerant CAN transceiver: up to 125 kBd, automatic single-wire
  fallback on bus faults, ±58 V bus robustness, ±8 kV HBM ESD on CANH/CANL/RTH/RTL
- **Native 3.3 V interface.** The `/3` variant has open-drain RXD and ERR outputs; the
  required pull-ups (R5, R1) are on the board
- **Two ways to reach the bus**, jumper-selectable:
  - the on-board **MCP2515** over SPI — for Raspberry Pi, or an ESP32 with no free TWAI
    controller
  - an **external CAN controller** wired straight to TXD/RXD — e.g. the ESP32 TWAI
    peripheral
- **Switchable termination**: ~547 Ω or 4.7 kΩ per bus line
- **Local wake-up input** for waking the transceiver out of standby or sleep
- **INH output** for enabling an external regulator
- Screw terminal and pin header in parallel for the CAN bus
- Reverse-polarity protection on the 12 V input

## What changed in Rev 1.1

| Change | Reason |
|---|---|
| **R13 (1 kΩ)** in series with pin BAT | Automotive transient protection; NXP AH0801 §4.2.5 recommends 1–2 kΩ |
| **C7 (10 nF)** from BAT to GND | Transient suppression close to the BAT pin |
| **R14 (47 kΩ) / R4 (10 kΩ)** wake network, **J12** header | Local wake-up per AH0801 §4.2.2 — R14 pulls to battery, R4 limits current into the pin |
| **D1 (PMEG4010CEH)** on the 12 V input | Reverse-polarity blocking for the whole battery branch |
| **C8 (100 µF alu-polymer)** on the 5 V rail | Buffers the transceiver's peak current during bus faults |
| **INH** on J1 with **R15 (100 kΩ)** pull-down | Lets the board control an external regulator |
| **J10/J11 + R16/R17 (10 kΩ)** SWITCH jumpers | Optional strap of STB and EN to 3.3 V |
| **R10/R11 raised to 620 Ω** | Keeps the parallel termination safely above the 500 Ω minimum |
| J5 folded into J1, J1 grown to 1×10 | Fewer connectors, RX/TX now on the main header |

Rev 1.0 ran in-vehicle with both a Raspberry Pi and an ESP32.

## Block diagram

```
  J1.4 BAT ─[D1]─┬──────────────────────────────────────────────┐
                 │                                              │
                 ├─[R13 1k]──┬── BAT ┐                   [R14 47k]
                 │        [C7 10n]   │                          │
                 │           │       │      ┌── RTH ─[R9 4k7]───┴──┬── CANH ── HI
  J1.5 +5V ──────┴───────────┴── VCC │      │      [J6+R10 620] ───┘
              [C1 100n, C8 100u]     │      │
                                     │      ├── RTL ─[R12 4k7]──┬── CANL ── LO
  J1.2 TX ──┬───────────────────── TXD      │      [J7+R11 620] ─┘
            │   (J2/J4: MCP or external)    │
  J1.1 RX ──┴──[R5 4k7 → 3V3]────────── RXD │      C5/C6 100 pF: CANH/CANL → GND
  J1.10 ERR ───[R1 4k7 → 3V3]────────── ERR │
  J1.8 STB ─┬────────────────────────── STB │      STB/EN have internal pull-down
  J1.9 EN  ─┤                           EN  │      current sources — floating means
            └─[J10/J11 + R16/R17 10k → 3V3] │      sleep, not normal mode
  J1.7 INH ─┬────────────────────────── INH │
         [R15 100k]                         │
                                            │
  J12 ── button ── GND ───────[R4 10k]── WAKE
                                            │
  J3 SPI ── MCP2515 (3V3, 8 MHz) ── TXCAN/RXCAN ── J2/J4
```

## Connectors

All through-hole positions take standard 2.54 mm male pin headers.

### J1 — power, control and external CAN controller (1×10)

| Pin | Silk | Signal | Direction | Notes |
|---|---|---|---|---|
| 1 | `RX` | RXD | out | from the transceiver to your controller (J2 set left) |
| 2 | `TX` | TXD | in | from your controller to the transceiver (J4 set left) |
| 3 | `GND` | GND | — | must be bonded to vehicle ground |
| 4 | `BAT` | +12 V | in | battery feed, behind D1. 5–40 V is within the transceiver's rating |
| 5 | `5V` | +5 V | in | transceiver V<sub>CC</sub> — see [Power supply](#power-supply) |
| 6 | `3V3` | +3.3 V | in | MCP2515 supply and reference for every pull-up on the board |
| 7 | `INH` | INH | out | regulator enable, **battery-referenced** |
| 8 | `STB` | STB | in | mode pin, active low |
| 9 | `EN` | EN | in | mode pin |
| 10 | `ERR` | ERR | out (open drain) | error / wake-up / power-on flag, active low, pulled up to 3V3 |

> ⚠️ **INH is not a GPIO.** It is a high-side switch to BAT and outputs V<sub>BAT</sub> when
> active. Do not connect it to a 3.3 V input without level shifting.

### J3 — SPI to the MCP2515 (1×5)

| Pin | Silk | Signal | Pi (BCM) | ESP32 (example) |
|---|---|---|---|---|
| 1 | `CS` | CS | GPIO8 (CE0) | GPIO5 |
| 2 | `SO` | MISO | GPIO9 | GPIO19 |
| 3 | `SI` | MOSI | GPIO10 | GPIO23 |
| 4 | `SCK` | SCK | GPIO11 | GPIO18 |
| 5 | `INT` | INT | GPIO25 | GPIO4 |

### J8 / J9 — CAN bus

J8 is a screw terminal, J9 a pin header, wired in parallel so the board can be daisy-chained.
Both are marked `HI` and `LO` on the silkscreen, and **viewed from above, HI and LO are on
the same side of both connectors** — go by the silkscreen, not by pin number

| Silk | Signal |
|---|---|
| `HI` | CANH |
| `LO` | CANL |

> Fault-tolerant CAN has **no** 120 Ω resistor across the pair. Plugging in a high-speed CAN
> tool with its 120 Ω termination forces the whole bus into single-wire mode.

### J12 — local wake-up input (1×1)

Connect a button or contact between J12 and ground; J1 pin 3 is convenient. See
[Local wake-up](#local-wake-up).

## Jumpers

| Ref | Silk | Function | Default |
|---|---|---|---|
| J4 | `RX/TX ↦ … ↦ MCP2515` | TXD source: left = J1.2, right = MCP2515 | right |
| J2 | `RX/TX ↦ … ↦ MCP2515` | RXD destination: left = J1.1, right = MCP2515 | right |
| J6 | `TERM` | 620 Ω in parallel on RTH (CANH side) | open |
| J7 | `TERM` | 620 Ω in parallel on RTL (CANL side) | open |
| J10 | `SWITCH` `GPIO`/`ON` | straps STB to 3.3 V through R16 (10 kΩ) | fitted |
| J11 | `SWITCH` `GPIO`/`ON` | straps EN to 3.3 V through R17 (10 kΩ) | fitted |

### J2 / J4 — controller select

The two 1×3 headers sit one above the other and are mirrored, so the pads line up in three
columns regardless of pin numbering:

| Column | Upper row (J2, RXD) | Lower row (J4, TXD) |
|---|---|---|
| left | J1.1 RX | J1.2 TX |
| middle | transceiver RXD | transceiver TXD |
| right | MCP2515 RXCAN | MCP2515 TXCAN |

Move both jumpers to the same side. With neither fitted, TXD floats — the TJA1055 has an
internal pull-up on TXD, so the bus stays recessive and the board will not disturb traffic,
but it will not transmit or receive either.

### J10 / J11 — SWITCH

STB and EN have internal pull-down current sources of roughly 10 µA. Left alone the
transceiver falls into a low-power mode on its own, which is the fail-safe behaviour NXP
intends. Fitting a SWITCH jumper straps the pin to 3.3 V through 10 kΩ, so the board comes
up in normal mode without the host driving anything.

The 10 kΩ matters: with a jumper fitted **and** a GPIO connected, the GPIO still wins — it
only has to sink 330 µA — so you keep full mode control without a short between the GPIO
and the 3.3 V rail. You never have to pull the jumper.

> Fitted jumpers keep the transceiver in normal mode permanently, drawing several mA.
> On permanent battery power that will slowly discharge the vehicle battery.

## Operating modes

| STB (J1.8) | EN (J1.9) | Mode |
|---|---|---|
| 1 | 1 | **Normal** — transmit and receive; ERR reports bus faults |
| 1 | 0 | Power-on standby — ERR reports the power-on flag |
| 0 | 1 | Go-to-sleep — hold for at least 50 µs |
| 0 | 0 | Standby / sleep — RXD and ERR report the wake-up flag |

INH is active in every mode except sleep.

## Bus termination

Fault-tolerant CAN uses **distributed** termination: every node carries one resistor from
RTH to CANH and one from RTL to CANL.

- Per node: **500 Ω to 6 kΩ**
- Across the whole network: about **100 Ω per line**

| TERM jumpers | R<sub>RTH</sub> / R<sub>RTL</sub> | When |
|---|---|---|
| open | 4.7 kΩ | **default in a vehicle** — the bus is already terminated |
| fitted | 4.7 kΩ ∥ 620 Ω ≈ 547 Ω | bench setup where this board is the terminating (gateway) node |

Set J6 and J7 together — the network only stays balanced if both lines are terminated
symmetrically. Two boards on a bench with both fitted gives 547 Ω ∥ 547 Ω ≈ 274 Ω, which
works but is higher impedance than the nominal design point.

## Power supply

There is no regulator on the board. All three rails come in through J1.

| Rail | Load | Requirement |
|---|---|---|
| BAT (12 V) | pin BAT via D1 and R13, plus R14 and R15 | tens of µA in low-power mode, ~1.7 mA worst case |
| +5 V | transceiver V<sub>CC</sub> | ~38 mA average, **peaks to 135 mA** for up to 6 bit times during a bus short |
| +3.3 V | MCP2515 and all pull-ups | ~10 mA |

C8 buffers the 5 V peaks. NXP's worst case for a separately supplied transceiver is roughly
25 µF at 125 kBd and 95 µF at 33.3 kBd, assuming a regulator that cannot respond inside the
fault window; a real supply needs less.

D1 and R13 together drop up to about 2 V at worst-case current, so the board needs roughly
7.5 V at J1.4 to keep BAT above its 5 V minimum. That only matters during cranking.

## Local wake-up

```
BAT ──[R14 47k]──┬──[R4 10k]── WAKE (pin 7)
                 │
                 └── J12 ── button ── GND
```

With nothing on J12, WAKE sits at battery level through R14 and R4 — the inactive state.
Pulling J12 to ground wakes the transceiver.

- **WAKE is edge-sensitive in both directions.** A button generates a wake-up event on press
  *and* on release.
- R14 sets the contact wetting current to roughly 255 µA at 12 V, enough for ordinary
  mechanical contacts.
- R4 is the protection resistor required by AH0801 §4.2.2. It limits current into the pin if
  the board loses its ground connection while the button is still tied to vehicle ground.

## Raspberry Pi

Set J2 and J4 to the right (MCP2515), fit both SWITCH jumpers.

`/boot/firmware/config.txt`:

```ini
dtparam=spi=on
dtoverlay=mcp2515-can0,oscillator=8000000,interrupt=25,spimaxfrequency=10000000
```

After rebooting:

```bash
sudo apt install can-utils
sudo ip link set can0 up type can bitrate 33333 restart-ms 100
candump can0
cansend can0 123#DEADBEEF
```

> `oscillator` must be **8000000**. Passing 16000000 by mistake halves every bit rate you
> configure and nothing will decode.

## ESP32

### Option A — MCP2515 over SPI

Set J2/J4 to the right. Use any MCP2515 library, configured for an **8 MHz** crystal.

### Option B — the ESP32's own TWAI controller

Set J2/J4 to the left and wire J1.1 (RX) and J1.2 (TX) to your chosen GPIOs.

```c
twai_general_config_t g = TWAI_GENERAL_CONFIG_DEFAULT(TX_GPIO, RX_GPIO, TWAI_MODE_NORMAL);
twai_timing_config_t  t = TWAI_TIMING_CONFIG_125KBITS();
twai_filter_config_t  f = TWAI_FILTER_CONFIG_ACCEPT_ALL();
```

ESP-IDF has no preset for 33.333 kBd — set the timing struct manually.

## Bit timing at 8 MHz

t<sub>Q</sub> = 2 × (BRP+1) / f<sub>OSC</sub>

| Bit rate | BRP | t<sub>Q</sub> | TQ per bit | Typical use |
|---|---|---|---|---|
| 33.333 kBd | 7 | 2.00 µs | 15 | GM low-speed CAN |
| 50 kBd | 4 | 1.25 µs | 16 | |
| 83.333 kBd | 2 | 0.75 µs | 16 | |
| 100 kBd | 4 | 1.25 µs | 8 | |
| 125 kBd | 1 | 0.50 µs | 16 | ISO 11898-3 maximum |

All of these are exact at 8 MHz — no timing error.

## Bill of materials

Passives are 0603. LCSC part numbers are the ones used for the JLCPCB assembly order.

| Ref | Value / part | Package | LCSC | Notes |
|---|---|---|---|---|
| IC1 | **NXP TJA1055T/3/1J** | SO14 | C2998207 | **The `/3` suffix is mandatory.** The plain TJA1055T drives RXD and ERR push-pull to 5 V and will damage 3.3 V GPIOs |
| U1 | Microchip MCP2515-I/SO | SOIC-18W | C12368 | |
| Y1 | YXC X49SM8MSD2SC, 8 MHz | HC-49S-SMD | C12674 | C<sub>L</sub> = 20 pF, ±20 ppm, ESR 70 Ω |
| C1, C2 | 100 nF X7R | 0603 | C14663 | V<sub>CC</sub> and V<sub>DD</sub> decoupling |
| C3, C4 | 22 pF C0G | 0603 | C1653 | see [Known limitations](#known-limitations) |
| C5, C6 | 100 pF C0G | 0603 | C14858 | CANH/CANL to GND, EMC |
| C7 | 10 nF 50 V X7R | 0603 | C57112 | BAT transient suppression |
| C8 | 100 µF 16 V alu-polymer | D6.3 × L5.8 | C5246523 | 5 V bulk, polarised |
| D1 | Nexperia PMEG4010CEH | SOD-123F | C193342 | reverse-polarity blocking |
| R1, R5, R9, R12 | 4.7 kΩ 1 % | 0603 | C23162 | ERR/RXD pull-ups and RTH/RTL termination |
| R4, R6, R7, R8, R16, R17 | 10 kΩ 1 % | 0603 | C25804 | WAKE series, MCP2515 pull-ups, SWITCH straps |
| R10, R11 | 620 Ω 1 % | 0603 | C23220 | switched termination |
| R13 | 1 kΩ 1 % | 0603 | C21190 | BAT series resistor |
| R14 | 47 kΩ 1 % | 0603 | C25819 | WAKE pull-up to battery |
| R15 | 100 kΩ 1 % | 0603 | C25803 | INH pull-down |
| J1 | 1×10 pin header | 2.54 mm | C5156617 | |
| J2, J4 | 1×3 pin header | 2.54 mm | C49257 | |
| J3 | 1×5 pin header | 2.54 mm | C5156614 | |
| J6, J7, J9, J10, J11 | 1×2 pin header | 2.54 mm | C5360898 | |
| J8 | TE 282834-2 screw terminal | 2.54 mm | C592983 | |
| J12 | 1×1 pin header | 2.54 mm | — | not carried by JLCPCB, fit by hand |

There are deliberately **no resistors permanently tied to STB and EN**. The transceiver
provides internal pull-down current sources, which is the fail-safe behaviour NXP intends;
R16/R17 are only in circuit when the SWITCH jumpers are fitted.

## Assembly

All SMD parts are on the top side and can be ordered pre-assembled. Every through-hole
part — the pin headers, the screw terminal and the single-pin wake header — is fitted by
hand.

You also need five 2.54 mm jumper shunts: two for J2/J4, two for the SWITCH positions, and
optionally two more if you use the TERM jumpers.

## Known limitations

1. **No fuse and no load-dump clamp** on the 12 V input. D1 blocks reverse polarity and R13
   limits current into the BAT pin, but for permanent in-vehicle use add an external fuse or
   PTC, and consider an SMBJ26A/SMBJ30A TVS after D1.
2. **No bus ESD/transient diodes.** The transceiver's own ±8 kV HBM rating covers bench and
   diagnostic use; for harsher environments NXP suggests a PESD1CAN close to the connector.
3. **The oscillator is deliberately under-loaded.** The crystal specifies C<sub>L</sub> =
   20 pF, which would call for roughly 33 pF per leg. C3/C4 are 22 pF instead — the value
   Microchip tested at 8 MHz (DS20001801K Table 8-2) — so the crystal sees about 15 pF and
   runs some 75 ppm fast. The MCP2515 allows 1.7 % node-to-node oscillator variation, so
   that is two orders of magnitude inside tolerance, and under-loading buys start-up margin
   and lower crystal drive. Do not "correct" this to 33 pF.
4. **J1 is an unkeyed pin header.** Plugging it in backwards puts 12 V on the 3.3 V rail.
5. **No sleep-capable power path by default.** INH is available on J1, but the board has no
   regulator of its own, so a full sleep setup needs external hardware.
6. **J6 and J7 share one `TERM` label** and are not individually marked HI and LO. They are
   always set together, so this is cosmetic.

## Design notes

- NXP **TJA1055** product data sheet (Rev. 5, 2013). For the `/3` variant, RXD and ERR are
  open-drain and require external pull-ups.
- NXP **AH0801 — Application Hints TJA1055T** (Rev. 1.5, 2016). The basis for R13/C7
  (§4.2.5), the wake circuit (§4.2.2), the µC interface including the pull-up sizing
  calculation (§4.2.4), bus capacitors (§6.1.3) and termination (§9.1). The design checklist
  in §7 is worth walking through before any layout review.
- Microchip **MCP2515** data sheet (DS20001801K). §8 and Table 8-2 give the tested
  oscillator load capacitors; §5.6 states the 1.7 % oscillator tolerance. The TXnRTS pins
  have internal 100 kΩ pull-ups and may be left open.

Layout points from the NXP checklist that apply here: keep CANH and CANL short, parallel and
symmetrical on the way to the connector; put decoupling capacitors right at the supply pins;
keep a continuous ground plane; keep the controller-to-transceiver run short.

## License

| Part | License |
|---|---|
| Hardware — schematic, layout, gerbers | [CERN-OHL-P-2.0](https://cern-ohl.web.cern.ch/) |
| Example code | [MIT](LICENSE) |
| Documentation | [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/) |

---

*Rev 1.1 — drawn in KiCad 10.0.6*
