# STM32 Sensor Interface Board

Schematic-level hardware design for an automotive-telematics sensor node — built as a portfolio project targeting hardware design (schematic capture, power/clock/reset design, CAN bus integration, PCB bring-up prep).

## Overview

A STM32F103C8Tx-based board with regulated power, CAN bus connectivity, a UART peripheral header, and an SWD programming interface. Designed in KiCad, schematic-first with PCB placement in progress.

> **Status:** Schematic complete and ERC-clean. PCB footprint placement done; routing in progress (X/53 nets routed).

## Block Diagram

```
5V in --> AMS1117-3.3 --> 3.3V rail --> STM32F103C8Tx --> CAN transceiver (SN65HVD230) --> CANH/CANL
                                              |--> USART1 header (J2)
                                              |--> SWD programming header (J1)
                                              |--> 8MHz HSE crystal
```

## Design Decisions

**Power stage — AMS1117-3.3**
Linear regulator chosen for prototype simplicity over a switching converter. Tradeoff: lower efficiency (acceptable at this current draw, not appropriate for a battery-powered final product — noted as a known limitation, not an oversight). 10µF bulk caps on input and output per datasheet's typical application circuit.

**Decoupling**
One 100nF ceramic cap per digital VDD pin (pins 24, 36, 48). VDDA (pin 9, analog supply) gets its own 100nF + 1µF pair, isolated per ST's datasheet guidance — analog supply noise directly affects ADC accuracy, so it's decoupled separately from the digital rails.

**Reset (NRST) — R1, 10kΩ pull-up**
STM32 has an internal NRST pull-up, so this is a deliberate redundant external pull-up rather than a requirement — added for reliability margin and to make the reset node easier to probe during bring-up.

**Boot mode (BOOT0) — R2, 10kΩ pull-down**
Pulled low for normal flash-boot operation. Pulling BOOT0 high on next reset would instead enter the STM32 system bootloader — relevant if a UART-based firmware recovery path is ever needed.

**Crystal — 8MHz HSE, C8/C9 load caps**
Load capacitor values calculated from the crystal datasheet's specified load capacitance using `C_load = (C1 × C2)/(C1 + C2) + C_stray`, with C_stray assumed ~3–5pF for board parasitics. Resulted in 18pF caps on both OSC pins.

**CAN transceiver — SN65HVD230**
3.3V-native transceiver chosen to match the STM32's I/O voltage directly (no level shifting needed). CANH/CANL broken out to the edge for external bus connection; 120Ω termination to be added only if this node is a physical bus endpoint.

**Debug/expansion**
- J1: 5-pin SWD header (SWDIO, SWCLK, NRST, 3V3, GND) for programming and debug
- J2: 3-pin USART1 header for peripheral/serial debug access

## Bill of Materials

| Ref | Part | Value | Notes |
|---|---|---|---|
| U1 | STM32F103C8Tx | — | 64KB flash, LQFP-48 |
| U2 | AMS1117-3.3 | — | Linear LDO |
| U3 | SN65HVD230 | — | 3.3V CAN transceiver |
| C1, C2 | Ceramic/Tantalum | 10µF | LDO input/output bulk caps |
| C3, C4, C5 | Ceramic | 100nF | VDD decoupling (pins 24, 36, 48) |
| C6 | Ceramic | 100nF | VDDA decoupling |
| C7 | Ceramic | 1µF | VDDA decoupling |
| C8, C9 | Ceramic | 18pF | Crystal load caps |
| R1 | Resistor | 10kΩ | NRST pull-up (redundant, reliability margin) |
| R2 | Resistor | 10kΩ | BOOT0 pull-down (normal boot) |
| Y1 | Crystal | 8MHz | HSE clock source |
| J1 | Header, 1x05 | — | SWD programming |
| J2 | Header, 1x03 | — | USART1 |

## Bring-Up Test Plan

| Test | Expected Result | Instrument |
|---|---|---|
| LDO output voltage | 3.3V ± 3% | DMM |
| VDD ripple (under load) | < 50mV pk-pk | Oscilloscope |
| NRST idle state | High (~3.3V) | DMM |
| BOOT0 idle state | Low (~0V) | DMM |
| HSE crystal oscillation | Clean 8MHz, stable amplitude | Oscilloscope |
| CAN idle bus state | ~2.5V on CANH/CANL (recessive) | DMM |
| SWD connectivity | Debugger enumerates STM32 | ST-Link / OpenOCD |

**Debug order if the board doesn't boot:** power rail → clock (crystal) → reset (NRST/BOOT0) → peripheral bus (CAN/UART). Check each stage before moving to the next rather than probing randomly.

## Tools

- KiCad 8.x — schematic capture and PCB layout
- Reference datasheets: STM32F103C8Tx, AMS1117-3.3, SN65HVD230

## Repo Structure

```
/schematic     -- .kicad_sch source + exported PDF
/pcb           -- .kicad_pcb source + exported PDF (routing in progress)
/docs          -- this README, BOM, bring-up test plan
```

## Known Limitations / Next Steps

- [ ] Finish PCB routing (13 nets outstanding as of last checkpoint)
- [ ] Add GNSS module footprint (planned, not yet implemented)
- [ ] Add 120Ω CAN termination resistor if this board is confirmed as a bus endpoint
