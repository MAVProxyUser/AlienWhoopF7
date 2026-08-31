# AlienWhoopF7 → ESP32 conversion

Replaces the STM32F405RGT6 / STM32F722RET6 (LQFP64) with an **ESP32-WROOM-32E**
module. The GPIO map is not arbitrary: it is the exact map the
OpenPilotESP32/NinjaPilot port verified on real hardware (gyro on VSPI
18/19/23, CS 5, INT 34; outputs on 25/27/33/32; RMT on 4; console off UART0),
so the `esp32wroom` firmware target drives this board with minimal changes.

**Status: paper design.** The schematic `AlienWhoopF7-ESP32.sch` is generated,
netlist-verified XML (Eagle 9). No board layout has been done, nothing has
been fabbed, nothing has flown. See the repo-level CLAUDE.md before touching
anything.

## Why this swap works

The F405 runs Betaflight at 168 MHz with an FPU. The ESP32 is 240 MHz
dual-core with an FPU, 520 KB SRAM — and adds WiFi/BT, which on a whoop means
ESP-NOW telemetry/RC and browser-based configuration with **no external
receiver**. The cost: no native USB (bridged, see below), and the flight
firmware is the NinjaPilot ESP32 port or ESP-Drone, not Betaflight — there is
no Betaflight ESP32 target.

## Exact wiring: STM32 pin → ESP32 GPIO

Sensor bus (MPU-6500 stays on SPI — dedicated VSPI, deterministic, no bus
sharing with blackbox):

| Function | was STM32 | now ESP32 GPIO | WROOM pin |
| --- | --- | --- | --- |
| Gyro SCLK | PA5 (SPI1) | IO18 (VSPI CLK) | 30 |
| Gyro MISO | PA6 | IO19 | 31 |
| Gyro MOSI | PA7 | IO23 | 37 |
| Gyro CS | PA4 | IO5 (+10K pullup R9) | 29 |
| Gyro INT | PC14 | IO34 (input-only, fine) | 6 |
| Blackbox flash SCLK | PB3 (SPI3) | IO14 (HSPI CLK) | 13 |
| Blackbox flash MISO | PB4 | IO12 | 14 |
| Blackbox flash MOSI | PB5 | IO13 | 16 |
| Blackbox flash CS | PA15 | IO15 (+10K pullup R8) | 23 |

Motors (FDMA410NZ gates, 10K pulldowns R1–R4 unchanged — all four GPIOs are
quiet at boot, no strapping functions):

| Motor | was | now | WROOM pin |
| --- | --- | --- | --- |
| M1 (PWM1) | PC9 | IO25 | 10 |
| M2 (PWM2) | PC8 | IO27 | 12 |
| M3 (PWM3) | PC7 | IO33 | 9 |
| M4 (PWM4) | PC6 | IO32 | 8 |

UARTs and pads:

| Pad | was | now | Notes |
| --- | --- | --- | --- |
| RX2 / TX2 | PA3/PA2 (USART2) | IO16 / IO17 (UART2) | SBUS: **inverter U4 deleted** — ESP32 UART inverts RX in silicon (`UART_RXD_INV`) |
| RX3 / TX3 | PC11/PC10 | IO21 / IO22 (UART1) | TX3 doubles as the IDF console (console stays OFF UART0) |
| TX4 pad | PA0 | renamed **IO4** | RMT: PPM-in or WS2812 |
| RX4 pad | PA1 | renamed **IO36** | input-only spare (RSSI / current sense) |
| USB D−/D+ | PA11/PA12 | via CH340C to IO3/IO1 (UART0) | see below |
| BOOT pad | BOOT0 | IO0 (R5 is now a **pullup**) | hold LOW at reset = download mode |
| LED0 | PC12 | IO26 (active low, unchanged polarity) | |
| LED1 | PD2 | IO2, **polarity flipped**: anode→IO2, cathode→GND | a pull-to-3V3 LED on IO2 would fight download mode; pull-down direction helps it |
| *(new)* VBAT sense | — | IO35 = ADC1_CH7 via new R10/R11 10K/10K divider | the F7 never had battery telemetry; ADC1 because ADC2 is unusable with WiFi on |

