# AlienWhoopF7 → ESP32 conversion

Replaces the STM32F405RGT6 / STM32F722RET6 (LQFP64) with an **ESP32-MINI-1**
module (ESP32-U4WDH inside: dual-core LX6 @ 240 MHz, 4 MB in-package flash,
40 MHz crystal on module). The GPIO map follows the OpenPilotESP32/NinjaPilot
port verified on real ESP32 hardware — gyro on VSPI 18/19/23, CS 5, INT 34,
outputs on 25/27/33/32, RMT on 4, console off UART0 — so the `esp32wroom`
firmware target drives this board with minimal changes (same silicon, same
IDF target).

**Status: generated schematic + placement-study board, nothing fabbed.**
`AlienWhoopF7-ESP32.sch` and `AlienWhoopF7-ESP32.brd` are generated,
programmatically verified Eagle 9 XML. The board has final placement, the
power pours, and all copper whose endpoints did not move; new connections are
airwires awaiting an interactive routing pass in Eagle. See the repo-level
CLAUDE.md before touching anything.

## Why the MINI-1 and not a WROOM

The first draft of this conversion specified an ESP32-WROOM-32E. The layout
pass falsified it: the motor-plug through-hole pins leave a **14.4 mm-wide
corridor** across this 29 mm board, and an 18 mm-wide WROOM does not fit in
any orientation without relocating motor plugs. The MINI-1 is 13.2 mm wide,
same silicon family, same IDF target — it drops into the corridor with
clearance to spare and its antenna area hangs **fully off the top board
edge** (the best RF case: zero copper under the antenna).

One trap came with it: the U4WDH's in-package flash consumes GPIO6/7/8/11
**and GPIO16/17** — IO16/IO17 do not exist on this module. In exchange the
module exposes **IO9/IO10**, which are the native U1RXD/U1TXD. The receiver
UART moved there.

The **ESP32-MINI-1U** (u.FL connector for an external antenna, no PCB
antenna, 5.5 mm shorter) shares the same land pattern — a population-time
choice on one footprint, in the spirit of this board's own F4/F7
passive-selection hack.

## Exact wiring: STM32 pin → ESP32 GPIO

Sensor bus (MPU-6500 stays on SPI — dedicated VSPI, no bus sharing with
blackbox):

| Function | was STM32 | now ESP32 GPIO | MINI-1 pad |
| --- | --- | --- | --- |
| Gyro SCLK | PA5 (SPI1) | IO18 (VSPI CLK) | 30 |
| Gyro MISO | PA6 | IO19 | 32 |
| Gyro MOSI | PA7 | IO23 | 31 |
| Gyro CS | PA4 | IO5 (+10K pullup R9) | 29 |
| Gyro INT | PC14 | IO34 (input-only, fine) | 9 |
| Blackbox flash SCLK | PB3 (SPI3) | IO14 (HSPI CLK) | 16 |
| Blackbox flash MISO | PB4 | IO12 | 17 |
| Blackbox flash MOSI | PB5 | IO13 | 18 |
| Blackbox flash CS | PA15 | IO15 (+10K pullup R8) | 19 |

Motors (FDMA410NZ gates, 10K pulldowns R1–R4 unchanged — all four GPIOs are
quiet at boot, no strapping functions):

| Motor | was | now | MINI-1 pad |
| --- | --- | --- | --- |
| M1 (PWM1) | PC9 | IO25 | 13 |
| M2 (PWM2) | PC8 | IO27 | 15 |
| M3 (PWM3) | PC7 | IO33 | 12 |
| M4 (PWM4) | PC6 | IO32 | 11 |

UARTs and pads:

| Pad | was | now | Notes |
| --- | --- | --- | --- |
| RX2 / TX2 | PA3/PA2 (USART2) | IO9 / IO10 (native U1RXD/U1TXD) | SBUS: **inverter U4 deleted** — ESP32 UART inverts RX in silicon (`UART_RXD_INV`) |
| RX3 / TX3 | PC11/PC10 | IO21 / IO22 | TX3 doubles as the IDF console (console stays OFF UART0) |
| TX4 pad | PA0 | renamed **IO4** (pad 22) | RMT: PPM-in or WS2812 |
| RX4 pad | PA1 | renamed **IO36** (pad 4) | input-only spare (RSSI / current sense) |
| USB D−/D+ | PA11/PA12 | via CH340C to IO3/IO1 (UART0, pads 35/36) | see below |
| BOOT pad | BOOT0 | IO0, pad 21 (R5 is now a **pullup**) | hold LOW at reset = download mode |
| LED0 | PC12 | IO26, pad 14 (active low, unchanged polarity) | |
| LED1 | PD2 | IO2, pad 20, **polarity flipped**: anode→IO2, cathode→GND | a pull-to-3V3 LED on IO2 would fight download mode; pull-down direction helps it |
| *(new)* VBAT sense | — | IO35 = ADC1_CH7, pad 10, via new R10/R11 10K/10K divider | the F7 never had battery telemetry; ADC1 because ADC2 is unusable with WiFi on |

USB: **the ESP32 has no USB peripheral** (that arrived with the S3). The
micro-USB connector now feeds a **CH340C** USB-UART bridge (U5) into UART0,
with the standard two-transistor auto-reset (Q6/Q7 MMBT3904, R6/R7 10K):
DTR→R6→Q6.B, Q6.C→EN, Q6.E→RTS; RTS→R7→Q7.B, Q7.C→IO0, Q7.E→DTR. esptool and
the GCS serial link both work through it. CH340C runs at 3.3 V (V3 tied to
VCC). The USB VBUS→battery-rail tie of the original board is retained, so
USB-only powering still works.

Reset/boot: EN (pad 8) gets a new 10K pullup (R12) and C9 becomes 1 µF (was
the 0.1 µF NRST cap). R5 (10K) moves from BOOT0-pulldown to IO0-pullup.

## Deleted outright

- **Y1** 8 MHz resonator — the module has its own 40 MHz crystal
- **U4** SN74LVC1G04 SBUS inverter + the RX2/INV pad — UART RX inversion is a
  register bit on ESP32
- **C7, C10, C15** — retired with honors. These three sites are the board's
  dual-MCU selection trick: they sit on exactly the LQFP64 pins whose meaning
  differs between the F405 and the F722. C15 hangs on the pin the F405 symbol
  calls PB11 — on a net the schematic itself names `VCAP_1_F7`, because on the
  F722 that same physical pin *is* VCAP_1. Populating 4.7 µF vs 0R per the
  original README reconfigures one PCB for either chip. The ESP32 has no VCAP
  pins (core regulation lives inside the module), so the mechanism has nothing
  left to select between — its spiritual successor is the module footprint
  itself, which takes a MINI-1 (PCB antenna) or MINI-1U (u.FL) as a
  population-time choice
- The practice of grounding unused MCU pins. **Unused ESP32 pins float.**

## Power

TPS63001 (fixed 3.3 V buck-boost from 1S) already on this board is exactly
right for an ESP32 — WiFi TX bursts (~400–500 mA) ride through buck-boost
regardless of battery sag. Added bulk: C17 22 µF on 3V3, C18 0.1 µF + C19
10 µF at the module. Reverse-polarity P-FET Q1 unchanged.

## The layout (AlienWhoopF7-ESP32.brd)

Generated as a transplant of the original board: every surviving part keeps
its original position and the GND/3V3 pours, battery/regulator copper and
motor-drain traces carry over. Verified collision-free at 0.2 mm pad
clearance, with through-hole punch-through and the module body keepout
checked. What moved, and why:

- **MINI-1 at (16.2, 21.9), top side** — the TH-free corridor between M4's
  pins (x≈8.0) and M2's pins (x≥24.2); antenna overhangs the top edge, edge
  GND pads sit 0.3 mm inside the outline
- **U2 gyro → bottom side, board center (14.6, 14.6)** — its old top-right
  corner is under the module; center is where a whoop gyro wants to be
  anyway. C3/C5 (VDDIO/REGOUT caps) and R9 follow it. Gyro orientation is a
  firmware constant — it changed, set it once
- C8, C9, C6 decouplers relocate out of the module zone; C4, C13, Q5, LED0,
  R5, C12, both pad rows, all connectors, FETs and the regulator stay put
- New bottom-side cluster: CH340C (U5) where the deleted inverter lived,
  Q6/Q7+R6/R7 auto-reset in the lower-center clear zone, R10/R11 divider by
  the battery, C18/C19 next to the module's 3V3 entry
- **Airwires (34 nets)**: everything module-bound. The fixed IO12 rule from
  CLAUDE.md applies when routing: no pullup on flash-MISO, it is the
  flash-voltage strap. Fixed while here: MPU VDDIO now actually ties to 3V3
  (the original net was a cap and the pin, nothing else)

Remaining for a human in Eagle: route the airwires (2 layers, mostly short),
repour, DRC with the original board's rule set (carried over in the file).

## Firmware

Target `esp32wroom` from OpenPilotESP32 (NinjaPilot port) — the MINI-1's
U4WDH is the same `esp32` IDF target. Needed changes:

1. `pios_servo` MCPWM backend needs a **brushed mode**: ~24 kHz duty-cycle
   drive, not 1–2 ms servo pulses. These are gate-driven MOSFETs.
2. MPU-6500 WHO_AM_I is 0x70 — already accepted by the port's driver.
3. Blackbox: MX25L6406E on HSPI wants a JEDEC-flash driver; internal ESP32
   flash writes stall XIP mid-flight-loop, which is exactly why the external
   chip stays.
4. UART pin map: receiver UART on IO9/IO10, `UART_RXD_INV` for SBUS.

Alternative: Espressif's ESP-Drone (Crazyflie-derived, built for brushed
ESP32 quads) — a working reference for MCPWM brushed drive at least.

## BOM delta

Removed: STM32F405RGT6/F722RET6, CSTCE8M00G55-R0, SN74LVC1G04DBVR,
C7/C10/C15 (the dual-MCU selection sites and their F4-build 0R alternates).

Added: ESP32-MINI-1-N4 (or -H4 for 105 °C; MINI-1U variants take the same
footprint); CH340C (SOP-16); MMBT3904 ×2; 10K 0603 ×7 (R6–R12); 0.1 µF ×2
(C16, C18); 10 µF 0805 (C19); 22 µF 0805 (C17).

Changed: C9 0.1 µF → 1 µF. Full list with MPNs: `AlienWhoopF7-ESP32_BOM.csv`.