USB: **the ESP32 has no USB peripheral** (that arrived with the S3). The
micro-USB connector now feeds a **CH340C** USB-UART bridge (U5) into UART0,
with the standard two-transistor auto-reset (Q6/Q7 MMBT3904, R6/R7 10K):
DTR→R6→Q6.B, Q6.C→EN, Q6.E→RTS; RTS→R7→Q7.B, Q7.C→IO0, Q7.E→DTR. esptool and
the GCS serial link both work through it. CH340C runs at 3.3 V (V3 tied to
VCC). The USB VBUS→battery-rail tie of the original board is retained, so
USB-only powering still works.

Reset/boot: EN gets a new 10K pullup (R12) and C9 becomes 1 µF (was the 0.1 µF
NRST cap). R5 (10K) moves from BOOT0-pulldown to IO0-pullup.

## Deleted outright

- **Y1** 8 MHz resonator — the WROOM has its own 40 MHz crystal
- **U4** SN74LVC1G04 SBUS inverter + the RX2/INV pad — UART RX inversion is a
  register bit on ESP32
- **C7, C10, C15** — retired with honors. These three sites are the board's
  dual-MCU selection trick: they sit on exactly the LQFP64 pins whose meaning
  differs between the F405 and the F722. C15 hangs on the pin the F405 symbol
  calls PB11 — on a net the schematic itself names `VCAP_1_F7`, because on the
  F722 that same physical pin *is* VCAP_1. Populating 4.7 µF vs 0R per the
  original README reconfigures one PCB for either chip: a cap where the pin is
  VCAP, a jumper-to-ground where that pin is a dead GPIO. The ESP32 has no
  VCAP pins (core regulation lives inside the module), so the mechanism has
  nothing left to select between — its spiritual successor here is the module
  footprint itself, which takes a WROOM-32E (PCB antenna) or WROOM-32UE
  (u.FL, no antenna overhang needed) as a population-time choice
- The practice of grounding unused MCU pins. **Unused ESP32 pins float.**
  Grounding IO0/IO2/IO12/IO15 changes boot behavior; the schematic leaves
  every unused module pin unconnected on purpose.

## Power

TPS63001 (fixed 3.3 V buck-boost from 1S) already on this board is exactly
right for an ESP32 — WiFi TX bursts (~400–500 mA) ride through buck-boost
regardless of battery sag. Added bulk: C17 22 µF on 3V3, C18 0.1 µF + C19
10 µF at the module pins. Reverse-polarity P-FET Q1 unchanged.

## Layout notes (for whoever does the .brd)

- Module antenna end **must overhang the 29 mm board edge** or sit over a
  full copper keepout; never over the battery or motor wiring.
- The MPU-6500 keeps its position/orientation if possible; gyro orientation is
  a firmware constant either way.
- Keep the HSPI flash traces short; IO12 (flash MISO) must carry **no
  external pullup** — it is the flash-voltage strapping pin and floats low.
- Fixed schematic bug carried from the original: MPU VDDIO now actually
  connects to 3V3 (the original F7 net "VDDIO" was only a cap and the pin).

## Firmware

Target `esp32wroom` from OpenPilotESP32 (NinjaPilot port). Needed changes:

1. `pios_servo` MCPWM backend needs a **brushed mode**: 24 kHz duty-cycle
   drive, not 1–2 ms servo pulses. These are gate-driven MOSFETs.
2. MPU-6500 WHO_AM_I is 0x70 — already accepted by the port's driver.
3. Blackbox: MX25L6406E on HSPI wants a JEDEC-flash driver; internal ESP32
   flash writes stall XIP mid-flight-loop, which is exactly why the external
   chip stays.

Alternative: Espressif's ESP-Drone (Crazyflie-derived, built for brushed
ESP32 quads) — a working reference for MCPWM brushed drive at least.

## BOM delta

Removed: STM32F405RGT6/F722RET6, CSTCE8M00G55-R0, SN74LVC1G04DBVR,
C7/C10/C15 (4.7 µF ×3 and their F4-build 0R alternates).

Added: ESP32-WROOM-32E-N4; CH340C (SOP-16); MMBT3904 ×2; 10K 0603 ×7
(R6–R12); 0.1 µF ×2 (C16, C18); 10 µF 0805 (C19); 22 µF 0805 (C17).

Changed: C9 0.1 µF → 1 µF. Full list with MPNs: `AlienWhoopF7-ESP32_BOM.csv`.
